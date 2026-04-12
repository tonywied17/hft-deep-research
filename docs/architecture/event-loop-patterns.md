# Event Loop Patterns for HFT Systems

## Overview

The event loop is the heartbeat of an HFT system. Its design determines the baseline latency, jitter profile, and throughput ceiling. This document covers production-grade event loop patterns with implementation details.

---

## 1. The Busy-Poll Loop (Core Pattern)

Every HFT hot path is, at its core, a busy-poll loop. The CPU core spins continuously polling for work — never sleeps, never context-switches, never yields.

```cpp
// The fundamental HFT event loop
void hot_loop() {
    // Pre-startup: mlockall, CPU pin, isolcpus, hugepages already configured
    
    while (running_.load(std::memory_order_relaxed)) {
        // Phase 1: Poll network for incoming packets
        uint16_t n = nic_poll(rx_bufs, MAX_BURST);
        
        // Phase 2: Process each packet
        for (uint16_t i = 0; i < n; ++i) {
            MessageType type = parse_header(rx_bufs[i]);
            
            switch (type) {
                case MessageType::ADD_ORDER:
                    handle_add_order(rx_bufs[i]);
                    break;
                case MessageType::ORDER_EXECUTED:
                    handle_execution(rx_bufs[i]);
                    break;
                case MessageType::ORDER_DELETE:
                    handle_delete(rx_bufs[i]);
                    break;
                // ... all message types
            }
        }
        
        // Phase 3: Check command ring (config updates, shutdown signal)
        Command cmd;
        if (cmd_ring_.try_pop(cmd)) {
            handle_command(cmd);
        }
        
        // Phase 4: Periodic maintenance (every N iterations)
        if (UNLIKELY(++loop_count & MAINTENANCE_MASK == 0)) {
            publish_telemetry();
            check_timers();
        }
    }
}
```

### 1.1 Why Busy-Poll?

| Alternative | Problem |
|---|---|
| `epoll_wait()` | Syscall overhead: 200-1000 ns. Wakeup latency from sleep: 1-10 μs |
| `select()`/`poll()` | Even worse — O(n) scanning of fd set |
| Interrupt-driven | IRQ handler → softirq → context switch → wake process: 3-15 μs |
| `io_uring` | Kernel involvement, CQ polling overhead: 1-2 μs |

Busy-poll achieves: **< 100 ns** from packet arrival to first instruction of processing (with kernel bypass NIC).

### 1.2 The Cost of Busy-Poll

- **One full CPU core** dedicated per loop (100% utilization)
- **Power consumption:** ~100-200W per CPU (TDP), fully loaded
- **Thermal:** Requires active cooling; sustained frequency may throttle without proper C-state/P-state config

This is an acceptable cost. A single microsecond of latency advantage can be worth millions in annual PnL.

---

## 2. Single-Core Complete Loop

Everything — NIC poll, parse, book update, strategy, risk check, order send — runs in a single function on a single core within a single iteration of the loop.

```cpp
class CompleteTradingLoop {
    NicDriver nic_;
    ProtocolParser parser_;
    OrderBookManager books_;
    StrategyEngine strategy_;
    RiskEngine risk_;
    OrderGateway gateway_;
    
    alignas(64) uint64_t loop_count_ = 0;
    alignas(64) std::atomic<bool> running_{true};
    
public:
    [[gnu::hot, gnu::flatten]]
    void run() {
        Packet rx_bufs[64];
        
        while (LIKELY(running_.load(std::memory_order_relaxed))) {
            // 1. Poll NIC
            int n = nic_.poll(rx_bufs, 64);
            
            for (int i = 0; i < n; ++i) {
                // 2. Parse
                auto msgs = parser_.parse(rx_bufs[i]);
                
                for (auto& msg : msgs) {
                    // 3. Update book
                    BookUpdate update = books_.apply(msg);
                    
                    if (LIKELY(update.type != UpdateType::NONE)) {
                        // 4. Strategy evaluation
                        OrderAction action = strategy_.evaluate(update);
                        
                        if (action.type != ActionType::NONE) {
                            // 5. Risk check (inline)
                            if (LIKELY(risk_.check(action))) {
                                // 6. Send order
                                gateway_.send(action);
                            }
                        }
                    }
                }
            }
            
            // 7. Check for inbound execution reports
            int acks = nic_.poll_order_channel(rx_bufs, 64);
            for (int i = 0; i < acks; ++i) {
                auto report = parser_.parse_exec_report(rx_bufs[i]);
                strategy_.on_execution(report);
                risk_.on_execution(report);
            }
            
            ++loop_count_;
        }
    }
};
```

### 2.1 Cache Budget Analysis

For this to work, ALL hot-path state must fit in L1+L2 cache (~48KB + 512KB on modern Intel):

| Data | Size | Fits? |
|---|---|---|
| Top-of-book for 100 instruments (64B each) | 6.4 KB | L1 ✓ |
| Strategy state for 100 instruments (128B each) | 12.8 KB | L1 ✓ |
| Risk state (position array, 32B × 100) | 3.2 KB | L1 ✓ |
| Parser state + temp buffers | ~4 KB | L1 ✓ |
| SPSC ring metadata | ~1 KB | L1 ✓ |
| Code (hot functions) | ~16 KB | L1-I ✓ |
| **Total hot** | **~44 KB** | **L1 ✓** |
| Full book depth (10 levels × 100 instruments × 64B) | 64 KB | L2 ✓ |
| Order pool (1000 live orders × 64B) | 64 KB | L2 ✓ |

If trading 10,000+ instruments, you'll spill to L3 and need the pipeline pattern.

---

## 3. Pipeline Event Loop

Decompose into stages, each on its own core, connected by SPSC rings.

### 3.1 Stage Design

```cpp
template <typename Input, typename Output, typename Handler>
class PipelineStage {
    SPSCRing<Input, 4096>& input_;
    SPSCRing<Output, 4096>& output_;
    Handler handler_;
    
public:
    [[gnu::hot]]
    void run() {
        Input item;
        while (running) {
            if (input_.try_pop(item)) {
                Output result = handler_.process(item);
                if (result.valid()) {
                    output_.try_push(result);
                }
            }
        }
    }
};

// Instantiation
auto md_to_strategy = SPSCRing<BookUpdate, 4096>{};
auto strategy_to_orders = SPSCRing<OrderAction, 4096>{};

// Core 2: Market Data → BookUpdate
PipelineStage<Packet, BookUpdate, MDHandler> md_stage(nic_ring, md_to_strategy);

// Core 3: BookUpdate → OrderAction
PipelineStage<BookUpdate, OrderAction, Strategy> strat_stage(md_to_strategy, strategy_to_orders);

// Core 4: OrderAction → Wire
PipelineStage<OrderAction, void, OrderSender> order_stage(strategy_to_orders, null_ring);
```

### 3.2 Latency Overhead per Stage Hop

Each SPSC ring transfer involves a cross-core cache line transfer:

| CPU Architecture | L3 Hop (Same Socket) | Cross-Socket (QPI/UPI) |
|---|---|---|
| Intel Alder Lake | ~25-40 ns | N/A (desktop) |
| Intel Sapphire Rapids | ~30-50 ns | ~100-150 ns |
| AMD EPYC (Zen 4) | ~30-40 ns (same CCD) | ~80-120 ns (cross-CCD) |

**Rule of thumb:** Each pipeline stage adds ~30-80 ns on same-socket, same-CCD. Keep all pipeline cores on the same NUMA node and ideally same CCD/chiplet.

### 3.3 Batching in Pipeline Stages

To amortize per-hop overhead, process in batches:

```cpp
void run() {
    Input batch[BATCH_SIZE];
    while (running) {
        size_t n = input_.try_pop_batch(batch, BATCH_SIZE);
        for (size_t i = 0; i < n; ++i) {
            Output result = handler_.process(batch[i]);
            if (result.valid()) {
                output_.try_push(result);
            }
        }
    }
}
```

Batch size trade-off: larger batches improve throughput but increase latency for the first item in the batch. Typical: 1-8 items.

---

## 4. Timer Management in the Loop

### 4.1 Why No `sleep()` or `timerfd`

- `sleep()` / `usleep()` / `nanosleep()` — context switch, minimum ~50 μs wakeup time
- `timerfd_create` / `epoll_wait` — syscall overhead
- `clock_gettime()` — syscall (unless vDSO) — ~20-30 ns

### 4.2 TSC-Based Timer Wheel

Use the CPU's Time Stamp Counter (no syscall) and a hierarchical timer wheel:

```cpp
struct Timer {
    uint64_t expiry_tsc;
    uint32_t id;
    void (*callback)(uint32_t id);
};

class TSCTimerWheel {
    static constexpr size_t WHEEL_SIZE = 4096;
    static constexpr size_t WHEEL_MASK = WHEEL_SIZE - 1;
    
    std::array<std::vector<Timer>, WHEEL_SIZE> wheel_;
    uint64_t current_tick_ = 0;
    uint64_t tsc_per_tick_;  // Calibrated at startup
    
public:
    void add_timer(uint64_t delay_ns, uint32_t id, void (*cb)(uint32_t)) {
        uint64_t ticks = delay_ns * tsc_freq_ghz_;  // Convert ns to TSC ticks
        uint64_t expiry = __rdtsc() + ticks;
        size_t slot = (expiry / tsc_per_tick_) & WHEEL_MASK;
        wheel_[slot].push_back({expiry, id, cb});
    }
    
    void check_timers() {
        uint64_t now = __rdtsc();
        size_t slot = (now / tsc_per_tick_) & WHEEL_MASK;
        
        auto& timers = wheel_[slot];
        for (auto it = timers.begin(); it != timers.end(); ) {
            if (now >= it->expiry_tsc) {
                it->callback(it->id);
                it = timers.erase(it);
            } else {
                ++it;
            }
        }
    }
};
```

### 4.3 Periodic Tasks Without Timers

For tasks that need to run "approximately every N microseconds," use loop iteration counting:

```cpp
constexpr uint64_t TELEMETRY_INTERVAL = 1 << 16;  // Every 65536 iterations
constexpr uint64_t HEARTBEAT_INTERVAL = 1 << 20;  // Every ~1M iterations

if (UNLIKELY((loop_count & (TELEMETRY_INTERVAL - 1)) == 0)) {
    publish_telemetry();
}
if (UNLIKELY((loop_count & (HEARTBEAT_INTERVAL - 1)) == 0)) {
    send_heartbeat();
}
```

Cheap — single bitwise AND and branch (predicted not-taken 99.998% of the time).

---

## 5. Multi-Feed Aggregation

When receiving market data from multiple exchanges, a single event loop can poll multiple NICs or multicast groups:

```cpp
void multi_feed_loop() {
    // Poll all feeds in round-robin or priority order
    while (running) {
        // High-priority: primary exchange
        int n1 = nic_primary_.poll(bufs1, 64);
        process_packets(bufs1, n1, Exchange::PRIMARY);
        
        // Medium-priority: secondary exchanges
        int n2 = nic_secondary_.poll(bufs2, 32);
        process_packets(bufs2, n2, Exchange::SECONDARY);
        
        // Low-priority: reference data feeds
        int n3 = nic_reference_.poll(bufs3, 16);
        process_packets(bufs3, n3, Exchange::REFERENCE);
        
        // Strategy evaluation after all feeds processed
        for (auto& [id, inst] : updated_instruments_) {
            evaluate_strategy(inst);
        }
        updated_instruments_.clear();
    }
}
```

### 5.1 Feed Priority

Not all feeds are equal. Priority should reflect:
1. **Latency sensitivity** of the strategy on that feed
2. **Message rate** of the feed (high-rate feeds should be polled more frequently)
3. **Stale data cost** (what's the cost of processing feed B before feed A?)

For cross-exchange arbitrage, all feeds are equally critical and should be polled in the same iteration.

---

## 6. Jitter Analysis in Event Loops

### 6.1 Sources of Jitter

| Source | Magnitude | Mitigation |
|---|---|---|
| OS scheduler preemption | 1-100 μs | `isolcpus`, real-time priority |
| TLB miss (page fault) | 10-100 μs | `mlockall`, hugepages |
| Cache miss (L3 → DRAM) | 50-100 ns | Keep hot data in L1/L2 |
| NIC interrupt coalescing | 1-10 μs | Disable with `ethtool -C eth0 rx-usecs 0` |
| Transparent Hugepages (THP) compaction | 1-100 ms | Disable THP entirely |
| SMI (System Management Interrupt) | 50-500 μs | BIOS config, low-SMI platforms |
| TSC migration (if not invariant) | Variable | Use `invariant_tsc` CPU |
| Timer interrupts (LOC) | 1-5 μs per tick | `nohz_full` for tickless |
| RCU callbacks | 1-10 μs | `rcu_nocbs` for isolated cores |
| Kernel workqueues | Variable | `nohz_full` + CPU isolation |
| NMI watchdog | 1-5 μs | Disable: `nmi_watchdog=0` |

### 6.2 Measuring Jitter

```cpp
// Jitter measurement in the event loop
uint64_t prev_tsc = __rdtsc();
uint64_t max_gap = 0;
uint64_t histogram[64] = {};  // Power-of-2 buckets

while (running) {
    uint64_t now_tsc = __rdtsc();
    uint64_t gap = now_tsc - prev_tsc;
    
    if (gap > max_gap) max_gap = gap;
    
    // Log to histogram (bucket by magnitude)
    int bucket = gap > 0 ? (63 - __builtin_clzll(gap)) : 0;
    histogram[bucket]++;
    
    prev_tsc = now_tsc;
    
    // ... normal event loop work ...
}
```

Target: p99 loop iteration time < 1 μs, p99.9 < 5 μs, max < 50 μs.

---

## 7. Warm-Up Procedure

### 7.1 Why Warm-Up Matters

Before the trading session:
- JIT compilation (if using any JIT components): not applicable in C++ HFT
- CPU branch predictor: needs training on actual code paths
- Cache warming: hot data and code must be in L1/L2
- CPU frequency: must be at maximum (P-state 0)

### 7.2 Warm-Up Sequence

```cpp
void warm_up(const std::vector<Packet>& historical_packets) {
    // 1. Touch all memory to pull into cache hierarchy
    volatile char sink;
    for (size_t i = 0; i < sizeof(instruments_); i += 64) {
        sink = reinterpret_cast<char*>(&instruments_)[i];
    }
    
    // 2. Replay historical packets to train branch predictor
    for (int iteration = 0; iteration < 3; ++iteration) {
        for (const auto& pkt : historical_packets) {
            auto msg = parser_.parse(pkt);
            books_.apply(msg);  // Updates book, exercises all code paths
            // Don't send any real orders during warm-up
        }
    }
    
    // 3. Spin for a few million iterations to stabilize CPU frequency
    for (volatile int i = 0; i < 10'000'000; ++i) {}
    
    // 4. Reset all telemetry/histograms (discard warm-up noise)
    telemetry_.reset();
    
    // Now ready for live trading
}
```

# HFT System Architecture — End-to-End Design

## Overview

A production HFT system is a vertically-integrated, latency-optimized pipeline that transforms raw market data into order flow within single-digit microseconds. Every architectural decision — from process layout to memory allocation — is subordinated to the constraint of deterministic, minimal latency on the critical (hot) path.

---

## 1. Canonical System Topology

```
                          ┌──────────────────────────────────────────┐
                          │          Exchange Co-location Rack        │
                          │                                          │
  Market Data             │   ┌─────────┐    ┌──────────────────┐   │
  (UDP Multicast) ───────►│   │  NIC    │───►│  Market Data     │   │
                          │   │ (ef_vi) │    │  Handler (Core 2)│   │
                          │   └─────────┘    └────────┬─────────┘   │
                          │                           │ SPSC Ring    │
                          │                  ┌────────▼─────────┐   │
                          │                  │  Strategy Engine  │   │
                          │                  │  (Core 3)         │   │
                          │                  └────────┬─────────┘   │
                          │                           │ SPSC Ring    │
                          │   ┌─────────┐    ┌────────▼─────────┐   │
  Order Gateway  ◄────────│   │  NIC    │◄───│  Order Manager   │   │
  (TCP/Binary)            │   │ (ef_vi) │    │  (Core 4)        │   │
                          │   └─────────┘    └──────────────────┘   │
                          │                                          │
                          │   ┌──────────────────────────────────┐   │
                          │   │  Risk Engine (inline or Core 5)  │   │
                          │   └──────────────────────────────────┘   │
                          │                                          │
                          │   ┌──────────────────────────────────┐   │
                          │   │  Management / Monitoring (Cores  │   │
                          │   │  6-7, non-isolated)              │   │
                          │   └──────────────────────────────────┘   │
                          └──────────────────────────────────────────┘
```

## 2. Component Breakdown

### 2.1 Market Data Handler

**Responsibility:** Receive raw exchange packets, parse protocol messages, and maintain the normalized order book.

**Design Principles:**
- Runs on a dedicated, isolated CPU core
- Uses kernel bypass (ef_vi or DPDK) — zero syscalls on hot path
- Parses binary protocols (ITCH, CME MDP 3.0, PITCH) directly from packet buffers — no intermediate copy
- Maintains per-instrument order books in pre-allocated, cache-aligned structures
- Publishes book updates to the strategy engine via a lock-free SPSC ring buffer

**Data Flow:**
```
Wire → NIC RX descriptor ring → ef_vi/DPDK poll → Protocol parser 
  → Order book update → SPSC ring → Strategy
```

**Key Implementation Details:**
- **Zero-copy parsing:** Parse directly from the NIC's DMA buffer. Avoid `memcpy`.
- **Batch processing:** Process all available packets in a burst before yielding (`rte_eth_rx_burst` with batch size 32-64).
- **Template-based parsers:** Use compile-time dispatch for different message types to eliminate virtual function overhead.
- **Instrument lookup:** Use a flat array indexed by instrument ID (exchange-assigned) rather than a hash map. O(1) with no hash computation.

```cpp
// Example: Instrument lookup by exchange-assigned locate code
// NASDAQ ITCH assigns a 16-bit "stock locate" to each symbol
struct InstrumentState {
    OrderBook book;
    StrategyState strategy;
    RiskState risk;
};

// Direct-indexed array — no hashing, no pointer chasing
alignas(64) InstrumentState instruments[65536];  // Indexed by stock locate

void on_add_order(const itch::AddOrder& msg) {
    auto& inst = instruments[msg.stock_locate];  // Single array dereference
    inst.book.add(msg.order_ref, msg.side, msg.price, msg.shares);
}
```

### 2.2 Strategy Engine

**Responsibility:** Evaluate trading signals and generate order instructions.

**Design Principles:**
- Single-threaded (no locks, no contention)
- All state pre-allocated at startup
- Strategy logic is a pure function: `(BookUpdate, StrategyState) → OrderAction`
- No I/O, no allocation, no exceptions on the hot path
- Configurable via startup parameters, not runtime messages on the critical path

**Strategy Loop:**
```cpp
void strategy_loop(SPSCRing<BookUpdate>& updates_in, 
                   SPSCRing<OrderAction>& actions_out) {
    BookUpdate update;
    while (running) {
        if (updates_in.try_pop(update)) {
            auto& state = strategies[update.instrument_id];
            OrderAction action = state.evaluate(update);
            if (action.type != OrderAction::NONE) {
                actions_out.try_push(action);
            }
        }
        // Optional: _mm_pause() to reduce power/thermal, or spin continuously
    }
}
```

### 2.3 Order Manager / Gateway

**Responsibility:** Translate order actions into exchange-specific protocol messages, manage order lifecycle (new → ack → fill/cancel), and enforce pre-trade risk checks.

**Design Principles:**
- Maintains full order state machine per live order
- Pre-trade risk checks are evaluated inline (not a separate function call — the checks are branch conditions in the order submission path)
- Outbound orders are sent via kernel bypass (same NIC path as market data, or a dedicated order-entry NIC)
- Inbound execution reports (acks, fills, rejects) are fed back to the strategy engine

**Order State Machine:**
```
                ┌─── REJECTED
                │
PENDING_NEW ────┼─── NEW ──────┬─── PARTIALLY_FILLED ──── FILLED
                │              │
                │              ├─── PENDING_CANCEL ──── CANCELED
                │              │
                │              └─── PENDING_REPLACE ──── REPLACED
                │
                └─── EXPIRED
```

### 2.4 Risk Engine

**Responsibility:** Enforce position limits, loss limits, message rate limits, and provide kill-switch capability.

**Two deployment models:**

1. **Inline risk (preferred for speed):** Risk checks are embedded directly in the order manager's submission path. Adds 10-50 ns. No separate component.

2. **Sidecar risk (preferred for compliance):** Separate process/core that independently monitors positions and can pull the kill switch. Adds 100-500 ns if synchronous, or zero if asynchronous (audit-only).

### 2.5 Management Plane

**Responsibility:** Configuration, monitoring, logging, and control — everything that is NOT on the critical trading path.

**Runs on non-isolated cores.** Uses standard TCP/IPC. Can use the kernel network stack.

Components:
- **Telemetry collector:** Reads latency histograms, fill statistics, position snapshots from shared memory
- **Config server:** Pushes parameter updates (spread width, position limits) to the strategy engine via a command ring
- **Logger:** Writes events to disk asynchronously. Never blocks the hot path.
- **Alerting:** Monitors kill-switch thresholds, connectivity, gap detection

---

## 3. Memory Architecture

### 3.1 Shared Memory Layout

```
┌─────────────────────────────────────────────────┐
│  Hugepage Region (2MB pages, mlocked)           │
│                                                 │
│  ┌─────────────────────────────────────────┐    │
│  │  Order Books (per-instrument array)     │    │
│  │  64-byte aligned price levels           │    │
│  │  Pre-allocated for max instruments      │    │
│  └─────────────────────────────────────────┘    │
│                                                 │
│  ┌─────────────────────────────────────────┐    │
│  │  SPSC Ring Buffers (inter-core comms)   │    │
│  │  Producer/consumer on separate cache    │    │
│  │  lines (64-byte padding)               │    │
│  └─────────────────────────────────────────┘    │
│                                                 │
│  ┌─────────────────────────────────────────┐    │
│  │  Object Pools (orders, messages)        │    │
│  │  Fixed-size slab allocator              │    │
│  └─────────────────────────────────────────┘    │
│                                                 │
│  ┌─────────────────────────────────────────┐    │
│  │  Strategy State (positions, signals)    │    │
│  │  Per-instrument, cache-aligned          │    │
│  └─────────────────────────────────────────┘    │
│                                                 │
│  ┌─────────────────────────────────────────┐    │
│  │  Telemetry / Shared Metrics             │    │
│  │  Seqlock-published snapshots            │    │
│  └─────────────────────────────────────────┘    │
└─────────────────────────────────────────────────┘
```

### 3.2 NUMA Topology Awareness

On a dual-socket system:
- **All hot-path processes** and their memory must be on the SAME NUMA node as the NIC
- Cross-NUMA memory access adds ~100 ns penalty
- Verify with: `numactl --hardware` and `lstopo`

```bash
# Pin to NUMA node 0 (where NIC is connected)
numactl --cpunodebind=0 --membind=0 ./trading_engine

# Verify NIC's NUMA node
cat /sys/class/net/eth0/device/numa_node
```

---

## 4. Process Model Patterns

### 4.1 Monolithic Single-Process (Lowest Latency)

Everything in one process, one thread, one core:

```
┌──────────────────────────────────────────────┐
│  Single Core (isolated)                       │
│                                              │
│  while (true) {                              │
│      packets = poll_nic();                   │
│      for (pkt : packets) {                   │
│          msg = parse(pkt);                   │
│          book_update = update_book(msg);      │
│          action = evaluate_strategy(update);  │
│          if (action && risk_check(action))    │
│              send_order(action);              │
│      }                                        │
│  }                                            │
└──────────────────────────────────────────────┘
```

**Pros:** Absolute minimum latency. No inter-core transfer. All state in L1/L2 cache.  
**Cons:** Limited by single-core throughput. Strategy complexity constrained by CPU budget per tick.

### 4.2 Pipeline (Balanced Latency/Throughput)

Each stage on a dedicated core, connected by SPSC rings:

```
Core 2           Core 3            Core 4           Core 5
[NIC Poll]  →   [Parser/Book]  →  [Strategy]  →   [Order Mgr]
          SPSC              SPSC             SPSC
```

**Inter-stage latency:** ~20-80 ns per SPSC ring hop (cache-line transfer between cores).  
**Total pipeline latency:** ~60-240 ns overhead vs monolithic + parallelism benefit.

### 4.3 Fan-Out (Multi-Strategy)

Market data is broadcast to multiple independent strategy cores:

```
                   ┌──→ [Strategy A, Core 3] ──→ [Gateway A]
NIC → Parser ──────┼──→ [Strategy B, Core 4] ──→ [Gateway B]
   (Core 2)        └──→ [Strategy C, Core 5] ──→ [Gateway C]
```

Fan-out via seqlock-published book state or replicated SPSC rings.

### 4.4 Hybrid (Production Reality)

Most production systems combine patterns:

```
Fast Path (isolated cores, kernel bypass):
  NIC → [MD Parser + Book + Strategy + Risk → Order Send]    (Cores 2-5)

Slow Path (normal scheduler, kernel network):
  [Position Manager] ← fills from fast path via SPSC
  [Risk Monitor] ← snapshots from fast path via shared memory
  [Logger] ← events via SPSC ring to disk writer
  [Config/Control] ← operator commands via TCP
  [Recovery] ← snapshot/gap recovery via TCP
```

---

## 5. Startup & Shutdown Sequence

### 5.1 Startup

```
1.  Load configuration (instrument universe, strategy params, risk limits)
2.  Allocate hugepages; mlockall()
3.  Pre-allocate all objects: order books, object pools, SPSC rings, strategy state
4.  Initialize NIC (DPDK EAL / ef_vi driver init)
5.  Join multicast groups for market data channels
6.  Connect to snapshot/recovery channel
7.  Build initial book state from snapshot
8.  Transition to incremental feed; apply queued updates
9.  Verify book state (cross-check with snapshot)
10. Connect order entry session (FIX logon / OUCH session)
11. Sequence number sync
12. Enable strategy (begin generating orders)
13. Confirm heartbeats, begin telemetry publishing
```

### 5.2 Graceful Shutdown

```
1.  Disable strategy (stop new order generation)
2.  Cancel all open orders (bulk cancel)
3.  Wait for cancel confirmations (with timeout)
4.  Flatten remaining positions (if configured)
5.  Log final positions and PnL
6.  Disconnect order entry session (FIX logout)
7.  Leave multicast groups
8.  Flush logger
9.  Release resources (hugepages, NIC)
10. Exit
```

### 5.3 Emergency Kill Switch

Must be triggerable from:
- Inline risk (automatic: P&L breach, position breach, message rate breach)
- Operator console (manual)
- External monitoring system (heartbeat timeout)

Action:
1. Immediately stop sending new orders
2. Send mass cancel to all venues
3. Log the kill event with full state snapshot
4. Alert operations team

---

## 6. Failure Modes & Recovery

### 6.1 Market Data Gap

**Detection:** Sequence number discontinuity in ITCH/MDP messages.

**Recovery:**
1. Flag affected instruments as "stale" — strategy should widen spreads or halt
2. Request retransmission (if available) via TCP recovery channel
3. Or wait for next snapshot cycle and rebuild book
4. Resume normal operation when book is verified consistent

### 6.2 Order Entry Disconnection

**Detection:** TCP connection drop, heartbeat timeout, or FIX session-level error.

**Recovery:**
1. Kill switch — immediately stop strategy
2. Upon reconnection: sequence number negotiation, resend request
3. Reconcile order state: query exchange for open orders
4. Verify positions match internal state
5. Resume only after full reconciliation

### 6.3 Strategy Runaway

**Detection:** Message rate exceeds threshold, or rapid position accumulation.

**Response:**
1. Auto-triggered kill switch
2. All open orders cancelled
3. Requires manual operator intervention to re-enable

---

## 7. Deployment Architecture Across Venues

For a firm trading on multiple exchanges:

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│ Mahwah NJ        │     │ Carteret NJ      │     │ Aurora IL        │
│ (NYSE colo)      │     │ (NASDAQ colo)    │     │ (CME colo)       │
│                  │     │                  │     │                  │
│ [MD Handler]     │     │ [MD Handler]     │     │ [MD Handler]     │
│ [Strategy A]     │     │ [Strategy B]     │     │ [Strategy C]     │
│ [Order Mgr]      │     │ [Order Mgr]      │     │ [Order Mgr]      │
│ [Risk]           │     │ [Risk]           │     │ [Risk]           │
└───────┬──────────┘     └───────┬──────────┘     └───────┬──────────┘
        │ Microwave               │ Dark Fiber             │ Microwave
        └────────────┬────────────┘                        │
                     │                                     │
              ┌──────▼─────────────────────────────────────▼──┐
              │            Central Aggregator                   │
              │  (Position aggregation, global risk, reporting) │
              │  (Located at primary colo or separate DC)       │
              └────────────────────────────────────────────────┘
```

Each colo site is self-contained for its venue. Cross-venue strategies require inter-site communication with the lowest latency link available (microwave for Chicago ↔ NJ).

---

## 8. Technology Stack Reference

| Layer | Technology | Rationale |
|---|---|---|
| **Language** | C++ (17/20) | Zero-cost abstractions, deterministic memory, SIMD intrinsics |
| **Compiler** | GCC 12+ / Clang 15+ | PGO, LTO, `-O3 -march=native` |
| **NIC** | Solarflare X2522/X3522 | ef_vi kernel bypass, hardware timestamping |
| **Kernel bypass** | ef_vi (Solarflare) or DPDK | Sub-microsecond wire-to-app |
| **OS** | Linux (RHEL/Rocky 8/9) | `isolcpus`, `nohz_full`, hugepages |
| **IPC** | SPSC ring buffers | Lock-free, cache-friendly, zero-copy |
| **Persistence** | Memory-mapped files, custom binary logs | No filesystem overhead on hot path |
| **Config** | TOML/YAML + binary snapshot | Human-readable config, fast binary state |
| **Monitoring** | Prometheus + Grafana / custom | Pull-based metrics from shared memory |
| **Build** | CMake + Ninja | Fast incremental builds |
| **Testing** | Google Test + custom replay framework | Deterministic replay of pcap data |

---

## 9. Capacity Planning

| Metric | Typical Range | Planning Factor |
|---|---|---|
| Instruments traded | 100 – 10,000 | Determines book memory footprint |
| Peak messages/sec (inbound) | 500K – 5M | Determines parser throughput requirements |
| Orders/sec (outbound) | 1K – 100K | Determines gateway capacity |
| Live orders (open at once) | 100 – 50,000 | Determines order state memory |
| Position entries | 100 – 5,000 | Determines risk check lookup |
| Tick-to-trade target | 1 – 10 μs | Determines architecture pattern |
| Daily data volume | 10 – 100 GB | Determines logging/storage |
| Cores needed (hot path) | 2 – 8 | Determines CPU selection |
| Cores needed (management) | 2 – 4 | Can share with OS |
| Memory (hot path) | 2 – 16 GB | Hugepages, mlocked |

# Deployment, Colocation & Production Operations

---

## 1. Data Center & Colocation

### 1.1 Major Exchange Co-Location Sites

| Exchange | Data Center | Location | Provider |
|---|---|---|---|
| NYSE/NYSE Arca | NY5 | Mahwah, NJ | NYSE (self) |
| NASDAQ | NY5 / DC3 | Carteret, NJ | Equinix |
| CME Group | Aurora | Aurora, IL | CME (self) |
| Cboe | NY5 | Secaucus, NJ → Equinix | Equinix |
| ICE | NY5 | Multiple | Self-managed |
| LSE | LD4 | Basildon, UK | Equinix |
| Eurex | FR2 | Frankfurt, DE | Equinix |
| TSE | TY3 | Tokyo, JP | Equinix |

### 1.2 Colocation Setup

```
Physical Setup:
  ┌──────────────────────────────────────────────┐
  │ Exchange Matching Engine (ME)                  │
  │  Cross-connect: ~1-5m fiber                    │
  └───────┬──────────────────────────────────────┘
          │ 
  ┌───────▼──────────────────────────────────────┐
  │ Your Rack (42U cabinet)                        │
  │                                                │
  │  ┌─────────┐  ┌─────────┐  ┌─────────┐       │
  │  │ MD Server│  │Strategy │  │ OE Server│       │
  │  │ (1U)    │  │ (1U)    │  │ (1U)    │       │
  │  └────┬────┘  └────┬────┘  └────┬────┘       │
  │       └──────┬─────┘       ┌────┘            │
  │       ┌──────▼─────────────▼────┐            │
  │       │   Network Switch (1U)   │            │
  │       │   (Cut-through, <1μs)   │            │
  │       └─────────────────────────┘            │
  │                                                │
  │  Power: Dual PDU (A+B feed)                    │
  │  Cooling: Rear-door heat exchanger             │
  │  UPS: Battery + generator backup               │
  └────────────────────────────────────────────────┘
  
Cross-Connect Options:
  - Fiber: ~$300-500/month for exchange cross-connect
  - Copper: Possible but higher latency
  - Managed fiber: Exchange-provided, guaranteed equal length
  
Total co-lo cost: $10,000 - $50,000/month per cabinet
  (Varies by exchange, location, power requirements)
```

### 1.3 Inter-DC Connectivity

```
NJ ↔ Chicago:
  Fiber:          ~13.1 ms RTT (speed of light in glass)
  Microwave:      ~8.0 ms RTT  (speed of light in air, line-of-sight)
  Millimeter wave: ~8.2 ms RTT (slightly slower but more bandwidth)
  
  Providers: McKay Brothers, Anova Technologies, Jump Trading

NJ ↔ London:
  Submarine fiber: ~65 ms RTT
  No microwave option (ocean)
  
  Hibernia Express: lowest-latency transatlantic cable

NJ ↔ Tokyo:
  ~160 ms RTT (submarine cable via Pacific)
```

---

## 2. Operating System & Kernel Tuning

### 2.1 CPU Isolation

```bash
# GRUB boot parameters (add to /etc/default/grub)
GRUB_CMDLINE_LINUX="isolcpus=2-7 nohz_full=2-7 rcu_nocbs=2-7 \
  intel_pstate=disable processor.max_cstate=0 idle=poll \
  transparent_hugepage=never audit=0 nosoftlockup \
  tsc=reliable clocksource=tsc"

# isolcpus=2-7:     Remove CPUs 2-7 from scheduler
# nohz_full=2-7:    Disable timer ticks on isolated cores
# rcu_nocbs=2-7:    Offload RCU callbacks from isolated cores
# intel_pstate=disable: Allow manual frequency control
# processor.max_cstate=0: Prevent deep C-states
# idle=poll:        Busy-wait instead of sleep (uses more power)
# transparent_hugepage=never: Prevent THP compaction jitter
```

### 2.2 IRQ Affinity

```bash
# Pin all IRQs to housekeeping cores (0-1)
for irq in /proc/irq/*/smp_affinity; do
    echo 3 > "$irq" 2>/dev/null  # Binary mask: cores 0+1
done

# Specifically pin NIC IRQs to the MD processing core
# Find NIC IRQ numbers
cat /proc/interrupts | grep eth0

# Pin NIC RX queue 0 to core 2
echo 4 > /proc/irq/XXX/smp_affinity  # Binary mask: core 2
```

### 2.3 CPU Frequency

```bash
# Set CPU governor to performance (fixed high frequency)
for cpu in /sys/devices/system/cpu/cpu[2-7]/cpufreq/scaling_governor; do
    echo "performance" > "$cpu"
done

# Disable turbo boost (causes frequency jitter on neighboring cores)
echo 1 > /sys/devices/system/cpu/intel_pstate/no_turbo

# OR with msr-tools:
wrmsr -p 2 0x1a0 0x4000850089  # Disable turbo on core 2
```

### 2.4 Memory Configuration

```bash
# Hugepages (2MB)
echo 4096 > /proc/sys/vm/nr_hugepages  # 8 GB of hugepages

# 1GB hugepages (requires boot param: hugepagesz=1G hugepages=8)
# Must be allocated at boot time

# Disable NUMA balancing (prevents page migration jitter)
echo 0 > /proc/sys/kernel/numa_balancing

# Disable swap (never swap in HFT)
swapoff -a

# Lock all application memory
# In application code: mlockall(MCL_CURRENT | MCL_FUTURE)
```

### 2.5 Network Stack Tuning

```bash
# Increase socket buffer sizes
sysctl -w net.core.rmem_max=67108864
sysctl -w net.core.rmem_default=67108864
sysctl -w net.core.wmem_max=67108864
sysctl -w net.core.netdev_max_backlog=100000

# TCP tuning (for FIX connections)
sysctl -w net.ipv4.tcp_no_metrics_save=1
sysctl -w net.ipv4.tcp_tw_reuse=1
sysctl -w net.ipv4.tcp_timestamps=0
sysctl -w net.ipv4.tcp_sack=0
sysctl -w net.ipv4.tcp_low_latency=1

# Disable Nagle algorithm (application should also set TCP_NODELAY)
# Done in application: setsockopt(fd, IPPROTO_TCP, TCP_NODELAY, ...)

# Busy polling
sysctl -w net.core.busy_poll=50
sysctl -w net.core.busy_read=50
```

---

## 3. Time Synchronization

### 3.1 PTP (Precision Time Protocol / IEEE 1588v2)

```
Architecture:
  Grandmaster Clock (exchange timestamping source)
      │
      ▼
  PTP-Aware Network Switch (boundary clock)
      │
      ▼
  PTP Hardware Timestamping NIC → System Clock
  
Accuracy: < 100 ns with hardware timestamps
  Without HW timestamps: ~1-10 μs
  With HW timestamps: < 50 ns
  
Software: linuxptp (ptp4l + phc2sys)
```

```bash
# Run PTP daemon
ptp4l -i eth0 -m -H -s  # -H = hardware timestamping, -s = slave only

# Synchronize system clock from PTP hardware clock
phc2sys -a -r -r -m

# Verify synchronization
pmc -u -b 0 'GET PORT_DATA_SET'
pmc -u -b 0 'GET CURRENT_DATA_SET'  # Check offsetFromMaster
```

### 3.2 TSC (Time Stamp Counter)

```cpp
// Read TSC directly — sub-nanosecond resolution
inline uint64_t rdtsc() {
    unsigned int lo, hi;
    __asm__ __volatile__("rdtsc" : "=a"(lo), "=d"(hi));
    return ((uint64_t)hi << 32) | lo;
}

// RDTSCP — serializing version (waits for prior instructions)
inline uint64_t rdtscp() {
    unsigned int lo, hi, aux;
    __asm__ __volatile__("rdtscp" : "=a"(lo), "=d"(hi), "=c"(aux));
    return ((uint64_t)hi << 32) | lo;
}

// Convert TSC ticks to nanoseconds
// Calibrate tsc_freq at startup using clock_gettime
struct TSCCalibration {
    uint64_t tsc_freq;  // TSC ticks per second
    
    void calibrate() {
        struct timespec ts_start, ts_end;
        
        clock_gettime(CLOCK_MONOTONIC, &ts_start);
        uint64_t tsc_start = rdtscp();
        
        // Busy-wait ~100ms
        struct timespec req = {0, 100000000};
        nanosleep(&req, nullptr);
        
        clock_gettime(CLOCK_MONOTONIC, &ts_end);
        uint64_t tsc_end = rdtscp();
        
        uint64_t ns_elapsed = (ts_end.tv_sec - ts_start.tv_sec) * 1000000000ULL +
                               (ts_end.tv_nsec - ts_start.tv_nsec);
        uint64_t tsc_elapsed = tsc_end - tsc_start;
        
        tsc_freq = tsc_elapsed * 1000000000ULL / ns_elapsed;
    }
    
    uint64_t tsc_to_ns(uint64_t tsc) const {
        return tsc * 1000000000ULL / tsc_freq;
    }
};
```

---

## 4. FPGA Deployment

### 4.1 FPGA Use Cases in HFT

```
Tier 1: Market Data Parsing (most common)
  - Parse ITCH/MDP/FIX directly on the NIC/FPGA
  - Build order book in fabric
  - Latency: < 1 μs wire-to-signal
  
Tier 2: Simple Strategy Logic
  - Implement crossing logic (ETF arb, index arb)
  - Price threshold triggers
  - Latency: < 500 ns wire-to-wire
  
Tier 3: Full Strategy on FPGA
  - Complete tick-to-trade in hardware
  - Market making, simple stat arb
  - Latency: < 1 μs tick-to-order
  
Tier 4: Network Infrastructure
  - FPGA-based switches (Arista 7130)
  - Time stamping appliances
  - Risk check appliances
```

### 4.2 Hardware Options

| FPGA Board | Vendor | Chip | Network | Use Case |
|---|---|---|---|---|
| Alveo U250 | Xilinx/AMD | UltraScale+ | 2×100GbE | General HFT |
| Alveo U55C | Xilinx/AMD | UltraScale+ HBM | 2×100GbE | Market data (high memory) |
| Arista 7130 | Arista | Xilinx | 48×10GbE + FPGA | Switch + compute |
| Silicom N5010 | Silicom | Intel Agilex | 4×100GbE | Intel ecosystem |
| ExaNIC X25 | Cisco/Exablaze | Xilinx | 2×25GbE | Ultra-low latency NIC |

### 4.3 FPGA Development Flow

```
High-Level Synthesis (HLS) — C++ → FPGA:
  
  1. Write algorithm in C++ with HLS pragmas
  2. Vivado HLS compiles to RTL (Verilog/VHDL)
  3. Vivado synthesizes and routes on target FPGA
  4. Generate bitstream → load onto FPGA
  
  Advantages: Familiar language, faster development
  Disadvantages: Less control, sometimes suboptimal

Direct RTL (Verilog/VHDL):
  
  1. Write pipeline stages in RTL
  2. Synthesize → place & route
  3. Timing closure (meet clock frequency target)
  4. Generate bitstream
  
  Advantages: Full control, optimal performance
  Disadvantages: Longer development, hardware expertise required
```

### 4.4 Simplified ITCH Parser in HLS

```cpp
// Xilinx HLS pseudocode for ITCH Add Order parser
void itch_parser(
    hls::stream<ap_axiu<64,0,0,0>>& input,   // AXI stream from NIC
    hls::stream<OrderUpdate>& output           // Parsed order updates
) {
    #pragma HLS INTERFACE axis port=input
    #pragma HLS INTERFACE axis port=output
    #pragma HLS PIPELINE II=1  // Process one word per clock cycle
    
    ap_axiu<64,0,0,0> word;
    input.read(word);
    
    uint8_t msg_type = word.data(7, 0);
    
    if (msg_type == 'A') {  // Add Order
        // Read remaining words (36 bytes = ~5 64-bit words)
        // Extract fields at known bit positions
        ap_axiu<64,0,0,0> w1, w2, w3, w4;
        input.read(w1);
        input.read(w2);
        input.read(w3);
        input.read(w4);
        
        OrderUpdate update;
        update.order_ref = w1.data(63, 0);
        update.side = w2.data(7, 0);
        update.shares = w2.data(39, 8);
        update.price = w3.data(63, 32);
        update.stock_locate = word.data(23, 8);
        
        output.write(update);
    }
}
```

---

## 5. Monitoring & Telemetry

### 5.1 Latency Measurement

```cpp
struct LatencyTracker {
    // Historgram bins (logarithmic, nanoseconds)
    // 0-100ns, 100-200, 200-500, 500-1μs, 1-2μs, 2-5μs, 5-10μs, 10+μs
    std::atomic<uint64_t> bins[16] = {};
    std::atomic<uint64_t> min_ns{UINT64_MAX};
    std::atomic<uint64_t> max_ns{0};
    std::atomic<uint64_t> total_ns{0};
    std::atomic<uint64_t> count{0};
    
    void record(uint64_t latency_ns) {
        int bin = latency_to_bin(latency_ns);
        bins[bin].fetch_add(1, std::memory_order_relaxed);
        
        // Track min/max (relaxed — approximate is fine for monitoring)
        uint64_t cur_min = min_ns.load(std::memory_order_relaxed);
        while (latency_ns < cur_min && 
               !min_ns.compare_exchange_weak(cur_min, latency_ns));
        
        uint64_t cur_max = max_ns.load(std::memory_order_relaxed);
        while (latency_ns > cur_max && 
               !max_ns.compare_exchange_weak(cur_max, latency_ns));
        
        total_ns.fetch_add(latency_ns, std::memory_order_relaxed);
        count.fetch_add(1, std::memory_order_relaxed);
    }
    
    // Compute percentiles from histogram
    uint64_t percentile(double p) const {
        uint64_t total = count.load();
        uint64_t target = static_cast<uint64_t>(total * p);
        uint64_t cumulative = 0;
        
        for (int i = 0; i < 16; ++i) {
            cumulative += bins[i].load();
            if (cumulative >= target) {
                return bin_to_upper_bound(i);
            }
        }
        return max_ns.load();
    }
    
private:
    static int latency_to_bin(uint64_t ns) {
        if (ns < 100)    return 0;
        if (ns < 200)    return 1;
        if (ns < 500)    return 2;
        if (ns < 1000)   return 3;
        if (ns < 2000)   return 4;
        if (ns < 5000)   return 5;
        if (ns < 10000)  return 6;
        if (ns < 20000)  return 7;
        if (ns < 50000)  return 8;
        if (ns < 100000) return 9;
        return 10;
    }
    
    static uint64_t bin_to_upper_bound(int bin) {
        const uint64_t bounds[] = {100, 200, 500, 1000, 2000, 5000, 
                                   10000, 20000, 50000, 100000, UINT64_MAX};
        return bounds[bin];
    }
};
```

### 5.2 Key Metrics to Monitor

```
Latency Metrics:
  - Tick-to-trade (market data in → order out): Target < 5 μs
  - Parse latency (packet in → structured data): Target < 200 ns
  - Order round-trip (send → ack from exchange): Target < 20 μs
  - P99 latency (99th percentile): Should be < 2x median
  
System Metrics:
  - CPU utilization per core (should be 100% for polling cores)
  - Cache miss rate (L1/L2/L3): Should be < 1% on hot path
  - Context switches per second: Should be 0 on isolated cores
  - Page faults: Should be 0 after startup (mlockall)
  - IRQ count on trading cores: Should be 0

Business Metrics:
  - Realized PnL (real-time, with <1s lag)
  - Position per instrument
  - Fill rate (posted orders / filled orders)
  - Exchange message rate (orders/second, messages/second)
  - Queue position distribution
  - Adverse selection per fill
```

### 5.3 Alerting Thresholds

```
CRITICAL (immediate page):
  - Kill switch triggered
  - Position limit breach
  - Loss limit breach (> 80% of max loss)
  - Strategy heartbeat timeout
  - Exchange disconnect
  - Sequence gap (missed market data)
  
WARNING (Slack/email):
  - P99 latency > 2x baseline
  - Fill rate below threshold
  - Unusual cancel ratio
  - Market data staleness > 1 second
  - System resource anomaly (CPU, memory, network)
  
INFO (dashboard):
  - PnL updates
  - Position changes
  - Order statistics
  - Market regime changes
```

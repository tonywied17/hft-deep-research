# Cache Optimization for HFT

## Overview

Cache performance is the dominant factor in HFT software latency. A single L3 miss costs ~50-100 ns — comparable to the entire tick-to-trade budget. This document covers every aspect of CPU cache optimization relevant to trading systems.

---

## 1. CPU Cache Hierarchy (x86-64)

### 1.1 Modern Intel/AMD Cache Specs

| Level | Size (per core) | Latency | Associativity | Line Size | Bandwidth |
|---|---|---|---|---|---|
| L1-D | 48 KB (Intel 12th+), 32 KB (AMD) | ~1 ns (4-5 cycles) | 12-way | 64 B | ~2 TB/s |
| L1-I | 32 KB | ~1 ns | 8-way | 64 B | ~1 TB/s |
| L2 | 1.25 MB (Intel), 512 KB–1 MB (AMD) | ~3-5 ns | 10-20 way | 64 B | ~500 GB/s |
| L3 (shared) | 2-4 MB/core (Intel), 4 MB/core (AMD Zen4) | ~10-20 ns | 12-16 way | 64 B | ~200 GB/s |
| DRAM | 64-512 GB | ~50-100 ns | N/A | N/A | ~50 GB/s |

### 1.2 Cache Line — The Fundamental Unit

Everything moves between cache levels in **64-byte cache lines**. Understanding this is non-negotiable:

```
Cache line (64 bytes):
┌────────────────────────────────────────────────────────────────┐
│ Byte 0 │ Byte 1 │ ... │ Byte 62 │ Byte 63 │
└────────────────────────────────────────────────────────────────┘

Accessing ANY byte in the line loads the ENTIRE 64-byte line into cache.
```

**Implications:**
- Accessing `struct.field_a` at offset 0 also pulls `field_b` through `field_g` (if they're within the same 64 bytes) — **spatial locality**
- If you only need `field_a`, the rest is wasted cache space — **cache pollution**
- If two threads modify different fields in the same cache line — **false sharing**

---

## 2. False Sharing — The Silent Killer

### 2.1 The Problem

When two CPU cores write to different variables that reside on the same cache line, the hardware cache coherence protocol (MESI/MESIF/MOESI) bounces the cache line between cores:

```
Core 0 writes var_a (offset 0)  → Line transitions to Modified on Core 0
Core 1 writes var_b (offset 8)  → Core 0's line Invalidated, Core 1 gets Exclusive, writes
Core 0 writes var_a again       → Core 1's line Invalidated, Core 0 reclaims
...repeat: ~40-70 ns per bounce
```

This can cause a **10-100x slowdown** compared to when each variable is on its own cache line.

### 2.2 Detection

```bash
# perf tool to detect false sharing
perf c2c record -g -- ./trading_engine
perf c2c report --call-graph --stdio

# Look for "HITM" (Hit Modified — cache line was modified on remote core)
# High HITM counts on specific addresses = false sharing
```

### 2.3 Fixing False Sharing

```cpp
// Pattern 1: alignas(64) on each hot variable
struct alignas(64) ProducerState {
    std::atomic<uint64_t> write_pos{0};
    // 56 bytes of padding implicit from alignas
};

struct alignas(64) ConsumerState {
    std::atomic<uint64_t> read_pos{0};
};

// Pattern 2: Explicit padding struct
struct PaddedAtomic {
    std::atomic<uint64_t> value{0};
    char padding[64 - sizeof(std::atomic<uint64_t>)];
};

// Pattern 3: C++17 hardware_destructive_interference_size
#include <new>
struct alignas(std::hardware_destructive_interference_size) AlignedCounter {
    std::atomic<uint64_t> count{0};
};

// Pattern 4: Group read-only and write-heavy data separately
struct OrderBook {
    // Cache line 1: Written by market data handler
    alignas(64) int64_t best_bid_price;
    int32_t best_bid_qty;
    int64_t best_ask_price;
    int32_t best_ask_qty;
    uint64_t last_update_tsc;
    
    // Cache line 2: Written by strategy engine
    alignas(64) int32_t our_bid_qty;
    int32_t our_ask_qty;
    double theoretical_value;
    double spread_target;
    
    // Cache line 3: Written by risk engine
    alignas(64) int32_t net_position;
    double unrealized_pnl;
    double realized_pnl;
};
```

---

## 3. Data Layout Optimization

### 3.1 Structure of Arrays (SoA) vs Array of Structures (AoS)

```cpp
// === AoS: Array of Structures ===
struct Instrument_AoS {
    int64_t  bid_price;      // 8 bytes
    int64_t  ask_price;      // 8 bytes  
    int32_t  bid_qty;        // 4 bytes
    int32_t  ask_qty;        // 4 bytes
    double   fair_value;     // 8 bytes
    double   spread;         // 8 bytes
    int32_t  position;       // 4 bytes
    uint32_t flags;          // 4 bytes
    char     symbol[8];      // 8 bytes
    // Total: 56 bytes — almost one cache line per instrument
};
Instrument_AoS instruments[10000];

// Scanning all fair_values:
// Loads 56 bytes per instrument, uses only 8 → 85% cache waste
for (int i = 0; i < 10000; ++i) {
    if (instruments[i].fair_value > threshold) { ... }
}
// Accesses 10000 * 56 = 560 KB of data = blows L2, hits L3

// === SoA: Structure of Arrays ===
struct Instruments_SoA {
    int64_t  bid_prices[10000];    // 80 KB
    int64_t  ask_prices[10000];    // 80 KB
    int32_t  bid_qtys[10000];      // 40 KB
    int32_t  ask_qtys[10000];      // 40 KB
    double   fair_values[10000];   // 80 KB
    double   spreads[10000];       // 80 KB
    int32_t  positions[10000];     // 40 KB
    uint32_t flags_arr[10000];     // 40 KB
};
Instruments_SoA instruments;

// Scanning all fair_values:
// Loads ONLY fair_values array = 80 KB → fits in L2
for (int i = 0; i < 10000; ++i) {
    if (instruments.fair_values[i] > threshold) { ... }
}
// 7x less memory accessed. SIMD-vectorizable.
```

### 3.2 Hybrid: Hot/Cold Splitting

For HFT, use a hybrid approach — put the fields accessed on the hot path into a dense "hot" struct, and everything else into a "cold" struct:

```cpp
// Hot data: accessed every tick
struct alignas(64) InstrumentHot {
    int64_t bid_price;       // 8
    int64_t ask_price;       // 8
    int32_t bid_qty;         // 4
    int32_t ask_qty;         // 4
    double  fair_value;      // 8
    int32_t position;        // 4
    uint32_t flags;          // 4
    uint64_t last_update;    // 8
    // Padding to 64 bytes
    char pad[16];
};
static_assert(sizeof(InstrumentHot) == 64);  // One cache line exactly

// Cold data: accessed rarely
struct InstrumentCold {
    char symbol[16];
    double realized_pnl;
    double max_position;
    double risk_limit;
    uint64_t exchange_id;
    uint32_t lot_size;
    uint32_t tick_size;
    // ... more fields
};

// Arrays
InstrumentHot hot_instruments[10000];  // 640 KB — fits in L3
InstrumentCold cold_instruments[10000]; // Separate, rarely touched
```

---

## 4. Prefetching

### 4.1 Hardware Prefetch

Modern CPUs have hardware prefetchers that detect sequential access patterns and prefetch ahead. Works well for:
- Linear array scans
- Stride-1 access patterns
- Some constant-stride patterns

Does NOT work for:
- Pointer chasing (linked lists, trees)
- Random access patterns
- Complex computed indices

### 4.2 Software Prefetch

```cpp
// __builtin_prefetch(addr, rw, locality)
//   rw: 0 = read, 1 = write
//   locality: 0 = no temporal locality (use once), 3 = high locality (keep in all caches)

// Pattern: Prefetch the NEXT iteration's data in the CURRENT iteration
void process_orders(const Order* orders, size_t n) {
    for (size_t i = 0; i < n; ++i) {
        // Prefetch data for 8 iterations ahead
        if (i + 8 < n) {
            __builtin_prefetch(&orders[i + 8], 0, 1);  // Read, moderate locality
        }
        process(orders[i]);
    }
}

// Pattern: Prefetch into L1 for pointer-chasing
void traverse_book(PriceLevel* level) {
    while (level) {
        PriceLevel* next = level->next;
        if (next) __builtin_prefetch(next, 0, 3);  // Prefetch next node
        process_level(level);
        level = next;
    }
}

// Pattern: Prefetch for hash table lookup
template <typename V>
V* HashTable<V>::find(uint64_t key) {
    size_t bucket = hash(key) & mask_;
    __builtin_prefetch(&buckets_[bucket], 0, 1);
    // ... some other work (allows prefetch to complete) ...
    return &buckets_[bucket];  // Data is now in L1
}
```

### 4.3 Non-Temporal Stores

For data that won't be read again soon (e.g., writing to a log buffer), use non-temporal stores to bypass the cache entirely:

```cpp
#include <immintrin.h>

// Write 64 bytes without polluting cache
void write_log_entry(void* dest, const void* src) {
    __m256i* d = (__m256i*)dest;
    const __m256i* s = (const __m256i*)src;
    
    _mm256_stream_si256(d,     _mm256_load_si256(s));
    _mm256_stream_si256(d + 1, _mm256_load_si256(s + 1));
    _mm_sfence();  // Ensure streaming stores are visible
}
```

---

## 5. Cache-Friendly Data Structures

### 5.1 Order Book — Price Level Array

Instead of a tree (red-black, AVL) for price levels, use a sorted array with direct indexing:

```cpp
// Direct-indexed price level array
// For instruments with known tick size, map price → array index
class DirectIndexedBook {
    static constexpr int MAX_TICKS_FROM_REF = 4096;
    
    struct PriceLevel {
        int32_t total_qty;
        int16_t order_count;
        int16_t head_idx;  // Into order pool
    };
    static_assert(sizeof(PriceLevel) == 8);
    
    // 8 price levels per cache line
    PriceLevel levels_[MAX_TICKS_FROM_REF * 2];  // Bid and ask sides
    int64_t ref_price_;  // Reference price (center of the array)
    int32_t tick_size_;   // Minimum tick in price units
    
public:
    PriceLevel& at(int64_t price) {
        int offset = (price - ref_price_) / tick_size_ + MAX_TICKS_FROM_REF;
        return levels_[offset];
    }
    
    // Top-of-book scan: just walk the array from the center outward
    // Sequential memory access — hardware prefetcher handles it
    int64_t best_bid() const {
        for (int i = MAX_TICKS_FROM_REF - 1; i >= 0; --i) {
            if (levels_[i].total_qty > 0) 
                return ref_price_ + (i - MAX_TICKS_FROM_REF) * tick_size_;
        }
        return 0;
    }
};
```

### 5.2 Flat Hash Map for Order Lookup

Orders need to be looked up by order ID (64-bit). Standard `std::unordered_map` has terrible cache behavior (linked list chains).

Use an open-addressing hash map with linear probing:

```cpp
template <typename V, size_t Capacity>
class FlatHashMap {
    static_assert((Capacity & (Capacity - 1)) == 0);
    
    struct Entry {
        uint64_t key;
        V value;
        bool occupied;
    };
    
    alignas(64) Entry entries_[Capacity] = {};
    static constexpr uint64_t MASK = Capacity - 1;
    
    static uint64_t hash(uint64_t key) {
        // Fibonacci hashing — excellent distribution, single multiply
        return key * 11400714819323198485ull;
    }
    
public:
    V* find(uint64_t key) {
        size_t idx = hash(key) >> (64 - __builtin_ctzll(Capacity));
        for (size_t i = 0; i < 16; ++i) {  // Max 16 probes
            auto& e = entries_[(idx + i) & MASK];
            if (e.occupied && e.key == key) return &e.value;
            if (!e.occupied) return nullptr;
        }
        return nullptr;
    }
    
    bool insert(uint64_t key, const V& value) {
        size_t idx = hash(key) >> (64 - __builtin_ctzll(Capacity));
        for (size_t i = 0; i < 16; ++i) {
            auto& e = entries_[(idx + i) & MASK];
            if (!e.occupied) {
                e = {key, value, true};
                return true;
            }
        }
        return false;  // Table full or too many collisions
    }
    
    bool erase(uint64_t key) {
        // Robin Hood deletion or tombstone marking
        // ...
    }
};
```

Linear probing gives sequential cache access — the probe sequence stays within 1-2 cache lines in the common case.

---

## 6. Instruction Cache Optimization

### 6.1 Hot/Cold Code Placement

```cpp
// The hot path must fit in L1-I (32 KB)
// Mark critical functions:
[[gnu::hot]] void on_market_data(const Packet& pkt);
[[gnu::hot]] void update_book(const Message& msg);
[[gnu::hot]] void evaluate_strategy(const BookUpdate& update);
[[gnu::hot]] void send_order(const OrderAction& action);

// Rare paths — keep out of L1-I:
[[gnu::cold]] void handle_gap(uint64_t expected_seq, uint64_t received_seq);
[[gnu::cold]] void handle_reject(const ExecReport& report);
[[gnu::cold]] void log_event(const char* fmt, ...);

// Linker script can group hot functions:
// -Wl,--section-ordering-file=hot_functions.txt
```

### 6.2 Measuring I-Cache Misses

```bash
# Count L1 instruction cache misses
perf stat -e L1-icache-load-misses,instructions,cycles ./trading_engine

# Detailed: which functions cause the most I-cache misses
perf record -e L1-icache-load-misses -- ./trading_engine
perf report
```

Target: < 0.1% L1-I miss rate on the hot path.

---

## 7. NUMA-Aware Allocation

### 7.1 The NUMA Penalty

On multi-socket systems, accessing memory attached to the remote NUMA node costs ~100 ns vs ~50 ns for local memory — a 2x penalty.

```bash
# Check NUMA topology
numactl --hardware

# Example output (2-socket system):
# node 0: CPUs 0-15,32-47    memory: 128 GB
# node 1: CPUs 16-31,48-63   memory: 128 GB
# distances:
#   node  0  1
#     0   10 21    ← 2.1x latency for cross-NUMA
#     1   21 10

# Find which NUMA node the NIC is on
cat /sys/class/net/ens1f0/device/numa_node
# Output: 0 → Pin all hot-path processes and memory to node 0
```

### 7.2 Allocating on Specific NUMA Nodes

```cpp
#include <numa.h>

// Allocate on a specific NUMA node
void* alloc_on_node(size_t size, int node) {
    void* ptr = numa_alloc_onnode(size, node);
    if (!ptr) return nullptr;
    
    // Touch all pages to force physical allocation on the correct node
    memset(ptr, 0, size);
    return ptr;
}

// Verify allocation placement
void verify_numa_placement(void* ptr, size_t size) {
    int pages = size / 4096;
    std::vector<void*> page_addrs(pages);
    std::vector<int> status(pages);
    
    for (int i = 0; i < pages; ++i) {
        page_addrs[i] = (char*)ptr + i * 4096;
    }
    
    move_pages(0, pages, page_addrs.data(), nullptr, status.data(), 0);
    
    for (int i = 0; i < pages; ++i) {
        if (status[i] != expected_node) {
            fprintf(stderr, "WARNING: Page %d on NUMA node %d, expected %d\n",
                    i, status[i], expected_node);
        }
    }
}
```

---

## 8. TLB Optimization with Hugepages

### 8.1 The TLB Miss Problem

The Translation Lookaside Buffer (TLB) caches virtual → physical page translations.

| Page Size | Typical TLB Entries | Coverage |
|---|---|---|
| 4 KB (default) | ~1536 (L1 DTLB: 64, L2 STLB: 1536) | 6 MB |
| 2 MB (hugepage) | ~32 (L1), ~1536 (L2) | ~3 GB |
| 1 GB (gigantic page) | ~4 | ~4 GB |

For an HFT process using 50 MB of hot data with 4 KB pages:
- 50 MB / 4 KB = 12,800 pages → TLB miss rate of ~90%
- Each TLB miss: page table walk = ~7-50 ns

With 2 MB hugepages:
- 50 MB / 2 MB = 25 pages → TLB miss rate ≈ 0%

### 8.2 Hugepage Configuration

```bash
# Reserve 2 MB hugepages at boot (kernel parameter)
hugepagesz=2M hugepages=1024

# Or at runtime
echo 1024 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages

# Mount hugetlbfs
mount -t hugetlbfs nodev /dev/hugepages

# DPDK hugepage reservation
dpdk-hugepages.py --setup 4G
```

```cpp
// Allocate memory on hugepages
#include <sys/mman.h>

void* alloc_hugepage(size_t size) {
    void* ptr = mmap(nullptr, size,
                     PROT_READ | PROT_WRITE,
                     MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB | MAP_POPULATE,
                     -1, 0);
    if (ptr == MAP_FAILED) return nullptr;
    
    // Lock pages in memory (prevent swap-out)
    mlock(ptr, size);
    return ptr;
}

// At process startup:
mlockall(MCL_CURRENT | MCL_FUTURE);
```

---

## 9. Benchmarking Cache Performance

### 9.1 Key perf Counters

```bash
# Cache miss counters
perf stat -e \
    cache-misses,cache-references,\
    L1-dcache-load-misses,L1-dcache-loads,\
    L1-icache-load-misses,\
    LLC-load-misses,LLC-loads,\
    dTLB-load-misses,dTLB-loads,\
    iTLB-load-misses \
    -- ./trading_engine --replay data.pcap

# Target metrics for hot path:
# L1-D miss rate: < 1%
# L1-I miss rate: < 0.1%
# LLC miss rate:  < 0.01%
# dTLB miss rate: < 0.001% (with hugepages)
```

### 9.2 Cache Line Bounce Detection

```bash
# Detect cache-to-cache transfers (false sharing, true sharing)
perf c2c record -- ./trading_engine --replay data.pcap
perf c2c report --call-graph --stdio

# Key metrics:
# HITM: Cache line was Modified on another core when we tried to load
# Remote HITM: Cross-socket cache-to-cache transfer (worst case)
# Local HITM: Same-socket core-to-core transfer
```

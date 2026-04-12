# Compiler Optimization & Memory Management for HFT

## Part 1: Compiler Optimization

---

### 1. Compiler Flags Deep Dive

```bash
# Production HFT build command
g++ -std=c++20 \
    -O3 \                       # Maximum optimization
    -march=native \             # Use all CPU features (AVX2, FMA, BMI2, etc.)
    -mtune=native \             # Tune instruction scheduling for this CPU
    -flto=auto \                # Link-Time Optimization with parallel workers
    -fno-exceptions \           # Remove exception handling overhead
    -fno-rtti \                 # Remove Run-Time Type Information
    -ffast-math \               # Allow FP reordering (VERIFY CORRECTNESS)
    -funroll-loops \            # Unroll loops for ILP
    -fno-omit-frame-pointer \   # Keep frame pointer (for perf profiling)
    -fno-plt \                  # Remove PLT indirection for extern calls
    -fvisibility=hidden \       # Hide symbols by default (smaller binary, faster linking)
    -DNDEBUG \                  # Remove assert() calls
    -pipe \                     # Use pipes instead of temp files (faster compile)
    -o trading_engine \
    src/*.cpp
```

### 1.1 Individual Flag Analysis

| Flag | Effect | Latency Impact | Risk |
|---|---|---|---|
| `-O3` | Inlining, vectorization, loop transforms | -20-40% vs `-O2` | May increase code size (I-cache pressure) |
| `-march=native` | Enables AVX2, FMA, BMI2, POPCNT, etc. | -5-15% | Binary not portable to different CPUs |
| `-flto` | Cross-module inlining and optimization | -5-15% | Longer link times |
| `-fno-exceptions` | Removes landing pads, unwind tables | -2-5% code size | Cannot use try/catch/throw |
| `-fno-rtti` | Removes `typeid`, `dynamic_cast` | -1-2% code size | Cannot use `dynamic_cast` |
| `-ffast-math` | FP reordering, reciprocal approx | -5-20% for FP code | **Dangerous**: may change numerical results |
| `-funroll-loops` | Unrolls loops with known trip count | -5-10% for tight loops | Code size increase |
| `-fno-plt` | Direct calls to shared lib functions | -1-2 ns per call | None |

### 1.2 Profile-Guided Optimization (PGO)

PGO uses runtime profiling data to make informed optimization decisions:

```bash
# Step 1: Instrumented build
g++ -O3 -march=native -fprofile-generate=./profile_data -o trading_engine src/*.cpp

# Step 2: Run with representative workload
./trading_engine --replay historical_data_2024_volatile.pcap
./trading_engine --replay historical_data_2024_normal.pcap
./trading_engine --replay historical_data_2024_opening_bell.pcap

# Step 3: Optimized build using profile
g++ -O3 -march=native -fprofile-use=./profile_data -o trading_engine src/*.cpp
```

**What PGO optimizes:**
- Branch prediction hints (likely/unlikely paths)
- Basic block reordering (hot code together)
- Function inlining decisions (inline hot callees)
- Loop unrolling decisions (based on actual trip counts)
- Register allocation (optimize for hot variables)

**Typical improvement:** 10-20% additional speedup over `-O3` alone.

### 1.3 BOLT (Binary Optimization and Layout Tool)

Post-link binary optimization (Facebook/Meta tool):

```bash
# Step 1: Build with relocation info
g++ -O3 -march=native -flto -Wl,--emit-relocs -o trading_engine src/*.cpp

# Step 2: Profile the binary
perf record -e cycles:u -j any,u -- ./trading_engine --replay data.pcap
perf2bolt -p perf.data -o perf.fdata ./trading_engine

# Step 3: Optimize binary layout
llvm-bolt ./trading_engine -o trading_engine.bolt \
    -data=perf.fdata \
    -reorder-blocks=ext-tsp \
    -reorder-functions=hfsort+ \
    -split-functions \
    -split-all-cold \
    -use-gnu-stack
```

**What BOLT does:**
- Reorders basic blocks to minimize branch mispredictions
- Reorders functions to improve I-cache locality
- Splits cold code into separate sections
- Additional 5-10% improvement on top of PGO

---

### 2. Inlining Control

```cpp
// Force inline — compiler MUST inline (used for trivial hot-path functions)
[[gnu::always_inline]] inline int64_t price_to_ticks(double price, double tick_size) {
    return static_cast<int64_t>(price / tick_size);
}

// Never inline — prevent cold code from being pulled into hot functions
[[gnu::noinline]] void handle_error(int code, const char* msg) {
    log_error(code, msg);
}

// Flatten — inline ALL calls within this function (even if they wouldn't normally be)
[[gnu::flatten]] void hot_path_handler(const Packet& pkt) {
    // Everything called from here is inlined
    auto msg = parse(pkt);
    auto update = apply_to_book(msg);
    evaluate(update);
}
```

### 3. Branch Prediction Hints

```cpp
// GCC/Clang built-in hints
#define LIKELY(x)   __builtin_expect(!!(x), 1)
#define UNLIKELY(x) __builtin_expect(!!(x), 0)

// C++20 standard attributes
if (order.qty > 0) [[likely]] {
    process_order(order);
} else [[unlikely]] {
    handle_invalid_order(order);
}

// Branchless alternatives (avoid branches entirely)

// Branchless absolute value
int32_t branchless_abs(int32_t x) {
    int32_t mask = x >> 31;
    return (x ^ mask) - mask;
}

// Branchless clamp
int32_t branchless_clamp(int32_t val, int32_t lo, int32_t hi) {
    val = val < lo ? lo : val;
    val = val > hi ? hi : val;
    return val;
}

// Conditional move (CMOV — always executes both paths, no branch)
double compute_bid(double fair_value, double spread, int32_t inventory) {
    double base = fair_value - spread / 2;
    double skewed = base - inventory * skew_factor;
    // Compiler generates CMOV for ternary with -O2+
    return inventory > max_inventory ? 0.0 : skewed;
}
```

---

## Part 2: Memory Management

---

### 4. Arena Allocator

For objects that are allocated together and freed together (e.g., all objects related to a single market data message):

```cpp
class Arena {
    static constexpr size_t BLOCK_SIZE = 2 * 1024 * 1024;  // 2 MB (hugepage-aligned)
    
    struct Block {
        char data[BLOCK_SIZE];
        size_t used{0};
        Block* next{nullptr};
    };
    
    Block* current_;
    Block* blocks_;  // Head of allocated block list
    
public:
    Arena() {
        blocks_ = allocate_block();
        current_ = blocks_;
    }
    
    void* alloc(size_t size, size_t align = 8) {
        size_t aligned_offset = (current_->used + align - 1) & ~(align - 1);
        
        if (aligned_offset + size > BLOCK_SIZE) {
            // Allocate new block
            Block* new_block = allocate_block();
            new_block->next = current_;
            current_ = new_block;
            aligned_offset = 0;
        }
        
        void* ptr = &current_->data[aligned_offset];
        current_->used = aligned_offset + size;
        return ptr;
    }
    
    template <typename T, typename... Args>
    T* create(Args&&... args) {
        void* mem = alloc(sizeof(T), alignof(T));
        return new (mem) T(std::forward<Args>(args)...);
    }
    
    // Reset all allocations (O(1) — does NOT call destructors)
    void reset() {
        Block* b = current_;
        while (b) {
            b->used = 0;
            if (!b->next) break;
            b = b->next;
        }
        current_ = blocks_;
    }
    
    ~Arena() {
        Block* b = blocks_;
        while (b) {
            Block* next = b->next;
            munmap(b, sizeof(Block));
            b = next;
        }
    }
    
private:
    Block* allocate_block() {
        void* mem = mmap(nullptr, sizeof(Block),
                        PROT_READ | PROT_WRITE,
                        MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB | MAP_POPULATE,
                        -1, 0);
        return new (mem) Block();
    }
};
```

### 5. Slab Allocator (Fixed-Size Pool)

For objects of uniform size (orders, price levels, etc.):

```cpp
template <typename T, size_t PoolSize = 65536>
class SlabAllocator {
    union Slot {
        T object;
        Slot* next_free;
        
        Slot() : next_free(nullptr) {}
        ~Slot() {}
    };
    
    alignas(64) Slot pool_[PoolSize];
    Slot* free_head_;
    size_t in_use_{0};
    
public:
    SlabAllocator() {
        // Build free list
        free_head_ = &pool_[0];
        for (size_t i = 0; i < PoolSize - 1; ++i) {
            pool_[i].next_free = &pool_[i + 1];
        }
        pool_[PoolSize - 1].next_free = nullptr;
    }
    
    T* allocate() {
        if (!free_head_) return nullptr;
        
        Slot* slot = free_head_;
        free_head_ = slot->next_free;
        ++in_use_;
        
        return &slot->object;
    }
    
    void deallocate(T* ptr) {
        Slot* slot = reinterpret_cast<Slot*>(ptr);
        slot->next_free = free_head_;
        free_head_ = slot;
        --in_use_;
    }
    
    size_t in_use() const { return in_use_; }
    size_t capacity() const { return PoolSize; }
    
    // Get index of an object (useful for external arrays indexed by object)
    size_t index_of(const T* ptr) const {
        const Slot* slot = reinterpret_cast<const Slot*>(ptr);
        return slot - pool_;
    }
};
```

### 6. Memory Locking & Page Fault Prevention

```cpp
#include <sys/mman.h>
#include <sys/resource.h>

void configure_memory() {
    // 1. Remove memory lock limit
    struct rlimit rl;
    rl.rlim_cur = RLIM_INFINITY;
    rl.rlim_max = RLIM_INFINITY;
    setrlimit(RLIMIT_MEMLOCK, &rl);
    
    // 2. Lock ALL current and future memory pages
    if (mlockall(MCL_CURRENT | MCL_FUTURE) != 0) {
        perror("mlockall failed");
        exit(1);
    }
    
    // 3. Disable core dumps (may cause latency spike on crash)
    rl.rlim_cur = 0;
    rl.rlim_max = 0;
    setrlimit(RLIMIT_CORE, &rl);
    
    // 4. Pre-fault stack pages
    volatile char stack_probe[8 * 1024 * 1024];  // 8 MB stack
    for (size_t i = 0; i < sizeof(stack_probe); i += 4096) {
        stack_probe[i] = 0;
    }
    
    // 5. Disable swap for this process
    // (mlockall already handles this, but belt-and-suspenders)
}
```

### 7. Fixed-Point Arithmetic (Avoid FP Allocation/Conversion)

```cpp
// Represent prices as int64_t in units of 10^-8
// Avoids floating-point entirely on the hot path
struct FixedPrice {
    int64_t raw;  // Value × 10^8
    
    static constexpr int64_t SCALE = 100'000'000LL;
    
    static FixedPrice from_double(double v) { return {static_cast<int64_t>(v * SCALE)}; }
    double to_double() const { return static_cast<double>(raw) / SCALE; }
    
    FixedPrice operator+(FixedPrice o) const { return {raw + o.raw}; }
    FixedPrice operator-(FixedPrice o) const { return {raw - o.raw}; }
    bool operator<(FixedPrice o) const { return raw < o.raw; }
    bool operator>(FixedPrice o) const { return raw > o.raw; }
    bool operator==(FixedPrice o) const { return raw == o.raw; }
    
    // Multiply by integer quantity — no FP
    int64_t notional(int32_t qty) const { return raw * qty; }
};
```

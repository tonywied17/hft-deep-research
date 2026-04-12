# Lock-Free Data Structures for HFT

## Overview

Lock-free data structures eliminate mutex contention, priority inversion, and unbounded blocking — all unacceptable in HFT. This document provides production-ready implementations and the theory behind them.

---

## 1. Memory Ordering on x86-64

### 1.1 The x86 TSO Memory Model

x86-64 has **Total Store Order (TSO)**:
- Loads are NOT reordered with other loads
- Stores are NOT reordered with other stores
- Loads are NOT reordered with older stores **to the same location**
- Stores CAN be reordered with older loads (the only reordering x86 allows)

**Practical impact:**
- `std::memory_order_acquire` (on loads) is FREE on x86 — no fence instruction emitted
- `std::memory_order_release` (on stores) is FREE on x86 — no fence instruction emitted
- `std::memory_order_seq_cst` on stores emits `MFENCE` or `LOCK XCHG` — expensive (~20-40 cycles)

### 1.2 C++ Memory Orderings Cheat Sheet

```
Ordering             x86 Cost    Guarantee
─────────────────────────────────────────────────────
relaxed              FREE        Atomicity only
acquire (on load)    FREE        No reads/writes move before this load
release (on store)   FREE        No reads/writes move after this store
acq_rel (on RMW)     FREE        Both acquire and release
seq_cst (on load)    FREE*       Global total order
seq_cst (on store)   MFENCE      Global total order
─────────────────────────────────────────────────────
* seq_cst loads are free on x86, but seq_cst stores are expensive
```

**HFT Rule:** Use `acquire`/`release` everywhere possible. Avoid `seq_cst` on stores.

### 1.3 Compiler vs Hardware Fences

```cpp
// Compiler fence only (prevents compiler reordering, no hardware fence)
std::atomic_signal_fence(std::memory_order_acq_rel);
// Or: asm volatile("" ::: "memory");

// Hardware fence (prevents both compiler and CPU reordering)
std::atomic_thread_fence(std::memory_order_acq_rel);  // Compiles to nothing on x86 TSO
std::atomic_thread_fence(std::memory_order_seq_cst);   // Compiles to MFENCE
```

---

## 2. SPSC Ring Buffer (Definitive Implementation)

### 2.1 Variable-Size Message SPSC Ring

For messages of varying size (critical for protocol bridging):

```cpp
class VariableSPSCRing {
    static constexpr size_t CAPACITY = 1 << 20;  // 1 MB
    static constexpr size_t MASK = CAPACITY - 1;
    
    alignas(64) std::atomic<size_t> write_pos_{0};
    alignas(64) std::atomic<size_t> read_pos_{0};
    alignas(64) char buffer_[CAPACITY];
    
    struct Header {
        uint32_t length;  // Payload length
        uint32_t padding; // Align to 8 bytes
    };
    
public:
    // Write a variable-length message
    bool try_write(const void* data, uint32_t length) {
        const size_t total = sizeof(Header) + ((length + 7) & ~7);  // 8-byte aligned
        const size_t wp = write_pos_.load(std::memory_order_relaxed);
        const size_t rp = read_pos_.load(std::memory_order_acquire);
        
        // Check available space (handle wrap-around)
        size_t available = (rp - wp - 1) & MASK;
        if (total > available) return false;
        
        // Write header
        Header hdr{length, 0};
        write_bytes(wp, &hdr, sizeof(Header));
        
        // Write payload
        write_bytes((wp + sizeof(Header)) & MASK, data, length);
        
        write_pos_.store((wp + total) & MASK, std::memory_order_release);
        return true;
    }
    
    // Read next message (returns pointer valid until next read)
    bool try_read(const void*& data, uint32_t& length) {
        const size_t rp = read_pos_.load(std::memory_order_relaxed);
        const size_t wp = write_pos_.load(std::memory_order_acquire);
        
        if (rp == wp) return false;
        
        Header hdr;
        read_bytes(rp, &hdr, sizeof(Header));
        length = hdr.length;
        
        // Return pointer to data in ring buffer
        data = &buffer_[(rp + sizeof(Header)) & MASK];
        
        const size_t total = sizeof(Header) + ((length + 7) & ~7);
        read_pos_.store((rp + total) & MASK, std::memory_order_release);
        return true;
    }
    
private:
    void write_bytes(size_t pos, const void* src, size_t n) {
        size_t first = std::min(n, CAPACITY - (pos & MASK));
        memcpy(&buffer_[pos & MASK], src, first);
        if (first < n) {
            memcpy(&buffer_[0], (const char*)src + first, n - first);
        }
    }
    
    void read_bytes(size_t pos, void* dst, size_t n) {
        size_t first = std::min(n, CAPACITY - (pos & MASK));
        memcpy(dst, &buffer_[pos & MASK], first);
        if (first < n) {
            memcpy((char*)dst + first, &buffer_[0], n - first);
        }
    }
};
```

### 2.2 SPSC with Batch/Peek Support

```cpp
template <typename T, size_t Cap>
class SPSCRingWithPeek {
    static_assert((Cap & (Cap - 1)) == 0);
    static constexpr size_t MASK = Cap - 1;
    
    alignas(64) std::atomic<size_t> write_pos_{0};
    size_t cached_read_{0};
    alignas(64) std::atomic<size_t> read_pos_{0};
    size_t cached_write_{0};
    alignas(64) T buffer_[Cap];
    
public:
    // Peek at the front without consuming
    const T* peek() noexcept {
        size_t rp = read_pos_.load(std::memory_order_relaxed);
        if (rp == cached_write_) {
            cached_write_ = write_pos_.load(std::memory_order_acquire);
            if (rp == cached_write_) return nullptr;
        }
        return &buffer_[rp & MASK];
    }
    
    // Consume the front element (call after peek)
    void pop() noexcept {
        size_t rp = read_pos_.load(std::memory_order_relaxed);
        read_pos_.store((rp + 1) & MASK, std::memory_order_release);
    }
    
    // Number of available items to read
    size_t available() noexcept {
        size_t wp = write_pos_.load(std::memory_order_acquire);
        size_t rp = read_pos_.load(std::memory_order_relaxed);
        return (wp - rp) & MASK;
    }
    
    // Contiguous read pointer (for batch processing without copy)
    // Returns pointer and count of contiguous elements available
    std::pair<const T*, size_t> read_ptr() noexcept {
        size_t rp = read_pos_.load(std::memory_order_relaxed);
        size_t wp = write_pos_.load(std::memory_order_acquire);
        size_t avail = (wp - rp) & MASK;
        // Contiguous count limited by wrap-around
        size_t contig = std::min(avail, Cap - (rp & MASK));
        return {&buffer_[rp & MASK], contig};
    }
    
    void advance_read(size_t count) noexcept {
        size_t rp = read_pos_.load(std::memory_order_relaxed);
        read_pos_.store((rp + count) & MASK, std::memory_order_release);
    }
};
```

---

## 3. MPSC (Multi-Producer Single-Consumer) Queue

For scenarios where multiple threads need to submit to a single consumer (e.g., multiple strategy threads submitting orders to a single gateway):

```cpp
template <typename T>
class MPSCQueue {
    struct Node {
        std::atomic<Node*> next{nullptr};
        T data;
    };
    
    // Stub node avoids empty queue edge case
    Node stub_;
    alignas(64) std::atomic<Node*> head_{&stub_};  // Producers push here
    alignas(64) Node* tail_{&stub_};                // Consumer pops from here
    
    // Free list to avoid allocation on hot path
    std::atomic<Node*> free_list_{nullptr};
    
public:
    // Multiple producers can call push concurrently
    void push(const T& item) {
        Node* node = alloc_node();
        node->data = item;
        node->next.store(nullptr, std::memory_order_relaxed);
        
        // Atomic swap: insert at head
        Node* prev = head_.exchange(node, std::memory_order_acq_rel);
        prev->next.store(node, std::memory_order_release);
    }
    
    // Single consumer
    bool try_pop(T& result) {
        Node* tail = tail_;
        Node* next = tail->next.load(std::memory_order_acquire);
        
        if (next == nullptr) return false;
        
        result = next->data;
        tail_ = next;
        free_node(tail);  // Return old tail to free list
        return true;
    }
    
private:
    Node* alloc_node() {
        // Try free list first
        Node* node = free_list_.load(std::memory_order_acquire);
        while (node) {
            if (free_list_.compare_exchange_weak(node, 
                node->next.load(std::memory_order_relaxed),
                std::memory_order_acq_rel)) {
                return node;
            }
        }
        return new Node();  // Fallback (should be rare after warmup)
    }
    
    void free_node(Node* node) {
        Node* head = free_list_.load(std::memory_order_relaxed);
        do {
            node->next.store(head, std::memory_order_relaxed);
        } while (!free_list_.compare_exchange_weak(head, node,
                    std::memory_order_release, std::memory_order_relaxed));
    }
};
```

---

## 4. Seqlock Variants

### 4.1 Multi-Field Seqlock with Version Check

```cpp
template <typename T>
class VersionedSeqlock {
    static_assert(std::is_trivially_copyable_v<T>);
    
    struct alignas(64) {
        std::atomic<uint64_t> version{0};
        T data{};
        uint64_t checksum;  // Optional: integrity check
    } state_;
    
public:
    // Writer (must be single-threaded)
    void update(const T& value) {
        uint64_t v = state_.version.load(std::memory_order_relaxed);
        
        // Odd version = write in progress
        state_.version.store(v + 1, std::memory_order_release);
        std::atomic_signal_fence(std::memory_order_acq_rel);
        
        state_.data = value;
        state_.checksum = compute_checksum(value);
        
        std::atomic_signal_fence(std::memory_order_acq_rel);
        state_.version.store(v + 2, std::memory_order_release);
    }
    
    // Reader (lock-free, multiple threads)
    std::optional<T> try_read() const {
        for (int attempt = 0; attempt < 4; ++attempt) {
            uint64_t v1 = state_.version.load(std::memory_order_acquire);
            if (v1 & 1) continue;  // Write in progress
            
            T copy = state_.data;
            uint64_t cs = state_.checksum;
            
            std::atomic_thread_fence(std::memory_order_acquire);
            uint64_t v2 = state_.version.load(std::memory_order_relaxed);
            
            if (v1 == v2 && cs == compute_checksum(copy)) {
                return copy;
            }
        }
        return std::nullopt;  // Could not get consistent read
    }
    
    uint64_t version() const {
        return state_.version.load(std::memory_order_acquire) >> 1;
    }
    
private:
    static uint64_t compute_checksum(const T& value) {
        // Simple XOR-based checksum for integrity
        const uint64_t* p = reinterpret_cast<const uint64_t*>(&value);
        uint64_t cs = 0;
        for (size_t i = 0; i < sizeof(T) / 8; ++i) {
            cs ^= p[i];
        }
        return cs;
    }
};
```

### 4.2 Double-Buffer (Wait-Free Read)

For data larger than a cache line where seqlocks may retry too often:

```cpp
template <typename T>
class DoubleBuffer {
    T buffers_[2];
    alignas(64) std::atomic<uint32_t> active_{0};  // Index of the "live" buffer
    
public:
    // Writer: write to inactive buffer, then swap
    void update(const T& value) {
        uint32_t inactive = 1 - active_.load(std::memory_order_relaxed);
        buffers_[inactive] = value;
        std::atomic_thread_fence(std::memory_order_release);
        active_.store(inactive, std::memory_order_release);
    }
    
    // Reader: always reads the active buffer (no retry, no waiting)
    const T& read() const {
        uint32_t idx = active_.load(std::memory_order_acquire);
        return buffers_[idx];
    }
};
```

**Caveat:** The reader may see stale data if the writer updates during the read. For HFT, this is usually acceptable — the next read (microseconds later) will get fresh data.

---

## 5. Lock-Free Object Pool

Pre-allocate objects at startup, hand them out without allocation:

```cpp
template <typename T, size_t N>
class LockFreePool {
    struct alignas(64) Slot {
        T object;
        std::atomic<Slot*> next;
    };
    
    Slot storage_[N];
    alignas(64) std::atomic<Slot*> free_head_;
    
public:
    LockFreePool() {
        // Build free list
        free_head_.store(&storage_[0], std::memory_order_relaxed);
        for (size_t i = 0; i < N - 1; ++i) {
            storage_[i].next.store(&storage_[i + 1], std::memory_order_relaxed);
        }
        storage_[N - 1].next.store(nullptr, std::memory_order_relaxed);
    }
    
    T* allocate() {
        Slot* head = free_head_.load(std::memory_order_acquire);
        while (head) {
            Slot* next = head->next.load(std::memory_order_relaxed);
            if (free_head_.compare_exchange_weak(head, next,
                    std::memory_order_acq_rel, std::memory_order_acquire)) {
                return &head->object;
            }
        }
        return nullptr;  // Pool exhausted
    }
    
    void deallocate(T* ptr) {
        Slot* slot = reinterpret_cast<Slot*>(
            reinterpret_cast<char*>(ptr) - offsetof(Slot, object));
        
        Slot* head = free_head_.load(std::memory_order_relaxed);
        do {
            slot->next.store(head, std::memory_order_relaxed);
        } while (!free_head_.compare_exchange_weak(head, slot,
                    std::memory_order_release, std::memory_order_relaxed));
    }
};
```

---

## 6. Hazard Pointers (Safe Memory Reclamation)

When using lock-free structures that involve pointer dereferencing (stacks, queues, linked lists), you need safe reclamation to prevent use-after-free:

```cpp
class HazardPointerDomain {
    static constexpr int MAX_THREADS = 64;
    static constexpr int MAX_RETIRED = 256;
    
    struct ThreadState {
        std::atomic<void*> hazard{nullptr};  // Currently protected pointer
        void* retired[MAX_RETIRED];          // Pointers awaiting deletion
        int retired_count{0};
    };
    
    alignas(64) ThreadState threads_[MAX_THREADS];
    
    static thread_local int thread_id_;
    
public:
    // Protect a pointer before dereferencing
    template <typename T>
    T* protect(std::atomic<T*>& source) {
        T* ptr;
        do {
            ptr = source.load(std::memory_order_acquire);
            threads_[thread_id_].hazard.store(ptr, std::memory_order_release);
            // Re-read to ensure protection was established before the pointer changed
        } while (ptr != source.load(std::memory_order_acquire));
        return ptr;
    }
    
    void clear_hazard() {
        threads_[thread_id_].hazard.store(nullptr, std::memory_order_release);
    }
    
    // Retire a pointer for later deletion
    void retire(void* ptr) {
        auto& ts = threads_[thread_id_];
        ts.retired[ts.retired_count++] = ptr;
        
        if (ts.retired_count >= MAX_RETIRED / 2) {
            scan_and_reclaim();
        }
    }
    
private:
    void scan_and_reclaim() {
        auto& ts = threads_[thread_id_];
        
        // Collect all hazard pointers
        std::unordered_set<void*> protected_ptrs;
        for (int i = 0; i < MAX_THREADS; ++i) {
            void* hp = threads_[i].hazard.load(std::memory_order_acquire);
            if (hp) protected_ptrs.insert(hp);
        }
        
        // Delete anything not protected
        int remaining = 0;
        for (int i = 0; i < ts.retired_count; ++i) {
            if (protected_ptrs.count(ts.retired[i])) {
                ts.retired[remaining++] = ts.retired[i];
            } else {
                free(ts.retired[i]);
            }
        }
        ts.retired_count = remaining;
    }
};
```

---

## 7. Epoch-Based Reclamation (Simpler Alternative)

```cpp
class EpochBasedReclamation {
    static constexpr int NUM_EPOCHS = 3;
    
    std::atomic<uint64_t> global_epoch_{0};
    
    struct ThreadState {
        std::atomic<uint64_t> local_epoch{0};
        std::atomic<bool> active{false};
        std::vector<void*> retire_lists[NUM_EPOCHS];
    };
    
    std::array<ThreadState, 64> threads_;
    
public:
    // Enter critical section
    void enter(int tid) {
        threads_[tid].active.store(true, std::memory_order_release);
        threads_[tid].local_epoch.store(
            global_epoch_.load(std::memory_order_acquire),
            std::memory_order_release);
    }
    
    // Exit critical section  
    void exit(int tid) {
        threads_[tid].active.store(false, std::memory_order_release);
    }
    
    // Retire a pointer
    void retire(int tid, void* ptr) {
        uint64_t epoch = global_epoch_.load(std::memory_order_relaxed);
        threads_[tid].retire_lists[epoch % NUM_EPOCHS].push_back(ptr);
        try_advance_epoch();
    }
    
private:
    void try_advance_epoch() {
        uint64_t current = global_epoch_.load(std::memory_order_acquire);
        
        // Can advance if all active threads are in the current epoch
        for (auto& ts : threads_) {
            if (ts.active.load(std::memory_order_acquire) &&
                ts.local_epoch.load(std::memory_order_acquire) != current) {
                return;  // Someone is behind
            }
        }
        
        // Advance epoch
        if (global_epoch_.compare_exchange_strong(current, current + 1)) {
            // Reclaim from epoch (current - 2)
            uint64_t old_epoch = (current - 1) % NUM_EPOCHS;
            for (auto& ts : threads_) {
                for (void* ptr : ts.retire_lists[old_epoch]) {
                    free(ptr);
                }
                ts.retire_lists[old_epoch].clear();
            }
        }
    }
};
```

---

## 8. Testing Lock-Free Structures

### 8.1 Stress Testing

```cpp
// SPSC ring stress test
void stress_test_spsc() {
    SPSCRing<uint64_t, 65536> ring;
    std::atomic<bool> done{false};
    std::atomic<uint64_t> total_produced{0};
    std::atomic<uint64_t> total_consumed{0};
    
    // Producer thread
    std::thread producer([&] {
        pin_to_core(2);
        for (uint64_t i = 0; i < 100'000'000; ++i) {
            while (!ring.try_push(i)) {}  // Spin until space available
            total_produced.fetch_add(1, std::memory_order_relaxed);
        }
        done.store(true, std::memory_order_release);
    });
    
    // Consumer thread
    std::thread consumer([&] {
        pin_to_core(3);
        uint64_t expected = 0;
        uint64_t value;
        while (!done.load(std::memory_order_acquire) || !ring.empty()) {
            if (ring.try_pop(value)) {
                assert(value == expected++);  // Verify ordering
                total_consumed.fetch_add(1, std::memory_order_relaxed);
            }
        }
    });
    
    producer.join();
    consumer.join();
    
    assert(total_produced.load() == total_consumed.load());
}
```

### 8.2 Using ThreadSanitizer

```bash
# Compile with TSan
g++ -O2 -g -fsanitize=thread -o test_lockfree test_lockfree.cpp -lpthread

# Run — TSan will detect data races, use-after-free, lock order violations
./test_lockfree
```

**Note:** TSan may report false positives on valid lock-free code due to its imprecise understanding of custom memory orderings. Use `__tsan_acquire()` / `__tsan_release()` annotations if needed.

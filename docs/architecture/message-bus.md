# Inter-Component Communication & Message Bus Patterns

## Overview

HFT systems require sub-100 ns communication between components. Traditional message buses (ZeroMQ, RabbitMQ, Kafka) add microseconds to milliseconds of overhead. This document covers the lock-free, zero-copy communication patterns used in production HFT.

---

## 1. SPSC Ring Buffer — The Foundation

The Single-Producer Single-Consumer ring buffer is the primary IPC primitive. It provides:
- **Zero contention** (exactly one reader, one writer)
- **No locks** (purely atomic operations with acquire/release semantics)
- **Zero syscalls** (pure user-space)
- **Cache-friendly** (sequential memory access, producer and consumer metadata on separate cache lines)

### 1.1 Production Implementation

```cpp
#include <atomic>
#include <cstddef>
#include <cstring>
#include <new>

template <typename T, size_t Capacity>
class SPSCRingBuffer {
    static_assert((Capacity & (Capacity - 1)) == 0, "Capacity must be power of 2");
    static_assert(std::is_trivially_copyable_v<T>, "T must be trivially copyable");
    
    static constexpr size_t MASK = Capacity - 1;
    
    // Producer cache line
    alignas(64) std::atomic<size_t> write_pos_{0};
    size_t cached_read_pos_{0};  // Producer's local cache of read_pos
    
    // Consumer cache line
    alignas(64) std::atomic<size_t> read_pos_{0};
    size_t cached_write_pos_{0};  // Consumer's local cache of write_pos
    
    // Data buffer (aligned to cache line boundary, on hugepage if possible)
    alignas(64) T buffer_[Capacity];
    
public:
    // Producer side
    bool try_push(const T& item) noexcept {
        const size_t wp = write_pos_.load(std::memory_order_relaxed);
        const size_t next = (wp + 1) & MASK;
        
        // Check if full — use cached read position first
        if (next == cached_read_pos_) {
            // Refresh cache from atomic
            cached_read_pos_ = read_pos_.load(std::memory_order_acquire);
            if (next == cached_read_pos_) {
                return false;  // Ring is full
            }
        }
        
        buffer_[wp] = item;
        write_pos_.store(next, std::memory_order_release);
        return true;
    }
    
    // Consumer side
    bool try_pop(T& item) noexcept {
        const size_t rp = read_pos_.load(std::memory_order_relaxed);
        
        // Check if empty — use cached write position first
        if (rp == cached_write_pos_) {
            // Refresh cache from atomic
            cached_write_pos_ = write_pos_.load(std::memory_order_acquire);
            if (rp == cached_write_pos_) {
                return false;  // Ring is empty
            }
        }
        
        item = buffer_[rp];
        read_pos_.store((rp + 1) & MASK, std::memory_order_release);
        return true;
    }
    
    // Batch operations for throughput
    size_t try_push_batch(const T* items, size_t count) noexcept {
        const size_t wp = write_pos_.load(std::memory_order_relaxed);
        cached_read_pos_ = read_pos_.load(std::memory_order_acquire);
        
        const size_t available = (cached_read_pos_ - wp - 1) & MASK;
        const size_t to_write = std::min(count, available);
        
        for (size_t i = 0; i < to_write; ++i) {
            buffer_[(wp + i) & MASK] = items[i];
        }
        
        write_pos_.store((wp + to_write) & MASK, std::memory_order_release);
        return to_write;
    }
    
    size_t try_pop_batch(T* items, size_t max_count) noexcept {
        const size_t rp = read_pos_.load(std::memory_order_relaxed);
        cached_write_pos_ = write_pos_.load(std::memory_order_acquire);
        
        const size_t available = (cached_write_pos_ - rp) & MASK;
        const size_t to_read = std::min(max_count, available);
        
        for (size_t i = 0; i < to_read; ++i) {
            items[i] = buffer_[(rp + i) & MASK];
        }
        
        read_pos_.store((rp + to_read) & MASK, std::memory_order_release);
        return to_read;
    }
    
    size_t size() const noexcept {
        return (write_pos_.load(std::memory_order_acquire) - 
                read_pos_.load(std::memory_order_acquire)) & MASK;
    }
    
    bool empty() const noexcept { return size() == 0; }
    static constexpr size_t capacity() noexcept { return Capacity; }
};
```

### 1.2 Cache Optimization Tricks

**Cached position variables:** The `cached_read_pos_` / `cached_write_pos_` pattern avoids touching the other thread's atomic in the common case. On x86, loading a remote core's atomic involves a cache-line transfer (~30-50 ns). By caching the last known position, we only pay this cost when the ring appears full/empty.

**Benchmark impact:**
- Without caching: ~40 ns per push/pop (always reads remote atomic)
- With caching: ~12 ns per push/pop (reads local cached value 95%+ of the time)

---

## 2. Seqlock-Based Shared State Publishing

For data that is written infrequently by one thread and read by many (e.g., BBO, position snapshot, telemetry), a seqlock provides zero-overhead for the writer and wait-free reads.

```cpp
template <typename T>
class SeqlockPublisher {
    static_assert(std::is_trivially_copyable_v<T>);
    
    alignas(64) std::atomic<uint64_t> version_{0};
    alignas(64) T data_{};
    
public:
    // Single writer — must be called from one thread only
    void publish(const T& value) noexcept {
        version_.store(version_.load(std::memory_order_relaxed) + 1, 
                       std::memory_order_release);
        
        // Compiler barrier to prevent reordering of data writes
        std::atomic_signal_fence(std::memory_order_acq_rel);
        
        data_ = value;
        
        std::atomic_signal_fence(std::memory_order_acq_rel);
        
        version_.store(version_.load(std::memory_order_relaxed) + 1, 
                       std::memory_order_release);
    }
    
    // Multiple readers — lock-free, restarts on torn read
    T read() const noexcept {
        T result;
        uint64_t v1, v2;
        do {
            v1 = version_.load(std::memory_order_acquire);
            result = data_;
            std::atomic_thread_fence(std::memory_order_acquire);
            v2 = version_.load(std::memory_order_relaxed);
        } while (v1 != v2 || (v1 & 1));
        return result;
    }
};

// Usage:
struct BBOSnapshot {
    int64_t bid_price;
    int64_t ask_price;
    int32_t bid_qty;
    int32_t ask_qty;
    uint64_t timestamp_tsc;
};

SeqlockPublisher<BBOSnapshot> bbo_publisher;

// Strategy thread publishes:
bbo_publisher.publish({bid, ask, bid_qty, ask_qty, __rdtsc()});

// Monitoring thread reads:
BBOSnapshot snapshot = bbo_publisher.read();
```

---

## 3. Shared Memory IPC (Cross-Process)

For communication between independent processes (e.g., risk monitor, strategy engine, logger):

### 3.1 Memory-Mapped SPSC Ring

```cpp
#include <sys/mman.h>
#include <fcntl.h>

struct SharedRingHeader {
    alignas(64) std::atomic<size_t> write_pos;
    alignas(64) std::atomic<size_t> read_pos;
    size_t capacity;
    size_t element_size;
};

class SharedMemoryRing {
    int fd_;
    void* mapped_;
    SharedRingHeader* header_;
    char* data_;
    
public:
    static SharedMemoryRing create(const char* name, size_t capacity, size_t elem_size) {
        SharedMemoryRing ring;
        
        size_t total = sizeof(SharedRingHeader) + capacity * elem_size;
        
        // Use /dev/hugepages for hugepage-backed shared memory
        ring.fd_ = open("/dev/hugepages/hft_ring", O_CREAT | O_RDWR, 0600);
        ftruncate(ring.fd_, total);
        
        ring.mapped_ = mmap(nullptr, total, PROT_READ | PROT_WRITE,
                           MAP_SHARED | MAP_POPULATE | MAP_LOCKED, ring.fd_, 0);
        
        ring.header_ = static_cast<SharedRingHeader*>(ring.mapped_);
        ring.header_->capacity = capacity;
        ring.header_->element_size = elem_size;
        ring.header_->write_pos.store(0);
        ring.header_->read_pos.store(0);
        
        ring.data_ = reinterpret_cast<char*>(ring.header_ + 1);
        
        return ring;
    }
    
    static SharedMemoryRing open(const char* name) {
        SharedMemoryRing ring;
        // Similar to create but without O_CREAT and initialization
        ring.fd_ = ::open("/dev/hugepages/hft_ring", O_RDWR);
        // ... mmap same way
        return ring;
    }
};
```

### 3.2 Latency Comparison

| IPC Method | Typical Latency | Use Case |
|---|---|---|
| SPSC ring (same process) | 10-30 ns | Pipeline stages |
| SPSC ring (shared memory, same NUMA) | 30-80 ns | Cross-process hot path |
| SPSC ring (shared memory, cross-NUMA) | 100-200 ns | Avoid at all costs |
| Unix domain socket | 2-5 μs | Management plane |
| TCP loopback | 5-10 μs | Management plane |
| Kernel pipe | 3-8 μs | Not used in HFT |
| ZeroMQ (IPC) | 10-30 μs | Not used on hot path |

---

## 4. Message Encoding

### 4.1 Flat Binary Messages (Preferred)

```cpp
// Fixed-size, POD message — no serialization cost
struct alignas(8) BookUpdateMsg {
    uint8_t  type;           // Message type discriminator
    uint8_t  side;           // 0=bid, 1=ask
    uint16_t instrument_id;  // Direct index into instrument array
    int32_t  qty_delta;      // Quantity change
    int64_t  price;          // Price in fixed-point (10^-8)
    uint64_t order_id;       // Exchange order reference
    uint64_t timestamp_tsc;  // TSC at receive time
};
static_assert(sizeof(BookUpdateMsg) == 32);  // Fits in half a cache line

struct alignas(8) OrderActionMsg {
    uint8_t  type;           // NEW, CANCEL, REPLACE
    uint8_t  side;           
    uint16_t instrument_id;
    int32_t  quantity;
    int64_t  price;
    uint64_t client_order_id;
    uint64_t timestamp_tsc;
};
static_assert(sizeof(OrderActionMsg) == 32);
```

**Why 32 bytes?** Two messages fit in one 64-byte cache line. When the consumer reads one message and immediately reads the next, the second is already in L1.

### 4.2 Tagged Union for Polymorphic Messages

```cpp
enum class MsgType : uint8_t {
    BOOK_UPDATE = 1,
    ORDER_ACTION = 2,
    EXECUTION_REPORT = 3,
    RISK_ALERT = 4,
    HEARTBEAT = 5,
};

struct Message {
    MsgType type;
    uint8_t padding[7];
    
    union {
        BookUpdateMsg book_update;
        OrderActionMsg order_action;
        ExecutionReportMsg exec_report;
        RiskAlertMsg risk_alert;
    };
};
static_assert(sizeof(Message) <= 64);  // One cache line

// Processing with switch (compiler generates jump table)
void process(const Message& msg) {
    switch (msg.type) {
        case MsgType::BOOK_UPDATE:    handle_book_update(msg.book_update); break;
        case MsgType::ORDER_ACTION:   handle_order_action(msg.order_action); break;
        case MsgType::EXECUTION_REPORT: handle_exec(msg.exec_report); break;
        default: break;
    }
}
```

---

## 5. Fan-Out Patterns

### 5.1 One-to-Many via Replicated SPSC Rings

```
Producer ──┬──→ SPSC Ring A ──→ Consumer A
           ├──→ SPSC Ring B ──→ Consumer B
           └──→ SPSC Ring C ──→ Consumer C
```

The producer writes to ALL rings. Simple, no contention, but producer does N writes per message.

```cpp
template <size_t N>
class FanOut {
    SPSCRingBuffer<Message, 4096>* rings_[N];
    
public:
    void publish(const Message& msg) {
        for (size_t i = 0; i < N; ++i) {
            rings_[i]->try_push(msg);  // Fire-and-forget; consumers must keep up
        }
    }
};
```

### 5.2 Shared Broadcast Buffer

A single shared memory region with a seqlock or sequenced broadcast:

```cpp
class BroadcastBuffer {
    struct Slot {
        alignas(64) std::atomic<uint64_t> sequence;
        Message data;
    };
    
    alignas(64) Slot buffer_[BUFFER_SIZE];
    alignas(64) std::atomic<uint64_t> write_cursor_{0};
    
public:
    // Producer
    void publish(const Message& msg) {
        uint64_t seq = write_cursor_.fetch_add(1, std::memory_order_relaxed);
        auto& slot = buffer_[seq & (BUFFER_SIZE - 1)];
        slot.data = msg;
        slot.sequence.store(seq + 1, std::memory_order_release);
    }
    
    // Consumer (each consumer maintains its own read_cursor)
    bool try_read(uint64_t& read_cursor, Message& out) {
        auto& slot = buffer_[read_cursor & (BUFFER_SIZE - 1)];
        uint64_t seq = slot.sequence.load(std::memory_order_acquire);
        if (seq == read_cursor + 1) {
            out = slot.data;
            ++read_cursor;
            return true;
        }
        if (seq > read_cursor + 1) {
            // Consumer fell behind — data was overwritten
            read_cursor = write_cursor_.load(std::memory_order_acquire) - BUFFER_SIZE + 1;
            return false;  // Signal gap to caller
        }
        return false;  // No new data
    }
};
```

---

## 6. Backpressure & Flow Control

### 6.1 The Problem

If the consumer can't keep up with the producer, the SPSC ring fills up. Options:

1. **Drop messages** (acceptable for telemetry, unacceptable for market data)
2. **Block the producer** (never acceptable on hot path — stalls the entire pipeline)
3. **Overwrite oldest** (ring as circular buffer — consumer must detect gaps)
4. **Adaptive backpressure** (producer slows itself when ring is N% full)

### 6.2 Production Strategy

The hot-path SPSC rings should NEVER fill up — this indicates a design problem. Size them large enough for worst-case burst (e.g., market open on high-volatility day):

```
Peak message rate: 2M msg/sec
Consumer processing time: 500 ns/msg = 2M msg/sec throughput
Safety factor: 4x burst
Ring capacity: 2M × 4 / 1000 = 8,192 slots per millisecond of burst

Choose: 2^14 = 16,384 slots = sufficient for ~8 ms of burst at peak rate
```

If rings fill despite sizing, instrument the fill level and alert:

```cpp
void push_with_monitoring(const Message& msg) {
    if (UNLIKELY(!ring_.try_push(msg))) {
        ++stats_.ring_full_count;
        // ALERT: This should never happen. Ring is full.
        // Options: log and drop, or spin-wait (adds latency to producer)
    }
    
    size_t fill = ring_.size();
    if (UNLIKELY(fill > HIGH_WATERMARK)) {
        ++stats_.high_watermark_count;
    }
}
```

---

## 7. Command Channel (Slow Path → Hot Path)

The management plane needs to send commands to the hot path (parameter updates, enable/disable strategy, kill switch). This uses a separate SPSC ring in the opposite direction:

```cpp
enum class CommandType : uint8_t {
    UPDATE_PARAM,           // Change strategy parameter
    ENABLE_INSTRUMENT,      // Enable trading for instrument 
    DISABLE_INSTRUMENT,     // Disable trading for instrument
    KILL_SWITCH,           // Emergency stop all trading
    UPDATE_RISK_LIMIT,     // Change risk limit
};

struct Command {
    CommandType type;
    uint16_t instrument_id;
    double value;           // Parameter value (interpreted by type)
    uint64_t sequence;      // For ack tracking
};

// Management thread → Hot path
SPSCRingBuffer<Command, 256> command_ring;

// In the hot loop (checked every N iterations):
Command cmd;
while (command_ring.try_pop(cmd)) {
    switch (cmd.type) {
        case CommandType::KILL_SWITCH:
            kill_all_strategies();
            cancel_all_orders();
            break;
        case CommandType::UPDATE_PARAM:
            strategies[cmd.instrument_id].update_param(cmd.value);
            break;
        // ...
    }
}
```

This ring is checked in the maintenance phase of the event loop — not on every packet. The kill switch command must be checked more frequently (every iteration) for safety.

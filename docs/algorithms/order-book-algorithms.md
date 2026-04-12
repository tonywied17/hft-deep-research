# Order Book Algorithms & Data Structures

## Overview

The order book is the most latency-critical data structure in an HFT system. Every market data tick touches it. This document covers efficient representations, update algorithms, and reconstruction from exchange protocols.

---

## 1. Order Book Representations

### 1.1 Price-Level Book (L2)

Aggregated quantities at each price. Used by most strategies.

```cpp
struct PriceLevel {
    int64_t price;        // Fixed-point price (ticks × price_increment)
    int32_t total_qty;    // Aggregate quantity
    int16_t order_count;  // Number of orders
    int16_t padding;
};
static_assert(sizeof(PriceLevel) == 16);  // 4 levels per cache line

class L2Book {
    static constexpr int MAX_LEVELS = 20;
    
    PriceLevel bids[MAX_LEVELS];
    PriceLevel asks[MAX_LEVELS];
    int bid_depth = 0;
    int ask_depth = 0;
    
public:
    // O(1) best bid/ask
    const PriceLevel& best_bid() const { return bids[0]; }
    const PriceLevel& best_ask() const { return asks[0]; }
    
    int64_t mid_price() const {
        return (bids[0].price + asks[0].price) / 2;
    }
    
    double spread() const {
        return asks[0].price - bids[0].price;
    }
    
    // Update a price level (insert, modify, or remove)
    void update_level(int side, int64_t price, int32_t qty, int16_t count) {
        PriceLevel* levels = (side == 0) ? bids : asks;
        int& depth = (side == 0) ? bid_depth : ask_depth;
        
        // Find the level
        int idx = find_level(levels, depth, price, side);
        
        if (qty == 0) {
            // Remove level — shift remaining levels up
            memmove(&levels[idx], &levels[idx + 1], 
                    (depth - idx - 1) * sizeof(PriceLevel));
            --depth;
        } else if (idx < depth && levels[idx].price == price) {
            // Update existing level
            levels[idx].total_qty = qty;
            levels[idx].order_count = count;
        } else {
            // Insert new level — shift levels down
            if (depth < MAX_LEVELS) {
                memmove(&levels[idx + 1], &levels[idx],
                        (depth - idx) * sizeof(PriceLevel));
                levels[idx] = {price, qty, count, 0};
                ++depth;
            }
        }
    }
    
private:
    // Binary search for price level
    int find_level(const PriceLevel* levels, int depth, int64_t price, int side) {
        int lo = 0, hi = depth;
        while (lo < hi) {
            int mid = (lo + hi) / 2;
            if (side == 0) {  // Bids: descending order
                if (levels[mid].price > price) lo = mid + 1;
                else hi = mid;
            } else {  // Asks: ascending order
                if (levels[mid].price < price) lo = mid + 1;
                else hi = mid;
            }
        }
        return lo;
    }
};
```

### 1.2 Order-Level Book (L3)

Tracks individual orders. Required for queue position estimation and ITCH book reconstruction.

```cpp
struct Order {
    uint64_t order_id;
    int64_t price;
    int32_t remaining_qty;
    int32_t original_qty;
    uint16_t instrument_id;
    uint8_t side;           // 0=bid, 1=ask
    uint8_t padding;
    // Linked list for orders at same price
    int32_t next_order_idx;  // Index into order pool (-1 = end)
    int32_t prev_order_idx;
};
static_assert(sizeof(Order) == 40);

struct L3PriceLevel {
    int64_t price;
    int32_t total_qty;
    int32_t order_count;
    int32_t head_order_idx;  // First order at this level
    int32_t tail_order_idx;  // Last order (for O(1) append)
};

class L3Book {
    // Order pool (pre-allocated, no dynamic allocation)
    SlabAllocator<Order, 1'000'000> order_pool;
    
    // Order ID → order index lookup (flat hash map)
    FlatHashMap<int32_t, 1'048'576> order_map;  // Key: order_id, Value: pool index
    
    // Price levels (direct-indexed by price offset)
    static constexpr int PRICE_RANGE = 8192;  // ±4096 ticks from reference
    L3PriceLevel bid_levels[PRICE_RANGE];
    L3PriceLevel ask_levels[PRICE_RANGE];
    int64_t ref_price;
    int64_t tick_size;
    
public:
    void add_order(uint64_t order_id, int side, int64_t price, int32_t qty) {
        // Allocate from pool
        Order* order = order_pool.allocate();
        int32_t idx = order_pool.index_of(order);
        
        *order = {order_id, price, qty, qty, 0, 
                  static_cast<uint8_t>(side), 0, -1, -1};
        
        // Register in lookup map
        order_map.insert(order_id, idx);
        
        // Add to price level's linked list
        auto& level = get_level(side, price);
        if (level.tail_order_idx >= 0) {
            auto& tail = order_pool[level.tail_order_idx];
            tail.next_order_idx = idx;
            order->prev_order_idx = level.tail_order_idx;
        } else {
            level.head_order_idx = idx;
        }
        level.tail_order_idx = idx;
        level.total_qty += qty;
        level.order_count++;
    }
    
    void cancel_order(uint64_t order_id, int32_t cancel_qty) {
        auto* idx_ptr = order_map.find(order_id);
        if (!idx_ptr) return;
        
        auto& order = order_pool[*idx_ptr];
        auto& level = get_level(order.side, order.price);
        
        int32_t actual_cancel = std::min(cancel_qty, order.remaining_qty);
        order.remaining_qty -= actual_cancel;
        level.total_qty -= actual_cancel;
        
        if (order.remaining_qty == 0) {
            remove_from_level(order, level, *idx_ptr);
            order_map.erase(order_id);
            order_pool.deallocate(&order);
        }
    }
    
    void execute_order(uint64_t order_id, int32_t exec_qty) {
        cancel_order(order_id, exec_qty);  // Same logic
    }
    
    // Queue position estimation for our own order
    int32_t queue_ahead(uint64_t our_order_id) const {
        auto* idx_ptr = order_map.find(our_order_id);
        if (!idx_ptr) return -1;
        
        const auto& our_order = order_pool[*idx_ptr];
        const auto& level = get_level(our_order.side, our_order.price);
        
        int32_t qty_ahead = 0;
        int32_t cursor = level.head_order_idx;
        
        while (cursor >= 0 && cursor != *idx_ptr) {
            qty_ahead += order_pool[cursor].remaining_qty;
            cursor = order_pool[cursor].next_order_idx;
        }
        
        return qty_ahead;
    }
    
private:
    L3PriceLevel& get_level(int side, int64_t price) {
        int offset = (price - ref_price) / tick_size + PRICE_RANGE / 2;
        return (side == 0) ? bid_levels[offset] : ask_levels[offset];
    }
    
    void remove_from_level(Order& order, L3PriceLevel& level, int32_t idx) {
        // Update linked list
        if (order.prev_order_idx >= 0)
            order_pool[order.prev_order_idx].next_order_idx = order.next_order_idx;
        else
            level.head_order_idx = order.next_order_idx;
        
        if (order.next_order_idx >= 0)
            order_pool[order.next_order_idx].prev_order_idx = order.prev_order_idx;
        else
            level.tail_order_idx = order.prev_order_idx;
        
        level.order_count--;
    }
};
```

### 1.3 Direct-Indexed Book (Fastest for Known Tick Instruments)

For instruments with fixed tick size, map price directly to an array index:

```cpp
class DirectIndexBook {
    // For an instrument with tick_size = $0.01, price range $50-$150:
    // Array size = (150 - 50) / 0.01 = 10,000 entries
    
    struct Level {
        int32_t qty;    // 0 = empty level
        int16_t count;
    };
    
    Level levels[20000];  // Pre-allocated for full price range
    int64_t base_price;   // Lowest possible price (in ticks)
    
    // Best bid/ask tracking (avoid scanning)
    int best_bid_offset = -1;
    int best_ask_offset = -1;
    
public:
    // O(1) update
    void set_level(int64_t price_ticks, int32_t qty, int16_t count) {
        int offset = price_ticks - base_price;
        levels[offset] = {qty, count};
        
        // Update best bid/ask
        if (qty > 0) {
            if (offset > best_bid_offset) best_bid_offset = offset;
            if (best_ask_offset < 0 || offset < best_ask_offset) best_ask_offset = offset;
        }
    }
    
    // O(1) lookup by price
    const Level& at(int64_t price_ticks) const {
        return levels[price_ticks - base_price];
    }
};
```

---

## 2. ITCH Book Reconstruction

### 2.1 Complete ITCH 5.0 Book Builder

```cpp
class ITCHBookBuilder {
    // Per-instrument books
    L3Book books[65536];  // Indexed by stock locate (16-bit)
    
    // Sequence tracking
    uint64_t expected_seq = 0;
    
public:
    void process_message(const uint8_t* data, size_t len) {
        uint8_t msg_type = data[0];
        
        switch (msg_type) {
            case 'A':  // Add Order (no MPID)
                on_add_order(data);
                break;
            case 'F':  // Add Order with MPID
                on_add_order_mpid(data);
                break;
            case 'E':  // Order Executed
                on_order_executed(data);
                break;
            case 'C':  // Order Executed with Price
                on_order_executed_price(data);
                break;
            case 'X':  // Order Cancel
                on_order_cancel(data);
                break;
            case 'D':  // Order Delete
                on_order_delete(data);
                break;
            case 'U':  // Order Replace
                on_order_replace(data);
                break;
            case 'P':  // Trade (non-cross)
                on_trade(data);
                break;
        }
    }
    
private:
    void on_add_order(const uint8_t* data) {
        // ITCH Add Order format (offset from message start):
        // Byte 0: Message type 'A'
        // Bytes 1-2: Stock locate (uint16 big-endian)
        // Bytes 3-4: Tracking number
        // Bytes 5-10: Timestamp (6 bytes, nanoseconds since midnight)
        // Bytes 11-18: Order reference number (uint64 big-endian)
        // Byte 19: Buy/Sell indicator ('B' or 'S')
        // Bytes 20-23: Shares (uint32 big-endian)
        // Bytes 24-31: Stock (8 bytes, space-padded ASCII)
        // Bytes 32-35: Price (uint32 big-endian, 4 decimal places)
        
        uint16_t locate = read_u16_be(data + 1);
        uint64_t order_ref = read_u64_be(data + 11);
        uint8_t side = (data[19] == 'B') ? 0 : 1;
        uint32_t shares = read_u32_be(data + 20);
        uint32_t price = read_u32_be(data + 32);
        
        books[locate].add_order(order_ref, side, price, shares);
    }
    
    void on_order_executed(const uint8_t* data) {
        uint16_t locate = read_u16_be(data + 1);
        uint64_t order_ref = read_u64_be(data + 11);
        uint32_t exec_shares = read_u32_be(data + 19);
        
        books[locate].execute_order(order_ref, exec_shares);
    }
    
    void on_order_cancel(const uint8_t* data) {
        uint16_t locate = read_u16_be(data + 1);
        uint64_t order_ref = read_u64_be(data + 11);
        uint32_t cancel_shares = read_u32_be(data + 19);
        
        books[locate].cancel_order(order_ref, cancel_shares);
    }
    
    void on_order_delete(const uint8_t* data) {
        uint16_t locate = read_u16_be(data + 1);
        uint64_t order_ref = read_u64_be(data + 11);
        
        books[locate].cancel_order(order_ref, INT32_MAX);  // Delete = cancel all
    }
    
    void on_order_replace(const uint8_t* data) {
        uint16_t locate = read_u16_be(data + 1);
        uint64_t old_ref = read_u64_be(data + 11);
        uint64_t new_ref = read_u64_be(data + 19);
        uint32_t shares = read_u32_be(data + 27);
        uint32_t price = read_u32_be(data + 31);
        
        // Replace = delete old + add new (loses queue position)
        auto* old_idx = books[locate].order_map.find(old_ref);
        uint8_t side = 0;
        if (old_idx) {
            side = books[locate].order_pool[*old_idx].side;
        }
        
        books[locate].cancel_order(old_ref, INT32_MAX);
        books[locate].add_order(new_ref, side, price, shares);
    }
    
    // Endian conversion helpers
    static uint16_t read_u16_be(const uint8_t* p) {
        return (p[0] << 8) | p[1];
    }
    
    static uint32_t read_u32_be(const uint8_t* p) {
        return (p[0] << 24) | (p[1] << 16) | (p[2] << 8) | p[3];
    }
    
    static uint64_t read_u64_be(const uint8_t* p) {
        return ((uint64_t)read_u32_be(p) << 32) | read_u32_be(p + 4);
    }
};
```

---

## 3. Performance Benchmarks

| Operation | L2 (sorted array) | L3 (linked list + hash) | Direct-indexed |
|---|---|---|---|
| Best bid/ask | O(1) | O(1) with tracking | O(1) with tracking |
| Price level lookup | O(log n) binary search | O(1) hash + O(1) offset | O(1) |
| Add order | O(n) shift | O(1) amortized | O(1) |
| Cancel order | O(n) shift | O(1) | O(1) |
| Full book scan | O(n) sequential | O(n) pointer chase | O(range) |
| Cache behavior | Excellent (contiguous) | Poor (pointer chasing) | Excellent |
| Memory usage | ~640 B (20 levels) | ~40 B/order | ~80 KB/instrument |

**Recommendation:** Use L2 sorted array for strategy evaluation (cache-friendly, SIMD-able). Use L3 with flat hash map + direct-indexed levels for book reconstruction from ITCH.

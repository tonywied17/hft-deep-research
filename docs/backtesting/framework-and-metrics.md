# Backtesting Framework Design

---

## 1. Architecture

### 1.1 Event-Driven Backtesting Engine

```
┌─────────────────────────────────────────────────┐
│                 Data Manager                      │
│  [Historical Market Data] → Normalized Feed       │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│               Event Engine                        │
│  Timeline: sorted events (ticks, fills, timers)   │
│  Dispatches events in chronological order         │
└───────────────────┬─────────────────────────────┘
                    │
          ┌─────────┼─────────┐
          ▼         ▼         ▼
    ┌──────────┐ ┌──────────┐ ┌──────────┐
    │ Strategy │ │ Strategy │ │ Strategy │
    │    A     │ │    B     │ │    C     │
    └────┬─────┘ └────┬─────┘ └────┬─────┘
         │            │            │
         ▼            ▼            ▼
    ┌─────────────────────────────────────┐
    │          Simulated Exchange          │
    │  Order book, matching, fills, fees   │
    └───────────────────┬─────────────────┘
                        │
                        ▼
    ┌─────────────────────────────────────┐
    │        Performance Analytics         │
    │  PnL, Sharpe, drawdown, fill stats   │
    └─────────────────────────────────────┘
```

### 1.2 Core Engine

```cpp
struct Event {
    uint64_t timestamp_ns;    // Nanoseconds since epoch
    uint16_t type;            // MARKET_DATA, FILL, TIMER, etc.
    uint16_t instrument_id;
    uint32_t payload_size;
    uint8_t  payload[];       // Variable-size data
};

class BacktestEngine {
    // Priority queue of events sorted by timestamp
    // Use a sorted array (not std::priority_queue) for cache efficiency
    std::vector<Event*> event_queue;
    
    uint64_t current_time = 0;
    
    struct Strategy {
        virtual void on_market_data(uint16_t instrument, const MarketUpdate& update) = 0;
        virtual void on_fill(const Fill& fill) = 0;
        virtual void on_timer(uint64_t timer_id) = 0;
        virtual ~Strategy() = default;
    };
    
    std::vector<Strategy*> strategies;
    SimulatedExchange exchange;
    
    void run() {
        while (!event_queue.empty()) {
            Event* event = event_queue.front();
            event_queue.erase(event_queue.begin());
            
            current_time = event->timestamp_ns;
            
            switch (event->type) {
                case MARKET_DATA:
                    dispatch_market_data(event);
                    break;
                case FILL:
                    dispatch_fill(event);
                    break;
                case TIMER:
                    dispatch_timer(event);
                    break;
            }
        }
    }
    
    void dispatch_market_data(const Event* event) {
        auto& update = *reinterpret_cast<const MarketUpdate*>(event->payload);
        
        // Update exchange simulator (book, trades)
        exchange.process_update(event->instrument_id, update, current_time);
        
        // Notify all strategies
        for (auto* strategy : strategies) {
            strategy->on_market_data(event->instrument_id, update);
        }
        
        // Process any resulting orders from strategies
        exchange.match_orders(current_time);
    }
};
```

---

## 2. Simulated Exchange

### 2.1 Fill Simulation

The fill model is the most critical part of a backtester. Poor fill assumptions lead to wildly optimistic results.

```cpp
class SimulatedExchange {
    struct SimOrder {
        uint64_t order_id;
        uint16_t instrument_id;
        int8_t side;              // 1=buy, 2=sell
        int64_t price;
        int32_t remaining_qty;
        uint64_t submit_time;
        int64_t queue_position;    // Estimated shares ahead
    };
    
    std::vector<SimOrder> resting_orders;
    
    struct FillModel {
        // Conservative fill assumptions
        double queue_fill_fraction = 0.5;  // Assume only 50% of queue trades through
        bool require_trade_through = true;  // Only fill if trade occurs at our price
        uint64_t latency_ns = 1000;         // Simulated order latency (1 μs)
    };
    
    FillModel fill_model;
    
    void check_fills(uint16_t instrument, int64_t trade_price, int32_t trade_qty) {
        for (auto& order : resting_orders) {
            if (order.instrument_id != instrument) continue;
            
            bool price_match = false;
            
            if (order.side == 1) {  // Buy
                if (fill_model.require_trade_through) {
                    // Only fill if trade at or below our price
                    price_match = (trade_price <= order.price);
                } else {
                    price_match = (trade_price <= order.price);
                }
            } else {  // Sell
                price_match = (trade_price >= order.price);
            }
            
            if (!price_match) continue;
            
            // Queue position simulation
            if (trade_price == order.price) {
                // At our price — need to consider queue
                int64_t effective_volume = trade_qty * fill_model.queue_fill_fraction;
                
                if (order.queue_position > 0) {
                    order.queue_position -= effective_volume;
                    if (order.queue_position > 0) continue;  // Not our turn yet
                }
                
                // Fill (partial or full)
                int32_t fill_qty = std::min(order.remaining_qty, trade_qty);
                generate_fill(order, fill_qty, order.price);
                order.remaining_qty -= fill_qty;
            } else {
                // Trade through our price — guaranteed fill
                generate_fill(order, order.remaining_qty, order.price);
                order.remaining_qty = 0;
            }
        }
    }
};
```

### 2.2 Latency Simulation

```cpp
struct LatencyModel {
    uint64_t base_latency_ns = 500;     // Wire + exchange processing
    uint64_t jitter_ns = 200;           // Random variation
    
    // For more realistic simulation:
    // - Add separate ingress (market data) and egress (order) latencies
    // - Model queue processing time at exchange
    // - Account for co-location vs remote
    
    uint64_t order_latency() const {
        // Simple model: base + uniform jitter
        return base_latency_ns + (rand() % jitter_ns);
    }
    
    uint64_t market_data_latency() const {
        return base_latency_ns / 2;  // MD typically faster (multicast)
    }
};
```

---

## 3. Data Management

### 3.1 Data Requirements

```
L1 Data (Top of Book):
  - Timestamp (nanosecond)
  - Bid/Ask price and size
  - Adequate for: simple strategies, backtesting prototypes
  
L2 Data (Depth of Book):
  - Full price ladder (typically 5-20 levels)
  - Aggregate quantity and order count per level
  - Required for: market making, book-based signals
  
L3 Data (Order-Level / ITCH):
  - Individual order add/modify/cancel/execute
  - Enables exact queue position tracking
  - Required for: precise fill simulation, queue-based strategies
  
Trade Data:
  - Every trade: price, quantity, aggressor side
  - Required for: VPIN calculation, trade flow analysis

Tick Data Sources:
  - LOBSTER (academic): NASDAQ ITCH reconstructed books
  - TAQ (NYSE): Trades and quotes
  - QuantQuote/Algoseek: Commercial tick data
  - Exchange historical data feeds: Direct from exchanges
```

### 3.2 Data Format

```cpp
// Efficient binary format for backtesting
struct __attribute__((packed)) TickRecord {
    uint64_t timestamp_ns;    // 8 bytes
    uint16_t instrument_id;   // 2 bytes
    uint8_t  record_type;     // 1 byte (QUOTE=1, TRADE=2, BOOK=3)
    uint8_t  flags;           // 1 byte
    
    union {
        struct {  // QUOTE
            int64_t bid_price;
            int32_t bid_size;
            int64_t ask_price;
            int32_t ask_size;
        } quote;
        
        struct {  // TRADE
            int64_t price;
            int32_t quantity;
            uint8_t aggressor;  // 1=buy, 2=sell
        } trade;
    };
};

// Memory-mapped file reader for zero-copy replay
class TickDataReader {
    const uint8_t* mapped_data;
    size_t file_size;
    size_t offset = 0;
    
public:
    bool open(const char* path) {
        // Memory-map the file
        int fd = ::open(path, O_RDONLY);
        file_size = ::lseek(fd, 0, SEEK_END);
        mapped_data = (const uint8_t*)::mmap(nullptr, file_size, 
                                              PROT_READ, MAP_PRIVATE, fd, 0);
        // Advise sequential access
        ::madvise((void*)mapped_data, file_size, MADV_SEQUENTIAL);
        ::close(fd);
        return mapped_data != MAP_FAILED;
    }
    
    const TickRecord* next() {
        if (offset + sizeof(TickRecord) > file_size) return nullptr;
        const TickRecord* record = reinterpret_cast<const TickRecord*>(mapped_data + offset);
        offset += sizeof(TickRecord);
        return record;
    }
};
```

---

## 4. Performance Metrics

### 4.1 Core Metrics

```cpp
struct PerformanceMetrics {
    std::vector<double> daily_returns;
    std::vector<double> equity_curve;
    
    // Sharpe Ratio (annualized)
    double sharpe_ratio() const {
        double mean = accumulate(daily_returns) / daily_returns.size();
        double variance = 0;
        for (double r : daily_returns) variance += (r - mean) * (r - mean);
        variance /= (daily_returns.size() - 1);
        
        return (mean / std::sqrt(variance)) * std::sqrt(252);  // Annualize
    }
    
    // Sortino Ratio (only penalizes downside vol)
    double sortino_ratio() const {
        double mean = accumulate(daily_returns) / daily_returns.size();
        double downside_var = 0;
        int n = 0;
        for (double r : daily_returns) {
            if (r < 0) { downside_var += r * r; ++n; }
        }
        if (n == 0) return 0;
        downside_var /= n;
        
        return (mean / std::sqrt(downside_var)) * std::sqrt(252);
    }
    
    // Maximum Drawdown
    double max_drawdown() const {
        double peak = equity_curve[0];
        double max_dd = 0;
        
        for (double eq : equity_curve) {
            peak = std::max(peak, eq);
            double dd = (peak - eq) / peak;
            max_dd = std::max(max_dd, dd);
        }
        
        return max_dd;
    }
    
    // Profit Factor = Gross Profit / Gross Loss
    double profit_factor() const {
        double gross_profit = 0, gross_loss = 0;
        for (double r : daily_returns) {
            if (r > 0) gross_profit += r;
            else gross_loss -= r;
        }
        return (gross_loss > 0) ? gross_profit / gross_loss : 0;
    }
    
    // Win Rate
    double win_rate() const {
        int wins = 0;
        for (double r : daily_returns) if (r > 0) ++wins;
        return static_cast<double>(wins) / daily_returns.size();
    }
    
    // Calmar Ratio = Annualized Return / Max Drawdown
    double calmar_ratio() const {
        double ann_return = accumulate(daily_returns) / daily_returns.size() * 252;
        double mdd = max_drawdown();
        return (mdd > 0) ? ann_return / mdd : 0;
    }
    
private:
    static double accumulate(const std::vector<double>& v) {
        double s = 0; for (double x : v) s += x; return s;
    }
};
```

### 4.2 HFT-Specific Metrics

```
Fill Rate:          Filled orders / Total posted orders
Edge per Trade:     Average PnL per fill (in ticks)
PnL per Message:    Revenue per exchange message sent
Adverse Selection:  Average mid-price move after fill (should be small)
Inventory Duration: Average time to flatten position
Turnover:           Daily traded volume / Average position size
Queue Efficiency:   Fills from front-of-queue events / Total fills
Cancel Ratio:       Cancelled orders / Total orders (regulatory scrutiny if > 90%)
```

---

## 5. Common Pitfalls

### 5.1 Biases to Avoid

```
Lookahead Bias:
  Using information not available at the time of the decision.
  Examples:
    - Using the closing price before the close
    - VWAP calculation using future volume
    - Signal computed on data that hasn't arrived yet
  Fix: Strict timestamp ordering, separate data availability times

Survivorship Bias:
  Backtesting on stocks that still exist — ignoring delistings.
  Fix: Use point-in-time universe membership (e.g., historical S&P 500 constituents)

Overfitting:
  Strategy performs well in-sample but fails out-of-sample.
  Indicators:
    - Many tunable parameters
    - High sensitivity to parameter values
    - Narrow in-sample period
  Fix: Walk-forward testing, cross-validation, regularization

Data Snooping:
  Running many strategy variants until one "works" — selection bias.
  Fix: Bonferroni correction, hold out a test set, record all attempts

Market Impact Ignorance:
  Assuming your orders don't move the market.
  Fix: Model market impact, especially for larger positions
  
Latency Ignorance:
  Assuming zero-latency execution.
  Fix: Add realistic latency model (500 ns – 5 μs for co-located)

Fill Assumption Bias:
  Assuming all limit orders fill when price touches.
  Fix: Queue position modeling, require trade-through for fills
```

### 5.2 Walk-Forward Testing

```
Instead of a single in-sample/out-of-sample split:

│──── Train 1 ────│── Test 1 ──│
        │──── Train 2 ────│── Test 2 ──│
                │──── Train 3 ────│── Test 3 ──│
                        │──── Train 4 ────│── Test 4 ──│

For each window:
  1. Optimize parameters on training data
  2. Evaluate on test data
  3. Slide window forward
  
Aggregate PnL from all test periods = realistic performance estimate
```

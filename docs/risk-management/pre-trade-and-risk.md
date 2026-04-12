# Pre-Trade Risk Controls & Risk Management

---

## 1. Risk Control Architecture

### 1.1 Defense in Depth

```
Layer 1: Strategy-Level Checks (< 50 ns)
  → Per-order validation in the hot path
  → Branchless, cache-resident, zero allocation
  
Layer 2: Risk Engine Checks (< 200 ns)  
  → Aggregate position and exposure checks
  → Updated atomically via shared memory
  
Layer 3: Gateway-Level Checks (< 1 μs)
  → Pre-send validation before wire
  → Message rate limiting, fat-finger protection
  
Layer 4: Exchange-Level Checks
  → Exchange-enforced risk limits (Reg SCI)
  → Credit checks by clearing firm
  
Layer 5: Post-Trade Monitoring
  → T+0 reconciliation
  → PnL and position reporting
  → Regulatory reporting
```

### 1.2 Hot-Path Risk Engine

```cpp
struct PreTradeRisk {
    // All fields on a single cache line for speed
    alignas(64) struct Limits {
        int32_t max_order_qty;           // Maximum shares per order
        int32_t max_notional;            // Maximum notional per order ($)
        int32_t max_position;            // Maximum net position (shares)
        int32_t max_gross_exposure;      // Maximum total long + short ($)
        int32_t max_order_rate;          // Maximum orders per second
        int32_t max_loss;                // Maximum daily loss ($) (negative = loss)
        int64_t max_notional_daily;      // Maximum daily traded notional ($)
        int32_t max_open_orders;         // Maximum live orders
        int32_t max_message_rate;        // Maximum messages/second to exchange
    };
    
    // Current state (updated after every fill/order)
    alignas(64) struct State {
        int32_t net_position;
        int32_t gross_long;
        int32_t gross_short;
        int32_t realized_pnl;           // In cents (fixed-point)  
        int32_t unrealized_pnl;         // Updated on every tick
        int32_t open_order_count;
        int32_t orders_this_second;
        int32_t messages_this_second;
        int64_t traded_notional_today;
        uint64_t second_boundary_tsc;    // TSC at start of current second
    };
    
    Limits limits;
    State state;
    
    enum RejectReason : uint8_t {
        PASS = 0,
        MAX_ORDER_QTY,
        MAX_NOTIONAL,
        MAX_POSITION,
        GROSS_EXPOSURE,
        ORDER_RATE,
        DAILY_LOSS,
        DAILY_NOTIONAL,
        OPEN_ORDERS,
        MESSAGE_RATE,
        KILL_SWITCH_ACTIVE,
    };
    
    // Hot path — must be < 50 ns
    RejectReason check_order(int32_t qty, int64_t price, int8_t side) const {
        // 1. Order size
        if (__builtin_expect(qty > limits.max_order_qty, 0))
            return MAX_ORDER_QTY;
        
        // 2. Order notional
        int64_t notional = (int64_t)qty * price / 10000;  // Fixed-point price
        if (__builtin_expect(notional > limits.max_notional, 0))
            return MAX_NOTIONAL;
        
        // 3. Position limit
        int32_t new_pos = state.net_position + (side == 1 ? qty : -qty);
        if (__builtin_expect(std::abs(new_pos) > limits.max_position, 0))
            return MAX_POSITION;
        
        // 4. Gross exposure
        int32_t new_gross = state.gross_long + state.gross_short 
                          + (side == 1 ? qty : 0) + (side == 2 ? qty : 0);
        if (__builtin_expect(new_gross > limits.max_gross_exposure, 0))
            return GROSS_EXPOSURE;
        
        // 5. Order rate (per second)
        if (__builtin_expect(state.orders_this_second >= limits.max_order_rate, 0))
            return ORDER_RATE;
        
        // 6. Daily loss limit
        int32_t total_pnl = state.realized_pnl + state.unrealized_pnl;
        if (__builtin_expect(total_pnl < limits.max_loss, 0))
            return DAILY_LOSS;
        
        // 7. Daily notional
        if (__builtin_expect(state.traded_notional_today + notional > limits.max_notional_daily, 0))
            return DAILY_NOTIONAL;
        
        // 8. Open orders
        if (__builtin_expect(state.open_order_count >= limits.max_open_orders, 0))
            return OPEN_ORDERS;
        
        return PASS;
    }
};
```

---

## 2. Kill Switch

### 2.1 Automated Kill Switch

```cpp
class KillSwitch {
    std::atomic<bool> triggered{false};
    
    // Conditions that trigger automatic kill
    struct Triggers {
        int32_t max_daily_loss;          // Emergency stop loss
        int32_t max_drawdown;            // Peak-to-trough loss
        int32_t max_position_breach;     // Hard position limit
        int32_t max_error_count;         // Consecutive exchange rejects
        uint32_t max_latency_ns;         // Abnormal system latency
        uint32_t heartbeat_timeout_ms;   // Strategy process health
    };
    
    Triggers triggers;
    int32_t peak_pnl = 0;
    int32_t consecutive_errors = 0;
    
public:
    void evaluate(int32_t total_pnl, int32_t position, uint32_t latency_ns,
                  bool got_heartbeat) {
        // Track peak PnL for drawdown
        peak_pnl = std::max(peak_pnl, total_pnl);
        int32_t drawdown = peak_pnl - total_pnl;
        
        bool should_kill = false;
        
        should_kill |= (total_pnl < triggers.max_daily_loss);
        should_kill |= (drawdown > triggers.max_drawdown);
        should_kill |= (std::abs(position) > triggers.max_position_breach);
        should_kill |= (consecutive_errors > triggers.max_error_count);
        should_kill |= (latency_ns > triggers.max_latency_ns);
        should_kill |= (!got_heartbeat);
        
        if (should_kill) {
            trigger();
        }
    }
    
    void trigger() {
        if (triggered.exchange(true)) return;  // Already triggered
        
        // Emergency actions:
        // 1. Cancel all outstanding orders (mass cancel)
        // 2. Flatten all positions (market orders)
        // 3. Disconnect from exchanges
        // 4. Alert operations team
        // 5. Log reason and state
    }
    
    bool is_active() const { return triggered.load(std::memory_order_relaxed); }
};
```

### 2.2 Exchange-Level Kill Switches

```
CME Self-Match Prevention:
  - Tags on orders prevent self-trading
  - Matching engine cancels if same firm on both sides
  
NASDAQ Risk Management:
  - Credit-based risk controls
  - Real-time position monitoring
  - Ability to cancel all orders via single message

Exchange "Cancel on Disconnect":
  - All orders cancelled if TCP session drops
  - Prevents stale quotes from disconnected firm
  - Configurable per session
```

---

## 3. Position Management

### 3.1 Position Tracking

```cpp
struct PositionManager {
    struct Position {
        int32_t net_qty;           // Net position (positive = long)
        int64_t average_cost;      // Weighted average entry price (fixed-point)
        int64_t realized_pnl;     // Closed trades PnL
        int64_t total_bought_qty;
        int64_t total_sold_qty;
        int64_t total_bought_notional;
        int64_t total_sold_notional;
    };
    
    std::array<Position, 65536> positions;  // By instrument ID
    
    void on_fill(uint16_t instrument, int32_t qty, int64_t price, bool is_buy) {
        auto& pos = positions[instrument];
        
        if (is_buy) {
            if (pos.net_qty >= 0) {
                // Adding to long position
                int64_t new_cost = pos.average_cost * pos.net_qty + price * qty;
                pos.net_qty += qty;
                pos.average_cost = new_cost / pos.net_qty;
            } else {
                // Covering short position
                int32_t cover_qty = std::min(qty, -pos.net_qty);
                pos.realized_pnl += (pos.average_cost - price) * cover_qty;
                pos.net_qty += qty;
                
                if (pos.net_qty > 0) {
                    // Flipped to long
                    pos.average_cost = price;
                }
            }
            pos.total_bought_qty += qty;
            pos.total_bought_notional += price * qty;
        } else {
            // Sell — symmetric logic
            if (pos.net_qty <= 0) {
                int64_t new_cost = pos.average_cost * (-pos.net_qty) + price * qty;
                pos.net_qty -= qty;
                pos.average_cost = new_cost / (-pos.net_qty);
            } else {
                int32_t sell_qty = std::min(qty, pos.net_qty);
                pos.realized_pnl += (price - pos.average_cost) * sell_qty;
                pos.net_qty -= qty;
                
                if (pos.net_qty < 0) {
                    pos.average_cost = price;
                }
            }
            pos.total_sold_qty += qty;
            pos.total_sold_notional += price * qty;
        }
    }
    
    int64_t unrealized_pnl(uint16_t instrument, int64_t current_price) const {
        auto& pos = positions[instrument];
        return pos.net_qty * (current_price - pos.average_cost);
    }
};
```

### 3.2 PnL Decomposition

```
Total PnL = Realized PnL + Unrealized PnL

Realized PnL = Σ (exit_price - entry_price) × quantity × side
Unrealized PnL = net_position × (current_price - average_cost)

Further decomposition:
  Spread Capture PnL:    Revenue from earning bid-ask spread
  Inventory PnL:         Gain/loss from holding inventory
  Rebate PnL:            Exchange rebates earned
  Fee PnL:               Fees paid (taker fees, connectivity)
  Slippage PnL:          Execution price vs theoretical price
```

---

## 4. Value at Risk (VaR)

### 4.1 Historical VaR

```cpp
struct HistoricalVaR {
    static constexpr int WINDOW = 252;  // 1 year of trading days
    double returns[WINDOW];
    int count = 0;
    int head = 0;
    
    void add_return(double r) {
        returns[head] = r;
        head = (head + 1) % WINDOW;
        if (count < WINDOW) ++count;
    }
    
    double var_95() const {
        // Sort returns, find 5th percentile
        double sorted[WINDOW];
        std::copy(returns, returns + count, sorted);
        std::sort(sorted, sorted + count);
        
        int index = static_cast<int>(0.05 * count);
        return -sorted[index];  // Positive number representing loss
    }
    
    double cvar_95() const {
        // Conditional VaR = average of losses beyond VaR
        double sorted[WINDOW];
        std::copy(returns, returns + count, sorted);
        std::sort(sorted, sorted + count);
        
        int cutoff = static_cast<int>(0.05 * count);
        double sum = 0;
        for (int i = 0; i < cutoff; ++i) {
            sum += sorted[i];
        }
        return -(sum / cutoff);
    }
};
```

### 4.2 Parametric VaR

$$\text{VaR}_{1-\alpha} = -(\mu - z_\alpha \sigma) \times \text{Portfolio Value}$$

where $z_\alpha$ = normal quantile (1.645 for 95%, 2.326 for 99%).

### 4.3 Portfolio VaR (Multi-Asset)

$$\text{VaR}_p = z_\alpha \sqrt{w' \Sigma w} \times V$$

where $w$ = position weights, $\Sigma$ = covariance matrix.

---

## 5. Circuit Breakers

### 5.1 Exchange-Level Circuit Breakers

```
Market-Wide Circuit Breakers (Rule 80B):
  Level 1: S&P 500 drops 7%   → 15-min halt (before 3:25 PM)
  Level 2: S&P 500 drops 13%  → 15-min halt (before 3:25 PM)
  Level 3: S&P 500 drops 20%  → Market closed for day
  
  Reset: Based on prior day closing price
  Time: Only during regular trading hours (9:30 AM - 4:00 PM ET)

Limit Up / Limit Down (LULD):
  Price bands calculated from 5-min reference price:
  
  Tier 1 (S&P 500, Russell 1000):
    Band: ±5% from reference price (±10% during open/close)
  
  Tier 2 (Other NMS stocks):
    Band: ±10% from reference price (±20% during open/close)
  
  Band violation process:
    1. Straddle state: Trades only within band
    2. 15-second limit state: No trades at or beyond limit
    3. Trading pause: 5-minute halt (if limit not cleared)
```

### 5.2 Internal Circuit Breakers

```cpp
struct InternalCircuitBreaker {
    // Detect abnormal conditions and halt trading
    
    struct MarketCondition {
        double spread_multiple;    // Current spread / median spread
        double volume_multiple;    // Current rate / median rate
        double return_zscore;      // Short-term return standardized
    };
    
    bool should_halt(const MarketCondition& cond) const {
        // Abnormally wide spread (2.5x+ normal)
        if (cond.spread_multiple > 2.5) return true;
        
        // Extreme short-term move (> 5 sigma)
        if (std::abs(cond.return_zscore) > 5.0) return true;
        
        // Volume surge (could be news or flash crash)
        if (cond.volume_multiple > 10.0) return true;
        
        return false;
    }
};
```

---

## 6. Regulatory Requirements

### 6.1 Key Regulations

```
SEC Rule 15c3-5 (Market Access Rule):
  - Broker-dealers must implement pre-trade risk controls
  - Controls must be under exclusive control of the broker-dealer
  - Annual review required
  - Cannot rely solely on customer controls

Regulation SHO (Short Selling):
  - Locate requirement before short selling
  - Close-out requirement for fail-to-deliver
  - Circuit breaker: short sale price test (Rule 201)
    → Triggered when stock drops 10% from prior close
    → Applies for rest of day + next trading day
    → Short sales only allowed at price > current NBB

Regulation NMS:
  - Order Protection Rule (Rule 611): No trade-throughs
  - Access Fee Cap (Rule 610): Max $0.003/share
  - Sub-penny rule (Rule 612): No sub-penny quotes > $1.00
  - Market Data (Rule 603): SIP consolidation

CAT (Consolidated Audit Trail):
  - Every order and trade reported with unique identifiers
  - Links customer→order→execution across venues
  - Intraday reporting requirement
```

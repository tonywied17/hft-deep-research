# Market Making Algorithms

## Overview

Market making is the backbone HFT strategy: continuously providing liquidity by quoting bid and ask prices, earning the spread while managing inventory risk and adverse selection.

---

## 1. The Market Making Problem

### 1.1 Revenue Sources
- **Bid-Ask Spread Capture:** Buy at bid, sell at ask → earn the spread
- **Maker Rebates:** Exchanges pay $0.10-0.35 per 100 shares for posted liquidity
- **Information Edge:** Faster processing reveals favorable conditions before others

### 1.2 Cost Structure
- **Adverse Selection:** Fills correlate with unfavorable price moves (informed traders pick you off)
- **Inventory Risk:** Holding inventory while prices move against you
- **Taker Fees:** Aggressively crossing the spread to flatten position
- **Technology:** Co-location, hardware, development

### 1.3 Profitability Condition

A market maker is profitable if:
$$E[\text{Half-Spread}] + E[\text{Rebate}] > E[\text{Adverse Selection}] + E[\text{Inventory Cost}]$$

---

## 2. Avellaneda-Stoikov Framework

The foundational theoretical model for optimal market making (Avellaneda & Stoikov, 2008).

### 2.1 Setup

- Mid-price follows: $dS_t = \sigma \, dW_t$ (arithmetic Brownian motion)
- Market maker has inventory $q_t$ and exponential utility: $U(x) = -e^{-\gamma x}$
- Order arrival follows Poisson process with intensity: $\Lambda(\delta) = A \exp(-\kappa \delta)$
  - $\delta$ = distance from mid-price to our quote
  - $A$ = baseline arrival rate
  - $\kappa$ = arrival decay parameter

### 2.2 Optimal Quotes

**Reservation Price (Indifference Price):**

The price at which the market maker is indifferent to trading:
$$r(s, q, t) = s - q \gamma \sigma^2 (T - t)$$

This shifts the "center" of quotes based on inventory:
- Long inventory ($q > 0$): Reservation price drops → more eager to sell
- Short inventory ($q < 0$): Reservation price rises → more eager to buy

**Optimal Spread:**
$$\delta^* = \gamma \sigma^2 (T-t) + \frac{2}{\gamma} \ln\left(1 + \frac{\gamma}{\kappa}\right)$$

**Optimal Bid and Ask:**
$$P_{\text{bid}} = r - \frac{\delta^*}{2}$$
$$P_{\text{ask}} = r + \frac{\delta^*}{2}$$

### 2.3 Implementation

```cpp
struct AvellanedaStoikov {
    double gamma;     // Risk aversion (higher = more conservative)
    double sigma;     // Price volatility (annualized)
    double kappa;     // Order arrival decay parameter
    double T;         // Session end time (fraction of day)
    
    struct QuoteResult {
        double bid_price;
        double ask_price;
        double spread;
        double reservation_price;
    };
    
    QuoteResult compute_quotes(double mid_price, int inventory, double time_remaining) {
        // Reservation price
        double r = mid_price - inventory * gamma * sigma * sigma * time_remaining;
        
        // Optimal spread
        double spread = gamma * sigma * sigma * time_remaining 
                       + (2.0 / gamma) * std::log(1.0 + gamma / kappa);
        
        return {
            .bid_price = r - spread / 2.0,
            .ask_price = r + spread / 2.0,
            .spread = spread,
            .reservation_price = r,
        };
    }
};
```

### 2.4 Parameter Calibration

| Parameter | Calibration Method | Typical Range |
|---|---|---|
| $\gamma$ | Set based on desired inventory tolerance | 0.001 – 0.1 |
| $\sigma$ | Realized volatility from recent data | Annualized vol |
| $\kappa$ | Fit to historical fill rate vs. distance from mid | 1 – 100 |
| $A$ | Observed order arrival rate at BBO | 10 – 10,000/sec |
| $T$ | Time until market close or position flatten | 0 – 1 (fraction of day) |

---

## 3. Practical Market Making Enhancements

### 3.1 Inventory Skewing

The simplest and most important adjustment — skew quotes based on inventory:

```cpp
struct InventorySkew {
    int max_position;         // Maximum allowed inventory
    double skew_per_unit;     // Price offset per unit of inventory
    double max_skew;          // Maximum skew (caps extreme adjustments)
    
    std::pair<double, double> skew_quotes(double bid, double ask, int inventory) {
        double skew = std::clamp(inventory * skew_per_unit, -max_skew, max_skew);
        
        // Lower both quotes when long, raise when short
        double skewed_bid = bid - skew;
        double skewed_ask = ask - skew;
        
        return {skewed_bid, skewed_ask};
    }
};
```

### 3.2 Volatility-Adaptive Spread

Widen the spread during high volatility to compensate for adverse selection:

```cpp
double adaptive_spread(double base_spread, double current_vol, double baseline_vol) {
    double vol_ratio = current_vol / baseline_vol;
    
    // Square root scaling (empirically motivated)
    return base_spread * std::sqrt(vol_ratio);
}

// Estimating short-term volatility (exponential moving variance)
struct EMVar {
    double lambda = 0.99;  // Decay factor
    double variance = 0;
    double last_price = 0;
    bool initialized = false;
    
    void update(double price) {
        if (initialized) {
            double ret = std::log(price / last_price);
            variance = lambda * variance + (1 - lambda) * ret * ret;
        }
        last_price = price;
        initialized = true;
    }
    
    double vol() const { return std::sqrt(variance); }
};
```

### 3.3 Queue Position Management

Your position in the order queue determines fill probability. Strategies to maintain queue priority:

```cpp
struct QueueManager {
    // Cancel and re-send only when necessary
    // Every cancel-replace loses queue position
    
    bool should_requote(double current_quote, double target_quote, 
                        double tick_size, int queue_position, int queue_depth) {
        double price_diff = std::abs(current_quote - target_quote);
        
        // Don't move if price change is less than one tick
        if (price_diff < tick_size) return false;
        
        // Don't move if we have excellent queue position (front 20%)
        double queue_fraction = static_cast<double>(queue_position) / queue_depth;
        if (queue_fraction < 0.2 && price_diff < 2 * tick_size) return false;
        
        return true;
    }
};
```

### 3.4 Adverse Selection Detection

Detect and avoid toxic order flow:

```cpp
struct ToxicityDetector {
    // Track realized spread vs quoted spread
    double sum_realized_spread = 0;
    double sum_quoted_spread = 0;
    int fill_count = 0;
    
    void on_fill(double fill_price, double mid_at_fill, double mid_after_100ms, int side) {
        double signed_edge = (side == BID) 
            ? (mid_after_100ms - fill_price) 
            : (fill_price - mid_after_100ms);
        
        sum_realized_spread += signed_edge;
        fill_count++;
    }
    
    double toxicity_ratio() const {
        if (fill_count == 0) return 0;
        // Negative = adverse selection exceeds spread capture
        return sum_realized_spread / fill_count;
    }
    
    bool is_toxic_flow() const {
        return fill_count > 100 && toxicity_ratio() < 0;
    }
};
```

---

## 4. Multi-Level Quoting

### 4.1 Layered Quotes

Place orders at multiple price levels with decreasing size:

```cpp
struct MultiLevelQuoter {
    int num_levels = 5;
    double base_size = 100;
    double size_decay = 0.7;     // Each level: 70% of previous
    double level_offset = 1.0;   // Ticks between levels
    
    struct Quote {
        double price;
        int quantity;
    };
    
    std::vector<Quote> generate_bid_ladder(double best_bid, double tick_size) {
        std::vector<Quote> quotes;
        double size = base_size;
        
        for (int i = 0; i < num_levels; ++i) {
            double price = best_bid - i * level_offset * tick_size;
            quotes.push_back({price, static_cast<int>(size)});
            size *= size_decay;
        }
        
        return quotes;
    }
};
```

### 4.2 Benefits of Multi-Level Quoting

1. **Higher fill rate:** Capture sweeps that go through top of book
2. **Better average fill price:** Deeper fills happen at better prices for the maker
3. **Reduced adverse selection:** Deeper fills less likely to be informed (larger spread cushion)
4. **Queue position diversity:** Maintain priority at multiple levels

---

## 5. Hedging & Inventory Flattening

### 5.1 Passive Hedging

Let inventory naturally mean-revert through quote skewing — the skewed quotes attract fills on the side that reduces inventory.

### 5.2 Active Hedging

When inventory exceeds threshold, aggressively cross the spread:

```cpp
struct ActiveHedger {
    int soft_limit;    // Start passive hedging
    int hard_limit;    // Start active hedging
    int max_limit;     // Emergency flatten
    
    enum Action { NONE, WIDEN, AGGRESSIVE_CROSS, EMERGENCY_FLATTEN };
    
    Action evaluate(int inventory) {
        int abs_inv = std::abs(inventory);
        
        if (abs_inv <= soft_limit) return NONE;
        if (abs_inv <= hard_limit) return WIDEN;  // Widen spread on full side
        if (abs_inv <= max_limit)  return AGGRESSIVE_CROSS;
        return EMERGENCY_FLATTEN;
    }
};
```

### 5.3 Cross-Asset Hedging

For related instruments (e.g., futures vs ETF, correlated stocks):

```
If long 1000 shares of SPY:
  → Sell 2 E-mini S&P futures (approximately equivalent exposure)
  → Or short a basket of constituent stocks
  
Hedge ratio from regression:
  β = Cov(R_instrument, R_hedge) / Var(R_hedge)
```

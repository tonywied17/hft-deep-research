# Statistical Arbitrage & Signal Generation

## Part 1: Statistical Arbitrage

---

## 1. Pairs Trading

### 1.1 Overview

Pairs trading exploits temporary divergences in the price relationship between two cointegrated securities. When the spread deviates from its mean, trade in the direction of convergence.

### 1.2 Pair Selection Pipeline

```
Step 1: Universe filtering
  → Same sector, similar market cap, high correlation (ρ > 0.8)
  
Step 2: Cointegration testing (Engle-Granger)
  → Regress Y on X: Y_t = α + β·X_t + ε_t
  → Test residuals ε_t for stationarity (ADF test, p < 0.05)
  
Step 3: Half-life estimation
  → Fit OU model to spread: dx = θ(μ - x)dt + σdW
  → Half-life = ln(2)/θ
  → Reject pairs with half-life < 1 day or > 30 days
  
Step 4: Economic validation
  → Confirm economic rationale for relationship
  → Check for structural breaks (rolling cointegration test)
```

### 1.3 Spread Construction

```cpp
struct PairsSpread {
    double beta;          // Hedge ratio from cointegration regression
    double mean;          // Long-run mean of spread
    double std_dev;       // Standard deviation of spread
    double half_life;     // Half-life in same time units as data
    
    // Compute current spread
    double spread(double price_y, double price_x) const {
        return price_y - beta * price_x;
    }
    
    // Z-score normalization
    double z_score(double price_y, double price_x) const {
        return (spread(price_y, price_x) - mean) / std_dev;
    }
};
```

### 1.4 Trading Signals

```cpp
struct PairsSignal {
    double entry_z = 2.0;     // Enter when |z| > entry_z
    double exit_z = 0.5;      // Exit when |z| < exit_z  
    double stop_z = 4.0;      // Stop loss when |z| > stop_z
    
    enum Signal { NONE, LONG_SPREAD, SHORT_SPREAD, EXIT, STOP };
    
    Signal evaluate(double z, int current_position) {
        if (current_position == 0) {
            // Entry signals
            if (z > entry_z)  return SHORT_SPREAD;  // Spread too wide, sell Y buy X
            if (z < -entry_z) return LONG_SPREAD;    // Spread too narrow, buy Y sell X
        } else {
            // Exit signals
            if (std::abs(z) < exit_z) return EXIT;
            if (std::abs(z) > stop_z) return STOP;
        }
        return NONE;
    }
};
```

### 1.5 Position Sizing (Kelly Criterion)

For a mean-reverting spread with known parameters:

$$f^* = \frac{\mu}{\sigma^2}$$

where:
- $f^*$ = optimal fraction of capital
- $\mu$ = expected return per period (from OU model)
- $\sigma^2$ = variance of returns

In practice, use **half-Kelly** ($f^*/2$) for robustness to parameter estimation error.

```cpp
double kelly_fraction(double expected_return, double return_variance) {
    double full_kelly = expected_return / return_variance;
    return std::clamp(full_kelly * 0.5, 0.0, 0.25);  // Half-Kelly, capped at 25%
}
```

---

## 2. Multi-Asset Statistical Arbitrage

### 2.1 PCA-Based Approach

Use Principal Component Analysis to find common factors, then trade residuals:

```
1. Compute returns matrix R (n_assets × n_observations)
2. PCA: R ≈ F·L' + ε
   F = factor returns (market, sector, etc.)
   L = factor loadings
   ε = residuals (idiosyncratic returns)
3. Trade assets where ε is unusually large (mean-reverting)
```

### 2.2 Eigenportfolio Construction

```
Given eigenvectors v_1, v_2, ..., v_k from PCA:

Market-neutral portfolio weights:
  w = ε_t / ||ε_t||  (trade proportional to residual magnitude)
  
  Constraint: Σ w_i = 0  (dollar-neutral)
  Constraint: Σ β_i · w_i = 0  (beta-neutral)
```

### 2.3 Kalman Filter for Dynamic Hedge Ratio

Static hedge ratios become stale. Use a Kalman filter to track the evolving relationship:

```cpp
struct KalmanHedgeRatio {
    // State: x = [alpha, beta]  (intercept and hedge ratio)
    // Observation: y_t = alpha + beta * x_t + noise
    
    double alpha = 0;
    double beta = 1;
    
    // State covariance
    double P[2][2] = {{1, 0}, {0, 1}};
    
    // Process noise (how fast the relationship changes)
    double Q_alpha = 1e-6;
    double Q_beta = 1e-6;
    
    // Observation noise
    double R = 1e-3;
    
    double update(double y, double x) {
        // Predict (identity transition — random walk model)
        P[0][0] += Q_alpha;
        P[1][1] += Q_beta;
        
        // Innovation
        double y_hat = alpha + beta * x;
        double innovation = y - y_hat;
        
        // Innovation covariance
        // H = [1, x], S = H * P * H' + R
        double S = P[0][0] + 2 * x * P[0][1] + x * x * P[1][1] + R;
        
        // Kalman gain
        double K0 = (P[0][0] + x * P[0][1]) / S;
        double K1 = (P[0][1] + x * P[1][1]) / S;
        
        // Update state
        alpha += K0 * innovation;
        beta += K1 * innovation;
        
        // Update covariance
        double P00 = P[0][0], P01 = P[0][1], P11 = P[1][1];
        P[0][0] = P00 - K0 * (P00 + x * P01);
        P[0][1] = P01 - K0 * (P01 + x * P11);
        P[1][0] = P[0][1];
        P[1][1] = P11 - K1 * (P01 + x * P11);
        
        // Return spread (innovation)
        return innovation;
    }
};
```

---

## 3. ETF Arbitrage

### 3.1 NAV Premium/Discount

```
ETF Price vs Net Asset Value:
  Premium = (ETF_Price - NAV) / NAV

  If Premium > threshold:
    → Short ETF, Buy constituent basket (creation arbitrage)
  
  If Premium < -threshold:
    → Buy ETF, Short constituent basket (redemption arbitrage)
```

### 3.2 Implementation Considerations

```cpp
struct ETFArbitrage {
    struct Constituent {
        uint16_t instrument_id;
        double weight;          // Portfolio weight
        double shares_per_cu;   // Shares per creation unit
    };
    
    std::vector<Constituent> basket;
    int creation_unit_size;      // Typically 50,000 shares of ETF
    
    double compute_nav(const double* mid_prices) const {
        double nav = 0;
        for (const auto& c : basket) {
            nav += c.weight * mid_prices[c.instrument_id];
        }
        return nav;
    }
    
    double premium(double etf_price, const double* mid_prices) const {
        double nav = compute_nav(mid_prices);
        return (etf_price - nav) / nav;
    }
    
    // Check if all constituents are tradeable
    bool basket_executable(const bool* active_flags) const {
        for (const auto& c : basket) {
            if (!active_flags[c.instrument_id]) return false;
        }
        return true;
    }
};
```

---

## Part 2: Signal Generation

---

## 4. Order Book Microstructure Signals

### 4.1 Order Book Imbalance (OBI)

The most powerful short-term directional predictor:

$$\text{OBI}_L = \frac{\sum_{l=1}^{L} w_l \cdot Q_{\text{bid}}^{(l)} - \sum_{l=1}^{L} w_l \cdot Q_{\text{ask}}^{(l)}}{\sum_{l=1}^{L} w_l \cdot Q_{\text{bid}}^{(l)} + \sum_{l=1}^{L} w_l \cdot Q_{\text{ask}}^{(l)}}$$

where $w_l = e^{-\alpha l}$ gives exponentially decaying weights across levels.

```cpp
struct OBISignal {
    double alpha = 0.5;  // Decay rate across levels
    int num_levels = 5;
    
    double compute(const PriceLevel* bids, const PriceLevel* asks) const {
        double bid_weight_sum = 0, ask_weight_sum = 0;
        double w = 1.0;
        
        for (int l = 0; l < num_levels; ++l) {
            bid_weight_sum += w * bids[l].total_qty;
            ask_weight_sum += w * asks[l].total_qty;
            w *= std::exp(-alpha);
        }
        
        double denom = bid_weight_sum + ask_weight_sum;
        if (denom < 1e-10) return 0;
        
        return (bid_weight_sum - ask_weight_sum) / denom;
    }
};
```

### 4.2 Trade Flow Imbalance

Track signed trade volume over rolling windows:

```cpp
struct TradeFlowImbalance {
    // Circular buffers for rolling calculation
    static constexpr int WINDOW = 100;  // Last 100 trades
    
    struct Trade {
        int32_t signed_volume;  // Positive = buy, negative = sell
        uint64_t timestamp;
    };
    
    Trade trades[WINDOW];
    int head = 0;
    int count = 0;
    
    void add_trade(int32_t qty, bool is_buy) {
        trades[head] = {is_buy ? qty : -qty, __rdtsc()};
        head = (head + 1) % WINDOW;
        if (count < WINDOW) ++count;
    }
    
    double imbalance() const {
        int64_t buy_vol = 0, sell_vol = 0;
        for (int i = 0; i < count; ++i) {
            if (trades[i].signed_volume > 0) buy_vol += trades[i].signed_volume;
            else sell_vol -= trades[i].signed_volume;
        }
        double total = buy_vol + sell_vol;
        if (total == 0) return 0;
        return static_cast<double>(buy_vol - sell_vol) / total;
    }
};
```

### 4.3 VPIN (Volume-Synchronized Probability of Informed Trading)

```cpp
struct VPIN {
    double bucket_volume;  // Volume per bucket (e.g., 1/50 of daily volume)
    int num_buckets;       // Number of buckets in window (e.g., 50)
    
    struct Bucket {
        double buy_volume;
        double sell_volume;
    };
    
    std::deque<Bucket> buckets;
    double current_buy = 0;
    double current_sell = 0;
    double current_total = 0;
    
    void add_trade(double price, double volume, double prev_mid) {
        // Bulk volume classification
        // Classify trade as buy or sell based on price relative to mid
        double buy_frac = cdf_normal((price - prev_mid) / tick_vol);
        current_buy += volume * buy_frac;
        current_sell += volume * (1 - buy_frac);
        current_total += volume;
        
        while (current_total >= bucket_volume) {
            double excess = current_total - bucket_volume;
            double ratio = 1.0 - excess / current_total;
            
            buckets.push_back({current_buy * ratio, current_sell * ratio});
            if (buckets.size() > num_buckets) buckets.pop_front();
            
            current_buy *= (1 - ratio);
            current_sell *= (1 - ratio);
            current_total = excess;
        }
    }
    
    double compute() const {
        if (buckets.size() < num_buckets) return 0;
        
        double sum_abs_imbalance = 0;
        double total_volume = 0;
        
        for (const auto& b : buckets) {
            sum_abs_imbalance += std::abs(b.buy_volume - b.sell_volume);
            total_volume += b.buy_volume + b.sell_volume;
        }
        
        return sum_abs_imbalance / total_volume;
    }
};
```

### 4.4 Micro-Price Signal

The volume-weighted mid provides a more informative fair value estimate:

$$P_{\text{micro}} = P_{\text{ask}} \cdot \frac{Q_{\text{bid}}}{Q_{\text{bid}} + Q_{\text{ask}}} + P_{\text{bid}} \cdot \frac{Q_{\text{ask}}}{Q_{\text{bid}} + Q_{\text{ask}}}$$

```cpp
double micro_price(double bid_px, double ask_px, int bid_qty, int ask_qty) {
    double total = bid_qty + ask_qty;
    if (total == 0) return (bid_px + ask_px) / 2;
    return (ask_px * bid_qty + bid_px * ask_qty) / total;
}
```

### 4.5 Spread Crossing Probability

Estimate the probability that the next trade will occur at the bid vs the ask:

```cpp
struct CrossingProbability {
    // Based on empirical observation:
    // P(next trade at ask) ≈ Q_bid / (Q_bid + Q_ask)
    // When bid is heavy, more likely to see a buy (exhaust the ask)
    
    double prob_buy(int bid_qty, int ask_qty) const {
        double total = bid_qty + ask_qty;
        if (total == 0) return 0.5;
        return static_cast<double>(bid_qty) / total;
    }
};
```

### 4.6 Combined Signal Scoring

```cpp
struct SignalCombiner {
    double obi_weight = 0.3;
    double trade_flow_weight = 0.25;
    double vpin_weight = 0.15;
    double micro_weight = 0.3;
    
    double combined_signal(double obi, double trade_flow, 
                          double vpin, double micro_vs_mid) {
        // Normalize each signal to [-1, 1] range
        double norm_obi = std::clamp(obi, -1.0, 1.0);
        double norm_tf = std::clamp(trade_flow, -1.0, 1.0);
        double norm_vpin = std::clamp(vpin * 2 - 1, -1.0, 1.0);  // VPIN is [0,1]
        double norm_micro = std::clamp(micro_vs_mid * 10000, -1.0, 1.0);
        
        return obi_weight * norm_obi 
             + trade_flow_weight * norm_tf 
             + vpin_weight * norm_vpin  
             + micro_weight * norm_micro;
    }
};
```

---

## 5. Cross-Asset Signals

### 5.1 Lead-Lag Relationships

Some assets lead others due to liquidity concentration or information flow:

```
Typical lead-lag chains:
  E-mini S&P futures → SPY ETF → Large-cap stocks
  EUR/USD futures → EUR/USD spot → European stocks
  Treasury futures → Interest rate ETFs
  Crude oil futures → Energy stocks
```

```cpp
struct LeadLagSignal {
    // Track price changes in leader instrument
    double leader_return = 0;
    double leader_return_ema = 0;
    double ema_decay = 0.95;
    
    void on_leader_update(double new_mid, double old_mid) {
        leader_return = (new_mid - old_mid) / old_mid;
        leader_return_ema = ema_decay * leader_return_ema + (1 - ema_decay) * leader_return;
    }
    
    // Signal for the follower: predict it will follow the leader
    double follower_signal() const {
        return leader_return_ema;  // Positive = expect follower to go up
    }
};
```

### 5.2 Index-Component Signals

```cpp
struct IndexComponentSignal {
    // If the index moves but a component doesn't, expect the component to follow
    double compute(double component_return, double index_return, double beta) {
        double expected_return = beta * index_return;
        double residual = component_return - expected_return;
        
        // Negative residual = component lagging the move → expect it to catch up
        return -residual;
    }
};
```

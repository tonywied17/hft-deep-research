# Order Book Mechanics & Market Microstructure

---

## 1. Matching Engine Architecture

### 1.1 Price-Time Priority (FIFO)

The dominant matching algorithm used by most exchanges:

```
Priority Rules:
  1. Price: Better price always matched first
  2. Time: At the same price, earliest order matched first
  
Buy side:  Higher price = better (willing to pay more)
Sell side: Lower price = better (willing to accept less)
```

**Match algorithm pseudocode:**
```cpp
void match_incoming_order(Order& incoming, OrderBook& book) {
    auto& opposite_side = (incoming.side == BUY) ? book.asks : book.bids;
    
    while (incoming.remaining_qty > 0 && !opposite_side.empty()) {
        auto& best = opposite_side.front();
        
        // Check if prices cross
        if (incoming.side == BUY && incoming.price < best.price) break;
        if (incoming.side == SELL && incoming.price > best.price) break;
        
        int fill_qty = std::min(incoming.remaining_qty, best.remaining_qty);
        
        // Execute at resting order's price (price improvement for aggressor)
        execute_trade(incoming, best, fill_qty, best.price);
        
        incoming.remaining_qty -= fill_qty;
        best.remaining_qty -= fill_qty;
        
        if (best.remaining_qty == 0) {
            opposite_side.pop_front();
        }
    }
    
    // Remaining quantity: post to book (limit) or cancel (market/IOC)
    if (incoming.remaining_qty > 0 && incoming.type == LIMIT) {
        book.add_order(incoming);
    }
}
```

### 1.2 Pro-Rata Matching (CME for some products)

Used for certain CME products (e.g., Eurodollar futures):

```
At the same price level:
  1. First, a "lead market maker" gets priority fill (if applicable)
  2. Then, remaining quantity allocated proportionally by order size
  3. Minimum allocation applies (prevents dust fills)

Formula:
  Fill_i = incoming_qty × (order_size_i / total_resting_qty)
  
  Rounded down, minimum allocation enforced
  Remainders distributed by time priority
```

### 1.3 Other Priority Types

| Exchange | Model | Notes |
|---|---|---|
| NASDAQ | Price-Time | Standard FIFO |
| NYSE | Price-Time + DMM | Designated Market Maker has priority |
| CME (ES) | Price-Time (FIFO) | Top-of-book, then pro-rata for some |
| CME (ED) | Pro-Rata | With lead market maker allocation |
| ICE | Price-Time | Standard FIFO |
| Cboe (Options) | Customer Priority | Retail gets priority over professional |

---

## 2. Order Types

### 2.1 Standard Order Types

| Type | Behavior | HFT Usage |
|---|---|---|
| **Limit** | Post at specified price | Primary — quotes and passive fills |
| **Market** | Execute immediately at best available | Rare — no price control |
| **IOC** (Immediate or Cancel) | Fill what you can, cancel rest | Aggressive takes, hedging |
| **FOK** (Fill or Kill) | Fill entirely or cancel | Large block execution |
| **GTC** (Good Till Cancel) | Persists until cancelled | Not used in HFT |
| **Day** | Valid until market close | Common for limit orders |

### 2.2 Advanced Order Types

| Type | Behavior | HFT Usage |
|---|---|---|
| **Peg** | Price follows NBBO | Auto-repricing without cancel/replace |
| **Midpoint Peg** | Post at NBBO midpoint | Price improvement |
| **Reserve/Iceberg** | Display only part of quantity | Hide true size |
| **ISO** (Intermarket Sweep) | Bypass trade-through protection | Multi-venue sweep |
| **Post Only** | Reject if would cross (take) | Ensure maker rebate |
| **Minimum Quantity** | Reject if less than min filled | Avoid small fills |

### 2.3 Post-Only Orders (Critical for HFT)

```
Post-Only (also called "Add Liquidity Only"):
  
  If the order would execute immediately (price crosses):
    Option 1: REJECT the order (don't execute)
    Option 2: SLIDE the price to just avoid crossing
    
  Why: Ensure you always receive maker rebate, never pay taker fee
  
  Revenue impact per 100 shares:
    Maker rebate: +$0.20 to +$0.35
    Taker fee:    -$0.25 to -$0.30
    Difference:   $0.45 to $0.65 per 100 shares
    
  At 10M shares/day: $4,500 - $6,500/day difference
```

---

## 3. Market Microstructure Concepts

### 3.1 National Best Bid and Offer (NBBO)

```
NBBO = Best bid across ALL exchanges, Best ask across ALL exchanges

Example:
  NYSE:    Bid 180.50 × 500   Ask 180.52 × 300
  NASDAQ:  Bid 180.51 × 200   Ask 180.53 × 400
  ARCA:    Bid 180.49 × 800   Ask 180.51 × 100
  
  NBBO = 180.51 (NASDAQ) × 180.51 (ARCA)
         Midpoint = 180.51

Regulation NMS requires:
  - No trade-throughs (can't execute at worse than NBBO)
  - Access fee cap ($0.003/share)
  - Sub-penny prohibition for stocks > $1
```

### 3.2 Maker-Taker Fee Model

```
Exchange Fee Schedule (typical):
  
  Maker (adds liquidity):   -$0.0020/share  (receives rebate)
  Taker (removes liquidity): $0.0030/share  (pays fee)
  
  "Inverted" exchanges (e.g., BX, EDGA):
  Maker:  $0.0005/share  (pays small fee)
  Taker: -$0.0015/share  (receives rebate)
  
  Strategy implications:
    - Route passive orders to highest-rebate venue
    - Route aggressive orders to lowest-fee or inverted venue
    - "Rebate arbitrage": post on high-rebate, take on inverted simultaneously
```

### 3.3 Tick Size & Price Granularity

```
Tick sizes (US equities):
  Stocks > $1.00:    $0.01 (1 cent)
  Stocks < $1.00:    $0.0001 (1/100th of a cent)
  
Tick sizes (Futures):
  E-mini S&P (ES):   0.25 index points = $12.50/contract
  E-mini NASDAQ (NQ): 0.25 index points = $5.00/contract
  Crude Oil (CL):    $0.01/barrel = $10.00/contract
  Treasury Notes:    1/64th of a point = $15.625/contract

Tick-to-revenue relationship:
  Wider ticks → more profit per trade (spread ≥ 1 tick)
  Narrower ticks → less profit but more opportunities
```

### 3.4 Latency & Queue Dynamics

```
Fill probability depends on:
  1. Queue position (earlier = higher probability)
  2. Queue depth (more ahead of you = lower probability)
  3. Expected volume at that price level
  4. Adverse selection (fills during toxic flow are costly)

Probability of fill ≈ E[Volume at level] / (Queue ahead + Order size)

Queue position decays when:
  - Cancel and resubmit (go to back of queue)
  - Exchange-enforced priority changes
  - Other orders cancel ahead of you (you move up)
```

---

## 4. Venue Analysis

### 4.1 US Equity Venues

| Venue | Type | Market Share | Median Latency | Notes |
|---|---|---|---|---|
| NYSE | Primary | ~22% | ~30 μs | DMM system, opening/closing auctions |
| NASDAQ | Primary | ~18% | ~15 μs | Multiple matching engines |
| NYSE Arca | ECN | ~10% | ~20 μs | ETF center |
| Cboe BZX | ECN | ~12% | ~10 μs | High rebates |
| Cboe EDGX | ECN | ~6% | ~10 μs | Midpoint orders |
| IEX | Exchange | ~4% | ~350 μs | Speed bump (350μs delay) |
| MEMX | Exchange | ~5% | ~10 μs | Low-fee structure |

### 4.2 Dark Pools

```
Dark pools: No pre-trade transparency (orders not displayed)

Advantages:
  - Midpoint execution (better price than lit markets)
  - Reduced market impact for large orders
  - Anonymous (no information leakage)

Disadvantages:
  - No guarantee of execution
  - Risk of gaming (HFTs may detect dark pool flow)
  - Less price discovery contribution
  
Major dark pools:
  - Crossfinder (Credit Suisse)
  - MS Pool (Morgan Stanley)  
  - Sigma X2 (Goldman Sachs)
  - Level ATS (LEVEL)
```

---

## 5. Auction Mechanics

### 5.1 Opening Auction (NASDAQ)

```
Pre-Market Phase (4:00 AM - 9:30 AM ET):
  - Orders accumulate
  - Indicative prices published
  
  Opening cross at 9:30:
  1. Calculate maximum executable volume price
  2. Break ties: minimum imbalance, then reference price
  3. Execute all crossing orders at single price
  4. Transition to continuous trading

Strategy:
  - Observe indicative cross price for opening direction signal
  - Place orders in cross (better fills, no adverse selection from HFTs)
  - Immediately trade after open based on imbalance information
```

### 5.2 Closing Auction (NYSE)

```
NYSE Market on Close (MOC) / Limit on Close (LOC):
  - MOC cutoff: 3:45 PM (can offset imbalance until 3:50 PM)
  - Published imbalance data starting 3:45 PM
  - D-Quote (designated market maker) can supplement liquidity
  
  Closing cross:
  - Single price at 4:00 PM
  - ~8-12% of daily NYSE volume in closing auction
  - Critical for index funds (NAV calculation)
```

---

## 6. Information Theoretic View

### 6.1 Price Impact Model (Kyle's Lambda)

$$\Delta P = \lambda \cdot Q + \epsilon$$

where:
- $\Delta P$ = price change
- $\lambda$ = Kyle's lambda (price impact coefficient)
- $Q$ = signed order flow (positive = net buy)

**Interpretation:** $\lambda$ measures how much each unit of flow moves the price. Higher $\lambda$ = less liquid / more information in flow.

### 6.2 Estimating Lambda

```cpp
struct KyleLambda {
    // Rolling regression of price change on signed volume
    double sum_qp = 0;    // Σ Q_i * ΔP_i
    double sum_qq = 0;    // Σ Q_i²
    int count = 0;
    
    void update(double signed_volume, double price_change) {
        sum_qp += signed_volume * price_change;
        sum_qq += signed_volume * signed_volume;
        count++;
    }
    
    double lambda() const {
        if (sum_qq < 1e-10) return 0;
        return sum_qp / sum_qq;  // OLS estimate
    }
    
    // Effective spread approximation
    double effective_spread() const {
        return 2 * lambda();
    }
};
```

### 6.3 Roll's Spread Estimator

Estimate the effective spread from trade prices alone:

$$S = 2\sqrt{-\text{Cov}(\Delta P_t, \Delta P_{t-1})}$$

This works because the bid-ask bounce creates negative serial correlation in price changes.

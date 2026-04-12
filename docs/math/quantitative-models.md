# Stochastic Calculus & Quantitative Models for HFT

---

## 1. Brownian Motion & Geometric Brownian Motion

### 1.1 Standard Brownian Motion (Wiener Process)

A stochastic process $W_t$ with the properties:
- $W_0 = 0$
- Increments $W_{t+s} - W_t \sim \mathcal{N}(0, s)$
- Independent increments
- Continuous paths (but nowhere differentiable)

### 1.2 Geometric Brownian Motion (GBM)

The standard model for stock prices:

$$dS_t = \mu S_t \, dt + \sigma S_t \, dW_t$$

**Solution:**
$$S_t = S_0 \exp\left[\left(\mu - \frac{\sigma^2}{2}\right)t + \sigma W_t\right]$$

```cpp
// GBM price path simulation
double simulate_gbm(double S0, double mu, double sigma, double dt) {
    // Standard normal via Box-Muller
    double u1 = uniform_random();
    double u2 = uniform_random();
    double z = std::sqrt(-2.0 * std::log(u1)) * std::cos(2.0 * M_PI * u2);
    
    return S0 * std::exp((mu - 0.5 * sigma * sigma) * dt + sigma * std::sqrt(dt) * z);
}
```

### 1.3 Why GBM Is Wrong for HFT

GBM assumptions fail at microsecond timescales:
- **Fat tails:** Real returns have kurtosis >> 3 (heavy tails)
- **Volatility clustering:** Periods of high/low vol cluster together
- **Mean reversion:** Microstructure effects cause short-term mean reversion
- **Discrete prices:** Prices move in ticks, not continuously
- **Jumps:** Order book events cause discontinuous price changes

---

## 2. Itô's Lemma

The chain rule for stochastic calculus. If $X_t$ follows an Itô process:

$$dX_t = \mu_t \, dt + \sigma_t \, dW_t$$

Then for a twice-differentiable function $f(X_t, t)$:

$$df = \left(\frac{\partial f}{\partial t} + \mu_t \frac{\partial f}{\partial x} + \frac{1}{2} \sigma_t^2 \frac{\partial^2 f}{\partial x^2}\right) dt + \sigma_t \frac{\partial f}{\partial x} \, dW_t$$

The extra $\frac{1}{2}\sigma^2 f''$ term is what distinguishes stochastic calculus from regular calculus.

### 2.1 Application: Deriving Black-Scholes PDE

Let stock follow GBM: $dS = \mu S\, dt + \sigma S \, dW$

Apply Itô to option value $V(S,t)$:

$$dV = \left(\frac{\partial V}{\partial t} + \mu S \frac{\partial V}{\partial S} + \frac{1}{2}\sigma^2 S^2 \frac{\partial^2 V}{\partial S^2}\right) dt + \sigma S \frac{\partial V}{\partial S} \, dW$$

Construct a delta-hedged portfolio $\Pi = V - \Delta S$ where $\Delta = \frac{\partial V}{\partial S}$:

$$d\Pi = dV - \Delta \, dS = \left(\frac{\partial V}{\partial t} + \frac{1}{2}\sigma^2 S^2 \frac{\partial^2 V}{\partial S^2}\right)dt$$

This is **riskless** (no $dW$ term), so must earn risk-free rate:

$$\frac{\partial V}{\partial t} + rS \frac{\partial V}{\partial S} + \frac{1}{2}\sigma^2 S^2 \frac{\partial^2 V}{\partial S^2} - rV = 0$$

---

## 3. Ornstein-Uhlenbeck Process

The fundamental model for mean-reverting processes in HFT:

$$dX_t = \theta(\mu - X_t) \, dt + \sigma \, dW_t$$

where:
- $\theta > 0$: Speed of mean reversion
- $\mu$: Long-run mean
- $\sigma$: Volatility of the process

### 3.1 Solution

$$X_t = \mu + (X_0 - \mu) e^{-\theta t} + \sigma \int_0^t e^{-\theta(t-s)} \, dW_s$$

**Conditional distribution:**
$$X_t | X_0 \sim \mathcal{N}\left(\mu + (X_0-\mu)e^{-\theta t}, \; \frac{\sigma^2}{2\theta}(1-e^{-2\theta t})\right)$$

**Stationary (long-run) distribution:**
$$X_\infty \sim \mathcal{N}\left(\mu, \; \frac{\sigma^2}{2\theta}\right)$$

### 3.2 Half-Life

The time for the expected value to move halfway back to the mean:

$$t_{1/2} = \frac{\ln 2}{\theta}$$

### 3.3 Parameter Estimation (Maximum Likelihood)

Given discrete observations $(x_0, x_1, \ldots, x_n)$ at equal intervals $\Delta t$:

```cpp
struct OUEstimator {
    double mu;     // Mean-reversion level
    double theta;  // Mean-reversion speed
    double sigma;  // Volatility
    
    void fit(const double* data, int n, double dt) {
        // Step 1: Run OLS regression: x_{t+1} = a + b * x_t + epsilon
        double sx = 0, sy = 0, sxx = 0, sxy = 0;
        for (int i = 0; i < n - 1; ++i) {
            double x = data[i];
            double y = data[i + 1];
            sx += x;
            sy += y;
            sxx += x * x;
            sxy += x * y;
        }
        int m = n - 1;
        double b = (m * sxy - sx * sy) / (m * sxx - sx * sx);
        double a = (sy - b * sx) / m;
        
        // Step 2: Convert AR(1) parameters to OU parameters
        theta = -std::log(b) / dt;
        mu = a / (1.0 - b);
        
        // Step 3: Estimate sigma from residual variance
        double sse = 0;
        for (int i = 0; i < n - 1; ++i) {
            double predicted = a + b * data[i];
            double resid = data[i + 1] - predicted;
            sse += resid * resid;
        }
        double residual_var = sse / (m - 2);
        sigma = std::sqrt(residual_var * 2 * theta / (1 - b * b));
    }
    
    double half_life() const { return std::log(2.0) / theta; }
};
```

### 3.4 Trading the OU Process

**Optimal entry/exit for a mean-reverting spread:**

If the spread follows OU with known parameters:
- **Enter** when the z-score exceeds the optimal entry threshold $b^*$
- **Exit** when the z-score returns to the optimal exit threshold $a^*$

The optimal thresholds can be found by solving a free-boundary problem. Practical approximation:

| Half-life | Entry z-score | Exit z-score | Stop z-score |
|---|---|---|---|
| 1–5 days | ±1.5σ | ±0.5σ | ±4.0σ |
| 5–15 days | ±2.0σ | ±0.5σ | ±4.0σ |
| 15–30 days | ±2.5σ | ±1.0σ | ±5.0σ |

---

## 4. Black-Scholes & Greeks

### 4.1 Black-Scholes Formulas

**European Call:**
$$C = S \Phi(d_1) - K e^{-rT} \Phi(d_2)$$

**European Put:**
$$P = K e^{-rT} \Phi(-d_2) - S \Phi(-d_1)$$

where:
$$d_1 = \frac{\ln(S/K) + (r + \sigma^2/2)T}{\sigma \sqrt{T}}, \quad d_2 = d_1 - \sigma \sqrt{T}$$

```cpp
struct BlackScholes {
    static double call(double S, double K, double T, double r, double sigma) {
        double d1 = (std::log(S / K) + (r + 0.5 * sigma * sigma) * T) 
                     / (sigma * std::sqrt(T));
        double d2 = d1 - sigma * std::sqrt(T);
        return S * norm_cdf(d1) - K * std::exp(-r * T) * norm_cdf(d2);
    }
    
    static double put(double S, double K, double T, double r, double sigma) {
        double d1 = (std::log(S / K) + (r + 0.5 * sigma * sigma) * T)
                     / (sigma * std::sqrt(T));
        double d2 = d1 - sigma * std::sqrt(T);
        return K * std::exp(-r * T) * norm_cdf(-d2) - S * norm_cdf(-d1);
    }
    
    // Standard normal CDF (Abramowitz-Stegun approximation)
    static double norm_cdf(double x) {
        const double a1 = 0.254829592, a2 = -0.284496736, a3 = 1.421413741;
        const double a4 = -1.453152027, a5 = 1.061405429, p = 0.3275911;
        
        int sign = (x < 0) ? -1 : 1;
        x = std::abs(x) / std::sqrt(2.0);
        double t = 1.0 / (1.0 + p * x);
        double y = 1.0 - (((((a5*t + a4)*t) + a3)*t + a2)*t + a1)*t * std::exp(-x*x);
        return 0.5 * (1.0 + sign * y);
    }
    
    static double norm_pdf(double x) {
        return std::exp(-0.5 * x * x) / std::sqrt(2.0 * M_PI);
    }
};
```

### 4.2 The Greeks

| Greek | Formula | Interpretation | HFT Use |
|---|---|---|---|
| **Delta** ($\Delta$) | $\Phi(d_1)$ (call) | Directional exposure | Delta hedging |
| **Gamma** ($\Gamma$) | $\frac{\phi(d_1)}{S\sigma\sqrt{T}}$ | Delta sensitivity | Gamma scalping |
| **Theta** ($\Theta$) | $-\frac{S\sigma\phi(d_1)}{2\sqrt{T}} - rKe^{-rT}\Phi(d_2)$ | Time decay | Theta capture |
| **Vega** ($\nu$) | $S\sqrt{T}\phi(d_1)$ | Vol sensitivity | Vol trading |
| **Rho** ($\rho$) | $KTe^{-rT}\Phi(d_2)$ | Rate sensitivity | Usually small |

```cpp
struct Greeks {
    double delta, gamma, theta, vega, rho;
    
    static Greeks compute_call(double S, double K, double T, double r, double sigma) {
        double sqrt_T = std::sqrt(T);
        double d1 = (std::log(S / K) + (r + 0.5 * sigma * sigma) * T) / (sigma * sqrt_T);
        double d2 = d1 - sigma * sqrt_T;
        double pdf_d1 = BlackScholes::norm_pdf(d1);
        double cdf_d1 = BlackScholes::norm_cdf(d1);
        double cdf_d2 = BlackScholes::norm_cdf(d2);
        double disc = std::exp(-r * T);
        
        Greeks g;
        g.delta = cdf_d1;
        g.gamma = pdf_d1 / (S * sigma * sqrt_T);
        g.theta = -(S * sigma * pdf_d1) / (2 * sqrt_T) - r * K * disc * cdf_d2;
        g.vega = S * sqrt_T * pdf_d1;
        g.rho = K * T * disc * cdf_d2;
        return g;
    }
};
```

### 4.3 Implied Volatility (Newton-Raphson)

Given an observed option price, find the volatility that makes Black-Scholes match:

```cpp
double implied_vol(double market_price, double S, double K, double T, double r,
                   bool is_call, double initial_guess = 0.2) {
    double sigma = initial_guess;
    
    for (int i = 0; i < 100; ++i) {
        double price = is_call ? BlackScholes::call(S, K, T, r, sigma)
                               : BlackScholes::put(S, K, T, r, sigma);
        
        double vega = Greeks::compute_call(S, K, T, r, sigma).vega;
        if (std::abs(vega) < 1e-12) break;
        
        double diff = price - market_price;
        if (std::abs(diff) < 1e-8) break;
        
        sigma -= diff / vega;
        sigma = std::max(sigma, 0.001);
    }
    
    return sigma;
}
```

---

## 5. Volatility Models

### 5.1 Realized Volatility

**Standard estimator (close-to-close):**
$$\hat{\sigma}^2 = \frac{1}{n-1}\sum_{i=1}^n (r_i - \bar{r})^2$$

**Yang-Zhang estimator (open, high, low, close):**
Better efficiency, handles overnight jumps:

$$\hat{\sigma}_{YZ}^2 = \hat{\sigma}_o^2 + k\hat{\sigma}_c^2 + (1-k)\hat{\sigma}_{RS}^2$$

where $\hat{\sigma}_o^2$ = overnight variance, $\hat{\sigma}_c^2$ = close-to-close, $\hat{\sigma}_{RS}^2$ = Rogers-Satchell.

### 5.2 EWMA Volatility

Exponentially weighted — recent observations matter more:

$$\hat{\sigma}_t^2 = \lambda \hat{\sigma}_{t-1}^2 + (1-\lambda) r_t^2$$

```cpp
struct EWMAVol {
    double lambda = 0.94;  // RiskMetrics default
    double variance = 0;
    
    void update(double return_val) {
        variance = lambda * variance + (1.0 - lambda) * return_val * return_val;
    }
    
    double vol() const { return std::sqrt(variance); }
};
```

### 5.3 GARCH(1,1)

$$\sigma_t^2 = \omega + \alpha r_{t-1}^2 + \beta \sigma_{t-1}^2$$

Constraints: $\omega > 0$, $\alpha \geq 0$, $\beta \geq 0$, $\alpha + \beta < 1$

Long-run variance: $\sigma_\infty^2 = \omega / (1 - \alpha - \beta)$

```cpp
struct GARCH11 {
    double omega = 1e-6;
    double alpha = 0.05;
    double beta = 0.94;
    double variance;
    
    void update(double return_val) {
        variance = omega + alpha * return_val * return_val + beta * variance;
    }
    
    double long_run_vol() const {
        return std::sqrt(omega / (1.0 - alpha - beta));
    }
};
```

---

## 6. Cointegration

### 6.1 Engle-Granger Two-Step Method

```
Step 1: Estimate the cointegrating regression
  Y_t = α + β X_t + ε_t    (OLS)

Step 2: Test residuals for stationarity (ADF test)
  Δε_t = ρ ε_{t-1} + Σ γ_i Δε_{t-i} + u_t
  
  H₀: ρ = 0 (no cointegration)
  H₁: ρ < 0 (cointegrated)
  
  Use Engle-Granger critical values (NOT standard ADF tables)
```

### 6.2 Johansen Test (Multiple Variables)

For a system of $k$ variables, tests for $r$ cointegrating relationships:

```
VAR(p): ΔY_t = Π Y_{t-1} + Σ Γ_i ΔY_{t-i} + ε_t

Rank of Π determines number of cointegrating vectors:
  rank(Π) = 0 → No cointegration
  rank(Π) = r → r cointegrating relationships
  rank(Π) = k → All variables are stationary
```

### 6.3 ADF Test Implementation

```cpp
struct ADFTest {
    // Returns the ADF t-statistic
    // More negative = stronger evidence of stationarity
    static double compute(const double* data, int n, int lag_order) {
        // ΔY_t = ρ Y_{t-1} + Σ γ_i ΔY_{t-i} + ε_t
        
        int T = n - lag_order - 1;
        
        // Build regression matrix (simplified for lag_order = 1)
        double sy = 0, syx = 0, sx = 0, sxx = 0;
        
        for (int t = lag_order + 1; t < n; ++t) {
            double dy = data[t] - data[t-1];         // ΔY_t
            double y_lag = data[t-1];                  // Y_{t-1}
            
            sy += dy;
            sx += y_lag;
            sxy += dy * y_lag;
            sxx += y_lag * y_lag;
        }
        
        // OLS estimate of ρ
        double rho_hat = (T * sxy - sx * sy) / (T * sxx - sx * sx);
        
        // Standard error of ρ
        double sse = 0;
        for (int t = lag_order + 1; t < n; ++t) {
            double predicted = (sy / T) + rho_hat * (data[t-1] - sx / T);
            double resid = (data[t] - data[t-1]) - predicted;
            sse += resid * resid;
        }
        double se_rho = std::sqrt(sse / (T - 2) / (sxx - sx * sx / T));
        
        return rho_hat / se_rho;  // ADF t-statistic
    }
    
    // Critical values (Engle-Granger, 2 variables, constant, 5% level)
    static bool is_cointegrated_5pct(double t_stat) {
        return t_stat < -3.37;  // Approximate critical value
    }
};
```

---

## 7. Kalman Filter

### 7.1 General Linear Model

**State equation:** $x_{t} = A x_{t-1} + B u_t + w_t$, where $w_t \sim \mathcal{N}(0, Q)$

**Observation equation:** $z_t = H x_t + v_t$, where $v_t \sim \mathcal{N}(0, R)$

### 7.2 Algorithm

```
Predict:
  x̂_{t|t-1} = A x̂_{t-1|t-1}
  P_{t|t-1} = A P_{t-1|t-1} A' + Q

Update:
  Innovation:    ỹ_t = z_t - H x̂_{t|t-1}
  Innov. cov:    S_t = H P_{t|t-1} H' + R
  Kalman gain:   K_t = P_{t|t-1} H' S_t^{-1}
  State update:  x̂_{t|t} = x̂_{t|t-1} + K_t ỹ_t
  Cov update:    P_{t|t} = (I - K_t H) P_{t|t-1}
```

### 7.3 Kalman Filter for Price Estimation

```cpp
struct KalmanPriceFilter {
    // State: [price, velocity]  (price and its rate of change)
    double x[2] = {0, 0};     // State estimate
    double P[2][2] = {{100, 0}, {0, 1}};  // Covariance
    
    double Q_price = 0.01;    // Process noise (price)
    double Q_vel = 0.001;     // Process noise (velocity)
    double R = 0.1;           // Observation noise
    double dt = 1.0;          // Time step
    
    double update(double observed_price) {
        // Predict
        double x_pred[2] = {x[0] + dt * x[1], x[1]};
        double P_pred[2][2] = {
            {P[0][0] + dt * P[1][0] + dt * P[0][1] + dt * dt * P[1][1] + Q_price,
             P[0][1] + dt * P[1][1]},
            {P[1][0] + dt * P[1][1],
             P[1][1] + Q_vel}
        };
        
        // Update
        double y = observed_price - x_pred[0];  // Innovation
        double S = P_pred[0][0] + R;            // Innovation covariance
        double K[2] = {P_pred[0][0] / S, P_pred[1][0] / S};  // Kalman gain
        
        x[0] = x_pred[0] + K[0] * y;
        x[1] = x_pred[1] + K[1] * y;
        
        P[0][0] = (1 - K[0]) * P_pred[0][0];
        P[0][1] = (1 - K[0]) * P_pred[0][1];
        P[1][0] = -K[1] * P_pred[0][0] + P_pred[1][0];
        P[1][1] = -K[1] * P_pred[0][1] + P_pred[1][1];
        
        return x[0];  // Filtered price estimate
    }
    
    double predicted_price(double steps_ahead) const {
        return x[0] + steps_ahead * dt * x[1];
    }
};
```

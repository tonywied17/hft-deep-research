# HFT Research Prompts

## Market Data & Protocol Research

### Prompt: ITCH Protocol Deep Dive
```
Research NASDAQ ITCH 5.0 protocol in depth. Cover:
- Complete message type catalog with binary layouts (byte offsets, sizes, endianness)
- MoldUDP64 transport framing (session management, sequence numbering, gap recovery)
- SoupBinTCP for gap recovery (request/response format)
- Order book reconstruction from ITCH messages (Add, Execute, Cancel, Delete, Replace)
- Cross trade and auction messages
- Performance optimization: SIMD parsing, zero-copy techniques, struct packing
- Provide production C++ parsing code for each message type
```

### Prompt: CME MDP 3.0 Analysis
```
Research CME Market Data Platform 3.0. Cover:
- SBE (Simple Binary Encoding) message format
- Incremental refresh message templates (book, trade, statistics)
- Instrument definition and security definition messages
- Recovery process: TCP snapshot + incremental replay, RptSeq tracking
- Channel architecture and multicast group assignment
- Match event indicator batching semantics
- Implied quote handling (direct vs implied book levels)
- Provide C++ parsing code for incremental book refresh
```

### Prompt: FIX Protocol Optimization
```
Research FIX protocol optimization for HFT. Cover:
- FIX 4.2 vs 5.0 SP2 differences relevant to performance
- Session layer: logon, heartbeat, resend request, sequence management
- Zero-allocation parsing techniques (tag dispatch table, in-place value extraction)
- SIMD-accelerated SOH delimiter scanning
- Pre-computed message templates for common order types
- Incremental checksum computation
- Scatter-gather I/O (writev) for multi-buffer sends
- Benchmarks: parsing latency per message type
```

---

## Architecture & Systems Research

### Prompt: Event Loop Architecture
```
Research optimal event loop designs for HFT systems. Cover:
- Busy-poll vs epoll vs io_uring tradeoffs
- Single-core complete loop (all stages on one core)
- Pipeline architecture (separate cores per stage)
- Timer management without system calls (TSC-based timer wheels)
- Multi-feed aggregation patterns
- Jitter sources and measurement techniques
- Warm-up procedures (cache warming, JIT compilation avoidance)
- Provide benchmarks for each architecture variant
```

### Prompt: Inter-Process Communication
```
Research IPC mechanisms for ultra-low-latency HFT. Cover:
- SPSC ring buffer with cached positions (production implementation)
- Seqlock-based state publishing
- Shared memory via hugepage-backed mmap
- Message encoding (flat binary vs FlatBuffers vs Cap'n Proto vs SBE)
- Fan-out patterns (single writer, multiple readers)
- Backpressure mechanisms and ring sizing calculations
- DPDK rte_ring vs custom implementations
- Benchmark: message passing latency (single-producer, single-consumer)
```

---

## C++ Performance Research

### Prompt: Cache Optimization Techniques
```
Research CPU cache optimization for HFT C++ code. Cover:
- Cache hierarchy (L1/L2/L3) sizes and latencies for Intel Xeon and AMD EPYC
- False sharing detection (perf c2c) and prevention (alignas, padding)
- Structure of Arrays (SoA) vs Array of Structures (AoS) with benchmarks
- Hot/cold data splitting techniques
- Software prefetching (__builtin_prefetch) and when it helps vs hurts
- Cache-friendly order book data structures
- Instruction cache optimization (function ordering, __attribute__((hot/cold)))
- Hugepages and TLB optimization
- Provide benchmarking code using perf hardware counters
```

### Prompt: SIMD for Financial Computing
```
Research SIMD (SSE/AVX/AVX-512) applications in HFT. Cover:
- x86 SIMD extension landscape and instruction sets
- FIX message parsing with AVX2 (delimiter search, tag extraction)
- ITCH message classification and routing
- Batch price comparison and threshold checking
- VWAP calculation with FMA instructions
- Risk check vectorization (multiple limits in parallel)
- String comparison and symbol matching
- Auto-vectorization hints and compiler support
- Runtime SIMD dispatch (detect and use best available)
- Provide benchmarks: scalar vs SSE vs AVX2 vs AVX-512
```

---

## Quantitative Finance Research

### Prompt: Ornstein-Uhlenbeck Process for Trading
```
Research the Ornstein-Uhlenbeck process applied to HFT spread trading. Cover:
- Mathematical definition and solution of the OU SDE
- Parameter estimation (MLE from discrete observations)
- Half-life calculation and interpretation
- Optimal entry/exit thresholds (free boundary problem)
- Connection to AR(1) process in discrete time
- Multi-dimensional OU for basket trading
- Regime detection (parameter instability, structural breaks)
- Provide C++ implementation: estimator, simulator, and signal generator
```

### Prompt: Volatility Modeling
```
Research volatility estimation and modeling for HFT. Cover:
- Realized volatility estimators (close-to-close, Parkinson, Garman-Klass, Yang-Zhang)
- EWMA (exponentially weighted moving average) volatility
- GARCH(1,1) model and parameter estimation
- Intraday volatility patterns (U-shaped, event-driven)
- Microstructure noise and its impact on vol estimation
- Kernel-based realized volatility (noise-robust)
- Implied volatility surface construction
- Forward volatility and term structure
- Provide C++ implementations of each estimator
```

---

## Strategy Research

### Prompt: Avellaneda-Stoikov Market Making
```
Research the Avellaneda-Stoikov market making model in depth. Cover:
- Full mathematical derivation (HJB equation, value function, optimal controls)
- Reservation price formula and intuition
- Optimal spread formula and decomposition
- Parameter calibration from market data (gamma, kappa, sigma)
- Practical modifications for real markets (discrete ticks, multiple levels)
- Inventory skewing strategies
- Queue position management and requoting logic
- Extensions: multi-asset, time-varying parameters, adverse selection
- Provide production C++ implementation with all practical adjustments
```

### Prompt: Statistical Arbitrage
```
Research statistical arbitrage strategies for HFT. Cover:
- Pairs trading with cointegration (Engle-Granger, Johansen)
- Spread construction and hedge ratio estimation
- Kalman filter for dynamic hedge ratios
- OU-based optimal entry/exit thresholds
- PCA-based multi-asset arbitrage (eigenportfolios)
- ETF arbitrage (NAV tracking, creation/redemption)
- Risk management for stat arb (regime breaks, correlation breakdown)
- Provide C++ implementations: cointegration test, Kalman filter, trading signals
```

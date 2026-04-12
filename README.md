# HFT — High-Frequency Trading System

> A complete knowledge base, implementation guide, and business plan for building an HFT operation from scratch.

---

## Quick Start — Read Order

If you're new to this repo, follow this path:

### 1. Understand the Business (Day 1)
- **[INVESTMENT_PLAN.md](INVESTMENT_PLAN.md)** — Leveled roadmap from $0 to profitable. Costs, projected returns, milestones. Read this first to understand where you're going.
- **[BOOTSTRAP.md](BOOTSTRAP.md)** — Free-first build order. What to build, what to read, which prompts to run, and when — organized cheapest-first.

### 2. Learn the Markets (Week 1)
- [docs/market-data/order-book-mechanics.md](docs/market-data/order-book-mechanics.md) — How exchanges work: matching engines, order types, NBBO, maker-taker fees, venue landscape
- [docs/market-data/exchange-protocols.md](docs/market-data/exchange-protocols.md) — FIX, ITCH, CME MDP 3.0 protocol formats with parsing code

### 3. Learn the Architecture (Week 2)
- [docs/architecture/system-design.md](docs/architecture/system-design.md) — End-to-end HFT system topology, component breakdown
- [docs/architecture/event-loop-patterns.md](docs/architecture/event-loop-patterns.md) — Busy-poll loops, pipeline design, timer management
- [docs/architecture/message-bus.md](docs/architecture/message-bus.md) — SPSC ring buffers, shared memory IPC, message encoding

### 4. Learn C++ Performance (Week 3-4)
- [docs/cpp-performance/cache-optimization.md](docs/cpp-performance/cache-optimization.md) — Cache hierarchy, false sharing, SoA/AoS, prefetching
- [docs/cpp-performance/lock-free-structures.md](docs/cpp-performance/lock-free-structures.md) — Memory ordering, SPSC/MPSC queues, seqlocks
- [docs/cpp-performance/simd-vectorization.md](docs/cpp-performance/simd-vectorization.md) — AVX2/AVX-512 for protocol parsing, batch operations
- [docs/cpp-performance/compiler-and-memory.md](docs/cpp-performance/compiler-and-memory.md) — Compiler flags, PGO, arena allocators, fixed-point math

### 5. Learn Networking (Week 5)
- [docs/networking/kernel-bypass.md](docs/networking/kernel-bypass.md) — DPDK, Solarflare ef_vi, OpenOnload
- [docs/networking/hardware-and-switches.md](docs/networking/hardware-and-switches.md) — NIC architecture, hardware timestamps, switches, cabling

### 6. Learn the Math (Week 6)
- [docs/math/quantitative-models.md](docs/math/quantitative-models.md) — Brownian motion, Itô's lemma, OU process, Black-Scholes, Greeks, volatility models, Kalman filter, cointegration

### 7. Learn the Strategies (Week 7-8)
- [docs/algorithms/market-making.md](docs/algorithms/market-making.md) — Avellaneda-Stoikov, inventory skewing, queue management, multi-level quoting
- [docs/algorithms/strategies-and-signals.md](docs/algorithms/strategies-and-signals.md) — Pairs trading, ETF arb, order book imbalance, VPIN, lead-lag signals
- [docs/algorithms/order-book-algorithms.md](docs/algorithms/order-book-algorithms.md) — L2/L3 book structures, ITCH reconstruction, performance benchmarks

### 8. Learn Risk & Operations (Week 9)
- [docs/risk-management/pre-trade-and-risk.md](docs/risk-management/pre-trade-and-risk.md) — Pre-trade checks, kill switch, position management, VaR, circuit breakers, regulations
- [docs/backtesting/framework-and-metrics.md](docs/backtesting/framework-and-metrics.md) — Backtest engine design, fill simulation, Sharpe/Sortino, pitfalls

### 9. Plan Deployment (Week 10)
- [docs/deployment/operations-and-infrastructure.md](docs/deployment/operations-and-infrastructure.md) — Colocation, OS tuning, PTP time sync, FPGA, monitoring
- [hardware/hardware-reference.md](hardware/hardware-reference.md) — NICs, switches, servers, FPGAs, cables — names, costs, links, cost efficiency analysis

### 10. Start Building (Week 11+)
- [prompts/phases/development-phases.md](prompts/phases/development-phases.md) — 10-phase development plan with concrete tasks and deliverables
- [prompts/phases/research-prompts.md](prompts/phases/research-prompts.md) — Deep-research prompt templates for each domain

---

## Repository Structure

```
HFT/
│
├── README.md                          ← You are here
├── INVESTMENT_PLAN.md                 ← Business plan & investor pitch (Level 0-5)
├── BOOTSTRAP.md                       ← Free-first dev sandbox & build order
│
├── docs/                              ← Deep technical documentation
│   ├── architecture/                  ← System design & patterns
│   │   ├── system-design.md           ← Full system topology & components
│   │   ├── event-loop-patterns.md     ← Busy-poll, pipeline, timers
│   │   └── message-bus.md             ← SPSC rings, seqlocks, shared memory
│   │
│   ├── cpp-performance/               ← C++ optimization techniques
│   │   ├── cache-optimization.md      ← Cache hierarchy, false sharing, NUMA
│   │   ├── lock-free-structures.md    ← Memory ordering, SPSC, MPSC, seqlocks
│   │   ├── simd-vectorization.md      ← SSE/AVX for parsing & computation
│   │   └── compiler-and-memory.md     ← Flags, PGO, allocators, fixed-point
│   │
│   ├── networking/                    ← Network layer
│   │   ├── kernel-bypass.md           ← DPDK, ef_vi, OpenOnload
│   │   └── hardware-and-switches.md   ← NICs, switches, cabling, microwave
│   │
│   ├── algorithms/                    ← Trading strategies & signals
│   │   ├── market-making.md           ← Avellaneda-Stoikov, inventory management
│   │   ├── strategies-and-signals.md  ← Stat arb, pairs, OBI, VPIN, lead-lag
│   │   └── order-book-algorithms.md   ← Book data structures, ITCH reconstruction
│   │
│   ├── math/                          ← Quantitative foundations
│   │   └── quantitative-models.md     ← Stochastic calc, BSM, OU, Kalman, GARCH
│   │
│   ├── market-data/                   ← Exchange protocols & microstructure
│   │   ├── exchange-protocols.md      ← FIX, ITCH, CME MDP, OUCH, PITCH
│   │   └── order-book-mechanics.md    ← Matching engines, order types, fees
│   │
│   ├── risk-management/               ← Risk controls & compliance
│   │   └── pre-trade-and-risk.md      ← Hot-path checks, kill switch, VaR, regs
│   │
│   ├── backtesting/                   ← Strategy validation
│   │   └── framework-and-metrics.md   ← Engine design, fill sim, metrics, pitfalls
│   │
│   └── deployment/                    ← Production operations
│       └── operations-and-infrastructure.md  ← Colo, OS tuning, PTP, FPGA, monitoring
│
├── hardware/                          ← Hardware reference & procurement
│   └── hardware-reference.md          ← NICs, switches, servers, FPGAs, budget tiers
│
├── prompts/                           ← Development guides
│   └── phases/
│       ├── development-phases.md      ← 10-phase implementation roadmap
│       └── research-prompts.md        ← Deep-research prompt templates
│
└── reference/                         ← Quick reference materials
    └── glossary-and-reading.md        ← Glossary, recommended books & papers
```

---

## For Co-Investors

Start with **[INVESTMENT_PLAN.md](INVESTMENT_PLAN.md)** for the business case, then **[BOOTSTRAP.md](BOOTSTRAP.md)** for the technical build path. The investment plan covers:
- 6-level growth plan (Level 0-5) with costs at each stage
- Projected revenue ranges with probability-weighted scenarios
- Hardware and operational cost breakdowns
- Key performance metrics to track
- Risk analysis

---

## For Engineers Joining the Team

1. Read the **Quick Start** section above in order
2. Open **[BOOTSTRAP.md](BOOTSTRAP.md)** — it tells you exactly what to build, read, and which prompt to run at each step
3. Set up your dev environment following [docs/deployment/operations-and-infrastructure.md](docs/deployment/operations-and-infrastructure.md) (Section 2: OS Tuning)
4. Start building following [prompts/phases/development-phases.md](prompts/phases/development-phases.md) Phase 1
5. Use [prompts/phases/research-prompts.md](prompts/phases/research-prompts.md) when you need to go deeper on any topic

---

## Content Depth

This isn't a surface-level overview. Every document contains:
- **Production C++ code** — not pseudocode, real implementations you can compile
- **Mathematical formulas** — full derivations, not just results
- **Architecture diagrams** — ASCII diagrams showing data flow and layout
- **Comparison tables** — benchmarks, cost analysis, tradeoff matrices
- **Configuration commands** — actual bash/sysctl commands, not "configure your system"
- **Performance numbers** — nanosecond-level latency data from real hardware

Total documentation: ~20 files, ~8,000+ lines of focused technical content.

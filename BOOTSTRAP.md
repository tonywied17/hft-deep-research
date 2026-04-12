# HFT Bootstrap — Free-First Dev Sandbox & Build Order

> Start building today with $0. This guide orders everything by cost so you always know what to tackle next, what to spend first, and when to use each prompt/doc.

---

## How to Use This File

Each tier below lists **what to build**, **which docs to read**, **which prompts to run**, and **what it costs**. Work top-to-bottom — every tier unlocks the next. Prompt files are meant to be fed to an AI coding assistant to generate deep research or production code for that step.

---

## Tier 0 — Completely Free ($0)

> *Everything here runs on whatever machine you already own.*

### 0.1 — Environment Setup (Day 1)

| Do | Read | Prompt |
|---|---|---|
| Install Linux (bare metal or WSL2) | [deployment/operations-and-infrastructure.md §2](docs/deployment/operations-and-infrastructure.md) — OS Tuning | — |
| Set up C++20 toolchain (GCC 12+ or Clang 15+) | [cpp-performance/compiler-and-memory.md §1](docs/cpp-performance/compiler-and-memory.md) — Compiler Flags | — |
| Configure hugepages, isolcpus (even on dev box) | [deployment/operations-and-infrastructure.md §2](docs/deployment/operations-and-infrastructure.md) | — |
| Install perf, VTune (free), CMake, Ninja | — | — |

**Cost: $0**

### 0.2 — Learn the Domain (Week 1)

| Do | Read | Prompt |
|---|---|---|
| Understand exchanges, matching engines, order types | [market-data/order-book-mechanics.md](docs/market-data/order-book-mechanics.md) | — |
| Understand protocols (FIX, ITCH, CME MDP) | [market-data/exchange-protocols.md](docs/market-data/exchange-protocols.md) | [research-prompts.md](prompts/phases/research-prompts.md) → "ITCH Protocol Deep Dive" |
| Understand market microstructure, fees, NBBO | [market-data/order-book-mechanics.md](docs/market-data/order-book-mechanics.md) | [research-prompts.md](prompts/phases/research-prompts.md) → "FIX Protocol Optimization" |
| Read glossary for unfamiliar terms | [reference/glossary-and-reading.md §Glossary](reference/glossary-and-reading.md) | — |

**Cost: $0**

### 0.3 — Build ITCH Parser (Week 2)

| Do | Read | Prompt |
|---|---|---|
| Implement zero-allocation ITCH 5.0 parser | [market-data/exchange-protocols.md §ITCH](docs/market-data/exchange-protocols.md) | [research-prompts.md](prompts/phases/research-prompts.md) → "ITCH Protocol Deep Dive" |
| Implement MoldUDP64 transport layer | [market-data/exchange-protocols.md](docs/market-data/exchange-protocols.md) | — |
| Parse sample data (free ITCH sample files from NASDAQ) | — | — |
| Benchmark: target >5M messages/sec | [cpp-performance/cache-optimization.md](docs/cpp-performance/cache-optimization.md) | — |

> **Dev Phases reference:** [development-phases.md Phase 1](prompts/phases/development-phases.md) tasks 2-4

**Cost: $0**

### 0.4 — Build Order Book (Week 3)

| Do | Read | Prompt |
|---|---|---|
| Build L2 order book (sorted array) | [algorithms/order-book-algorithms.md](docs/algorithms/order-book-algorithms.md) | — |
| Build L3 order book (hash map + linked list per level) | [algorithms/order-book-algorithms.md](docs/algorithms/order-book-algorithms.md) | — |
| Compute micro-price, BBO, spread signals | [math/quantitative-models.md](docs/math/quantitative-models.md) | — |
| Validate against known replay outputs | — | — |

> **Dev Phases reference:** [development-phases.md Phase 1](prompts/phases/development-phases.md) tasks 5-7

**Cost: $0**

### 0.5 — Build SPSC Ring Buffer & IPC (Week 4)

| Do | Read | Prompt |
|---|---|---|
| Implement lock-free SPSC ring buffer | [architecture/message-bus.md](docs/architecture/message-bus.md) | [research-prompts.md](prompts/phases/research-prompts.md) → "Inter-Process Communication" |
| Build shared memory IPC (hugepage-backed mmap) | [architecture/message-bus.md](docs/architecture/message-bus.md) | — |
| Design flat binary message format (32-byte aligned) | [architecture/message-bus.md](docs/architecture/message-bus.md) | — |
| Implement seqlock for state publishing | [cpp-performance/lock-free-structures.md](docs/cpp-performance/lock-free-structures.md) | — |

> **Dev Phases reference:** [development-phases.md Phase 3](prompts/phases/development-phases.md) tasks 1-3, 7

**Cost: $0**

### 0.6 — Build Event Loop & Pipeline (Week 5)

| Do | Read | Prompt |
|---|---|---|
| Implement busy-poll event loop (zero-syscall) | [architecture/event-loop-patterns.md](docs/architecture/event-loop-patterns.md) | [research-prompts.md](prompts/phases/research-prompts.md) → "Event Loop Architecture" |
| Connect: Parser → Book → Strategy → Order Manager | [architecture/system-design.md](docs/architecture/system-design.md) | — |
| Add TSC-based timer wheel | [architecture/event-loop-patterns.md](docs/architecture/event-loop-patterns.md) | — |
| Measure end-to-end latency (TSC timestamps) | [cpp-performance/compiler-and-memory.md §TSC](docs/cpp-performance/compiler-and-memory.md) | — |

> **Dev Phases reference:** [development-phases.md Phase 3](prompts/phases/development-phases.md) tasks 4-6, 8

**Cost: $0**

### 0.7 — Build Backtesting Engine (Week 6-7)

| Do | Read | Prompt |
|---|---|---|
| Event-driven backtest engine | [backtesting/framework-and-metrics.md](docs/backtesting/framework-and-metrics.md) | — |
| Fill simulation model (queue position, latency) | [backtesting/framework-and-metrics.md](docs/backtesting/framework-and-metrics.md) | — |
| Performance metrics (Sharpe, Sortino, max DD) | [backtesting/framework-and-metrics.md](docs/backtesting/framework-and-metrics.md) | — |
| Replay historical ITCH data through full pipeline | — | — |

> **Dev Phases reference:** [development-phases.md Phase 5](prompts/phases/development-phases.md)

**Cost: $0** (using free ITCH sample data)

### 0.8 — Build Market Making Strategy (Week 7-8)

| Do | Read | Prompt |
|---|---|---|
| Implement Avellaneda-Stoikov optimal quoting | [algorithms/market-making.md](docs/algorithms/market-making.md) | [research-prompts.md](prompts/phases/research-prompts.md) → "Avellaneda-Stoikov Market Making" |
| Add inventory skewing | [algorithms/market-making.md](docs/algorithms/market-making.md) | — |
| Add volatility-adaptive spread | [math/quantitative-models.md §GARCH](docs/math/quantitative-models.md) | [research-prompts.md](prompts/phases/research-prompts.md) → "Volatility Modeling" |
| Add VPIN toxicity detection | [algorithms/strategies-and-signals.md §VPIN](docs/algorithms/strategies-and-signals.md) | — |

> **Dev Phases reference:** [development-phases.md Phase 4](prompts/phases/development-phases.md)

**Cost: $0**

### 0.9 — Build Stat Arb Signals (Week 8-9)

| Do | Read | Prompt |
|---|---|---|
| Kalman filter for dynamic hedge ratios | [math/quantitative-models.md §Kalman](docs/math/quantitative-models.md) | [research-prompts.md](prompts/phases/research-prompts.md) → "Statistical Arbitrage" |
| Cointegration testing (ADF, Johansen) | [algorithms/strategies-and-signals.md §Pairs](docs/algorithms/strategies-and-signals.md) | — |
| Pairs trading with OU-based entry/exit | [math/quantitative-models.md §OU](docs/math/quantitative-models.md) | [research-prompts.md](prompts/phases/research-prompts.md) → "Ornstein-Uhlenbeck Process" |
| Order book imbalance + lead-lag signals | [algorithms/strategies-and-signals.md §OBI](docs/algorithms/strategies-and-signals.md) | — |

> **Dev Phases reference:** [development-phases.md Phase 6](prompts/phases/development-phases.md)

**Cost: $0**

### 0.10 — Build Risk Engine & Kill Switch (Week 9-10)

| Do | Read | Prompt |
|---|---|---|
| Pre-trade risk checks (< 50 ns hot path) | [risk-management/pre-trade-and-risk.md](docs/risk-management/pre-trade-and-risk.md) | — |
| Position tracker & real-time PnL | [risk-management/pre-trade-and-risk.md](docs/risk-management/pre-trade-and-risk.md) | — |
| Kill switch (auto + manual trigger) | [risk-management/pre-trade-and-risk.md](docs/risk-management/pre-trade-and-risk.md) | — |
| Know the regulations before going live | [reference/glossary-and-reading.md §Regulatory](reference/glossary-and-reading.md) | — |

> **Dev Phases reference:** [development-phases.md Phase 2](prompts/phases/development-phases.md) tasks 4-7

**Cost: $0**

### Tier 0 Checkpoint

At this point you have **a complete trading system** (parser → book → signals → strategy → risk → backtest) running on free tools and sample data. Nothing else cost a dime.

**Total spent: $0**
**Lines of C++ written: ~5,000-10,000**
**Capabilities:** Parse ITCH, build book, run strategies, backtest, measure latency — all on your dev machine.

---

## Tier 1 — Cheap Data & Cloud ($200 – $1,000)

> *Real data makes your backtests meaningful.*

### 1.1 — Historical Data ($0 – $500)

| Item | Cost | Source |
|---|---|---|
| LOBSTER academic dataset (L2/L3) | Free – $500 | lobsterdata.com (academic pricing) |
| IEX DEEP historical | Free | iexcloud.io (free tier) |
| NASDAQ TotalView replay files | $0 – $300 | NASDAQ historical data shop |
| Polygon.io historical | $0 – $200/mo | polygon.io starter plan |

### 1.2 — Cloud Backtesting ($0 – $500)

| Item | Cost | Notes |
|---|---|---|
| AWS/GCP spot instances for large replays | $50–300 | c6i.4xlarge spot ~ $0.30/hr |
| S3/GCS storage for data | $10–50/mo | ~50 GB historical data |

### 1.3 — What to Do

| Do | Read | Prompt |
|---|---|---|
| Re-run all backtests with real data (not samples) | [backtesting/framework-and-metrics.md](docs/backtesting/framework-and-metrics.md) | — |
| Walk-forward optimization | [backtesting/framework-and-metrics.md](docs/backtesting/framework-and-metrics.md) | — |
| Validate fill simulation against real fills | [backtesting/framework-and-metrics.md](docs/backtesting/framework-and-metrics.md) | — |
| Identify best instrument/venue for go-live | [algorithms/strategies-and-signals.md](docs/algorithms/strategies-and-signals.md) | — |

**Total spent so far: $200 – $1,000**

---

## Tier 2 — Lab Hardware ($5,000 – $15,000)

> *Real hardware. Real feeds. Still in your office/home.*
> See [hardware-reference.md](hardware/hardware-reference.md) for all specs, links, and prices.

### 2.1 — Server ($3,000 – $8,000)

| Item | Cost | Hardware Reference Section |
|---|---|---|
| Dev server (Supermicro, EPYC or i9) | $3,000–8,000 | [hardware-reference.md §Servers](hardware/hardware-reference.md) |

Build or buy a server with: 16+ cores, 64+ GB DDR5, NVMe, PCIe 4.0+.

### 2.2 — NIC ($500 – $1,000)

| Item | Cost | Hardware Reference Section |
|---|---|---|
| Intel E810-CQDA2 | $500–1,000 | [hardware-reference.md §NICs → Intel E810](hardware/hardware-reference.md) |

Best budget option — DPDK + PTP hardware timestamps. Perfect for learning kernel bypass before spending on Solarflare.

### 2.3 — Switch ($1,000 – $3,000)

| Item | Cost | Hardware Reference Section |
|---|---|---|
| Used Arista 7050 (eBay/refurb) | $1,000–3,000 | [hardware-reference.md §Switches](hardware/hardware-reference.md) |

### 2.4 — Cables & Optics ($100 – $200)

| Item | Cost | Hardware Reference Section |
|---|---|---|
| DAC cables, SFP+ optics | $100–200 | [hardware-reference.md §Cables & Optics](hardware/hardware-reference.md) |

### 2.5 — What to Do

| Do | Read | Prompt |
|---|---|---|
| Set up DPDK on Intel E810 | [networking/kernel-bypass.md §DPDK](docs/networking/kernel-bypass.md) | [research-prompts.md](prompts/phases/research-prompts.md) → "Inter-Process Communication" (DPDK rte_ring section) |
| Full OS tuning (isolcpus, hugepages, IRQ affinity) | [deployment/operations-and-infrastructure.md §2](docs/deployment/operations-and-infrastructure.md) | — |
| Process live delayed/replay market data through full pipeline | — | — |
| Measure real tick-to-signal latency on actual hardware | [cpp-performance/cache-optimization.md §Profiling](docs/cpp-performance/cache-optimization.md) | [research-prompts.md](prompts/phases/research-prompts.md) → "Cache Optimization Techniques" |
| **Target: tick-to-signal < 5 μs** | — | — |

> **Dev Phases reference:** [development-phases.md Phase 7-8](prompts/phases/development-phases.md)

**Total spent so far: $5,200 – $16,000**

**This maps to [INVESTMENT_PLAN.md Level 1](INVESTMENT_PLAN.md)** — the Apprentice tier.

---

## Tier 3 — Live Market Data ($500 – $2,000/month)

> *Paper trade against the real market.*

### 3.1 — Data Feeds

| Item | Cost/month | Notes |
|---|---|---|
| Polygon.io (real-time L2) | $200–400 | Good for equities |
| IEX real-time DEEP | Free | Single venue, good for testing |
| NASDAQ TotalView (via broker) | $0 – $50 | Through broker like IBKR |
| Delayed feeds (various) | Free | 15-min delay, enough for integration testing |

### 3.2 — What to Do

| Do | Read | Prompt |
|---|---|---|
| Receive live market data via kernel bypass | [networking/kernel-bypass.md](docs/networking/kernel-bypass.md) | — |
| Build live book from real feed | [algorithms/order-book-algorithms.md](docs/algorithms/order-book-algorithms.md) | — |
| Paper trade: strategy generates orders but doesn't send | [risk-management/pre-trade-and-risk.md](docs/risk-management/pre-trade-and-risk.md) | — |
| Log every decision for analysis | [backtesting/framework-and-metrics.md](docs/backtesting/framework-and-metrics.md) | — |
| Run for 20+ trading days, track paper PnL | — | — |
| Validate kill switch and risk limits in live conditions | [risk-management/pre-trade-and-risk.md](docs/risk-management/pre-trade-and-risk.md) | — |

> **Dev Phases reference:** [development-phases.md Phase 8](prompts/phases/development-phases.md) (network infra) + Phase 2 tasks 1-3 (exchange connectivity)

**Total spent so far: $6K – $20K** (one-time) + **$500–2K/month**

---

## Tier 4 — Go Live ($30,000 – $60,000)

> *Real money. Real fills. Real lessons.*

See **[INVESTMENT_PLAN.md Level 2](INVESTMENT_PLAN.md)** for full cost breakdown. This tier adds:

| Item | Cost | Hardware Reference Section |
|---|---|---|
| Solarflare X2522 NIC upgrade | $1,500–3,000 | [hardware-reference.md §NICs → Solarflare X2522](hardware/hardware-reference.md) |
| Broker/DMA account | $5,000–10,000 | — |
| Trading capital | $10,000–25,000 | — |
| Co-location (shared/Equinix Metal) | $5,000–15,000/yr | [hardware-reference.md §Co-location](hardware/hardware-reference.md) |
| Direct exchange feed | $2,000–5,000 | — |

### What to Do

| Do | Read | Prompt |
|---|---|---|
| Deploy Solarflare + OpenOnload | [networking/kernel-bypass.md §ef_vi](docs/networking/kernel-bypass.md) | — |
| Implement FIX/OUCH order entry | [market-data/exchange-protocols.md §FIX](docs/market-data/exchange-protocols.md) | [research-prompts.md](prompts/phases/research-prompts.md) → "FIX Protocol Optimization" |
| Build order lifecycle manager | [architecture/system-design.md §Order Manager](docs/architecture/system-design.md) | — |
| Build reconciliation system | — | — |
| Go live: min risk limits, single instrument | [risk-management/pre-trade-and-risk.md](docs/risk-management/pre-trade-and-risk.md) | — |

> **Dev Phases reference:** [development-phases.md Phase 9](prompts/phases/development-phases.md)

**Total spent: $30K – $60K**

---

## Tier 5+ — Scale (See Investment Plan)

Beyond this point, costs and decisions are covered in detail:

| Level | Cost | Where to Read |
|---|---|---|
| Competitor (co-located, multi-instrument) | $100K–250K | [INVESTMENT_PLAN.md Level 3](INVESTMENT_PLAN.md) |
| Professional (multi-venue, FPGA) | $500K–1.5M | [INVESTMENT_PLAN.md Level 4](INVESTMENT_PLAN.md) |
| Elite (custom hardware, global) | $2M–5M+ | [INVESTMENT_PLAN.md Level 5](INVESTMENT_PLAN.md) |

Hardware at each level: [hardware-reference.md §Budget Tiers](hardware/hardware-reference.md)

---

## Complete Build Order — At a Glance

```
TIER 0 — FREE                                           WEEK
  ├── 0.1  Environment setup ............................ 1
  ├── 0.2  Learn the domain ............................. 1
  ├── 0.3  ITCH parser .................................. 2
  ├── 0.4  Order book ................................... 3
  ├── 0.5  SPSC ring buffer & IPC ....................... 4
  ├── 0.6  Event loop & pipeline ........................ 5
  ├── 0.7  Backtesting engine ........................... 6-7
  ├── 0.8  Market making strategy ....................... 7-8
  ├── 0.9  Stat arb signals ............................. 8-9
  └── 0.10 Risk engine & kill switch .................... 9-10
                                            TOTAL: $0

TIER 1 — DATA ($200–$1K)
  ├── 1.1  Historical data .............................. 10-11
  └── 1.2  Cloud backtesting ............................ 11-12
                                            RUNNING: ~$500

TIER 2 — LAB HARDWARE ($5K–$15K)
  ├── 2.1  Server ....................................... 12-13
  ├── 2.2  NIC (Intel E810) ............................. 13
  ├── 2.3  Switch (used Arista) ......................... 13
  └── 2.5  DPDK + OS tuning + benchmarks ................ 14-16
                                            RUNNING: ~$10K

TIER 3 — LIVE DATA ($500–$2K/mo)
  ├── 3.1  Data feeds ................................... 16-17
  └── 3.2  Paper trading (20+ days) ..................... 17-22
                                            RUNNING: ~$15K

TIER 4 — GO LIVE ($30K–$60K)
  ├── NIC upgrade (Solarflare) ......................... 22-23
  ├── Broker + capital .................................. 23
  └── First live trades ................................. 24+
                                            RUNNING: ~$50K
```

---

## Prompt File Quick Reference

Use these prompts (feed them to your AI assistant) at the right time:

| When | Prompt File | Which Prompt Inside |
|---|---|---|
| Week 1-2 (protocols) | [research-prompts.md](prompts/phases/research-prompts.md) | "ITCH Protocol Deep Dive" |
| Week 2 (FIX) | [research-prompts.md](prompts/phases/research-prompts.md) | "FIX Protocol Optimization" |
| Week 3 (SIMD parsing) | [research-prompts.md](prompts/phases/research-prompts.md) | "SIMD for Financial Computing" |
| Week 4 (IPC) | [research-prompts.md](prompts/phases/research-prompts.md) | "Inter-Process Communication" |
| Week 5 (event loop) | [research-prompts.md](prompts/phases/research-prompts.md) | "Event Loop Architecture" |
| Week 6 (cache perf) | [research-prompts.md](prompts/phases/research-prompts.md) | "Cache Optimization Techniques" |
| Week 7-8 (market making) | [research-prompts.md](prompts/phases/research-prompts.md) | "Avellaneda-Stoikov Market Making" |
| Week 8 (volatility) | [research-prompts.md](prompts/phases/research-prompts.md) | "Volatility Modeling" |
| Week 8-9 (stat arb) | [research-prompts.md](prompts/phases/research-prompts.md) | "Statistical Arbitrage" |
| Week 8-9 (OU process) | [research-prompts.md](prompts/phases/research-prompts.md) | "Ornstein-Uhlenbeck Process" |
| Week 14+ (CME) | [research-prompts.md](prompts/phases/research-prompts.md) | "CME MDP 3.0 Analysis" |

The [development-phases.md](prompts/phases/development-phases.md) file maps the same work into 10 numbered phases with concrete task lists and deliverables — use it as your project management checklist.

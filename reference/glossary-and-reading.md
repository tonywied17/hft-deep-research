# HFT Glossary, Reading List & Regulatory Reference

> Extracted from original monolithic reference. Quick-lookup companion to the detailed docs/ files.

---

## Glossary

| Term | Definition |
|---|---|
| **BBO** | Best Bid and Offer — top-of-book prices |
| **CLOB** | Central Limit Order Book — the standard exchange matching model |
| **CME** | Chicago Mercantile Exchange |
| **DMA** | Direct Market Access — broker-provided direct exchange connectivity |
| **DMM** | Designated Market Maker |
| **EMS** | Execution Management System |
| **GBM** | Geometric Brownian Motion |
| **IGMP** | Internet Group Management Protocol (multicast joining) |
| **IOC** | Immediate or Cancel order type |
| **LULD** | Limit Up-Limit Down circuit breaker |
| **MPID** | Market Participant Identifier |
| **NBBO** | National Best Bid and Offer — best prices across all US exchanges |
| **NMS** | National Market System (Reg NMS) |
| **OMS** | Order Management System |
| **OU** | Ornstein–Uhlenbeck process |
| **PCIe** | Peripheral Component Interconnect Express |
| **PMD** | Poll Mode Driver (DPDK) |
| **PTP** | Precision Time Protocol (IEEE 1588) |
| **SBE** | Simple Binary Encoding |
| **SIP** | Securities Information Processor (consolidates NBBO) |
| **SPSC** | Single-Producer Single-Consumer |
| **TSC** | Time Stamp Counter (CPU register) |
| **VPIN** | Volume-Synchronized Probability of Informed Trading |
| **VWAP** | Volume-Weighted Average Price |

---

## Recommended Reading

### Books

| Title | Author | Focus |
|---|---|---|
| Trading and Exchanges | Larry Harris | Market microstructure bible |
| Algorithmic Trading & DMA | Barry Johnson | Exchange connectivity |
| Market Microstructure in Practice | Lehalle & Laruelle | Modern microstructure |
| All About High-Frequency Trading | Michael Durbin | HFT overview |
| High-Frequency Trading: A Practical Guide | Irene Aldridge | Practical HFT |
| Options, Futures, and Other Derivatives | John Hull | Quant finance reference |
| Stochastic Calculus for Finance I & II | Steven Shreve | Mathematical foundations |
| Optimal Mean Reversion Trading | Leung & Li | OU-based trading |
| C++ High Performance | Andrist & Sehr | Systems programming |
| Is Parallel Programming Hard? | Paul McKenney | Lock-free, memory ordering |

### Papers

| Paper | Topic |
|---|---|
| Kyle (1985) — "Continuous Auctions and Insider Trading" | Price impact model |
| Almgren & Chriss (2001) — "Optimal Execution of Portfolio Transactions" | Optimal execution |
| Avellaneda & Stoikov (2008) — "High-Frequency Trading in a Limit Order Book" | Market-making model |
| Cartea, Jaimungal & Penalva (2015) — "Algorithmic and High-Frequency Trading" | Comprehensive textbook |

### Conferences & Technical Sources

- **CppCon** — Low-latency C++ talks (Carl Cook, Fedor Pikus)
- **STAC Summit** — Securities Technology Analysis Center benchmarks
- **DPDK documentation** — doc.dpdk.org

---

## Regulatory Landscape

### Key Regulations Affecting HFT

| Regulation | Jurisdiction | Key Requirements |
|---|---|---|
| **Reg NMS** | US (SEC) | Order Protection Rule, Access Rule, Sub-Penny Rule |
| **Reg SHO** | US (SEC) | Short selling rules, locate requirements |
| **MiFID II / MiFIR** | EU (ESMA) | Algo trading obligations, market making requirements, tick sizes |
| **MAR** | EU (ESMA) | Prohibition of spoofing, layering, wash trading |
| **CAT** | US (SEC/FINRA) | Comprehensive trade reporting with timestamps |
| **Reg AT (proposed)** | US (CFTC) | Registration, risk controls, source code retention |

### Prohibited Practices

| Practice | Description | Regulation |
|---|---|---|
| **Spoofing** | Placing orders intended to be cancelled before execution to create false impression of demand | Dodd-Frank §747, MAR |
| **Layering** | Placing multiple non-bona-fide orders at different prices on one side to move the market | MAR |
| **Quote stuffing** | Flooding an exchange with orders and cancellations to slow competitors | Exchange rules |
| **Front-running** | Trading ahead of a client order using inside knowledge | Securities law |
| **Wash trading** | Simultaneously buying and selling to inflate volume | MAR, exchange rules |
| **Insider trading** | Trading on material non-public information | SEC Rule 10b-5, MAR |

### Compliance Requirements for HFT Firms

- **Source code retention:** Keep all algorithm source code with version history
- **Order & execution records:** Full audit trail of every order, modification, cancellation, and fill
- **Risk control documentation:** Document all pre-trade and post-trade risk controls
- **Clock synchronization:** Maintain and document time synchronization accuracy
- **Change management:** Formal testing and approval process for algorithm changes
- **Kill switch capability:** Ability to immediately cancel all outstanding orders and halt trading

---

## Quick-Reference Formulas

### Core Financial

$$P_{\text{mid}} = \frac{P_{\text{bid}} + P_{\text{ask}}}{2}$$

$$P_{\text{micro}} = \frac{P_{\text{bid}} Q_{\text{ask}} + P_{\text{ask}} Q_{\text{bid}}}{Q_{\text{bid}} + Q_{\text{ask}}}$$

$$\text{VWAP} = \frac{\sum_i P_i \cdot V_i}{\sum_i V_i}$$

$$\text{Sharpe} = \frac{E[R - R_f]}{\sigma_R} \qquad S_{\text{ann}} = S_{\text{daily}} \cdot \sqrt{252}$$

### Market Impact (Almgren-Chriss)

$$\text{Temporary: } h(\dot{x}) = \eta \cdot \dot{x} \qquad \text{Permanent: } g(\dot{x}) = \gamma \cdot \dot{x}$$

$$\text{Square-Root Law: Impact} \approx \sigma \cdot \sqrt{\frac{Q}{V}}$$

### Latency Quick Reference

| Measurement | Method | Overhead |
|---|---|---|
| TSC counter | `__rdtsc()` | ~10 clock cycles |
| Monotonic time | `clock_gettime(CLOCK_MONOTONIC)` | ~20 ns (VDSO) |
| HW NIC timestamp | Solarflare ef_vi | ~0 (packet metadata) |

### Network Speed Reference

| Medium | Speed | Latency/meter |
|---|---|---|
| Fiber optic | ~200,000 km/s | ~5 ns/m |
| Microwave (air) | ~300,000 km/s | ~3.3 ns/m |
| PCIe Gen 4 | 16 GT/s per lane | ~100 ns round-trip |

### Memory Size Reference

| Data | Size | Cache Lines |
|---|---|---|
| L1 cache line | 64 bytes | 1 |
| Typical order struct | 64–128 bytes | 1–2 |
| ITCH Add Order msg | 36 bytes | 1 |
| Full order book (1000 levels, 2 sides) | ~128 KB | ~2000 |

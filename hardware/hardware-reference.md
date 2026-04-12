# HFT Hardware Reference

## Quick Navigation
- [NICs (Network Interface Cards)](#nics)
- [Switches](#switches)
- [Servers](#servers)
- [FPGAs](#fpgas)
- [Time Synchronization](#time-synchronization)
- [Cables & Optics](#cables--optics)
- [Co-location Accessories](#co-location-accessories)

---

## NICs

### Solarflare X2522 / X3522 (AMD Xilinx)

| Field | Detail |
|---|---|
| **Purpose** | Ultra-low-latency NIC with kernel bypass (OpenOnload, ef_vi) |
| **Ports** | 2× 10/25GbE SFP28 |
| **Latency** | ~260 ns port-to-port (cut-through) |
| **Key Feature** | OpenOnload user-space TCP/UDP stack, ef_vi raw API, hardware timestamps |
| **Cost** | $1,500–$3,000 |
| **Link** | https://www.xilinx.com/products/boards-and-kits/alveo/x3522.html |
| **Cost Efficiency** | **Best value for HFT entry.** OpenOnload is a drop-in acceleration (no code changes). ef_vi gives sub-microsecond latency. The go-to NIC for most HFT firms. |

### Solarflare XtremeScale X25 (Cisco/Exablaze)

| Field | Detail |
|---|---|
| **Purpose** | Lowest-latency commercially available NIC |
| **Ports** | 2× 25GbE SFP28 |
| **Latency** | < 200 ns (wire-to-wire) |
| **Key Feature** | ExaSock user-space stack, hardware timestamping at PHY, SmartNIC programmable pipeline |
| **Cost** | $3,000–$5,000 |
| **Link** | https://www.cisco.com/c/en/us/products/switches/nexus-smartnic/ |
| **Cost Efficiency** | Premium over Solarflare X2522 but ~30% lower latency. Justified only if you're competitive at the nanosecond level. |

### Mellanox ConnectX-7 (NVIDIA)

| Field | Detail |
|---|---|
| **Purpose** | High-bandwidth NIC with RDMA, DPDK support |
| **Ports** | 1-2× 100/200/400GbE QSFP |
| **Latency** | ~600 ns (DPDK), ~900 ns (kernel) |
| **Key Feature** | DPDK native support, hardware offloads, huge bandwidth, InfiniBand support |
| **Cost** | $800–$2,500 (varies by port config) |
| **Link** | https://www.nvidia.com/en-us/networking/ethernet-adapters/ |
| **Cost Efficiency** | **Best for bandwidth-heavy workloads** (multi-venue, high message rate). Cheaper than Solarflare but higher latency. Good for non-latency-critical paths (risk, logging, management). |

### Intel E810-CQDA2

| Field | Detail |
|---|---|
| **Purpose** | Budget kernel bypass NIC with DPDK and PTP |
| **Ports** | 2× 100GbE QSFP28 |
| **Latency** | ~800 ns (DPDK) |
| **Key Feature** | Native DPDK, hardware PTP timestamping, ADQ (Application Device Queues), AF_XDP support |
| **Cost** | $500–$1,000 |
| **Link** | https://www.intel.com/content/www/us/en/products/details/ethernet/800-network-adapters.html |
| **Cost Efficiency** | **Best budget option.** PTP hardware timestamps at a fraction of Solarflare's price. Good for development, backtesting infrastructure, and non-critical paths. |

### NIC Comparison Summary

| NIC | Wire-to-Wire Latency | Price | Best For |
|---|---|---|---|
| Exablaze X25 | < 200 ns | $3,000–5,000 | Absolute lowest latency |
| Solarflare X3522 | ~260 ns | $1,500–3,000 | Best all-around HFT NIC |
| Mellanox CX-7 | ~600 ns | $800–2,500 | High bandwidth, RDMA |
| Intel E810 | ~800 ns | $500–1,000 | Budget, development, PTP |

---

## Switches

### Arista 7130 (FPGA Switch)

| Field | Detail |
|---|---|
| **Purpose** | Ultra-low-latency switch with embedded FPGA |
| **Ports** | 24-48× 10/25GbE + Xilinx FPGA |
| **Latency** | ~80 ns (Layer 1 bypass), ~350 ns (Layer 2) |
| **Key Feature** | Inline FPGA allows custom packet processing (parsing, risk checks, simple strategies) directly in the switch fabric |
| **Cost** | $30,000–$80,000 |
| **Link** | https://www.arista.com/en/products/7130-series |
| **Cost Efficiency** | **Expensive but unique.** Only switch that lets you run custom FPGA logic inline. Eliminates an entire hop. Justified for firms doing FPGA-based trading. |

### Arista 7050X4

| Field | Detail |
|---|---|
| **Purpose** | Standard low-latency data center switch |
| **Ports** | 32× 100GbE QSFP28 |
| **Latency** | ~380 ns (cut-through) |
| **Key Feature** | Cut-through switching, LANZ latency monitoring, PTP boundary clock |
| **Cost** | $15,000–$25,000 |
| **Link** | https://www.arista.com/en/products/7050x4-series |
| **Cost Efficiency** | **Best general-purpose HFT switch.** Sub-400 ns, PTP support, reliable. The standard choice for most trading firms. |

### Cisco Nexus 3550-F (formerly Exablaze ExaLINK Fusion)

| Field | Detail |
|---|---|
| **Purpose** | Layer 1 switch / network tap with hardware timestamping |
| **Ports** | 16-48× 10/25GbE |
| **Latency** | ~5 ns (Layer 1 mode) |
| **Key Feature** | Layer 1 (physical layer) switching — essentially a configurable patch panel with nanosecond-precision timestamps on every packet |
| **Cost** | $20,000–$50,000 |
| **Link** | https://www.cisco.com/c/en/us/products/switches/nexus-3550-f-fusion.html |
| **Cost Efficiency** | **Essential for latency measurement** even if not used as primary switch. 5 ns switching is unbeatable for A/B feed selection. |

### Metamako (Arista subsidiary)

| Field | Detail |
|---|---|
| **Purpose** | Ultra-low-latency Layer 1 / FPGA hybrid switch |
| **Ports** | Various 10/25/100GbE |
| **Latency** | ~4 ns (Layer 1), ~100 ns (with FPGA logic) |
| **Key Feature** | Stackable, FPGA-programmable, hardware timestamping on every port |
| **Cost** | $25,000–$60,000 |
| **Link** | https://www.arista.com/en/products/7130-series (now part of 7130 line) |
| **Cost Efficiency** | Similar to 7130 — combined switch + FPGA platform. Good if you need both switching and inline processing. |

### Switch Comparison Summary

| Switch | Latency | Price | Best For |
|---|---|---|---|
| Nexus 3550-F | ~5 ns (L1) | $20K–50K | Feed selection, timestamping |
| Arista 7130 | ~80 ns (L1) | $30K–80K | Inline FPGA processing |
| Arista 7050X4 | ~380 ns | $15K–25K | General-purpose HFT switch |

---

## Servers

### Supermicro SYS-1029U-TRTP

| Field | Detail |
|---|---|
| **Purpose** | Standard 1U co-location server |
| **Specs** | Dual Intel Xeon Scalable, up to 6TB DDR4, 10× 2.5" NVMe |
| **Key Feature** | Compact 1U form factor (rack space is expensive in colo), IPMI remote management |
| **Cost** | $5,000–$15,000 (configured) |
| **Link** | https://www.supermicro.com/en/products/system/1U/ |
| **Cost Efficiency** | **Best colo server value.** 1U saves rack space ($500-1500/U/month). Dual socket gives NUMA flexibility. Most HFT firms use Supermicro. |

### Dell PowerEdge R660

| Field | Detail |
|---|---|
| **Purpose** | Enterprise-grade 1U server |
| **Specs** | Dual Intel Xeon Scalable 4th/5th Gen, up to 8TB DDR5, PCIe 5.0 |
| **Key Feature** | DDR5 for lower memory latency, PCIe 5.0 for faster NIC communication, iDRAC remote management |
| **Cost** | $8,000–$25,000 (configured) |
| **Link** | https://www.dell.com/en-us/shop/servers-storage-and-networking/poweredge-r660-rack-server/ |
| **Cost Efficiency** | Premium over Supermicro but better support. DDR5 gives ~15% lower memory latency. PCIe 5.0 matters if using multiple NICs. |

### CPU Selection Guide

| CPU | Cores | Base/Boost | L3 Cache | Best For | Price |
|---|---|---|---|---|---|
| Intel Xeon w5-3435X | 16C/32T | 3.1/4.7 GHz | 45 MB | HFT (high single-thread) | $1,100 |
| Intel Xeon w9-3495X | 56C/112T | 1.9/4.8 GHz | 105 MB | Multi-strategy | $5,900 |
| AMD EPYC 9554 | 64C/128T | 3.1/3.75 GHz | 256 MB | Huge L3 cache | $4,500 |
| AMD EPYC 9174F | 16C/32T | 4.1/4.4 GHz | 256 MB | HFT (frequency + cache) | $3,500 |
| Intel Core i9-14900KS | 24C/32T | 3.2/6.2 GHz | 36 MB | Dev/test, highest clock | $700 |

**HFT CPU Priority:** Single-thread frequency > core count > cache size. The AMD EPYC 9174F is exceptionally strong: 4.1 GHz base with 256 MB L3 cache at reasonable cost.

### Memory Configuration

```
Optimal DDR5 Configuration for HFT:
  - Populate all channels (8 channels for dual-socket)
  - Use single-rank DIMMs for lowest latency
  - DDR5-5600 or DDR5-6400 preferred
  - 64 GB minimum (32 GB × 2 for single socket)
  - Enable interleaving across channels
  
  Latency: DDR5-5600 ~70 ns, DDR4-3200 ~80 ns
  Bandwidth: DDR5-5600 ~90 GB/s per channel
```

---

## FPGAs

### AMD Xilinx Alveo U250

| Field | Detail |
|---|---|
| **Purpose** | General-purpose FPGA accelerator for HFT |
| **Chip** | Xilinx UltraScale+ VU13P |
| **Resources** | 1.3M LUTs, 54 MB BRAM, 2× 100GbE |
| **Key Feature** | Vitis/Vivado development, large fabric for complex logic, 2× QSFP28 network ports onboard |
| **Cost** | $5,000–$8,000 |
| **Link** | https://www.xilinx.com/products/boards-and-kits/alveo/u250.html |
| **Cost Efficiency** | **Best general HFT FPGA.** Large enough for full ITCH parser + order book + simple strategy. Two 100G ports for market data + order entry. |

### AMD Xilinx Alveo U55C (HBM)

| Field | Detail |
|---|---|
| **Purpose** | High-memory FPGA for large-state applications |
| **Chip** | Xilinx UltraScale+ VU35P |
| **Resources** | 573K LUTs, 16 GB HBM2, 2× 100GbE |
| **Key Feature** | 16 GB HBM2 on-card (460 GB/s bandwidth) — can hold entire order books for all instruments in FPGA memory |
| **Cost** | $8,000–$12,000 |
| **Link** | https://www.xilinx.com/products/boards-and-kits/alveo/u55c.html |
| **Cost Efficiency** | Worth the premium if your application needs large state (multi-instrument book, full market snapshot). HBM eliminates PCIe round-trips to host memory. |

### Intel Agilex 7 (via Silicom/BittWare)

| Field | Detail |
|---|---|
| **Purpose** | Alternative FPGA ecosystem (Intel OneAPI/Quartus) |
| **Chip** | Intel Agilex 7 F-Series |
| **Resources** | 1.4M ALMs, HBM2e, 2-4× 100GbE |
| **Key Feature** | OneAPI high-level synthesis, PCIe 5.0 (lower host latency), CXL support |
| **Cost** | $6,000–$15,000 (board-dependent) |
| **Link** | https://www.intel.com/content/www/us/en/products/details/fpga/agilex/7.html |
| **Cost Efficiency** | Competitive with Xilinx on specs. PCIe 5.0 advantage. Smaller ecosystem/community than Xilinx in HFT — most firms use Xilinx. |

### FPGA Cost-Benefit Analysis

```
ROI calculation for FPGA market data parsing:

  Software parsing: ~200-500 ns per message
  FPGA parsing:     ~50-100 ns per message
  Improvement:      ~150-400 ns
  
  If this edge earns an extra $0.001/trade across 100,000 trades/day:
    Daily revenue:  $100
    Annual revenue: $25,200
    
  FPGA cost:        $8,000 (card) + $50,000 (6 months dev)
  Break-even:       ~2.3 years
  
  Only justified if:
    - You're competitive at the 100 ns level
    - Trade volume is high enough
    - You have FPGA engineering talent
```

---

## Time Synchronization

### Meinberg IMS LANTIME M1000

| Field | Detail |
|---|---|
| **Purpose** | PTP Grandmaster Clock with GPS/GNSS |
| **Accuracy** | < 10 ns to UTC (GPS-disciplined) |
| **Ports** | 4× GbE with hardware PTP, 1PPS output |
| **Key Feature** | Independent GPS/GLONASS/Galileo receiver, holdover oscillator (maintains accuracy during GPS outage) |
| **Cost** | $10,000–$20,000 |
| **Link** | https://www.meinbergglobal.com/english/products/ieee-1588-grand-master-clock.htm |
| **Cost Efficiency** | **Essential investment.** A $15K grandmaster gives < 50 ns accuracy across your entire network. Exchange timestamps are PTP-synchronized; you need the same. |

### Spectracom SecureSync 2400

| Field | Detail |
|---|---|
| **Purpose** | Enterprise PTP/NTP time server |
| **Accuracy** | < 15 ns to UTC |
| **Ports** | Multiple GbE, optional 10GbE |
| **Key Feature** | Multi-GNSS, rubidium oscillator option (weeks of holdover) |
| **Cost** | $15,000–$30,000 |
| **Link** | https://safran-navigation-timing.com/product/securesync-2400/ |
| **Cost Efficiency** | Premium but offers rubidium oscillator for exceptional holdover. Redundant with Meinberg for critical setups. |

### Endrun Meridian II

| Field | Detail |
|---|---|
| **Purpose** | Compact PTP Grandmaster |
| **Accuracy** | < 25 ns to UTC |
| **Ports** | 2× GbE with PTP |
| **Key Feature** | Small form factor (1U), GPS-disciplined OCXO |
| **Cost** | $5,000–$10,000 |
| **Link** | https://endruntechnologies.com/products/gps-time-servers/meridian2 |
| **Cost Efficiency** | **Budget grandmaster.** Good enough for most HFT setups. Half the price of Meinberg with only marginally worse accuracy. |

---

## Cables & Optics

### Direct Attach Copper (DAC) Cables

| Type | Length | Latency/m | Price | Best For |
|---|---|---|---|---|
| SFP28 25G DAC | 1-5m | ~5.0 ns/m | $20–50 | Intra-rack connections |
| QSFP28 100G DAC | 1-5m | ~5.0 ns/m | $40–80 | Switch uplinks |
| QSFP56 200G DAC | 1-3m | ~5.0 ns/m | $80–150 | High-bandwidth links |

**Cost Efficiency:** Always use DAC within racks. Cheaper, lower latency, and more reliable than optics for short distances.

### Fiber Optic Cables and Transceivers

| Type | Distance | Latency/m | Transceiver Cost | Best For |
|---|---|---|---|---|
| OM4 Multimode + SR | < 100m | ~4.9 ns/m | $30–80 (SFP28 SR) | Intra-DC, cross-connect |
| OS2 Singlemode + LR | < 10 km | ~4.9 ns/m | $200–500 (SFP28 LR) | Inter-building |
| OS2 Singlemode + ER | < 40 km | ~4.9 ns/m | $1,000–3,000 | Metro links |

**Cost Efficiency:** Within the data center, OM4 multimode is optimal — cheaper transceivers, sufficient distance. Use singlemode only for longer runs.

### Cable Selection Rules

```
Within same rack (< 3m):  → DAC copper ($30, lowest latency)
Same row (3-30m):         → OM4 multimode fiber ($100 total)
Cross-connect to exchange: → Managed fiber from exchange (varies)
Between buildings:        → OS2 singlemode ($500+ total)

Latency comparison for 5m cable:
  DAC copper:    25 ns
  OM4 fiber:     24.5 ns
  Difference:    0.5 ns (negligible)
  
  → Use DAC whenever distance allows (cheaper, no transceiver issues)
```

---

## Co-location Accessories

### PDU (Power Distribution Unit)

| Product | Purpose | Price |
|---|---|---|
| APC AP8886 Metered Rack PDU | Dual-input power monitoring | $1,500–2,500 |
| Raritan PX3-5902V | Per-outlet metering and switching | $2,000–3,500 |

Most co-location providers include basic PDUs. Metered PDUs help track exact power usage (billed per kW).

### KVM / Remote Management

| Product | Purpose | Price |
|---|---|---|
| Raritan Dominion KX IV | IP KVM for remote server access | $2,000–4,000 |
| Supermicro IPMI (built-in) | Basic remote management | Included with server |

**Cost Efficiency:** IPMI is sufficient for daily operations. KVM is insurance for when IPMI fails.

### Environmental Monitoring

| Product | Purpose | Price |
|---|---|---|
| APC NetBotz 250 | Temperature, humidity, airflow monitoring | $1,500–2,500 |
| Geist Environmental Monitor | Cabinet-level environmental sensors | $500–1,000 |

---

## Budget Tiers

### Tier 1: Development / Learning ($5K–$15K)

| Item | Product | Est. Cost |
|---|---|---|
| Server | Supermicro 1U or desktop with EPYC/i9 | $3,000–8,000 |
| NIC | Intel E810-CQDA2 (DPDK + PTP) | $500–1,000 |
| Switch | Used Arista 7050 series | $1,000–3,000 |
| Cables | DAC copper + OM4 fiber | $100–200 |
| **Total** | | **$4,600–$12,200** |

### Tier 2: Competitive Co-Located ($100K–$250K first year)

| Item | Product | Est. Cost |
|---|---|---|
| Co-location | 1/4 cabinet + cross-connect | $60K–120K/year |
| Server (×2) | Supermicro 1U, EPYC 9174F, 128GB DDR5 | $20,000 |
| NIC (×2) | Solarflare X3522 | $5,000 |
| Switch | Arista 7050X4 | $20,000 |
| PTP Grandmaster | Endrun Meridian II | $7,000 |
| Optics & cables | Mixed DAC + OM4 | $500 |
| **Total Year 1** | | **$112K–$173K** |

### Tier 3: Professional HFT ($500K–$2M+ first year)

| Item | Product | Est. Cost |
|---|---|---|
| Co-location | Full cabinet + multiple cross-connects | $200K–600K/year |
| Servers (×4-8) | Dell R660, DDR5, PCIe 5.0 | $100,000–200,000 |
| NICs (×4-8) | Solarflare X3522 + Exablaze X25 | $20,000–40,000 |
| Switch | Arista 7130 (FPGA) + 7050X4 | $80,000–150,000 |
| FPGA | Xilinx Alveo U250 (×2) | $16,000 |
| PTP Grandmaster | Meinberg M1000 (×2 redundant) | $30,000 |
| Time switch | Cisco Nexus 3550-F | $30,000 |
| Microwave data | McKay Brothers / Anova subscription | $50K–200K/year |
| **Total Year 1** | | **$526K–$1.4M** |

# Network Hardware, Switches & Cabling

## 1. NIC Architecture for HFT

### 1.1 How a Modern NIC Works (DMA Model)

```
CPU ←──── PCIe Bus ────→ NIC
            │                │
     ┌──────▼──────┐   ┌────▼─────┐
     │ Host Memory  │   │ NIC ASIC │
     │              │   │          │
     │ ┌──────────┐ │   │ ┌──────┐ │
     │ │ RX Desc  ◄─┼───┼─┤ DMA  │ │
     │ │ Ring     │ │   │ │Engine│ │
     │ └──────────┘ │   │ └──────┘ │
     │              │   │          │
     │ ┌──────────┐ │   │ ┌──────┐ │
     │ │ Packet   ◄─┼───┼─┤ DMA  │ │
     │ │ Buffers  │ │   │ │Engine│ │
     │ └──────────┘ │   │ └──────┘ │
     │              │   │          │
     │ ┌──────────┐ │   │ ┌──────┐ │
     │ │ TX Desc  ──┼───┼─► DMA  │ │
     │ │ Ring     │ │   │ │Engine│ │
     │ └──────────┘ │   │ └──────┘ │
     └──────────────┘   └──────────┘
```

**Packet Receive Flow:**
1. Packet arrives on wire
2. NIC writes packet data to pre-posted buffer in host memory via DMA
3. NIC writes completion descriptor to RX descriptor ring
4. Application polls descriptor ring (no interrupt → kernel bypass)
5. Application reads packet data directly from host memory buffer

### 1.2 Hardware Timestamping

Modern HFT NICs timestamp packets in hardware at the PHY layer:

```
Wire ──► PHY ──► MAC ──► Filters ──► DMA ──► Host Memory
          │
          └── Timestamp captured here
              (before any NIC processing delay)
```

**Timestamp Sources:**
| Source | Precision | Latency | Method |
|---|---|---|---|
| NIC PHY hardware | <10 ns | None | PTP-synchronized NIC clock |
| NIC MAC | ~50 ns | Minimal | After PHY processing |
| Kernel `SO_TIMESTAMPNS` | ~1 μs | Kernel overhead | Software timestamp |
| Application `clock_gettime` | ~20 ns | After full stack | vDSO |
| Application `__rdtsc()` | ~1 ns | After full processing | TSC counter |

**Solarflare Hardware Timestamps:**
```c
// ef_vi: Hardware timestamp in RX event
ef_event events[64];
int n = ef_eventq_poll(&vi, events, 64);
for (int i = 0; i < n; i++) {
    if (EF_EVENT_TYPE(events[i]) == EF_EVENT_TYPE_RX) {
        // Get hardware timestamp (nanoseconds since NIC epoch)
        struct ef_timespec ts;
        ef_vi_receive_get_timestamp(&vi, pkt_data, &ts);
        // ts.tv_sec, ts.tv_nsec — PTP-synchronized
    }
}
```

### 1.3 Receive Side Scaling (RSS) & Flow Director

NICs can distribute incoming packets across multiple RX queues:

```
Incoming Packets
       │
       ▼
  ┌──────────┐
  │ NIC ASIC │
  │          │
  │ ┌──────┐ │     
  │ │ RSS  │ │  Hash(src_ip, dst_ip, src_port, dst_port) % num_queues
  │ │ Hash │ │     
  │ └──┬───┘ │     
  │    │     │     
  │ ┌──▼──┐  │     Queue 0 → Core 2 (Instruments A-F)
  │ │Queue│  │     Queue 1 → Core 3 (Instruments G-M)
  │ │ Map │──┼──── Queue 2 → Core 4 (Instruments N-Z)
  │ └─────┘  │     Queue 3 → Core 5 (Order responses)
  └──────────┘
```

**Flow Director (Intel):** Exact match rules — pin specific flows to specific queues:
```bash
# Pin specific multicast group to RX queue 0
ethtool -N eth0 flow-type udp4 dst-ip 233.1.0.1 dst-port 12001 action 0
```

**Solarflare Filters:**
```c
// ef_vi filter API — hardware-level packet steering
ef_filter_spec spec;
ef_filter_spec_init(&spec, EF_FILTER_FLAG_NONE);
ef_filter_spec_set_ip4_local(&spec, IPPROTO_UDP, 
    inet_addr("233.1.0.1"), htons(12001));
ef_vi_filter_add(&vi, dh, &spec, NULL);
```

---

## 2. Switch Architecture

### 2.1 Cut-Through vs Store-and-Forward

**Store-and-Forward:**
```
Receive entire packet → Verify FCS → Forward
Latency: proportional to packet size (416 ns for 64-byte at 10G)
```

**Cut-Through:**
```
Receive first 14 bytes (Ethernet header) → Begin forwarding immediately
Latency: fixed (~100-300 ns regardless of packet size)
```

HFT uses exclusively cut-through switches.

### 2.2 FPGA-Based Switches (Arista 7130 / Metamako)

The Arista 7130 series contains an embedded FPGA that can:
- Timestamp packets with <5 ns precision
- Perform in-line packet modification
- Execute custom logic on every packet
- Multicast with near-zero additional latency

```
Port 1 (Exchange) ──► [FPGA Logic] ──► Port 2 (Server 1)
                          │
                          ├──► Port 3 (Server 2)  ← Hardware multicast
                          └──► Port 4 (Timestamp capture)
                          
Total port-to-port: <50 ns
```

### 2.3 Switch Latency Comparison

| Switch | Architecture | Port-to-Port | Ports | Year |
|---|---|---|---|---|
| Arista 7130-48 | FPGA (Metamako) | <50 ns | 48×10G/25G | 2020+ |
| Arista 7130L-48 | FPGA | <50 ns | 48×1G/10G | 2019 |
| Arista 7050X4 | Memory-based, cut-through | ~380 ns | 32×100G | 2021 |
| Arista 7280R3 | Cut-through | ~300 ns | Flexible | 2022 |
| Cisco N3K-C3548 | Cut-through | ~250 ns | 48×10G | Legacy |
| Cisco N9K-C9336C | Cut-through | ~900 ns | 36×100G | 2020 |
| Juniper QFX5120 | Cut-through | ~350 ns | 48×25G | 2020 |

---

## 3. Cabling & Physical Layer

### 3.1 Cable Types & Latency

| Cable Type | Speed | Latency per Meter | Max Length | Use Case |
|---|---|---|---|---|
| DAC (Direct Attach Copper) | 10-100G | ~5 ns/m | 5-7m | Within rack |
| Active Optical Cable (AOC) | 10-100G | ~5 ns/m | 100m | Within data center |
| Single-Mode Fiber (SMF) | 10-400G | ~4.9 ns/m | 40 km | Data center to exchange |
| Multi-Mode Fiber (MMF) | 10-100G | ~5.0 ns/m | 550m | Within data center |
| Cat6a Copper | 10G | ~5.3 ns/m | 100m | Legacy |

### 3.2 Cross-Connect Latency

In a co-location facility:
```
Your Rack ──── Cross-Connect Cable ──── Exchange Rack
    │                                        │
    │    Typical: 10-50 meters               │
    │    Latency: 50-250 ns round-trip       │
    │                                        │
    │    Most exchanges equalize:            │
    │    All firms get same cable length     │
    │    (e.g., 30m standard)               │
    │                                        │
```

### 3.3 Inter-Data-Center Links

| Route | Distance | Fiber (RTT) | Microwave (RTT) | Difference |
|---|---|---|---|---|
| Mahwah NJ ↔ Carteret NJ | ~35 km | ~350 μs | ~240 μs | ~110 μs |
| Carteret NJ ↔ Secaucus NJ | ~15 km | ~150 μs | ~100 μs | ~50 μs |
| NJ ↔ Chicago (Aurora) | ~1,200 km | ~12.6 ms | ~7.9 ms | ~4.7 ms |
| NJ ↔ London | ~5,500 km | ~65 ms (sub) | N/A (ocean) | N/A |
| Chicago ↔ Tokyo | ~10,000 km | ~130 ms | N/A | N/A |

---

## 4. Microwave & Millimeter-Wave Networks

### 4.1 Why Microwave?

Speed of light:
- In vacuum/air: 299,792 km/s (c)
- In fiber optic glass: ~200,000 km/s (~0.67c)

Microwave travels through air at ~c, giving a **~33% speed advantage** over fiber.

### 4.2 Microwave Link Architecture

```
Tower A ──── Line of Sight ──── Tower B ──── ... ──── Tower N
(NJ)     ~30 km per hop     (PA relay)            (Chicago)

Requirements per hop:
- Clear line of sight (no terrain obstructions)
- Licensed frequency band (6-42 GHz)
- Dish antennas (1-3 meter diameter)
- Weather resilience (rain fade at higher frequencies)
```

### 4.3 Microwave Network Providers

| Provider | Route | One-Way Latency | Technology |
|---|---|---|---|
| McKay Brothers | NJ ↔ Chicago | ~3.95 ms | Microwave |
| Anova Technologies | NJ ↔ Chicago | ~3.96 ms | Microwave |
| Jump Trading (HRT) | NJ ↔ Chicago | ~3.9 ms | Microwave (proprietary) |
| Hibernia Networks | NY ↔ London | ~29.7 ms | Submarine fiber (straight-line) |
| Spread Networks | NJ ↔ Chicago | ~6.35 ms | Dark fiber (straight route) |

### 4.4 Millimeter-Wave (60-90 GHz)

Higher bandwidth than microwave but more weather-sensitive:
- **Bandwidth:** 1-10 Gbps (vs ~100 Mbps for microwave)
- **Rain attenuation:** Severe at 60+ GHz
- **Range per hop:** 1-5 km (vs 30+ km for microwave)
- **Use case:** Short-distance, high-bandwidth (within a metro area)

---

## 5. PCIe Topology & NIC Placement

### 5.1 PCIe Latency

| PCIe Gen | Bandwidth per Lane | Round-Trip Latency |
|---|---|---|
| PCIe 3.0 | 8 GT/s (985 MB/s/lane) | ~300-500 ns |
| PCIe 4.0 | 16 GT/s (1,969 MB/s/lane) | ~200-400 ns |
| PCIe 5.0 | 32 GT/s (3,938 MB/s/lane) | ~150-300 ns |

### 5.2 Optimal NIC Placement

```
CPU Socket 0                    CPU Socket 1
├── PCIe Root Complex           ├── PCIe Root Complex
│   ├── Slot 0: Trading NIC ✓  │   ├── Slot 4: (unused)
│   ├── Slot 1: (unused)       │   ├── Slot 5: Management NIC
│   └── Slot 2: NVMe (logs)    │   └── Slot 6: (unused)
└── DRAM (Node 0) ✓             └── DRAM (Node 1)

Rule: Trading NIC MUST be on the same NUMA node as the trading cores
      Cross-socket PCIe access adds ~200-300 ns
```

```bash
# Verify NIC NUMA node
cat /sys/class/net/ens1f0/device/numa_node  # Should match your trading cores

# Verify PCIe link status
lspci -vvs 17:00.0 | grep -i "lnksta"
# LnkSta: Speed 16GT/s, Width x16  ← PCIe 4.0 x16 = max bandwidth
```

# Kernel Bypass Networking for HFT

## Overview

The Linux kernel network stack adds 5-15 μs of latency per direction. Kernel bypass eliminates this by mapping NIC hardware directly into user-space, allowing the application to send/receive packets without any syscall or context switch.

---

## 1. Why the Kernel Stack Is Slow

### 1.1 Latency Breakdown of a Normal Packet Receive

```
1. Packet arrives at NIC                           →   0 ns
2. NIC DMA to kernel ring buffer                   → +200 ns
3. NIC raises interrupt (or interrupt coalescing)   → +0-5 μs (coalescing delay)
4. Kernel interrupt handler (top half)             → +200 ns
5. NAPI softirq (bottom half) — process SKBs       → +1-2 μs
6. IP/TCP/UDP stack processing                      → +1-3 μs
7. Socket buffer copy (kernel → user space)         → +500 ns
8. Context switch to wake blocked process           → +1-5 μs
   ─────────────────────────────────────────────────
   TOTAL:                                            ~5-15 μs
```

### 1.2 What Kernel Bypass Eliminates

```
1. Packet arrives at NIC                           →   0 ns
2. NIC DMA to user-space ring buffer (hugepages)   → +200 ns
3. Application polls ring buffer (busy-wait)        → +0-100 ns
4. Application reads packet directly from DMA buffer → +10 ns
   ─────────────────────────────────────────────────
   TOTAL:                                            ~300-500 ns
```

Eliminated:
- ✗ Hardware interrupts  
- ✗ Softirq processing  
- ✗ SKB allocation  
- ✗ Kernel TCP/IP stack  
- ✗ Socket buffer copy  
- ✗ Context switch  

---

## 2. DPDK (Data Plane Development Kit)

### 2.1 Architecture Deep Dive

```
┌─────────────────────────────────────────────────────────┐
│ User Space                                               │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ Application  │  │ Application  │  │ Application  │  │
│  │ (Core 2)     │  │ (Core 3)     │  │ (Core 4)     │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │
│         │                 │                  │          │
│  ┌──────▼─────────────────▼──────────────────▼───────┐  │
│  │              DPDK Libraries                        │  │
│  │  ┌─────────┐  ┌──────────┐  ┌──────────────────┐  │  │
│  │  │ EAL     │  │ Mempool  │  │ Poll Mode Driver │  │  │
│  │  │ (Init)  │  │ (mbufs)  │  │ (NIC driver)     │  │  │
│  │  └─────────┘  └──────────┘  └────────┬─────────┘  │  │
│  └───────────────────────────────────────┼───────────┘  │
│                                          │              │
│  ┌───────────────────────────────────────▼───────────┐  │
│  │           Hugepage Memory (DMA-able)               │  │
│  │  ┌──────────────┐  ┌──────────────────────────┐   │  │
│  │  │ RX Ring      │  │ mbuf Pool                │   │  │
│  │  │ (descriptors)│  │ (packet data buffers)    │   │  │
│  │  └──────────────┘  └──────────────────────────┘   │  │
│  └───────────────────────────────────────────────────┘  │
│                            ▲                            │
└────────────────────────────┼────────────────────────────┘
                             │ DMA (no CPU involvement)
┌────────────────────────────▼────────────────────────────┐
│ NIC Hardware                                             │
│  ┌──────────────────────────────────────────────┐       │
│  │ NIC RX/TX Descriptor Rings (hardware)         │       │
│  │ DMA engine writes packets to hugepage memory  │       │
│  └──────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────┘
```

### 2.2 Complete DPDK Setup

```bash
# 1. Install DPDK (from source for latest)
git clone https://github.com/DPDK/dpdk.git
cd dpdk
meson build -Dplatform=native
ninja -C build
sudo ninja -C build install

# 2. Reserve hugepages
echo 2048 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
mkdir -p /dev/hugepages
mount -t hugetlbfs nodev /dev/hugepages

# 3. Bind NIC to DPDK-compatible driver
modprobe vfio-pci
# Get NIC PCI address
dpdk-devbind.py --status
# Unbind from kernel driver, bind to vfio-pci
dpdk-devbind.py --bind=vfio-pci 0000:17:00.0

# 4. Verify
dpdk-devbind.py --status
# 0000:17:00.0 'Ethernet Controller' drv=vfio-pci
```

### 2.3 Production DPDK Application

```c
#include <rte_eal.h>
#include <rte_ethdev.h>
#include <rte_mbuf.h>
#include <rte_mempool.h>
#include <rte_cycles.h>

#define RX_RING_SIZE 4096
#define TX_RING_SIZE 4096
#define NUM_MBUFS 8191
#define MBUF_CACHE_SIZE 250
#define BURST_SIZE 64

static struct rte_mempool *mbuf_pool;

static int init_port(uint16_t port) {
    struct rte_eth_conf port_conf = {
        .rxmode = {
            .mq_mode = RTE_ETH_MQ_RX_NONE,
            .max_lro_pkt_size = RTE_ETHER_MAX_LEN,
        },
        .txmode = {
            .mq_mode = RTE_ETH_MQ_TX_NONE,
        },
    };
    
    struct rte_eth_dev_info dev_info;
    rte_eth_dev_info_get(port, &dev_info);
    
    // Configure port
    int ret = rte_eth_dev_configure(port, 1, 1, &port_conf);
    if (ret != 0) return ret;
    
    // Adjust RX/TX descriptors
    uint16_t nb_rxd = RX_RING_SIZE;
    uint16_t nb_txd = TX_RING_SIZE;
    rte_eth_dev_adjust_nb_rx_tx_desc(port, &nb_rxd, &nb_txd);
    
    // Setup RX queue on the correct NUMA node
    int socket_id = rte_eth_dev_socket_id(port);
    ret = rte_eth_rx_queue_setup(port, 0, nb_rxd, socket_id, NULL, mbuf_pool);
    if (ret < 0) return ret;
    
    // Setup TX queue
    ret = rte_eth_tx_queue_setup(port, 0, nb_txd, socket_id, NULL);
    if (ret < 0) return ret;
    
    // Enable promiscuous mode for multicast
    rte_eth_promiscuous_enable(port);
    
    // Start the port
    ret = rte_eth_dev_start(port);
    return ret;
}

// Main poll loop
static void rx_loop(uint16_t port) {
    struct rte_mbuf *bufs[BURST_SIZE];
    
    while (likely(running)) {
        // Poll NIC — returns immediately with 0 if no packets
        uint16_t nb_rx = rte_eth_rx_burst(port, 0, bufs, BURST_SIZE);
        
        if (unlikely(nb_rx == 0)) continue;
        
        for (uint16_t i = 0; i < nb_rx; i++) {
            // Access packet data directly — zero copy
            uint8_t *pkt_data = rte_pktmbuf_mtod(bufs[i], uint8_t*);
            uint32_t pkt_len = rte_pktmbuf_pkt_len(bufs[i]);
            
            // Parse protocol (skip Ethernet + IP + UDP headers)
            process_market_data(pkt_data + 42, pkt_len - 42);
            
            // Hardware timestamp (if NIC supports it)
            uint64_t hw_timestamp = bufs[i]->timestamp;
            
            rte_pktmbuf_free(bufs[i]);
        }
    }
}

// Sending a packet
static void send_order(uint16_t port, const uint8_t *order_data, uint16_t len) {
    struct rte_mbuf *pkt = rte_pktmbuf_alloc(mbuf_pool);
    
    // Build Ethernet + IP + TCP/UDP headers
    uint8_t *data = rte_pktmbuf_mtod(pkt, uint8_t*);
    build_headers(data, len);
    memcpy(data + 42, order_data, len);
    
    pkt->data_len = 42 + len;
    pkt->pkt_len = pkt->data_len;
    
    // Send — returns number of packets actually sent
    uint16_t sent = rte_eth_tx_burst(port, 0, &pkt, 1);
    if (unlikely(sent == 0)) {
        rte_pktmbuf_free(pkt);
    }
}

int main(int argc, char *argv[]) {
    // Initialize DPDK
    int ret = rte_eal_init(argc, argv);
    
    // Create mbuf pool on correct NUMA node
    mbuf_pool = rte_pktmbuf_pool_create("MBUF_POOL",
        NUM_MBUFS, MBUF_CACHE_SIZE, 0,
        RTE_MBUF_DEFAULT_BUF_SIZE,
        rte_socket_id());
    
    // Initialize port
    init_port(0);
    
    // Enter main loop (this core is now dedicated to polling)
    rx_loop(0);
    
    return 0;
}
```

### 2.4 DPDK Multicast Configuration

```c
// Join multicast group on DPDK port
struct rte_ether_addr mcast_addr;
// Convert IP multicast 233.1.2.3 to Ethernet multicast
mcast_addr.addr_bytes[0] = 0x01;
mcast_addr.addr_bytes[1] = 0x00;
mcast_addr.addr_bytes[2] = 0x5E;
mcast_addr.addr_bytes[3] = 0x01;  // Lower 23 bits of IP
mcast_addr.addr_bytes[4] = 0x02;
mcast_addr.addr_bytes[5] = 0x03;

rte_eth_dev_mac_addr_add(port, &mcast_addr, 0);

// For many multicast groups, enable promiscuous mode instead:
rte_eth_promiscuous_enable(port);
// Then filter in software (faster than NIC filter for many groups)
```

---

## 3. Solarflare ef_vi / OpenOnload

### 3.1 OpenOnload (Zero-Code-Change Acceleration)

```bash
# Install OpenOnload
tar xzf openonload-8.1.3.40.tgz
cd openonload-8.1.3.40
scripts/onload_install

# Run any application with acceleration — no code changes
onload ./trading_engine

# Or via LD_PRELOAD
LD_PRELOAD=libonload.so ./trading_engine

# Key environment variables
export EF_POLL_USEC=-1              # Busy-poll (never sleep)
export EF_SPIN_USEC=-1              # Spin instead of blocking
export EF_MULTICAST_LOOP_OFF=1      # Disable multicast loopback
export EF_UDP_RECV_SPIN=1           # Spin-wait on UDP recv
export EF_TCP_RECV_SPIN=1           # Spin-wait on TCP recv
export EF_INT_DRIVEN=0              # Disable interrupts (pure polling)
export EF_RXQ_SIZE=4096             # Increase RX queue depth
```

**OpenOnload Performance:**
- UDP multicast receive: ~1.5-3 μs (vs ~8-15 μs kernel)
- TCP round-trip: ~3-5 μs (vs ~15-25 μs kernel)
- Fully compatible with standard POSIX socket API

### 3.2 ef_vi (Raw Hardware API)

For sub-microsecond latency, bypass even OpenOnload's TCP/IP stack:

```c
#include <etherfabric/vi.h>
#include <etherfabric/pd.h>
#include <etherfabric/memreg.h>

struct ef_driver_handle dh;
struct ef_vi vi;
struct ef_pd pd;
struct ef_memreg mr;

// Initialization
void init_ef_vi(const char* interface) {
    // Open driver
    ef_driver_open(&dh);
    
    // Allocate protection domain
    ef_pd_alloc_by_name(&pd, dh, interface, EF_PD_DEFAULT);
    
    // Allocate virtual interface
    ef_vi_alloc_from_pd(&vi, dh, &pd, dh, 
                        -1,      // evq_capacity: use default
                        1024,    // rxq_capacity
                        512,     // txq_capacity  
                        NULL,    // evq: use built-in
                        -1,      // evq_dh
                        EF_VI_FLAGS_DEFAULT);
    
    // Allocate hugepage-backed buffer for DMA
    size_t buf_size = 4096 * 2048;  // 8 MB
    void* buf;
    posix_memalign(&buf, 4096, buf_size);
    
    // Register memory with NIC for DMA
    ef_memreg_alloc(&mr, dh, &pd, dh, buf, buf_size);
    
    // Post RX buffers
    for (int i = 0; i < 1024; i++) {
        ef_vi_receive_init(&vi, ef_memreg_dma_addr(&mr, i * 2048), i);
    }
    ef_vi_receive_push(&vi);
}

// Poll loop
void poll_loop() {
    ef_event events[64];
    
    while (running) {
        int n_events = ef_eventq_poll(&vi, events, 64);
        
        for (int i = 0; i < n_events; i++) {
            switch (EF_EVENT_TYPE(events[i])) {
                case EF_EVENT_TYPE_RX: {
                    int id = EF_EVENT_RX_RQ_ID(events[i]);
                    int len = EF_EVENT_RX_BYTES(events[i]);
                    
                    // Access packet data directly from DMA buffer
                    void* pkt = get_buffer(id);
                    process_packet(pkt, len);
                    
                    // Repost RX buffer
                    ef_vi_receive_init(&vi, ef_memreg_dma_addr(&mr, id * 2048), id);
                    ef_vi_receive_push(&vi);
                    break;
                }
                case EF_EVENT_TYPE_TX: {
                    // TX completion — buffer can be reused
                    int id = EF_EVENT_TX_RQ_ID(events[i]);
                    release_tx_buffer(id);
                    break;
                }
            }
        }
    }
}
```

**ef_vi Performance:**
- Wire-to-application: ~500-700 ns
- Application-to-wire: ~500 ns
- Hardware timestamp precision: <10 ns

---

## 4. Multicast Market Data Reception

### 4.1 Exchange Multicast Architecture

```
Exchange Matching Engine
        │
        ▼
    [Multicast Publisher]
        │
        ├──► Group A: 233.1.0.1:12001 (Equities A-F)
        ├──► Group B: 233.1.0.2:12002 (Equities G-M)
        ├──► Group C: 233.1.0.3:12003 (Equities N-S)
        ├──► Group D: 233.1.0.4:12004 (Equities T-Z)
        ├──► Group E: 233.1.1.1:13001 (Snapshot channel A)
        └──► Group F: 233.1.1.2:13002 (Snapshot channel B)

Subscriber Network:
        │
   [Switch with IGMP Snooping]
        │
    ┌───┼───┬───┐
    │   │   │   │
  Firm  Firm Firm ...
  A     B    C
  (only receives groups it subscribed to)
```

### 4.2 Optimal Multicast Configuration

```bash
# Kernel parameters for multicast
sysctl -w net.core.rmem_max=268435456        # 256 MB max receive buffer
sysctl -w net.core.rmem_default=16777216     # 16 MB default
sysctl -w net.core.netdev_max_backlog=30000  # Increase backlog queue
sysctl -w net.ipv4.igmp_max_memberships=1024 # Allow many multicast groups

# NIC offload settings
ethtool -K eth0 rx-checksum on              # HW checksum offload
ethtool -K eth0 gro off                     # Disable GRO (adds latency)
ethtool -K eth0 lro off                     # Disable LRO  
ethtool -C eth0 rx-usecs 0                  # Disable interrupt coalescing
ethtool -C eth0 rx-frames 1                 # Interrupt per packet
ethtool -G eth0 rx 4096                     # Increase ring buffer size

# NIC receive flow steering (pin multicast group to specific RX queue → specific CPU)
ethtool -N eth0 flow-type udp4 dst-ip 233.1.0.1 dst-port 12001 action 0
ethtool -N eth0 flow-type udp4 dst-ip 233.1.0.2 dst-port 12002 action 1
```

### 4.3 Gap Detection & Recovery

```cpp
class GapDetector {
    uint64_t expected_seq_ = 0;
    bool in_gap_ = false;
    uint64_t gap_start_tsc_ = 0;
    
    // Gap statistics
    uint64_t gap_count_ = 0;
    uint64_t total_missing_ = 0;
    uint64_t max_gap_size_ = 0;
    
public:
    enum class Result { OK, GAP_DETECTED, DUPLICATE };
    
    Result check(uint64_t seq) {
        if (seq == expected_seq_) {
            expected_seq_ = seq + 1;
            if (in_gap_) {
                in_gap_ = false;
                // Gap recovered — log duration
            }
            return Result::OK;
        }
        
        if (seq < expected_seq_) {
            return Result::DUPLICATE;
        }
        
        // Gap detected
        uint64_t missing = seq - expected_seq_;
        ++gap_count_;
        total_missing_ += missing;
        max_gap_size_ = std::max(max_gap_size_, missing);
        
        if (!in_gap_) {
            in_gap_ = true;
            gap_start_tsc_ = __rdtsc();
            // Trigger recovery: request retransmission or await snapshot
        }
        
        expected_seq_ = seq + 1;
        return Result::GAP_DETECTED;
    }
};
```

---

## 5. Network Hardware Selection

### 5.1 NIC Comparison

| Feature | Solarflare X3522 | Mellanox CX-7 | Intel E810 |
|---|---|---|---|
| **Kernel bypass** | ef_vi + OpenOnload | VMA + DPDK | DPDK only |
| **Wire-to-app** | ~500 ns | ~600 ns | ~1000 ns |
| **HW timestamp** | <10 ns precision | <10 ns | <100 ns |
| **Multicast** | HW filtering | HW filtering | HW filtering |
| **Price** | $$$$  | $$$ | $$ |
| **HFT adoption** | Dominant | Growing | Budget option |
| **OS support** | Linux | Linux, FreeBSD | Linux |

### 5.2 Switch Selection

| Switch | Port-to-Port Latency | Use Case |
|---|---|---|
| Arista 7130 (Metamako) | <50 ns (FPGA) | Lowest latency, HW timestamping, in-line FPGA |
| Arista 7050X4 | ~380 ns (cut-through) | Standard HFT network fabric |
| Arista 7280R3 | ~300 ns | Deep buffer for burst absorption |
| Cisco Nexus 3548 | ~250 ns | Legacy HFT |

### 5.3 Network Topology for HFT

```
               Exchange Cross-Connect
                      │
                      ▼
              ┌───────────────┐
              │  Arista 7130  │ ← Primary switch (FPGA-based, <50 ns)
              │  (Top of Rack)│
              └──┬───────┬────┘
                 │       │
         ┌───────▼─┐ ┌───▼──────┐
         │ Server 1│ │ Server 2 │
         │ (MD+Strat)│ │(OMS+Risk)│
         └─────────┘ └──────────┘
         
         NIC → PCIe → CPU → Strategy → PCIe → NIC → Switch → Cross-connect
              ~1μs    ~1μs    ~1μs      ~1μs   ~0.3μs  ~0.3μs
              ─────────────────────────────────────────────────
              Total: ~4-5 μs tick-to-trade
```

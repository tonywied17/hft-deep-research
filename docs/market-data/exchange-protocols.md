# Exchange Protocols Deep Dive

---

## 1. FIX Protocol (Financial Information eXchange)

### 1.1 Overview

FIX is the industry standard for order entry and execution reporting. Text-based (tag=value pairs). FIX 4.2 is most common for equities; FIX 5.0 SP2 is the latest standard.

### 1.2 Message Structure

```
8=FIX.4.2|9=178|35=D|49=SENDER|56=TARGET|34=12|52=20230901-14:30:00.123|
11=ORDER001|21=1|55=AAPL|54=1|38=1000|40=2|44=180.50|59=0|10=045|

Header:
  8  = BeginString (protocol version)
  9  = BodyLength
  35 = MsgType (D = New Order Single)
  49 = SenderCompID
  56 = TargetCompID
  34 = MsgSeqNum
  52 = SendingTime

Body:
  11 = ClOrdID (client order ID)
  21 = HandlInst (1=automated)
  55 = Symbol
  54 = Side (1=Buy, 2=Sell)
  38 = OrderQty
  40 = OrdType (1=Market, 2=Limit)
  44 = Price
  59 = TimeInForce (0=Day, 3=IOC, 6=GTD)

Trailer:
  10 = CheckSum (mod 256 of all bytes)
```

### 1.3 Key Message Types

| MsgType (35) | Name | Direction | Purpose |
|---|---|---|---|
| A | Logon | Both | Establish session |
| 0 | Heartbeat | Both | Keep-alive |
| 1 | Test Request | Both | Probe heartbeat |
| 2 | Resend Request | Both | Gap fill recovery |
| 5 | Logout | Both | Terminate session |
| D | New Order Single | Client→Exchange | Submit order |
| F | Order Cancel Request | Client→Exchange | Cancel order |
| G | Order Cancel/Replace | Client→Exchange | Modify order |
| 8 | Execution Report | Exchange→Client | Fill/ack/reject |
| 9 | Order Cancel Reject | Exchange→Client | Cancel rejected |

### 1.4 High-Performance FIX Parsing

FIX parsing is a critical hot path. Avoid `std::string`, `sscanf`, or any memory allocation:

```cpp
struct FIXField {
    uint16_t tag;
    const char* value;     // Pointer into raw buffer (zero-copy)
    uint16_t value_len;
};

class FastFIXParser {
    // Pre-built lookup table: tag → handler function
    using Handler = void(*)(const char* val, int len, void* ctx);
    Handler tag_handlers[10000] = {};  // Direct index by tag number
    
public:
    void register_handler(int tag, Handler h) {
        tag_handlers[tag] = h;
    }
    
    // Parse message in-place, zero allocation
    void parse(const char* msg, int len, void* ctx) {
        const char* p = msg;
        const char* end = msg + len;
        
        while (p < end) {
            // Parse tag number (digits before '=')
            uint16_t tag = 0;
            while (*p != '=' && p < end) {
                tag = tag * 10 + (*p - '0');
                ++p;
            }
            ++p;  // Skip '='
            
            // Find value (everything before SOH delimiter '\x01')
            const char* val_start = p;
            while (*p != '\x01' && p < end) ++p;
            int val_len = p - val_start;
            ++p;  // Skip SOH
            
            // Dispatch to handler
            if (tag < 10000 && tag_handlers[tag]) {
                tag_handlers[tag](val_start, val_len, ctx);
            }
        }
    }
    
    // Fast integer parsing (no stdlib)
    static int64_t parse_int(const char* s, int len) {
        int64_t result = 0;
        bool negative = false;
        int i = 0;
        
        if (s[0] == '-') { negative = true; i = 1; }
        
        for (; i < len; ++i) {
            result = result * 10 + (s[i] - '0');
        }
        
        return negative ? -result : result;
    }
    
    // Fast fixed-point price parsing: "180.50" → 1805000 (4 decimal places)
    static int64_t parse_price(const char* s, int len) {
        int64_t integer_part = 0;
        int64_t frac_part = 0;
        int frac_digits = 0;
        bool in_frac = false;
        
        for (int i = 0; i < len; ++i) {
            if (s[i] == '.') {
                in_frac = true;
                continue;
            }
            if (in_frac) {
                frac_part = frac_part * 10 + (s[i] - '0');
                frac_digits++;
            } else {
                integer_part = integer_part * 10 + (s[i] - '0');
            }
        }
        
        // Normalize to 4 decimal places
        while (frac_digits < 4) { frac_part *= 10; frac_digits++; }
        while (frac_digits > 4) { frac_part /= 10; frac_digits--; }
        
        return integer_part * 10000 + frac_part;
    }
};
```

### 1.5 FIX Session Management

```
Session layer handles:
  1. Sequence number tracking (MsgSeqNum)
  2. Gap detection (received seq > expected → request resend)
  3. Heartbeat monitoring (default 30s interval)
  4. PossDup handling (re-sent messages after gap fill)
  5. Message persistence (required for recovery)

Critical for HFT:
  - Pre-compute common message templates at startup
  - Use writev() for scatter-gather I/O (avoid copy)
  - Maintain pre-serialized header (SenderCompID, etc.)
  - Checksum: compute incrementally as you build the message
```

---

## 2. NASDAQ ITCH 5.0

### 2.1 Overview

Binary protocol for NASDAQ market data. Extremely efficient — no parsing overhead from text. Delivered via UDP multicast (Moldudp64 framing).

### 2.2 Message Types and Formats

All multi-byte integers are **big-endian**. All messages include:
- Stock Locate (2 bytes): Per-symbol identifier
- Tracking Number (2 bytes): Internal NASDAQ reference
- Timestamp (6 bytes): Nanoseconds since midnight

| Type | Byte | Size | Description |
|---|---|---|---|
| S | System Event | 12 | Market open/close events |
| R | Stock Directory | 39 | Ticker definitions |
| A | Add Order | 36 | New order (no MPID) |
| F | Add Order + MPID | 40 | New order with market participant |
| E | Order Executed | 31 | Partial/full fill |
| C | Executed w/ Price | 36 | Fill at different price |
| X | Order Cancel | 23 | Partial cancel |
| D | Order Delete | 19 | Full cancel |
| U | Order Replace | 35 | Modify (price or qty) |
| P | Trade | 44 | Non-displayable trade |
| Q | Cross Trade | 40 | opening/closing cross |
| B | Broken Trade | 19 | Trade break |

### 2.3 Add Order (Type 'A') Binary Layout

```
Offset  Size  Field
  0      1    Message Type = 'A' (0x41)
  1      2    Stock Locate
  3      2    Tracking Number
  5      6    Timestamp (nanoseconds since midnight)
 11      8    Order Reference Number
 19      1    Buy/Sell Indicator ('B' or 'S')
 20      4    Shares
 24      8    Stock (right-padded with spaces)
 32      4    Price (4 implied decimal places)
                Total: 36 bytes
```

### 2.4 High-Performance ITCH Parser

```cpp
struct __attribute__((packed)) ITCHAddOrder {
    uint8_t  msg_type;           // 'A'
    uint16_t stock_locate;       // Big-endian
    uint16_t tracking_number;
    uint8_t  timestamp[6];       // 6-byte big-endian nanoseconds
    uint64_t order_ref;          // Big-endian
    uint8_t  side;               // 'B' or 'S'
    uint32_t shares;             // Big-endian
    uint8_t  stock[8];           // Space-padded
    uint32_t price;              // Big-endian, 4 decimal places
};
static_assert(sizeof(ITCHAddOrder) == 36);

class ITCHParser {
    // Pre-allocated handlers per message type
    using MsgHandler = void(*)(const uint8_t* data, void* ctx);
    MsgHandler handlers[256] = {};
    
public:
    void set_handler(uint8_t msg_type, MsgHandler h) {
        handlers[msg_type] = h;
    }
    
    // Parse a stream of ITCH messages (after MoldUDP64 framing is removed)
    void process_packet(const uint8_t* data, size_t len) {
        const uint8_t* p = data;
        const uint8_t* end = data + len;
        
        while (p < end) {
            // MoldUDP64 message block: 2-byte big-endian length prefix
            uint16_t msg_len = (p[0] << 8) | p[1];
            p += 2;
            
            if (msg_len == 0 || p + msg_len > end) break;
            
            uint8_t msg_type = p[0];
            if (handlers[msg_type]) {
                handlers[msg_type](p, ctx);
            }
            
            p += msg_len;
        }
    }
};

// Example: Extract order reference from Add Order
inline uint64_t extract_order_ref(const uint8_t* data) {
    // Bytes 11-18, big-endian uint64
    return __builtin_bswap64(*reinterpret_cast<const uint64_t*>(data + 11));
}

inline uint32_t extract_price(const uint8_t* data) {
    // Bytes 32-35 for Add Order
    return __builtin_bswap32(*reinterpret_cast<const uint32_t*>(data + 32));
}
```

### 2.5 MoldUDP64 Transport

```
MoldUDP64 Packet Header (20 bytes):
  Bytes 0-9:    Session (10 ASCII characters)
  Bytes 10-17:  Sequence Number (uint64 big-endian)
  Bytes 18-19:  Message Count (uint16 big-endian)

Followed by N message blocks, each:
  2 bytes:  Message Length (uint16 big-endian)
  N bytes:  Message Data

Gap Detection:
  If received_seq > expected_seq → gap detected
  Request retransmission via TCP (SoupBinTCP rewind server)
```

---

## 3. CME MDP 3.0 (Market Data Platform)

### 3.1 Overview

CME Group's binary market data feed for futures and options. Uses **Simple Binary Encoding (SBE)** over UDP multicast.

### 3.2 SBE Message Structure

```
SBE Message Layout:
  [MessageHeader]     12 bytes
    BlockLength:      uint16 (size of root fields)
    TemplateId:       uint16 (message type ID)
    SchemaId:         uint16
    Version:          uint16
    NumGroups:        uint16
    NumVarData:       uint16
  [Root Fields]       Fixed-size based on template
  [Repeating Groups]  Variable number of repeated blocks
  [Variable Data]     Variable-length fields
```

### 3.3 Key Message Templates

| Template ID | Name | Description |
|---|---|---|
| 32 | MDIncrementalRefreshBook | Book updates (adds/modifies/deletes) |
| 33 | MDIncrementalRefreshDailyStatistics | Daily stats |
| 36 | MDIncrementalRefreshTradeSummary | Trade events |
| 38 | MDSnapshotFullRefresh | Full book snapshot |
| 46 | MDIncrementalRefreshOrderBook | Individual order updates |
| 4 | ChannelReset | Reset all instruments on channel |
| 12 | AdminHeartbeat | Keep-alive |

### 3.4 Incremental Refresh Processing

```cpp
struct CMEBookUpdate {
    int64_t price;          // Fixed-point (price * price_display_factor)
    int32_t qty;
    int32_t num_orders;
    uint8_t  level;         // Price level (0 = best)
    uint8_t  update_action; // 0=New, 1=Change, 2=Delete
    uint8_t  entry_type;    // 0=Bid, 1=Offer, 2=ImpliedBid, etc.
};

class CMEMDPHandler {
    // Instrument definition map
    struct InstrumentDef {
        uint32_t security_id;
        int32_t  price_display_factor;
        int64_t  min_price_increment;
    };
    
    std::array<InstrumentDef, 1'000'000> instruments;
    
    // Implied book handling
    // CME has both direct and implied quotes
    // Implied = derived from related instruments (e.g., spread legs)
    
    void on_incremental_refresh(const uint8_t* msg) {
        // Parse SBE header
        uint16_t block_length = read_u16_le(msg);
        uint16_t template_id = read_u16_le(msg + 2);
        
        if (template_id != 32) return;  // Only process book updates
        
        const uint8_t* p = msg + 12;  // Skip header
        
        // TransactTime (root field, 8 bytes)
        uint64_t transact_time = read_u64_le(p);
        p += 8;
        
        // MatchEventIndicator (1 byte)
        uint8_t match_event = *p++;
        bool last_in_batch = (match_event & 0x04) != 0;
        
        // Skip padding to align to block boundary
        p = msg + 12 + block_length;
        
        // Repeating group: MDEntries
        uint16_t group_block_len = read_u16_le(p);
        uint8_t num_entries = p[3];
        p += 4;  // Group header
        
        for (int i = 0; i < num_entries; ++i) {
            CMEBookUpdate update;
            update.price = read_i64_le(p);
            update.qty = read_i32_le(p + 8);
            update.num_orders = read_i32_le(p + 12);
            update.level = p[16];
            update.update_action = p[17];
            update.entry_type = p[18];
            
            uint32_t security_id = read_u32_le(p + 19);
            
            apply_update(security_id, update);
            
            p += group_block_len;
        }
        
        if (last_in_batch) {
            // All updates for this event applied — book is consistent
            // Signal strategy engine
        }
    }
};
```

### 3.5 CME Recovery Process

```
Recovery State Machine:

  1. SUBSCRIBING
     → Join instrument recovery channel (TCP snapshot)
     → Continue receiving incremental feed
     → Buffer incremental messages

  2. RECOVERING  
     → Receive full snapshot with RptSeq
     → Apply snapshot to book
     → Replay any buffered incrementals with RptSeq > snapshot.RptSeq

  3. SYNCHRONIZED
     → Normal incremental processing
     → On gap: return to state 1

  Important:
     - Match event indicator groups related updates
     - Don't update strategy between updates in same batch
     - RptSeq is per-instrument (not per-channel)
```

---

## 4. Other Protocols

### 4.1 OUCH (NASDAQ Order Entry)

Binary protocol for entering orders on NASDAQ. Simpler and faster than FIX.

```
Enter Order Message (49 bytes):
  Type:           'O'
  Token:          14 bytes (order identifier)
  Side:           'B' or 'S'
  Shares:         uint32
  Symbol:         8 bytes (space-padded)
  Price:          uint32 (4 decimal places)
  TimeInForce:    uint32
  Firm:           4 bytes
  Display:        'Y' or 'N'
  Capacity:       'A' (agency), 'P' (principal), 'R' (riskless)
  IntMktSweep:    'Y' or 'N' (ISO sweep)
  MinQty:         uint32
  CrossType:      'N' (no cross), 'O'(opening), 'C'(closing)
```

### 4.2 PITCH (Cboe)

Binary protocol for Cboe equities and options. Similar structure to ITCH.

```
Key differences from ITCH:
  - Order IDs are base-36 encoded (12 bytes)
  - Timestamps are 32-bit (microseconds since midnight) in header
  - Messages have 8-byte length-type header
  - Supports options-specific fields (strike, expiry)
```

### 4.3 NYSE Pillar

NYSE's current market data platform. Uses **XDP (eXchange Data Publisher)** binary format.

```
XDP Packet Header:
  PacketSize:    uint16
  DeliveryFlag:  uint8
  NumberMsgs:    uint8
  SeqNum:        uint32
  SendTime:      uint32 (nanoseconds)
  SendTimeNS:    uint32

Key Message Types:
  100 = SymbolIndex    (instrument definition)
  101 = SourceTime     (timing reference)
  320 = AddOrder       
  321 = ModifyOrder    
  322 = DeleteOrder    
  323 = OrderExecution
  220 = Trade          
```

### 4.4 Eurex EOBI (Enhanced Order Book Interface)

Binary feed for Eurex derivatives. Uses **ETI (Enhanced Trading Interface)** format.

### 4.5 Protocol Comparison

| Feature | FIX | ITCH | CME MDP 3.0 | OUCH |
|---|---|---|---|---|
| Format | Text | Binary | Binary (SBE) | Binary |
| Direction | Order entry + data | Data only | Data only | Order entry only |
| Typical Latency | 5-15 μs parse | < 100 ns parse | 200-500 ns parse | < 200 ns parse |
| Transport | TCP | UDP multicast | UDP multicast | TCP |
| Sequence Recovery | FIX resend | MoldUDP64 → TCP | Snapshot + replay | SoupBinTCP |
| Human Readable | Yes | No | No | No |
| Standard | Industry-wide | NASDAQ | CME Group | NASDAQ |

---

## 5. Protocol Performance Tips

### 5.1 Parsing Strategy

```
DO:
  ✓ Use SIMD for FIX delimiter scanning
  ✓ Direct-cast structs for binary protocols (avoid byte-by-byte)
  ✓ Pre-compute handler dispatch tables
  ✓ Use __builtin_bswap for endian conversion
  ✓ Zero-copy: parse in-place from network buffer
  ✓ Pre-filter by instrument before full parse

DON'T:
  ✗ Allocate memory during parsing
  ✗ Use std::string, std::map, std::unordered_map in hot path
  ✗ Copy message data unnecessarily
  ✗ Parse fields you don't need
  ✗ Use virtual dispatch for message handling
```

### 5.2 Timestamp Extraction

```cpp
// ITCH: 6-byte big-endian nanoseconds since midnight
inline uint64_t itch_timestamp(const uint8_t* p) {
    return ((uint64_t)p[0] << 40) | ((uint64_t)p[1] << 32) |
           ((uint64_t)p[2] << 24) | ((uint64_t)p[3] << 16) |
           ((uint64_t)p[4] << 8)  | p[5];
}

// CME MDP: 8-byte little-endian nanoseconds since epoch
inline uint64_t cme_timestamp(const uint8_t* p) {
    return *reinterpret_cast<const uint64_t*>(p);  // Already little-endian on x86
}
```

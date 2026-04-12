# SIMD Vectorization for HFT

## Overview

Single Instruction, Multiple Data (SIMD) processes multiple data elements in a single CPU cycle. In HFT, SIMD accelerates protocol parsing, price scanning, risk checks, and mathematical computations.

---

## 1. x86 SIMD Extension Landscape

| Extension | Register Width | Integer Ops | Float Ops | Year | Key CPUs |
|---|---|---|---|---|---|
| SSE2 | 128-bit (XMM) | 16×i8, 8×i16, 4×i32, 2×i64 | 4×f32, 2×f64 | 2001 | Pentium 4+ |
| SSE4.2 | 128-bit | + string ops, CRC32 | same | 2008 | Nehalem+ |
| AVX | 256-bit (YMM) | (128-bit only) | 8×f32, 4×f64 | 2011 | Sandy Bridge+ |
| AVX2 | 256-bit | 32×i8, 16×i16, 8×i32, 4×i64 | 8×f32, 4×f64 | 2013 | Haswell+ |
| AVX-512 | 512-bit (ZMM) | 64×i8, 32×i16, 16×i32, 8×i64 | 16×f32, 8×f64 | 2017 | Skylake-X+ |

**HFT Recommendation:** Target AVX2 as the baseline (all HFT-class hardware supports it). Use AVX-512 where available with runtime detection.

**Warning:** On some Intel CPUs (Ice Lake, Rocket Lake), using AVX-512 causes a clock frequency downshift that can hurt single-threaded performance. Benchmark carefully.

---

## 2. Protocol Parsing with SIMD

### 2.1 FIX Message Delimiter Search

FIX protocol uses SOH (0x01) as field delimiter. Finding delimiters is a critical inner loop:

```cpp
#include <immintrin.h>

// Find all SOH delimiters in a FIX message using AVX2
// Returns positions of all delimiters in the output array
int find_fix_delimiters_avx2(const char* data, int len, int* positions) {
    __m256i delim = _mm256_set1_epi8(0x01);  // SOH
    int count = 0;
    
    int i = 0;
    for (; i + 32 <= len; i += 32) {
        __m256i chunk = _mm256_loadu_si256((__m256i*)(data + i));
        __m256i cmp = _mm256_cmpeq_epi8(chunk, delim);
        uint32_t mask = _mm256_movemask_epi8(cmp);
        
        while (mask) {
            int bit = __builtin_ctz(mask);
            positions[count++] = i + bit;
            mask &= mask - 1;  // Clear lowest bit
        }
    }
    
    // Scalar remainder
    for (; i < len; ++i) {
        if (data[i] == 0x01) positions[count++] = i;
    }
    
    return count;
}
```

**Performance:** ~32 bytes per cycle vs ~1 byte per cycle scalar = **32x speedup**.

### 2.2 FIX Tag Number Parsing (SIMD Integer Conversion)

```cpp
// Parse a FIX tag number (e.g., "35=" → 35) using SIMD
// Handles tags up to 8 digits
uint32_t parse_tag_simd(const char* str, int len) {
    // Load digits
    __m128i raw = _mm_loadl_epi64((__m128i*)str);
    
    // Subtract '0' to get digit values
    __m128i zeros = _mm_set1_epi8('0');
    __m128i digits = _mm_sub_epi8(raw, zeros);
    
    // Multiply by positional weights: 10000000, 1000000, ..., 10, 1
    // Using widening multiply and horizontal add
    const __m128i mult10  = _mm_setr_epi8(10, 1, 10, 1, 10, 1, 10, 1, 0,0,0,0,0,0,0,0);
    __m128i pairs = _mm_maddubs_epi16(digits, mult10);
    
    const __m128i mult100 = _mm_setr_epi16(100, 1, 100, 1, 0, 0, 0, 0);
    __m128i quads = _mm_madd_epi16(pairs, mult100);
    
    // Final horizontal add depends on actual digit count
    // For common tags (1-4 digits), extract and combine
    uint32_t vals[4];
    _mm_storeu_si128((__m128i*)vals, quads);
    
    switch (len) {
        case 1: return vals[0] & 0xF;
        case 2: return (vals[0] >> 8) & 0xFF;  // Simplified; actual impl depends on layout
        case 3: return vals[0];
        default: return vals[0] * 10000 + vals[1];
    }
}
```

### 2.3 ITCH Message Type Detection

Process 32 ITCH messages's type bytes at once to determine handling:

```cpp
// Given a buffer of ITCH message headers, classify message types using SIMD
void classify_itch_messages_avx2(const uint8_t* types, int count, 
                                  uint8_t* is_add, uint8_t* is_exec, uint8_t* is_delete) {
    __m256i add_type = _mm256_set1_epi8('A');     // Add Order
    __m256i exec_type = _mm256_set1_epi8('E');    // Order Executed
    __m256i del_type = _mm256_set1_epi8('D');     // Order Delete
    
    for (int i = 0; i + 32 <= count; i += 32) {
        __m256i t = _mm256_loadu_si256((__m256i*)(types + i));
        
        __m256i add_mask = _mm256_cmpeq_epi8(t, add_type);
        __m256i exec_mask = _mm256_cmpeq_epi8(t, exec_type);
        __m256i del_mask = _mm256_cmpeq_epi8(t, del_type);
        
        _mm256_storeu_si256((__m256i*)(is_add + i), add_mask);
        _mm256_storeu_si256((__m256i*)(is_exec + i), exec_mask);
        _mm256_storeu_si256((__m256i*)(is_delete + i), del_mask);
    }
}
```

---

## 3. Price & Quantity Operations

### 3.1 Batch Price Comparison

Check 4 prices against a threshold simultaneously:

```cpp
// Check if any of 4 order prices exceed a limit
bool any_price_exceeds_avx2(const double* prices, double limit) {
    __m256d p = _mm256_loadu_pd(prices);
    __m256d l = _mm256_set1_pd(limit);
    __m256d cmp = _mm256_cmp_pd(p, l, _CMP_GT_OQ);
    return _mm256_movemask_pd(cmp) != 0;
}

// Find the best (minimum ask / maximum bid) price in an array
double find_best_ask_avx2(const double* prices, int n) {
    __m256d best = _mm256_set1_pd(std::numeric_limits<double>::max());
    
    int i = 0;
    for (; i + 4 <= n; i += 4) {
        __m256d p = _mm256_loadu_pd(&prices[i]);
        best = _mm256_min_pd(best, p);
    }
    
    // Horizontal min across the 4 lanes
    alignas(32) double vals[4];
    _mm256_store_pd(vals, best);
    double result = std::min({vals[0], vals[1], vals[2], vals[3]});
    
    // Scalar remainder
    for (; i < n; ++i) {
        result = std::min(result, prices[i]);
    }
    
    return result;
}
```

### 3.2 Volume-Weighted Average Price (VWAP) Calculation

```cpp
// Compute VWAP = sum(price * volume) / sum(volume) using AVX2
double vwap_avx2(const double* prices, const double* volumes, int n) {
    __m256d sum_pv = _mm256_setzero_pd();
    __m256d sum_v = _mm256_setzero_pd();
    
    int i = 0;
    for (; i + 4 <= n; i += 4) {
        __m256d p = _mm256_loadu_pd(&prices[i]);
        __m256d v = _mm256_loadu_pd(&volumes[i]);
        
        sum_pv = _mm256_fmadd_pd(p, v, sum_pv);  // FMA: p*v + sum_pv
        sum_v = _mm256_add_pd(sum_v, v);
    }
    
    // Horizontal reduction
    alignas(32) double pv[4], vv[4];
    _mm256_store_pd(pv, sum_pv);
    _mm256_store_pd(vv, sum_v);
    
    double total_pv = pv[0] + pv[1] + pv[2] + pv[3];
    double total_v = vv[0] + vv[1] + vv[2] + vv[3];
    
    // Scalar remainder
    for (; i < n; ++i) {
        total_pv += prices[i] * volumes[i];
        total_v += volumes[i];
    }
    
    return total_pv / total_v;
}
```

---

## 4. Risk Check Vectorization

### 4.1 Batch Position Limit Check

Check all positions against their limits in parallel:

```cpp
// Check 8 instrument positions against their limits simultaneously
// Returns bitmask of instruments that are in violation
uint32_t check_position_limits_avx2(const int32_t* positions, 
                                      const int32_t* limits, 
                                      int count) {
    uint32_t violations = 0;
    
    for (int i = 0; i + 8 <= count; i += 8) {
        __m256i pos = _mm256_loadu_si256((__m256i*)(positions + i));
        __m256i lim = _mm256_loadu_si256((__m256i*)(limits + i));
        
        // abs(position) > limit ?
        __m256i abs_pos = _mm256_abs_epi32(pos);
        __m256i cmp = _mm256_cmpgt_epi32(abs_pos, lim);
        
        uint32_t mask = _mm256_movemask_epi8(cmp);
        // Convert byte mask to per-element mask
        // Each int32 contributes 4 bytes to the mask
        for (int j = 0; j < 8; ++j) {
            if (mask & (0xF << (j * 4))) {
                violations |= (1u << (i + j));
            }
        }
    }
    
    return violations;
}
```

---

## 5. String Operations

### 5.1 Symbol Comparison

Compare 8-byte stock symbols using SIMD instead of `strcmp`:

```cpp
// Compare two 8-byte symbols (padded with spaces or nulls)
bool symbol_equals_sse(const char* a, const char* b) {
    __m128i va = _mm_loadl_epi64((__m128i*)a);
    __m128i vb = _mm_loadl_epi64((__m128i*)b);
    __m128i cmp = _mm_cmpeq_epi8(va, vb);
    // All 8 lower bytes must match — check lower 8 bits of mask
    return (_mm_movemask_epi8(cmp) & 0xFF) == 0xFF;
}

// Find a symbol in a sorted symbol array using SIMD
int find_symbol_avx2(const uint64_t* symbols_as_u64, int n, uint64_t target) {
    __m256i tgt = _mm256_set1_epi64x(target);
    
    for (int i = 0; i + 4 <= n; i += 4) {
        __m256i syms = _mm256_loadu_si256((__m256i*)(symbols_as_u64 + i));
        __m256i cmp = _mm256_cmpeq_epi64(syms, tgt);
        int mask = _mm256_movemask_epi8(cmp);
        if (mask) {
            // Find which of the 4 positions matched
            return i + (__builtin_ctz(mask) / 8);
        }
    }
    return -1;  // Not found
}
```

### 5.2 Byte Swap (Network to Host Order)

Exchange protocols send multi-byte fields in big-endian. SIMD can byte-swap multiple fields at once:

```cpp
// Byte-swap 4 × 64-bit values from big-endian to little-endian
void bswap64_avx2(uint64_t* data) {
    __m256i v = _mm256_loadu_si256((__m256i*)data);
    
    // Shuffle mask to reverse bytes within each 64-bit lane
    __m256i shuffle = _mm256_setr_epi8(
        7,6,5,4,3,2,1,0,   15,14,13,12,11,10,9,8,
        7,6,5,4,3,2,1,0,   15,14,13,12,11,10,9,8
    );
    
    v = _mm256_shuffle_epi8(v, shuffle);
    _mm256_storeu_si256((__m256i*)data, v);
}

// Byte-swap 8 × 32-bit values
void bswap32_avx2(uint32_t* data) {
    __m256i v = _mm256_loadu_si256((__m256i*)data);
    
    __m256i shuffle = _mm256_setr_epi8(
        3,2,1,0, 7,6,5,4, 11,10,9,8, 15,14,13,12,
        3,2,1,0, 7,6,5,4, 11,10,9,8, 15,14,13,12
    );
    
    v = _mm256_shuffle_epi8(v, shuffle);
    _mm256_storeu_si256((__m256i*)data, v);
}
```

---

## 6. Auto-Vectorization Guide

Sometimes you don't need intrinsics — the compiler can vectorize for you:

```cpp
// Compiler auto-vectorization hints

// 1. Use restrict pointers (no aliasing)
void add_arrays(double* __restrict__ out, 
                const double* __restrict__ a, 
                const double* __restrict__ b, int n) {
    for (int i = 0; i < n; ++i) {
        out[i] = a[i] + b[i];  // Compiler will auto-vectorize
    }
}

// 2. OpenMP SIMD pragma
#pragma omp simd
for (int i = 0; i < n; ++i) {
    prices[i] *= adjustment;
}

// 3. GCC vector extensions
typedef double v4d __attribute__((vector_size(32)));

v4d add_v4d(v4d a, v4d b) {
    return a + b;  // Compiles to vaddpd ymm
}

// 4. Compiler flags that enable auto-vectorization
// -O3 -march=native -ftree-vectorize -fopt-info-vec-optimized
// The last flag reports which loops were vectorized
```

### 6.1 What Prevents Auto-Vectorization

| Issue | Example | Fix |
|---|---|---|
| Pointer aliasing | `a[i] = b[i] + c[i]` where a might overlap b | Use `__restrict__` |
| Data dependencies | `a[i] = a[i-1] + 1` | Restructure algorithm |
| Non-unit stride | `a[i*3]` | Use gather instructions or restructure |
| Function calls in loop | `a[i] = expensive_fn(b[i])` | Inline the function |
| Conditional stores | `if (b[i]) a[i] = c[i]` | Use masked stores or `_mm256_blendv_pd` |
| Early exit | `if (found) break` | Process full batch, check after |
| Floating-point ordering | `sum += a[i]` (FP associativity) | Use `-ffast-math` or separate accumulators |

---

## 7. Runtime SIMD Dispatch

Detect CPU capabilities at runtime and dispatch to the optimal implementation:

```cpp
#include <cpuid.h>

enum class SIMDLevel { SSE2, AVX2, AVX512 };

SIMDLevel detect_simd() {
    uint32_t eax, ebx, ecx, edx;
    
    // Check AVX-512
    __cpuid_count(7, 0, eax, ebx, ecx, edx);
    if (ebx & (1 << 16))  // AVX-512F
        return SIMDLevel::AVX512;
    
    // Check AVX2
    if (ebx & (1 << 5))  // AVX2
        return SIMDLevel::AVX2;
    
    return SIMDLevel::SSE2;  // Always available
}

// Function pointer dispatch table
using FindDelimFn = int(*)(const char*, int, int*);

FindDelimFn find_delimiters;

void init_simd_dispatch() {
    switch (detect_simd()) {
        case SIMDLevel::AVX512: find_delimiters = find_fix_delimiters_avx512; break;
        case SIMDLevel::AVX2:   find_delimiters = find_fix_delimiters_avx2; break;
        default:                find_delimiters = find_fix_delimiters_scalar; break;
    }
}
```

---

## 8. Performance Measurement

```bash
# Count SIMD instruction throughput
perf stat -e \
    fp_arith_inst_retired.256b_packed_double,\
    fp_arith_inst_retired.256b_packed_single,\
    fp_arith_inst_retired.scalar_double \
    -- ./trading_engine --replay data.pcap

# Check vectorization rate
# Ratio of packed to (packed + scalar) = vectorization efficiency
```

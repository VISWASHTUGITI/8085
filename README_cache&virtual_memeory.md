# Computer Architecture Notes — Cache & Virtual Memory

A consolidated set of notes covering cache coherence, cache mapping techniques, replacement policies, cache optimization, and virtual memory (page tables / MMU / TLB).

---

## 1. Cache Coherence Problem

In multi-core systems, each core has its own local cache, but all cores share the same main memory. **Cache coherence** is the problem of keeping cached copies of shared data consistent across all these caches.

**Example of the problem:**
1. Core A and Core B both cache `x = 5`.
2. Core A updates `x` to 10 — but only in its own cache.
3. Core B still sees the stale value `x = 5`.

**A coherent system must guarantee:**
- **Write propagation** — updates eventually become visible to all caches holding a copy.
- **Write serialization** — all processors observe writes to a location in the same order.

**Solutions / Mechanisms:**

| Mechanism | Idea | Best for |
|---|---|---|
| **Snooping** | Caches "listen" to a shared bus and invalidate/update their copy when they see another cache write to a shared address | Small core counts |
| **MESI Protocol** | Each cache line tagged with a state: **M**odified, **E**xclusive, **S**hared, **I**nvalid | Most CPUs (including Intel) |
| **Directory-Based Protocols** | A central/distributed directory tracks which caches hold which memory blocks; only relevant caches are notified | Large multi-core/multi-socket systems |
| **Write-Invalidate** | On a write, all other cached copies are marked invalid | Most common in real CPUs |
| **Write-Update** | On a write, the new value is pushed to all caches holding it | Rare — wastes bandwidth |

Coherence is distinct from **memory consistency**: coherence concerns a *single* memory location; consistency concerns the *ordering* of operations across *multiple* locations.

---

## 2. Cache Mapping Techniques

### Why Mapping Is Needed
Cache is tiny (KB–MB) compared to RAM (GB), so a rule is needed to decide *where* in the cache a given memory block goes.

### The CPU Lookup Process (general flow)
```
CPU requests Address
        ↓
Split Address into Tag + Index + Offset
        ↓
Go to the Cache Line indicated by Index
        ↓
Compare stored Tag vs incoming Tag
        ↓
   Match?  YES → Hit, use cached data
            NO  → Miss, load from Main Memory
```

### 2.1 Direct Mapping

**Definition:** Each memory block maps to exactly **one** fixed cache line.

**Formula:** `Cache Line = (Block Address) MOD (Number of Cache Lines)`

**Address format:**
```
| TAG | INDEX | OFFSET |
```
- Offset bits = log2(Block Size)
- Index bits = log2(Number of Cache Lines)
- Tag bits = Total bits − Index − Offset

**Worked example** (8 lines, 4-byte blocks, 16-bit address):
```
Offset = log2(4) = 2 bits
Index  = log2(8) = 3 bits
Tag    = 16 - 3 - 2 = 11 bits
```

**Advantages**
- Fastest lookup — only 1 comparator needed
- Cheapest to build (low power, low heat)
- Simple to implement

**Disadvantages**
- **Conflict misses / thrashing**: two blocks mapping to the same line keep evicting each other, even if other lines are empty
- Poor cache utilization

| Feature | Rating |
|---|---|
| Speed | ⭐⭐⭐⭐⭐ Fastest |
| Hardware Cost | ⭐⭐⭐⭐⭐ Cheapest |
| Flexibility | ⭐ Lowest |
| Conflict Misses | ⭐ Worst |
| Cache Utilization | ⭐⭐ Poor |

---

### 2.2 Fully Associative Mapping

**Definition:** A memory block can go into **any** cache line — no fixed rule.

**Address format:**
```
| TAG | OFFSET |
```
- No Index field at all
- Tag = Total bits − Offset (much larger than in Direct Mapping)

**Worked example** (8 lines, 4-byte blocks, 16-bit address):
```
Offset = log2(4) = 2 bits
Index  = 0 bits (none)
Tag    = 16 - 0 - 2 = 14 bits
```

**How lookup works:** Tag is compared against **every line simultaneously** (parallel comparators).

**Advantages**
- No conflict misses — thrashing (from mapping restrictions) cannot happen
- Best possible cache utilization / hit rate
- Most flexible replacement policy choice

**Disadvantages**
- Very slow — needs one comparator per line, running in parallel
- Most expensive hardware (e.g. 1000 lines = 1000 comparators)
- Complex replacement logic (e.g. true LRU is expensive to implement)
- High power consumption / heat

| Feature | Rating |
|---|---|
| Speed | ⭐ Slowest |
| Hardware Cost | ⭐ Most Expensive |
| Flexibility | ⭐⭐⭐⭐⭐ Highest |
| Conflict Misses | ⭐⭐⭐⭐⭐ None |
| Cache Utilization | ⭐⭐⭐⭐⭐ Best |

---

### 2.3 Set-Associative Mapping (the real-world compromise)

**Definition:** Cache is divided into **sets**, each containing multiple lines ("ways"). A block maps to one fixed **set** (like Direct), but can sit in **any line within that set** (like Fully Associative).

**Address format:**
```
| TAG | SET INDEX | OFFSET |
```
- Offset bits = log2(Block Size)
- Set Index bits = log2(Number of Sets)
- Tag bits = Total bits − Set Index − Offset
- Number of Sets = Total Cache Lines ÷ Ways (Associativity)

**Worked example** (8 lines total, 4-way associative, 4-byte blocks, 16-bit address):
```
Number of Sets = 8 ÷ 4 = 2 Sets
Offset    = log2(4) = 2 bits
Set Index = log2(2) = 1 bit
Tag       = 16 - 1 - 2 = 13 bits
```

**How lookup works:** Set Index picks the set; Tag is compared against only the lines **within that set** (e.g. 4 comparators for 4-way).

**Advantages**
- Drastically reduces conflict misses vs Direct Mapping (multiple competing blocks can coexist in a set)
- Better cache utilization than Direct Mapping
- Cheaper than Fully Associative — only checks tags within one set
- Good balance of speed and flexibility → **used in nearly all real CPU caches**

**Disadvantages**
- More complex than Direct Mapping (multiple simultaneous tag checks, per-set LRU)
- Still has some conflict misses if more blocks map to a set than it has lines
- Slightly slower than Direct Mapping

| Feature | Rating |
|---|---|
| Speed | ⭐⭐⭐⭐ Fast |
| Hardware Cost | ⭐⭐⭐ Medium |
| Flexibility | ⭐⭐⭐⭐ Good |
| Conflict Misses | ⭐⭐⭐⭐ Few |
| Cache Utilization | ⭐⭐⭐⭐ Good |

**Note:** Direct Mapping = "1-way set associative"; Fully Associative = "N-way set associative" (N = total lines, i.e. one giant set). Set-Associative is the general case.

### The Golden Formula (all techniques)
```
Total Address Bits = Tag Bits + Index/Set Bits + Offset Bits

Offset = log2(Block Size)
Index  = log2(Number of Lines or Sets)
Tag    = Total Bits - Index - Offset
```

### Final Comparison — All Three Mapping Techniques

| Feature | Direct Mapped | Set Associative | Fully Associative |
|---|---|---|---|
| Where can a block go? | Only 1 specific line | Any line in a specific set | Any line anywhere |
| Speed | Fastest | Fast | Slowest |
| Hardware Cost | Cheapest | Medium | Most Expensive |
| Conflict Misses | Most | Some | None |
| Cache Utilization | Poor | Good | Best |
| Complexity | Simple | Medium | Very Complex |
| Used in Real CPUs? | Rarely | Yes (most common) | Only small caches (e.g. TLB) |

**Bottom line:**
- Want speed + low cost → Direct Mapping
- Want best performance, cost no object → Fully Associative
- Want best real-world balance → Set-Associative (4-way / 8-way) — what Intel/AMD actually use for L1/L2/L3

---

## 3. Cache Replacement Techniques

### Why Needed
Only relevant when there's a **choice** of where a block can go:
- **Fully Associative** → choose from all lines
- **Set Associative** → choose from lines within a set
- **Direct Mapping** → NOT needed (only one possible location)

### 3.1 FIFO (First In First Out)
**Idea:** The block that entered the cache first is evicted first (queue-based).

**Advantages:** Very simple, low hardware cost, fair (equal time for every block).

**Disadvantages:**
- Ignores usage frequency — a heavily-used block can still get evicted just because it arrived first
- Subject to **Belady's Anomaly**: increasing cache size can sometimes *increase* miss count

| Feature | Rating |
|---|---|
| Simplicity | ⭐⭐⭐⭐⭐ |
| Performance | ⭐⭐ Poor |
| Hardware Cost | ⭐⭐⭐⭐⭐ Cheapest |
| Considers Usage? | No |

### 3.2 LRU (Least Recently Used)
**Idea:** Evict the block that hasn't been accessed for the longest time. Based on **Temporal Locality**.

**Hardware implementation:** Each line has a counter/timestamp; on access, reset to 0 and increment all others; evict the line with the highest counter.

**Advantages:** Best practical performance, matches real program behavior, no Belady's Anomaly, more cache space always helps or is neutral.

**Disadvantages:** More complex/expensive hardware — tracking counters for every line doesn't scale well for large fully-associative caches (approximations used instead).

| Feature | Rating |
|---|---|
| Simplicity | ⭐⭐⭐ Medium |
| Performance | ⭐⭐⭐⭐⭐ Best |
| Hardware Cost | ⭐⭐⭐ Medium |
| Considers Usage? | Yes (Recency) |

### 3.3 LFU (Least Frequently Used)
**Idea:** Evict the block used the **fewest number of times** (frequency counter, not recency).

**Advantages:** Good when some data is accessed very frequently; protects "hot" blocks; generally better than FIFO.

### Final Comparison — Replacement Techniques

| Technique | Core Idea | Performance | Cost | Real-World Use? |
|---|---|---|---|---|
| FIFO | Kick out oldest loaded block | Poor | Cheapest | Rarely |
| LRU | Kick out least recently used | Best Practical | Medium | Yes (most common) |
| LFU | Kick out least frequently used | Medium | Medium | Sometimes |

---

## 4. Cache Optimization

### Goals
1. Increase **Hit Rate**
2. Reduce **Miss Rate**
3. Reduce **Miss Penalty** (recovery time on a miss)
4. Reduce **Hit Time**

### The 3 C's of Cache Misses
| Type | Cause | Notes |
|---|---|---|
| **Compulsory (Cold) Miss** | First-ever access to data | Unavoidable |
| **Capacity Miss** | Cache too small to hold all needed data | Happens even with Fully Associative |
| **Conflict Miss** | Two blocks fight over the same line while other lines sit empty | Only in Direct / Set-Associative |

### Category 1 — Reducing Miss Rate

**Increase Block Size**
- Loads more neighboring data per fetch → exploits spatial locality → reduces compulsory misses
- Trade-off: too large → fewer blocks fit (more capacity misses), longer load time (more miss penalty), cache pollution
- Sweet spot: ~64 bytes (modern CPUs)

**Increase Cache Size**
- Directly reduces capacity misses
- Trade-off: more expensive, more chip space, more power, slightly slower search
- Why L1 is small-but-fast and L3 is large-but-slower

**Higher Associativity**
- Reduces conflict misses (e.g. Direct → 2-way cuts miss rate ~20–30%; 2-way → 4-way another ~10–15%; diminishing returns past 8-way)
- Trade-off: more complex hardware, more comparators, slightly slower hit time

**Victim Cache**
- Small, fully-associative cache between L1 and L2 that catches recently evicted blocks
- Gives evicted blocks a "second chance" without a full trip to L2
- Small (4–16 entries), effective against conflict misses, adds controller complexity

**Prefetching**
- Loads data before the CPU actually requests it
- Types: Hardware (pattern detection), Software (compiler/programmer prefetch instructions), Sequential (always load next block)
- Can hide miss penalty entirely for sequential patterns; wrong predictions waste bandwidth and pollute cache

### Category 2 — Reducing Miss Penalty

**Multi-Level Cache (L1/L2/L3)**
- Instead of going straight to main memory on a miss, check L2, then L3
- Typical characteristics:

| Level | Size | Speed |
|---|---|---|
| L1 | 32–64 KB | 4–5 cycles |
| L2 | 256 KB–1 MB | 10–15 cycles |
| L3 | 8–32 MB | 30–40 cycles |

**AMAT (Average Memory Access Time) formula:**
```
AMAT = HitTime(L1) + MissRate(L1) × 
       [HitTime(L2) + MissRate(L2) × 
       [HitTime(L3) + MissRate(L3) × MainMemoryTime]]
```

**Critical Word First**
- Deliver the specific word the CPU needs *first*, then load the rest of the block in the background
- CPU doesn't wait for the whole block

**Early Restart**
- As soon as the needed word arrives (even mid-block-load), send it to the CPU immediately

### Category 3 — Reducing Hit Time

**VIPT (Virtually Indexed, Physically Tagged)**
- Uses virtual address bits to find the set *while simultaneously* translating to a physical address for tag comparison
- Saves translation delay; used in most modern high-performance CPUs
- Must handle aliasing issues

**Pipelining the Cache**
- Breaks cache access into pipeline stages so multiple requests overlap
- Increases throughput but increases single-request latency

---

## 5. Virtual Memory: Page Tables, MMU, and TLB

### Core Components
| Component | What it is |
|---|---|
| **Page Table** | OS-managed data structure stored in **RAM**, maps Page → Frame |
| **MMU (Memory Management Unit)** | Hardware that performs address translation |
| **CR3 register** (x86-64) | Special CPU register holding the physical address of the *current process's* page-table root |
| **TLB (Translation Lookaside Buffer)** | Small hardware cache inside/near the CPU that stores recently used translations — **not created by the OS**, populated automatically by the MMU |

### How Translation Works

**Step 1 — Context switch:** OS loads the new process's page-table root address into CR3.
```
Process A running → CR3 → A's page table
        (context switch)
Process B running → CR3 → B's page table
```

**Step 2 — Address split:** CPU generates a virtual address, split into Page Number + Offset.

**Step 3 — TLB check:**
```
Virtual Address → MMU → TLB?
                        /    \
                     Hit     Miss
                      ↓        ↓
                 Physical   Page-table
                 Address     walk
```

- **TLB Hit** → translation obtained immediately (Page → Frame already cached)
- **TLB Miss** → must consult the page table in RAM

### Resolving a TLB Miss (Page-Table Walk)

Key insight: on a TLB miss, the MMU isn't starting from nothing — it **already knows the physical address of the page-table root** via CR3.

```
CR3 = 100000  → "page-table root is at physical address 100000"

PTE address = Page Table Base + (Page Number × PTE Size)
Example: 100000 + (5 × 8) = 100040

MMU reads PTE at 100040 → e.g. "Page 5 → Frame 20"
Physical Address = Frame 20 + Offset
```

So the correct mental model is:
```
Virtual Address → Page-Table Walk → Physical Frame → Physical Address
```
(not simply "virtual → physical" in one step, on a TLB miss)

### Multi-Level Page Tables (x86-64)
Real systems don't use a single flat page table — they use multiple levels to save space:
```
Virtual Address
   → PML4 index → Page-Table Level 4
   → PDPT index → Page Directory Pointer Table
   → PD index   → Page Directory
   → PT index   → Page Table
   → Page Frame → Physical Address
```
The MMU walks through these levels using physical addresses stored in each level's entries.

### Key Clarification
> "OS transfers the page table from RAM into the TLB" — **this is incorrect.**

Correct model:
- The **OS** creates and maintains the page tables (in RAM).
- The **MMU** consults those tables during translation.
- The **TLB** is a hardware cache that stores individual translations as they're resolved — much smaller than the full page table (which can have thousands/millions of entries).

---

## 6. Reference: Your Example CPU (Intel Core i5-1335U)

From a real Task Manager screenshot:
- **13th Gen Intel Core i5-1335U**
- **10 Cores** total: 2 Performance-cores (P-cores) + 8 Efficiency-cores (E-cores)
- **12 Logical Processors (threads)**: P-cores support Hyper-Threading (2 × 2 = 4 threads) + E-cores do not (8 × 1 = 8 threads) → 4 + 8 = 12
- Base speed: 1.30 GHz, boosts up to ~4.60 GHz
- Cache: L1 = 928 KB, L2 = 6.5 MB, L3 = 12.0 MB (shared across cores)
- Uses **Set-Associative** caching at all levels (like virtually all modern CPUs)

**AMD comparison:** AMD Ryzen CPUs (pre-hybrid designs) typically use uniform cores (no P-core/E-core split) with SMT (AMD's equivalent of Hyper-Threading) — e.g. Ryzen 5 7530U = 6 cores × 2 threads (SMT) = 12 threads, with simpler, more predictable per-core performance.

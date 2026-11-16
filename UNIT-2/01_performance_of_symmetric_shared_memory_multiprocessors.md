# Unit 2 — Part 1: Performance of Symmetric Shared-Memory Multiprocessors

---

## 1. Multiprocessor Cache Performance: The Two Components

In a multicore system or symmetric shared-memory multiprocessor (SMP) using a snooping coherence protocol, overall cache performance is governed by two interacting traffic sources:

```
                            Total Cache Misses
                                    │
         ┌──────────────────────────┴──────────────────────────┐
         ▼                                                     ▼
1. Uniprocessor Misses (The 3 C's)                   2. Coherence Misses
   • Compulsory (Cold)                                  (Interprocessor Communication)
   • Capacity                                           • True Sharing Misses
   • Conflict                                           • False Sharing Misses
```

1. **Uniprocessor Cache Misses:** The traditional "Three C's":
   * **Compulsory (Cold) Misses:** First-time access to a memory block.
   * **Capacity Misses:** The cache cannot hold all blocks needed during execution.
   * **Conflict (Collision) Misses:** Multiple blocks compete for the same set in a direct-mapped or set-associative cache.
2. **Coherence Misses:** Misses arising directly from interprocessor communication and the cache invalidation mechanism required to maintain coherent memory.

Changing the **processor count**, **cache size**, or **cache block size** affects these miss components in fundamentally different ways.

---

## 2. Coherence Misses: True Sharing vs. False Sharing

Coherence misses in invalidation-based shared-memory protocols fall into two distinct categories:

```
                                Coherence Misses
                                       │
            ┌──────────────────────────┴──────────────────────────┐
            ▼                                                     ▼
   True Sharing Misses                                   False Sharing Misses
• Same data word is shared                            • Different data words in the SAME block
• Communication is REAL                               • Words are INDEPENDENT
• Occurs even if block size = 1 word                  • Miss disappears if block size = 1 word
```

### 2.1 True Sharing Misses
* **Definition:** Misses that result from the **actual communication of shared data** among processors through memory.
* **Mechanism:**
  * When processor $P1$ writes to a shared cache block, it sends an invalidation to establish exclusive ownership of the block.
  * When processor $P2$ subsequently reads or writes that modified word, a cache miss occurs in $P2$, and the updated block is transferred from $P1$.
* **Key Characteristic:** A true sharing miss occurs **regardless of block size**. Even if the block size were reduced to a single word, the miss would still occur because the data value itself is being communicated.

---

### 2.2 False Sharing Misses
* **Definition:** Misses that occur because an invalidation-based protocol manages coherence on a **per-block basis** (with a single valid bit per cache block), rather than on a per-word basis.
* **Mechanism:**
  * A block in processor $P2$'s cache is invalidated because processor $P1$ wrote to word $x$ in that block.
  * However, processor $P2$ only wants to read or write word $y$ in that same block ($x \ne y$).
  * Processor $P2$ suffers a cache miss on its next access to word $y$, even though the value of word $y$ was never modified by $P1$!
* **Key Characteristic:** In a false sharing miss, **the cache block is shared, but no individual variable/word within the block is actually shared**. If the cache block size were reduced to a single word, **the miss would completely disappear**.

---

### 2.3 Detailed Step-by-Step Trace Example (Textbook Example)

> **Scenario:** Assume words $z1$ and $z2$ reside in the **same cache block**, which is initially in the **Shared** state in the private caches of both processor $P1$ and processor $P2$.  
> The following sequence of events occurs in chronological order:

```
Initial State: Cache block containing [ z1 | z2 ] is SHARED in both P1 and P2 caches.
```

| Time Step | Processor | Action | Event Classification | Detailed Explanation & State Transitions |
| :---: | :---: | :---: | :---: | :--- |
| **1** | **$P1$** | **Write $z1$** | **True Sharing Miss** | The block is shared in $P2$'s cache. To write $z1$, $P1$ must invalidate the block in $P2$ to gain exclusive ownership. Because $P1$ writes a word that was shared, this is a true sharing invalidation/miss. |
| **2** | **$P2$** | **Read $z2$** | **False Sharing Miss** | $P2$ experiences a cache miss because the block was invalidated in Step 1. However, $P1$ wrote to $z1$, and $P2$ is reading $z2$! The value of $z2$ was unchanged. If the block size were 1 word, $z2$ would still be in $P2$'s cache. |
| **3** | **$P1$** | **Write $z1$** | **False Sharing Miss** | When $P2$ read $z2$ in Step 2, the block was loaded back into both caches in the **Shared** state. Now $P1$ must invalidate $P2$ again to write $z1$. This invalidation is false sharing because $P2$ never read or needed $z1$. *(Handled as a write miss / upgrade request).* |
| **4** | **$P2$** | **Write $z2$** | **False Sharing Miss** | The block was marked Modified/Exclusive in $P1$ during Step 3. When $P2$ writes $z2$, it causes a miss to invalidate $P1$'s block, even though $P1$ is not using $z2$. |
| **5** | **$P2$** | **Read $z2$** | **True Sharing Miss / Hit** | $P2$ reads $z2$, which it just wrote in Step 4. If $P2$ still holds the modified block, this is a local hit. If evicted or downgraded, the value read was produced by $P2$ itself. |

---

## 3. Case Study 1: Performance of a Commercial Workload (OLTP)

To understand how these misses behave in real-world servers, Hennessy & Patterson analyze an **Online Transaction Processing (OLTP)** workload modeled after **TPC-B** using the Oracle database system on a 4-processor symmetric multiprocessor.

### 3.1 Experimental Baseline System: AlphaServer 4100 vs. Intel Core i7
The study evaluates the memory hierarchy of the AlphaServer 4100 (Alpha 21164 4-issue superscalar, 300 MHz) and compares its structural cache hierarchy with modern processors like the Intel Core i7:

| Cache Level | Architectural Characteristic | AlphaServer 4100 (Alpha 21164) | Intel Core i7 (6700 Skylake) |
| :--- | :--- | :--- | :--- |
| **Level 1 (L1)** | **Size** | 8 KB I-cache / 8 KB D-cache | 32 KB I-cache / 32 KB D-cache |
| | **Associativity** | Direct-mapped | 8-way set associative |
| | **Block Size** | 32 Bytes | 64 Bytes |
| | **Miss Penalty** | 7 clock cycles | 10 clock cycles |
| **Level 2 (L2)** | **Size** | 96 KB Unified | 256 KB Unified |
| | **Associativity** | 3-way set associative | 8-way set associative |
| | **Block Size** | 32 Bytes | 64 Bytes |
| | **Miss Penalty** | 21 clock cycles | 35 clock cycles |
| **Level 3 (L3)** | **Size** | 2 MiB per core (8 MiB unshared) | 2 MiB per core (8 MiB shared) |
| | **Associativity** | Direct-mapped (or 2-way) | 16-way set associative |
| | **Block Size** | 64 Bytes | 64 Bytes |
| | **Miss Penalty** | 80 clock cycles | 100 clock cycles |
| **Remote Transfer**| **Cache-to-Cache Transfer** | $\ge 125\text{ clock cycles}$ | $\ge 125\text{ clock cycles}$ |

---

### 3.2 Impact of Increasing L3 Cache Size (1 MiB to 8 MiB)

In commercial OLTP workloads, processors spend up to **90% of execution cycles stalled**, with memory accesses responsible for over half of all stall cycles.

```
       Memory Access Cycles Contributed per Instruction (Shift in Misses)
1 MiB L3 Cache:
┌─────────────────────────────────┬───────────────────┬──────────────┐
│  Capacity & Conflict Misses     │Instruction Misses │Coherence     │
│  (DOMINANT: ~60-70%)            │(~20%)             │(~10-15%)     │
└─────────────────────────────────┴───────────────────┴──────────────┘
                                     │
                                     ▼ (Increase L3 size to 4-8 MiB)
4-8 MiB L3 Cache:
┌───────────┬─────────────┬──────────────────────────────────────────┐
│ Capacity  │ Instruction │ True Sharing Misses                      │
│ (SHRUNK)  │ (SHRUNK)    │ (DOMINANT & UNCHANGED: ~70-80%)          │
└───────────┴─────────────┴──────────────────────────────────────────┘
```

#### Observations from Figure 5.10 & 5.11:
1. **Diminishing Returns Beyond 2 MiB:** Increasing L3 from 1 MiB to 2 MiB yields substantial performance gains. However, expanding L3 from 2 MiB to 4 MiB or 8 MiB produces **almost negligible additional performance improvement**.
2. **Why Bigger Caches Stop Helping:**
   * Increasing cache capacity eliminates almost all uniprocessor capacity and conflict misses.
   * However, **True Sharing Misses and False Sharing Misses are completely unaffected by cache size**!
   * Once capacity misses are eliminated, coherence misses become the overwhelming bottleneck. Because true sharing misses cannot be reduced by simply adding more cache memory, execution time plateaus.

---

### 3.3 Impact of Increasing Processor Count (Figure 5.12)
* When the processor count is scaled (e.g., from 1 to 4 to 8 processors):
  * Uniprocessor capacity and conflict misses remain constant or drop slightly per processor.
  * **True sharing misses increase substantially**, roughly scaling with the number of communicating cores.
  * Consequently, the total memory access cycles per instruction **increase as more processors are added**, eroding multi-core scaling efficiency.

---

### 3.4 Impact of Increasing L3 Cache Block Size (32 to 256 Bytes)

Varying the L3 block size from 32 bytes to 64, 128, and 256 bytes affects the four miss components differently:

```
                            Increasing Block Size (32B ──► 256B)
                                              │
         ┌────────────────────────┬───────────┴───────────┬────────────────────────┐
         ▼                        ▼                       ▼                        ▼
1. True Sharing Misses   2. Compulsory Misses    3. Capacity/Conflict     4. False Sharing Misses
   DROPS by > 2x            DROPS significantly     DROPS slightly (1.26x)   NEARLY DOUBLES
   (Spatial locality in     (Brings more data)      (Low spatial locality    (More independent
    shared variables)                                in uniprocessor misses)  words in one block)
```

1. **True Sharing Misses Decrease by $> 2\times$:** Indicates significant **spatial locality** in shared data structures (e.g., records, locks, and task queues).
2. **Compulsory Misses Decrease Substantially:** Standard spatial locality benefit.
3. **Capacity / Conflict Misses Show Only Minor Reduction ($1.26\times$):** Database and OLTP workloads exhibit low spatial locality in their private uniprocessor data references.
4. **False Sharing Misses Nearly Double:** Larger blocks increase the likelihood that unrelated variables used by different cores pack into the same 256-byte cache line.
5. **Instruction Misses Remain High:** Database workloads have large instruction footprints with short basic blocks and frequent branching, limiting instruction spatial locality.

#### L3 Miss Penalty Multiplier Relative to a 32-Byte Block:
As block size increases, transferring more bytes over the memory bus increases the miss latency:

$$\text{Effective Penalty} = \text{Base Penalty (32B)} \times \text{Multiplier}$$

| Block Size | Miss Penalty Multiplier Relative to 32-Byte Block |
| :---: | :---: |
| **32 Bytes** | **$1.00$** (Baseline) |
| **64 Bytes** | **$1.19$** |
| **128 Bytes** | **$1.36$** |
| **256 Bytes** | **$1.52$** |

*Because the miss rate drops by a factor of 2 while the miss penalty increases by only $1.36\times$, modern architectures (like the Intel Core i7) adopt **64-byte or 128-byte block sizes** as an optimal compromise.*

---

## 4. Case Study 2: Multiprogramming and OS Workloads

To evaluate multi-threaded multiprocessor behavior outside databases, Hennessy & Patterson analyze a **multiprogrammed software development workload**: a parallel compilation of the **Andrew Benchmark** running parallel `make` across 8 processors.

### 4.1 Workload Execution Characteristics (Figure 5.14)
* **Execution Time Breakdown:**
  * **User Execution:** **$27\%$**
  * **Kernel (OS) Execution:** **$7\%$**
  * **Synchronization Wait:** **$2\%$**
  * **Processor Idle (waiting for disk I/O):** **$64\%$**
* **Instruction Count vs. Time Spent:**
  * User code executes **$69\%$** of all instructions (over 8 times more than the kernel).
  * Yet, user code takes only **$27\%$** of execution time.
  * **Why?** Operating system kernel instructions experience significantly higher cache miss rates than user instructions!

```
Instructions Executed:                      Execution Time:
┌────────────────────────────┬────┬───┐     ┌──────────┬────┬──┬────────────────────────┐
│ User Execution (69%)       │OS  │Syn│     │User (27%)│OS  │S │ Idle (Waiting for I/O) │
│                            │(27)│(3)│     │          │(7%)│  │ (64%)                  │
└────────────────────────────┴────┴───┘     └──────────┴────┴──┴────────────────────────┘
```

---

### 4.2 Why the Operating System Causes Disproportionately High Cache Misses

The OS kernel has a data miss rate **$5\times$ to $6\times$ higher** than user applications due to three factors:
1. **Lack of Locality & Large Code Size:** OS routines (exception handlers, virtual memory managers, device drivers) are invoked infrequently across scattered addresses.
2. **Page Zeroing / Allocation (High Compulsory Misses):** Before allocating a physical page to a process, the OS kernel writes zeros across all 4 KB bytes to prevent security leaks. This flushes caches and generates massive compulsory cold misses.
3. **Shared OS Data Structures (High Coherence Misses):** The kernel manages centralized process tables, run queues, and buffer caches that are constantly read and written by all 8 processors, creating high coherence traffic.

---

### 4.3 Sensitivity to L1 Cache Size and Block Size (Figure 5.15 & 5.16)

```
                            L1 Data Cache Variations
                                       │
         ┌─────────────────────────────┴─────────────────────────────┐
         ▼                                                           ▼
Increasing L1 Cache Size (32 KB ──► 256 KB)         Increasing L1 Block Size (16B ──► 128B)
• User Miss Rate drops by almost 3x                 • Kernel Miss Rate drops by nearly 4x
• Kernel Miss Rate drops by only 1.3x               • User Miss Rate drops by nearly 3x
• Reason: Kernel dominated by Compulsory            • Reason: Larger blocks eliminate
  and Coherence misses (unaffected by size)           compulsory page-zeroing misses
```

#### Kernel Miss Breakdown Shift (Figure 5.16):
* **When L1 Cache Size increases:**
  * Compulsory misses stay flat.
  * Capacity/conflict misses drop by more than $2\times$.
  * **Coherence misses nearly double** (as cache size grows, lines stay resident longer, increasing the probability that an invalidation from another core hits a resident line before capacity eviction).
* **When L1 Block Size increases:**
  * Compulsory misses drop drastically.
  * Coherence misses remain steady at about $15\%$ of total misses.

---

### 4.4 Memory Traffic Growth (Figure 5.17)
* Increasing block size from 16 bytes to 128 bytes improves hit rate, but **memory bus traffic (bytes transferred per reference) grows significantly**:
  * While the miss rate drops by $3.7\times$, the data payload per miss grows by $8\times$.
  * Total memory bus traffic **more than doubles**!
* **Architectural Insight:** While larger cache blocks reduce core stalls, in systems with high processor counts, the resulting flood of bus traffic creates severe memory interconnect contention.

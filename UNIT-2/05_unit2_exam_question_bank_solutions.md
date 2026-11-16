# Unit 2: Exam Question Bank & Model Solutions

This document compiles comprehensive, high-scoring model answers for the Unit 2 exam and test questions covering **Symmetric Shared-Memory Multiprocessors, Distributed Directory Coherence, Synchronization Primitives, and Memory Consistency Models**.

---

## Question 1: Give any two reasons (conditions) to say a memory system is coherent. [02 Marks]

### Model Answer:
According to Hennessy & Patterson, a memory system is coherent if it satisfies the following three foundational conditions (any two qualify):

1. **Program Order Preservation (Single-Processor Correctness):**
   * A read by processor $P$ to location $X$ that follows a write by $P$ to $X$, with no writes of $X$ by another processor occurring between the write and the read, must always return the value written by $P$.
2. **Write Propagation (Coherent Eventual Visibility):**
   * A read by processor $P_1$ to location $X$ that follows a write by another processor $P_2$ to $X$ must eventually return the value written by $P_2$, provided the read and write are sufficiently separated in time and no other writes to $X$ occur in between.
3. **Write Serialization (Consistent Global View):**
   * Writes to the same location must be **serialized**: that is, two writes to location $X$ by any two processors (e.g., $P_1$ writes 1, then $P_2$ writes 2) must be seen in the exact same sequential order by all processors.

---

## Question 2: List the methods/types to implement snooping protocols. [02 Marks]

### Model Answer:
Snooping protocols track the sharing state of cache blocks across a broadcast bus using two primary mechanisms:

1. **Write-Invalidate Protocol (Most Widely Adopted):**
   * When a processor writes to a shared cache block, it broadcasts an **invalidation signal** over the bus.
   * All other processors snooping the bus invalidate their local cached copies.
   * **Advantage:** Low bus bandwidth on subsequent writes because the writing processor obtains exclusive ownership and can perform multiple consecutive writes locally without further bus traffic.
2. **Write-Update (Write-Broadcast) Protocol:**
   * When a processor writes to a shared cache block, it broadcasts the **new data word** over the bus to all other caches.
   * All other caches snooping the bus update their internal copies immediately.
   * **Advantage:** Eliminates read miss latency for other sharing caches.
   * **Disadvantage:** Consumes immense bus bandwidth because every individual write to a shared variable generates broadcast traffic, even if the other caches never read that variable again.

---

## Question 3: What is the key bottleneck in a Symmetric Shared-Memory (UMA) architecture? [02 Marks]

### Model Answer:
* **The Key Bottleneck:** The **Centralized Shared Physical Bus and Centralized Memory Controller Bandwidth**.
* **Reason:** In Uniform Memory Access (UMA) symmetric multiprocessors, all processor cores communicate with centralized main memory over a single shared bus (or crossbar). 
  * As the core count increases (typically beyond 8 to 16 cores), coherence broadcasts (invalidations/updates) and cache miss servicing saturate the bus.
  * Bus contention dramatically increases memory access latency, limiting the scalability of UMA architectures.

---

## Question 4: True Sharing vs. False Sharing Event Classification [02 Marks]

### Problem:
Assuming that words $x_1$ and $x_2$ reside in the **same cache block** that is initially in the **Shared** state in the caches of both processor $P_1$ and processor $P_2$. Identify the following sequential events as a *true sharing miss*, a *false sharing miss*, or a *hit*:

| Time | Processor $P_1$ | Processor $P_2$ |
|:---:|:---:|:---:|
| 1 | Write $x_1$ | — |
| 2 | — | Read $x_2$ |

### Step-by-Step Classification & Explanation:

1. **Time 1 ($P_1$ writes $x_1$):**
   * **Classification:** **Write Miss / Invalidation (Ownership Acquisition)**.
   * **Explanation:** Because the cache block was initially in the `Shared` state, $P_1$ does not have exclusive permission. $P_1$ issues a bus invalidate. The copy in $P_2$'s cache transitions from `Shared` to `Invalid`. $P_1$'s copy transitions to `Modified` (Exclusive).
2. **Time 2 ($P_2$ reads $x_2$):**
   * **Classification:** **False Sharing Miss**.
   * **Explanation:** 
     * $P_2$ is attempting to read $x_2$, which was **never written or modified** by $P_1$ ($P_1$ only wrote $x_1$).
     * If cache blocks were of single-word granularity, $P_2$'s copy of $x_2$ would still be completely valid and would result in a cache hit.
     * However, because $x_1$ and $x_2$ reside in the **same multi-word cache block**, $P_1$'s write to $x_1$ invalidated the entire block in $P_2$'s cache.
     * When $P_2$ reads $x_2$, it experiences a cache miss solely due to block size allocation, making it a classic **False Sharing Miss**.

---

## Question 5: Explain Atomic Exchange in Synchronization. [03–04 Marks]

### Model Answer:
The **Atomic Exchange (Swap)** primitive interchanges the value of a CPU register with a value in memory as a single, indivisible hardware operation:
$$\text{Temp} = \text{Memory}[\text{Address}]; \quad \text{Memory}[\text{Address}] = \text{Reg}[\text{Value}]; \quad \text{Reg}[\text{Value}] = \text{Temp};$$

```
    Processor Core                     Shared Memory Location (Lock)
   ┌──────────────┐                             ┌─────────────┐
   │ Register R2  │ ◄────── Indivisible ──────► │  Lock (x1)  │
   │  (Value = 1) │          Exchange           │ (0:Free/1:Busy)
   └──────────────┘                             └─────────────┘
```

### Role in Building Mutual Exclusion Locks:
* Let `0` indicate that the lock is **free (unlocked)**.
* Let `1` indicate that the lock is **held (unavailable)**.

1. **Acquiring the Lock:**
   * A processor loads `1` into a register and executes an atomic exchange instruction targeting the lock memory address.
   * **Case A (Returns 0):** The lock was free. The processor successfully acquired the lock and atomically set the memory value to `1`, blocking all other processors.
   * **Case B (Returns 1):** The lock was already held by another processor. The exchange writes `1` (leaving it locked) and returns `1`. The processor knows it failed and must retry.
2. **Breaking Race Conditions (Write Serialization):**
   * If two processors simultaneously execute an atomic exchange on the same address, the memory arbiter/bus write serialization mechanism forces an exact order:
     * The first processor receives `0` (claims the lock).
     * The second processor receives `1` (denied).
   * It is impossible for two processors to simultaneously believe they acquired the lock.
3. **Releasing the Lock:**
   * The holding processor releases the lock by simply writing `0` into the lock address.

---

## Question 6: Examine Synchronization Primitives with Pseudo-Code [08 Marks]

### (a) Load-Reserved (`lr`) and Store-Conditional (`sc`)
Single-instruction read-modify-write operations complicate hardware design and cache coherence because they lock the memory bus across both read and write cycles. Modern RISC architectures (RISC-V, MIPS `LL`/`SC`, ARM `LDREX`/`STREX`) resolve this using a split instruction pair:

1. **`lr.d rd, (rs1)` (Load-Reserved / Load-Linked):**
   * Loads the data from `Memory[rs1]` into register `rd`.
   * Sets a hardware **reservation flag** on the target cache block and records the address in a dedicated internal **Reservation Register**.
2. **`sc.d rd, rs2, (rs1)` (Store-Conditional):**
   * Checks if the reservation on `Memory[rs1]` is still valid.
   * **Success:** If no other processor has written to the block and no interrupt/context switch occurred, the store writes `rs2` to `Memory[rs1]` and returns `0` in `rd` (indicating success).
   * **Failure:** If another processor invalidated the cache block or an exception occurred, the reservation is cleared. The memory write is aborted, and `rd` receives a non-zero value.

#### Pseudo-Code / Assembly for Atomic Exchange using `lr`/`sc` (RISC-V):
```assembly
# Atomic Exchange: Swap value in x4 with memory location at address x1
try:
    lr.d    x2, 0(x1)         # Load current lock value and set reservation
    sc.d    x3, x4, 0(x1)     # Conditionally store new value from x4 into 0(x1)
    bnez    x3, try           # If sc failed (x3 != 0), retry loop
    mv      x4, x2            # Move original memory value into x4
```

#### Pseudo-Code / Assembly for Atomic Fetch-and-Increment:
```assembly
# Atomically increment memory location at address x1 by 1
try:
    lr.d    x2, 0(x1)         # Load current value into x2 & establish reservation
    addi    x3, x2, 1         # Increment value
    sc.d    x3, x3, 0(x1)     # Store back conditionally
    bnez    x3, try           # If another store intervened, retry
```

---

### (b) Spinlocks

A **spinlock** is a synchronization mechanism where a processor continuously executes a tight loop testing whether a lock variable has become free. They are optimal when lock holding times are known to be very short.

#### 1. Naive Spinlock (Spinning on Atomic Swap):
```assembly
        addi    x2, x0, 1         # Load locked value (1)
lockit: exch    x2, 0(x1)         # Atomically swap x2 with Memory[x1]
        bnez    x2, lockit        # If returned 1 (busy), spin and repeat swap
```
* **Severe Performance Flaw:** Every execution of `exch` is a write operation that requests exclusive cache ownership. When multiple processors spin, the interconnect/bus is inundated with continuous write invalidations, degrading overall system throughput.

#### 2. Optimized Cache-Coherent Spinlock: Test-and-Test-and-Set (TTAS)
Instead of executing atomic writes in a loop, processors spin by performing **ordinary reads** on their locally cached copy of the lock variable:
```assembly
lockit: ld      x2, 0(x1)         # 1. Test: Ordinary cached read of lock variable
        bnez    x2, lockit        # 2. Local Spin: Loop locally as long as lock == 1 (NO bus traffic)
        addi    x2, x0, 1         # Load 1
        exch    x2, 0(x1)         # 3. Test-and-Set: Lock observed free (0); race to swap
        bnez    x2, lockit        # 4. If swap returned 1, lost the race; resume reading
```
* **Advantage:** While the lock is held, all waiting processors read hits in their local `Shared` cache state with zero bus transactions. Bus traffic only occurs when the lock is released.

#### 3. Spinlock Implementation using `lr`/`sc`:
```assembly
lockit: lr.d    x2, 0(x1)         # Read lock and set reservation
        bnez    x2, lockit        # If busy (1), spin reading locally
        addi    x2, x0, 1         # Prepare locked value 1
        sc.d    x2, x2, 0(x1)     # Attempt atomic write
        bnez    x2, lockit        # If store failed (reservation broken), retry
```

---

## Question 7: Illustrate Directory-Based Protocol with State Transition Diagrams [08 Marks]

In distributed shared-memory (DSM) architectures, directory coherence replaces broadcast snooping. A centralized or distributed directory tracks the caching state and sharer set for each memory block.

### 1. States of an Individual Cache Block:
* **Invalid:** The cache does not hold a valid copy of the block.
* **Shared:** The cache holds an up-to-date read-only copy; the block may also be cached in other nodes.
* **Modified (Exclusive):** The cache holds the sole, dirty, read/write copy; main memory copy is stale.

#### Cache Block State Transition Diagram:
Below is the exact finite state machine governing transitions based on CPU requests (black) and Directory incoming messages (gray):

![Cache Block State Transition Diagram](file:///v:/ACADEMIC/7TH%20SEM/2.PAGP/NOTES/UNIT-2/images/fig_5_20_cache_block_directory_state_diagram.png)

```
                       ┌───────────────────────┐
                       │        INVALID        │
                       └───────────────────────┘
                        │ ▲                 │ ▲
      CPU Read Miss:    │ │ Invalidate:     │ │ CPU Write Miss:
      Send Read Miss    │ │ Ack Invalidate  │ │ Send Write Miss
      to Directory      │ │                 │ │ to Directory
                        ▼ │                 │ │
              ┌───────────────┐             │ │
              │    SHARED     │             │ │
              └───────────────┘             │ │
                        │ ▲                 │ │
      CPU Write Miss:   │ │ Fetch:          │ │ Data Write-Back:
      Send Write Miss   │ │ Data Write-Back │ │ Cache Block Evicted
      to Directory      │ │                 ▼ │
                        │ └─────────► ┌───────────────┐
                        │             │   MODIFIED    │
                        └────────────►│  (Exclusive)  │
                                      └───────────────┘
```

---

### 2. States of the Directory Controller:
For each block in main memory, the home directory controller maintains:
1. **Coherence State:**
   * **Uncached:** No node has a copy of the block.
   * **Shared:** One or more nodes have read-only copies; memory is current.
   * **Exclusive:** Exactly one node (the owner) has a read-write copy; memory is stale.
2. **Sharers Bit-Vector ($Sharers$):** A bitmask indicating all nodes caching the block in the `Shared` state.

#### Directory Controller State Transition Diagram:
Below is the directory state machine showing message transactions between the Home Directory, Requester ($P$), and Sharers:

![Directory Controller State Transition Diagram](file:///v:/ACADEMIC/7TH%20SEM/2.PAGP/NOTES/UNIT-2/images/fig_5_21_directory_state_diagram.png)

#### Summary of Directory Actions:
* **Read Miss on Uncached:** Directory supplies data to node $P$; state becomes `Shared`; $Sharers = \{P\}$.
* **Read Miss on Shared:** Directory sends data to node $P$; $Sharers = Sharers \cup \{P\}$.
* **Write Miss on Shared:** Directory sends point-to-point **Invalidate** messages to all nodes in $Sharers$; clears sharer list; sets state to `Exclusive`; $Sharers = \{P\}$.
* **Read/Write Miss on Exclusive:** Directory sends a **Fetch / Fetch-Invalidate** message to the owner node; owner writes dirty block back to directory/memory; directory forwards data to requester.

---

## Question 8: Why is Sequential Consistency Simple to Understand but Difficult to Implement Efficiently? [04 Marks]

### 1. Why it is Simple to Understand:
* **Natural Programmer Intuition:** Sequential Consistency (Leslie Lamport) guarantees that memory operations execute in **exact program order** within each thread, and that all operations appear to be interleaved in some global sequential timeline.
* Programmers do not have to reason about memory buffers, reordering hardware, or write delays; memory acts like a single shared pool serviced one request at a time.

### 2. Why it is Difficult to Implement Efficiently (The Performance Bottleneck):
Under strict Sequential Consistency, an access to memory cannot complete until all prior memory accesses by that processor have fully completed globally.
* A processor **cannot use write buffers** to hide store latencies (because subsequent reads cannot bypass pending writes to different addresses).
* A processor **must stall on write misses** until all invalidation messages are sent, received, and explicitly acknowledged across the network.
* Modern out-of-order execution, speculative loads, and non-blocking caches are severely restricted.

#### Numerical Demonstration of the Stall Penalty:
* Assume a multiprocessor where:
  * Time to obtain ownership on a write miss: $50\text{ cycles}$
  * Time to issue each invalidate message: $10\text{ cycles}$
  * Invalidation acknowledgement round-trip latency: $80\text{ cycles}$
  * Number of sharers = 4.
* **Under Sequential Consistency:**
  * The last invalidate is issued after $4 \times 10 = 40\text{ cycles}$.
  * The processor must wait for the last acknowledgement:
    $$\text{Stall}_{\text{SC}} = 50 + 40 + 80 = 170\text{ clock cycles}$$
* **Under a Relaxed Consistency Model:**
  * The processor resumes execution immediately after obtaining ownership ($50\text{ cycles}$), or continues immediately by buffering the write ($0\text{ stall cycles}$).
* **Conclusion:** Sequential consistency enforces a **$170\text{ cycle}$ penalty vs. $50\text{ cycles}$** (over $3.4\times$ slowdown), creating an intolerable bottleneck in scalable architectures.

---

## Question 9: List the orderings that Sequential Consistency needs to maintain. [02 Marks]

### Model Answer:
Sequential Consistency enforces all **four possible combinations** of program-order memory accesses across different memory locations:

1. **$R \to R$ (Read-to-Read):** A read to memory location $A$ must complete before a subsequent read to memory location $B$ can be issued.
2. **$R \to W$ (Read-to-Write):** A read to memory location $A$ must complete before a subsequent write to memory location $B$ can take effect.
3. **$W \to R$ (Write-to-Read):** A write to memory location $A$ must complete and be globally observed before a subsequent read to memory location $B$ can execute.
4. **$W \to W$ (Write-to-Write):** A write to memory location $A$ must complete and be globally observed before a subsequent write to memory location $B$ can take effect.

---

## Question 10: Differentiate between Total Store Ordering (TSO) and Partial Store Order (PSO). [02 Marks]

### Model Answer:

| Feature / Property | Total Store Ordering (TSO) | Partial Store Order (PSO) |
|:---|:---|:---|
| **Relaxed Ordering** | Relaxes **$W \to R$ only**. | Relaxes **both $W \to R$ and $W \to W$**. |
| **Maintained Orderings** | $R \to R, \; R \to W, \; W \to W$ | $R \to R, \; R \to W$ |
| **Store Buffer Behavior** | Stores commit to memory strictly in FIFO program order; subsequent loads may bypass older stores to different addresses. | Stores to different addresses can be retired and committed to the memory system **out of order**. |
| **Write-to-Write ($W \to W$) Ordering** | **Guaranteed.** Flags and shared data written in sequence maintain their order automatically. | **Not Guaranteed.** Requires explicit memory fence/barrier instructions between consecutive stores. |
| **Hardware Example** | x86 (Intel / AMD), SPARC TSO | SPARC PSO |

---

## Question 11: Discuss Various Relaxed Consistency Models. [08 Marks]

### Model Answer:

### 1. Motivation for Relaxed Consistency:
Strict Sequential Consistency forces severe processor stalls to maintain orderings that most programs do not require. Relaxed consistency models allow memory reads and writes to complete **out of order**, relying on synchronization operations (locks, barriers, fences) to enforce order only where necessary.

---

### 2. Classification of Models:

```
                            RELAXED MEMORY MODELS TAXONOMY
                                          │
                   Sequential Consistency (Maintains R→R, R→W, W→R, W→W)
                                          │
                               Relax W→R  ▼
                    Total Store Ordering (TSO) / Processor Consistency
                                          │
                               Relax W→W  ▼
                            Partial Store Order (PSO)
                                          │
                       Relax R→R  ▼
                       Relax R→W
                    Weak Ordering (WO) / Release Consistency (RC)
```

#### A. Total Store Ordering (TSO) / Processor Consistency:
* **Relaxes:** $W \to R$ ordering.
* **Mechanism:** Allows a processor to place writes in a local store buffer and execute subsequent independent reads without waiting for writes to be globally visible.
* **Architectures:** x86 (Intel, AMD), IBM System/370, SPARC TSO.
* **Property:** Retains $W \to W$ order, allowing many lock-free algorithms to run without code modification.

#### B. Partial Store Order (PSO):
* **Relaxes:** $W \to R$ and $W \to W$ orderings.
* **Mechanism:** Stores in write buffers can be reordered and committed to memory non-sequentially, enabling memory controller write-coalescing.
* **Architectures:** SPARC PSO mode.

#### C. Weak Ordering (WO):
* **Relaxes:** All four data access orderings ($R \to R, R \to W, W \to R, W \to W$).
* **Mechanism:** Decouples ordinary data accesses from synchronization variables. Ordinary data operations can float freely between synchronization points.
* **Rule:** A synchronization access ($S$) acts as a two-way barrier: all previous memory operations must complete before $S$ is executed, and no operations after $S$ may begin until $S$ completes.
* **Architectures:** PowerPC.

#### D. Release Consistency (RC):
Adopted by **RISC-V, ARMv8, and C/C++11**, Release Consistency splits synchronization accesses into two directional operations:
1. **Acquire ($S_A$):** Operations used to gain access to shared variables (e.g., `lock()`).
   * **Rule:** Memory accesses following the acquire cannot move above $S_A$ ($S_A \to \text{Read}/\text{Write}$).
2. **Release ($S_R$):** Operations used to release access to shared variables (e.g., `unlock()`).
   * **Rule:** Memory accesses preceding the release cannot move below $S_R$ ($\text{Read}/\text{Write} \to S_R$).
* **Advantage:** Unlocks one-way barrier optimizations: operations prior to an acquire can safely migrate past $S_A$, and operations after a release can safely migrate above $S_R$.

---

### 3. Summary Ordering Comparison Table (Table 5.23):

| Model | Relaxed Ordinary Orderings | Maintained Ordinary Orderings | Maintained Synchronization Orderings |
|:---|:---|:---|:---|
| **Sequential Consistency (SC)** | *None* | $R \to R, \; R \to W, \; W \to R, \; W \to W$ | $S \to W, \; S \to R, \; R \to S, \; W \to S, \; S \to S$ |
| **Total Store Order (TSO)** | $W \to R$ | $R \to R, \; R \to W, \; W \to W$ | $S \to W, \; S \to R, \; R \to S, \; W \to S, \; S \to S$ |
| **Partial Store Order (PSO)** | $W \to R, \; W \to W$ | $R \to R, \; R \to W$ | $S \to W, \; S \to R, \; R \to S, \; W \to S, \; S \to S$ |
| **Weak Ordering (WO)** | All ($R \to R, R \to W, W \to R, W \to W$) | *None* | $S \to W, \; S \to R, \; R \to S, \; W \to S, \; S \to S$ |
| **Release Consistency (RC)** | All ($R \to R, R \to W, W \to R, W \to W$) | *None* | $S_A \to W, \; S_A \to R, \; R \to S_R, \; W \to S_R,$ <br> $S_A \to S_A, \; S_A \to S_R, \; S_R \to S_A, \; S_R \to S_R$ |

---

### 4. Memory Fences:
To enforce explicit memory ordering on relaxed architectures, explicit fence instructions are provided:
* **RISC-V:** `FENCE rw, rw` (enforces full predecessor-to-successor ordering).
* **ARMv8:** `DMB` (Data Memory Barrier), `DSB` (Data Synchronization Barrier).
* **x86:** `MFENCE`, `SFENCE`, `LFENCE`.

---

## Question 12: Define Loop-Carried Dependence, and Why Does It Reduce Loop-Level Parallelism? [02–04 Marks]

### 1. Definition:
A **loop-carried dependence** exists when a statement in one iteration of a loop depends on a value produced by a statement in an **earlier iteration** of the same loop.

### 2. Code Example:
```c
for (int i = 1; i <= N; i++) {
    A[i] = A[i - 1] + B[i];   // Statement S1
}
```
* In iteration $i = 1$, $S1$ computes $A[1]$ using $A[0]$.
* In iteration $i = 2$, $S1$ reads $A[1]$ to compute $A[2]$.
* Because iteration $2$ cannot begin until iteration $1$ completes and stores $A[1]$, this is a **True (RAW) Loop-Carried Dependence**.

### 3. Why It Reduces Loop-Level Parallelism:
1. **Enforces Sequential Serialization:**
   * A loop with a loop-carried dependence cannot have its iterations dispatched across multiple parallel cores, vector lanes, or GPU threads simultaneously. 
   * Iteration $i$ must wait for iteration $i-1$ to compute and write back its result.
2. **Prevents Vectorization & Parallelization:**
   * SIMD and vector execution require that all elements in a vector chunk be evaluated concurrently.
   * If a loop-carried dependence exists, parallelizing the loop naively introduces **race conditions** and produces mathematically incorrect results.


---

## Question 13: Quick / Fill-in-the-Blank Exam Answers [01–02 Marks]

1. **"Relaxing the $W \to W$ ordering yields a ___________ model."**
   * **Answer:** **Partial Store Order (PSO)** *(Specifically, relaxing both $W \to R$ and $W \to W$ orderings yields PSO)*.
2. **"Analyze Sequential Consistency."**
   * **Answer:** Sequential Consistency (Leslie Lamport, 1979) requires that:
     1. All memory accesses from each individual processor execute in strict **program order**.
     2. Memory accesses from different processors are interleaved in some single, global sequential order observed identically by all cores.
     * Enforces all 4 access orderings: $R \to R, R \to W, W \to R, W \to W$.

---

## Question 14: Differentiate between Temporal Locality and Spatial Locality. [02–04 Marks]

### Model Answer:
The Principle of Locality states that programs tend to reuse data and instructions they have used recently. It manifests in two distinct dimensions:

| Dimension / Metric | Temporal Locality (Locality in Time) | Spatial Locality (Locality in Space) |
|:---|:---|:---|
| **Definition** | If a specific memory location is referenced once, it will tend to be **referenced again soon** in the near future. | If a memory location is referenced, memory locations with **neighboring/nearby addresses** will tend to be referenced soon. |
| **Typical Code Patterns** | • Program loops (executing the same instructions repeatedly). <br> • Induction variables and loop counters (e.g., `i++`). <br> • Accumulator variables (e.g., `sum += A[i]`). | • Sequential instruction stream execution. <br> • Traversing arrays sequentially (e.g., `A[0], A[1], A[2]...`). <br> • Accessing consecutive fields in a struct/record. |
| **Hardware Cache Exploitation** | • Retaining recently accessed cache blocks in the cache hierarchy using eviction policies like **LRU (Least Recently Used)**. | • Employing **multi-word cache blocks (lines)** (e.g., 64 bytes) rather than 1-word blocks, so fetching word $i$ pre-fetches words $i+1, i+2...$ |
| **Miss Type Addressed** | Reduces **Capacity and Conflict Misses** through temporal reuse. | Reduces **Compulsory (Cold) Misses** because an entire block of adjacent data is brought into cache on the first miss. |

---

## Question 15: Differentiate between Cache Coherence and Memory Consistency. Discuss Under What Scenarios a Memory System is Called Coherent. [06 Marks]

### 1. Distinction between Coherence and Consistency:

| Feature / Aspect | Cache Coherence | Memory Consistency |
|:---|:---|:---|
| **Scope of Addresses** | Focuses on accesses to a **single, specific memory location** ($X$). | Focuses on accesses across **multiple different memory locations** ($A, B, X, Y$). |
| **Core Question Answered** | *"WHAT value does a read return?"* | *"WHEN does a processor observe a write made by another processor?"* |
| **Primary Guarantee** | Guarantees **write serialization** (all processors see writes to address $X$ in the same order) and **write propagation** (writes are eventually visible). | Guarantees the **global ordering of reads and writes** across threads and memory locations. |
| **Hardware Enforcement Level** | Maintained by **cache controllers** via snooping bus transactions or directory message protocols. | Maintained by **instruction pipeline rules**, store buffers, memory fences, and processor execution order. |
| **Programmer Control** | Transparent to software; hardware automatically invalidates or updates stale cache blocks. | Handled via software memory models, synchronization primitives (`lock`/`unlock`), and explicit memory barriers (`FENCE`). |

```
    ┌────────────────────────────────────────────────────────────┐
    │ Cache Coherence: Governs Single Address (e.g., Location X) │
    │    Write to X ────────────────────────► Read from X        │
    └────────────────────────────────────────────────────────────┘
    ┌────────────────────────────────────────────────────────────┐
    │ Memory Consistency: Governs Cross-Address Ordering         │
    │    Write to A ════════════════════════► Read from B        │
    │    (Does Processor 2 see Write(A) before it reads B?)      │
    └────────────────────────────────────────────────────────────┘
```

---

### 2. Scenarios Under Which a Memory System is Called Coherent:
According to Hennessy & Patterson, a memory system is formally defined as **coherent** if it satisfies all three of the following operational scenarios:

1. **Preserving Program Order (Single-Processor Correctness):**
   * *Scenario:* Processor $P$ writes a value to location $X$, and subsequently reads from location $X$, with no intervening writes to $X$ by any other processor.
   * *Requirement:* The read must **always return the value written by $P$**.
2. **Coherent Write Propagation (Visibility across Processors):**
   * *Scenario:* Processor $P_1$ writes to location $X$, and processor $P_2$ reads location $X$ after a sufficient time delay, with no other writes to $X$ occurring in between.
   * *Requirement:* The read by $P_2$ must **eventually return the updated value written by $P_1$**. (Ensures that caches cannot indefinitely retain stale values).
3. **Write Serialization (Uniform Global Timeline):**
   * *Scenario:* Two processors $P_1$ and $P_2$ write values to the same location $X$ (e.g., $P_1$ writes 10, $P_2$ writes 20).
   * *Requirement:* **All processors in the system must observe these two writes in the exact same sequential order.** (e.g., If one processor sees 10 followed by 20, no processor is permitted to see 20 followed by 10).

---

## Question 16: Illustrate the Snooping Cache Coherence Protocol with an Example. [04–06 Marks]

### 1. Conceptual Principle:
In a snooping-based symmetric multiprocessor, all processor caches are connected to a **shared broadcast bus**. Every cache controller continuously monitors ("snoops") the bus for addresses requested by other processors and reacts to maintain coherence.

---

### 2. Step-by-Step Example Walkthrough:
Assume three processors ($P_1, P_2, P_3$) with private write-back caches sharing memory variable $X$ (initially $X = 0$ in main memory):

```
                       SHARED BROADCAST BUS
  ═══════════════════════════════════════════════════════════════════════
       │                      │                      │              │
   ┌───┴────┐             ┌───┴────┐             ┌───┴────┐     ┌───┴────┐
   │ Cache  │             │ Cache  │             │ Cache  │     │ Main   │
   │   P1   │             │   P2   │             │   P3   │     │ Memory │
   └────────┘             └────────┘             └────────┘     └────────┘
```

| Step | Processor Action | Bus Transaction Broadcast | State in Cache $P_1$ | State in Cache $P_2$ | State in Cache $P_3$ | Value in Memory | Description / Snooping Reaction |
|:---:|:---|:---|:---:|:---:|:---:|:---:|:---|
| **0** | Initial State | — | Invalid | Invalid | Invalid | $X = 0$ | $X$ is not cached anywhere. |
| **1** | $P_1$ Reads $X$ | `BusRd(X)` | **Shared ($X=0$)** | Invalid | Invalid | $X = 0$ | Read miss on $P_1$. Main memory services the block. |
| **2** | $P_2$ Reads $X$ | `BusRd(X)` | Shared ($X=0$) | **Shared ($X=0$)** | Invalid | $X = 0$ | Read miss on $P_2$. Memory supplies data. Both $P_1$ & $P_2$ hold read-only shared copies. |
| **3** | **$P_1$ Writes $X = 15$** | **`BusInval(X)`** | **Modified ($X=15$)** | **Invalid** | Invalid | $X = 0$ *(stale)* | Write hit on Shared block. $P_1$ broadcasts `BusInval`. $P_2$ snoops and invalidates its copy. $P_1$ becomes exclusive owner. |
| **4** | $P_1$ Writes $X = 25$ | *(None)* | **Modified ($X=25$)** | Invalid | Invalid | $X = 0$ *(stale)* | $P_1$ is in Modified state; writes locally with **zero bus traffic**. |
| **5** | **$P_3$ Reads $X$** | `BusRd(X)` | **Shared ($X=25$)** | Invalid | **Shared ($X=25$)** | **$X = 25$** *(updated)* | $P_3$ generates a read miss. $P_1$ snoops `BusRd`, halts memory, flushes $X=25$ to bus and memory, and transitions to Shared. |

---

## Question 17: Analyze Write-Invalidate Snooping Cache Coherence Protocol for a Write-Back Cache (3-State MSI Protocol) with State Transition Diagram. [08 Marks]

### 1. States for Each Cache Block:
A basic write-invalidate protocol for write-back caches maintains three distinct states for every cache block (**MSI Protocol**):

1. **Invalid (I):**
   * The cache block contains no valid data. Any read or write by the local CPU will result in a cache miss.
2. **Shared (S):**
   * The block is valid and clean (identical to main memory).
   * Other caches may also hold a copy in the `Shared` state.
   * Read-only: CPU read hits do not generate bus traffic; CPU writes require upgrading permissions.
3. **Modified (M) / Exclusive Dirty:**
   * The block is valid, dirty, and held **exclusively** by this cache.
   * Main memory is stale.
   * The local CPU has full read and write permissions without generating bus traffic.

---

### 2. Bus Transactions Triggered by Cache Misses:
* **`BusRd` (Bus Read):** Broadcast on a CPU read miss when the cache does not intend to modify the data.
* **`BusRdX` (Bus Read Invalidate / Read with Intent to Modify):** Broadcast on a CPU write miss when the cache does not have the block. It requests data and forces all other caches to invalidate their copies.
* **`BusInval` / `BusUpgr` (Bus Invalidate):** Broadcast on a CPU write hit to a block currently in `Shared` state to invalidate all other copies without refetching data.
* **`WriteBack`:** Flush of a modified block to main memory upon eviction or when servicing another processor's request.

---

### 3. State Transition Matrix:

| Current State | Local CPU Event | Action Taken / Bus Message Sent | New State |
|:---:|:---:|:---:|:---:|
| **Invalid (I)** | CPU Read | Issue `BusRd` request; wait for data | **Shared (S)** |
| **Invalid (I)** | CPU Write | Issue `BusRdX` request; fetch data and write | **Modified (M)** |
| **Shared (S)** | CPU Read | Read hit; no bus action | **Shared (S)** |
| **Shared (S)** | CPU Write | Issue `BusInval` (or `BusRdX`) to invalidate sharers | **Modified (M)** |
| **Modified (M)** | CPU Read | Read hit; no bus action | **Modified (M)** |
| **Modified (M)** | CPU Write | Write hit locally; no bus action | **Modified (M)** |
| **Shared (S)** | Snooped `BusInval` or `BusRdX` | Invalidate local copy | **Invalid (I)** |
| **Shared (S)** | Snooped `BusRd` | No action (multiple sharers allowed) | **Shared (S)** |
| **Modified (M)** | Snooped `BusRd` | Intervene: flush dirty block to bus/memory (`WriteBack`) | **Shared (S)** |
| **Modified (M)** | Snooped `BusRdX` | Intervene: flush dirty block to bus (`WriteBack`); invalidate | **Invalid (I)** |

---

### 4. State Transition Diagram (MSI Write-Back Snooping):

```
                           ┌─────────────────────────┐
                           │       INVALID (I)       │
                           └─────────────────────────┘
                            │ ▲                   ▲ │
           CPU Read Miss:   │ │ Snooped BusRdX or │ │ CPU Write Miss:
           Issue BusRd      │ │ Snooped BusInval  │ │ Issue BusRdX
                            ▼ │                   │ │
                  ┌──────────────────┐            │ │
                  │    SHARED (S)    │            │ │
                  └──────────────────┘            │ │
                   │ ▲      │                     │ │
  CPU Read Hit:    │ │      │ CPU Write Hit:      │ │
  No Bus Action ───┘ │      │ Issue BusInval      │ │
                     │      ▼                     │ │
     Snooped BusRd:  │    ┌──────────────────┐    │ │
     Flush Data to   └────┤   MODIFIED (M)   ├────┘ │
     Bus & Memory         └──────────────────┘      │
                           │ ▲                      │
          CPU Read/Write:  │ │   Snooped BusRdX:    │
          Hit (No Bus Act) ┘ └── Flush Data to Bus ─┘
```

---

## Question 18: Compare Shared Memory and Distributed Memory Programming Models. [04 Marks]

### Model Answer:

| Feature / Aspect | Shared-Memory Programming Model | Distributed-Memory Programming Model |
|:---|:---|:---|
| **Address Space** | **Single Unified Address Space:** All processor cores can directly address and access all physical memory locations. | **Multiple Disjoint Private Address Spaces:** Each node has its own private memory; no node can directly address another's memory. |
| **Communication Mechanism** | **Implicit Communication:** Threads communicate by reading and writing to shared memory locations (`load`/`store`). | **Explicit Communication:** Nodes communicate via explicit network message passing routines (`Send`/`Receive`). |
| **Synchronization** | Primitive locks, mutexes, condition variables, semaphores, and memory fences (e.g., Pthreads, OpenMP). | Message acknowledgements, synchronous/blocking messages, and collective barriers (e.g., `MPI_Barrier`). |
| **Underlying Hardware Architecture** | Symmetric Multiprocessors (SMP), Uniform Memory Access (UMA), or Cache-Coherent NUMA systems. | Clusters, Massively Parallel Processors (MPPs), networked workstations (commodity ethernet/InfiniBand). |
| **Programming Standards / APIs** | **OpenMP**, **POSIX Threads (Pthreads)**, **C++11 Threads**. | **MPI (Message Passing Interface)**, **PVM**. |
| **Hardware Complexity** | **High:** Requires complex hardware cache coherence protocols and high-bandwidth low-latency interconnects. | **Low:** Relies on standard network interface controllers (NICs); hardware cache coherence across nodes is not needed. |
| **Scalability** | **Limited Scalability:** Hardware coherence and bus/memory bandwidth typically saturate beyond tens to hundreds of cores. | **Massive Scalability:** Easily scales to tens of thousands or millions of processor cores (supercomputers). |

---

## Question 19: Analyze Basic Hardware Synchronization Primitives: Test-and-Set, Fetch-and-Increment, and Load-Linked / Store-Conditional. [06–08 Marks]

### 1. Test-and-Set
* **Operation & Semantics:**
  * Tests the current value of a memory address against a condition (typically testing if it is `0`).
  * Atomically sets the memory location to `1` and returns the old value stored at that location:
    $$\text{TestAndSet}(M) = [\text{old} = \text{Memory}[M]; \quad \text{Memory}[M] = 1; \quad \text{return } \text{old};]$$
* **Lock Construction:**
  ```c
  while (TestAndSet(&lock_var) == 1) {
      // Busy wait (spin) until lock is acquired
  }
  ```
* **Pros & Cons:** Simple to implement in single-core or early multiprocessors, but causes severe bus traffic in cache-coherent multiprocessors because every test is an atomic write request.

---

### 2. Fetch-and-Increment
* **Operation & Semantics:**
  * Atomically reads the value from a memory location, increments the memory location by a specified step (usually `1`), and returns the **original (unincremented)** value:
    $$\text{FetchAndIncrement}(M) = [\text{old} = \text{Memory}[M]; \quad \text{Memory}[M] = \text{old} + 1; \quad \text{return } \text{old};]$$
* **Usage in Synchronization:**
  * **Ticket Locks:** Guarantees FIFO fairness and starvation-free lock acquisition (like taking a numbered ticket at a deli).
  * **Work Queues & Dynamic Scheduling:** Worker threads atomically fetch the next task index from a shared global index variable without requiring a mutual exclusion lock.
* **Pros & Cons:** Eliminates lock contention for counters and work-lists, but hardware support for multi-operand read-modify-write complicates coherence controllers.

---

### 3. Load-Linked (`LL`) and Store-Conditional (`SC`)
* **Operation & Semantics:**
  * Modern RISC alternative (RISC-V `lr`/`sc`, MIPS `LL`/`SC`, ARM `LDREX`/`STREX`) that splits the read and write operations into two distinct instructions:
    1. **`Load-Linked` / `Load-Reserved` (`LL`):** Loads value from memory into a register and registers a hardware reservation on that memory address.
    2. **`Store-Conditional` (`SC`):** Stores a new value into memory **only if no other processor has written to that address** since the `LL` was executed. Returns `0` on success and non-zero on failure.
* **Assembly Implementation (Atomic Swap in RISC-V):**
  ```assembly
  try:
      lr.d    x2, 0(x1)         # Load reserved from lock address x1
      sc.d    x3, x4, 0(x1)     # Store new value x4 conditionally
      bnez    x3, try           # If store failed (reservation broken), retry
      mv      x4, x2            # Put loaded value into x4
  ```
* **Pros & Cons:**
  * **Major Advantage:** Extremely flexible. Can be used to synthesize any arbitrary atomic primitive (atomic exchange, fetch-and-increment, compare-and-swap) using simple register arithmetic in between.
  * **Hardware Simplicity:** Does not require locking memory buses; relies entirely on standard cache coherence invalidations to clear reservation registers.


---

## Question 20: Discuss Briefly Two Most Common Extensions to the Basic (MSI) Coherence Protocol. [04–06 Marks]

### Model Answer:
The baseline 3-state write-invalidate protocol (**MSI**: Modified, Shared, Invalid) suffers from two major inefficiencies:
1. Every write to a clean block requires broadcasting a bus invalidation (`BusInval`), even if no other cache currently holds a copy.
2. Servicing a read miss from a modified cache block requires an expensive write-back of the full block to main memory before it can transition to Shared.

To optimize these cases, modern architectures introduce two widely adopted extensions:

```
    MSI Baseline ───► Add 'Exclusive' State ───► MESI Protocol (Intel Core, x86)
         │
         └──────────► Add 'Owner' State     ───► MOESI Protocol (AMD Opteron/Ryzen, ARM)
```

---

### 1. Extension 1: The MESI Protocol (Addition of the 'Exclusive' / 'E' State)
* **What is the Exclusive (E) State?**
  * The cache block is **clean** (identical to main memory, unlike Modified).
  * The block is held **exclusively by only this one cache** (no other cache has a copy, unlike Shared).
* **Operational Benefit:**
  * When a processor reads a block that is not cached by any other processor, it loads it directly into the **Exclusive (E)** state (indicated by a shared bus line left unasserted during `BusRd`).
  * If the processor subsequently decides to **write** to this block, it can silently promote the state from `Exclusive` to `Modified` **with zero bus traffic and zero invalidation latency**!
* **Where Used:** Intel Core i3/i5/i7/i9 processor families, standard x86 architectures.

---

### 2. Extension 2: The MOESI Protocol (Addition of the 'Owner' / 'O' State)
* **What is the Owner (O) State?**
  * The cache block is **dirty** (modified relative to main memory).
  * However, the block is currently **shared among multiple caches** (other caches hold it in the read-only `Shared` state).
  * The cache in the `Owner` state **owns** the dirty data and takes full responsibility for:
    1. Supplying the data directly to other caches when they experience read misses (cache-to-cache transfer).
    2. Writing the dirty block back to main memory only when it is eventually evicted.
* **Operational Benefit:**
  * Eliminates the expensive main memory write-back latency when a modified block transitions to being shared among multiple reading processors.
* **Where Used:** AMD Zen / Opteron / Ryzen architectures, ARM big.LITTLE / Cortex processors.

---

## Question 21: Explain Snooping Cache Coherence Protocol with State Transition Diagram Based Specifically on Requests Generated from the CPU. [08 Marks]

### Model Answer:
In a symmetric multiprocessor running a write-invalidate write-back protocol (MSI), state transitions within a local cache can be analyzed strictly from the perspective of **CPU-generated operations**:
1. **`PrRd` (Processor Read):** CPU attempts to load a data word from memory.
2. **`PrWr` (Processor Write):** CPU attempts to store a data word to memory.

---

### 1. Detailed Transitions for CPU-Generated Requests:

1. **State: Invalid (I)**
   * **On `PrRd` (CPU Read Miss):**
     * The cache does not have the block. It broadcasts a `BusRd` request onto the shared bus to fetch the block from memory or a peer cache.
     * The data block is installed in the cache.
     * **Transition:** $\text{Invalid} \xrightarrow{\text{PrRd / BusRd}} \text{Shared}$.
   * **On `PrWr` (CPU Write Miss):**
     * The cache does not have write permission. It broadcasts a `BusRdX` (Bus Read Invalidate / Read with Intent to Modify) request to fetch the block and invalidate all other copies.
     * The CPU writes the new data into the cache block.
     * **Transition:** $\text{Invalid} \xrightarrow{\text{PrWr / BusRdX}} \text{Modified}$.

2. **State: Shared (S)**
   * **On `PrRd` (CPU Read Hit):**
     * The cache already holds an up-to-date read-only copy.
     * The read is satisfied immediately from the local cache with zero bus traffic.
     * **Transition:** $\text{Shared} \xrightarrow{\text{PrRd / (No Bus Action)}} \text{Shared}$ *(Self-loop)*.
   * **On `PrWr` (CPU Write Hit / Permission Upgrade Miss):**
     * The cache holds the data, but cannot write because other caches may share it.
     * It broadcasts a `BusInval` (Bus Invalidate / Bus Upgrade) signal to force all other caches to discard their copies.
     * The CPU writes the data into the block and claims exclusive ownership.
     * **Transition:** $\text{Shared} \xrightarrow{\text{PrWr / BusInval}} \text{Modified}$.

3. **State: Modified (M)**
   * **On `PrRd` (CPU Read Hit):**
     * The cache is the exclusive owner with the newest dirty copy.
     * Read is satisfied locally with zero bus delay.
     * **Transition:** $\text{Modified} \xrightarrow{\text{PrRd / (No Bus Action)}} \text{Modified}$ *(Self-loop)*.
   * **On `PrWr` (CPU Write Hit):**
     * The cache is already exclusive.
     * Write completes locally in the cache with zero bus delay.
     * **Transition:** $\text{Modified} \xrightarrow{\text{PrWr / (No Bus Action)}} \text{Modified}$ *(Self-loop)*.

---

### 2. CPU-Generated State Transition Diagram:

```
                            ┌────────────────────────┐
                            │      INVALID (I)       │
                            └────────────────────────┘
                             │                      │
             PrRd / BusRd    │                      │ PrWr / BusRdX
             (Fetch data)    │                      │ (Fetch data &
                             ▼                      │  invalidate peers)
                 ┌──────────────────────┐           │
                 │      SHARED (S)      │           │
                 └──────────────────────┘           │
                  │ ▲                │              │
     PrRd / (Hit) │ │                │ PrWr /       │
     (No bus act) └─┘                │ BusInval     │
                                     ▼              ▼
                         ┌──────────────────────────────┐
                         │         MODIFIED (M)         │
                         └──────────────────────────────┘
                          │ ▲
     PrRd / (Hit)         │ │
     PrWr / (Hit) ────────┘ │
     (Zero bus transactions)
```

---

## Question 22: Draw and Explain Different Multiprocessor Architectures. [08 Marks]

### Model Answer:
Multiprocessors are broadly categorized based on **memory organization**, **address space visibility**, and **physical interconnect topology**:

```
                       MULTIPROCESSOR ARCHITECTURES
                                    │
       ┌────────────────────────────┴───────────────────────────┐
       ▼                                                        ▼
Shared-Memory Architectures                             Distributed-Memory Architectures
(Single Unified Address Space)                          (Disjoint Private Address Spaces)
       │                                                        │
 ┌─────┴──────────────────┐                               ┌─────┴──────────────────┐
 ▼                        ▼                               ▼                        ▼
SMP / UMA                DSM / NUMA                      Clusters                MPPs
(Uniform Memory Access)  (Non-Uniform Memory Access)     (Commodity Network)      (Specialized Network)
```

---

### 1. Symmetric Shared-Memory Multiprocessors (SMPs) / Uniform Memory Access (UMA)

```
      Core 1         Core 2         Core 3         Core N
    ┌────────┐     ┌────────┐     ┌────────┐     ┌────────┐
    │ Cache  │     │ Cache  │     │ Cache  │     │ Cache  │
    └────┬───┘     └────┬───┘     └────┬───┘     └────┬───┘
  ═══════╧══════════════╧══════════════╧══════════════╧════════ (Snooping Bus)
                                │
                        ┌───────┴────────┐
                        │  Centralized   │
                        │  Main Memory   │
                        └────────────────┘
```

* **Characteristics:**
  * Uses a single centralized main memory shared uniformly by all processors.
  * **Uniform Memory Access (UMA):** Memory access latency is identical for all cores, regardless of which core initiates the request.
  * Cores and memory are interconnected by a **shared physical broadcast bus** or small crossbar.
  * Cache coherence is enforced using **Snooping protocols**.
* **Limitations:**
  * The centralized bus saturates rapidly beyond **8 to 16 cores**, making UMA unscalable for large-scale parallel processing.

---

### 2. Distributed Shared-Memory Architectures (DSMs) / Non-Uniform Memory Access (NUMA)

```
        Node 0                           Node 1                           Node N
 ┌────────────────────┐           ┌────────────────────┐           ┌────────────────────┐
 │  Core  │  Cache    │           │  Core  │  Cache    │           │  Core  │  Cache    │
 ├────────┴───────────┤           ├────────┴───────────┤           ├────────┴───────────┤
 │ Directory │ Memory │           │ Directory │ Memory │           │ Directory │ Memory │
 └─────────┬──────────┘           └─────────┬──────────┘           └─────────┬──────────┘
           │                                │                                │
 ══════════╧════════════════════════════════╧════════════════════════════════╧═════════
                       Scalable Interconnect Network (Mesh / Fat-Tree)
```

* **Characteristics:**
  * Physical memory is partitioned and distributed across multiple compute nodes.
  * **Unified Global Address Space:** Every processor can directly address memory located on any node via ordinary load/store instructions.
  * **Non-Uniform Memory Access (NUMA):** Accessing local on-node memory is significantly faster than accessing remote memory located on another node across the network.
  * Cache coherence is enforced using **Directory-Based protocols** with point-to-point network messaging.
* **Advantages:** High memory bandwidth and excellent scalability up to hundreds or thousands of processor cores.

---

### 3. Distributed-Memory Multicomputers / Clusters (Message Passing)
* **Characteristics:**
  * Each computing node consists of an autonomous processor and private memory with **completely separate, disjoint physical address spaces**.
  * No core can directly read or write memory residing on another node.
  * Nodes communicate strictly via explicit software message passing across external networks (e.g., Ethernet, InfiniBand) using standards like **MPI (Message Passing Interface)**.
* **Advantages:** No hardware cache coherence complexity; virtually unlimited scalability (powers modern supercomputers and cloud data centers).

---

## Question 23: What is Multiprocessor Cache Coherence? Differentiate Between Coherence and Consistency, and Discuss Basic Schemes for Enforcing Coherence. [08 Marks]

### 1. Definition of Multiprocessor Cache Coherence:
In a shared-memory multiprocessor where multiple cores maintain local private caches, multiple copies of the same physical memory block can exist simultaneously in different caches. 
* **The Cache Coherence Problem:** If one processor modifies its local cached copy without notifying or updating other caches, other processors will read stale data, leading to computational corruption.
* **Cache Coherence Definition:** A memory system is coherent if any read to a memory location returns the most recently written value to that location, and all writes to that location are seen in the exact same sequential order by all processors.

---

### 2. Difference Between Coherence and Consistency:

| Dimension | Cache Coherence | Memory Consistency |
|:---|:---|:---|
| **Memory Scope** | Single individual memory address ($X$). | Cross-address ordering among multiple variables ($A, B, C$). |
| **Question Answered** | *"What value does a read return?"* | *"When does a write become visible to other processors?"* |
| **System Domain** | Local caches and interconnect controllers. | CPU pipeline, store buffers, and instruction commit order. |
| **Resolution Method** | Snooping or Directory state machines. | Memory consistency models (SC, TSO, PSO, RC) & Fences. |

---

### 3. Basic Schemes for Enforcing Cache Coherence:

#### Scheme A: Snooping Protocols (Broadcast-Based)
* **Underlying Media:** Shared physical broadcast bus or ring interconnect.
* **Mechanism:** Every private cache controller monitors ("snoops") the bus for addresses requested by any other processor.
  * **Write-Invalidate:** The writing core broadcasts an invalidation address; all sharing caches invalidate their local copies.
  * **Write-Update:** The writing core broadcasts the updated data word; all sharing caches update their local copies.
* **Best Suited For:** Small-scale systems (SMPs / UMA) up to 8–16 cores.

#### Scheme B: Directory-Based Protocols (Point-to-Point)
* **Underlying Media:** Scalable point-to-point interconnect networks (2D mesh, torus, fat-tree).
* **Mechanism:** Avoids expensive broadcast storms by maintaining a centralized or distributed **Directory**:
  * Each memory block has an entry in a directory controller tracking its state (`Uncached`, `Shared`, `Exclusive`) and a **bit-vector of sharers** identifying which nodes currently cache that block.
  * On a write miss, the directory sends invalidation messages **only to the specific nodes listed in the sharer set**, leaving all other nodes undisturbed.
* **Best Suited For:** Large-scale Distributed Shared-Memory (DSM / NUMA) systems.


---

## Question 24: 7-Step True Sharing Miss vs. False Sharing Miss Trace Problem [06 Marks]

### Problem Statement:
Assume that words $x_1$ and $x_2$ reside in the **same cache block**, which is initially in the **Uncached** state for both processors $P_1$ and $P_2$. Assuming the following sequence of events, identify each event as a **true sharing miss**, a **false sharing miss**, or a **hit**, and explain the cache state transitions:

```
TIME    Processor P1        Processor P2
 1        Read x1                 —
 2           —                 Read x1
 3           —                 Write x2
 4        Write x2                —
 5           —                 Write x1
 6        Read x2                 —
 7           —                 Read x1
```

---

### Step-by-Step Execution Trace & Classification Table:

| Time | Processor & Action | Event Classification | State in $P_1$'s Cache | State in $P_2$'s Cache | Detailed Explanation & Coherence Mechanism |
|:---:|:---|:---|:---:|:---:|:---|
| **1** | **$P_1$: Read $x_1$** | **Cold (Compulsory) Miss** | **Shared** | Invalid | The cache block is uncached. $P_1$ brings the block containing both $x_1$ and $x_2$ from memory into its cache in the `Shared` state. |
| **2** | **$P_2$: Read $x_1$** | **Cold (Compulsory) Miss** *(or Shared Read Miss)* | Shared | **Shared** | $P_2$ does not have the block. It fetches the block from memory or $P_1$. Both caches now hold clean copies in the `Shared` state. |
| **3** | **$P_2$: Write $x_2$** | **Write Miss (Coherence / Upgrade Miss)** | **Invalid** | **Modified** | $P_2$ writes to $x_2$. It must obtain exclusive ownership, so it broadcasts an invalidate. $P_1$'s cache is invalidated. $P_2$ transitions to `Modified`. |
| **4** | **$P_1$: Write $x_2$** | **True Sharing Miss** | **Modified** | **Invalid** | $P_1$ writes to $x_2$, the **exact same word** that was written by $P_2$ at Time 3. Since $P_1$ was invalidated, it misses. Because both processors are accessing and modifying the same variable $x_2$, this is a **True Sharing Miss**. $P_2$ flushes its block, becomes `Invalid`, and $P_1$ enters `Modified`. |
| **5** | **$P_2$: Write $x_1$** | **False Sharing Miss** | **Invalid** | **Modified** | $P_2$ writes to $x_1$. $P_2$'s cache block was invalidated at Time 4 because $P_1$ wrote to $x_2$. However, $P_2$ is modifying $x_1$, which $P_1$ did not write! If block sizes were single words, $P_2$'s write to $x_1$ would not have missed. Thus, this miss is solely caused by sharing the block: a **False Sharing Miss**. $P_1$'s block is invalidated. |
| **6** | **$P_1$: Read $x_2$** | **False Sharing Miss** | **Shared** | **Shared** *(or Owner)* | $P_1$ reads $x_2$. $P_1$'s block was invalidated at Time 5 because $P_2$ wrote $x_1$. But $P_2$ did NOT modify $x_2$ at Time 5! The data for $x_2$ was already updated by $P_1$ itself at Time 4. The invalidation of $x_2$ was collateral damage from $P_2$ writing $x_1$. Hence, this is a **False Sharing Miss**. |
| **7** | **$P_2$: Read $x_1$** | **Cache Hit** | Shared | Shared | $P_2$ already has the block in its cache from Time 5 (when it wrote $x_1$), and $P_1$'s read at Time 6 was a read-only request that left $P_2$'s copy in `Shared` state. The read hits locally with **zero bus traffic**. |

---

## Question 25: Demonstrate How Memory Location `XX21` Can Be Locked Using Atomic Test-and-Set Instruction. [06 Marks]

### 1. The Test-and-Set Concept:
The `TestAndSet` primitive atomically reads the current value at a specified memory address, overwrites the address with `1`, and returns the old value.
$$\text{TestAndSet}(\text{Address}) = \{\text{old} = \text{Memory}[\text{Address}]; \quad \text{Memory}[\text{Address}] = 1; \quad \text{return } \text{old};\}$$

* **Convention:**
  * `0` = Lock is **Free / Unlocked**.
  * `1` = Lock is **Held / Locked**.

---

### 2. Assembly / High-Level Routine to Lock Location `XX21`:

```assembly
# Assume Register R1 holds the memory address XX21
# Register R2 will hold the result of the TestAndSet

        la      R1, XX21              # Load the address of lock variable XX21 into R1

spin_loop:
        TestAndSet  R2, 0(R1)         # Atomically read Memory[XX21] into R2 and write 1 to XX21
        bnez        R2, spin_loop     # If R2 != 0 (lock was already held by someone else), retry / spin

# ─── CRITICAL SECTION ───
# If the branch falls through, R2 was 0:
# The processor successfully acquired the lock and set Memory[XX21] to 1.
        # ... Execute Critical Section Operations ...

# ─── UNLOCK OPERATION ───
unlock:
        sw      R0, 0(R1)             # Atomically release lock by storing 0 back into Memory[XX21]
```

### 3. Operational Walkthrough:
1. **Processor Attempts Lock:** The CPU executes `TestAndSet` on location `XX21`.
2. **If Lock is Free (`Memory[XX21] == 0`):**
   * The instruction loads `0` into `R2`.
   * Simultaneously, it sets `Memory[XX21] = 1`.
   * The conditional branch `bnez R2, spin_loop` tests `0` (false) and does not jump. The CPU immediately enters the critical section.
3. **If Lock is Held (`Memory[XX21] == 1`):**
   * The instruction loads `1` into `R2` and leaves `Memory[XX21] = 1`.
   * The branch `bnez R2, spin_loop` tests `1` (true) and loops back, repeatedly executing the atomic test until another core writes `0` into `XX21`.
4. **Releasing the Lock:** The owning core simply stores `0` into memory location `XX21`.

---

## Question 26: Discuss How Memory Consistency is Maintained in Modern CPUs and GPUs. [06–08 Marks]

### 1. Memory Consistency in Modern CPUs:
Modern multicore CPUs (e.g., Intel x86, AMD Zen, ARMv8, RISC-V) maintain consistency using a combination of hardware memory models and software synchronization primitives:

* **Hardware Architectural Models:**
  * **x86-64 (Total Store Order - TSO):** Hardware enforces that all writes appear in strict program order ($W \to W$), and reads are ordered ($R \to R, R \to W$). Only store-to-load reordering ($W \to R$) is permitted via hardware store buffers.
  * **ARMv8 & RISC-V (Weak / Release Consistency):** All ordinary data memory accesses ($R \to R, R \to W, W \to R, W \to W$) can be freely reordered by out-of-order execution cores to hide memory latency.
* **Enforcement Mechanisms:**
  * **Explicit Memory Fences / Barriers:** The CPU pipeline stalls or drains store buffers when it encounters instructions like `MFENCE` (x86), `DMB` (ARM), or `FENCE rw, rw` (RISC-V).
  * **Acquire-Release Instructions:** ARMv8 (`LDAR` / `STLR`) and RISC-V provide one-way ordering constraints directly tagged onto load and store instructions, preventing instructions inside critical sections from migrating out.

---

### 2. Memory Consistency in Modern GPUs:
GPUs (e.g., NVIDIA architectures like Volta, Ampere, Hopper) feature tens of thousands of concurrent threads executing under SIMT. Enforcing standard CPU-style cache-coherent sequential consistency across all threads would completely destroy memory bandwidth.

* **Relaxed Consistency by Default:**
  * Memory accesses within GPU thread warps and thread blocks are **weakly ordered**.
  * Writes to global or shared memory are not immediately visible to other threads unless explicitly synchronized.
* **Scoped Memory Consistency (CUDA / PTX):**
  * Modern GPUs implement **Scoped Consistency**, where synchronization operations specify the exact scope of threads that must observe memory orderings:
    1. **Thread Scope:** Ordering maintained only within the single executing thread.
    2. **Block Scope (`__syncthreads()`):** Memory barrier and barrier synchronization across all 32-1024 threads residing in the **same Thread Block**.
    3. **Device / GPU Scope (`__threadfence()`):** Ensures all global memory writes made by the calling thread are committed and visible to **all threads across all SMs (Streaming Multiprocessors)** on the entire GPU chip.
    4. **System Scope (`__threadfence_system()`):** Memory writes are guaranteed visible across the PCIe/NVLink bus to both the **host CPU and peer GPUs**.
* **Independent Thread Scheduling (Volta+ Architecture):**
  * NVIDIA Volta introduced per-thread program counters and call stacks, maintaining separate convergence barriers to prevent deadlocks in spinlocks and lock-free data structures.

---

## Question 27: Compare Directory-Based Cache Coherence Protocols with Snooping Protocols. [06–08 Marks]

### Model Comparison Table:

| Evaluation Metric | Snooping Cache Coherence Protocol | Directory-Based Cache Coherence Protocol |
|:---|:---|:---|
| **Interconnect Architecture** | Requires a **shared physical broadcast bus** or centralized broadcast ring. | Operates over **scalable point-to-point interconnect networks** (2D Mesh, Torus, Crossbar, Fat-Tree). |
| **Communication Mechanism** | **Broadcast-based:** Every request, invalidation, and data reply is broadcast to every cache on the bus. | **Unicast / Multicast Point-to-Point:** Messages are routed specifically to the Home directory node and nodes in the sharer set. |
| **State Storage Location** | Coherence state is stored **locally inside each private cache tag array** (e.g., 2 bits for MSI/MESI). | Coherence state is stored in a **Directory Controller** associated with main memory (tracks state + bitmask of sharers). |
| **Bandwidth Consumption** | **Very High:** Broadcast traffic grows quadratically with core count ($O(N^2)$), saturating the interconnect. | **Low & Scalable:** Invalidation messages are directed only to active sharers ($O(\text{Sharers})$), saving network bandwidth. |
| **System Scalability** | **Small Scale:** Efficient only up to **8–16 cores** (Uniform Memory Access / SMPs). | **Large Scale:** Highly scalable to **hundreds or thousands of cores** (Distributed Shared-Memory / NUMA). |
| **Cache Miss Latency** | Low latency: Usually resolved in **2 hops** (Request $\to$ Bus $\to$ Memory / Owner Cache $\to$ Requester). | Higher latency: Can require **3 hops** (Requester $\to$ Home Directory $\to$ Owner Node $\to$ Requester). |
| **Hardware Overhead** | Minimal directory storage overhead; relies on broadcast bus snooping logic. | High memory storage overhead to maintain the directory bit-vector ($N$ bits per memory block for $N$ nodes). |

---

## Question 28: Explain the Role of Memory Fences and Barriers in Enforcing Consistency. [04–06 Marks]

### 1. Why Memory Fences are Needed:
In modern out-of-order processors, compilers reorder instructions during optimization, and hardware executes loads and stores speculatively through store buffers and non-blocking caches. 
* While this preserves single-threaded program correctness, it allows memory operations across different variables to be visible out of order to other parallel threads.
* **Memory Fences (also called Memory Barriers)** are explicit machine instructions provided to programmers and compilers to enforce ordering boundaries.

### 2. Functional Roles of Fences:
1. **Preventing Store-Load Reordering ($W \to R$):**
   * Under relaxed models (like TSO/x86), stores can linger in write buffers while later loads execute. A fence forces the store buffer to completely drain to the cache before any subsequent load can issue.
2. **Preventing Compiler Reordering:**
   * Acts as a barrier that prevents the optimizing compiler from hoisting or sinking memory instructions across the fence line.
3. **Establishing Visibility Boundaries (Release and Acquire):**
   * **Acquire Barrier:** Placed immediately after acquiring a lock; ensures that no reads or writes inside the critical section can be reordered *above* the lock acquisition.
   * **Release Barrier:** Placed immediately before releasing a lock; ensures that all updates made inside the critical section are globally committed and visible *before* the lock release is published.

```
       [ Ordinary Memory Reads / Writes ]
                       │
                       ▼
    ══════════════════════════════════════════
               MEMORY FENCE / BARRIER
    ══════════════════════════════════════════
                       │ (No prior instruction can pass below;
                       ▼  no subsequent instruction can pass above)
       [ Dependent Memory Operations ]
```

---

## Question 29: Discuss Performance Issues and Overheads in Distributed Shared Memory (DSM) Systems. [06 Marks]

### Model Answer:
Distributed Shared Memory (DSM) systems provide a unified global address space across distributed physical memory nodes. While highly scalable, they suffer from four primary architectural performance bottlenecks:

1. **The Three-Hop Cache Miss Latency Penalty:**
   * When node $A$ experiences a miss on a memory block whose Home Directory is on node $B$, and the block is currently held dirty by node $C$ (the Owner):
     * **Hop 1:** Requester $A$ sends a request message across the network to Home Node $B$.
     * **Hop 2:** Home Node $B$ inspects its directory and forwards a fetch-invalidate message to Owner Node $C$.
     * **Hop 3:** Owner Node $C$ writes back the data and sends the block directly to Requester $A$.
   * This multi-hop network round-trip significantly magnifies miss penalties compared to symmetric single-bus systems.

2. **Directory Memory Storage Overhead:**
   * A full bit-vector directory requires 1 bit per node for every cache block in main memory.
   * For a system with $N$ nodes and $M$ memory blocks, directory overhead scales as $N \times M$ bits. In large systems (e.g., 512 nodes), directory storage can consume up to **10% to 20% of total physical DRAM** unless sparse directory optimizations are used.

3. **Hot-Spotting and Contention at Home Directories:**
   * If multiple nodes simultaneously access shared data or synchronization variables whose home directory resides on the same node, that home node’s memory controller and directory logic become severe network bottlenecks (hot-spotting).

4. **Exaggerated False Sharing Penalties:**
   * Because cache block sizes in DSM systems are typically large (64 or 128 bytes) to amortize network packet overhead, multiple independent variables frequently reside in the same block.
   * Interleaving writes to independent variables generates continuous, expensive cross-node directory invalidation and data-forwarding messages across the interconnection network.

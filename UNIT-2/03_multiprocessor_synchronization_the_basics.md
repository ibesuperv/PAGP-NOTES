# Unit 2 — Part 3: Multiprocessor Synchronization: The Basics

---

## 1. Multiprocessor Synchronization: The Hardware Foundation

Synchronization mechanisms in parallel computing (mutex locks, semaphores, barriers) are constructed with user-level software libraries that rely directly on **hardware-supplied atomic synchronization primitives**.

### 1.1 The Need for Atomic Read-Modify-Write
* To coordinate concurrent threads safely without race conditions, the hardware must provide an **uninterruptible (atomic) operation** capable of reading a memory location and updating it in a single logical step.
* If a read and a write to a synchronization variable were executed as two separate non-atomic instructions, two processors could interleave their operations, both observing the lock as free and both mistakenly acquiring it simultaneously.

---

## 2. Basic Hardware Synchronization Primitives

Architectures implement atomic synchronization using either **single-instruction primitives** or **split-instruction pairs**.

### 2.1 Single-Instruction Atomic Primitives
1. **Atomic Exchange (Swap):**
   * Atomically interchanges the value of a register with a value in memory:
     $$\text{Temp} = \text{Memory}[x1]; \quad \text{Memory}[x1] = \text{Reg}[x2]; \quad \text{Reg}[x2] = \text{Temp};$$
   * **Lock Implementation:** Value `0` represents unlocked; `1` represents locked. A processor places `1` in a register and executes an atomic exchange with the lock address.
     * If the returned value is `0`: The processor successfully claimed the lock (and set it to `1`).
     * If the returned value is `1`: The lock was already occupied; the processor failed and must retry.
   * **Arbitration:** If two processors execute exchange simultaneously, the hardware **write serialization mechanism** forces an order: exactly one processor completes first (retrieving `0`), and the other receives `1`.
2. **Test-and-Set:** Tests whether a memory location holds a specific value (e.g., `0`) and sets it to `1` only if the test succeeds.
3. **Fetch-and-Increment:** Atomically reads the current value of a memory address and increments the value in memory, returning the old value. Useful for ticket locks and work-queue indices.

---

### 2.2 Split-Instruction Atomic Primitives: Load-Reserved & Store-Conditional (RISC-V / MIPS)

Implementing a single atomic read-modify-write instruction complicates cache coherence because it requires holding the memory bus or cache block locked across both a read and a write cycle without deadlocking.

Modern RISC architectures (RISC-V, MIPS, ARM `LDREX`/`STREX`) resolve this using a **pair of instructions**:

```
[ lr.d (Load Reserved / Linked) ]
  • Loads value from Memory[rs1] into rd.
  • Hardware sets a Reservation Bit on that cache block.
                  │
                  ▼ (Arbitrary register-register computation)
[ sc.d (Store Conditional) ]
  • Checks if reservation bit is STILL intact.
  • Success (No other store intervened): Writes rs2 to Memory[rs1], returns 0 in rd.
  • Failure (Intervening write or interrupt): Memory UNCHANGED, returns non-zero in rd.
```

1. **`lr` (Load Reserved / Load Linked):**
   * Loads the data from memory address `rs1` into register `rd`.
   * Sets a hardware reservation on that memory address (tracked in a special link/reservation register).
2. **`sc` (Store Conditional):**
   * Attempts to store register `rs2` into memory address `rs1`.
   * **Success:** If no other processor has written to that memory block (and no context switch or interrupt occurred) since the `lr`, the store succeeds, memory is updated, and `sc` writes **`0`** into `rd`.
   * **Failure:** If any other processor wrote to that cache block, the reservation is broken. The store fails, memory is left untouched, and `sc` writes a **non-zero value** into `rd`.

---

### 2.3 Assembly Code Implementations using `lr` and `sc`

#### 1. Atomic Exchange Sequence in RISC-V:
```assembly
# Atomically swap value in x4 with memory location 0(x1)
try:
    mov   x3, x4            # Put exchange value into x3
    lr    x2, 0(x1)         # Load reserved: x2 = Mem[x1], register reservation on x1
    sc    x3, 0(x1)         # Store conditional: try storing x3 into Mem[x1]
    bnez  x3, try           # Branch back to try if sc failed (x3 != 0)
    mov   x4, x2            # Put loaded old value into x4 (Swap complete!)
```

#### 2. Atomic Fetch-and-Increment Sequence in RISC-V:
```assembly
# Atomically increment memory location 0(x1) by 1
try:
    lr    x2, 0(x1)         # Load reserved: x2 = Mem[x1]
    addi  x3, x2, 1         # Increment value in register
    sc    x3, 0(x1)         # Store conditional: try writing incremented value
    bnez  x3, try           # Retry if store failed (x3 != 0)
```

#### Architectural Rules for `lr`/`sc`:
* **Only register-register instructions** are permitted between `lr` and `sc`. Memory loads, stores, or system calls between them can clear the reservation register and create an infinite livelock.
* The instruction count between `lr` and `sc` must be kept minimal to reduce the probability of external interrupts or collisions causing repetitive failure.

---

## 3. Implementing Spinlocks Using Cache Coherence

A **spinlock** is a lock where a processor continuously executes a busy-waiting loop (spins) until the lock variable becomes available. Spinlocks are ideal when critical sections are short and low-latency locking is required.

### 3.1 Naive Spinlock (Atomic Exchange in Spin Loop)
```assembly
    addi  x2, x0, 1         # x2 = 1 (locked value)
lockit:
    EXCH  x2, 0(x1)         # Atomic exchange with memory lock variable
    bnez  x2, lockit        # If returned value != 0, still locked; spin!
```
* **Performance Catastrophe:** Every iteration executes a write (atomic exchange). In a coherent cache system, every write generates an **invalidation request** on the bus/interconnect, constantly stealing the cache block back and forth between spinning cores and flooding the memory system with coherence traffic.

---

### 3.2 Optimized Spinlock: Test-and-Test-and-Set (TTAS)
Instead of executing an atomic write on every spin, the processor spins by executing **local cache reads** until it sees the lock become free (`0`), and only then attempts the atomic exchange:

```assembly
lockit:
    ld    x2, 0(x1)         # Step 1: Read lock variable locally in cache
    bnez  x2, lockit        # Spin-wait on local read (NO bus traffic!)
    addi  x2, x0, 1         # Lock appears free (0); prepare locked value (1)
    EXCH  x2, 0(x1)         # Step 2: Attempt atomic swap
    bnez  x2, lockit        # If swap returned 1, another core won; retry!
```

#### Why Caching the Lock Works:
1. **Zero Bus Traffic During Spinning:** While the lock is held, all spinning processors hold the lock block in the **Shared** state. Repeated reads hit locally in private L1/L2 caches without generating any network or bus traffic.
2. **Processor Locality:** If the same processor repeatedly re-acquires the lock, the lock variable remains in its local cache, resulting in immediate single-cycle lock acquisition.

---

### 3.3 Coherence Steps & Bus Traffic During Lock Release (Figure 5.22)

Consider three processors: $P0$ initially holds the lock, while $P1$ and $P2$ are spinning using Test-and-Test-and-Set.

```
       Timeline of Coherence Events Across P0, P1, P2 during Lock Handoff
P0 (Releases):   Write 0 to Lock ──► Invalidate P1, P2
                                           │
P1, P2 (Miss):   Detect Invalidation ──► Read Miss to fetch updated value (0)
                                           │
Race to Swap:    P2 wins swap (returns 0) ──► Enters Critical Section (Lock = 1)
                 P1 loses swap (returns 1) ──► Re-enters spin loop on local read
```

| Step | Processor $P0$ | Processor $P1$ | Processor $P2$ | Coherence State of Lock | Bus / Interconnect Activity |
| :---: | :--- | :--- | :--- | :---: | :--- |
| **1** | Has lock (value = 1) | Spins reading cached lock | Spins reading cached lock | **Shared** | All caches hold shared copy; zero bus traffic. |
| **2** | **Set lock to 0** (Release) | Receives invalidate | Receives invalidate | **Exclusive ($P0$)** | $P0$ issues **Write Invalidate** across interconnect. |
| **3** | Idle | Cache miss | Cache miss | **Shared** | $P2$'s cache miss serviced first; $P0$ writes back 0. |
| **4** | Idle | Waits for bus | **Test succeeds** (sees 0) | **Shared** | $P2$'s read completes; $P2$ exits spin loop. |
| **5** | Idle | Test succeeds (sees 0) | **Executes swap**; miss | **Shared** | $P1$'s read completes; $P2$ issues write miss for swap. |
| **6** | Idle | Executes swap; miss | **Completes swap** (gets 0, sets 1) | **Exclusive ($P2$)** | $P2$ wins lock! Enters critical section. |
| **7** | Idle | Swap completes (gets 1, sets 1) | **Inside critical section** | **Exclusive ($P1$)** | $P1$ loses race (reads 1 written by $P2$). |
| **8** | Idle | **Spins reading lock** | Inside critical section | **Shared** | $P1$ returns to local cache read spin loop. |

---

### 3.4 Spinlock Implementation Using `lr` and `sc`
Because `lr` and `sc` naturally separate the read from the write, an optimized spinlock requires only 4 instructions:

```assembly
lockit:
    lr    x2, 0(x1)         # Load reserved (local read; sets reservation)
    bnez  x2, lockit        # If lock != 0, spin locally!
    addi  x2, x0, 1         # Prepare locked value (1)
    sc    x2, 0(x1)         # Store conditional (attempt atomic acquisition)
    bnez  x2, lockit        # If sc failed due to collision or race, retry!
```
* **Branch 1 (`bnez x2, lockit` after `lr`):** Forms the local cache spin loop without bus writes.
* **Branch 2 (`bnez x2, lockit` after `sc`):** Resolves the race condition if multiple processors observe the lock as free simultaneously.

---

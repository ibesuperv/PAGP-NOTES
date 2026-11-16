# Unit 1 — Part 9: Control Dependences, Compiler Pipeline Scheduling, and Loop Unrolling

---

## 1. Control Dependences

### 1.1 What is a Control Dependence?
A **control dependence** determines the execution ordering of an instruction with respect to a conditional branch instruction. It ensures that an instruction executes in the correct program order and **only when it is supposed to execute**.

Except for instructions in the very first basic block of a program, **every instruction is control-dependent on some set of branches**.

#### Example
Consider the following high-level code:
```c
if (p1) {
    S1;
}
if (p2) {
    S2;
}
```
* **$S1$** is control-dependent on condition $p1$.
* **$S2$** is control-dependent on condition $p2$, but **not** on $p1$.

---

### 1.2 Two Fundamental Constraints of Control Dependences
Control dependences impose two strict ordering rules on code motion:

1. **Rule 1 (Cannot move before):** An instruction that is control-dependent on a branch **cannot be moved before the branch** such that its execution is no longer controlled by that branch.
   * *Example:* An instruction inside the `then` block of an `if` statement cannot simply be moved ahead of the `if` test.
2. **Rule 2 (Cannot move after):** An instruction that is **not** control-dependent on a branch **cannot be moved after the branch** such that its execution becomes controlled by that branch.
   * *Example:* A statement before an `if` statement cannot be pulled inside the `then` block.

---

### 1.3 Preserving Program Correctness: Exception Behavior and Data Flow
While preserving strict program order naturally satisfies control dependences, modern processors and optimizing compilers often execute instructions out of order or speculatively.

Control dependence itself is **not** the fundamental property that must be preserved at all costs. Instead, program correctness relies on two critical properties:
1. **Exception Behavior**
2. **Data Flow**

```
                         Program Correctness
                                  │
         ┌────────────────────────┴────────────────────────┐
         ▼                                                 ▼
1. Preserving Exception Behavior                 2. Preserving Data Flow
   (No NEW exceptions created)                     (Correct dynamic producer-consumer)
```

---

#### Property 1: Preserving Exception Behavior
Reordering instructions must **never change how exceptions are raised**, and specifically **must not cause new, bogus exceptions** that would not have occurred in the original sequential execution.

Consider this RISC-V code sequence:
```riscv
add  x2, x3, x4      # Calculate address pointer
beq  x2, x0, L1      # If x2 == 0 (null pointer), branch to L1
ld   x1, 0(x2)       # Load data from address in x2
L1:
```

* **The Problem with Violating Control Dependence:**
  * There is **no data dependence** preventing us from moving `ld x1, 0(x2)` before `beq`.
  * However, `ld` is **control-dependent** on `beq`.
  * If `x2 == 0`, sequential execution branches to `L1` without touching memory.
  * If the compiler or hardware moves `ld` before `beq`, loading from address `0` triggers a fatal **memory protection fault (segmentation fault / exception)** that would never have happened in the original program!
* **Solution:** To safely reorder these instructions, the processor or runtime must support **speculation**, which suppresses or ignores speculative exceptions until the branch outcome is definitively known.

---

#### Property 2: Preserving Data Flow
**Data flow** is the actual transmission of data values between instructions that produce results and instructions that consume them. 

Conditional branches make data flow **dynamic** because the source value for a given consumer instruction can originate from multiple paths.

Consider this RISC-V code sequence:
```riscv
      add  x1, x2, x3     # Producer A: computes x1
      beq  x4, x0, L      # Branch test
      sub  x1, x5, x6     # Producer B: overwrites x1 (executed only if branch NOT taken)
L:    ...
      or   x7, x1, x8     # Consumer: uses x1
```

* **Why Data Dependence Alone is Insufficient:**
  * The `or` instruction is data-dependent on `x1`.
  * Both `add` and `sub` write to `x1`.
  * Which instruction delivers the value of `x1` to `or`?
    * If the branch is **taken** $\rightarrow$ `x1` comes from `add`.
    * If the branch is **not taken** $\rightarrow$ `x1` comes from `sub`.
* **Preserving Correctness:**
  * Moving `sub` before the branch would illegally overwrite `x1` on taken branches, destroying the true data flow.
  * Preserving the control dependence of `or` and `sub` ensures the correct value is delivered under all execution paths.

---

### 1.4 Safe Violation of Control Dependences: Liveness and Software Speculation
Under certain conditions, a compiler can safely move an instruction across a branch without violating exception behavior or data flow.

Consider this code sequence:
```riscv
      add  x1, x2, x3
      beq  x12, x0, skip
      sub  x4, x5, x6
      add  x5, x4, x9
skip: or   x7, x8, x9
```

Suppose compiler analysis reveals the following:
1. **Liveness Analysis:** A variable/register is **live** if its current value will be read by an upcoming instruction; otherwise, it is **dead**.
   * If register `x4` is **dead** (never used) after label `skip`, then changing `x4` before the branch does not corrupt subsequent data flow.
2. **Exception Safety:** The `sub` instruction uses integer register arithmetic and cannot cause an unrecoverable memory exception.

* **Compiler Transformation (Software Speculation):**
  * The compiler moves `sub x4, x5, x6` **before** `beq`.
  * The compiler is making a bet that the branch is usually **not taken**.
  * If the branch is taken, the speculative execution of `sub` was useless work, but because `x4` is dead and no exception was raised, **program correctness is completely preserved**.

---

## 2. Basic Compiler Techniques for Exposing ILP

To keep a processor pipeline full and avoid stalls, the compiler must overlap sequences of independent instructions.

### 2.1 Pipeline Latency Model (FP Unit Latencies)
To avoid a pipeline data hazard stall, a dependent instruction must be separated from its source instruction by a distance in clock cycles equal to the **pipeline latency** of the producer.

The standard latencies assumed (based on *Computer Architecture: A Quantitative Approach*, Figure 3.2) are:

| Instruction Producing Result | Instruction Using Result | Latency in Clock Cycles (Intervening Stalls Needed) |
| :--- | :--- | :---: |
| **FP ALU operation** | Another FP ALU operation | **3** |
| **FP ALU operation** | Store double (`fsd`) | **2** |
| **Load double (`fld`)** | FP ALU operation | **1** |
| **Load double (`fld`)** | Store double (`fsd`) | **0** (result bypassed directly) |
| **Integer load (`ld`)** | Integer ALU operation | **1** |
| **Integer ALU operation** | Any integer ALU / Branch | **0** |

* **Branch Delay:** 1 clock cycle (standard 5-stage RISC pipeline).
* **Structural Assumptions:** Functional units are fully pipelined or replicated so an instruction can issue every clock cycle without structural hazards.

---

### 2.2 The Running Baseline Vector Loop
Consider a loop that adds a scalar value $s$ to all elements of a vector:
```c
for (i = 999; i >= 0; i = i - 1)
    x[i] = x[i] + s;
```

#### Loop Parallelism
The body of each iteration is completely independent of every other iteration. This loop has extensive **loop-level parallelism**.

#### RISC-V Register Setup
* `x1`: Initialized to the address of the element with the highest address ($x[999]$).
* `f2`: Holds the scalar constant value $s$.
* `x2`: Precomputed loop limit register such that `Regs[x2] + 8` is the address of the last element ($x[0]$).

#### Straightforward Unscheduled RISC-V Assembly
```riscv
Loop: fld    f0, 0(x1)         # f0 = array element x[i]
      fadd.d f4, f0, f2        # add scalar s (in f2)
      fsd    f4, 0(x1)         # store result back to x[i]
      addi   x1, x1, -8        # decrement pointer by 8 bytes (double precision)
      bne    x1, x2, Loop      # branch if pointer != x2
```

---

## 3. Detailed Step-by-Step Pipeline Scheduling Analysis

### 3.1 Case 1: Unscheduled Loop Execution (9 Clock Cycles)

Without any compiler scheduling, dependent instructions immediately follow one another, creating pipeline stalls:

```
Cycle 1:  fld    f0, 0(x1)
Cycle 2:  [stall]               <-- Load-to-FP ALU latency = 1 cycle
Cycle 3:  fadd.d f4, f0, f2
Cycle 4:  [stall]               <-- FP ALU-to-Store latency = 2 cycles
Cycle 5:  [stall]
Cycle 6:  fsd    f4, 0(x1)
Cycle 7:  addi   x1, x1, -8
Cycle 8:  [stall]               <-- Branch delay / test resolution
Cycle 9:  bne    x1, x2, Loop
```

* **Total Execution Time:** **9 clock cycles per element** (per loop iteration).
* **Efficiency:** Only 3 of the 9 cycles perform useful computational work (`fld`, `fadd.d`, `fsd`). The remaining 6 cycles are overhead (3 stalls + 1 stall/branch penalty + `addi` + `bne`).

---

### 3.2 Case 2: Scheduled Loop Without Unrolling (7 Clock Cycles)

The compiler reorders independent instructions within the single iteration body to fill stall slots.

#### The Optimization:
* Move `addi x1, x1, -8` **between** `fld` and `fadd.d`.
  * The `addi` instruction executes during Cycle 2, completely absorbing the 1-cycle load latency!
* Because `x1` is decremented early, the store address must be adjusted from `0(x1)` to `8(x1)` to write to the correct memory location.

#### Scheduled Assembly Code and Execution:
```riscv
Loop: fld    f0, 0(x1)         # Cycle 1: load element
      addi   x1, x1, -8        # Cycle 2: pointer decrement (fills load stall!)
      fadd.d f4, f0, f2        # Cycle 3: add scalar
      stall                    # Cycle 4: stall (FP ALU -> Store latency = 2)
      stall                    # Cycle 5: stall
      fsd    f4, 8(x1)         # Cycle 6: store result (compensated offset +8)
      bne    x1, x2, Loop      # Cycle 7: branch
```

* **Total Execution Time:** **7 clock cycles per element**.
* **Breakdown:**
  * Useful computational work: **3 cycles** (`fld`, `fadd.d`, `fsd`).
  * Loop overhead & stalls: **4 cycles** (2 stalls + 1 `addi` + 1 `bne`).
* **Limitation:** Within a single iteration, there are no other independent instructions available to fill the 2 stalls between `fadd.d` and `fsd`. To eliminate those stalls, instructions must be extracted from **other iterations**.

---

## 4. Loop Unrolling

### 4.1 Concept and Purpose
**Loop unrolling** replicates the loop body multiple times and adjusts the loop termination and pointer modification code.

Loop unrolling improves performance in two major ways:
1. **Reduces Loop Overhead:** Amortizes the branch (`bne`) and index calculation (`addi`) across multiple loop iterations.
2. **Exposes ILP for Scheduling:** Creates a large pool of independent instructions from adjacent iterations that can be interleaved to completely eliminate pipeline stalls.

---

### 4.2 Compiler Transformation Steps
When unrolling a loop by a factor of 4:
1. **Replicate the loop body 4 times.**
2. **Register Renaming (Eliminating False Dependences):**
   * Use different floating-point registers for each iteration copy (e.g., `f0/f4`, `f6/f8`, `f10/f12`, `f14/f16`).
   * *Why?* If all copies reused `f0` and `f4`, name dependences (WAR and WAW hazards) would prevent the compiler from moving instructions across iterations.
3. **Symbolic Substitution and Simplification:**
   * Merge four separate pointer decrements (`addi x1, x1, -8`) into a single decrement: `addi x1, x1, -32`.
   * Adjust the memory offsets in the load and store instructions:
     * Iteration 1: `0(x1)`
     * Iteration 2: `-8(x1)`
     * Iteration 3: `-16(x1)`
     * Iteration 4: `-24(x1)`
   * Drop 3 redundant branches and 3 pointer decrements.

---

### 4.3 Case 3: 4-Way Unrolled Loop — Unscheduled (26 Clock Cycles / 6.5 Cycles per Element)

#### Assembly Code (Unscheduled Unrolled Loop):
```riscv
Loop: fld    f0, 0(x1)
      fadd.d f4, f0, f2
      fsd    f4, 0(x1)

      fld    f6, -8(x1)
      fadd.d f8, f6, f2
      fsd    f8, -8(x1)

      fld    f10, -16(x1)
      fadd.d f12, f10, f2
      fsd    f12, -16(x1)

      fld    f14, -24(x1)
      fadd.d f16, f14, f2
      fsd    f16, -24(x1)

      addi   x1, x1, -32       # merged pointer decrement
      bne    x1, x2, Loop      # single branch for 4 elements
```

#### Cycle Count Calculation:
* **Instruction Issue Cycles:**
  * 4 loads + 4 FP adds + 4 stores + 1 addi + 1 bne = **14 instructions** = **14 cycles**.
* **Pipeline Stall Cycles:**
  * Each `fld` followed immediately by `fadd.d` produces **1 stall cycle**:
    $$\text{Load stalls} = 4 \times 1 = 4\text{ cycles}$$
  * Each `fadd.d` followed immediately by `fsd` produces **2 stall cycles**:
    $$\text{Add stalls} = 4 \times 2 = 8\text{ cycles}$$
* **Total Clock Cycles:**
  $$\text{Total Cycles} = 14 + 4 + 8 = 26\text{ clock cycles for 4 elements}$$
* **Time per Element:**
  $$\frac{26\text{ cycles}}{4\text{ elements}} = \mathbf{6.5\text{ clock cycles per element}}$$

Unrolling alone without scheduling reduced the per-element time from 9.0 (or 7.0) to 6.5 cycles simply by removing 3 branches and 3 pointer updates.

---

### 4.4 Case 4: 4-Way Unrolled Loop — Scheduled (14 Clock Cycles / 3.5 Cycles per Element)

Because each unrolled iteration uses distinct registers, instructions from different iterations are completely independent. The compiler can reschedule the unrolled loop by **interleaving operations across iterations**:
1. Issue all 4 independent loads first.
2. Issue all 4 independent FP additions.
3. Issue the 4 stores, along with the loop counter decrement and branch.

#### Scheduled Assembly Code:
```riscv
Loop: fld    f0, 0(x1)         # Cycle 1: load element 0
      fld    f6, -8(x1)        # Cycle 2: load element 1
      fld    f10, -16(x1)      # Cycle 3: load element 2
      fld    f14, -24(x1)      # Cycle 4: load element 3

      fadd.d f4, f0, f2        # Cycle 5: add 0 (issued 3 cycles after f0 load -> 0 stalls!)
      fadd.d f8, f6, f2        # Cycle 6: add 1 (issued 3 cycles after f6 load -> 0 stalls!)
      fadd.d f12, f10, f2      # Cycle 7: add 2 (issued 3 cycles after f10 load -> 0 stalls!)
      fadd.d f16, f14, f2      # Cycle 8: add 3 (issued 3 cycles after f14 load -> 0 stalls!)

      fsd    f4, 0(x1)         # Cycle 9: store 0 (issued 3 cycles after f4 add -> 0 stalls!)
      fsd    f8, -8(x1)        # Cycle 10: store 1 (issued 3 cycles after f8 add -> 0 stalls!)
      addi   x1, x1, -32       # Cycle 11: decrement pointer by 32
      fsd    f12, 16(x1)       # Cycle 12: store 2 (-16 + 32 = +16 relative to decremented x1)
      fsd    f16, 8(x1)        # Cycle 13: store 3 (-24 + 32 = +8 relative to decremented x1)
      bne    x1, x2, Loop      # Cycle 14: branch if not finished
```

#### Why There are ZERO Pipeline Stalls:
* **Load to FP Add:** `fadd.d f4` executes at Cycle 5, which is 4 cycles after `fld f0` (Cycle 1). The required 1-cycle latency is completely satisfied.
* **FP Add to Store:** `fsd f4` executes at Cycle 9, which is 4 cycles after `fadd.d f4` (Cycle 5). The required 2-cycle latency is completely satisfied.
* **Offset Compensation:** Moving `addi x1, x1, -32` before the final two stores allows `addi` to do useful work in Cycle 11; the stores for elements 2 and 3 simply use compensated offsets ($+16$ and $+8$) relative to the new pointer.

#### Performance Result:
* **Total Cycles:** **14 clock cycles for 4 elements**.
* **Stalls:** **0 cycles**.
* **Time per Element:**
  $$\frac{14\text{ cycles}}{4\text{ elements}} = \mathbf{3.5\text{ clock cycles per element}}$$

---

### 4.5 Summary Comparison of Optimization Stages

| Optimization Strategy | Total Clock Cycles | Elements Processed | Clock Cycles per Element | Speedup vs. Baseline |
| :--- | :---: | :---: | :---: | :---: |
| **1. Unscheduled Single Loop** | 9 | 1 | **9.0** | $1.00\times$ (Baseline) |
| **2. Scheduled Single Loop** | 7 | 1 | **7.0** | $1.29\times$ |
| **3. Unrolled (4x), Unscheduled** | 26 | 4 | **6.5** | $1.38\times$ |
| **4. Unrolled (4x) & Scheduled** | 14 | 4 | **3.5** | **$2.57\times$** |

---

## 5. Handling Unknown Loop Bounds ($n \pmod k$)

In real programs, the total loop trip count $n$ is rarely a compile-time constant multiple of the unrolling factor $k$.

### The Two-Loop Strategy
If array size is $n$ and the unrolling factor is $k$:
1. **Cleanup / Remainder Loop:**
   * Executes $n \pmod k$ iterations using the standard (non-unrolled) loop body.
   * Handles the leftover elements (e.g., if $n = 103$ and $k = 4$, this loop runs $103 \pmod 4 = 3$ times).
2. **Main Unrolled Loop:**
   * An outer loop executing $\lfloor n / k \rfloor$ iterations, each running the $k$-way unrolled and scheduled loop body.
   * Handles the bulk of the computation ($100 / 4 = 25$ iterations).

```
Total Iterations = n, Unroll Factor = k

Step 1: Compute (n mod k) elements
        ┌────────────────────────────────────────────────────────┐
        │ Cleanup Loop: runs (n mod k) times (original body)     │
        └────────────────────────────────────────────────────────┘
                                   │
                                   ▼
Step 2: Compute remaining elements
        ┌────────────────────────────────────────────────────────┐
        │ Unrolled Loop: runs (n / k) times with k copies body   │
        └────────────────────────────────────────────────────────┘
```

* For large $n$, almost all runtime is spent inside the stall-free unrolled loop body.
* **Vector Architecture Parallel:** This compiler transformation is directly analogous to **strip mining** used in vector processors.

---

## 6. Compiler Decisions and Limits of Loop Unrolling

To transform a simple loop into an aggressively unrolled, scheduled loop, a compiler (or hardware engine) must methodically make five critical decisions:
1. **Verify Iteration Independence:** Prove that iterations are independent of one another (except for the loop maintenance counter).
2. **Register Renaming:** Use distinct architectural registers across iteration copies to break artificial name dependences (anti-dependences / WAR, output dependences / WAW).
3. **Eliminate Redundant Overhead:** Remove duplicate branch instructions and loop maintenance code, adjusting termination bounds accordingly.
4. **Memory Disambiguation (Address Analysis):** Inspect memory address offsets to confirm that loads and stores from different iterations do not refer to identical memory locations (no memory aliasing), permitting them to be safely reordered.
5. **Dependence-Preserving Scheduling:** Reorder instructions across iterations to fill latency delay slots while strictly maintaining true data dependences.

---

### Three Factors Limiting Gains from Loop Unrolling

Despite its advantages, loop unrolling cannot be scaled indefinitely due to three practical constraints:

#### 1. Decreasing Amortization of Loop Overhead (Diminishing Returns)
* In the 4-way unrolled scheduled loop (14 cycles total), only 2 cycles are overhead (`addi` and `bne`), which equals $\frac{2}{4} = 0.5$ cycles per element.
* Unrolling 8 times reduces this overhead to $\frac{2}{8} = 0.25$ cycles per element (saving only 0.25 cycles).
* Successive unrolling yields progressively smaller performance gains while overhead costs multiply.

#### 2. Code Size Expansion and Instruction Cache Misses
* Replicating the loop body multiplies code size.
* For larger loops, bloated code increases instruction cache (I-cache) miss rates, which can easily wipe out any execution gains from unrolling.

#### 3. Register Pressure (The Primary Bottleneck)
* Aggressive scheduling separates instruction producers from consumers, which significantly increases the number of simultaneously **live values**.
* When the number of live values exceeds the number of physical/architectural registers, the compiler experiences **register pressure**.
* Inability to allocate all live variables forces the compiler to **spill registers to memory** (inserting additional loads and stores), severely degrading execution performance.


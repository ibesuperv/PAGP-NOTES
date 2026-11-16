# 08: Instruction-Level Parallelism (ILP) and Dependences

---

## 1. What is Instruction-Level Parallelism (ILP)?

**Instruction-Level Parallelism (ILP)** is the overlap of independent instructions during execution. Processors exploit ILP so that multiple instructions are evaluated simultaneously.

### 1.1 Two Approaches to Exploit ILP

1. **Dynamic Hardware-Based Approach:**
   * The processor hardware discovers and exploits parallelism at runtime as the program executes.
   * Dominates desktop, server, and smartphone processors (e.g., modern Intel, AMD, and ARM chips).
2. **Static Compiler-Based Approach:**
   * The software compiler analyzes code and rearranges instructions to find parallelism at compile time.
   * *Real-world experience:* Used in architectures like the Intel Itanium (VLIW). It proved very difficult for general-purpose programs and was successful only in scientific and domain-specific computing.

---

### 1.2 Pipeline CPI Breakdown Equation

The actual CPI of a pipelined processor is the sum of the ideal baseline CPI and all pipeline stalls:

$$\text{Pipeline CPI} = \text{Ideal Pipeline CPI} + \text{Structural Stalls} + \text{Data Hazard Stalls} + \text{Control Stalls}$$

#### How Different Techniques Reduce CPI Stalls:

| Technique | Component of CPI Reduced |
| :--- | :--- |
| **Forwarding & Bypassing** | Potential data hazard stalls |
| **Branch Prediction & Delayed Branches** | Control hazard stalls |
| **Compiler Pipeline Scheduling** | Data hazard stalls |
| **Dynamic Scheduling (Scoreboard / Tomasulo)** | Data hazard stalls from true dependences |
| **Loop Unrolling** | Control hazard stalls |
| **Dynamic Scheduling with Register Renaming** | Stalls from WAW (output) and WAR (anti) dependences |
| **Hardware Speculation** | Data hazard and control hazard stalls |
| **Dynamic Memory Disambiguation** | Data hazard stalls involving memory loads/stores |
| **Multiple Issue (Superscalar / VLIW)** | Lowers the Ideal Pipeline CPI (allows $\text{CPI} < 1$, $\text{IPC} > 1$) |

---

## 2. Basic Blocks vs. Loop-Level Parallelism

Where does ILP come from in a program?

### 2.1 The Limit of a Basic Block
* **Definition of a Basic Block:** A straight-line sequence of instructions with:
  * Only **one entry point** (no branches jump into the middle).
  * Only **one exit point** (no branches jump out except at the end).
* **The Problem:** In typical RISC programs, **15% to 25% of instructions are branches**. This means only **3 to 6 instructions** execute between any two branches.
* Because instructions inside such a small window often depend on one another, the available ILP within a single basic block is very small. To get large speedups, we must find parallelism **across multiple basic blocks**.

---

### 2.2 Loop-Level Parallelism
The most common way to get parallelism across basic blocks is across loop iterations:

```c
for (int i = 0; i < 1000; i++) {
    x[i] = x[i] + y[i];
}
```

* **Loop-Level Parallelism:** Every iteration of this loop is completely independent of every other iteration. All 1,000 additions can execute in parallel!
* **How Hardware & Compilers Exploit It:**
  1. **Loop Unrolling:** Replicating the loop body multiple times (either statically by the compiler or dynamically by the hardware) to expose more independent instructions.
  2. **SIMD & Vector Instructions:** Using a single instruction to operate on multiple array elements at once (e.g., processing 4 to 8 elements per instruction using AVX or GPUs).

---

## 3. Dependences vs. Hazards (Crucial Difference)

Students often confuse dependences and hazards:

* **A Dependence is a property of the program:** It depends strictly on the program's source code and logic. It tells us what must be calculated before what.
* **A Hazard is a property of the pipeline organization:** A dependence only turns into a hazard (and causes stalls) if the hardware overlaps instructions in a way that executes them out of order.

### A Data Dependence Tells Us Three Things:
1. The **possibility** of a hazard occurring.
2. The **strict order** in which calculations must take place.
3. The **upper limit (ceiling)** on how much parallelism can possibly be extracted from the code.

---

## 4. Types of Program Dependences

Programs exhibit three types of dependences:

```
                          Program Dependences
        +--------------------------+--------------------------+
        |                                                     |
1. Data Dependences                                   2. Name Dependences
   (True Dependences)                                    (False Dependences)
   - Real data flows from i to j                         - Same register/memory name used
   - Cannot be renamed away                              - Can be eliminated by RENAMING!
                                                         +------------+------------+
                                                         |                         |
                                                 Antidependence           Output Dependence
                                                 (i reads, j writes)     (both i & j write)
```

---

### 4.1 Data Dependences (True Data Dependences)
Instruction $j$ is data-dependent on instruction $i$ if:
* Instruction $i$ produces a result used by instruction $j$, **OR**
* Instruction $j$ is dependent on instruction $k$, and instruction $k$ is dependent on instruction $i$ (a transitive chain of dependences).

#### Concrete Code Example (RISC-V Vector Increment Loop):
```assembly
Loop:
    fld    f0, 0(x1)         ; 1. Load array element into f0
    fadd.d f4, f0, f2        ; 2. Add scalar f2 to f0 -> result in f4
    fsd    f4, 0(x1)         ; 3. Store f4 back to memory
    addi   x1, x1, -8        ; 4. Decrement address pointer x1 by 8 bytes
    bne    x1, x2, Loop      ; 5. Branch if x1 != x2
```

* **Floating-Point True Dependence Chain:**
  $$\text{fld } (f0) \longrightarrow \text{fadd.d } (f0 \to f4) \longrightarrow \text{fsd } (f4)$$
  `fadd.d` cannot run until `fld` produces `f0`. `fsd` cannot run until `fadd.d` produces `f4`.
* **Integer True Dependence Chain:**
  $$\text{addi } (x1) \longrightarrow \text{bne } (x1)$$
  `bne` cannot evaluate until `addi` updates `x1`.

> **Key Rule:** True data dependences **cannot** be eliminated by renaming. The only way to prevent them from causing pipeline stalls is **instruction scheduling** (interleaving other independent instructions in between).

---

### 4.2 Name Dependences (False Dependences)
A **Name Dependence** occurs when two instructions use the same register or memory address (the same "name"), but **no actual data flows between them**.

Because no data is passed, **name dependences can be completely eliminated by changing the register name (Register Renaming)**!

There are two types of Name Dependences:

#### 1. Antideependence (WAR)
* Instruction $j$ writes to a register that an earlier instruction $i$ reads.
* The original order must be preserved so that instruction $i$ reads the old, correct value before instruction $j$ overwrites it.
* *Example from the code above:*
  ```assembly
  fsd  f4, 0(x1)     ; Instruction i: READS pointer x1 to find store address
  addi x1, x1, -8    ; Instruction j: WRITES new value into x1
  ```
  `fsd` reads `x1`, and `addi` writes `x1`. If we rename `addi` to write into a new register `x3`, the antideependence disappears!

#### 2. Output Dependence (WAW)
* Both instruction $i$ and instruction $j$ write to the **same register or memory location**.
* The original order must be preserved so that the final value left in the register belongs to the later instruction $j$.
* If instruction $j$ is renamed to write to a different register, the output dependence is eliminated.

---

### 4.3 Registers vs. Memory Dependences
* **Registers:** Dependences are easy to detect because register numbers (`x1`, `f4`) are fixed directly inside the 32-bit instruction.
* **Memory:** Dependences are much harder to detect because two instructions might use different register formulas that point to the exact same memory address (e.g., `100(x4)` and `20(x6)` could point to byte 5000). This problem is known as **memory aliasing**.

---

## 5. Relationship Between Dependences and Pipeline Data Hazards

Hazards are named based on the program order of read/write accesses that the pipeline must protect:

### 5.1 RAW (Read After Write) Hazard
* **Corresponds to:** **True Data Dependence**.
* **Problem:** Instruction $j$ tries to read a source operand before instruction $i$ writes it.
* **Result if not prevented:** Instruction $j$ incorrectly reads the old, stale value.
* **Frequency:** The most common data hazard.

### 5.2 WAW (Write After Write) Hazard
* **Corresponds to:** **Output Dependence**.
* **Problem:** Instruction $j$ tries to write its result before an earlier instruction $i$ finishes writing its result.
* **Result if not prevented:** The final register contains the older value from instruction $i$ instead of the newer value from instruction $j$.
* **In 5-Stage RISC:** **Cannot occur** in a simple 5-stage in-order pipeline because all writes happen strictly in the WB stage. Occurs when pipelines allow instructions to finish out-of-order or have multi-cycle execution stages.

### 5.3 WAR (Write After Read) Hazard
* **Corresponds to:** **Antideependence**.
* **Problem:** Instruction $j$ tries to write a destination before an earlier instruction $i$ has read it.
* **Result if not prevented:** Instruction $i$ incorrectly reads the new value instead of the old value.
* **In 5-Stage RISC:** **Cannot occur** in standard in-order pipelines because all reads happen early (in ID) and all writes happen late (in WB). Occurs only when instructions are executed out of program order (dynamically scheduled pipelines).

> **Note:** **RAR (Read After Read)** is **never a hazard** because reading the same value multiple times never modifies machine state.

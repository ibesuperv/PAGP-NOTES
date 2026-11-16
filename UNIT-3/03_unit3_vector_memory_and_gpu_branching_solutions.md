# Unit – III: Vector Architectures, Memory Banks & GPU Branching
## Exam Question Bank Solutions (Part 3)

**Syllabus Mapping:**
* **Vector Architecture:** VMIPS components, Vector Length Register (`VLR`), Vector Mask Register (`VMR`), Strided access, Memory bank conflicts & latency calculation, MIPS vs. VMIPS DAXPY.
* **Graphics Processing Units (GPUs):** GPU conditional branching & branch divergence, NVIDIA Fermi architectural innovations.

---

# SECTION A: SHORT QUESTIONS (Unit-3 Only)

---

### Question 1.1 [01 Mark]
> **A vector processor can handle strides greater than one, called `_________`.**

#### Answer:
**non-unit stride** (or **strided addressing**).

#### Explanation:
* **Unit Stride (Stride = 1):** Successive vector elements occupy contiguous memory addresses ($A[0], A[1], A[2], \dots$).
* **Non-Unit Stride (Stride > 1):** Elements are separated by a constant step size in memory ($A[0], A[k], A[2k], \dots$, e.g., accessing a column of a row-major matrix). Special vector instructions (`LVWS` — Load Vector With Stride, `SVWS` — Store Vector With Stride) handle this by computing addresses as $\text{Base} + i \times \text{Stride}$.

---

### Question 1.5 [02 Marks]
> **Define chaining.**

#### Answer:
**Chaining** is a vector hardware pipelining optimization (analogous to scalar forwarding) that allows a dependent vector operation to begin execution as soon as the **first individual result element** of its source vector operand is produced by an earlier vector functional unit, rather than waiting for the entire vector register (all 64 elements) to finish computing. It enables dependent operations to be executed concurrently within the same **convoy**.

---

# SECTION B: LONG & NUMERICAL QUESTIONS (Unit-3 Only)

---

## Question 3.a [06 Marks]
> **Describe the basic structure of a vector architecture, VMIPS consisting of all the primary components.**

### VMIPS Structural Block Diagram:

```
                         Scalar Processor Core
                   (R0-R31, F0-F31, Scalar Registers)
                                   │
                                   ▼
    ══════════════════════════════════════════════════════════════════
                         VECTOR CONTROL & BUSES
    ══════════════════════════════════════════════════════════════════
            │                         ▲                    │
            │                         │ Chaining Buses     ▼
            ▼                         │           ┌─────────────────┐
 ┌───────────────────────────────────────┐        │Control Registers│
 │        Vector Register File           │        │ • VLR (Length)  │
 │  8 Vector Registers (V0 - V7)         │        │ • VMR (Mask)    │
 │  Each register holds 64 elements      │        └─────────────────┘
 │  Each element is 64-bit FP double     │
 └──────────────────┬────────────────────┘
                    │
       ┌────────────┼────────────┬────────────┐
       ▼            ▼            ▼            ▼
 ┌───────────┐┌───────────┐┌───────────┐┌───────────┐
 │ Vector    ││ Vector    ││ Vector    ││ Vector    │
 │ FP Add    ││ FP Mult   ││ FP Divide ││ Integer / │
 │ Unit      ││ Unit      ││ Unit      ││ Logical   │
 └─────┬─────┘└─────┬─────┘└─────┬─────┘└─────┬─────┘
       │            │            │            │
       └────────────┼────────────┴────────────┘
                    ▼
    ══════════════════════════════════════════════════════════════════
                       VECTOR LOAD / STORE UNIT
    ══════════════════════════════════════════════════════════════════
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │      Memory Interconnect      │
                    └───────────────┬───────────────┘
                                    ▼
        ┌───────────────┬───────────────┬───────────────┐
        │ Memory Bank 0 │ Memory Bank 1 │ Memory Bank N │
        └───────────────┴───────────────┴───────────────┘
```

### Primary Hardware Components:
1. **Vector Register File:**
   * Consists of **8 vector registers ($V0$ through $V7$)**.
   * Each vector register holds **64 elements**, where each element is a **64-bit double-precision** floating-point or integer value.
   * Multi-ported to support concurrent reads and writes across multiple functional units.
2. **Vector Functional Units:**
   * Fully pipelined arithmetic/logical units:
     * **Vector FP Add / Subtract Unit**
     * **Vector FP Multiply Unit**
     * **Vector FP Divide Unit**
     * **Vector Integer / Logical Unit**
   * Can initiate a new element operation every clock cycle.
3. **Vector Control Registers:**
   * **Vector Length Register (`VLR`):** Dynamically controls how many elements (from $1$ to $64$) a vector instruction operates on.
   * **Vector Mask Register (`VMR`):** A 64-bit register where each bit selectively enables or disables operations on the corresponding vector element for conditional code.
4. **Vector Load / Store Unit:**
   * Fully pipelined memory interface capable of moving elements between memory and vector registers at a rate of 1 word per clock cycle after initial memory latency.
   * Supports unit stride, non-unit stride (`LVWS`/`SVWS`), and gather/scatter indexed addressing (`LVI`/`SVI`).
5. **Memory System:**
   * Multi-bank interleaved memory system that eliminates memory cycle-time stalls.

---

## Question 3.b [06 Marks]
> **Identify the significance of Vector Length Register (`VLR`) and Vector Mask Register (`VMR`).**

---

### 1. Vector Length Register (`VLR`) [03 Marks]

#### Significance & Problem Solved:
* In a hardware vector architecture, the physical vector registers have a fixed maximum length ($\text{MVL} = 64$ elements).
* However, real-world application loops rarely have trip counts that are exact multiples of 64 (e.g., $N = 37$ or $N = 100$).
* **Function:** The `VLR` is a special scalar-accessible control register that specifies the **active vector length** for all subsequent vector instructions ($1 \le \text{length} \le \text{MVL}$).

#### Role in Strip Mining:
* When vectorizing a loop of arbitrary size $N$:
  1. The compiler executes a setup instruction `mtc1 VLR, Rlen` to set `VLR` to the remainder $N \pmod{\text{MVL}}$ (e.g., $100 \pmod{64} = 36$).
  2. The first pass executes vector instructions on only the first 36 elements.
  3. The `VLR` is then set to $\text{MVL} = 64$ for all subsequent passes.
* **Benefit:** Eliminates the need for separate scalar cleanup code while preventing vector instructions from writing past the valid array boundary.

---

### 2. Vector Mask Register (`VMR`) [03 Marks]

#### Significance & Problem Solved:
* In standard loops containing conditional `if` statements, different vector elements take different execution paths:
  ```c
  for (i = 0; i < 64; i++) {
      if (A[i] != 0)
          A[i] = A[i] - B[i];
  }
  ```
* Standard scalar branching would destroy vectorization because branches break the continuous instruction stream.
* **Function:** The `VMR` is a **64-bit register** where each bit corresponds to one element of a 64-element vector register.
  * Bit $i = 1$: The operation is performed and the result is written to destination element $i$.
  * Bit $i = 0$: The operation for element $i$ is suppressed (masked out), and the original destination element remains unchanged.

#### Vector Mask Instructions:
1. `SNEVS.D V1, F0` (Set vector-scalar if Not Equal): Compares vector $V1$ with scalar $F0$ (0.0) and writes a 64-bit boolean mask into `VMR`.
2. `SUBVV.D V1, V1, V2`: Subtracts $V2$ from $V1$, but updates **only** those elements where the corresponding bit in `VMR` is `1`.
* **Benefit:** Replaces control branching with **predication**, allowing loops with conditional logic to be fully vectorized without pipeline branch stalls.

---

## Question 3.c [04 Marks]
> **Briefly explain conditional branching in GPUs.**

### Answer:

In GPU architectures (e.g., NVIDIA CUDA), execution follows the **SIMT (Single Instruction, Multiple Threads)** model. Threads in a warp (32 threads) share a single instruction decoder and execute in lock-step.

```
                  Conditional Branch: if (threadIdx.x < 16)
                                     │
         ┌───────────────────────────┴───────────────────────────┐
         ▼                                                       ▼
  Threads 0 to 15 (Active)                                Threads 16 to 31 (Active)
  Threads 16 to 31 (Masked OFF)                           Threads 0 to 15 (Masked OFF)
  ┌───────────────────────────────┐                       ┌───────────────────────────────┐
  │   Execute THEN-path Code      │                       │   Execute ELSE-path Code      │
  └───────────────────────────────┘                       └───────────────────────────────┘
                                     │
                                     ▼
                           Both paths serialized!
                 Threads recombine at branch convergence
```

### Key Mechanisms:

1. **Warp Divergence:**
   * When threads within the same warp evaluate a data-dependent conditional branch differently (e.g., `if (x > 0)` where some threads are true and others are false), the warp **diverges**.
2. **Path Serialization:**
   * The GPU hardware cannot execute both the `then` and `else` branches simultaneously on the same warp.
   * Instead, it **serializes** execution:
     1. It executes the `then` branch: threads with condition `false` are masked off (disabled).
     2. It executes the `else` branch: threads with condition `true` are masked off.
3. **Hardware Predication & Mask Stack:**
   * The GPU maintains an active thread mask stack.
   * Branch instructions push divergent mask states onto the stack. At the reconvergence point, the full active mask is popped back.
4. **Performance Impact:**
   * If an `if-else` path has balanced divergence (16 threads take `then`, 16 take `else`), execution time doubles, and SIMD efficiency drops to $50\%$.
   * **Optimization Goal:** Avoid divergent control flow within warps by grouping threads with identical branch behavior into the same warp.

---

## Question 4.a [06 Marks]
> **Show the code for MIPS and VMIPS for the DAXPY loop ($Y = a \times X + Y$). Assume that the starting addresses of $X$ and $Y$ are in $Rx$ and $Ry$ respectively.**

### 1. Scalar MIPS Assembly Code (Iterates 64 times)
Assume $F0$ contains scalar $a$, $Rx$ and $Ry$ hold the base addresses of arrays $X$ and $Y$, and $R1$ holds the terminal address (`Rx + 512` bytes):

```assembly
Loop:
    L.D    F2, 0(Rx)        # Load X[i] from memory into F2
    MUL.D  F2, F2, F0       # Multiply: a * X[i]
    L.D    F4, 0(Ry)        # Load Y[i] from memory into F4
    ADD.D  F4, F4, F2       # Add: a * X[i] + Y[i]
    S.D    F4, 0(Ry)        # Store result into Y[i]
    DADDIU Rx, Rx, #8       # Increment X pointer by 8 bytes (double-precision)
    DADDIU Ry, Ry, #8       # Increment Y pointer by 8 bytes
    DSUBU  R2, R1, Rx       # Check if end of array reached
    BNEZ   R2, Loop         # Branch back if not zero
```
* **Dynamic Instruction Count:** $64 \times 9 = \mathbf{576\text{ instructions}}$ executed.

---

### 2. Vector VMIPS Assembly Code (Single Convoy Sequence)
Assume $F0$ contains scalar $a$, and vector length register `VL` is set to 64:

```assembly
LV      V1, Rx          # Load 64 double-precision words of X into vector register V1
MULVS.D V2, V1, F0      # Vector-Scalar Multiply: V2[i] = V1[i] * F0
LV      V3, Ry          # Load 64 double-precision words of Y into vector register V3
ADDVV.D V4, V2, V3      # Vector-Vector Add: V4[i] = V2[i] + V3[i]
SV      V4, Ry          # Store vector V4 back to memory at Ry
```
* **Dynamic Instruction Count:** Exactly **$5\text{ vector instructions}$** (a $115\times$ reduction in instruction fetch and decode traffic!).

---

## Question 4.b [06 Marks]
> **Illustrate the several innovations introduced in Fermi to bring GPUs much closer to mainstream system processors.**

### NVIDIA Fermi (GF100, 2010) Innovations:

```
                            FERMI INNOVATIONS
 ┌────────────────────────────────────────────────────────────────────────┐
 │ 1. True C/C++ Unified 64-bit Virtual Address Space                     │
 │ 2. Fast IEEE 754-2008 Double-Precision Floating-Point (FMA)            │
 │ 3. Unified Two-Level Cache Hierarchy (Configurable L1 / Shared + L2)   │
 │ 4. Full ECC (Error Correcting Code) Protection on DRAM & SRAM          │
 │ 5. Faster Context Switching & Dual Concurrent Kernel Execution         │
 └────────────────────────────────────────────────────────────────────────┘
```

1. **Unified Memory Addressing & 64-Bit Address Space:**
   * Prior GPUs used disjoint address spaces with segmented addressing. Fermi introduced a unified, flat 64-bit virtual address space.
   * Enabled standard C/C++ pointers to seamlessly reference registers, shared memory, and global DRAM interchangeably, unlocking full support for C++ features like virtual functions and templates.
2. **High-Performance IEEE 754-2008 Double-Precision Arithmetic:**
   * Provided dedicated FP64 units delivering up to **$0.5\times$ the throughput of FP32** (compared to $0.1\times$ in earlier architectures).
   * Implemented **Fused Multiply-Add (FMA)** in hardware, retaining full precision across multiplication and addition with a single rounding step.
3. **True Two-Level Cache Hierarchy (L1 + L2):**
   * First GPU architecture to integrate on-chip hardware-managed caching:
     * **Configurable L1 Cache / Shared Memory (64 KB per SM):** Programmers could configure it as $48\text{ KB Shared} / 16\text{ KB L1}$ or $16\text{ KB Shared} / 48\text{ KB L1}$.
     * **Unified 768 KB L2 Cache:** Coherent across all SMs; automatically caches global memory reads and stores.
4. **Full ECC (Error-Correcting Code) Memory Protection:**
   * Added hardware ECC support for register files, shared memory, L1/L2 caches, and off-chip GDDR5 DRAM.
   * Prevented silent single-bit soft errors from corrupting massive, multi-day scientific and datacenter simulations.
5. **Concurrent Kernel Execution:**
   * Allowed multiple distinct CUDA kernels from the same application to execute simultaneously on different SMs, maximizing GPU utilization for smaller tasks.

---

## Question 4.c [06 Marks]
> **Suppose we have 8 memory banks with a bank busy time of 6 clocks and a total memory latency of 12 cycles.**  
> **How long will it take to complete a 64-element vector load with a stride of 1? With a stride of 32?**

---

### Given Parameters:
* Number of Memory Banks, $B = 8$
* Bank Busy Time, $t_{\text{busy}} = 6\text{ clock cycles}$
* Memory Latency to First Element, $t_{\text{latency}} = 12\text{ clock cycles}$
* Number of Vector Elements, $N = 64$

---

### Part 1: Stride = 1 (Unit Stride)

#### Bank Conflict Analysis:
* With a stride of 1, consecutive vector elements $0, 1, 2, 3, 4, 5, 6, 7, 8, \dots$ access memory banks in sequence:
  $$\text{Bank Index} = i \pmod 8 \implies 0, 1, 2, 3, 4, 5, 6, 7, 0, 1, \dots$$
* The same bank (e.g., Bank 0) is accessed every **8 clock cycles**.
* Since the bank cycle time is $t_{\text{busy}} = 6\text{ cycles}$, and $8 \ge 6$:
  $$\mathbf{\text{Number of Banks } (8) > \text{Bank Busy Time } (6)}$$
* **No bank conflicts occur!** Each bank completes its cycle and is idle before the next access arrives.
* Once the first element arrives after $t_{\text{latency}} = 12$ cycles, one element is delivered **every single clock cycle** for the remaining $63$ elements.

#### Total Execution Time:
$$\text{Time}_{\text{stride}=1} = t_{\text{latency}} + (N - 1) \times 1$$
$$\text{Time}_{\text{stride}=1} = 12 + (64 - 1) = 12 + 63 = \mathbf{75\text{ clock cycles}}$$

---

### Part 2: Stride = 32 (Non-Unit Stride)

#### Bank Conflict Analysis:
* With a stride of 32, each element $i$ accesses memory address:
  $$\text{Address}_i = \text{Base} + i \times 32$$
* Assuming addresses are word-interleaved across the 8 banks:
  $$\text{Bank Index} = (i \times 32) \pmod 8$$
* Since $32 \pmod 8 = 0$:
  $$(i \times 32) \pmod 8 = 0 \quad \text{for all } i = 0, 1, 2, \dots, 63$$
* **Severe Bank Conflict (Pathological Stride):** Every single access ($i = 0$ to $63$) targets **the exact same memory bank (Bank 0)**!
* As a result, memory accesses cannot be overlapped. Every subsequent access must stall until Bank 0 completes its busy time of $6$ clock cycles!

#### Total Execution Time:
* Element 0 completes after the initial latency: $t_{\text{latency}} = 12\text{ cycles}$.
* Each of the remaining $63$ elements must wait the full bank busy time of $6$ clock cycles:
$$\text{Time}_{\text{stride}=32} = t_{\text{latency}} + (N - 1) \times t_{\text{busy}}$$
$$\text{Time}_{\text{stride}=32} = 12 + (64 - 1) \times 6$$
$$\text{Time}_{\text{stride}=32} = 12 + (63 \times 6) = 12 + 378 = \mathbf{390\text{ clock cycles}}$$

---

### Comparison Summary:
| Stride | Memory Banks Used | Bank Conflict? | Cycles to Complete | Relative Slowdown |
| :---: | :---: | :---: | :---: | :---: |
| **Stride = 1** | All 8 banks interleaved | **No** (Delivers 1 element/cycle) | **75 clock cycles** | $1.0\times$ (Baseline) |
| **Stride = 32** | Only Bank 0 (Single bank) | **Yes** (Every access stalls 6 cycles) | **390 clock cycles** | **$5.2\times$ slower** |


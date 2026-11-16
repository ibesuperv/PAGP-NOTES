# Unit – III: SIMD Thread Scheduling, VMIPS Scalar Registers & Fermi Innovations
## Exam Question Bank Solutions (Part 5)

**Syllabus Mapping:**
* **Vector Architecture:** VMIPS basic structure and components, Justification of Scalar Registers, Concept of "Vectorized Code", VMIPS vector assembly translation.
* **Graphics Processing Units (GPUs) & SIMD/OpenCL:** Scheduling threads of SIMD instructions, OpenCL work-item execution model, NVIDIA Fermi GPU architectural innovations.

---

# SECTION A: SHORT QUESTIONS (Unit-3 Only)

---

### Question 1.12 [01 Mark]
> **In OpenCL, the kernel is launched with a two-dimensional `___________` containing all work-items.**

#### Answer:
**`NDRange` (or index space / global work-space)**.

#### Explanation:
In OpenCL (the open, vendor-neutral GPU programming standard analogous to CUDA), the computational domain is defined as an **$N$-Dimensional Range (`NDRange`)**, which can be 1D, 2D, or 3D. The `NDRange` spans the entire collection of parallel execution units called **work-items** (equivalent to CUDA threads), grouped into **work-groups** (equivalent to CUDA thread blocks).

---

### Question 1.13 [02 Marks]
> **The OpenCL kernel calls the API functions `___________` & `___________` to fetch the total number of work-items in the x- and y-dimension of all work-items.**

#### Answer:
**`get_global_size(0)`** and **`get_global_size(1)`**.

#### Explanation:
* **`get_global_size(0)`:** Returns the total number of global work-items in the $x$-dimension (dimension index 0).
* **`get_global_size(1)`:** Returns the total number of global work-items in the $y$-dimension (dimension index 1).
*(In CUDA, the exact equivalents are `gridDim.x * blockDim.x` and `gridDim.y * blockDim.y`).*

---

# SECTION B: LONG QUESTIONS (08 & 10 Marks)

---

## Question 5.a [08 Marks]
> **With a neat diagram, explain the process of scheduling threads of SIMD instructions.**

### SIMD / GPU Thread & Warp Scheduling Architecture:

```
                            GRID OF THREAD BLOCKS
                                      │
                                      ▼
         Dynamic Distribution to Streaming Multiprocessors (SMs)
    ┌─────────────────────────────────────────────────────────────┐
    │                 STREAMING MULTIPROCESSOR (SM)               │
    │                                                             │
    │  Block 0: [Warp 0] [Warp 1] [Warp 2] ... [Warp M]           │
    │  Block 1: [Warp 0] [Warp 1] [Warp 2] ... [Warp K]           │
    │                             │                               │
    │                             ▼                               │
    │                   WARP SCHEDULER LOGIC                      │
    │              (Monitors Instruction Readiness)               │
    │                             │                               │
    │        ┌────────────────────┴────────────────────┐          │
    │        ▼                                         ▼          │
    │   Warp ELIGIBLE?                            Warp STALLED?   │
    │   • Operands ready in registers             • Global memory │
    │   • Functional unit free                      access pending│
    │        │                                         │          │
    │        ▼ (1 cycle issue)                         ▼          │
    │   Issue SIMD Instruction                   Zero-Overhead    │
    │   to 32 Execution Lanes                    Context Switch   │
    │   ┌───┬───┬───┬───────┬───┐                (Deselect warp   │
    │   │L0 │L1 │L2 │  ...  │L31│                 without penalty)│
    │   └───┴───┴───┴───────┴───┘                                 │
    └─────────────────────────────────────────────────────────────┘
```

### Key Principles of Scheduling SIMD Instruction Threads:

1. **SIMD/SIMT Thread Grouping (Warps):**
   * The thousands of scalar threads declared by a programmer are grouped into fixed-size hardware SIMD units called **warps** (or *wavefronts*), each consisting of **32 consecutive parallel threads**.
   * A warp is the fundamental unit of scheduling on a Streaming Multiprocessor (SM).
2. **SIMT Lock-Step Execution:**
   * At any cycle, a warp executes a single instruction across all 32 threads concurrently.
   * Each thread within the warp operates on its own private registers and unique data inputs across 32 physical SIMD ALU lanes.
3. **Scoreboarding & Eligible Warp Selection:**
   * Each SM contains dedicated hardware **Warp Schedulers**.
   * The scheduler tracks the status of all resident warps using hardware scoreboarding.
   * A warp is marked **eligible** if:
     1. Its instruction operands are available in the register file.
     2. The required execution pipe (e.g., FP32, INT32, Load/Store) is free.
4. **Zero-Overhead Context Switching (Latency Hiding):**
   * High-latency operations (like global DRAM access taking 200–400 cycles) stall the currently executing warp.
   * Unlike CPUs, which flush pipelines or perform expensive OS context switches, the GPU warp scheduler simply **switches to another ready warp in 0 clock cycles**.
   * Because each thread's register state remains permanently resident in the massive physical register file, context switching incurs **zero cycle overhead**.
5. **Handling Divergent Threads:**
   * If threads in a warp take different execution paths at a conditional branch (`if-else`), the scheduler uses **hardware predication masks** to serialize the paths:
     - Cycle $T$: Executes the `then`-clause with `false` threads disabled.
     - Cycle $T + \Delta$: Executes the `else`-clause with `true` threads disabled.
     - Reconverges threads back to full 32-thread lock-step execution at the join point.

---

## Question 5.b [08 Marks]
> **Draw the basic structure of vector architecture for VMIPS and explain its components.**

### VMIPS Architectural Block Diagram:

```
                         Scalar Processor Core
                   (R0-R31, F0-F31, Scalar Registers)
                                   │
                                   ▼
    ══════════════════════════════════════════════════════════════════
                         VECTOR CONTROL & BUSES
    ══════════════════════════════════════════════════════════════════
            │                         ▲                    │
            │ Register Access         │ Chaining Buses     ▼
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
        ┌───────────────┬───────────────┬───────────────┐
        │ Memory Bank 0 │ Memory Bank 1 │ Memory Bank N │
        └───────────────┴───────────────┴───────────────┘
```

### Detailed Functional Components:
1. **Vector Register File:**
   * Consists of **8 vector registers ($V0$ through $V7$)**.
   * Each vector register holds **64 elements**, where each element is a **64-bit double-precision** floating-point value.
   * Multi-ported to allow parallel reads and writes across multiple functional units.
2. **Vector Functional Units:**
   * Independent, fully pipelined execution units for FP Addition/Subtraction, FP Multiplication, FP Division, and Integer/Bitwise operations.
   * Capable of initiating and retiring one element operation every single clock cycle.
3. **Vector Control Registers:**
   * **Vector Length Register (`VLR`):** Dynamically dictates how many elements ($1 \le \text{length} \le 64$) are processed by vector instructions when handling arbitrary loop trip counts.
   * **Vector Mask Register (`VMR`):** A 64-bit boolean register used for predicated element updates during conditional (`if-else`) execution.
4. **Vector Load / Store Unit:**
   * Pipelined memory unit that transfers entire vector registers to/from memory at a sustained bandwidth of 1 word per clock cycle after initial latency.
   * Supports unit stride, non-unit stride (`LVWS`/`SVWS`), and gather/scatter indexed addressing (`LVI`/`SVI`).
5. **Multi-Bank Interleaved Memory System:**
   * Spreads memory addresses across independent memory banks to sustain the massive memory bandwidth demanded by vector execution.

---

## Question 6.a [10 Marks]
> **Justify the need for scalar registers included as a primary component in VMIPS architecture.**  
> **Discussing the term 'code is vectorized', provide the VMIPS code considering the code segment given below.**  
> **Assume that the starting addresses of $m, s, t,$ and $u$ are in $Rm, Rs, Rt,$ and $Ru$, respectively, and $\text{MVL} = 64$.**  
> *(Target loop: $m[i] = s \times t[i] + u[i]$ for $i = 0$ to $63$)*

---

### Part 1: Justification of Scalar Registers in Vector Architectures [03 Marks]
Vector processors cannot operate in isolation without a tightly coupled scalar execution unit and scalar register file for three fundamental reasons:
1. **Passing Scalar Operands to Vector Instructions:**
   * Many linear algebra algorithms involve vector-scalar arithmetic (e.g., in DAXPY: $Y = a \times X + Y$, where $a$ is a scalar constant).
   * Storing the scalar in a scalar floating-point register ($F0$) allows instructions like `MULVS.D V2, V1, F0` to broadcast the scalar across all 64 elements without wasting an entire 64-element vector register.
2. **Address and Stride Calculations:**
   * Memory base addresses ($Rx, Ry, Rm$) and non-unit stride values (step size $k$) are computed and maintained in integer scalar registers ($R0 - R31$).
3. **Loop Control and Vector Length Setting:**
   * Setting the active vector length in `VLR` requires moving scalar values from integer registers to `VLR` (`mtc1 VLR, Rlen`).
   * Non-vectorizable scalar operations (pointer updates, loop counters, scalar reductions) must execute on the scalar core.

---

### Part 2: What Does "Code is Vectorized" Mean? [02 Marks]
* **Definition:** A loop is said to be **vectorized** when the compiler transforms a scalar loop that executes element-by-element across multiple sequential iterations into a sequence of **vector instructions** that operate concurrently across entire vectors of data elements (up to $\text{MVL} = 64$ elements per instruction).
* **Requirements for Vectorization:**
  1. The loop must have **loop-level parallelism** (no loop-carried true dependences between iterations).
  2. Memory access patterns must be affine/analyzable (unit stride, fixed stride, or indexed).
  3. The compiler replaces tens or hundreds of scalar instruction fetches and branches with a handful of vector instructions.

---

### Part 3: VMIPS Code Generation [05 Marks]

#### High-Level Loop:
```c
// Vector expression: m[i] = s * t[i] + u[i] for 64 elements
for (int i = 0; i < 64; i++) {
    m[i] = s * t[i] + u[i];
}
```
* **Given Register Mapping:**
  * Scalar $s$ is loaded into scalar FP register $F0$.
  * Base address of array $t$ is in $Rt$.
  * Base address of array $u$ is in $Ru$.
  * Base address of destination array $m$ is in $Rm$.
  * Vector Length Register (`VLR`) $= \text{MVL} = 64$.

#### VMIPS Assembly Code:
```assembly
# Complete VMIPS Vectorized Code for m[i] = s * t[i] + u[i]
LV      V1, Rt          # Load 64 double-precision elements of array t into V1
MULVS.D V2, V1, F0      # Vector-Scalar Multiply: V2[i] = t[i] * s
LV      V3, Ru          # Load 64 double-precision elements of array u into V3
ADDVV.D V4, V2, V3      # Vector-Vector Add: V4[i] = (s * t[i]) + u[i]
SV      V4, Rm          # Store result vector V4 into destination array m
```

#### Convoy Execution Breakdown:
* **Convoy 1:** `LV V1, Rt` and `MULVS.D V2, V1, F0` *(Chained together)*
* **Convoy 2:** `LV V3, Ru` and `ADDVV.D V4, V2, V3` *(Chained together)*
* **Convoy 3:** `SV V4, Rm`
* **Total Execution Time:** Exactly **$3\text{ chimes}$** ($3 \times 64 = 192$ cycles on a 1-lane processor, or $48$ cycles on a 4-lane processor).

---

## Question 6.b [06 Marks]
> **Discuss the major innovations introduced by Fermi to GPU architecture.**

### The NVIDIA Fermi (GF100, 2010) Microarchitecture
Fermi was the seminal architecture that transitioned GPUs from specialized 3D graphics chips into general-purpose, high-performance computing (HPC) processors.

```
                           FERMI KEY INNOVATIONS
 ┌────────────────────────────────────────────────────────────────────────┐
 │ 1. Unified 64-Bit Virtual Address Space (True C/C++ Pointers)          │
 │ 2. Fast IEEE 754-2008 Double-Precision Floating-Point (FMA)            │
 │ 3. Unified Two-Level Cache Hierarchy (Configurable L1 / Shared + L2)   │
 │ 4. Hardware ECC (Error-Correcting Code) for Datacenter Reliability     │
 │ 5. GigaThread Engine (Dual Concurrent Kernel Execution)                │
 └────────────────────────────────────────────────────────────────────────┘
```

1. **Unified 64-Bit Memory Addressing (True C++ Pointer Support):**
   * Prior GPUs used disjoint, segmented address spaces (separate spaces for registers, shared memory, and global memory).
   * Fermi unified these into a single, flat **64-bit virtual address space**, enabling standard C/C++ pointers, dynamic memory allocation (`malloc`), and C++ virtual functions.
2. **Fast IEEE 754-2008 Double-Precision Arithmetic:**
   * Provided hardware FP64 execution units capable of up to **$0.5\times$ FP32 performance** (a massive jump from $0.1\times$ in prior architectures).
   * Implemented hardware **Fused Multiply-Add (FMA)**, computing $A \times B + C$ with a single final rounding step, eliminating intermediate precision loss.
3. **First GPU Two-Level Cache Hierarchy:**
   * **Configurable L1 Cache / Shared Memory (64 KB per SM):** Programmers could configure on-chip SRAM via software as:
     * $48\text{ KB Shared Memory} / 16\text{ KB L1 Cache}$, OR
     * $16\text{ KB Shared Memory} / 48\text{ KB L1 Cache}$.
   * **Unified 768 KB Coherent L2 Cache:** Automatically cached all global memory loads, stores, and texture accesses across all SMs.
4. **Full ECC (Error-Correcting Code) Memory Protection:**
   * Provided hardware ECC protection for the register file, shared memory, L1/L2 caches, and off-chip GDDR5 DRAM.
   * Essential for mission-critical scientific computing and datacenter simulations where soft bit-flips caused by cosmic rays cannot be tolerated.
5. **Concurrent Kernel Execution (GigaThread Engine):**
   * Enabled different kernels from the same CUDA context to execute simultaneously across different SMs, preventing small kernels from underutilizing the GPU.


# Unit – III: Vector Architecture, GPU Thread Scheduling & GCD Test
## Exam Question Bank Solutions (Part 4)

**Syllabus Mapping:**
* **Vector Architecture:** VMIPS block diagram, DAXPY loop (MIPS vs. VMIPS), Vector Lanes (multiple functional units for $C = A + B$), Chimes, Vector Mask Register, Gather/Scatter.
* **Vector vs. GPU Comparison & SIMD Thread Scheduling:** Architectural differences, GPU Warp/Thread Scheduling.
* **Loop-Level Parallelism:** GCD Test for loop dependencies.

---

# SECTION A: SHORT QUESTIONS (1 Mark & 2 Marks)

---

### Question 1.3 [01 Mark]
> **`___________` is used to handle `if` statements in vector loops.**

#### Answer:
**Vector Mask Register (`VMR`)** (or **Masked Vector Execution / Vector Predication**).

#### Explanation:
In a vector processor, the Vector Mask Register (a 64-bit boolean register) conditionally controls the writeback of vector operations. When bit $i = 1$, the operation executes and stores the result into element $i$; when bit $i = 0$, writeback to element $i$ is suppressed, allowing vectorized execution of `if-then-else` blocks without scalar branch stalls.

---

### Question 1.4 [01 Mark]
> **Mention the use of Gather-Scatter with respect to vector architecture.**

#### Answer:
**Gather-Scatter** enables vector processors to operate efficiently on **sparse matrices** and **non-contiguous (indirect/pointer-based) memory structures**:
* **Gather (`LVI`):** Uses an index vector of memory offsets to gather elements from disparate, non-contiguous memory locations into a contiguous vector register:
  $$\text{V}[i] = \text{Memory}[\text{Base} + \text{IndexV}[i]]$$
* **Scatter (`SVI`):** Writes elements from a contiguous vector register out to non-contiguous memory locations dictated by an index vector:
  $$\text{Memory}[\text{Base} + \text{IndexV}[i]] = \text{V}[i]$$

---

### Question 1.5 [01 Mark]
> **Define Chime.**

#### Answer:
A **chime** is the basic unit of time used to measure vector execution. It represents the time required to execute an entire **vector convoy** (a set of vector instructions running concurrently without structural hazards) across the full length of a vector register ($\text{MVL} = 64$ elements). 
* For a vector processor with a single lane, $1\text{ chime} = \text{MVL} = 64\text{ clock cycles}$.
* For a processor with $L$ lanes, $1\text{ chime} = \frac{\text{MVL}}{L}\text{ clock cycles}$.

---

### Question 1.6 [02 Marks]
> **Differentiate between Vector Architecture and GPU.**

| Feature | Vector Architecture (e.g., VMIPS, Cray) | GPU Architecture (e.g., NVIDIA CUDA / SIMT) |
| :--- | :--- | :--- |
| **Instruction Stream** | A **single vector instruction** explicitly encodes operations across an entire array of data elements (e.g., `ADDVV.D`). | Instructions are written for a **single scalar thread**; hardware executes 32 threads together in a **warp** (SIMT model). |
| **Hardware Lanes & Registers** | Few wide registers ($8\text{ to }32$ vector registers) with 64 elements deep; fixed lanes execute striped data. | Massive register file (tens of thousands of 32-bit registers) dynamically partitioned among thousands of lightweight threads. |
| **Control Flow & Masking** | Uses an architectural **Vector Mask Register (`VMR`)** explicitly manipulated by vector mask instructions. | Uses **Hardware Branch Predication / Mask Stack** that automatically handles warp divergence and reconvergence. |
| **Latency Hiding Technique** | Relies on deep **pipelining, chaining**, and multi-bank interleaved memory to stream elements without stalls. | Relies on **massive Thread-Level Parallelism (TLP)** and **Zero-Overhead Warp Scheduling** (switching active warps when one stalls on DRAM). |

---

### Question 1.7 [02 Marks]
> **Use the GCD test to determine whether dependency exists in the loop:**
> ```c
> for (i = 0; i < 100; i++) {
>     X[2 * i + 3] = X[2 * i] * 5.0;
> }
> ```

#### Step-by-Step Solution:
1. **Identify Coefficients:**
   * Store index: `X[a * i + b]` $\implies a = 2, \quad b = 3$
   * Load index: `X[c * j + d]` $\implies c = 2, \quad d = 0$
2. **Apply GCD Theorem:**
   A loop-carried dependence exists between iterations $i$ and $j$ if and only if $\gcd(a, c)$ divides $(d - b)$.
3. **Compute:**
   * $\gcd(a, c) = \gcd(2, 2) = \mathbf{2}$
   * $d - b = 0 - 3 = \mathbf{-3}$
4. **Check Divisibility:**
   $$\frac{-3}{2} = -1.5 \quad (\text{Not an integer; 2 does not divide -3})$$

#### Conclusion:
Since $\gcd(2, 2) = 2$ does **not** divide $-3$, there are no integer solutions. **No loop-carried dependence exists**, and the loop **can be safely vectorized**.

---

# SECTION B: LONG QUESTIONS (08 Marks Each)

---

## Question 3.a [08 Marks]
> **Illustrate with a neat diagram the basic structure of a vector architecture.**

### VMIPS Architecture Block Diagram:

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

### Detailed Description of Primary Components:
1. **Vector Register File:**
   * Consists of **8 vector registers ($V0$ through $V7$)**.
   * Each register contains **64 elements**, where each element is a **64-bit double-precision** word.
   * Multi-ported to allow simultaneous reads and writes by arithmetic and memory units.
2. **Vector Functional Units:**
   * Independent, fully pipelined functional units:
     * **Vector FP Add / Subtract**
     * **Vector FP Multiply**
     * **Vector FP Divide**
     * **Vector Integer / Logical**
   * Pipelining allows one element computation to start and complete every clock cycle.
3. **Vector Control Registers:**
   * **Vector Length Register (`VLR`):** Controls the active vector length ($1 \le \text{length} \le 64$) when processing loops of arbitrary bounds.
   * **Vector Mask Register (`VMR`):** A 64-bit register for conditionally masking operations on individual elements during `if-else` execution.
4. **Vector Load / Store Unit:**
   * Highly pipelined memory interface capable of moving elements between memory and vector registers at a sustained rate of 1 word per clock cycle after initial memory latency.
   * Supports **Unit Stride**, **Non-Unit Stride** (`LVWS`/`SVWS`), and **Indexed Gather/Scatter** (`LVI`/`SVI`).
5. **Multi-Bank Interleaved Memory System:**
   * Memory is divided into multiple independent banks to allow overlapping accesses, preventing bank conflicts and delivering high bandwidth.

---

## Question 3.b [08 Marks]
> **Show the code for MIPS and VMIPS for DAXPY loop ($Y = a \times X + Y$). Assume that the starting addresses of $X$ and $Y$ are in $Rx$ and $Ry$ respectively.**

### 1. Scalar MIPS Assembly Code (Processes 64 Elements Iteratively)
Assume scalar $a$ is in register $F0$, $Rx$ and $Ry$ contain array base addresses, and $R1$ contains `Rx + 512` (end pointer for 64 double-precision elements):

```assembly
# Scalar MIPS DAXPY Loop
Loop:
    L.D    F2, 0(Rx)        # Load X[i] from memory into F2
    MUL.D  F2, F2, F0       # Compute a * X[i]
    L.D    F4, 0(Ry)        # Load Y[i] from memory into F4
    ADD.D  F4, F4, F2       # Compute a * X[i] + Y[i]
    S.D    F4, 0(Ry)        # Store result into Y[i]
    DADDIU Rx, Rx, #8       # Increment X pointer by 8 bytes (64-bit word)
    DADDIU Ry, Ry, #8       # Increment Y pointer by 8 bytes
    DSUBU  R2, R1, Rx       # Check if end address reached: R2 = R1 - Rx
    BNEZ   R2, Loop         # Branch to Loop if R2 != 0
```
* **Dynamic Instruction Count:** 9 instructions per loop $\times 64\text{ iterations} = \mathbf{576\text{ scalar instructions}}$.
* **Overhead:** Instruction fetch/decode bandwidth is heavily consumed by pointer increments and conditional branches.

---

### 2. Vector VMIPS Assembly Code (Processes 64 Elements at Once)
Assume scalar constant $a$ is in scalar register $F0$, and the Vector Length Register (`VLR`) is set to 64:

```assembly
# Vector VMIPS DAXPY Code
LV      V1, Rx          # Load vector X (64 double-precision words) into V1
MULVS.D V2, V1, F0      # Vector-Scalar Multiply: V2[i] = V1[i] * F0
LV      V3, Ry          # Load vector Y (64 double-precision words) into V3
ADDVV.D V4, V2, V3      # Vector-Vector Add: V4[i] = V2[i] + V3[i]
SV      V4, Ry          # Store vector V4 back to memory at address Ry
```
* **Dynamic Instruction Count:** Exactly **$5\text{ vector instructions}$**!
* **Comparison:**
  * Instruction count is reduced by **$115\times$** ($576 \to 5$).
  * Zero loop branches, zero address increment overhead, zero branch prediction hazards.

---

## Question 4.a [08 Marks]
> **Using multiple functional units, how do you improve the performance of a single vector add instruction $C = A + B$?**

### Answer:

A single vector instruction operating on a 64-element vector does not have to execute element-by-element across 64 sequential clock cycles on a single adder. Vector hardware achieves dramatic speedups by dividing execution across **Multiple Parallel Vector Lanes**.

```
Single 64-Element Vector Add: C = A + B executed across 4 Vector Lanes

Cycle:           1               2              ...           16
         ┌───────────────┬───────────────┬────── ... ─┬───────────────┐
Lane 0:  │ C[0]=A[0]+B[0]│ C[4]=A[4]+B[4]│            │C[60]=A[60]+B[60]
Lane 1:  │ C[1]=A[1]+B[1]│ C[5]=A[5]+B[5]│            │C[61]=A[61]+B[61]
Lane 2:  │ C[2]=A[2]+B[2]│ C[6]=A[6]+B[6]│            │C[62]=A[62]+B[62]
Lane 3:  │ C[3]=A[3]+B[3]│ C[7]=A[7]+B[7]│            │C[63]=A[63]+B[63]
         └───────────────┴───────────────┴────── ... ─┴───────────────┘
```

### 1. Architectural Organization of Vector Lanes
* Instead of a single 64-bit adder, the processor replicates the execution pipeline into $L$ identical **lanes** (e.g., $L = 4$).
* Each lane contains:
  1. A $\frac{1}{L}$ slice of the vector register file (16 elements per register for $\text{MVL} = 64$ across 4 lanes).
  2. A local pipelined vector arithmetic unit (FP adder).
* **Interleaved Striping:** Elements are striped across lanes cyclically:
  * **Lane 0:** Elements $0, 4, 8, \dots, 60$
  * **Lane 1:** Elements $1, 5, 9, \dots, 61$
  * **Lane 2:** Elements $2, 6, 10, \dots, 62$
  * **Lane 3:** Elements $3, 7, 11, \dots, 63$

### 2. Performance Analysis:
* **Single Functional Unit ($L = 1$):**
  $$\text{Execution Time} = \text{MVL} = 64\text{ clock cycles}$$
* **Multiple Functional Units ($L = 4$ Lanes):**
  $$\text{Execution Time} = \frac{\text{MVL}}{L} = \frac{64}{4} = \mathbf{16\text{ clock cycles}}$$
* **Four-Fold ($4\times$) Speedup:**
  * 4 elements are produced on every clock cycle.
  * Achieved without altering the binary instruction or ISA format—the hardware automatically distributes elements across lanes.

---

## Question 4.b [08 Marks]
> **Briefly describe about scheduling of threads of SIMD instruction.**

### Answer:

In modern SIMD/SIMT architectures (such as NVIDIA GPUs), thousands of threads are scheduled and executed across parallel compute cores using a **hierarchical, dynamic warp scheduling model**.

```
                           GRID OF THREAD BLOCKS
                                     │
                                     ▼
         Dynamic Assignment to Streaming Multiprocessors (SMs)
         ┌───────────────────────────────────────────────────┐
         │             STREAMING MULTIPROCESSOR              │
         │                                                   │
         │   Thread Block 0:  [Warp 0] [Warp 1] ... [Warp N] │
         │   Thread Block 1:  [Warp 0] [Warp 1] ... [Warp M] │
         │                           │                       │
         │                           ▼                       │
         │                 WARP SCHEDULER(S)                 │
         │         (Monitors instruction readiness)          │
         │                           │                       │
         │          ┌────────────────┴────────────────┐      │
         │          ▼                                 ▼      │
         │     Warp Ready?                       Warp Stalled?│
         │     (Operands in registers)           (Memory wait)│
         │          │                                 │      │
         │          ▼                                 ▼      │
         │  Issue SIMD Instruction            Deselect / Wait│
         │  to 32 CUDA Cores in 1 cycle       in 0-cycle switch
         └───────────────────────────────────────────────────┘
```

### Key Principles of GPU SIMD Thread Scheduling:

1. **Warp Partitioning (The Basic SIMD Unit):**
   * Individual scalar threads of a thread block are grouped into **warps of 32 consecutive threads** (`threadIdx` 0 to 31, 32 to 63, etc.).
   * A warp is the smallest scheduling entity recognized by the hardware.
2. **SIMT (Single Instruction, Multiple Threads) Execution:**
   * All 32 threads in a warp execute the exact same instruction simultaneously on different data across 32 physical SIMD lanes.
3. **Scoreboarding & Ready Warp Selection:**
   * Each SM contains one or more hardware **Warp Schedulers**.
   * The scheduler tracks execution readiness of all active warps using scoreboarding (checking if instruction operands are ready in registers).
   * At each clock cycle, the warp scheduler selects an eligible, active warp and dispatches its instruction to the execution units.
4. **Zero-Overhead Context Switching (Thread-Level Latency Hiding):**
   * Unlike CPUs that rely on complex out-of-order execution, branch predictors, and large caches to hide latency, GPUs use **Thread-Level Parallelism (TLP)**.
   * When an active warp initiates a long-latency memory access (200–400 cycles to global DRAM), the Warp Scheduler simply **switches to another ready warp in 0 clock cycles**.
   * Because all thread registers remain resident in the massive on-chip register file, no save/restore penalty exists.
5. **Handling Warp Divergence:**
   * When threads in a warp branch differently at an `if-else` block, the scheduler serializes the paths using hardware predication masks:
     * First passes through the `then` path with false-threads disabled.
     * Then passes through the `else` path with true-threads disabled.
     * Finally reconverges the warp.


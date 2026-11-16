# Unit – III: SIMD Multimedia Extensions, CPU vs. GPU, & Vector Architecture
## Exam Question Bank Solutions (Part 9)

**Syllabus Mapping:**
* **Vector Architecture:** VMIPS structure and functionalities, Convoys, Chimes and Cycles/FLOP calculation, Gather/Scatter, Matrix multiplication terms calculation.
* **SIMD Multimedia Extensions:** Motivation, packed data types, x86 SIMD evolution (MMX, SSE, AVX), programming challenges.
* **Graphics Processing Units (GPUs):** CPU vs. GPU architectural differences, Exploiting Data-Level Parallelism (DLP), Warps & SIMT execution.
* **Loop-Level Parallelism:** Loop-carried dependence definition and concrete code example.

---

# SECTION A: SHORT QUESTIONS (Unit-3 Only)

---

### Question 1.1 [02 Marks]
> **Define a warp.**

#### Answer:
In NVIDIA's CUDA / SIMT (Single Instruction, Multiple Threads) architecture, a **warp** is the basic unit of instruction dispatch and execution on a Streaming Multiprocessor (SM).
* **Size:** Consists of exactly **32 consecutive parallel threads** (threads with `threadIdx.x` from $0$ to $31$, $32$ to $63$, etc.).
* **SIMT Execution:** All 32 threads in a warp execute the same instruction in lock-step across 32 physical execution lanes.
* **Warp Divergence:** If threads branch in different directions during an `if-else` statement, the execution paths are serialized, reducing SIMD efficiency.

---

### Question 1.3 [02 Marks]
> **Define loop-carried dependence with example.**

#### Answer:
#### Definition:
A **loop-carried dependence** occurs when a memory location or variable produced/written in one iteration (e.g., iteration $i$) of a loop is accessed/read or overwritten in a *subsequent* iteration (e.g., iteration $i+1, i+2, \dots$) of the same loop.

#### Example:
```c
for (int i = 1; i < N; i++) {
    A[i] = A[i - 1] + B[i]; // A[i] depends on A[i-1] computed in the previous iteration
}
```
* In iteration $i = 1$, `A[1]` is computed using `A[0]`.
* In iteration $i = 2$, computing `A[2]` requires `A[1]`, which was just produced by iteration 1.
* **Impact:** Iteration 2 cannot start until iteration 1 completes; the iterations must execute sequentially and cannot be parallelized across vector lanes or CUDA threads.

---

### Question 1.5 [02 Marks]
> **Consider the multiplication of two $5 \times 5$ matrices using a vector processor. How many product terms are there in each inner product? How many multiply-add operations are needed to calculate the product matrix?**

#### Step-by-Step Solution:
Let the matrices be $A$ ($5 \times 5$) and $B$ ($5 \times 5$). The resulting matrix $C = A \times B$ has dimension $5 \times 5$ (total of $5 \times 5 = \mathbf{25\text{ elements}}$).

1. **Product terms in each inner product:**
   * Each element $C[i][j]$ is the dot product (inner product) of row $i$ of $A$ (length 5) and column $j$ of $B$ (length 5):
     $$C[i][j] = \sum_{k=0}^{4} A[i][k] \times B[k][j] = A[i][0]B[0][j] + A[i][1]B[1][j] + \dots + A[i][4]B[4][j]$$
   * **Number of product terms in each inner product = 5 product terms**.

2. **Multiply-Add operations to calculate the product matrix:**
   * For **1 element** $C[i][j]$:
     * Multiplications $= 5$
     * Additions $= 4$
     * Multiply-Accumulate / Multiply-Add operations $= \mathbf{5\text{ operations}}$
   * For the **entire product matrix** ($25$ elements):
     $$\text{Total Multiplications} = 25 \times 5 = \mathbf{125\text{ multiplications}}$$
     $$\text{Total Additions} = 25 \times 4 = \mathbf{100\text{ additions}}$$
     $$\text{Total Multiply-Add (MAC) Operations} = 25 \times 5 = \mathbf{125\text{ operations}}$$

---

### Question 1.7 [02 Marks]
> **The VMIPS instruction for gather and scatter are `_______` and `_______`.**

#### Answer:
* Gather: **`LVI` (Load Vector Indexed)**
* Scatter: **`SVI` (Store Vector Indexed)**

---

# SECTION B: LONG QUESTIONS (08 Marks Each)

---

## Question 5.a [08 Marks]
> **Illustrate the basic structure and functionalities of vector architecture, VMIPS.**

### VMIPS Architectural Block Diagram:

```
                         Scalar CPU Core (R0-R31, F0-F31)
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

### Primary Components and Functionalities:
1. **Vector Register File:**
   * Contains **8 vector registers ($V0$ through $V7$)**.
   * Each register contains **64 elements**, where each element is a **64-bit double-precision word**.
   * Multi-ported to allow parallel reads and writes by arithmetic and memory units.
2. **Vector Functional Units:**
   * Fully pipelined execution units for FP Add/Sub, FP Multiply, FP Divide, and Integer/Logical operations.
   * Capable of initiating and retiring one element operation every clock cycle.
3. **Vector Control Registers:**
   * **`VLR` (Vector Length Register):** Controls the active vector length ($1 \le \text{length} \le 64$) for loops with non-multiple-of-64 bounds.
   * **`VMR` (Vector Mask Register):** A 64-bit register for conditionally masking operations on individual elements during `if-else` execution.
4. **Vector Load / Store Unit:**
   * Pipelined memory unit transferring data at 1 word per clock cycle after initial latency.
   * Supports unit stride, non-unit stride (`LVWS`/`SVWS`), and gather/scatter (`LVI`/`SVI`).
5. **Multi-Bank Interleaved Memory System:**
   * Interleaves memory across independent banks to eliminate bank conflicts and sustain full memory throughput.

---

## Question 5.b [08 Marks]
> **Discuss the architectural difference of CPU and GPU. Explain how GPU exploits data-level parallelism.**

---

### 1. Architectural Comparison: CPU vs. GPU

```
          CPU (Latency-Oriented)                    GPU (Throughput-Oriented)
   ┌───────────────────────────────────┐    ┌───────────────────────────────────┐
   │ ┌───────┐ ┌───────┐ ┌───────────┐ │    │ ┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐ │
   │ │ ALU 0 │ │ ALU 1 │ │Control/OOO│ │    │ │ALU││ALU││ALU││ALU││ALU││ALU││ALU│ │
   │ └───────┘ └───────┘ └───────────┘ │    │ └───┴───┴───┴───┴───┴───┴───┴───┘ │
   │ ┌───────────────────────────────┐ │    │ Thousands of Tiny SIMD ALU Cores  │
   │ │ Large L1/L2/L3 Caches (SRAM)  │ │    │ ┌───────────────────────────────┐ │
   │ └───────────────────────────────┘ │    │ │ Small Cache / Shared Memory   │ │
   │ Large Silicon devoted to Cache    │    │ └───────────────────────────────┘ │
   │ & Out-of-Order Control Logic      │    │ Majority of Silicon = Raw ALUs    │
   └───────────────────────────────────┘    └───────────────────────────────────┘
```

| Architectural Feature | Central Processing Unit (CPU) | Graphics Processing Unit (GPU) |
| :--- | :--- | :--- |
| **Design Optimization** | **Low Latency** for single-threaded sequential execution. | **High Throughput** for massively parallel data workloads. |
| **Silicon Area Allocation** | Majority of chip area devoted to **large multi-level caches** and complex **control logic** (branch predictors, reorder buffers). | Majority of chip area devoted to **raw arithmetic execution units (ALUs / CUDA Cores)**. |
| **Execution Model** | Out-of-order superscalar execution across a few powerful cores ($4-32$ cores). | Single Instruction, Multiple Threads (**SIMT**) across thousands of lightweight cores ($1000-16000+$ cores). |
| **Latency Hiding Technique** | Large hardware caches, branch prediction, and dynamic speculative out-of-order scheduling. | **Massive Thread-Level Parallelism (TLP)** and **Zero-Overhead Warp Scheduling** (switching active warps when one stalls on DRAM). |
| **Memory Bandwidth** | Moderate ($50 - 150\text{ GB/s}$ via DDR4/DDR5). | Ultra-high ($1000 - 3000+\text{ GB/s}$ via GDDR6/HBM3). |

---

### 2. How GPUs Exploit Data-Level Parallelism (DLP)
1. **SIMT (Single Instruction, Multiple Threads):**
   * The programmer writes code for a single scalar thread. The GPU hardware automatically groups threads into **warps of 32 threads**.
   * A single instruction is broadcast across all 32 threads in the warp, operating on 32 different data elements simultaneously across 32 physical ALUs.
2. **Hierarchical Problem Decomposition:**
   * A large parallel problem is mapped to a **Grid of Thread Blocks**.
   * Blocks are distributed across multiple Streaming Multiprocessors (SMs), exploiting both task-level and data-level parallelism concurrently.
3. **Memory Coalescing:**
   * When threads in a warp access consecutive global memory addresses, the hardware combines the 32 individual memory requests into a single memory transaction (e.g., 128 bytes), fully saturating wide memory buses.

---

## Question 6.a [08 Marks]
> **Briefly describe SIMD instruction set extensions for multimedia.**

### 1. Motivation for SIMD Multimedia Extensions
* In the mid-1990s, microprocessors faced growing workloads in multimedia processing (graphics rendering, digital audio, image processing, and video codecs like MPEG).
* **Data Characteristic:** Multimedia applications operate on vectors of narrow data types—typically **8-bit pixel values (RGB/grayscale)** or **16-bit audio/sensor samples**.
* A standard 64-bit ALU was severely underutilized performing single 8-bit or 16-bit scalar operations.
* **The Solution:** Partition standard 64-bit or 128-bit adder and multiplier registers into multiple narrow data slices that operate simultaneously in parallel (**Subword Parallelism**).

---

### 2. Evolution of x86 Multimedia SIMD Extensions:

```
       MMX (1996)                  SSE (1999 - 2006)                 AVX (2011 - Present)
 ┌──────────────────────┐     ┌───────────────────────────┐     ┌───────────────────────────┐
 │ • 64-bit registers   │     │ • Dedicated 128-bit       │     │ • Dedicated 256/512-bit   │
 │   (aliased on FP)    │───► │   XMM registers (XMM0-15) │───► │   YMM/ZMM registers       │
 │ • 8x8-bit, 4x16-bit, │     │ • Single & Double Prec.   │     │ • Non-destructive 3-op    │
 │   2x32-bit integers  │     │   IEEE Floating Point     │     │   syntax: VADDPS ymm1,    │
 │ • NO Floating Point  │     │ • SSE, SSE2, SSE3, SSE4   │     │   ymm2, ymm3              │
 └──────────────────────┘     └───────────────────────────┘     └───────────────────────────┘
```

1. **Intel MMX (1996):**
   * Added 57 new multimedia instructions operating on 64-bit integer vectors.
   * Packed types: $8 \times 8$-bit bytes, $4 \times 16$-bit words, or $2 \times 32$-bit integers.
   * *Limitation:* Aliased onto the existing 80-bit x87 floating-point registers, preventing simultaneous FP and MMX execution.
2. **Streaming SIMD Extensions (SSE, SSE2, SSE3, SSE4) (1999–2006):**
   * Added **16 dedicated 128-bit vector registers (`XMM0`–`XMM15`)**, separating SIMD from legacy x87 FP state.
   * Packed types:
     * $4 \times 32$-bit single-precision floating-point numbers.
     * $2 \times 64$-bit double-precision floating-point numbers (SSE2).
     * Packed 8-bit, 16-bit, 32-bit, and 64-bit integers.
3. **Advanced Vector Extensions (AVX, AVX2, AVX-512) (2011–Present):**
   * Widened registers to **256 bits (`YMM0`–`YMM15`)** and **512 bits (`ZMM0`–`ZMM31`)**.
   * Packed types: $8 \times 32$-bit floats or $4 \times 64$-bit doubles per instruction.
   * Introduced non-destructive 3-operand instruction syntax (`VADDPS dst, src1, src2`), preserving source register values.
   * Introduced hardware **Fused Multiply-Add (FMA3)**.

---

### 3. Key Differences: SIMD Multimedia Extensions vs. Vector Architectures

| Feature | SIMD Multimedia Extensions (MMX/SSE/AVX) | Vector Architecture (VMIPS / Cray) |
| :--- | :--- | :--- |
| **Vector Length** | **Fixed** at hardware design time (e.g., exactly 128, 256, or 512 bits). | **Variable** via Vector Length Register (`VLR`), up to $\text{MVL} = 64$ words. |
| **Addressing Modes** | Primarily unit-stride memory accesses; requires explicit alignment to 16/32-byte boundaries. | Supports unit-stride, non-unit stride (`LVWS`), and indexed gather/scatter (`LVI`/`SVI`). |
| **Conditional Code** | Handled via bitwise masks and shuffle/blend instructions. | Architectural **Vector Mask Register (`VMR`)** provides native predication. |

---

## Question 6.b [08 Marks]
> **Show how the following code sequence lays out in convoys, assuming a single copy of each vector functional unit:**
> ```assembly
> LV      V1, Rx         ; load vector X
> MULVS.D V2, V1, F0     ; vector-scalar multiply
> LV      V3, Ry         ; load vector Y
> ADDVV.D V4, V2, V3     ; add two vectors
> SV      V4, Ry         ; store the sum
> ```
> **How many chimes will this vector sequence take? How many cycles per FLOP are needed, ignoring vector instruction issue overhead?**

---

### 1. Convoy Identification Rules
* A **convoy** is a group of vector instructions that can begin execution concurrently without structural conflicts.
* **Constraints for grouping into a convoy:**
  1. Instructions within the same convoy must not compete for the same functional unit (structural hazard: only 1 vector load/store unit exists in standard VMIPS).
  2. With vector register **chaining**, instructions with RAW data dependences can execute in the same convoy.
  3. A second memory instruction (`LV` or `SV`) cannot be in the same convoy as an earlier memory instruction because there is only **one vector memory port**.

---

### 2. Convoy Breakdown:

* **Convoy 1:**
  ```assembly
  LV      V1, Rx          # Uses Vector Load/Store unit
  MULVS.D V2, V1, F0      # Uses FP Multiplier (Chained to LV!)
  ```
  *(Can execute together because they use different functional units and are chained).*

* **Convoy 2:**
  ```assembly
  LV      V3, Ry          # Uses Vector Load/Store unit (new memory load)
  ADDVV.D V4, V2, V3      # Uses FP Adder (Chained to LV and MULVS.D)
  ```
  *(Cannot be placed in Convoy 1 because Convoy 1 already uses the single load/store unit for `LV V1`).*

* **Convoy 3:**
  ```assembly
  SV      V4, Ry          # Uses Vector Load/Store unit (memory store)
  ```
  *(Cannot be in Convoy 2 because Convoy 2 already uses the single load/store unit for `LV V3`).*

---

### 3. Total Chimes and Cycles Per FLOP Calculation:
* **Number of Chimes:**
  Each convoy requires **1 chime** (time to process the vector register, $\text{MVL} = 64$ clock cycles).
  $$\text{Total Chimes} = \mathbf{3\text{ chimes}}$$

* **Total Clock Cycles:**
  $$\text{Total Cycles} = 3\text{ chimes} \times 64 = \mathbf{192\text{ clock cycles}}$$

* **Floating-Point Operations (FLOPs):**
  * `MULVS.D` performs 64 multiplications.
  * `ADDVV.D` performs 64 additions.
  $$\text{Total FLOPs} = 64 + 64 = \mathbf{128\text{ FLOPs}}$$

* **Cycles Per FLOP:**
  $$\text{Cycles Per FLOP} = \frac{\text{Total Cycles}}{\text{Total FLOPs}} = \frac{192\text{ cycles}}{128\text{ FLOPs}} = \frac{3}{2} = \mathbf{1.5\text{ clock cycles per FLOP}}$$


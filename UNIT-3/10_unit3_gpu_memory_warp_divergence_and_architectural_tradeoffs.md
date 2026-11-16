# Unit – III: GPU Memory Hierarchy, Warp Divergence & Architectural Trade-offs
## Exam Question Bank Solutions (Part 10)

**Syllabus Mapping:**
* **Data-Level Parallelism:** Performance trade-offs in Vector, SIMD, and GPU architectures; Advantages of Vector Architectures; Loop unrolling for loop-level parallelism.
* **CUDA Programming Model:** Differentiate between CUDA Thread, Block, and Grid.
* **CUDA Execution Model:** Warp Divergence, Impact on Execution Efficiency, Methods to Reduce Divergence.
* **CUDA Memory Hierarchy:** Detailed usage and code examples for Shared Memory, Constant Memory, Texture Memory, and Surface Memory.

---

# SECTION A: SHORT QUESTIONS (Unit-3 Only)

---

### Question 1 [02 Marks]
> **State two advantages of vector architecture.**

#### Answer:
1. **Dramatic Reduction in Dynamic Instruction Bandwidth:** A single vector instruction replaces an entire scalar loop (e.g., in DAXPY, 5 vector instructions replace 576 scalar instructions), eliminating instruction-fetch bottlenecks, decode stalls, and branch mispredictions.
2. **Deterministic, Conflict-Free Memory Pipelining:** Because an entire vector operation specifies a regular memory access pattern (unit stride, known stride), multi-bank interleaved memory can pipeline accesses to deliver **one data word per clock cycle per processor**, hiding memory latency without requiring complex dynamic out-of-order execution logic.
*(Additional advantage: Significantly lower energy consumption per operation compared to superscalar CPUs).*

---

### Question 2 [02 Marks]
> **Differentiate between CUDA thread, block, and grid.**

```
GRID (Entire Problem Space launched by Kernel)
 ├── Block (0,0)  ──► Assigned to SM
 │    ├── Thread (0,0)
 │    ├── Thread (1,0) ...
 ├── Block (1,0)  ──► Assigned to SM
 └── Block (N,M)
```

| Entity | Hardware Mapping | Scope of Synchronization | Scope of Memory |
| :--- | :--- | :--- | :--- |
| **CUDA Thread** | Executes on a single physical **CUDA Core / ALU lane**. | Cannot synchronize independently across arbitrary threads. | Has private **Registers** and private **Local Memory**. |
| **CUDA Block** | Assigned to and executes exclusively on a **Streaming Multiprocessor (SM)**. | All threads within the same block can synchronize via **`__syncthreads()`**. | Threads in the block share high-speed on-chip **Shared Memory**. |
| **CUDA Grid** | Represents the entire kernel execution across the **whole GPU Device**. | No barrier synchronization across blocks during kernel execution. | All threads across all blocks in the grid access global **Device DRAM**. |

---

# SECTION B: LONG QUESTIONS (08 & 10 Marks Each)

---

## Question 3 [08 Marks]
> **Analyze the performance trade-offs in vector, SIMD, and GPU architectures.**

All three architectures exploit **Data-Level Parallelism (DLP)**, but they make fundamentally different microarchitectural trade-offs:

```
                            DATA-LEVEL PARALLELISM (DLP)
                                         │
        ┌────────────────────────────────┼────────────────────────────────┐
        ▼                                ▼                                ▼
  Vector Architecture           SIMD Extensions (Multimedia)       GPU Architecture (SIMT)
  (Cray, VMIPS, SX-Aurora)      (Intel AVX-512, ARM Neon)          (NVIDIA Hopper/Blackwell)
  • Explicit wide vectors       • Subword parallelism              • Massive multithreading
  • Variable length (VLR)       • Fixed narrow registers           • Thousands of scalar threads
  • Gather/Scatter hardware     • Stride-1 aligned data            • Zero-overhead warp sched.
```

### Comprehensive Architectural Comparison Table:

| Architectural Feature | Vector Architecture | SIMD Multimedia Extensions | GPU Architecture (SIMT) |
| :--- | :--- | :--- | :--- |
| **Basic Abstraction** | Operates on explicit vector registers of length $\text{MVL} = 64$. | Operates on fixed-width registers (128, 256, 512 bits). | Programmed as scalar threads; grouped in hardware into **warps of 32 threads**. |
| **Handling Arbitrary Loop Bounds** | Native hardware support via **Vector Length Register (`VLR`)** and Strip Mining. | Requires software scalar cleanup / remainder loops. | Built-in grid dimensioning with boundary guards (`if (i < n)`). |
| **Memory Access Flexibility** | Native unit stride, non-unit stride (`LVWS`), and indexed **gather/scatter (`LVI`/`SVI`)**. | Optimized for unit-stride aligned accesses; unaligned accesses cause major penalties. | Hardware **Memory Coalescing** merges adjacent thread accesses into wide 128-byte transactions. |
| **Control Flow & Branching** | Handled via architectural **Vector Mask Register (`VMR`)** (predication). | Bitwise vector masks, shuffles, and blend instructions. | **Hardware Predication & Mask Stack** (handles warp divergence and reconvergence automatically). |
| **Latency Hiding Mechanism** | Deep arithmetic pipelining, chaining, and multi-bank interleaved memory. | High-frequency out-of-order superscalar CPU core and large caches. | **Massive Thread-Level Parallelism (TLP)** and **Zero-Overhead Warp Scheduling** (masks 200–400 cycle DRAM latency). |
| **Hardware Complexity & Silicon Area** | Moderate control overhead, wide vector register files, massive memory bandwidth. | Minimal extra silicon added to existing superscalar CPU cores. | Dedicated throughput accelerator with thousands of execution lanes; relies on host CPU. |

---

## Question 4 [10 Marks]
> **Explain shared, constant, texture, and surface memory usage with examples.**

In NVIDIA CUDA, specialized memory spaces are tailored for specific algorithmic access patterns to bypass global DRAM bottlenecks:

```
┌────────────────────────────────────────────────────────────────────────┐
│                          CUDA DEVICE MEMORY                            │
├────────────────────────────────────────────────────────────────────────┤
│ 1. SHARED MEMORY: On-chip SRAM (~few cycles), per Thread Block         │
│ 2. CONSTANT MEMORY: 64 KB off-chip DRAM with on-chip Broadcast Cache   │
│ 3. TEXTURE MEMORY: Read-only off-chip DRAM with 2D/3D Spatial Cache    │
│ 4. SURFACE MEMORY: Read/Write off-chip DRAM with 2D/3D Spatial Cache   │
└────────────────────────────────────────────────────────────────────────┘
```

---

### 1. Shared Memory
* **Characteristics:** High-speed on-chip SRAM located directly on the Streaming Multiprocessor (SM). Latency is roughly 100x lower than global DRAM. Divided into 32 banks (subject to bank conflicts).
* **Scope:** Shared by all threads in the **same Thread Block**.
* **Use Case:** Tiled matrix multiplication, 1D/2D stencils, reduction operations.

#### Example (1D Stencil using Shared Memory):
```cuda
__global__ void stencil_1D(const float *in, float *out, int n) {
    __shared__ float s_data[256 + 2]; // Shared scratchpad memory

    int g_idx = blockDim.x * blockIdx.x + threadIdx.x;
    int s_idx = threadIdx.x + 1; // Center offset

    // Cooperative loading from global DRAM into on-chip shared memory
    if (g_idx < n) {
        s_data[s_idx] = in[g_idx];
    }
    __syncthreads(); // Barrier: Ensure all threads finished loading

    // Compute stencil using fast shared memory
    if (g_idx > 0 && g_idx < n - 1) {
        out[g_idx] = 0.25f * s_data[s_idx - 1] + 0.5f * s_data[s_idx] + 0.25f * s_data[s_idx + 1];
    }
}
```

---

### 2. Constant Memory
* **Characteristics:** 64 KB total allocation in off-chip DRAM, backed by an aggressive on-chip **constant cache**.
* **Key Feature (Broadcast Mechanism):** When all 32 threads of a warp read the **same memory address**, the value is broadcast to all 32 threads in **1 single clock cycle**. (If threads read different addresses, accesses are serialized).
* **Use Case:** Filter coefficients, convolution masks, physical constants.

#### Example (Image Blur with Constant Convolution Kernel):
```cuda
// Declared in global scope with __constant__ qualifier (Host writes, Device reads)
__constant__ float c_filter[9]; // 3x3 convolution mask

__global__ void applyFilter(const float *image, float *result, int width, int height) {
    int x = blockDim.x * blockIdx.x + threadIdx.x;
    int y = blockDim.y * blockIdx.y + threadIdx.y;

    if (x < width && y < height) {
        float sum = 0.0f;
        // All threads read identical filter coefficients -> Single-cycle broadcast!
        for (int i = 0; i < 9; i++) {
            sum += c_filter[i] * image[(y * width) + x];
        }
        result[y * width + x] = sum;
    }
}
```

---

### 3. Texture Memory
* **Characteristics:** Read-only data residing in device DRAM, cached via dedicated on-chip **texture caches**.
* **Key Feature (Spatial 2D/3D Locality):** Standard CPU/GPU caches exploit linear (1D) address locality. The texture cache is designed for **2D/3D spatial locality** (accessing $(x, y)$ brings neighboring pixels $(x \pm 1, y \pm 1)$ into cache).
* **Hardware Features:** Hardware boundary clamping, wrapping, and free hardware bilinear interpolation.
* **Use Case:** Ray tracing, computer graphics, medical imaging, seismic volume rendering.

---

### 4. Surface Memory
* **Characteristics:** Read/Write equivalent of texture memory. It allows reading and writing 2D or 3D arrays through the same hardware texture cache format.
* **Use Case:** Iterative image processing, compute-to-texture rendering pipelines.

---

## Question 5 [08 Marks]
> **Describe warp divergence and its impact on execution efficiency. Explain methods to reduce it.**

---

### 1. What is Warp Divergence?
In the SIMT architecture, all 32 threads within a warp share a single instruction decoder and execute in lock-step.
**Warp divergence** occurs when threads within the same warp evaluate a data-dependent conditional branch (`if-else`, `switch`, `while`) differently—some threads evaluate condition as `true`, while others evaluate it as `false`.

```
                  Conditional Statement: if (threadIdx.x % 2 == 0)
                                          │
         ┌────────────────────────────────┴────────────────────────────────┐
         ▼                                                                 ▼
  Even Threads (0, 2, 4...)                                         Odd Threads (1, 3, 5...)
  Active: 16 threads                                                Active: 16 threads
  Masked Out: 16 threads                                            Masked Out: 16 threads
  ┌─────────────────────────────────────────┐                       ┌─────────────────────────────────────────┐
  │         Execute THEN-Clause             │                       │         Execute ELSE-Clause             │
  └─────────────────────────────────────────┘                       └─────────────────────────────────────────┘
                                          │
                                          ▼
                               Path Serialization!
                         Both paths executed sequentially
```

---

### 2. Impact on Execution Efficiency:
1. **Path Serialization:** The hardware cannot execute both branches simultaneously for the same warp. It must execute the `then` path first (masking out false threads), and then execute the `else` path (masking out true threads).
2. **Drop in SIMD ALU Utilization:**
   * If 16 threads take `then` and 16 take `else`, half the execution lanes sit idle during each branch.
   * **Execution efficiency drops to 50%**.
   * If each of the 32 threads takes a distinct branch in a `switch` statement, execution is serialized 32 times (**SIMD efficiency drops to $\frac{1}{32} \approx 3.1\%$**).

---

### 3. Methods to Reduce Warp Divergence:
1. **Align Branch Conditions with Warp Boundaries (Multiples of 32):**
   * Structure conditional statements so that entire warps branch together rather than splitting internally:
     ```cuda
     // BAD: High Divergence (Splits every warp in half!)
     if (threadIdx.x % 2 == 0) { ... }

     // GOOD: Zero Divergence (Entire warps of 32 threads take the same path!)
     if (threadIdx.x / 32 == 0) { ... } // Warp 0 takes path A; Warp 1 takes path B
     ```
2. **Data Partitioning & Sorting:**
   * Sort input data before launching the kernel so that items requiring similar computational paths are processed by consecutive threads.
3. **Use Predication / Branch Elimination:**
   * Replace simple `if-else` assignments with mathematical expressions or ternary operators that the compiler translates into hardware predicated instructions (without branching):
     ```cuda
     // Avoids explicit branch:
     float val = (x > 0) ? (x * 2.0f) : 0.0f;
     ```
4. **Interleaved vs. Strided Loop Transformations (e.g., Parallel Reductions):**
   * In tree-based reductions, use strided thread indexing rather than interleaved modulo indexing to ensure idle threads are packed into entirely deactivated warps.

---

## Question 6 [08 Marks]
> **Explain loop unrolling in enhancing loop-level parallelism with example.**

### 1. Concept:
**Loop unrolling** replicates the loop body multiple times while proportionately reducing the trip count and adjusting loop termination tests.
It exposes **Loop-Level Parallelism (LLP)** by:
1. **Amortizing Branch & Counter Overhead:** Decreases the total number of conditional branches and pointer increment instructions.
2. **Exposing Independent Operations:** Enables the compiler or out-of-order execution engine to overlap independent operations from adjacent iterations, filling pipeline latency slots and eliminating data hazard stalls.

---

### 2. Concrete Example:
Consider a scalar loop computing vector addition:
```c
for (int i = 0; i < 1000; i++) {
    A[i] = B[i] + C[i];
}
```

#### Unrolled by a factor of 4:
```c
for (int i = 0; i < 1000; i += 4) {
    A[i]     = B[i]     + C[i];     // Iteration 0
    A[i + 1] = B[i + 1] + C[i + 1]; // Iteration 1 (Independent of Iteration 0!)
    A[i + 2] = B[i + 2] + C[i + 2]; // Iteration 2 (Independent of Iteration 0 & 1!)
    A[i + 3] = B[i + 3] + C[i + 3]; // Iteration 3 (Independent!)
}
```

#### Why it enhances parallelism:
* **In the original loop:** A dependent instruction in iteration $i$ is immediately followed by loop counter updates and branch resolution.
* **In the unrolled loop:** The 4 load instructions (`B[i], B[i+1], B[i+2], B[i+3]`) are completely independent. The hardware can issue all 4 loads concurrently, issue all 4 adds concurrently, and issue all 4 stores concurrently, **reducing CPI from 9.0 cycles per element to 3.5 cycles per element with zero stalls**.


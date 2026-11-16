# Unit – III: CUDA Processing Flow, Matrix Multiplication, DAXPY & GPU Architecture
## Exam Question Bank Solutions (Part 6)

**Syllabus Mapping:**
* **Vector Architecture & Parallelism:** Linpack DAXPY (MIPS vs. VMIPS with performance gain analysis), Strip Mining, Loop-Carried Dependences, GCD Test.
* **GPU Programming with CUDA:** Relationship between Warps, Thread Blocks, and CUDA Cores; Typical CUDA Program Processing Flow; CUDA C Matrix Multiplication Kernel & Host Program; NVIDIA Fermi Architectural Innovations.

---

# SECTION A: SHORT QUESTIONS (Unit-3 Only)

---

### Question 1.1 [01 Mark]
> **Identify the relationship between Warps, Thread Blocks, and CUDA Cores.**

#### Answer:
```
Thread Block (up to 1024 threads)
       │
       ▼ Partitioned in hardware into:
Warps (Groups of 32 consecutive threads)
       │
       ▼ Dispatched to:
CUDA Cores (32 physical ALU lanes executing 1 warp in lock-step per cycle)
```
* **Thread Block:** A software abstraction representing a group of up to 1024 concurrent threads assigned to a single Streaming Multiprocessor (SM).
* **Warp:** The physical unit of SIMT instruction dispatch. Each thread block is divided into **warps of 32 threads** (`threadIdx` 0–31, 32–63, etc.).
* **CUDA Cores:** The physical execution hardware (ALU pipelines). An SM dispatches a warp so that all 32 threads execute in parallel across **32 CUDA Cores** in lock-step.

---

### Question 1.6 [02 Marks]
> **Enumerate the use of Strip Mining.**

#### Answer:
**Strip mining** is a compiler loop-transformation technique used in vector processors when a loop's iteration count $N$ exceeds the hardware **Maximum Vector Length (`MVL`)** (typically 64 elements):
1. **Splits Arbitrary Loop Bounds:** Breaks a loop of arbitrary size $N$ into an initial remainder chunk of size $N \pmod{\text{MVL}}$ followed by full chunks of size $\text{MVL}$.
2. **Eliminates Code Explosion:** Replaces thousands of scalar instruction loops with a compact outer scalar loop and an inner vectorized body.
3. **Bounds Protection:** Sets the Vector Length Register (`VLR`) dynamically, preventing vector units from writing outside valid array boundaries.

---

### Question 1.7 [02 Marks]
> **What are loop-carried dependences?**

#### Answer:
A **loop-carried dependence** is a data dependence where a memory location or variable produced/written in one iteration (e.g., iteration $i$) of a loop is accessed/read in a *subsequent* iteration (e.g., iteration $i+1, i+2, \dots$) of the same loop.
* *Example:*
  ```c
  for (int i = 1; i < N; i++) {
      A[i] = A[i - 1] + B[i]; // Iteration i depends on A[i-1] from iteration i-1
  }
  ```
* **Impact:** Prevents concurrent loop execution across parallel vector lanes or CUDA threads; enforces sequential ordering.

---

### Question 1.8 [02 Marks]
> **Use the GCD test to determine whether dependences exist in loops.**

#### Answer:
For an array access with store index $a \cdot i + b$ and load index $c \cdot j + d$, a loop-carried dependence exists between iterations $i$ and $j$ if and only if:
$$\mathbf{\gcd(a, c) \text{ divides } (d - b)}$$
* If $\gcd(a, c)$ does **not** divide $(d - b)$, **no integer solution exists**, meaning the store and load never access the same memory location. The loop is guaranteed to be parallelizable without synchronization.

---

# SECTION B: LONG QUESTIONS (07, 08 & 09 Marks)

---

## Question 5.a [07 Marks]
> **Consider the DAXPY loop that forms the inner loop of the Linpack benchmark ($Y = a \times X + Y$). $X$ and $Y$ are vectors, initially resident in memory, and $a$ is a scalar.**  
> **Show the code for MIPS and VMIPS for this loop. Assume that the starting addresses of $X$ and $Y$ are in $Rx$ and $Ry$ respectively. Indicate the performance gain obtained using VMIPS.**

---

### 1. Scalar MIPS Assembly Code (Processes 64 Elements Iteratively)
Assume scalar $a$ is in $F0$, $Rx$ and $Ry$ are base addresses of arrays $X$ and $Y$, and $R1$ contains the terminal address (`Rx + 512` bytes):

```assembly
# Scalar MIPS DAXPY Loop (Processes 1 element per iteration)
Loop:
    L.D    F2, 0(Rx)        # Load X[i]
    MUL.D  F2, F2, F0       # Compute a * X[i]
    L.D    F4, 0(Ry)        # Load Y[i]
    ADD.D  F4, F4, F2       # Compute a * X[i] + Y[i]
    S.D    F4, 0(Ry)        # Store result into Y[i]
    DADDIU Rx, Rx, #8       # Increment X pointer by 8 bytes (double-precision)
    DADDIU Ry, Ry, #8       # Increment Y pointer by 8 bytes
    DSUBU  R2, R1, Rx       # Check boundary: R2 = R1 - Rx
    BNEZ   R2, Loop         # Branch back to Loop if R2 != 0
```

---

### 2. Vector VMIPS Assembly Code (Processes 64 Elements in One Pass)
Assume scalar $a$ is in $F0$, and the Vector Length Register (`VLR`) is set to 64:

```assembly
# Vector VMIPS DAXPY Code (Processes 64 elements at once)
LV      V1, Rx          # Load 64 double-precision words of array X into V1
MULVS.D V2, V1, F0      # Vector-Scalar Multiply: V2[i] = V1[i] * F0
LV      V3, Ry          # Load 64 double-precision words of array Y into V3
ADDVV.D V4, V2, V3      # Vector-Vector Add: V4[i] = V2[i] + V3[i]
SV      V4, Ry          # Store vector V4 back to memory at Ry
```

---

### 3. Detailed Performance Gain Analysis of VMIPS over MIPS:

| Performance Metric | Scalar MIPS | Vector VMIPS | Advantage of VMIPS |
| :--- | :---: | :---: | :--- |
| **Dynamic Instruction Count** | **576 instructions** ($64 \times 9$) | **5 instructions** | **$115\times$ reduction** in dynamic instruction fetches |
| **Loop Control & Branch Overhead** | 64 pointer updates + 64 branches | **0 branches, 0 pointer updates** | Completely eliminates branch hazards and mispredictions |
| **Pipeline Hazarding** | Stalls on load-use & FP latency each iteration | Overlapped across functional units via **Chaining** | Eliminates inter-iteration stalls |
| **Convoys and Chimes** | Not applicable | **3 Convoys (3 Chimes)** | Runs in $3 \times 64 = 192$ cycles (or 48 cycles on 4 lanes) |
| **Cycles per FLOP** | $\approx 4.5 - 6.0\text{ cycles/FLOP}$ | **$1.5\text{ cycles/FLOP}$** | **$3\times \text{ to } 4\times$ speedup** on a single lane |
| **Multi-Lane Scaling ($L = 4$)** | Cannot scale across lanes | $48\text{ cycles total} \implies \mathbf{0.375\text{ cycles/FLOP}}$ | **Over $10\times - 15\times$ total speedup** |

---

## Question 5.b [09 Marks]
> **Analyze the major innovations introduced by Fermi to bring GPUs much closer to mainstream system processors.**

### NVIDIA Fermi Architecture Overview:
Before Fermi (GF100, 2010), GPUs were primarily graphics accelerators with rigid memory structures and limited floating-point precision. Fermi introduced five architectural pillars that turned GPUs into true general-purpose HPC processors:

```
                            FERMI INNOVATIONS
 ┌────────────────────────────────────────────────────────────────────────┐
 │ 1. Unified 64-Bit Virtual Address Space (C/C++ Pointer Compliance)     │
 │ 2. IEEE 754-2008 Double-Precision Floating-Point with Hardware FMA     │
 │ 3. True Two-Level Hardware Cache Hierarchy (Configurable L1 + L2)      │
 │ 4. Hardware ECC (Error-Correcting Code) for Datacenter Reliability     │
 │ 5. GigaThread Engine Supporting Concurrent Kernel Execution            │
 └────────────────────────────────────────────────────────────────────────┘
```

1. **Unified 64-Bit Memory Addressing (True C++ Pointer Support):**
   * *Limitation in Pre-Fermi GPUs:* Memory was divided into disjoint, non-contiguous physical spaces (registers, shared, global). Pointers could not cross spaces.
   * *Fermi Innovation:* Created a single, unified **flat 64-bit virtual address space**. A single generic pointer can reference shared memory, local memory, or global DRAM interchangeably. Unlocked full support for C++ virtual functions, dynamic object instantiation (`new`/`delete`), and function pointers.
2. **High-Performance IEEE 754-2008 Double-Precision Floating-Point:**
   * *Limitation in Pre-Fermi GPUs:* Double-precision (FP64) performance was an afterthought ($0.1\times$ of FP32).
   * *Fermi Innovation:* Dedicated FP64 units delivering **$0.5\times$ FP32 throughput**. Implemented hardware **Fused Multiply-Add (FMA)**, computing $A \times B + C$ with a single rounding step at the end, eliminating intermediate precision loss and exceeding IEEE 754 standards.
3. **True Two-Level Hardware Cache Hierarchy:**
   * *First GPU with true L1 and L2 caches:*
     * **Configurable L1 Cache / Shared Memory (64 KB per SM):** Programmers could configure on-chip SRAM via software:
       - $48\text{ KB Shared Memory} / 16\text{ KB L1 Cache}$ (for algorithms heavily utilizing shared scratchpad), OR
       - $16\text{ KB Shared Memory} / 48\text{ KB L1 Cache}$ (for irregular codes relying on automated hardware caching).
     * **Unified 768 KB Coherent L2 Cache:** Coherent across all SMs; automatically caches global memory reads and writes.
4. **Full ECC (Error-Correcting Code) Memory Protection:**
   * Provided hardware single-bit error correction and double-bit error detection (SECDED) across register files, shared memory, L1/L2 caches, and off-chip GDDR5 DRAM.
   * Essential for mission-critical scientific applications and supercomputers where soft errors caused by thermal or cosmic radiation cannot be tolerated.
5. **GigaThread Engine & Concurrent Kernel Execution:**
   * Enabled up to 16 different CUDA kernels from the same application to execute concurrently across separate SMs, preventing smaller parallel kernels from underutilizing GPU resources.

---

## Question 6.b [10 Marks]
> **Illustrate the typical processing flow of a CUDA program highlighting all the important components.**  
> **Write a CUDA C program to compute the product of two matrices.**

---

### Part 1: Typical CUDA Processing Flow

```
   HOST (CPU)                                            DEVICE (GPU)
   ──────────                                            ────────────
┌─────────────────────────────────┐
│ 1. Allocate Host Memory         │
│    (malloc) & Initialize Data   │
└────────────────┬────────────────┘
                 │
                 ▼
┌─────────────────────────────────┐                 ┌───────────────────────────────┐
│ 2. Allocate Device Memory       │────────────────►│ Device DRAM Allocated         │
│    (cudaMalloc)                 │                 │ (d_A, d_B, d_C)               │
└────────────────┬────────────────┘                 └───────────────┬───────────────┘
                 │                                                  │
                 ▼                                                  │
┌─────────────────────────────────┐   PCIe Host-to-Device Transfer  │
│ 3. Copy Inputs to GPU           │────────────────────────────────►│
│    (cudaMemcpyHostToDevice)     │                                 │
└────────────────┬────────────────┘                                 │
                 │                                                  │
                 ▼                                                  │
┌─────────────────────────────────┐   Launch Kernel                 ▼
│ 4. Configure Grid & Launch      │────────────────►┌───────────────────────────────┐
│    Kernel <<<Grids, Blocks>>>   │                 │ GPU Executes Thousands        │
└────────────────┬────────────────┘                 │ of Threads in Parallel (SIMT) │
                 │                                  └───────────────┬───────────────┘
                 ▼                                                  │
┌─────────────────────────────────┐   PCIe Device-to-Host Transfer  │
│ 5. Copy Results to CPU          │◄────────────────────────────────┘
│    (cudaMemcpyDeviceToHost)     │
└────────────────┬────────────────┘
                 │
                 ▼
┌─────────────────────────────────┐
│ 6. Verify Results & Free Memory │
│    (cudaFree, free)             │
└─────────────────────────────────┘
```

#### Important Components:
1. **Host (CPU) vs. Device (GPU):** The CPU acts as the control processor; the GPU acts as the massively parallel co-processor.
2. **`cudaMalloc` & `cudaFree`:** Manages memory allocation/deallocation on GPU off-chip DRAM.
3. **`cudaMemcpy`:** Manages explicit data transfers across the high-latency PCIe bus.
4. **Execution Configuration `<<<gridDim, blockDim>>>`:** Maps the software problem space into a grid of 1D/2D/3D thread blocks.
5. **Device Kernel (`__global__`):** The parallel code executed across all threads.

---

### Part 2: CUDA C Program to Compute Product of Two Matrices ($C = A \times B$)

```cuda
#include <stdio.h>
#include <stdlib.h>
#include <cuda_runtime.h>

#define N 512 // Matrix dimensions: N x N
#define BLOCK_SIZE 16 // 16x16 threads per block = 256 threads

// CUDA Kernel for Matrix Multiplication: C = A * B
__global__ void matrixMulKernel(const float *A, const float *B, float *C, int n) {
    // Compute 2D row and column index for this thread
    int row = blockDim.y * blockIdx.y + threadIdx.y;
    int col = blockDim.x * blockIdx.x + threadIdx.x;

    // Check boundary
    if (row < n && col < n) {
        float sum = 0.0f;
        // Compute dot product of row of A and column of B
        for (int k = 0; k < n; k++) {
            sum += A[row * n + k] * B[k * n + col];
        }
        C[row * n + col] = sum;
    }
}

int main() {
    size_t size = N * N * sizeof(float);

    // 1. Allocate Host memory
    float *h_A = (float *)malloc(size);
    float *h_B = (float *)malloc(size);
    float *h_C = (float *)malloc(size);

    // Initialize matrices
    for (int i = 0; i < N * N; i++) {
        h_A[i] = 1.0f;
        h_B[i] = 2.0f;
    }

    // 2. Allocate Device global memory
    float *d_A, *d_B, *d_C;
    cudaMalloc((void **)&d_A, size);
    cudaMalloc((void **)&d_B, size);
    cudaMalloc((void **)&d_C, size);

    // 3. Copy matrices from Host to Device
    cudaMemcpy(d_A, h_A, size, cudaMemcpyHostToDevice);
    cudaMemcpy(d_B, h_B, size, cudaMemcpyHostToDevice);

    // 4. Configure 2D Grid and Block Dimensions
    dim3 threadsPerBlock(BLOCK_SIZE, BLOCK_SIZE); // 16 x 16 = 256 threads
    dim3 blocksPerGrid((N + BLOCK_SIZE - 1) / BLOCK_SIZE, 
                       (N + BLOCK_SIZE - 1) / BLOCK_SIZE);

    printf("Launching Matrix Mul Kernel with Grid (%d, %d)...\n", 
           blocksPerGrid.x, blocksPerGrid.y);

    // 5. Launch Kernel on Device
    matrixMulKernel<<<blocksPerGrid, threadsPerBlock>>>(d_A, d_B, d_C, N);

    // Synchronize to ensure GPU completes
    cudaDeviceSynchronize();

    // 6. Copy computed result matrix C back to Host
    cudaMemcpy(h_C, d_C, size, cudaMemcpyDeviceToHost);

    // 7. Verify result (each element of C should be N * (1.0 * 2.0) = 512 * 2 = 1024.0)
    printf("Sample result C[0][0] = %f (Expected: %f)\n", h_C[0], (float)(N * 2.0f));

    // 8. Free Device and Host memory
    cudaFree(d_A);
    cudaFree(d_B);
    cudaFree(d_C);
    free(h_A);
    free(h_B);
    free(h_C);

    return 0;
}
```

### Technical Highlights of the Matrix Multiplication Code:
1. **2D Thread Indexing:**
   $$\text{Row} = \text{blockDim.y} \times \text{blockIdx.y} + \text{threadIdx.y}$$
   $$\text{Col} = \text{blockDim.x} \times \text{blockIdx.x} + \text{threadIdx.x}$$
2. **Flattened 1D Array Offset:** 2D element $[i][j]$ maps to 1D index `row * n + col`.
3. **Execution Configuration:** Uses a 2D grid of 2D blocks `dim3(16, 16)`, launching $512 \times 512 = 262,144$ concurrent threads, computing all matrix elements in parallel!


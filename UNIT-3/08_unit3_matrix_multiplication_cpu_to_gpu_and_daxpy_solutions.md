# Unit – III: Host-to-Device Matrix Multiplication, CUDA DAXPY & Fermi Innovations
## Exam Question Bank Solutions (Part 8)

**Syllabus Mapping:**
* **GPU Programming with CUDA:** Shared memory scope, CUDA Kernels definition, CPU Host Matrix Multiplication vs. GPU Device Matrix Multiplication, CUDA DAXPY Kernel & Launch.
* **Vector Architecture:** VMIPS components, DAXPY in VMIPS, Convoy and Chime identification.
* **Graphics Processing Units:** NVIDIA Fermi architectural innovations.

---

# SECTION A: SHORT QUESTIONS (Unit-3 Only)

---

### Question 1 [01 Mark]
> **With multiple threads, shared memory in CUDA is accessible to `___________`.**

#### Answer:
**all threads within the same Thread Block** (or **a single Thread Block**).

#### Explanation:
In the CUDA memory hierarchy, **Shared Memory** is an on-chip, low-latency scratchpad memory allocated per Streaming Multiprocessor (SM). Its scope is strictly confined to a single **Thread Block**:
* Any thread within that block can read and write to it.
* Threads from different blocks cannot access each other's shared memory.
* It facilitates low-latency data reuse and inter-thread synchronization via `__syncthreads()`.

---

### Question 1.8 [01 Mark]
> **Parallel portions of an application executed on the device are referred to as `___________`.**

#### Answer:
**kernels** (or **CUDA kernels** / **device code**).

#### Explanation:
In CUDA, an application is split into:
1. **Host Code:** Runs on the CPU, handles sequential logic, system I/O, memory allocations, and kernel launches.
2. **Device Code (Kernel):** Functions declared with the `__global__` qualifier that are executed in parallel across thousands of threads on the GPU device.

---

### Question 3 [02 Marks]
> **The following code sequence lays out into how many convoys? Indicate them.**
> ```assembly
> LV      V1, Rx         ; load vector X
> MULVS.D V2, V1, F0     ; vector-scalar multiply
> LV      V3, Ry         ; load vector Y
> ADDVV.D V4, V2, V3     ; add two vectors
> SV      V4, Ry         ; store the sum
> ```

#### Answer:
Assuming a standard VMIPS vector processor with **one memory port** and support for **chaining**, the sequence requires **3 Convoys (3 Chimes)**:

* **Convoy 1:**
  ```assembly
  LV      V1, Rx
  MULVS.D V2, V1, F0      ; Chained to LV V1
  ```
  *(Can run in parallel because they use different functional units: Vector Load Unit and FP Multiplier).*

* **Convoy 2:**
  ```assembly
  LV      V3, Ry          ; Requires memory unit (structural conflict with Convoy 1)
  ADDVV.D V4, V2, V3      ; Chained to LV V3 and MULVS.D V2
  ```
  *(Must be placed in a new convoy because there is only one memory port for loads/stores).*

* **Convoy 3:**
  ```assembly
  SV      V4, Ry          ; Requires memory unit (store)
  ```
  *(Must be in a new convoy because Convoy 2 is using the memory port for LV V3).*

**Total Convoys = 3 Convoys (3 Chimes).**

---

# SECTION B: LONG & CODE QUESTIONS (04, 08 & 10 Marks)

---

## Question 5.a [08 Marks]
> **Discuss the major innovations introduced by Fermi to GPU architecture.**

### The NVIDIA Fermi (GF100, 2010) Microarchitecture
Fermi fundamentally transformed GPUs from specialized 3D graphics hardware into high-performance general-purpose computing (HPC) processors.

```
                           FERMI KEY INNOVATIONS
 ┌────────────────────────────────────────────────────────────────────────┐
 │ 1. Unified 64-Bit Virtual Address Space (C/C++ Pointer Compliance)     │
 │ 2. Fast IEEE 754-2008 Double-Precision Floating-Point (FMA)            │
 │ 3. Unified Two-Level Cache Hierarchy (Configurable L1 / Shared + L2)   │
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

## Question 5.b [04 Marks]
> **Show the code of VMIPS for the DAXPY loop ($Y = a \times X + Y$). Assume that the starting addresses of $X$ and $Y$ are in $Rx$ and $Ry$, respectively.**

### Vector VMIPS Assembly Code (Processes 64 Elements in One Pass):
Assume scalar $a$ is in scalar FP register $F0$, and the Vector Length Register (`VLR`) is set to 64:

```assembly
# Vector VMIPS DAXPY Code (Processes 64 elements at once)
LV      V1, Rx          # Load 64 double-precision words of array X into V1
MULVS.D V2, V1, F0      # Vector-Scalar Multiply: V2[i] = V1[i] * F0
LV      V3, Ry          # Load 64 double-precision words of array Y into V3
ADDVV.D V4, V2, V3      # Vector-Vector Add: V4[i] = V2[i] + V3[i]
SV      V4, Ry          # Store vector V4 back to memory at Ry
```

* **Dynamic Instruction Count:** Exactly **$5\text{ vector instructions}$** (down from 576 scalar instructions in MIPS).
* **Execution:** Operates in 3 convoys with chaining.

---

## Question 5.c [04 Marks]
> **Write CUDA code snippet for the DAXPY loop ($Y = a \times X + Y$), to launch $n$ threads, one per vector element, with 256 CUDA threads per Thread block in a multithreaded SIMD processor.**

### CUDA Kernel and Launch Code Snippet:

```cuda
// 1. CUDA Device Kernel for DAXPY: Y[i] = a * X[i] + Y[i]
__global__ void daxpy_kernel(int n, double a, const double *X, double *Y) {
    // Unique global thread ID:
    int i = blockDim.x * blockIdx.x + threadIdx.x;

    // Boundary check to prevent out-of-bounds access
    if (i < n) {
        Y[i] = a * X[i] + Y[i];
    }
}

// 2. Host Launch Snippet
void launch_daxpy(int n, double a, double *d_X, double *d_Y) {
    // Exactly 256 threads per thread block as requested
    int threadsPerBlock = 256;

    // Calculate number of blocks needed to launch at least n threads
    int blocksPerGrid = (n + threadsPerBlock - 1) / threadsPerBlock;

    // Kernel launch on GPU
    daxpy_kernel<<<blocksPerGrid, threadsPerBlock>>>(n, a, d_X, d_Y);

    // Wait for all GPU threads to finish
    cudaDeviceSynchronize();
}
```

---

## Question 6.a [08 Marks]
> **Draw the basic structure of vector architecture for VMIPS and explain its components.**

### VMIPS Block Diagram:

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

### Components:
1. **Vector Register File:** 8 vector registers ($V0 - V7$), each holding $64 \times 64$-bit double-precision elements.
2. **Pipelined Functional Units:** Pipelined FP Add, FP Mult, FP Div, and Integer units producing 1 result/cycle.
3. **Control Registers:** `VLR` (sets active length for non-multiple-of-64 loops) and `VMR` (64-bit mask for conditional branches).
4. **Vector Load / Store Unit:** Pipelined memory unit supporting unit stride, non-unit stride (`LVWS`), and gather/scatter (`LVI`/`SVI`).
5. **Multi-Bank Interleaved Memory:** Interleaves memory across independent banks to avoid bank conflicts and sustain full memory bandwidth.

---

## Question 6.b [08 Marks]
> **Write a simple function in CUDA for matrix multiplication with only host code. Discuss the salient aspects of CUDA device to execute the same. Revise the function that moves matrix multiplication to a device.**

---

### Part 1: Simple Matrix Multiplication on CPU (Host-Only Code)

On the CPU, matrix multiplication of two $N \times N$ matrices ($C = A \times B$) is executed sequentially using three nested loops:

```c
// Host-only CPU function for Matrix Multiplication
void matrixMulHost(const float *A, const float *B, float *C, int n) {
    for (int i = 0; i < n; i++) {           // Iterate over rows of A
        for (int j = 0; j < n; j++) {       // Iterate over columns of B
            float sum = 0.0f;
            for (int k = 0; k < n; k++) {   // Dot product
                sum += A[i * n + k] * B[k * n + j];
            }
            C[i * n + j] = sum;             // Store element C[i][j]
        }
    }
}
```
* **Performance Characteristic:** Complexity is $\mathcal{O}(N^3)$. For $N = 1024$, this requires over $2 \times 10^9$ operations executed sequentially on a single thread.

---

### Part 2: Salient Aspects of CUDA Device to Execute Matrix Multiplication

To accelerate this workload on a GPU device, several architectural capabilities are leveraged:
1. **Massive Fine-Grained Parallelism (SIMT):**
   * Instead of computing $N \times N$ elements serially, the GPU launches **$N \times N$ threads concurrently**—each thread computes exactly **one element** of matrix $C$.
2. **2D Thread and Block Hierarchical Organization:**
   * The matrix naturally maps onto a **2D Grid of 2D Blocks** (`dim3(BLOCK_SIZE, BLOCK_SIZE)`), matching the 2D coordinate system of rows and columns.
3. **Thread Mapping Coordinates:**
   * Each thread independently computes its target coordinates without loop control:
     $$\text{Row } (i) = \text{blockDim.y} \times \text{blockIdx.y} + \text{threadIdx.y}$$
     $$\text{Col } (j) = \text{blockDim.x} \times \text{blockIdx.x} + \text{threadIdx.x}$$
4. **Latency Hiding via Warp Scheduling:**
   * While threads wait to fetch elements of $A$ and $B$ from global memory (200–400 cycle latency), the warp scheduler switches to other ready warps in 0 cycles, maintaining maximum ALU throughput.

---

### Part 3: Revised Function Moving Matrix Multiplication to Device (GPU)

```cuda
#include <stdio.h>
#include <stdlib.h>
#include <cuda_runtime.h>

#define BLOCK_SIZE 16 // 16x16 threads = 256 threads per block

// 1. Device Kernel: Executed in parallel on GPU
__global__ void matrixMulDeviceKernel(const float *A, const float *B, float *C, int n) {
    // Determine 2D coordinates for this thread
    int row = blockDim.y * blockIdx.y + threadIdx.y;
    int col = blockDim.x * blockIdx.x + threadIdx.x;

    // Boundary check
    if (row < n && col < n) {
        float sum = 0.0f;
        // Compute dot product for element C[row][col]
        for (int k = 0; k < n; k++) {
            sum += A[row * n + k] * B[k * n + col];
        }
        C[row * n + col] = sum;
    }
}

// 2. Revised Host Wrapper Function that offloads execution to the Device
void matrixMulDevice(const float *h_A, const float *h_B, float *h_C, int n) {
    size_t size = n * n * sizeof(float);

    // Step a: Allocate Device memory
    float *d_A, *d_B, *d_C;
    cudaMalloc((void **)&d_A, size);
    cudaMalloc((void **)&d_B, size);
    cudaMalloc((void **)&d_C, size);

    // Step b: Transfer input matrices from Host to Device
    cudaMemcpy(d_A, h_A, size, cudaMemcpyHostToDevice);
    cudaMemcpy(d_B, h_B, size, cudaMemcpyHostToDevice);

    // Step c: Configure 2D Grid and Block execution dimensions
    dim3 threadsPerBlock(BLOCK_SIZE, BLOCK_SIZE);
    dim3 blocksPerGrid((n + BLOCK_SIZE - 1) / BLOCK_SIZE, 
                       (n + BLOCK_SIZE - 1) / BLOCK_SIZE);

    // Step d: Launch Device Kernel
    matrixMulDeviceKernel<<<blocksPerGrid, threadsPerBlock>>>(d_A, d_B, d_C, n);

    // Wait for all threads to finish
    cudaDeviceSynchronize();

    // Step e: Transfer result matrix C back from Device to Host
    cudaMemcpy(h_C, d_C, size, cudaMemcpyDeviceToHost);

    // Step f: Free Device memory
    cudaFree(d_A);
    cudaFree(d_B);
    cudaFree(d_C);
}
```


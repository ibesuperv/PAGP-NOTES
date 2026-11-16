# Unit – III: Data-Level Parallelism & GPU Programming with CUDA
## Exam Question Bank Solutions (QUIZ-2 & TEST-2)

**Syllabus Mapping:**
* **Part A:** Data-Level Parallelism in Vector, SIMD, and GPU Architectures (Vector Architecture, Gather/Scatter, Chaining, Memory Banks, Strip Mining, Loop-Level Parallelism).
* **Part B:** GPU Programming with CUDA (Threads/Blocks/Grids, Streaming Multiprocessors & Warps, Memory Hierarchy, Kernel Invocation & Execution Model).

---

# PART 1: QUIZ-2 QUESTIONS (Short Answer)

---

### Question 1 [2 Marks]
> **The VMIPS instruction for gather and scatter are `_______` and `_______`.**

#### Answer:
* Gather: **`LVI` (Load Vector Indexed)**
* Scatter: **`SVI` (Store Vector Indexed)**

#### Technical Explanation:
* **Gather (`LVI` / `LVX`):** Loads non-contiguous data from memory into a contiguous vector register using a vector of memory offsets/addresses:
  $$\text{VectorReg}[i] = \text{Memory}[\text{Base} + \text{IndexVector}[i]]$$
* **Scatter (`SVI` / `SVX`):** Takes elements from a contiguous vector register and writes them out to non-contiguous, disparate memory addresses specified by an index vector:
  $$\text{Memory}[\text{Base} + \text{IndexVector}[i]] = \text{VectorReg}[i]$$
* **Use Case:** Crucial for sparse matrix computations and pointer-based data structures in scientific computing.

---

### Question 2 [2 Marks]
> **What is vector register chaining?**

#### Answer:
**Vector register chaining** is a hardware pipelining optimization in vector processors (analogous to operand forwarding/bypassing in scalar pipelines) that allows a dependent vector operation to begin execution as soon as the **first individual result element** of its source vector operand becomes available from an earlier vector functional unit, rather than waiting for the entire vector register (all 64 elements) to be computed.

```
Without Chaining (Serialized across entire registers):
Vector Load : [ e0 | e1 | e2 | ... | e63 ] ──► (Wait 64 cycles)
Vector Mult :                                      └──► [ e0 | e1 | ... | e63 ]

With Chaining (Element-by-Element Pipelined Forwarding):
Vector Load : [ e0 | e1 | e2 | e3 | ... | e63 ]
                 │    │    │    │
                 ▼    ▼    ▼    ▼
Vector Mult :    [ e0 | e1 | e2 | e3 | ... | e63 ]
```
* **Benefit:** Allows dependent vector operations to execute in the **same convoy**, drastically reducing execution time (chimes).

---

### Question 3 [2 Marks]
> **Define loop-carried dependence, and why does it reduce loop-level parallelism?**

#### Answer:
#### Definition:
A **loop-carried dependence** occurs when an operation in iteration $i$ of a loop accesses a variable or memory location that was computed or modified in an earlier iteration (such as iteration $i-1$) of the same loop.
* *Example:*
  ```c
  for (int i = 1; i < N; i++) {
      A[i] = A[i - 1] + B[i]; // Iteration i depends on A[i-1] from iteration i-1
  }
  ```

#### Why it reduces loop-level parallelism:
1. **Forces Sequential Execution:** Iteration $i$ cannot start until iteration $i-1$ has finished computing and writing its result back to memory.
2. **Inhibits Vectorization / SIMD:** The loop iterations cannot be dispatched concurrently across independent vector lanes, SIMD execution units, or CUDA thread blocks. The compiler must retain serial execution order.

---

### Question 4 [2 Marks]
> **Define a warp.**

#### Answer:
In the NVIDIA CUDA / SIMT (Single Instruction, Multiple Threads) execution model, a **warp** is the basic unit of instruction scheduling and execution on a Streaming Multiprocessor (SM).
* **Composition:** A warp consists of exactly **32 consecutive parallel threads** (threads with `threadIdx.x` from $0$ to $31$, $32$ to $63$, etc.).
* **SIMT Execution:** All 32 threads in a warp share a single instruction fetch/decode unit and execute the same instruction at the same cycle.
* **Warp Divergence:** If threads within a warp branch in different directions during a conditional statement (`if-else`), the branches are serialized, degrading execution throughput.

---

# PART 2: TEST-2 QUESTIONS (Descriptive & Numerical)

---

## Question 2.a [10 Marks]
> **Write a CUDA program for Vector Addition.**

### Complete CUDA Source Code (`vector_add.cu`):

```cuda
#include <stdio.h>
#include <stdlib.h>
#include <cuda_runtime.h>

// CUDA Kernel: Executes on GPU (Device), called from CPU (Host)
__global__ void vectorAdd(const float *A, const float *B, float *C, int n) {
    // 1. Calculate global unique thread index across the entire grid
    int i = blockDim.x * blockIdx.x + threadIdx.x;

    // 2. Boundary condition: Ensure thread index is within array bounds
    if (i < n) {
        C[i] = A[i] + B[i];
    }
}

int main() {
    int n = 1000000; // 1 Million elements
    size_t size = n * sizeof(float);

    // Step 1: Allocate memory on Host (CPU)
    float *h_A = (float *)malloc(size);
    float *h_B = (float *)malloc(size);
    float *h_C = (float *)malloc(size);

    // Initialize input vectors
    for (int i = 0; i < n; i++) {
        h_A[i] = 1.0f;
        h_B[i] = 2.0f;
    }

    // Step 2: Allocate memory on Device (GPU DRAM)
    float *d_A, *d_B, *d_C;
    cudaMalloc((void **)&d_A, size);
    cudaMalloc((void **)&d_B, size);
    cudaMalloc((void **)&d_C, size);

    // Step 3: Copy input data from Host to Device
    cudaMemcpy(d_A, h_A, size, cudaMemcpyHostToDevice);
    cudaMemcpy(d_B, h_B, size, cudaMemcpyHostToDevice);

    // Step 4: Define Execution Configuration (Grid & Block Dimensions)
    int threadsPerBlock = 256;
    int blocksPerGrid = (n + threadsPerBlock - 1) / threadsPerBlock;

    printf("Launching CUDA Kernel with %d blocks of %d threads...\n", blocksPerGrid, threadsPerBlock);

    // Step 5: Launch Kernel on GPU
    vectorAdd<<<blocksPerGrid, threadsPerBlock>>>(d_A, d_B, d_C, n);

    // Wait for all GPU threads to finish
    cudaDeviceSynchronize();

    // Step 6: Copy computed result from Device back to Host
    cudaMemcpy(h_C, d_C, size, cudaMemcpyDeviceToHost);

    // Step 7: Verify correctness
    bool correct = true;
    for (int i = 0; i < n; i++) {
        if (h_C[i] != 3.0f) {
            correct = false;
            printf("Error at index %d: Expected 3.0, Got %f\n", i, h_C[i]);
            break;
        }
    }
    if (correct) {
        printf("SUCCESS: Vector addition completed correctly on GPU!\n");
    }

    // Step 8: Free Device and Host Memory
    cudaFree(d_A);
    cudaFree(d_B);
    cudaFree(d_C);
    free(h_A);
    free(h_B);
    free(h_C);

    return 0;
}
```

### Key Technical Concepts:
1. **`__global__` Qualifier:** Informs the NVIDIA `nvcc` compiler that this function runs on the GPU device and is invoked from the CPU host.
2. **Global Thread Indexing:**
   $$\text{Global ID } (i) = \text{blockDim.x} \times \text{blockIdx.x} + \text{threadIdx.x}$$
   * `blockDim.x`: Number of threads per block (256).
   * `blockIdx.x`: Index of the current block in the grid.
   * `threadIdx.x`: Local thread index within the block.
3. **Execution Configuration `<<<blocksPerGrid, threadsPerBlock>>>`:** Uses ceiling division `(n + 255) / 256` to ensure complete vector coverage.
4. **Boundary Guard `if (i < n)`:** Protects against illegal memory accesses when $n$ is not a multiple of 256.

---

## Question 4.a [04 Marks]
> **Write and explain the strip-mined version of the DAXPY loop in C.**

### 1. Purpose of Strip Mining
In vector architectures, hardware vector registers have a fixed physical capacity known as the **Maximum Vector Length (`MVL`)** (typically 64 elements in architectures like VMIPS). When operating on arrays with an arbitrary length $N$, where $N > \text{MVL}$, the compiler must decompose the loop into chunks of size at most `MVL`. This compiler optimization is called **strip mining**.

### 2. C Implementation:

```c
#define MVL 64 // Maximum hardware vector length

void daxpy_strip_mined(int n, double a, double *X, double *Y) {
    // 1. Calculate the remainder for the first chunk
    int vl = n % MVL; 
    int i = 0;

    // 2. Loop through all array elements in chunks <= MVL
    while (i < n) {
        // First iteration processes (n % MVL). If n is a multiple of MVL, process MVL.
        int current_len = (vl != 0) ? vl : MVL;
        vl = 0; // Subsequent iterations always process full MVL chunks

        // Vectorized chunk executed using hardware vector instructions
        for (int j = 0; j < current_len; j++) {
            Y[i + j] = a * X[i + j] + Y[i + j];
        }

        i += current_len; // Advance to next chunk
    }
}
```

### 3. Explanation:
* **Handling Arbitrary Loop Bounds:**
  * **First Iteration:** Handles the odd remainder elements ($N \pmod{\text{MVL}}$). For instance, if $N = 140$ and $\text{MVL} = 64$, the first chunk processes $140 \pmod{64} = 12$ elements.
  * **Subsequent Iterations:** Each iteration processes a full block of $\text{MVL} = 64$ elements ($2$ iterations of $64 = 128$ elements).
  * **Total:** $12 + 64 + 64 = 140$ elements.

---

## Question 4.b [06 Marks]
> **Why do most vector processors use memory banks?**  
> **Cray T90 has 32 processors, each capable of generating 4 loads and 2 stores per clock cycle. The processor clock cycle is 2.167 ns, while the cycle time of the SRAMs used in the memory system is 15 ns. Calculate the minimum number of memory banks required to allow all processors to run at full memory bandwidth.**

---

### Part 1: Why Most Vector Processors Use Memory Banks [02 Marks]
1. **High Memory Bandwidth Demand:** Vector processors execute operations on long data streams, requiring multiple memory words every single clock cycle.
2. **Latency vs. Clock Discrepancy:** The cycle time of a memory chip ($t_{\text{SRAM}}$) is substantially longer than the processor clock cycle ($t_{\text{CPU}}$). Accessing a single memory chip makes it busy for several CPU cycles.
3. **Pipelining Consecutive Accesses:** By interleaving addresses across independent memory banks:
   * Consecutive elements ($A, A+1, A+2, \dots$) or strided accesses map to different banks.
   * Multiple banks cycle concurrently in an overlapped fashion, delivering 1 memory word per clock cycle per processor without stalling.

---

### Part 2: Cray T90 Memory Bank Calculation [04 Marks]

#### Given Parameters:
* Number of processors, $P = 32$
* Memory operations per processor per clock cycle $= 4\text{ loads} + 2\text{ stores} = \mathbf{6\text{ references/cycle}}$
* Processor clock cycle time, $t_{\text{CPU}} = 2.167\text{ ns}$
* SRAM memory cycle time, $t_{\text{SRAM}} = 15\text{ ns}$

---

#### Step-by-Step Numerical Solution:

##### Step 1: Calculate Total Memory References Per Clock Cycle
$$\text{Total References/Cycle} = P \times (\text{References per Processor})$$
$$\text{Total References/Cycle} = 32 \times 6 = \mathbf{192\text{ memory references per clock cycle}}$$

##### Step 2: Calculate Bank Busy Time in Processor Clock Cycles
When a bank is accessed, it remains unavailable for $15\text{ ns}$. In terms of CPU clock cycles:
$$\text{Bank Busy Cycles} = \left\lceil \frac{t_{\text{SRAM}}}{t_{\text{CPU}}} \right\rceil = \left\lceil \frac{15\text{ ns}}{2.167\text{ ns}} \right\rceil = \lceil 6.922 \rceil = \mathbf{7\text{ clock cycles}}$$

##### Step 3: Calculate Minimum Number of Memory Banks
To prevent any processor from stalling due to bank busy conflicts:
$$\text{Minimum Banks} = (\text{Total References Per Clock Cycle}) \times (\text{Bank Busy Cycles})$$
$$\text{Minimum Banks} = 192 \times 7 = \mathbf{1344\text{ memory banks}}$$

##### Step 4: Hardware Architectural Realization (Power-of-2 Sizing)
Because memory banks are selected using hardware address bit decoding, the actual physical bank count must be an integer power of 2 ($2^k$):
$$2^{10} = 1024 < 1344 \le \mathbf{2048} = 2^{11}$$
* **Theoretical Minimum:** **$1344$ banks**
* **Practical Hardware Minimum (Power of 2):** **$2048$ banks** ($2^{11}$)

---

## Question 5.a [04 Marks]
> **Discuss the memory structures of an NVIDIA GPU.**

### Memory Hierarchy Overview:

```
┌───────────────────────────────────────────────────────────┐
│                       THREAD                              │
│   Registers (Private, 1-cycle) │ Local Memory (Private)   │
└─────────────────────────────┬─────────────────────────────┘
                              │
┌─────────────────────────────▼─────────────────────────────┐
│                    THREAD BLOCK (SM)                      │
│            Shared Memory (Scratchpad, ~few cycles)        │
└─────────────────────────────┬─────────────────────────────┘
                              │
┌─────────────────────────────▼─────────────────────────────┐
│                      DEVICE (GRID)                        │
│   Global Memory  │  Constant Memory  │  Texture Memory    │
│   (Off-chip DRAM)│  (Read-only, 64KB)│  (Spatial 2D cache)│
└───────────────────────────────────────────────────────────┘
```

### Detailed Breakdown:

1. **Registers (On-Chip):**
   * **Scope & Lifetime:** Private to each individual thread; lives for the duration of the thread.
   * **Speed:** Fastest memory on the GPU (~1 cycle latency). Allocated from a large physical register file per SM.
2. **Local Memory (Off-Chip DRAM):**
   * **Scope & Lifetime:** Private to each thread.
   * **Speed & Role:** High latency (DRAM), cached in L1/L2. Used for **register spilling** when a thread exceeds its register allocation, or for locally declared arrays with dynamic indexing.
3. **Shared Memory (On-Chip SRAM):**
   * **Scope & Lifetime:** Shared across all threads within the **same Thread Block** on an SM.
   * **Speed & Feature:** High bandwidth, low latency (~few cycles). Organized into 32 banks (subject to bank conflicts). Used as a user-managed scratchpad cache for inter-thread communication.
4. **Constant Memory (Off-Chip DRAM with On-Chip Cache):**
   * **Scope:** 64 KB total size, read-only for device, writable by host.
   * **Feature:** Features a dedicated constant cache. When all 32 threads of a warp read the **same address**, the value is broadcast in a single cycle.
5. **Texture and Surface Memory (Off-Chip with On-Chip Cache):**
   * **Feature:** Optimized for **2D/3D spatial locality** (image/graphics data). Provides hardware boundary addressing (clamping, wrapping) and bilinear filtering.
6. **Global Memory (Off-Chip Device DRAM):**
   * **Scope:** Accessible by all threads across all blocks and by the host CPU.
   * **Characteristics:** Large capacity (8 GB – 80+ GB), high latency (200–400 cycles). Requires **coalesced memory access** across threads in a warp to maximize throughput.

---

## Question 5.b [06 Marks]
> **Explain kernel invocation and execution model of CUDA.**

---

### 1. Kernel Invocation Model
A CUDA kernel is a function declared with `__global__` that executes on the GPU device across thousands of concurrent threads.

#### Invocation Syntax:
```cuda
kernelName<<<gridDim, blockDim, sharedMemBytes, stream>>>(args...);
```
* **`gridDim`:** Number of thread blocks in the grid (can be 1D, 2D, or 3D via `dim3`).
* **`blockDim`:** Number of threads per block (up to 1024 threads per block).
* **`sharedMemBytes`:** Optional dynamically allocated shared memory size in bytes.
* **`stream`:** Optional CUDA stream identifier for concurrent kernel execution.

#### Coordinate Mapping:
Each thread determines its unique global coordinates via built-in structs:
$$\text{1D Global ID} = \text{blockIdx.x} \times \text{blockDim.x} + \text{threadIdx.x}$$
$$\text{2D Matrix Coordinates: } \text{Row} = \text{blockIdx.y} \times \text{blockDim.y} + \text{threadIdx.y}, \quad \text{Col} = \text{blockIdx.x} \times \text{blockDim.x} + \text{threadIdx.x}$$

---

### 2. CUDA Execution Model (Hardware Mapping)

The CUDA software abstractions map directly onto GPU physical hardware:

```
SOFTWARE ABSTRACTION                            HARDWARE UNIT
────────────────────                            ─────────────
Grid (Entire Kernel Launch)          ─────►     GPU Device (Multi-SMs)
         │                                               │
Thread Block (up to 1024 Threads)   ─────►     Streaming Multiprocessor (SM)
         │                                               │
Warp (32 Consecutive Threads)        ─────►     Warp Scheduler / SIMD Unit
         │                                               │
Individual Thread                    ─────►     CUDA Core / Lane
```

#### Key Execution Mechanisms:
1. **Block Assignment:**
   * Thread blocks are dynamically distributed to available Streaming Multiprocessors (SMs).
   * Once scheduled, a block executes entirely on that assigned SM until completion.
2. **Warp Partitioning & SIMT:**
   * Each thread block is divided into **warps of 32 threads**.
   * A warp executes a single instruction across all 32 threads simultaneously (**SIMT — Single Instruction, Multiple Threads**).
3. **Zero-Overhead Context Switching (Latency Hiding):**
   * Unlike CPUs that rely on complex out-of-order execution logic, GPUs hide long memory delays (200–400 cycles) using **massive thread-level parallelism**.
   * When one warp stalls waiting for memory, the hardware Warp Scheduler instantly switches to another ready warp in **0 clock cycles**.
4. **Intra-Block Barrier Synchronization:**
   * Threads within the same block coordinate using `__syncthreads()`.
   * No thread can advance past the barrier until all threads in that block reach it.

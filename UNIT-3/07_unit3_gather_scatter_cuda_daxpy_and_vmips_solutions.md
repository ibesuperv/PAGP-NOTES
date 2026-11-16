# Unit – III: Gather-Scatter, CUDA DAXPY, VMIPS Architecture & SIMD Thread Scheduling
## Exam Question Bank Solutions (Part 7)

**Syllabus Mapping:**
* **Vector Architecture:** VMIPS Components & ISA features, Linpack DAXPY in MIPS vs. VMIPS, Convoys and Chimes, Gather-Scatter for Sparse Matrices.
* **GPU Programming with CUDA & OpenCL:** CUDA Kernel snippet for DAXPY (`<<<blocks, threads>>>`), SIMD / Warp thread scheduling, OpenCL NDRange execution model.

---

# SECTION A: SHORT QUESTIONS (Unit-3 Only)

---

### Question 1.6 [02 Marks]
> **Mention any two primary components of the instruction set architecture of VMIPS.**

#### Answer:
*(Any two of the following can be stated):*
1. **Vector Registers:** 8 vector registers ($V0$ through $V7$), each storing 64 elements of 64-bit double-precision floating-point or integer values.
2. **Vector Control Registers:** 
   * **`VLR` (Vector Length Register):** Dynamically dictates the active vector length ($1 \le \text{length} \le 64$) for arbitrary loop counts.
   * **`VMR` (Vector Mask Register):** A 64-bit boolean mask register used for conditional (`if-else`) vector execution.
3. **Pipelined Vector Instructions:** Vector-vector (`ADDVV.D`), vector-scalar (`MULVS.D`), and stride/gather-scatter memory instructions (`LVWS`, `LVI`, `SVI`).

---

### Question 1.7 [02 Marks]
> **Define Convoy. Identify the number of convoys required for the following code sequence:**
> ```assembly
> LV      V1, Rx
> MULVS.D V2, V1, F0
> LV      V3, Ry
> ADDVV.D V4, V2, V3
> SV      V4, Ry
> ```

#### Answer:
* **Definition:** A **convoy** is a set of vector instructions that can begin execution together in the same clock period without causing structural hazards (i.e., they use different functional units or chaining).

#### Number of Convoys Required:
Assuming a standard VMIPS architecture with **one vector memory port** and **chaining**:
* **Convoy 1:** `LV V1, Rx` and `MULVS.D V2, V1, F0` *(Chained together)*
* **Convoy 2:** `LV V3, Ry` and `ADDVV.D V4, V2, V3` *(Chained together; separated from Convoy 1 because it requires the single memory unit for a new load)*
* **Convoy 3:** `SV V4, Ry` *(Requires the single memory unit for store)*

**Total Number of Convoys = 3 Convoys (3 Chimes).**

---

### Question 1.11 [02 Marks]
> **In OpenCL, the kernel is launched with a two-dimensional `___________` containing all work-items.**

#### Answer:
**`NDRange` (or index space / global work-space)**.

---

### Question 1.12 [02 Marks]
> **In OpenCL, the kernel calls the API functions `___________` & `___________` to fetch the total number of work-items in the x- and y-dimensions of all work-items.**

#### Answer:
**`get_global_size(0)`** and **`get_global_size(1)`**.

---

# SECTION B: LONG QUESTIONS (04, 05, 07 & 08 Marks)

---

## Question 5.a [08 Marks]
> **Draw the basic structure of vector architecture for VMIPS and explain its components.**

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

### Component Details:
1. **Vector Register File:**
   * Contains **8 vector registers ($V0 - V7$)**.
   * Each register holds **64 elements**, each being a **64-bit double-precision** word.
   * Multi-ported to support simultaneous reads and writes by arithmetic pipelines and memory interfaces.
2. **Pipelined Vector Functional Units:**
   * Specialized units for Vector FP Add/Sub, Vector FP Mult, Vector FP Div, and Vector Integer/Logic.
   * Fully pipelined: can accept a new pair of operands and produce a new result on every clock cycle.
3. **Vector Control Registers:**
   * **`VLR` (Vector Length Register):** Configures active vector length ($1 \le \text{length} \le 64$) for loops with non-multiple-of-64 bounds.
   * **`VMR` (Vector Mask Register):** 64-bit register for conditional masking in `if-else` loops.
4. **Vector Load / Store Unit:**
   * Pipelined memory interface moving data at a sustained rate of 1 word per clock cycle after initial latency.
   * Handles unit stride, non-unit stride (`LVWS`/`SVWS`), and gather/scatter (`LVI`/`SVI`).
5. **Multi-Bank Interleaved Memory System:**
   * Provides massive sustained memory bandwidth by interleaving addresses across independent memory banks to eliminate bank conflicts.

---

## Question 5.b [08 Marks]
> **Let $X$ and $Y$ be vectors, initially resident in memory, and $a$ is a scalar. The following instruction forms the inner loop of the Linpack benchmark: $Y = a \times X + Y$.**  
> **Assume that the starting addresses of $X$ and $Y$ are in $Rx$ and $Ry$, respectively. Show the code for MIPS and VMIPS for the given loop.**

---

### 1. Scalar MIPS Assembly Code (Iterates 64 Times)
Assume scalar $a$ is in $F0$, $Rx$ and $Ry$ are base addresses of arrays $X$ and $Y$, and $R1$ contains the terminal address (`Rx + 512` bytes):

```assembly
# Scalar MIPS DAXPY Loop (Processes 1 element per iteration)
Loop:
    L.D    F2, 0(Rx)        # Load X[i] from memory into F2
    MUL.D  F2, F2, F0       # Compute a * X[i]
    L.D    F4, 0(Ry)        # Load Y[i] from memory into F4
    ADD.D  F4, F4, F2       # Compute a * X[i] + Y[i]
    S.D    F4, 0(Ry)        # Store result back into Y[i]
    DADDIU Rx, Rx, #8       # Increment X pointer by 8 bytes (double-precision)
    DADDIU Ry, Ry, #8       # Increment Y pointer by 8 bytes
    DSUBU  R2, R1, Rx       # Check boundary: R2 = R1 - Rx
    BNEZ   R2, Loop         # Branch back to Loop if R2 != 0
```
* **Dynamic Instruction Count:** $64 \times 9 = \mathbf{576\text{ instructions}}$ executed.

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
* **Dynamic Instruction Count:** Exactly **$5\text{ vector instructions}$**!
* **Performance Gain:** **$115\times$ reduction** in dynamic instruction count, complete elimination of branch overhead and stalls.

---

## Question 6.a [05 Marks]
> **Describe gather-scatter operations using index vectors for handling sparse matrices in vector architectures.**

### 1. The Sparse Matrix Problem:
In scientific computing, sparse matrices contain mostly zeros. To save memory and computation, sparse matrices are stored in compressed formats (e.g., Compressed Sparse Row — CSR):
* Non-zero values are stored in a dense data array `A[]`.
* The column/row indices of non-zero elements are stored in an integer index array `K[]`.

In high-level C, a sparse vector operation appears as:
```c
for (i = 0; i < n; i++) {
    A[K[i]] = A[K[i]] + C[M[i]]; // Non-contiguous, indirect indexing!
}
```
Standard unit-stride and non-unit stride vector loads **cannot** handle this because the distance between successive elements is irregular.

---

### 2. Gather Operation (`LVI` / Load Vector Indexed)
* **Mechanism:** Uses a vector register containing index offsets ($Vk$) to fetch data from non-contiguous, disparate memory locations into a contiguous vector destination register ($Vi$):
  $$\text{VectorDest}[i] = \text{Memory}[\text{BaseAddress} + \text{IndexVector}[i]]$$
* **VMIPS Instruction:**
  ```assembly
  LV  V_idx, Rk         # Load index offsets into vector register V_idx
  LVI V_dest, Ra, V_idx # Gather elements from memory at Ra + V_idx into V_dest
  ```

---

### 3. Scatter Operation (`SVI` / Store Vector Indexed)
* **Mechanism:** Takes consecutive elements from a vector source register ($Vi$) and writes them out to non-contiguous, disparate memory addresses specified by an index vector ($Vk$):
  $$\text{Memory}[\text{BaseAddress} + \text{IndexVector}[i]] = \text{VectorSource}[i]$$
* **VMIPS Instruction:**
  ```assembly
  SVI V_src, Ra, V_idx  # Scatter elements of V_src into memory at Ra + V_idx
  ```

---

### 4. Architectural Summary Diagram:

```
Index Vector K[]:   [ 2 | 8 | 15 | 29 | ... ]
                          │   │    │    │
                          ▼   ▼    ▼    ▼
Memory Array A[]:  ... |A[2]|...|A[8]|...|A[15]|...|A[29]| ...
                          │   │    │    │
                  Gather: │   │    │    │ (LVI)
                          ▼   ▼    ▼    ▼
Vector Register V1: [ A[2] | A[8] | A[15] | A[29] ]  <-- Dense Contiguous Register!
```

---

## Question 6.b [04 Marks]
> **Write CUDA code snippet for the DAXPY loop ($Y = a \times X + Y$), to launch $n$ threads, one per vector element, with 256 CUDA Threads per thread block in a multithreaded SIMD Processor.**

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

### Explanation of Constraints Met:
1. **One Thread per Vector Element:** Each thread computes exactly one index: `int i = blockDim.x * blockIdx.x + threadIdx.x`.
2. **256 Threads per Block:** `threadsPerBlock = 256`.
3. **Arbitrary Length $n$ Coverage:** Ceiling division `(n + 255) / 256` ensures that even if $n$ is not a multiple of 256, all elements are processed safely using the `if (i < n)` guard.

---

## Question 6.c [07 Marks]
> **With a neat diagram explain the process of scheduling threads of SIMD instructions.**

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

### Detailed Scheduling Process:
1. **Thread Block Mapping to SMs:**
   * Software thread blocks are assigned to physical Streaming Multiprocessors (SMs).
   * Multiple thread blocks can reside concurrently on an SM depending on hardware register and shared memory capacity.
2. **Warp Partitioning (SIMD Grouping):**
   * Each thread block is divided into **warps of 32 consecutive threads**.
   * A warp is the smallest schedulable entity in hardware.
3. **Scoreboarding & Ready Warp Selection:**
   * Dedicated hardware Warp Schedulers continuously evaluate the readiness of all active resident warps.
   * A warp is eligible to execute when its source operands are ready in registers and the required execution pipeline is available.
4. **Zero-Overhead Context Switching (Latency Hiding):**
   * When an active warp executes a high-latency instruction (e.g., global memory load taking 200–400 cycles), the scheduler deselects it and switches to another ready warp in **0 clock cycles**.
   * Because thread register states remain resident in the massive physical register file, no context saving or restoring is required.
5. **Handling Divergent SIMD Threads:**
   * When threads in a warp branch differently at an `if-else` block, the scheduler serializes the paths using execution masks:
     1. Executes the `then` path with false-condition threads disabled.
     2. Executes the `else` path with true-condition threads disabled.
     3. Reconverges the warp to full lock-step execution.


# 02: Classes of Parallelism and Flynn's Taxonomy

---

## 1. Classes of Parallelism

Parallelism is the primary driving force behind modern computer design, with energy and cost being the main constraints.

### 1.1 Two Types of Parallelism in Applications (Software Level)
In software applications, parallelism appears in two primary ways:

1. **Data-Level Parallelism (DLP):**
   * **Definition:** Arises because there are many data items that can be operated on at the same time.
   * **Example:** Adding two 1,000-element arrays together (`C[i] = A[i] + B[i]`). Every addition is independent and can happen simultaneously.

2. **Task-Level Parallelism (TLP):**
   * **Definition:** Arises because independent tasks or chunks of work are created that can execute concurrently.
   * **Example:** A video game where one task runs physics simulation, a second task renders graphics, and a third task plays audio.

---

### 1.2 Four Ways Hardware Exploits Application Parallelism
Hardware designs use four major techniques to take advantage of DLP and TLP:

1. **Instruction-Level Parallelism (ILP):**
   * Exploits data-level parallelism at modest to medium levels.
   * Uses compiler support and hardware techniques like **pipelining** and **speculative execution** (out-of-order execution) inside a single processor core.
2. **Vector Architectures, GPUs, and SIMD:**
   * Exploits data-level parallelism directly.
   * Applies a **single instruction** to a large collection of data items simultaneously in parallel.
3. **Thread-Level Parallelism (TLP):**
   * Exploits either data-level parallelism or task-level parallelism.
   * Uses a tightly coupled hardware model (multicore CPUs) where multiple parallel threads interact, share memory, and synchronize.
4. **Request-Level Parallelism (RLP):**
   * Exploits parallelism among largely decoupled, independent tasks.
   * Handled by operating systems and distributed networks across independent machines (e.g., handling millions of web search queries across a data center).

---

## 2. Flynn’s Computer Taxonomy

In 1966, Michael Flynn introduced a taxonomy to classify computers based on the number of concurrent **Instruction Streams** and **Data Streams**.

```
                           DATA STREAMS
                      Single           Multiple
                 +---------------+-----------------+
          Single |     SISD      |      SIMD       |
INSTRUCTION      | (Uniprocessor)| (Vector / GPU)  |
STREAM           +---------------+-----------------+
        Multiple |     MISD      |      MIMD       |
                 |  (No commercial  | (Multicore /    |
                 |     systems)  |  Clusters / WSC)|
                 +---------------+-----------------+
```

---

### 2.1 SISD (Single Instruction Stream, Single Data Stream)
* **Definition:** The standard sequential uniprocessor computer.
* **How it operates:** One processor fetches one instruction at a time and operates on one data value at a time.
* **Parallelism:** While the programming model is strictly sequential, the hardware can still exploit **Instruction-Level Parallelism (ILP)** internally through pipelining, superscalar execution, and speculation.

---

### 2.2 SIMD (Single Instruction Stream, Multiple Data Streams)
* **Definition:** A parallel architecture that exploits **Data-Level Parallelism (DLP)**.
* **How it operates:**
  * Uses **one central control processor** and **one instruction memory**.
  * The control processor fetches and issues a **single instruction**.
  * Multiple execution units execute that identical instruction at the same time, but each unit operates on **different data items**.
  * Each execution unit has its own private data memory.
* **Architectural Examples:**
  * **Vector Architectures:** Pipelined execution of operations on entire arrays/vectors.
  * **Multimedia Extensions:** SIMD instructions inside standard CPUs (e.g., Intel AVX, ARM NEON).
  * **Graphics Processing Units (GPUs):** Massive arrays of SIMD lanes executing shader/compute kernels.

---

### 2.3 MISD (Multiple Instruction Streams, Single Data Stream)
* **Definition:** Multiple processors execute different instructions concurrently on the exact same single stream of data.
* **Commercial Status:** **No commercial multiprocessor of this type has ever been built.** It exists in the taxonomy to complete the theoretical $2 \times 2$ grid.

---

### 2.4 MIMD (Multiple Instruction Streams, Multiple Data Streams)
* **Definition:** Multiple processors where each processor fetches its own instructions and operates on its own data.
* **Target Parallelism:** Directly targets **Task-Level Parallelism (TLP)**.
* **Characteristics:**
  * **More Flexible:** Can run completely different tasks or programs on each processor.
  * **More Expensive:** Each core requires its own instruction fetch unit, decode logic, and cache.
  * **Grain Size:** Because communication between cores has overhead, the grain size of work must be large enough to make parallel execution worthwhile.
* **Two Sub-Classes of MIMD Architectures:**
  1. **Tightly Coupled MIMD:**
     * Processors share a common memory space and communicate directly with low latency.
     * Exploits **Thread-Level Parallelism (TLP)** through cooperating parallel threads (e.g., standard multi-core processors like Intel Core i7, AMD Ryzen).
  2. **Loosely Coupled MIMD:**
     * Multiple independent nodes connected via high-speed local networks (clusters and Warehouse-Scale Computers).
     * Exploits **Request-Level Parallelism (RLP)**, where tasks run independently with little need for inter-processor communication or synchronization.

---

### 2.5 Modern Processors as Hybrids
Flynn's classification is a coarse model. Most modern parallel computers are **hybrids** combining multiple categories:
* A modern multi-core CPU is a **MIMD** system at the chip level (multiple independent cores).
* Each individual core is an **SISD** machine that pipelines instructions to exploit **ILP**.
* Each core also contains **SIMD** vector units (like AVX-512) to accelerate data-parallel math.

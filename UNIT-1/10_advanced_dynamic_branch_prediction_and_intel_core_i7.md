# Unit 1 — Part 10: Advanced Dynamic Branch Prediction and Intel Core i7 Case Study

---

## 1. Motivation for Advanced Dynamic Branch Prediction

In modern high-performance microprocessors, branch prediction accuracy is one of the most critical determinants of overall performance:
* **Deep Pipelines:** High-end processors feature deep pipelines (14 to 20+ pipeline stages). When a branch mispredicts, all instructions in the pipeline behind it must be flushed.
* **Multiple Issue (Superscalar Execution):** Processors issue 4 to 8 instructions per clock cycle. At any given moment, there may be **up to 256 instructions in-flight** (and typically at least 30) across reorder buffers and execution queues.
* **Energy Cost of Misspeculation:** Fetching, decoding, and executing instructions down a mispredicted path wastes substantial dynamic power and execution resources.

A simple 1-bit or 2-bit Branch History Table (BHT) tracks each branch strictly in isolation based only on its own local address. While effective for simple loops, it completely misses correlations between different branches.

---

## 2. Correlating Branch Predictors (Two-Level Predictors)

### 2.1 The Concept of Branch Correlation
A **correlating predictor** (also called a **two-level predictor**) makes a prediction by considering both the recent history of the current branch **and the recent outcomes of other preceding branches**.

#### Real-World Motivating Example (`eqntott` Benchmark)
Consider the following code snippet from the classic `eqntott` SPEC benchmark:
```c
if (aa == 2)
    aa = 0;
if (bb == 2)
    bb = 0;
if (aa != bb) {
    ...
}
```

Translated into RISC-V assembly language (where registers `x1 = aa` and `x2 = bb`):
```riscv
      addi x3, x1, -2
      bnez x3, L1          # Branch b1: checks if (aa != 2)
      add  x1, x0, x0      # aa = 0 (executes if b1 NOT taken)

L1:   addi x3, x2, -2
      bnez x3, L2          # Branch b2: checks if (bb != 2)
      add  x2, x0, x0      # bb = 0 (executes if b2 NOT taken)

L2:   sub  x3, x1, x2      # x3 = aa - bb
      beqz x3, L3          # Branch b3: checks if (aa == bb)
```

#### Why Correlation Matters:
* Let the three branches be labeled $b1$, $b2$, and $b3$.
* Notice the logical relationship: If branch $b1$ is **not taken** (`aa == 2` $\rightarrow$ `aa` becomes 0) and branch $b2$ is **not taken** (`bb == 2` $\rightarrow$ `bb` becomes 0), then both variables equal 0 (`aa == bb`).
* Therefore, **branch $b3$ is guaranteed to be taken!**
* A traditional predictor tracking only branch $b3$ in isolation has no knowledge of what happened at $b1$ and $b2$.
* A **correlating predictor** records the outcomes of $b1$ and $b2$, enabling it to predict $b3$ with 100% accuracy in this scenario.

---

### 2.2 Mathematical Formulation of $(m, n)$ Predictors
A correlating predictor that uses the behavior of the last $m$ branches to choose from $2^m$ predictor tables, each containing $n$-bit counters, is classified as an **$(m, n)$ predictor**:
* **$m$ (Global History Length):** The number of most recent branch outcomes tracked across the entire program. Stored in an **$m$-bit shift register** (Branch History Register - BHR), where each bit represents Taken (1) or Not Taken (0).
* **$n$ (Counter Size):** The number of bits in each saturating counter (typically 2-bit saturating counters).
* **Standard 2-Bit Predictor:** A predictor with zero global history bits is simply a **$(0, 2)$ predictor**.

```
           Global Branch History (m-bit shift register)
                     [ T | NT | T | ... | T ]
                                │
                                ▼
            Selects 1 of 2^m Prediction Tables
      ┌───────────┬───────────┬───────────┬───────────┐
      │  Table 0  │  Table 1  │  Table 2  │Table 2^m-1│
      └─────┬─────┴───────────┴───────────┴───────────┘
            │
            ▼
      Branch PC (low-order address bits) indexes 
      the specific n-bit saturating counter
```

---

### 2.3 Hardware Sizing Formula and Worked Example

The total number of storage bits required by an $(m, n)$ predictor is:
$$\text{Total Bits} = 2^m \times n \times (\text{Number of branch-selected entries})$$

#### Textbook Worked Example
> **Question:** 
> 1. How many bits are in a $(0, 2)$ branch predictor with 4K entries?
> 2. How many branch-selected entries can be accommodated in a $(2, 2)$ predictor that has the same total storage of 8K bits?

#### Step-by-Step Solution:
1. **For the $(0, 2)$ predictor:**
   $$\text{Total Bits} = 2^0 \times 2 \times 4096 = 1 \times 2 \times 4096 = \mathbf{8192\text{ bits (8K bits)}}$$

2. **For the $(2, 2)$ predictor with 8K total bits:**
   $$\text{Total Bits} = 2^2 \times 2 \times (\text{Entries}) = 8192$$
   $$4 \times 2 \times (\text{Entries}) = 8192$$
   $$8 \times (\text{Entries}) = 8192$$
   $$\text{Entries} = \frac{8192}{8} = \mathbf{1024\text{ entries (1K entries)}}$$

#### Performance Comparison:
* Although the $(2, 2)$ predictor has only 1K branch-selected entries compared to 4K entries in the $(0, 2)$ predictor, **the $(2, 2)$ predictor achieves a significantly lower misprediction rate**.
* In fact, a $(2, 2)$ predictor often outperforms a simple 2-bit predictor with an *unlimited* number of entries because global correlation eliminates systemic prediction errors that table size alone cannot fix.

---

## 3. The gshare Predictor (Scott McFarling)

A potential inefficiency of directly concatenating branch PC bits with global history bits is that many table entries may remain unused or cause collisions. 

Scott McFarling introduced the **gshare predictor**, which provides a highly efficient hashing mechanism:

```
        Branch PC (e.g., 10 bits)
                  │
                  ▼
              ┌───────┐
              │  XOR  │ ◄─── Global Branch History (10-bit shift register)
              └───┬───┘
                  │ (10-bit hashed index)
                  ▼
      ┌───────────────────────┐
      │  1024-entry Table of  │
      │ 2-bit Saturating Ctrs │
      └───────────────────────┘
                  │
                  ▼
             Prediction
         (Taken / Not Taken)
```

### Key Principles of gshare:
1. **Bitwise XOR Hashing:** An Exclusive-OR (XOR) operation combines the branch PC with the global branch history register.
2. **Uniform Distribution:** The XOR function acts as a pseudo-random hash, spreading branches and their varying execution contexts uniformly across the entire prediction table.
3. **Hardware Efficiency:** Achieves the accuracy of larger two-level predictors with minimal hardware overhead.
4. **Classification:** It is classified as an **alloyed / hybrid predictor** because it fuses local branch identity with global execution history into a single indexing key.

---

## 4. Tournament Predictors: Adaptively Combining Local and Global Predictors

### 4.1 Motivation
In complex programs, different branches exhibit fundamentally different behavior:
* **Integer Benchmarks (e.g., SPECint):** Control flow is highly dependent on input data and prior branch decisions. About **40% of branches** predict better using **global history**.
* **Floating-Point Benchmarks (e.g., SPECfp):** Regular mathematical loops dominate. Less than **15% of branches** benefit from global history; over **85% prefer local history**.

A **Tournament Predictor** (first commercialized in the DEC Alpha 21264 and later AMD processors) dynamically selects between two or more predictors on a per-branch basis using an adaptive selector.

---

### 4.2 Tournament Predictor Architecture

```
                       Branch Address (PC)
                                │
         ┌──────────────────────┼──────────────────────┐
         ▼                      ▼                      ▼
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│ Global Predictor │  │  Selector Table  │  │ Local Predictor  │
│ (Global History) │  │(2-bit Sat. Ctrs) │  │   (Branch PC)    │
└────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘
         │                     │                     │
  Global Prediction            │               Local Prediction
         │                     │                     │
         └─────────────► ┌─────▼─────┐ ◄─────────────┘
                         │ Mux / Sel │
                         └─────┬─────┘
                               ▼
                       Final Prediction
```

### 4.3 Selector Logic and State Machine
* **Selector Table:** An array of 2-bit saturating counters indexed by the branch address.
* **Interpretation of 2-bit Selector Counter:**
  * `00`: Strongly prefer Global Predictor
  * `01`: Weakly prefer Global Predictor
  * `10`: Weakly prefer Local Predictor
  * `11`: Strongly prefer Local Predictor
* **Two-Misprediction Rule:** Just like standard 2-bit branch counters, the selector requires **two consecutive mispredictions** by the currently chosen predictor before flipping its preference to the other predictor.

---

### 4.4 Example: The DEC Alpha 21264 Tournament Predictor
The DEC Alpha 21264 implemented a 29-kilobit tournament predictor consisting of:
1. **Local Predictor (Two-Level Structure, 29K bits total with selector):**
   * **Level 1:** A 1024-entry table of 10-bit local history shift registers (records the last 10 outcomes of that specific branch).
   * **Level 2:** A 1024-entry table of **3-bit saturating counters**, indexed by the 10-bit local history output.
2. **Global Predictor:**
   * A 4096-entry table of 2-bit saturating counters indexed by a 12-bit global history register.
3. **Selector Table:**
   * A 4096-entry table of 2-bit saturating counters indexed by the 12-bit global branch history.

---

## 5. Tagged Hybrid Predictors (TAGE Predictors)

**Tagged Hybrid Predictors** (specifically **TAGE — TAgged GEometric** predictors, designed by André Seznec and Pierre Michaud) represent the state-of-the-art in dynamic branch prediction. They have consistently won international branch prediction competitions.

### 5.1 PPM Inspiration
TAGE is loosely inspired by **Prediction by Partial Matching (PPM)**, an algorithm used in statistical data compression that predicts upcoming symbols based on variable-length historical context.

---

### 5.2 TAGE Architecture and Organization
A typical TAGE predictor consists of a **base predictor $P(0)$** and a bank of tagged prediction tables $P(1)$ to $P(n)$ (commonly 4 to 8 tagged tables):

```
                       Branch Address (PC) & Global History
                                         │
        ┌───────────────┬────────────────┼────────────────┬───────────────┐
        ▼               ▼                ▼                ▼               ▼
   ┌─────────┐    ┌───────────┐    ┌───────────┐    ┌───────────┐   ┌───────────┐
   │  P(0)   │    │   P(1)    │    │   P(2)    │    │   P(3)    │   │   P(4)    │
   │  Base   │    │ History   │    │ History   │    │ History   │   │ History   │
   │Predictor│    │ Length L1 │    │ Length L2 │    │ Length L3 │   │ Length L4 │
   │ (No Tag)│    │ (e.g., 4) │    │ (e.g., 16)│    │(e.g., 64) │   │(e.g., 256)│
   └────┬────┘    └─────┬─────┘    └─────┬─────┘    └─────┬─────┘   └─────┬─────┘
        │            Tag Match?       Tag Match?       Tag Match?      Tag Match?
        │               │                │                │               │
        └───────────────┴────────────────┴────────────────┴───────────────┘
                                         │
                                         ▼
                             Priority Selection Logic:
                  Choose prediction with LONGEST MATCHING HISTORY.
                  If none match, use default Base Predictor P(0).
```

### 5.3 Key Structural Features of TAGE:
1. **Geometric History Lengths:**
   * The history lengths $L(i)$ used by tables $P(1) \dots P(n)$ grow geometrically (e.g., 4, 16, 64, 256 branches).
   * Short histories capture local/tight loop behaviors; extremely long histories capture complex, distant algorithmic correlations.
2. **Tagged Entries:**
   * Tables $P(1)$ through $P(n)$ store short tags (**4 to 8 bits**).
   * A table entry is only valid for a branch if its stored tag matches the hash of $(PC \oplus \text{History})$.
   * Tags do not require 100% full-address matching; 4–8 bits provide near-optimal accuracy while minimizing silicon area.
3. **Base Predictor $P(0)$:**
   * $P(0)$ contains no tags and is indexed by PC only. It serves as the fallback/default prediction when no tagged table matches.
4. **2-Bit Use Field:**
   * Each entry in the tagged tables includes a 2-bit **use field** that records how recently and successfully that entry was used.
   * Stale or rarely used entries are periodically cleared or overwritten by new branches.
5. **Prediction Selection Rule:**
   $$\text{Final Prediction} = \text{Prediction from } P(i) \text{ with the longest history length } L(i) \text{ whose tag matches}.$$

---

### 5.4 Predictor Initialization and Cold Start Handling
When a large predictor is empty at program startup, random initial values degrade early performance. Modern predictors employ specific initialization schemes:
* **Valid Bits:** Track whether an entry has been officially allocated or remains in an "unused state."
* **Branch Hint Bits:** Modern ISAs provide optional static hint bits set by the compiler indicating expected direction.
* **Directional Static Heuristic:** Forward-going conditional branches are initialized as **Not Taken**; backward-going branches (loops) are initialized as **Taken**.

---

### 5.5 Performance Comparison: TAGE vs. gshare (Figure 3.8)

Comparing predictors of identical total memory budgets across SPEC CPU, multimedia, and server benchmarks:
* **Multimedia Benchmarks:**
  * gshare misprediction rate: $2.7\% - 3.4\%$
  * TAGE misprediction rate: $1.8\% - 2.2\%$
  * **Result:** TAGE achieves roughly **one-third ($\sim 33\%$) fewer mispredictions**.
* **Integer and Server Workloads:** TAGE delivers substantial misprediction reductions because geometric histories capture complex correlated branch patterns that single-history predictors miss.

---

## 6. Evolution of the Intel Core i7 Branch Predictor

Intel’s Core i7 microarchitectures (from Nehalem in 2008 to Skylake in 2016) demonstrate the real-world evolution of dynamic branch prediction in commercial high-performance computing.

```
       Core i7 920 (Nehalem, 2008)               Core i7 6700 (Skylake, 2016)
 ┌──────────────────────────────────────┐   ┌─────────────────────────────────────┐
 │ • Two-level hierarchy (L1 + L2)      │   │ • Tagged Hybrid (TAGE-style)        │
 │ • Multilevel Tournament:             │   │ • Subsumes L1/L2 and loop exit into │
 │   1. Simple 2-bit predictor          │   │   geometric history tables          │
 │   2. Global history predictor        │   │ • Handles up to 256 in-flight instrs│
 │   3. Dedicated Loop Exit Predictor   │   │ • Separate Return Address Stack     │
 └──────────────────────────────────────┘   └─────────────────────────────────────┘
```

### 6.1 Intel Core i7 920 (Nehalem Microarchitecture, 2008)
* **Two-Level Predictor Hierarchy:**
  * **Level 1 (L1) Predictor:** Small and ultra-fast, designed to satisfy the strict single-cycle clock timing requirement of predicting a branch every cycle.
  * **Level 2 (L2) Predictor:** Larger, highly accurate backup predictor accessed when L1 requires guidance.
* **Tournament Composition (3 Component Predictors):**
  1. **Simple 2-bit Predictor:** Provides standard local predictions.
  2. **Global History Predictor:** Tracks correlating branch histories.
  3. **Loop Exit Predictor:** A dedicated hardware counter that detects loop branches and counts the exact number of iterations, predicting the final loop-exit branch with 100% accuracy.
* **Auxiliary Predictors:**
  * An **Indirect Branch Predictor** (predicts target addresses of computed jumps and function pointers).
  * A **Return Address Stack (RAS)** (predicts subroutine return addresses).

---

### 6.2 Intel Core i7 6700 (Skylake Microarchitecture, 2016)
* **Tagged Hybrid (TAGE-style) Design:**
  * Intel replaced separate heterogeneous components with a unified **tagged hybrid predictor**.
  * **Why?** The multiple geometric history lengths of a TAGE predictor inherently subsume the function of:
    1. The loop exit predictor (short geometric histories capture repetitive loop counts).
    2. Local history predictors.
    3. Global history predictors.
* **Speculation and Measurement:**
  * To accurately evaluate branch prediction in the presence of aggressive speculative execution, performance is measured as **mispredictions per 1000 successfully retired instructions** on SPEC CPU2006.
  * This eliminates misleading counts caused by instructions fetched along already misspeculated paths.

# 05: Quantitative Principles and the Processor Performance Equation

---

## 1. Core Principles of Computer Design

When designing computer systems, architects rely on four fundamental quantitative principles to maximize performance and efficiency:

### 1.1 Principle 1: Take Advantage of Parallelism
Parallelism is one of the most powerful tools to improve performance across all design layers:
* **System Level:** Distributing requests across multiple CPU cores and multiple storage drives (e.g., in web servers and database systems). Spreading data across disks provides **Data-Level Parallelism**, while handling client requests provides **Request-Level Parallelism** or **Thread-Level Parallelism**.
* **Processor Core Level:** Exploiting **Instruction-Level Parallelism (ILP)** through **pipelining** (overlapping the execution of successive instructions so that multiple instructions execute at different pipeline stages at the same time).
* **Digital Circuit Level:**
  * Searching memory banks in parallel in set-associative caches.
  * Using **carry-lookahead adders** in ALUs to compute binary sums in logarithmic time ($O(\log n)$) instead of slow linear ripple-carry time ($O(n)$).

---

### 1.2 Principle 2: Principle of Locality
Programs do not access memory randomly; they tend to reuse code and data they have accessed recently.
* **The 90/10 Rule of Thumb:** A program spends **90% of its execution time in only 10% of the code**.
* **Temporal Locality:** If a data or instruction item is accessed now, it is very likely to be accessed again soon (e.g., loop variables, subroutine code).
* **Spatial Locality:** Items stored at adjacent memory addresses are likely to be accessed close together in time (e.g., scanning an array sequentially, fetching consecutive instructions).

---

### 1.3 Principle 3: Focus on the Common Case
When making design trade-offs, **always favor the frequent case over the infrequent case**.
* Spend hardware resources where the program spends most of its time and energy.
* *Example in CPU Design:* Instruction fetch and decode units are used on every single instruction, whereas floating-point multipliers are used only occasionally. Optimize the fetch/decode unit first.
* *Example in Arithmetic:* Adding two integers rarely results in an arithmetic overflow. The circuit should be heavily optimized for the common case (no overflow), even if handling a rare overflow becomes slightly slower.
* *Example in Dependability:* If a database server has 50 hard drives for every 1 processor, hard drive failures will dominate overall system failures.

---

## 2. Quantifying the Common Case: Amdahl’s Law

Amdahl’s Law quantifies the performance improvement (speedup) gained by optimizing a specific part of a computer system.

> **Amdahl’s Law:**  
> The performance gain obtained from using a faster execution mode is strictly limited by the fraction of time that the faster mode can actually be used.

### 2.1 The Speedup Formula

$$\text{Speedup}_{\text{overall}} = \frac{\text{Execution Time}_{\text{old}}}{\text{Execution Time}_{\text{new}}} = \frac{1}{(1 - \text{Fraction}_{\text{enhanced}}) + \frac{\text{Fraction}_{\text{enhanced}}}{\text{Speedup}_{\text{enhanced}}}}$$

Where:
* $\text{Fraction}_{\text{enhanced}}$: The fraction of the original execution time that can use the improvement ($\le 1$).
* $\text{Speedup}_{\text{enhanced}}$: How much faster that specific enhanced portion runs ($> 1$).
* $(1 - \text{Fraction}_{\text{enhanced}})$: The unenhanced fraction that still runs at the original speed.

#### Key Takeaway (Law of Diminishing Returns):
As you make one part of a system faster, the unoptimized remainder becomes the main bottleneck. If an enhancement applies to only a fraction $F$ of the program, the maximum speedup can **never exceed $\frac{1}{1 - F}$**, even if that enhanced portion takes zero time ($\text{Speedup}_{\text{enhanced}} \to \infty$).

> **Common Pitfall:** Always measure the fraction of time **before** the enhancement is applied (in the original code). Measuring the fraction of time *after* the enhancement is already running gives an incorrect result.

---

### 2.2 Textbook Example 1: Web Server Computation vs. I/O
**Problem:** A new processor runs computation **10 times faster** than an older processor. A web serving workload spends **40% of its time on computation** and **60% of its time waiting for I/O**. What is the overall system speedup?

**Solution:**
* $\text{Fraction}_{\text{enhanced}} = 0.40$
* $\text{Speedup}_{\text{enhanced}} = 10$
* $\text{Fraction}_{\text{unenhanced}} = 1 - 0.40 = 0.60$

$$\text{Speedup}_{\text{overall}} = \frac{1}{0.60 + \frac{0.40}{10}} = \frac{1}{0.60 + 0.04} = \frac{1}{0.64} \approx \mathbf{1.56\times}$$

*Notice:* Even though the processor compute power increased by $10\times$, the whole web server only speeds up by $1.56\times$ because I/O waiting time dominates.

---

### 2.3 Textbook Example 2: Comparing Two Hardware Design Choices
**Problem:** A graphics processor design team has enough engineering budget to implement only one of two performance proposals for a graphics benchmark:
* **Option A:** Speed up Floating-Point Square Root (`FSQRT`) by **$10\times$**. `FSQRT` accounts for **20%** of the original execution time.
* **Option B:** Speed up **all** Floating-Point (`FP`) instructions by **$1.6\times$**. FP instructions account for **50%** of the original execution time.

Which design alternative provides the better overall speedup?

**Solution:**
* **Calculate Option A (FSQRT):**
  $$\text{Speedup}_{\text{FSQRT}} = \frac{1}{(1 - 0.20) + \frac{0.20}{10}} = \frac{1}{0.80 + 0.02} = \frac{1}{0.82} \approx \mathbf{1.22\times}$$

* **Calculate Option B (All FP Instructions):**
  $$\text{Speedup}_{\text{FP}} = \frac{1}{(1 - 0.50) + \frac{0.50}{1.6}} = \frac{1}{0.50 + 0.3125} = \frac{1}{0.8125} \approx \mathbf{1.23\times}$$

* **Conclusion:** **Option B is slightly better** ($1.23\times$ vs. $1.22\times$). Making a moderately used feature much faster (Option A) is slightly less effective than making a heavily used feature moderately faster (Option B).

---

### 2.4 Textbook Example 3: Applying Amdahl’s Law to Reliability
Amdahl’s law applies to reliability as well as performance.
* In our earlier disk subsystem, total failure rate was **23 per million hours**, of which the power supply was responsible for **5 per million hours**.
* The power supply fraction of failures is:
  $$\text{Fraction} = \frac{5}{23} \approx 0.22$$
* By adding a redundant power supply, we improved the power supply reliability by **$4,150\times$**.
* **Overall System Reliability Improvement:**
  $$\text{System Improvement} = \frac{1}{(1 - 0.22) + \frac{0.22}{4150}} = \frac{1}{0.78 + 0.00005} \approx \frac{1}{0.78} \approx \mathbf{1.28\times}$$
* **Insight:** Even though one component became $4,150\times$ more reliable, the overall system reliability only improved by **$28\%$** because other components (disks, fan, controller) now dominate system failure.

---

## 3. The Processor Performance Equation (CPU Equation)

All modern computers run using a constant clock that ticks in discrete cycles. 
* **Clock Cycle Time ($T_c$):** The duration of one clock cycle (e.g., $0.5\text{ ns}$).
* **Clock Rate ($f$):** The frequency of the clock (e.g., $2\text{ GHz} = \frac{1}{0.5\text{ ns}}$).

### 3.1 The Fundamental Formula

$$\text{CPU Time} = \text{Instruction Count} \times \text{CPI} \times \text{Clock Cycle Time}$$

Or equivalently:

$$\text{CPU Time} = \frac{\text{Instruction Count} \times \text{CPI}}{\text{Clock Rate}}$$

Where:
* **Instruction Count (IC):** The total number of instructions executed while running the program (the path length).
* **CPI (Clock Cycles per Instruction):** The average number of clock cycles required to execute each instruction.
* **IPC (Instructions per Clock):** The inverse of CPI ($\text{IPC} = \frac{1}{\text{CPI}}$).

#### Dimensional Analysis (How the Units Cancel):
$$\frac{\text{Instructions}}{\text{Program}} \times \frac{\text{Clock Cycles}}{\text{Instruction}} \times \frac{\text{Seconds}}{\text{Clock Cycle}} = \frac{\text{Seconds}}{\text{Program}} = \mathbf{\text{CPU Time}}$$

---

### 3.2 Interdependence of the Three Factors
CPU execution time is equally sensitive to all three factors (reducing any of them by 10% reduces CPU time by 10%). However, an architect cannot change one in isolation because they depend on different technologies:

| Factor | Primary Contributing Technologies |
| :--- | :--- |
| **Clock Cycle Time ($T_c$)** | Silicon circuit design, semiconductor technology, and pipeline depth |
| **Cycles Per Instruction (CPI)** | Processor microarchitecture, pipeline organization, cache design, and ISA |
| **Instruction Count (IC)** | Instruction Set Architecture (ISA) design and compiler optimization |

---

### 3.3 Detailed CPI Calculation
Different instructions (e.g., integer adds vs. memory loads vs. floating-point divides) take different numbers of clock cycles. 

Total processor clock cycles is calculated by summing across all instruction classes:

$$\text{CPU Clock Cycles} = \sum_{i=1}^{n} (\text{IC}_i \times \text{CPI}_i)$$

Dividing both sides by the total Instruction Count gives the **Overall Average CPI**:

$$\text{Overall CPI} = \sum_{i=1}^{n} \left( \frac{\text{IC}_i}{\text{Instruction Count}} \times \text{CPI}_i \right) = \sum_{i=1}^{n} (\text{Frequency}_i \times \text{CPI}_i)$$

Where $\text{Frequency}_i$ is the fraction of total executed instructions that belong to class $i$.

---

### 3.4 Textbook Example 4: Comparing Design Options Using the CPU Equation
**Problem:** A benchmark has the following measured instruction frequencies and CPIs:
* Floating-Point (`FP`) operations: **25%** of instructions, average **$\text{CPI} = 4.0$**
* Other operations: **75%** of instructions, average **$\text{CPI} = 1.33$**
* Inside the FP operations, Floating-Point Square Root (`FSQRT`) makes up **2%** of all instructions with **$\text{CPI} = 20$**

Compare two design proposals:
1. Decrease the CPI of `FSQRT` from 20 down to **2**.
2. Decrease the average CPI of **all** FP operations from 4.0 down to **2.5**.

**Step-by-Step Solution:**
1. **Find original baseline CPI:**
   $$\text{CPI}_{\text{original}} = (0.25 \times 4.0) + (0.75 \times 1.33) = 1.0 + 1.0 = \mathbf{2.00}$$

2. **Option 1 (Enhance FSQRT only):**
   * Cycles saved per instruction = $\text{Frequency} \times (\text{CPI}_{\text{old}} - \text{CPI}_{\text{new}}) = 0.02 \times (20 - 2) = 0.02 \times 18 = 0.36$
   $$\text{CPI}_{\text{newFSQRT}} = 2.00 - 0.36 = \mathbf{1.64}$$
   $$\text{Speedup} = \frac{\text{CPI}_{\text{original}}}{\text{CPI}_{\text{newFSQRT}}} = \frac{2.00}{1.64} \approx \mathbf{1.22\times}$$

3. **Option 2 (Enhance all FP operations):**
   $$\text{CPI}_{\text{newFP}} = (0.75 \times 1.33) + (0.25 \times 2.5) = 1.00 + 0.625 = \mathbf{1.625}$$
   $$\text{Speedup} = \frac{\text{CPI}_{\text{original}}}{\text{CPI}_{\text{newFP}}} = \frac{2.00}{1.625} \approx \mathbf{1.23\times}$$

*Note:* Both the CPU performance equation and Amdahl's Law produce the exact same speedup ($1.22\times$ vs. $1.23\times$).

---

### 3.5 Measuring Factors in Practice
* **Hardware Performance Counters:** Modern processors include special hardware counters that count instructions executed and clock cycles in real time to calculate CPI and identify code bottlenecks.
* **Energy Management Impact:** Techniques like **DVFS (Dynamic Voltage and Frequency Scaling)** vary the clock speed dynamically to save energy, which makes CPU time measurements vary. For reproducible performance experiments, DVFS is often locked or disabled during benchmarking.

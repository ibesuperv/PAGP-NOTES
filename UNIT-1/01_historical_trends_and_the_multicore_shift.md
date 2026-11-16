# 01: Introduction and Computer Performance Trends

---

## 1. Growth in Processor Performance Over Time

In the last 70 years, computers have improved at an incredible pace. For example, today's smartphone (costing under $500) is just as fast as a $50 million supercomputer from 1993.

Historically, computer performance grew in different phases:

* **Before 1986 (First 25 Years):** Performance grew by about **25% per year**. Half of this came from better chip technology, and half came from better computer design.
* **Late 1970s (Microprocessors Arrive):** Microprocessors became cheap and easy to mass-produce. Performance growth rose to about **35% per year**.
* **1986 to 2003 (The RISC Era / Golden Age):** Performance grew at a record **52% per year** (doubling every 2 years). Microprocessors became roughly 25 times faster than what chip technology alone could have achieved.
* **2003 to 2011:** Growth slowed down to **23% per year** (doubling every 3.5 years) because chips were getting too hot and reaching power limits.
* **2011 to 2015:** Growth dropped below **12% per year** (doubling every 8 years) due to limits of parallel processing (Amdahl's Law).
* **Since 2015:** Performance growth is down to only **3.5% per year** (doubling every 20 years) because Moore's Law is ending.

### Summary Table: Performance Growth Rates

| Time Period | Yearly Growth Rate | Doubling Time | Main Reason |
| :--- | :--- | :--- | :--- |
| **Up to 1986** | ~25% to 35% | 2.5 to 3.5 years | Early microprocessors and basic chip improvements |
| **1986 – 2003** | **~52%** | **~2 years** | RISC design, pipelining, ILP, and caches |
| **2003 – 2011** | ~23% | ~3.5 years | End of Dennard scaling (chips got too hot) |
| **2011 – 2015** | < 12% | ~8 years | Amdahl's Law limits on adding cores |
| **Since 2015** | **~3.5%** | **~20 years** | Slowdown of Moore's Law |

---

## 2. Why New Architectures Succeeded in the 1980s

In the early 1980s, two big market changes made it easy to introduce new computer designs:

1. **People stopped writing in Assembly Language:** Programmers started using high-level languages like C. Software did not need to match the exact hardware instructions of old chips anymore.
2. **Standard Operating Systems (UNIX and Linux):** Companies didn't have to build custom operating systems for every new chip. A new chip could just run standard UNIX or Linux.

### The Rise of RISC
These changes gave rise to **RISC (Reduced Instruction Set Computer)** architectures. RISC focused on two main ideas:
* **Instruction-Level Parallelism (ILP):** Running parts of instructions at the same time using **pipelining** and **multiple instruction issue** (running more than one instruction per clock cycle).
* **Caches:** Using fast on-chip memory so the processor doesn't have to wait for slow main memory.

### How x86 and ARM Handled RISC
* Older complex machines (like DEC VAX) could not compete and died out.
* **Intel x86:** Intel kept its old x86 instruction set, but inside the chip, hardware translated complex x86 instructions into simple **RISC-like micro-operations ($\mu$-ops)**. As chips got more transistors, the extra hardware needed for this translation became very small and did not hurt desktop/server chips.
* **ARM:** For mobile phones, the power and space cost of translating x86 instructions was too high. Native RISC architectures like **ARM** became the market leader for mobile devices.

### Comparison: RISC vs. CISC

| Feature | RISC (Reduced Instruction Set Computer) | CISC (Complex Instruction Set Computer) |
| :--- | :--- | :--- |
| **Instruction Size** | **Fixed length** (usually 32 bits), simple format | **Variable length** (e.g., 1 to 15 bytes in x86) |
| **Execution Time** | Most instructions execute in **1 clock cycle** | Instructions take **multiple clock cycles** |
| **Memory Access** | **Load/Store only**: arithmetic works only on registers | Arithmetic operations can directly access memory |
| **Control Unit** | **Hardwired** (fast, simple silicon layout) | **Microprogrammed** (complex control logic) |
| **Registers** | **Many** general-purpose registers (32+) | **Fewer** general-purpose registers (8 to 16) |
| **Pipelining** | **Easy and efficient** due to uniform instruction lengths | **Difficult** due to varying instruction sizes and cycles |
| **Design Emphasis** | Simpler hardware, burden shifts to the **compiler** | Complex hardware, smaller burden on the compiler |
| **Code Size** | Larger binary size (needs more instructions) | Compact binary size (fewer instructions per task) |
| **Examples** | ARM, MIPS, RISC-V | Intel x86, AMD, DEC VAX |

#### Concrete Example: Multiplying Two Numbers in Memory
Suppose we want to multiply two numbers stored in memory locations `A` and `B`, and save the result into `A`.

* **CISC Approach (Single Complex Instruction):**
  ```text
  MULT A, B
  ```
  * What the hardware does: Directly loads `A` from RAM, loads `B` from RAM, multiplies them in the execution unit, and writes the product back to RAM location `A`.
  * *Advantage:* Very short code (1 instruction), uses less RAM space.
  * *Disadvantage:* Hardware execution logic is very complex, multi-cycle, and hard to pipeline.

* **RISC Approach (Load/Store Architecture):**
  ```text
  LOAD  R1, A        ; Read memory location A into register R1
  LOAD  R2, B        ; Read memory location B into register R2
  MULT  R1, R1, R2   ; Multiply registers R1 and R2; save result in R1
  STORE A, R1        ; Write value from register R1 back to memory location A
  ```
  * What the hardware does: Each instruction does only one simple step. Memory is accessed only through explicit `LOAD` and `STORE` instructions. Arithmetic happens strictly inside CPU registers.
  * *Advantage:* Every instruction is predictable, fixed in length, and can easily run in a smooth pipeline at high clock speed.
  * *Disadvantage:* Requires more instructions (4 instructions instead of 1) to do the same task.

---

## 3. Four Big Impacts of the 1986–2003 Performance Growth

Between 1978 and 2003, computer performance grew by **50,000 times**. This had four major effects on the industry:

1. **Massive Power in Small Devices:** Everyday consumer gadgets (like smartphones) became faster than old multi-million dollar supercomputers.
2. **New Types of Computers Emerged:**
   * 1980s: Personal Computers (PCs) and desktop workstations.
   * 2000s to Present: Smartphones, tablets, and **Warehouse-Scale Computers (WSCs)** (data centers with 100,000 servers connected together like one giant computer).
3. **Microprocessors Took Over Everything:** Minicomputers, mainframes, and supercomputers were all replaced by standard microprocessors or clusters of microprocessors.
4. **Software Swapped Performance for Productivity:**
   * Because computers became so fast, developers stopped worrying about saving every byte or cycle.
   * Developers moved from C/C++ to languages that are easier to write, like Java, Python, and JavaScript.
   * New tools emerged: Just-In-Time (JIT) compilers, web frameworks, and Software as a Service (SaaS, running apps in the cloud instead of installing them locally).
   * Enabled compute-heavy apps like Google Translate (taking a photo or voice, sending it to a data center, and translating it in real time).

---

## 4. Why Single-Processor Performance Stopped Growing

The 17-year period of 52% yearly growth ended around 2004 because of two main physical problems:

### 4.1 The End of Dennard Scaling (~2004)
* **What was Dennard Scaling?** In 1974, Robert Dennard observed that as transistors got smaller, power density (heat per area) stayed constant. Smaller transistors ran faster while using less power.
* **Why did it stop?** Around 2004, engineers could not lower the chip's voltage and current any further without making the circuits unreliable.
* **The Power Wall:** Because voltage could not be lowered, making chips run at higher clock speeds caused them to overheat. Air-cooled chips hit a limit of around **100W to 150W**. Intel had to cancel its high-speed single-core projects (like the Tejas chip at 4+ GHz).

### 4.2 The Slowdown of Moore's Law (~2015)
* **What was Moore's Law?** In 1965, Gordon Moore predicted that the number of transistors on a chip would double every year (changed in 1975 to **doubling every two years**).
* **The Reality:** Transistor growth has slowed down drastically.
  * In 2010, an Intel chip had **1.17 billion** transistors.
  * If Moore's Law stayed true, by 2016 a chip should have had **18.72 billion** transistors.
  * In reality, the equivalent 2016 Intel chip had only **1.75 billion** transistors (about 10 times less than predicted).

### The Four Combined Reasons Performance Slowed Down:
1. Transistors are no longer improving quickly (end of Dennard scaling and slowing of Moore's Law).
2. Chip power limits (cannot cool down hot chips).
3. Replacing one hot, fast processor with several slower, energy-efficient processors.
4. Limits to parallel processing described by Amdahl's Law.

---

## 5. The Shift to Multicore: Implicit vs. Explicit Parallelism

In 2004, the entire computer industry changed direction: instead of making a **single core faster**, they began putting **multiple cores on one chip**.

### Implicit Parallelism (Old Way)
* Used **Instruction-Level Parallelism (ILP)**.
* The hardware and compiler automatically found instructions that could run in parallel.
* The programmer did not have to change anything; regular sequential code ran faster automatically on newer chips.

### Explicit Parallelism (New Way)
* Uses multiple cores and parallel hardware:
  * **Data-Level Parallelism (DLP):** Doing the same operation on many data items at once (used in SIMD and GPUs).
  * **Thread-Level Parallelism (TLP):** Running separate tasks or threads concurrently (used in multicore CPUs).
  * **Request-Level Parallelism (RLP):** Handling many independent web requests at the same time (used in cloud servers and data centers).
* **The Problem:** The programmer now has the hard job of rewriting software to split work across multiple cores, manage threads, and prevent race conditions.

---

## 6. Limits to Parallel Speedup: Amdahl’s Law

Amdahl’s Law shows that adding more cores does not always make a program run much faster. The maximum speedup is always limited by the **serial part** of the code (the part that cannot be split among cores).

$$\text{Speedup} = \frac{1}{\text{Serial Fraction} + \frac{\text{Parallel Fraction}}{\text{Number of Cores}}}$$

### Simple Example from Textbook:
* Suppose **10%** of a task is serial, and **90%** can be parallelized.
* Even if you use an **infinite number of cores**, the parallel part takes almost 0 time, but the 10% serial part still takes its normal time.
* The maximum possible speedup is:
  $$\text{Maximum Speedup} = \frac{1}{0.10} = 10\times$$
* Adding 100 cores or 1,000 cores will never give you more than a **10x** speedup.

---

### Example Problem 1: Finding Speedup on a Multi-Core System
**Problem Statement:**  
A program runs in **100 seconds** on a single-core processor. Out of this, **80 seconds** can be parallelized across multiple cores, while **20 seconds** must run sequentially (serial code).
1. Calculate the overall speedup when running on an **8-core processor**.
2. What is the **maximum possible speedup** if we use an infinite number of cores?

**Step-by-Step Solution:**
1. **Identify fractions:**
   * Total time = $100\text{ s}$
   * Serial fraction ($F_{\text{serial}}$) = $\frac{20}{100} = 0.20$ (or $20\%$)
   * Parallel fraction ($F_{\text{parallel}}$) = $\frac{80}{100} = 0.80$ (or $80\%$)
   * Number of cores ($N$) = $8$

2. **Calculate speedup for 8 cores:**
   $$\text{Speedup} = \frac{1}{F_{\text{serial}} + \frac{F_{\text{parallel}}}{N}} = \frac{1}{0.20 + \frac{0.80}{8}} = \frac{1}{0.20 + 0.10} = \frac{1}{0.30} \approx \mathbf{3.33\times}$$
   *(The execution time drops from $100\text{ s}$ to $20 + \frac{80}{8} = 30\text{ s}$.)*

3. **Calculate maximum theoretical speedup ($N \to \infty$):**
   $$\text{Maximum Speedup} = \frac{1}{F_{\text{serial}}} = \frac{1}{0.20} = \mathbf{5\times}$$
   *(Even with thousands of cores, the program will never run faster than $5\times$ or take less than $20\text{ s}$.)*

---

### Example Problem 2: Finding Required Parallel Fraction for a Target Speedup
**Problem Statement:**  
A computer architect wants to achieve an overall speedup of **$4\times$** on an **8-core processor**. What fraction of the application code must be parallelized to hit this speedup target?

**Step-by-Step Solution:**
1. **Identify given values:**
   * Target Speedup ($S$) = $4$
   * Number of cores ($N$) = $8$
   * Let parallel fraction = $p$, so serial fraction = $1 - p$

2. **Apply Amdahl's Law formula:**
   $$S = \frac{1}{(1 - p) + \frac{p}{N}}$$
   $$4 = \frac{1}{(1 - p) + \frac{p}{8}}$$

3. **Solve for $p$:**
   $$(1 - p) + \frac{p}{8} = \frac{1}{4}$$
   $$1 - p + 0.125p = 0.25$$
   $$1 - 0.875p = 0.25$$
   $$0.875p = 0.75$$
   $$p = \frac{0.75}{0.875} \approx \mathbf{0.857} \quad (\text{or } \mathbf{85.7\%})$$

**Answer:** At least **$85.7\%$** of the application must be parallelized to achieve a $4\times$ speedup on 8 cores.

---

## 7. The Future: Domain-Specific Architectures (DSAs)

Because general-purpose CPUs can only grow by ~3.5% per year, the only way to get better performance and energy efficiency today is **specialization**.

* **What are DSAs?** Instead of building one chip that tries to do everything (general-purpose CPU), companies build special hardware that does **one specific job extremely well**.
* **Examples:** Chips designed specifically for machine learning (like Google TPUs and NVIDIA GPUs/Tensor Cores) or image/video processing.
* **Quantitative Approach:** Hardware design relies on real data, benchmark measurements, and simulation to make the best trade-offs for power, performance, and cost.

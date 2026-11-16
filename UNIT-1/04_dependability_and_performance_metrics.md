# 04: Dependability and Performance Metrics

---

## 1. Dependability in Computer Systems

Dependability is the ability of a computer system to provide its specified service reliably and continuously

Historically, silicon chips were among the most reliable computer components. However, at tiny modern feature sizes (16 nm and smaller), both **transient faults** (temporary glitches from cosmic rays or electrical noise) and **permanent faults** (burned-out circuits) happen more often.

### 1.1 Service Level Agreements (SLAs) and System States

In modern cloud computing, service quality is defined using a **Service Level Agreement (SLA)** or **Service Level Objective (SLO)**. An SLA is a contract specifying uptime guarantees, with financial penalties if service drops below the agreed limit.

A system alternates between two states with respect to an SLA:

1. **Service Accomplishment:** The system delivers service according to the SLA (the system is "UP").
2. **Service Interruption:** The delivered service fails to satisfy the SLA (the system is "DOWN").

```
             Failure (State transition 1 -> 2)
  +-----------------------+              +-----------------------+
  | Service Accomplishment| -----------> |  Service Interruption |
  |      (System UP)      | <----------- |     (System DOWN)     |
  +-----------------------+              +-----------------------+
            Restoration (State transition 2 -> 1)
```

---

### 1.2 Key Dependability Metrics: MTTF, MTTR, MTBF, FIT, and Availability

- **MTTF (Mean Time To Failure):** The average continuous time the system operates properly before a failure happens. This measures **Reliability**.
- **MTTR (Mean Time To Repair):** The average time taken to fix the system and restore service after a failure occurs. This measures **Service Interruption**.
- **MTBF (Mean Time Between Failures):** The total average time from one failure to the next:
  $$\text{MTBF} = \text{MTTF} + \text{MTTR}$$
- **Failure Rate and FIT (Failures In Time):**
  - Failure rate is the mathematical reciprocal of MTTF:
    $$\text{Failure Rate} = \frac{1}{\text{MTTF}}$$
  - **FIT:** Failures per **1 billion ($10^9$) hours** of continuous operation:
    $$\text{FIT} = \frac{10^9}{\text{MTTF}}$$
  - _Example:_ An MTTF of $1,000,000\text{ hours} = \frac{10^9}{10^6} = \mathbf{1,000\text{ FIT}}$.

- **Module Availability:** The fraction of total time that the system is successfully up and serving requests:
  $$\text{Availability} = \frac{\text{MTTF}}{\text{MTTF} + \text{MTTR}}$$

---

### 1.3 System Failure Rate (Sum of Components)

If component lifetimes follow an exponential distribution (meaning a part does not become more likely to fail simply because it is older), and failures are independent, the total system failure rate is the sum of all individual component failure rates:

$$\text{Failure Rate}_{\text{system}} = \sum_{i} \text{Failure Rate}_i$$

---

### 1.4 Textbook Example 1: Calculating System MTTF

**Problem:** A disk subsystem has the following components and MTTF values:

- 10 Hard Disks, each with $\text{MTTF} = 1,000,000\text{ hours}$
- 1 ATA Controller with $\text{MTTF} = 500,000\text{ hours}$
- 1 Power Supply with $\text{MTTF} = 200,000\text{ hours}$
- 1 Cooling Fan with $\text{MTTF} = 200,000\text{ hours}$
- 1 ATA Cable with $\text{MTTF} = 1,000,000\text{ hours}$

Assuming independent failures, calculate the total system failure rate (in FIT) and the overall system MTTF.

**Solution:**

1. **Sum the failure rates:**
   $$\text{Failure Rate}_{\text{system}} = 10 \left(\frac{1}{1,000,000}\right) + \frac{1}{500,000} + \frac{1}{200,000} + \frac{1}{200,000} + \frac{1}{1,000,000}$$
   $$\text{Failure Rate}_{\text{system}} = \frac{10 + 2 + 5 + 5 + 1}{1,000,000\text{ hours}} = \frac{23}{1,000,000\text{ hours}} = \frac{23,000}{1,000,000,000\text{ hours}}$$
   $$\text{Failure Rate}_{\text{system}} = \mathbf{23,000\text{ FIT}}$$

2. **Calculate system MTTF:**
   $$\text{MTTF}_{\text{system}} = \frac{1}{\text{Failure Rate}} = \frac{1,000,000,000\text{ hours}}{23,000} \approx \mathbf{43,500\text{ hours}} \quad (\approx 5\text{ years})$$

---

### 1.5 Redundancy to Improve Dependability

Redundancy is the primary technique used to survive failures:

- **Time Redundancy:** Re-running an operation if an error occurs (useful for transient faults).
- **Resource Redundancy:** Adding duplicate hardware components to take over when an active component breaks.

#### Textbook Example 2: Benefit of a Redundant Power Supply Pair

**Problem:** Suppose a storage system needs 1 power supply to operate ($\text{MTTF} = 200,000\text{ hours}$). We add a second identical redundant power supply so the system keeps running if one fails. An operator takes on average **24 hours** to replace a broken supply ($\text{MTTR} = 24\text{ hours}$). Calculate the MTTF of this redundant power supply pair.

**Solution:**

- For two independent components where either one can run the system, the MTTF of the pair is the mean time until the first supply fails divided by the probability that the second supply fails before the first one is replaced:
  $$\text{MTTF}_{\text{pair}} \approx \frac{\text{MTTF}_{\text{single}}^2}{2 \times \text{MTTR}}$$
- Plugging in the numbers:
  $$\text{MTTF}_{\text{pair}} = \frac{200,000^2}{2 \times 24} = \frac{40,000,000,000}{48} \approx \mathbf{830,000,000\text{ hours}}$$
- **Conclusion:** Adding one redundant power supply makes the power subsystem about **$4,150\times$ more reliable** than using a single power supply!

---

## 2. Measuring, Reporting, and Summarizing Performance

### 2.1 Defining Computer Performance: Time vs. Throughput

What do we mean when we say a computer is "faster"?

1. **Response Time (Elapsed / Wall-Clock Time):**
   - The total time between the start and completion of a single job.
   - Includes everything: processor compute time, disk access, memory access, input/output, and operating system overhead.
   - This is what an individual user cares about (e.g., a mobile phone or desktop user).
2. **CPU Execution Time:**
   - The time the CPU actually spends computing instructions for a specific program (excludes time spent waiting for I/O or running other background programs).
3. **Throughput:**
   - The total number of tasks or transactions completed per unit time.
   - This is what data center, cloud, and server administrators care about (e.g., web requests per second).

---

### 2.2 Relative Performance (Speedup)

When comparing two computers, $X$ and $Y$, the phrase **"$X$ is $n$ times as fast as $Y$"** mathematically means:

$$n = \frac{\text{Execution Time}_Y}{\text{Execution Time}_X} = \frac{\text{Performance}_X}{\text{Performance}_Y}$$

- If Computer A runs a program in 10 seconds and Computer B runs it in 15 seconds:
  $$n = \frac{15\text{ s}}{10\text{ s}} = 1.5 \implies \text{Computer A is } 1.5\times \text{ as fast as Computer B.}$$

---

## 3. Benchmarks

The only truly reliable measure of performance is running **real application programs**.

### 3.1 Types of Benchmark Programs

- **Real Applications (Best):** Full-scale programs used in real life (e.g., compilers like GCC, video encoders, web browsers).
- **Kernels (Flawed):** Small extracted loops from real code.
- **Toy Programs (Flawed):** Small 100-line coding assignments (e.g., Quicksort).
- **Synthetic Benchmarks (Flawed):** Artificial programs created to mimic typical instruction mixes (e.g., **Dhrystone**).
  - _Why Kernels, Toy, and Synthetic Benchmarks are Discredited:_ Compilers and chip architects can easily "cheat" by optimizing specifically for these small fake programs, producing numbers that do not reflect real-world application performance. (Yet, Dhrystone is still widely used in embedded computing).

---

### 3.2 Benchmark Suites: SPEC and TPC

To avoid judging a machine on just one program, industry uses **Benchmark Suites** (standard collections of real programs).

#### 1. SPEC (Standard Performance Evaluation Corporation)

- **SPEC CPU2017:** The gold standard for measuring processor performance. Contains two sets:
  - **CINT2017:** 10 compute-intensive integer programs (compilers, interpreters, AI search, compression).
  - **CFP2017:** 17 compute-intensive floating-point programs (fluid dynamics, ray tracing, weather forecasting).
- **Other SPEC Suites:**
  - **SPECrate:** Measures server throughput by running multiple copies of SPEC CPU simultaneously (Request-Level Parallelism).
  - **OMP / MPI / ACCEL:** Measures thread-level parallelism and accelerator/GPU performance using OpenMP, MPI, and OpenACC.
  - **SPECjbb:** Evaluates Java server performance.
  - **SPECpower:** Measures power consumption of servers while running workloads.

#### 2. TPC (Transaction Processing Council)

Measures database and online transaction processing (OLTP) performance in **transactions per second**:

- **TPC-C:** Simulates complex order-entry and query transactions.
- **TPC-H:** Ad-hoc decision support systems (queries are unrelated).
- **TPC-E:** Modern OLTP workload modeling stock brokerage accounts.
- **TPCx-HS:** Big Data benchmark measuring Hadoop MapReduce systems.
- _Key Rule:_ TPC requires that systems meet strict **response time limits** before transaction rates count, and system cost must be published to evaluate price-performance.

---

### 3.3 Principle of Reporting Results: Reproducibility

Reports must follow the rule of **Reproducibility**: an experimenter must provide all details (exact hardware specs, OS version, compiler flags, and base vs. peak tuning parameters) so that any third party can independently replicate the exact benchmark results.

---

## 4. Summarizing Performance: The Geometric Mean

When a computer runs an entire suite of different benchmarks, how do we calculate a single summary number?

### 4.1 The Flaw of Arithmetic Means

- Comparing the simple arithmetic mean of execution times is flawed because longer programs dominate the score, hiding speedups on shorter programs.
- Using weighted arithmetic means normalized to a reference machine introduces bias toward the chosen reference machine.

---

### 4.2 SPECRatio and the Geometric Mean

SPEC normalizes execution times against an agreed **reference computer**:

$$\text{SPECRatio} = \frac{\text{Execution Time}_{\text{reference}}}{\text{Execution Time}_{\text{rated}}}$$

- A higher SPECRatio means better performance (e.g., $\text{SPECRatio} = 20$ means the machine is 20 times faster than the reference machine).
- Because SPECRatios are **unitless ratios**, calculating an arithmetic mean is mathematically invalid.
- Instead, SPEC uses the **Geometric Mean**:

$$\text{Geometric Mean} = \sqrt[n]{\prod_{i=1}^{n} \text{SPECRatio}_i}$$

---

### 4.3 Why Geometric Mean Must Be Used (Two Key Properties)

Using the geometric mean provides two essential mathematical guarantees:

1. **The geometric mean of ratios equals the ratio of the geometric means:**
   $$\frac{\text{Geometric Mean}_A}{\text{Geometric Mean}_B} = \sqrt[n]{\prod_{i=1}^{n} \left(\frac{\text{SPECRatio}_{Ai}}{\text{SPECRatio}_{Bi}}\right)}$$

2. **The reference computer completely drops out (Irrelevance of Reference Machine):**
   $$\frac{\text{SPECRatio}_{Ai}}{\text{SPECRatio}_{Bi}} = \frac{\frac{\text{Time}_{\text{ref}}}{\text{Time}_{Ai}}}{\frac{\text{Time}_{\text{ref}}}{\text{Time}_{Bi}}} = \frac{\text{Time}_{Bi}}{\text{Time}_{Ai}} = \frac{\text{Performance}_{Ai}}{\text{Performance}_{Bi}}$$

   Because $\text{Time}_{\text{ref}}$ cancels out in the ratio, **the choice of reference machine has zero impact on the relative performance comparison between Computer A and Computer B**.

---

### 4.4 Textbook Example: Demonstrating Geometric Mean

Comparing an **AMD A10** processor against an **Intel Xeon E5-2690** across SPEC2006 integer programs:

- The geometric mean of the AMD SPECRatios is **31.91**.
- The geometric mean of the Intel SPECRatios is **63.72**.
- Comparing the two machines:
  $$\frac{\text{Intel Geometric Mean}}{\text{AMD Geometric Mean}} = \frac{63.72}{31.91} = \mathbf{2.00}$$
- The geometric mean of the individual speedup ratios ($\frac{\text{Time}_{\text{AMD}}}{\text{Time}_{\text{Intel}}}$) for every program in the suite gives the identical value of **$2.00$**.
- **Result:** Intel Xeon is twice as fast as AMD A10, regardless of which reference machine was used to compute the original SPECRatios.

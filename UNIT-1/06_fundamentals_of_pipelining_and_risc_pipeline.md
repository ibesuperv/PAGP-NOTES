# 06: Fundamentals of Pipelining and the Classic RISC Pipeline

---

## 1. What is Pipelining?

**Pipelining** is a hardware implementation technique where multiple instructions are overlapped in execution. It takes advantage of parallelism among the steps required to execute an instruction sequence.

### The Assembly Line Analogy
* Pipelining works like an automobile assembly line:
  * In a car factory, multiple tasks occur simultaneously (e.g., chassis assembly, engine installation, painting), each working on a *different* car.
  * In a processor pipeline, different hardware stages complete different parts of *different* instructions in parallel on every clock cycle.
* **Pipe Stage (or Segment):** Each step in the pipeline that performs a portion of an instruction.
* **Processor Cycle:** The time required to move an instruction one step forward through the pipeline. In computers, this processor cycle is almost always **1 clock cycle**.
* **Cycle Time Limit:** Because all stages must step forward together, the clock cycle is dictated by the **slowest pipeline stage**.

```
Clock Cycle:       1     2     3     4     5     6     7     8
Instruction 1:   [IF]  [ID]  [EX]  [MEM] [WB]
Instruction 2:         [IF]  [ID]  [EX]  [MEM] [WB]
Instruction 3:               [IF]  [ID]  [EX]  [MEM] [WB]
Instruction 4:                     [IF]  [ID]  [EX]  [MEM] [WB]
Instruction 5:                           [IF]  [ID]  [EX]  [MEM] [WB]
```

### Ideal Speedup Formula
If all $n$ pipe stages are perfectly balanced:

$$\text{Time per instruction on pipelined processor} = \frac{\text{Time per instruction on unpipelined machine}}{\text{Number of pipe stages } (n)}$$

$$\text{Ideal Speedup} = \text{Number of pipe stages } (n)$$

* **Key Performance Truth:** Pipelining increases instruction **throughput** (instructions completed per second), but does **not** decrease the execution latency of an individual instruction (individual instructions still take all 5 stages to complete).
* **Transparency:** Pipelining exploits instruction-level parallelism implicitly—it is **completely invisible to the programmer**.

---

## 2. Unpipelined RISC Implementation (The 5 Execution Cycles)

In a basic unpipelined implementation, every integer RISC instruction (ALU, Load/Store, Branch) executes across at most 5 discrete clock cycles:

1. **Instruction Fetch (IF):**
   * Send the Program Counter (PC) to instruction memory and fetch the instruction.
   * Increment PC by 4 (`PC = PC + 4`) to point to the next 32-bit instruction.
2. **Instruction Decode / Register Fetch (ID):**
   * Decode the opcode and read the source registers from the register file.
   * **Fixed-Field Decoding:** Because register field positions (`rs1`, `rs2`, `rd`) are fixed in RISC-V, reading registers happens in parallel with decoding.
   * Sign-extend the immediate offset field.
   * Compute the potential branch target address (`PC + sign-extended offset`).
3. **Execution / Effective Address (EX):**
   * The ALU operates on the prepared operands depending on instruction type:
     * *Memory reference (Load/Store):* ALU adds the base register and offset to compute the memory address.
     * *Register-Register ALU:* ALU performs the requested arithmetic/logical operation.
     * *Register-Immediate ALU:* ALU operates on the source register and sign-extended immediate.
     * *Branch:* Evaluates the branch condition.
   * *(Note: In load-store architectures, address calculation and data arithmetic never happen in the same instruction, so they share a single cycle).*
4. **Memory Access (MEM):**
   * *Load:* Reads data from memory using the effective address computed in the EX stage.
   * *Store:* Writes data from the second source register into memory at the effective address.
5. **Write-Back (WB):**
   * Writes the result into the destination register in the register file (result comes from the ALU for ALU instructions, or from memory for a Load).

### Unpipelined Performance Baseline
* Branch instructions take **3 cycles**.
* Store instructions take **4 cycles**.
* ALU and Load instructions take **5 cycles**.
* With a typical mix (e.g., 12% branches, 10% stores, 78% ALU/loads), the average unpipelined CPI is **~4.66**.

---

## 3. The Classic Five-Stage RISC Pipeline

By starting a brand new instruction on every single clock cycle, each of the 5 execution steps becomes an independent pipeline stage: **IF $\to$ ID $\to$ EX $\to$ MEM $\to$ WB**.

```
             +----+      +----+      +----+      +----+      +----+
             | IF | ---> | ID | ---> | EX | ---> | MEM| ---> | WB |
             +----+      +----+      +----+      +----+      +----+
               |           |           |           |           |
Hardware:   Inst Cache  Reg File/     ALU        Data Cache  Reg File
             & PC Adder   Decode                   (RAM)     (Write)
```

### Three Design Rules to Prevent Hardware Resource Conflicts:
To allow 5 instructions to overlap simultaneously without crashing into each other:

1. **Separate Instruction and Data Memories:**
   * Uses separate **Instruction Cache (IM)** and **Data Cache (DM)**.
   * *Why:* Prevents a structural conflict between the **IF** stage (fetching a new instruction) and the **MEM** stage (reading/writing data memory) in the same clock cycle.
   * *Trade-off:* The memory system must deliver **5 times the bandwidth** of an unpipelined machine.
2. **Split Register File Access (First Half / Second Half Cycle):**
   * The register file is accessed by two different stages on every cycle: **ID** reads registers, and **WB** writes a register.
   * *Solution:* **Writes are performed in the first half of the clock cycle, and reads are performed in the second half.** This allows an instruction in ID to read a value written by an instruction in WB within the exact same clock cycle.
3. **Dedicated Program Counter & Branch Adders:**
   * PC must be incremented on every clock cycle in the IF stage.
   * A separate adder calculates the branch target address during the ID stage so the main ALU is free for arithmetic in the EX stage.

---

## 4. Pipeline Registers (Latches)

To prevent instructions in different stages from interfering with one another, hardware registers are placed between successive pipeline stages:

```
        +----+        +----+        +----+        +----+        +----+
        | IF |        | ID |        | EX |        | MEM|        | WB |
        +----+        +----+        +----+        +----+        +----+
              \      /      \      /      \      /      \      /
             [ IF/ID ]     [ ID/EX ]     [ EX/MEM ]    [ MEM/WB ]
```

### Purpose of Pipeline Registers:
1. **Isolation:** They ensure data from an instruction in stage $i$ does not overwrite data being processed by stage $i+1$.
2. **Passing Intermediate Data Across Cycles:**
   * *Example 1 (Store Instruction):* The data value to be stored is read during the **ID** stage, but the memory write happens in the **MEM** stage. The value is safely carried across the clock boundaries through the `ID/EX` and `EX/MEM` pipeline registers.
   * *Example 2 (ALU Instruction):* The result computed in **EX** is carried through the `EX/MEM` and `MEM/WB` registers until it is written to the register file during **WB**.
3. **Edge-Triggered:** Registers change their values instantaneously on a clock edge.

---

## 5. Limits on Pipeline Depth and Overhead

In reality, pipelining does not yield a speedup exactly equal to the number of stages due to physical constraints:

1. **Pipeline Stage Imbalance:**
   * The execution times of the stages are never completely identical.
   * Because the clock cycle must accommodate the **slowest stage**, faster stages waste time waiting.
2. **Pipeline Register Overhead:**
   * **Setup Time:** The time the register inputs must stay stable before the clock edge arrives.
   * **Propagation Delay:** The time required for the register outputs to become valid after the clock edge.
3. **Clock Skew:**
   * The maximum difference in arrival time between the clock signal arriving at two different registers on the chip.

> **The Physical Pipelining Limit:**  
> Once the clock cycle time is reduced down to the sum of **clock skew + register latch overhead**, no further pipelining is possible because no time remains in the clock cycle to perform actual logic work.

---

## 6. Textbook Worked Example: Speedup with Pipeline Overhead

### Problem Statement:
Consider an unpipelined processor with:
* Clock rate = **4 GHz** (Clock cycle time = **0.5 ns**).
* ALU instructions and branches require **4 cycles** (frequencies: 40% ALU, 20% branches).
* Memory operations require **5 cycles** (frequency: 40%).

Suppose we pipeline this processor into 5 stages. Due to clock skew and register setup overhead, pipelining adds **0.1 ns of overhead** to the clock period.  
Calculate the overall speedup gained by pipelining.

---

### Step-by-Step Solution:

1. **Find the average CPI and execution time of the unpipelined processor:**
   * Total frequency of 4-cycle instructions = $40\% + 20\% = 60\%$ (0.60)
   * Frequency of 5-cycle instructions = $40\%$ (0.40)
   $$\text{Average CPI}_{\text{unpipelined}} = (0.60 \times 4) + (0.40 \times 5) = 2.4 + 2.0 = \mathbf{4.4}$$

   $$\text{Average Instruction Execution Time}_{\text{unpipelined}} = \text{Clock Cycle} \times \text{Average CPI}$$
   $$\text{Average Instruction Execution Time}_{\text{unpipelined}} = 0.5\text{ ns} \times 4.4 = \mathbf{2.2\text{ ns}}$$

2. **Find the instruction execution time of the pipelined processor:**
   * Under ideal pipelining (1 instruction completes per cycle in steady state), CPI = 1.
   * The pipelined clock cycle equals the slowest stage (0.5 ns) plus the pipeline overhead (0.1 ns):
     $$\text{Clock Cycle}_{\text{pipelined}} = 0.5\text{ ns} + 0.1\text{ ns} = \mathbf{0.6\text{ ns}}$$
   * Average instruction execution time = $0.6\text{ ns} \times 1 = \mathbf{0.6\text{ ns}}$.

3. **Calculate the Speedup:**
   $$\text{Speedup} = \frac{\text{Average Instruction Time}_{\text{unpipelined}}}{\text{Average Instruction Time}_{\text{pipelined}}} = \frac{2.2\text{ ns}}{0.6\text{ ns}} \approx \mathbf{3.7\times}$$

### Analytical Conclusion:
Although the processor was divided into 5 stages, the speedup is **$3.7\times$** (not $5\times$). The **0.1 ns overhead** and stage imbalance create an Amdahl's Law limit on the effectiveness of pipelining.

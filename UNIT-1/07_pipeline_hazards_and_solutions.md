# 07: Pipeline Hazards and Their Solutions

---

## 1. What are Pipeline Hazards?

A **hazard** is any condition that prevents the next instruction in the instruction stream from executing in its designated clock cycle. When a hazard occurs, the pipeline must **stall** (insert delay cycles).

### The Three Classes of Hazards:
1. **Structural Hazards:** Hardware resource conflicts where two instructions need the same hardware unit in the same clock cycle (e.g., a single memory port accessed for both instruction fetch and data load).
2. **Data Hazards:** An instruction depends on the result of a previous instruction that is still moving through the pipeline.
3. **Control Hazards (Branch Hazards):** Caused by branch or jump instructions that change the Program Counter (PC), meaning the pipeline doesn't know which instruction to fetch next.

---

## 2. Pipeline Performance with Stalls

When a hazard cannot be resolved immediately, the processor inserts **stalls** (also called **pipeline bubbles**):
* The stalled instruction and all younger instructions behind it freeze in place.
* All older instructions ahead in the pipeline continue executing to clear the hazard.
* No new instructions are fetched during the stall.

### Speedup Formula with Stalls

$$\text{CPI}_{\text{pipelined}} = \text{Ideal CPI} + \text{Pipeline Stall Cycles per Instruction} = 1 + \text{Stall Cycles per Instruction}$$

$$\text{Speedup} = \frac{\text{Pipeline Depth}}{1 + \text{Pipeline Stall Cycles per Instruction}}$$

If there are zero stalls, speedup equals the pipeline depth (e.g., $5\times$ for a 5-stage pipeline). Every stall cycle degrades this ideal speedup.

---

## 3. Data Hazards

Data hazards occur when overlapping instruction execution changes the order of reads and writes to registers.

### 3.1 Three Types of Data Dependencies

| Hazard Type | Full Name | Description | Occurrence in 5-Stage RISC Pipeline |
| :--- | :--- | :--- | :--- |
| **RAW** | **Read After Write** | Instruction $j$ tries to read a register *before* instruction $i$ writes it. | **Very Common** (True data dependency) |
| **WAR** | **Write After Read** | Instruction $j$ tries to write a register *before* instruction $i$ reads it. | **Impossible** (instructions execute in-order) |
| **WAW** | **Write After Write** | Instruction $j$ tries to write a register *before* instruction $i$ writes it. | **Impossible** (instructions write back in-order in WB) |

---

### 3.2 RAW Hazard Example

Consider this instruction sequence:
```assembly
add x1, x2, x3    ; Writes result to x1 in WB stage (Cycle 5)
sub x4, x1, x5    ; Reads x1 in ID stage (Cycle 2) -> HAZARD!
and x6, x1, x7    ; Reads x1 in ID stage (Cycle 3) -> HAZARD!
or  x8, x1, x9    ; Reads x1 in ID stage (Cycle 4) -> NO HAZARD!
xor x10, x1, x11  ; Reads x1 in ID stage (Cycle 5) -> NO HAZARD!
```

```
Cycle:            1     2     3     4     5     6     7
add x1, x2, x3: [IF]  [ID]  [EX]  [MEM] [WB: writes x1]
sub x4, x1, x5:       [IF]  [ID: reads x1] -> RAW Hazard! (Reads old/wrong x1)
and x6, x1, x7:             [IF]  [ID: reads x1] -> RAW Hazard!
or  x8, x1, x9:                   [IF]  [ID: reads x1] -> Safe (Writes 1st half, reads 2nd half)
xor x10, x1, x11:                       [IF]  [ID: reads x1] -> Safe (Reads after WB completes)
```

* `sub` and `and` read `x1` before `add` finishes writing `x1` at Cycle 5.
* `or` is safe because the register file performs **writes in the first half of the cycle and reads in the second half**.

---

### 3.3 Solution 1: Hardware Forwarding (Bypassing)
The result of `add` is already calculated at the end of the **EX stage** (Cycle 3). We do not have to wait until Cycle 5 for it to be written into the register file.

**How Forwarding Works:**
* The hardware routes the ALU result directly from the `EX/MEM` or `MEM/WB` pipeline registers back to the ALU input multiplexers.
* When the control unit detects that the source register of the current instruction matches the destination register of an earlier instruction in the pipeline, it selects the forwarded value instead of reading the register file.

```
       +----+      +----+      +----+      +----+      +----+
       | IF | ---> | ID | ---> | EX | ---> | MEM| ---> | WB |
       +----+      +----+      +----+      +----+      +----+
                                 ^           |           |
                                 |   EX/MEM  |   MEM/WB  |
                                 +-----------+-----------+
                                   Forwarding Paths
```

* Forwarding completely eliminates stalls for back-to-back ALU instructions!

---

### 3.4 Data Hazard Requiring a Stall: The Load-Use Interlock
Forwarding cannot solve all data hazards. Specifically, when an instruction immediately tries to use the result of a **Load** instruction:

```assembly
ld  x1, 0(x2)     ; Data is read from memory at the end of MEM (Cycle 4)
sub x4, x1, x5    ; ALU needs data at the beginning of EX (Cycle 3)
```

```
Cycle:          1     2     3            4            5
ld x1, 0(x2): [IF]  [ID]  [EX]         [MEM: Data ready here]
sub x4, x1:         [IF]  [ID]  [EX: Needs data here!]
                                      <---- Forwarding backward in time is impossible!
```

#### The Solution: Hardware Pipeline Interlock (1-Cycle Bubble)
The processor detects this condition and automatically inserts a **1-cycle stall (bubble)**:

```
Cycle:          1     2     3     4     5     6     7
ld x1, 0(x2): [IF]  [ID]  [EX]  [MEM] [WB]
sub x4, x1:         [IF]  [ID] [STALL][EX]  [MEM] [WB]
                                       ^
                                       | Forwarded from MEM/WB to EX
```

* The stall delays `sub` by 1 cycle, allowing the loaded data to be forwarded from `MEM/WB` to the `EX` stage just in time.

---

## 4. Control Hazards (Branch Hazards)

When a branch instruction executes, it can be:
* **Taken:** The branch condition is true; PC jumps to the target address.
* **Untaken (Not Taken):** The branch condition is false; execution falls through to `PC + 4`.

In the simple 5-stage RISC pipeline, the branch condition and target address are resolved in the **ID stage**. If we simply freeze the pipeline until the branch decision is known, every branch incurs a **1-cycle stall penalty**. A 1-cycle stall on every branch causes a **10% to 30% performance loss**.

---

### 4.1 Four Compile-Time (Static) Branch Handling Schemes

1. **Freeze / Flush the Pipeline:**
   * Hold or delete any instructions fetched after the branch until the branch target is known.
   * *Pros/Cons:* Simple hardware, but suffers the full branch penalty on every branch.
2. **Predicted-Not-Taken (Predict Untaken):**
   * Assume the branch will **not** be taken; continue fetching sequential instructions (`PC + 4`).
   * **If branch is Untaken:** Zero stall penalty! The fetched instruction continues normally.
   * **If branch is Taken:** The incorrectly fetched instruction is squashed (turned into a `nop` / bubble), and instruction fetch restarts from the branch target address (1-cycle penalty).
3. **Predicted-Taken:**
   * Assume the branch will be **taken**.
   * As soon as the target address is calculated in ID, start fetching from the target address.
   * Gives a 1-cycle improvement when the branch is actually taken.
4. **Delayed Branch:**
   * The instruction immediately following the branch (in the **branch delay slot**) is **always executed**, whether the branch is taken or not.
   * The compiler rearranges code to place a useful, independent instruction into the delay slot.
   * *Status:* Common in early RISC chips (MIPS, SPARC), but **omitted in modern RISC-V** because modern deep pipelines and dynamic branch prediction make delayed branches inefficient.

---

### 4.2 Branch Performance Equation

$$\text{Pipeline Stall Cycles from Branches} = \text{Branch Frequency} \times \text{Branch Penalty}$$

$$\text{Pipeline Speedup} = \frac{\text{Pipeline Depth}}{1 + (\text{Branch Frequency} \times \text{Branch Penalty})}$$

---

### 4.3 Textbook Worked Example: Comparing Static Branch Schemes in a Deep Pipeline

**Problem:** In a deeper pipeline (e.g., MIPS R4000), resolving the branch target takes 3 cycles. The branch penalties for different schemes are:

| Branch Scheme | Penalty: Unconditional | Penalty: Untaken Conditional | Penalty: Taken Conditional |
| :--- | :---: | :---: | :---: |
| **Stall Pipeline** | 2 | 3 | 3 |
| **Predicted Taken** | 2 | 3 | 2 |
| **Predicted Untaken** | 2 | 0 | 3 |

Assume the following program branch frequencies:
* Unconditional branches: **4%** (0.04)
* Conditional branches (Untaken): **6%** (0.06)
* Conditional branches (Taken): **10%** (0.10)
* *(Total branch frequency = 20%)*

Calculate the addition to CPI for each scheme and compare their performance.

---

**Step-by-Step Solution:**

1. **Calculate CPI penalty for Stall Pipeline:**
   $$\text{Stall Penalty} = (0.04 \times 2) + (0.06 \times 3) + (0.10 \times 3) = 0.08 + 0.18 + 0.30 = \mathbf{0.56\text{ cycles/instruction}}$$

2. **Calculate CPI penalty for Predicted Taken:**
   $$\text{Predicted Taken Penalty} = (0.04 \times 2) + (0.06 \times 3) + (0.10 \times 2) = 0.08 + 0.18 + 0.20 = \mathbf{0.46\text{ cycles/instruction}}$$

3. **Calculate CPI penalty for Predicted Untaken:**
   $$\text{Predicted Untaken Penalty} = (0.04 \times 2) + (0.06 \times 0) + (0.10 \times 3) = 0.08 + 0.00 + 0.30 = \mathbf{0.38\text{ cycles/instruction}}$$

4. **Performance Comparison:**
   * Assuming ideal baseline $\text{CPI} = 1.0$:
     * $\text{CPI}_{\text{stall}} = 1 + 0.56 = 1.56$
     * $\text{CPI}_{\text{predicted untaken}} = 1 + 0.38 = 1.38$
   * **Relative Speedup:**
     $$\text{Speedup} = \frac{1.56}{1.38} \approx \mathbf{1.13\times}$$
   * The **Predicted-Untaken** scheme is **$1.13\times$ faster** than simply stalling the pipeline.

---

## 5. Dynamic Branch Prediction

As pipelines grew deeper (8 to 20+ stages), branch penalties became too expensive for static prediction. Processors switched to **Dynamic Branch Prediction**, where hardware tracks past branch outcomes during runtime to predict future outcomes.

### 5.1 Branch-Prediction Buffer (Branch History Table - BHT)
* A small memory cache indexed by the lower-order bits of the branch instruction’s address (PC).
* It stores prediction bits indicating whether the branch was recently taken or not.
* During the **IF stage**, the processor looks up the PC in the buffer to immediately predict the next instruction fetch direction.

---

### 5.2 1-Bit Predictor vs. 2-Bit Saturating Counter

* **Weakness of a 1-Bit Predictor:**
  * Uses a single bit (1 = Taken, 0 = Not Taken).
  * In a loop that executes 9 times and exits once, a 1-bit predictor mispredicts **twice**:
    1. On the last iteration when the loop exits (bit flips to Not Taken).
    2. On the very first iteration when the loop starts again (bit flips back to Taken).

* **The 2-Bit Saturating Counter Predictor (Solution):**
  * A prediction must **miss twice** before the prediction changes.
  * Uses 4 states:

```
               Taken                     Taken
        +------------------+      +------------------+
        |                  |      |                  |
        v                  |      v                  |
   +----------+  Taken   +----------+  Taken   +----------+  Taken   +----------+
   | Strongly | -------> |  Weakly  | -------> |  Weakly  | -------> | Strongly |
   | Not Taken|          | Not Taken|          |  Taken   |          |  Taken   |
   |   (00)   | <------- |   (01)   | <------- |   (10)   | <------- |   (11)   |
   +----------+ Not Taken+----------+ Not Taken+----------+ Not Taken+----------+
        |                  ^      |                  ^
        |                  |      |                  |
        +------------------+      +------------------+
             Not Taken                 Not Taken
```

* **States 10 and 11:** Predict **Taken**.
* **States 00 and 01:** Predict **Not Taken**.
* Studies show that **2-bit predictors perform almost as well as general $n$-bit counters**, so almost all real processors use 2-bit counters.

---

### 5.3 Prediction Accuracy on Real Benchmarks (SPEC)
* A buffer with **4,096 (4K) entries** using 2-bit counters achieves **82% to 99% accuracy** (misprediction rate of only 1% to 18%).
* **Floating-point scientific code:** Branches are mostly loop iterations $\implies$ **~4% misprediction rate**.
* **Integer code:** Contains complex `if-else` control logic $\implies$ **~11% misprediction rate**.
* **Buffer Size Limit:** A 4K-entry buffer performs almost identically to an **infinite buffer**; increasing buffer size alone yields diminishing returns.

# Unit 1 — Part 11: Overcoming Data Hazards with Dynamic Scheduling and Tomasulo's Algorithm

---

## 1. Dynamic Scheduling: Motivation and Core Concepts

### 1.1 Limitations of Static Scheduling
In a simple statically scheduled in-order pipeline (like the classic 5-stage RISC pipeline):
* Instructions are **fetched, decoded, and issued in strict program order**.
* If an instruction encounters a data hazard (a dependency that cannot be resolved via operand forwarding/bypassing), the hardware **stalls the entire pipeline**.
* No subsequent instruction can issue or execute, even if it is completely independent and could safely run.

#### Motivating Example:
```riscv
fdiv.d f0,  f2,  f4     # Long-latency divide operation (e.g., 12 cycles)
fadd.d f10, f0,  f8     # Dependent on f0 (stalls waiting for fdiv.d)
fsub.d f12, f8,  f14    # Completely independent of f0 and fdiv.d!
```
* In an in-order pipeline, `fsub.d` is blocked from executing simply because `fadd.d` is stalled behind `fdiv.d`.
* Functional units remain idle, and available Instruction-Level Parallelism (ILP) is wasted.

---

### 1.2 What is Dynamic Scheduling?
**Dynamic scheduling** is a hardware technique where the processor hardware dynamically reorders instruction execution at runtime to eliminate pipeline stalls while strictly preserving data flow and exception behavior.

#### Key Advantages:
1. **Tolerates Unpredictable Delays (Cache Misses):** Dynamic scheduling allows independent instructions to execute while the processor waits for long, unpredictable memory delays such as cache misses.
2. **Handles Compile-Time Unknowns:** Resolves dependences that cannot be determined at compile time (e.g., pointer aliasing, memory addresses, data-dependent branches, and dynamic method dispatch).
3. **Hardware Portability / Binary Compatibility:** Software compiled for one specific microarchitecture can run efficiently on different processor implementations without requiring recompilation (critical for mass-market commercial binaries).

---

### 1.3 In-Order Issue vs. Out-of-Order Execution vs. Out-of-Order Completion
To support dynamic scheduling, the classic Instruction Decode (ID) pipeline stage is split into two distinct steps:
1. **Issue (Dispatch):** Decodes instructions and checks for **structural hazards**. Instructions are issued in **strict program order (In-Order Issue)**.
2. **Read Operands:** Waits until all data hazards are cleared, then reads source operands. Instructions enter execution as soon as their data operands become available (**Out-of-Order Execution**).

Because instructions take different numbers of cycles to execute and begin execution at different times, they finish at different times (**Out-of-Order Completion**).

```
[ Instruction Fetch ]
         │
         ▼
[ Stage 1: Issue ]           <-- IN-ORDER (checks structural hazards)
         │
         ▼
[ Stage 2: Read Operands ]   <-- OUT-OF-ORDER (waits for data operands)
         │
         ▼
[ Execution Units ]          <-- OUT-OF-ORDER EXECUTION
         │
         ▼
[ Write Result ]             <-- OUT-OF-ORDER COMPLETION
```

---

### 1.4 New Hazards Introduced by Out-of-Order Execution
In the simple 5-stage integer pipeline, only RAW hazards were possible. By allowing out-of-order execution and completion, two new hazard types emerge:

1. **WAR Hazards (Write After Read / Anti-dependence):**
   * Occurs when an instruction attempts to overwrite a register before an earlier instruction has finished reading it.
   * *Example:*
     ```riscv
     fdiv.d f0, f2,  f4      # Long divide
     fmul.d f6, f0,  f8      # Needs f0; waits for fdiv.d
     fadd.d f0, f10, f14     # Overwrites f0!
     ```
     If `fadd.d` completes and writes to `f0` before `fmul.d` reads `f0`, `fmul.d` will incorrectly read the new value rather than the old value.
2. **WAW Hazards (Write After Write / Output dependence):**
   * Occurs when two instructions write to the same register, and the second instruction finishes and updates the register before the first instruction finishes.
   * *Example:* In the sequence above, both `fdiv.d` and `fadd.d` write to `f0`. If `fadd.d` finishes first, `fdiv.d` must not overwrite `f0` later with its stale result.

**Solution:** Both WAR and WAW hazards are **name dependences** (they involve register names, not fundamental data flow). They are completely eliminated using **Hardware Register Renaming**.

---

### 1.5 Exception Handling in Out-of-Order Pipelines (Imprecise Exceptions)
Out-of-order completion complicates exception handling. While the processor must preserve exception behavior (only raising exceptions that strict program order would encounter), it can produce **imprecise exceptions**:

An exception is **imprecise** if the architectural state of the CPU when the exception occurs does not match a sequential execution model:
* **Condition 1:** The processor may have already executed and committed instructions that appear *later* in program order than the faulting instruction.
* **Condition 2:** The processor may have not yet completed instructions that appear *earlier* in program order than the faulting instruction.

*Imprecise exceptions make it extremely difficult to save process state, fix the error, and restart execution cleanly.* Modern dynamically scheduled architectures solve this using speculative reorder buffers.

---

## 2. Dynamic Scheduling Approaches: Scoreboard vs. Tomasulo

| Feature | CDC 6600 Scoreboard | IBM 360/91 Tomasulo’s Algorithm |
| :--- | :--- | :--- |
| **Pioneering Machine** | CDC 6600 (Seymour Cray, 1964) | IBM 360/91 (Robert Tomasulo, 1967) |
| **Hazard Detection** | Centralized in a single scoreboard unit | **Distributed** across individual Reservation Stations |
| **WAR Hazards** | Stalls instruction before write-back | **Eliminated** via register renaming & operand buffering |
| **WAW Hazards** | Stalls instruction at issue stage | **Eliminated** via dynamic register renaming |
| **Data Forwarding** | Centralized via register file buses | **Broadcast directly** via Common Data Bus (CDB) |
| **Speculation Support** | Difficult to extend | Naturally extensible with Reorder Buffers |

---

## 3. Tomasulo’s Algorithm Architecture

Robert Tomasulo designed this algorithm to achieve high floating-point performance on the IBM System/360 Model 91, which was constrained by having only **four double-precision FP registers** and long memory/FP execution latencies.

### 3.1 Hardware Organization
```
                  ┌───────────────────────────────┐
                  │    Instruction Queue (FIFO)   │
                  └───────────────┬───────────────┘
                                  │ Issue (In-Order)
                                  ▼
         ┌─────────────────────────────────────────────────┐
         │             FP Registers (Regs[])               │
         │          with Qi Register Status Tags           │
         └────────────────────────┬────────────────────────┘
                                  │ Source values / tags
                 ┌────────────────┴────────────────┐
                 ▼                                 ▼
       ┌────────────────────┐            ┌────────────────────┐
       │Reservation Stations│            │Reservation Stations│
       │    (FP Add/Sub)    │            │    (FP Mult/Div)   │
       │    Add1, Add2, Add3│            │     Mult1, Mult2   │
       └─────────┬──────────┘            └─────────┬──────────┘
                 │                                 │
                 ▼                                 ▼
       ┌────────────────────┐            ┌────────────────────┐
       │   FP Adder Unit    │            │ FP Multiplier Unit │
       └─────────┬──────────┘            └─────────┬──────────┘
                 │                                 │
                 └────────────────┬────────────────┘
                                  │
                                  ▼
        ═════════════════════════════════════════════════════
                      COMMON DATA BUS (CDB)
        ═════════════════════════════════════════════════════
                 ▲                                 ▲
                 │                                 │
       ┌─────────┴──────────┐            ┌─────────┴──────────┐
       │    Load Buffers    │            │   Store Buffers    │
       │    Load1, Load2    │            │    Store1, Store2  │
       └─────────▲──────────┘            └─────────▲──────────┘
                 │                                 │
                 └────────────────┬────────────────┘
                                  ▼
                            Memory Unit
```

---

### 3.2 Data Structures and Buffer Fields

#### 1. Reservation Stations (Associated with Functional Units)
Each reservation station entry contains 7 hardware fields:
* **`Op`**: The operation to perform (e.g., `ADD`, `SUB`, `MUL`, `DIV`).
* **`Vj`, `Vk`**: The **actual values** of the source operands.
* **`Qj`, `Qk`**: The **tags (names)** of the reservation stations that will produce the source operands. A value of `0` indicates that the operand is already available in `Vj` or `Vk`.
  *(Note: For any operand, exactly one of $V$ or $Q$ is valid).*
* **`A`**: Address field. Holds the immediate offset initially; holds the computed effective memory address after execution step 1.
* **`Busy`**: 1-bit flag indicating whether the reservation station is currently occupied (`Yes` / `No`).

#### 2. Register Status Table (`RegisterStat[]`)
Contains an entry for every architectural register (e.g., `f0` to `f31`):
* **`Qi`**: The reservation station tag whose upcoming result will be written into this register. If `Qi == 0` (or blank), no active instruction is writing to this register; the current valid value resides in `Regs[]`.

#### 3. Common Data Bus (CDB)
* A high-speed broadcast bus connected to all reservation stations, store buffers, and the register file.
* Carries both the **Result Data** and the producing reservation station’s **Tag**.
* When a result appears on the CDB, all units waiting for that tag capture the data simultaneously in a single cycle.

---

## 4. The Three Pipeline Steps of Tomasulo’s Algorithm

### Step 1: Issue (Dispatch)
* Retrieve the next instruction from the head of the FIFO Instruction Queue.
* **Structural Hazard Check:** Check if a reservation station or buffer $r$ of the requested type is free.
  * If no station is free $\rightarrow$ **Stall issue** (structural hazard).
  * If station $r$ is free $\rightarrow$ **Issue instruction to station $r$**:
    * **Operand Renaming ($rs$):**
      * If `RegisterStat[rs].Qi != 0` $\rightarrow$ set `RS[r].Qj = RegisterStat[rs].Qi` (operand not yet ready).
      * Else $\rightarrow$ set `RS[r].Vj = Regs[rs]` and `RS[r].Qj = 0` (operand ready).
    * **Operand Renaming ($rt$):**
      * If `RegisterStat[rt].Qi != 0` $\rightarrow$ set `RS[r].Qk = RegisterStat[rt].Qi`.
      * Else $\rightarrow$ set `RS[r].Vk = Regs[rt]` and `RS[r].Qk = 0`.
    * **Destination Register Renaming ($rd$):**
      * Set `RS[r].Busy = Yes`.
      * Set `RegisterStat[rd].Qi = r`. *(This renames register $rd$ to reservation station $r$, eliminating WAR and WAW hazards!)*

---

### Step 2: Execute
* If one or more source operands are not yet ready (`Qj != 0` or `Qk != 0`):
  * The reservation station monitors (snoops) the **Common Data Bus (CDB)**.
  * When a matching tag broadcasts its result on the CDB, copy the data value into `Vj` or `Vk` and clear the corresponding $Q$ field to `0`.
* When **both operands are available** (`Qj == 0` and `Qk == 0`):
  * The functional unit begins executing the instruction.
  * By delaying execution until all operands are valid, **RAW hazards are completely prevented**.
* **Loads and Stores (Two-Step Process):**
  1. *Step 1:* Compute effective memory address when base register is ready: `RS[r].A = RS[r].Vj + RS[r].A`.
  2. *Step 2 (Load):* Access memory data as soon as the memory unit is available. Stores wait in the store buffer for their data value before writing to memory.

---

### Step 3: Write Result
* When the functional unit finishes execution and the CDB is free:
  * Broadcast the `result` and the station identifier `r` across the **CDB**.
* **Update Waiting Units (Broadcast Forwarding):**
  * **Registers:** For every register $x$, if `RegisterStat[x].Qi == r`:
    $$\text{Regs}[x] = \text{result}, \quad \text{RegisterStat}[x].Qi = 0$$
  * **Reservation Stations:** For every station $x$:
    * If `RS[x].Qj == r`: `RS[x].Vj = result`, `RS[x].Qj = 0`.
    * If `RS[x].Qk == r`: `RS[x].Vk = result`, `RS[x].Qk = 0`.
  * **Store Buffers:** If store data tag matches `r`, write `result` into `Vk` and clear `Qk`.
* Mark the reservation station as available: `RS[r].Busy = No`.

---

## 5. Worked Examples: Step-by-Step Table Execution

### Assumed Pipeline Execution Latencies
* **Load:** 1 clock cycle
* **Add / Sub:** 2 clock cycles
* **Multiply:** 6 clock cycles
* **Divide:** 12 clock cycles

---

### 5.1 Linear Code Example
Consider the execution of the following sequence:
```riscv
1. fld    f6, 32(x2)
2. fld    f2, 44(x3)
3. fmul.d f0, f2, f4
4. fsub.d f8, f2, f6
5. fdiv.d f0, f0, f6
6. fadd.d f6, f8, f2
```

---

#### Snapshot 1: All Instructions Issued, Only First Load Completed (Figure 3.11)
At this moment, instruction 1 (`fld f6, 32(x2)`) has finished execution and written its result to the CDB. The second load has computed its address but is still accessing memory.

##### Instruction Status Table:
| Instruction | Issue | Execute | Write Result |
| :--- | :---: | :---: | :---: |
| `fld f6, 32(x2)` | $\checkmark$ | $\checkmark$ | $\checkmark$ |
| `fld f2, 44(x3)` | $\checkmark$ | $\checkmark$ | |
| `fmul.d f0, f2, f4` | $\checkmark$ | | |
| `fsub.d f8, f2, f6` | $\checkmark$ | | |
| `fdiv.d f0, f0, f6` | $\checkmark$ | | |
| `fadd.d f6, f8, f2` | $\checkmark$ | | |

##### Reservation Stations:
| Name | Busy | Op | $V_j$ | $V_k$ | $Q_j$ | $Q_k$ | $A$ |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **Load1** | No | | | | | | |
| **Load2** | Yes | Load | | | | | $44 + \text{Regs}[x3]$ |
| **Add1** | Yes | SUB | | $\text{Mem}[32 + \text{Regs}[x2]]$ | Load2 | 0 | |
| **Add2** | Yes | ADD | | | Add1 | Load2 | |
| **Add3** | No | | | | | | |
| **Mult1** | Yes | MUL | | $\text{Regs}[f4]$ | Load2 | 0 | |
| **Mult2** | Yes | DIV | | $\text{Mem}[32 + \text{Regs}[x2]]$ | Mult1 | 0 | |

##### Register Status (`Qi`):
| Field | `f0` | `f2` | `f4` | `f6` | `f8` | `f10` | `f12` |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **$Q_i$** | **Mult2** | **Load2** | | **Add2** | **Add1** | | |

##### Critical Observations on Hazard Elimination:
1. **WAR Elimination on `f6`:**
   * Instruction 5 (`fdiv.d`) reads `f6`, while instruction 6 (`fadd.d`) writes `f6`.
   * Because `fld f6` already wrote its result, `Mult2` already copied the data directly into its $V_k$ field (`Mem[32+Regs[x2]]`).
   * When `fadd.d` finishes and writes its result into `f6`, it **will not corrupt** the value being used by `fdiv.d`.
2. **WAW Elimination on `f0`:**
   * Both `fmul.d` and `fdiv.d` target `f0`.
   * When `fdiv.d` issued, it overwrote `RegisterStat[f0].Qi` from `Mult1` to `Mult2`.
   * When `fmul.d` finishes, the CDB broadcast will check `RegisterStat[f0]`, see that it points to `Mult2`, and **refuse to update register `f0`**, preserving the correct subsequent write!

---

#### Snapshot 2: When `fmul.d` is Ready to Write Result (Figure 3.12)
`fld 2`, `fsub.d`, and `fadd.d` have completed and retired. Only `fmul.d` and `fdiv.d` remain uncompleted.

##### Instruction Status Table:
| Instruction | Issue | Execute | Write Result |
| :--- | :---: | :---: | :---: |
| `fld f6, 32(x2)` | $\checkmark$ | $\checkmark$ | $\checkmark$ |
| `fld f2, 44(x3)` | $\checkmark$ | $\checkmark$ | $\checkmark$ |
| `fmul.d f0, f2, f4` | $\checkmark$ | $\checkmark$ | |
| `fsub.d f8, f2, f6` | $\checkmark$ | $\checkmark$ | $\checkmark$ |
| `fdiv.d f0, f0, f6` | $\checkmark$ | | |
| `fadd.d f6, f8, f2` | $\checkmark$ | $\checkmark$ | $\checkmark$ |

##### Reservation Stations:
| Name | Busy | Op | $V_j$ | $V_k$ | $Q_j$ | $Q_k$ | $A$ |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **Load1** | No | | | | | | |
| **Load2** | No | | | | | | |
| **Add1** | No | | | | | | |
| **Add2** | No | | | | | | |
| **Add3** | No | | | | | | |
| **Mult1** | Yes | MUL | $\text{Mem}[44 + \text{Regs}[x3]]$ | $\text{Regs}[f4]$ | 0 | 0 | |
| **Mult2** | Yes | DIV | | $\text{Mem}[32 + \text{Regs}[x2]]$ | Mult1 | 0 | |

##### Register Status (`Qi`):
| Field | `f0` | `f2` | `f4` | `f6` | `f8` | `f10` |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: |
| **$Q_i$** | **Mult2** | | | | | |

* `fadd.d` completed and updated `f6` in the register file long before `fdiv.d` even started execution, with zero hazard penalties!

---

### 5.2 Dynamic Loop Unrolling in Hardware
Consider a loop multiplying array elements by scalar `f2`:
```riscv
Loop: fld    f0, 0(x1)
      fmul.d f4, f0, f2
      fsd    f4, 0(x1)
      addi   x1, x1, -8
      bne    x1, x2, Loop
```

If branch prediction predicts the branch as **Taken**, Tomasulo's hardware dynamically unrolls the loop across iterations without any compiler intervention.

#### State with Two Active Iterations Issued (Figure 3.14):
* Iteration 1 uses: `Load1`, `Mult1`, `Store1`.
* Iteration 2 uses: `Load2`, `Mult2`, `Store2`.

##### Reservation Stations:
| Name | Busy | Op | $V_j$ | $V_k$ | $Q_j$ | $Q_k$ | $A$ |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **Load1** | Yes | Load | | | | | $\text{Regs}[x1] + 0$ |
| **Load2** | Yes | Load | | | | | $\text{Regs}[x1] - 8$ |
| **Mult1** | Yes | MUL | $\text{Regs}[f2]$ | | 0 | Load1 | |
| **Mult2** | Yes | MUL | $\text{Regs}[f2]$ | | 0 | Load2 | |
| **Store1** | Yes | Store | $\text{Regs}[x1]$ | | 0 | Mult1 | |
| **Store2** | Yes | Store | $\text{Regs}[x1] - 8$ | | 0 | Mult2 | |

##### Register Status (`Qi`):
| Field | `f0` | `f2` | `f4` |
| :--- | :---: | :---: | :---: |
| **$Q_i$** | **Load2** | | **Mult2** |

* The hardware transparently renamed `f0` and `f4` for Iteration 1 to `Load1`/`Mult1`, and for Iteration 2 to `Load2`/`Mult2`.
* Both iterations execute simultaneously and overlap, sustaining a CPI close to 1.0!

---

## 6. Memory Hazards in Dynamic Scheduling

Because memory addresses are dynamic (computed at runtime), registers and memory cannot be renamed identically.

### 6.1 Potential Memory Hazards
* **RAW Hazard:** A Load executed after a Store to the *same memory address*. Interchanging them causes the load to read stale data.
* **WAR Hazard:** A Store executed after a Load to the *same memory address*. Interchanging them causes the load to read future data prematurely.
* **WAW Hazard:** Two Stores executed out-of-order to the *same memory address*.

---

### 6.2 Hardware Handling Mechanism
1. **Effective Address Calculation in Program Order:** The processor preserves the relative program order of address calculations across loads and stores.
2. **Load Disambiguation:** 
   * When a load computes its effective address, it inspects the $A$ fields of all active entries in the **Store Buffer**.
   * If the load address matches an active store buffer address, the load stalls until the conflicting store completes (or uses **Store Forwarding** to bypass data directly from the store buffer).
3. **Store Disambiguation:** 
   * A store checks both active **Load Buffers** and active **Store Buffers** to ensure it never overtakes an earlier read or write to the same address.

---

## 7. Drawbacks and Historical Resurgence of Tomasulo’s Algorithm

### 7.1 Major Drawbacks
1. **Hardware Complexity:** Every reservation station requires high-speed, associative tag-comparison hardware and complex distributed control logic.
2. **CDB Bottleneck:** A single Common Data Bus limits write-backs to one per cycle. Multiple CDBs multiply associative comparison circuitry at every station.

---

### 7.2 Why Tomasulo's Algorithm Became the Industry Standard
Although dormant for years following the IBM 360/91, Tomasulo's algorithm became the core engine of almost all high-performance superscalar processors (Intel Pentium Pro, Core, AMD Zen, ARM Cortex-A) starting in the 1990s:
1. **The Memory Wall and Cache Miss Toleration:** Caches introduce unpredictable, multi-cycle latency spikes. Dynamic out-of-order execution keeps the CPU busy executing independent instructions while a cache miss is being serviced.
2. **Exploiting ILP in Non-Numeric Code:** Compilers struggle to statically schedule integer and branch-heavy code. Hardware dynamic scheduling with register renaming extracts ILP that static compilers cannot reach.
3. **Mass-Market Binary Compatibility:** Allows software distributed as shrink-wrapped binaries to run at peak speed across multiple processor generations without recompilation.

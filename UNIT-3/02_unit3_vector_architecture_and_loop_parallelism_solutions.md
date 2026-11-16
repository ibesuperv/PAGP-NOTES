# Unit – III: Vector Architectures & Loop-Level Parallelism
## Exam Question Bank Solutions (Part 2)

**Syllabus Mapping:**
* **Vector Architecture:** VMIPS instruction set, vector registers, vector functional units, vector lanes, vector chaining, convoys & chimes, MIPS vs. VMIPS DAXPY.
* **Detecting and Enhancing Loop-Level Parallelism:** Loop-carried dependences, GCD (Greatest Common Divisor) test, dependence analysis between statements.

---

# SECTION A: SHORT QUESTIONS (Unit-3 Only)

---

### Question 1.2 [02 Marks]
> **Write VMIPS code to add two vectors.**

#### Answer:
Assuming vector length register `VL` is set to the number of elements (e.g., 64), vector base addresses for $A$ and $B$ are stored in scalar registers `Ra` and `Rb`, and the destination vector $C$ address is in `Rc`:

```assembly
# VMIPS Assembly to compute C = A + B
LV      V1, Ra          # Load vector A from memory starting at address Ra into V1
LV      V2, Rb          # Load vector B from memory starting at address Rb into V2
ADDVV.D V3, V1, V2      # Add vector elements: V3[i] = V1[i] + V2[i] (double-precision)
SV      V3, Rc          # Store result vector V3 into memory starting at address Rc
```

---

### Question 1.3 [01 Mark]
> **Define Chaining.**

#### Answer:
**Chaining** is a vector hardware pipelining technique that forwards individual result elements from the output of one vector functional unit directly to the input of another vector functional unit as soon as each element is produced, without waiting for the entire vector register (all 64 elements) to complete. It is the vector equivalent of scalar data forwarding.

---

# SECTION B: LONG QUESTIONS (Unit-3 Only)

---

## Question 3.a [06 Marks]
> **How do you use multiple functional units to improve performance of a single vector add instruction `C = A + B`?**

### Answer:

A single vector operation on a 64-element vector does not have to execute element-by-element across 64 sequential clock cycles on a single arithmetic unit. Vector architectures achieve massive speedup by organizing the hardware execution engine into **multiple parallel Vector Lanes**.

```
Single 64-Element Vector Add: C = A + B executed across 4 Vector Lanes

Cycle:           1               2               ...           16
         ┌───────────────┬───────────────┬─────── ... ─┬───────────────┐
Lane 0:  │ C[0]=A[0]+B[0]│ C[4]=A[4]+B[4]│             │C[60]=A[60]+B[60]
         ├───────────────┼───────────────┼─────── ... ─┼───────────────┤
Lane 1:  │ C[1]=A[1]+B[1]│ C[5]=A[5]+B[5]│             │C[61]=A[61]+B[61]
         ├───────────────┼───────────────┼─────── ... ─┼───────────────┤
Lane 2:  │ C[2]=A[2]+B[2]│ C[6]=A[6]+B[6]│             │C[62]=A[62]+B[62]
         ├───────────────┼───────────────┼─────── ... ─┼───────────────┤
Lane 3:  │ C[3]=A[3]+B[3]│ C[7]=A[7]+B[7]│             │C[63]=A[63]+B[63]
         └───────────────┴───────────────┴─────── ... ─┴───────────────┘
```

### 1. Vector Lanes Structure
* Instead of having a single 64-bit adder, the processor replicates functional units into $L$ identical **lanes** (e.g., $L = 4$).
* Each lane contains:
  1. A slice of the vector register file (holding $\frac{1}{L}$ of the vector elements).
  2. A local pipelined vector arithmetic unit (e.g., an FP adder).
* Element distribution across $L$ lanes follows **interleaved striping**:
  * Lane $0$ holds elements $0, L, 2L, \dots$ (e.g., $0, 4, 8, \dots, 60$)
  * Lane $1$ holds elements $1, L+1, 2L+1, \dots$ (e.g., $1, 5, 9, \dots, 61$)
  * Lane $2$ holds elements $2, L+2, 2L+2, \dots$ (e.g., $2, 6, 10, \dots, 62$)
  * Lane $3$ holds elements $3, L+3, 2L+3, \dots$ (e.g., $3, 7, 11, \dots, 63$)

### 2. Execution Time Improvement
For a vector of length $\text{MVL} = 64$:
* **Single Functional Unit ($1$ lane):**
  $$\text{Execution Time} = 64\text{ clock cycles}$$
* **Multiple Functional Units ($L = 4$ lanes):**
  $$\text{Execution Time} = \frac{\text{MVL}}{L} = \frac{64}{4} = \mathbf{16\text{ clock cycles}}$$
* **Four-fold ($4\times$) speedup** is achieved for the single vector instruction with zero changes to the programmer's assembly code or binary instruction format!

---

## Question 3.b [04 Marks]
> **Consider the loop:**
> ```c
> for (i = 0; i < 100; i = i + 1) {
>     A[i + 1] = A[i] + C[i];       /* S1 */
>     B[i + 1] = B[i] + A[i + 1];   /* S2 */
> }
> ```
> **Assume that $A, B, C$ are distinct, non-overlapping arrays. What are the data dependences among statements $S1$ and $S2$ in the loop?**

### Answer:

Let us analyze the read and write sets for statements $S1$ and $S2$ across iterations:

```
Iteration i:
  S1: Writes A[i+1], Reads A[i], Reads C[i]
  S2: Writes B[i+1], Reads B[i], Reads A[i+1]
```

### 1. Dependences Within the Same Iteration (Non-Loop-Carried)
* **True Data Dependence (RAW) from $S1$ to $S2$:**
  * Statement $S1$ computes and writes into `A[i+1]`.
  * Statement $S2$ reads `A[i+1]` in the *very same iteration* $i$.
  * **Type:** Non-loop-carried true data dependence ($S1 \xrightarrow{\text{RAW}} S2$).
  * **Implication:** In any iteration $i$, $S1$ must complete before $S2$ can execute.

### 2. Dependences Across Different Iterations (Loop-Carried)
* **True Data Dependence (RAW) from $S1$ to $S1$ on array $A$:**
  * Statement $S1$ in iteration $i$ computes and writes `A[i+1]`.
  * In the next iteration $i+1$, statement $S1$ reads `A[i+1]` (which is `A[i]` for that iteration).
  * **Type:** **Loop-carried true data dependence** ($S1 \xrightarrow[\text{loop-carried}]{\text{RAW}} S1$).
  * **Implication:** Iteration $i+1$ of $S1$ cannot begin until iteration $i$ of $S1$ finishes!
* **True Data Dependence (RAW) from $S2$ to $S2$ on array $B$:**
  * Statement $S2$ in iteration $i$ writes `B[i+1]`.
  * In the next iteration $i+1$, statement $S2$ reads `B[i+1]` (which is `B[i]` for that iteration).
  * **Type:** **Loop-carried true data dependence** ($S2 \xrightarrow[\text{loop-carried}]{\text{RAW}} S2$).
* **True Data Dependence (RAW) from $S1$ to $S2$ across iterations:**
  * Statement $S1$ produces `A[i+1]`, which is read by $S2$ in iteration $i$. (Non-loop-carried).

### Summary Table of Dependences:
| Source Statement | Destination Statement | Variable / Array | Dependence Type | Loop-Carried? |
| :---: | :---: | :---: | :---: | :---: |
| **$S1$ (iteration $i$)** | **$S2$ (iteration $i$)** | `A[i+1]` | True Data Dependence (RAW) | **No** (intra-iteration) |
| **$S1$ (iteration $i$)** | **$S1$ (iteration $i+1$)**| `A[i+1] -> A[i]` | True Data Dependence (RAW) | **Yes** (loop-carried) |
| **$S2$ (iteration $i$)** | **$S2$ (iteration $i+1$)**| `B[i+1] -> B[i]` | True Data Dependence (RAW) | **Yes** (loop-carried) |

*Because of the loop-carried dependences on both $A$ and $B$, this loop **cannot be directly vectorized** in its present form.*

---

## Question 3.c [06 Marks]
> **Show the code for MIPS and VMIPS for DAXPY loop ($Y = a \times X + Y$). Assume that starting address of $X$ and $Y$ are in $Rx$ and $Ry$ respectively.**

### 1. Scalar MIPS Assembly Code (Iterates 64 times)
Assume $F0$ contains scalar constant $a$, $Rx$ and $Ry$ point to double-precision arrays, and $R1$ contains the upper limit address (`Rx + 512` bytes):

```assembly
# Scalar MIPS DAXPY Loop (Processes 1 element per iteration)
Loop:
    L.D    F2, 0(Rx)        # Load X[i]
    MUL.D  F2, F2, F0       # a * X[i]
    L.D    F4, 0(Ry)        # Load Y[i]
    ADD.D  F4, F4, F2       # a * X[i] + Y[i]
    S.D    F4, 0(Ry)        # Store back into Y[i]
    DADDIU Rx, Rx, #8       # Increment X pointer by 8 bytes
    DADDIU Ry, Ry, #8       # Increment Y pointer by 8 bytes
    DSUBU  R2, R1, Rx       # Check if Rx reached limit
    BNEZ   R2, Loop         # Branch if not done
```
* **Dynamic Instruction Count:** For 64 elements, $64 \times 9 = \mathbf{576\text{ instructions}}$ executed.

---

### 2. Vector VMIPS Assembly Code (Processes 64 elements at once)
Assume $F0$ contains scalar $a$, and vector length register `VL` is set to 64:

```assembly
# Vector VMIPS DAXPY Code (Processes 64 elements in ONE go)
LV      V1, Rx          # Load vector X (64 double-precision words) into V1
MULVS.D V2, V1, F0      # Vector-Scalar Multiply: V2[i] = V1[i] * F0
LV      V3, Ry          # Load vector Y (64 double-precision words) into V3
ADDVV.D V4, V2, V3      # Vector-Vector Add: V4[i] = V2[i] + V3[i]
SV      V4, Ry          # Store vector V4 back into memory at Ry
```
* **Dynamic Instruction Count:** Exactly **$5\text{ vector instructions}$** (down from 576 scalar instructions)!
* **Loop Overhead:** Zero branches, zero index calculations.

---

## Question 4.a [04 Marks]
> **Use GCD test to determine whether dependencies exist in the following loop:**
> ```c
> for (i = 0; i < 100; i = i + 1) {
>     x[2 * i + 3] = x[2 * i] * 5.0;
> }
> ```

### Answer:

### 1. GCD Test Formulation
Suppose a loop accesses an array element `x[a * i + b]` on a write (store) and `x[c * i + d]` on a read (load):
* Store index expression: $a \cdot i + b$
* Load index expression: $c \cdot i + d$

A loop-carried data dependence exists between two different iterations $i$ and $j$ ($i \ne j$) only if:
$$a \cdot i + b = c \cdot j + d \implies a \cdot i - c \cdot j = d - b$$

**The GCD Test Condition:**  
A linear Diophantine equation $a \cdot i - c \cdot j = d - b$ has an integer solution if and only if:
$$\mathbf{\gcd(a, c) \text{ divides } (d - b)}$$

If $\gcd(a, c)$ does **not** divide $(d - b)$, **no dependence can possibly exist**, and the loop is guaranteed to be parallelizable!

---

### 2. Step-by-Step Calculation:
From the given loop:
* Write index: `x[2 * i + 3]` $\implies a = 2, \quad b = 3$
* Read index: `x[2 * i + 0]` $\implies c = 2, \quad d = 0$

1. **Calculate $\gcd(a, c)$:**
   $$\gcd(2, 2) = \mathbf{2}$$

2. **Calculate $(d - b)$:**
   $$d - b = 0 - 3 = \mathbf{-3}$$

3. **Check Divisibility:**
   $$\frac{d - b}{\gcd(a, c)} = \frac{-3}{2} = -1.5 \quad (\text{Not an integer})$$
   $2$ does **NOT** divide $-3$.

### Conclusion:
Since $\gcd(2, 2) = 2$ does not divide $-3$, there are **no integer solutions** where the store and load access the same memory location.
Therefore, **NO loop-carried dependence exists**, and the loop **can be safely parallelized/vectorized**.

---

## Question 4.b [04 Marks]
> **Show how the following sequence lays out in convoys, assuming a single copy of each vector functional unit:**
> ```assembly
> LV      V1, Rx
> MULVS.D V2, V1, F0
> LV      V3, Ry
> ADDVV.D V4, V2, V3
> SV      V4, Ry
> ```
> **How many chimes will this vector sequence take? How many cycles per FLOP are needed (ignore vector instruction overhead)?**

---

### 1. Convoy Identification Rules
* A **convoy** is a set of vector instructions that could potentially execute together in parallel without structural conflicts.
* **Constraints for grouping into a convoy:**
  1. Instructions within the same convoy must not demand the same functional unit (structural hazard limit: only 1 vector load/store unit is available in standard VMIPS).
  2. With vector register **chaining**, instructions with RAW data dependences can be in the same convoy.
  3. A second memory instruction (`LV` or `SV`) cannot be in the same convoy as an earlier memory instruction because there is only **one vector memory port**.

---

### 2. Convoy Breakdown:

* **Convoy 1:**
  ```assembly
  LV      V1, Rx          # Uses Vector Load/Store unit
  MULVS.D V2, V1, F0      # Uses FP Multiplier (Chained to LV!)
  ```
  *(Can execute together because they use different functional units and are chained).*

* **Convoy 2:**
  ```assembly
  LV      V3, Ry          # Uses Vector Load/Store unit (new memory access)
  ADDVV.D V4, V2, V3      # Uses FP Adder (Chained to LV and MULVS.D)
  ```
  *(Cannot be placed in Convoy 1 because Convoy 1 already uses the single load/store unit for `LV V1`).*

* **Convoy 3:**
  ```assembly
  SV      V4, Ry          # Uses Vector Load/Store unit
  ```
  *(Cannot be in Convoy 2 because Convoy 2 already uses the single load/store unit for `LV V3`).*

---

### 3. Total Chimes and Cycles Per FLOP Calculation:
* **Number of Chimes:**
  Each convoy requires **1 chime** (time to process the vector register, $\text{MVL} = 64$ clock cycles).
  $$\text{Total Chimes} = \mathbf{3\text{ chimes}}$$

* **Total Clock Cycles:**
  $$\text{Total Cycles} = 3\text{ chimes} \times 64 = \mathbf{192\text{ clock cycles}}$$

* **Floating-Point Operations (FLOPs):**
  * `MULVS.D` performs 64 multiplications.
  * `ADDVV.D` performs 64 additions.
  $$\text{Total FLOPs} = 64 + 64 = \mathbf{128\text{ FLOPs}}$$

* **Cycles Per FLOP:**
  $$\text{Cycles Per FLOP} = \frac{\text{Total Cycles}}{\text{Total FLOPs}} = \frac{192\text{ cycles}}{128\text{ FLOPs}} = \frac{3}{2} = \mathbf{1.5\text{ clock cycles per FLOP}}$$

---

## Question 4.c [06 Marks]
> **Illustrate VMIPS architecture with a neat diagram.**

### Architecture Block Diagram:

```
                            Scalar MIPS Processor
                   ┌──────────────────────────────────────┐
                   │    Scalar Registers (R0 - R31)       │
                   │    Scalar FP Registers (F0 - F31)    │
                   │    Scalar Pipelined Execution Core   │
                   └──────────────────┬───────────────────┘
                                      │ Scalar Control & Addresses
                                      ▼
    ══════════════════════════════════════════════════════════════════
                         VECTOR CONTROL & BUSES
    ══════════════════════════════════════════════════════════════════
            │                         ▲                    │
            │ Register Read/Write     │ Chaining Buses     │
            ▼                         │                    ▼
 ┌───────────────────────────────────────┐    ┌─────────────────────────┐
 │        Vector Register File           │    │ Vector Control Registers│
 │  8 Vector Registers (V0 - V7)         │    │ • VLR (Vector Length)   │
 │  Each register holds 64 elements      │    │ • VMR (Vector Mask)     │
 │  Each element is 64-bit double prec.  │    └─────────────────────────┘
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
                    │ Results broadcast / Chaining
                    ▼
    ══════════════════════════════════════════════════════════════════
                       VECTOR LOAD / STORE UNIT
    ══════════════════════════════════════════════════════════════════
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │     Memory Interconnect       │
                    └───────────────┬───────────────┘
                                    ▼
        ┌───────────────┬───────────────┬───────────────┐
        │ Memory Bank 0 │ Memory Bank 1 │ Memory Bank N │
        └───────────────┴───────────────┴───────────────┘
```

### Components of VMIPS Architecture:
1. **Vector Register File:**
   * Contains **8 vector registers ($V0$ to $V7$)**.
   * Each vector register holds **64 elements**, and each element is a **64-bit double-precision word**.
2. **Vector Functional Units:**
   * Fully pipelined execution units for FP Add/Subtract, FP Multiply, FP Divide, and Integer/Logical operations.
   * Capable of producing one result per clock cycle.
3. **Vector Control Registers:**
   * **`VLR` (Vector Length Register):** Configures active vector length when loop iterations are $< 64$.
   * **`VMR` (Vector Mask Register):** 64-bit mask register used for conditional execution (`if-else` within loops).
4. **Vector Load / Store Unit:**
   * Fully pipelined unit that interfaces with an interleaved, multi-bank memory system to load/store entire vector registers (supporting unit stride, non-unit stride, and gather/scatter).


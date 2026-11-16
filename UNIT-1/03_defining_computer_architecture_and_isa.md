# 03: Defining Computer Architecture and Instruction Set Architecture (ISA)

---

## 1. What is Computer Architecture?

In the past, people mistakenly thought "Computer Architecture" meant **only** designing the Instruction Set (ISA), calling everything else just "implementation."

Today, computer architecture covers **three distinct layers**:

```
+-------------------------------------------------------------------+
| 1. Instruction Set Architecture (ISA)                             |
|    Software/Hardware boundary visible to the programmer/compiler  |
+-------------------------------------------------------------------+
                                  |
+-------------------------------------------------------------------+
| 2. Organization / Microarchitecture                               |
|    High-level design (pipeline stages, cache hierarchy, buses)     |
+-------------------------------------------------------------------+
                                  |
+-------------------------------------------------------------------+
| 3. Hardware Implementation                                        |
|    Detailed logic gates, circuit layout, packaging, and cooling   |
+-------------------------------------------------------------------+
```

### The Three Layers Explained:
1. **Instruction Set Architecture (ISA):**
   * The programmer-visible interface between software and hardware.
   * Defines the registers, memory addressing, instruction types, and binary encoding.
   * *Examples:* x86, ARMv8, RISC-V.
2. **Organization (or Microarchitecture):**
   * It is the internal design of the computer that decides how the instructions are executed.
   * It includes things like:
     * **Pipeline** → how instructions are processed step by step.
     * **Out-of-order execution** → doing instructions in a different order when useful.
     * **Cache** → fast memory inside/near the CPU.
     * **Memory controller** → manages communication with main memory.
   * *Example:* **Intel Core i7** and **AMD Opteron** run the exact same x86 ISA, but have completely different internal microarchitectures.
3. **Hardware Implementation:**
   * The actual physical construction of the chip: circuit design, clock frequency, lithography node, silicon packaging, and cooling.
   * *Example:* **Intel Core i7** (desktop) and **Intel Xeon E7** (server) have almost the same microarchitecture, but differ in clock speed, memory channels, reliability features, and packaging.

> **Note on Terminology:** Because almost all modern chips have multiple processors on a single die, the industry now uses the word **core** instead of processor, and the old term **CPU** is used less frequently.

---

## 2. Seven Dimensions of an Instruction Set Architecture (ISA)

When architects design an ISA, they must decide on seven key technical dimensions. We compare **80x86**, **ARMv8**, and **RISC-V**:

```
                          7 Dimensions of an ISA
    +-----------------------------------------------------------------+
    | 1. Class of ISA        (Register-Memory vs. Load-Store)         |
    | 2. Memory Addressing   (Byte addressing, Alignment)             |
    | 3. Addressing Modes    (Immediate, Displacement, Indexed)       |
    | 4. Types of Operands   (8-bit, 16-bit, 32-bit, 64-bit, Floats)  |
    | 5. Operations          (ALU, Data Transfer, Control, FP)        |
    | 6. Control Flow        (PC-relative Branches, Jumps, Calls)     |
    | 7. Binary Encoding     (Fixed 32-bit vs. Variable 1-18 bytes)   |
    +-----------------------------------------------------------------+
```

---

### 2.1 Dimension 1: Class of ISA
Nearly all modern processors are **General-Purpose Register (GPR)** architectures (operands are held in registers or memory). They fall into two models:

* **Register-Memory ISAs (e.g., 80x86):**
  * Instructions can perform arithmetic directly on memory operands.
  * *Example:* `ADD EAX, [EBX]` adds memory contents directly to a register.
  * Has 16 general-purpose registers and 16 floating-point registers.
* **Load-Store ISAs (e.g., ARMv8, RISC-V):**
  * Memory can **only** be accessed using explicit `LOAD` or `STORE` instructions.
  * Arithmetic operations can **only** work on registers.
  * RISC-V has **32 general-purpose registers** (`x0` to `x31`) and **32 floating-point registers** (`f0` to `f31`).
  * **Rule:** Virtually all ISAs designed since 1985 are Load-Store.

---

### 2.2 Dimension 2: Memory Addressing & Alignment
* **Byte Addressing:** 80x86, ARMv8, and RISC-V all use byte addressing (each memory address points to an 8-bit byte).
* **Memory Alignment:**
  * An access to an object of size $s$ bytes at memory address $A$ is **aligned** if:
    $$A \pmod s = 0$$
  * *Example:* A 4-byte word must be located at an address divisible by 4 (e.g., 0, 4, 8, 12).
  * **ARMv8:** Requires memory accesses to be aligned.
  * **80x86 and RISC-V:** Do not strictly require alignment (hardware handles unaligned accesses), but aligned accesses run significantly faster.

---

### 2.3 Dimension 3: Addressing Modes
Addressing modes define how the processor calculates the memory address of an operand:

* **RISC-V Addressing Modes (Lean and Simple):**
  1. **Register:** Operand is inside a register (`x1`).
  2. **Immediate:** Operand is a constant number inside the instruction itself (`addi x1, x2, 5`).
  3. **Displacement:** A constant offset is added to a base register (`lw x1, 8(x2)` $\to$ Address = `x2 + 8`).
* **80x86 Addressing Modes (Complex):**
  * Supports Register, Immediate, Displacement, plus:
  * Absolute (direct memory address with no register).
  * Based indexed with displacement (`Base + Index + Offset`).
  * Scaled index (`Base + Index * Scale + Offset`), where scale matches data size (1, 2, 4, or 8 bytes).
* **ARMv8 Addressing Modes:**
  * Supports RISC-V modes, plus PC-relative, sum of two registers, scaled register offsets, and **autoincrement/autodecrement** (the base register automatically updates after the address is used).

---

### 2.4 Dimension 4: Types and Sizes of Operands
Common operand formats supported by 80x86, ARMv8, and RISC-V:
* **8-bit:** Byte (ASCII characters).
* **16-bit:** Half-word (Unicode characters, 16-bit integers).
* **32-bit:** Word (standard integers).
* **64-bit:** Double-word (long integers, memory pointers).
* **Floating Point (IEEE 754):** 32-bit Single Precision (`float`) and 64-bit Double Precision (`double`).
* *Note:* 80x86 also supports an 80-bit extended precision floating-point format.

---

### 2.5 Dimension 5: Operations
Instructions are grouped into four main operational categories:
1. **Data Transfers:** Moving values between registers and memory (`lw`, `sw`, `ld`, `sd`) or between integer and floating-point registers.
2. **Arithmetic & Logical:** Operations like `add`, `sub`, `mul`, `div`, bitwise `and`, `or`, `xor`, and shifts (`sll`, `srl`).
3. **Control:** Conditional branches, unconditional jumps, and function calls.
4. **Floating Point:** Operations on floating-point registers (`fadd`, `fsub`, `fmul`, `fdiv`, `fsqrt`).

---

### 2.6 Dimension 6: Control Flow Instructions
Control flow instructions change the execution order by changing the **Program Counter (PC)** (e.g., branches, jumps, and function calls).

* **How the Target Address is Found (PC-Relative):**
  * The branch address is calculated by adding an offset to the current PC (`PC + offset`).
* **How Conditions are Checked:**
  * **RISC-V (Simple & Direct):** Compares two registers directly inside the branch instruction (e.g., `beq x1, x2, label`). It does **not** use condition code flags.
  * **80x86 and ARMv8:** Check condition flags (Zero, Carry, Negative, Overflow) that were set as side effects of earlier arithmetic operations.
* **How Function Calls Save the Return Address:**
  * **RISC-V and ARMv8 (Fast):** Store the return address in a **register** (Link Register `ra` / `x1`).
  * **80x86 (Memory Stack):** Pushes the return address onto the **stack in memory** using the `CALL` instruction.

---

### 2.7 Dimension 7: Encoding of an ISA
How instructions are represented as binary numbers:

| Feature | Fixed-Length Encoding (ARMv8, RISC-V) | Variable-Length Encoding (80x86) |
| :--- | :--- | :--- |
| **Instruction Size** | Exactly **32 bits (4 bytes)** for all instructions | Varies from **1 byte to 18 bytes** |
| **Decoding Complexity** | Very simple and fast to decode in hardware | Complex decoder hardware needed to find instruction boundaries |
| **Pipelining** | Easy to pipeline because sizes and fields are uniform | Harder to pipeline smoothly |
| **Code Size** | Programs take more memory bytes | Programs are more compact in memory |
| **Compressed Option** | RISC architectures later added 16-bit compressed extensions (ARM Thumb-2, RISC-V RV64IC) to achieve even smaller binaries than x86 | Native variable sizing |

---

## 3. RISC-V Register Set and Instruction Formats

### 3.1 RISC-V Register Set
RISC-V has **32 Integer Registers** (`x0` – `x31`) of 64-bit width (in RV64) and **32 Floating-Point Registers** (`f0` – `f31`):

* `x0` (`zero`): Hardwired to constant value `0` (writes to `x0` are discarded).
* `x1` (`ra`): Return Address (saved by caller).
* `x2` (`sp`): Stack Pointer (saved by callee).
* `x3` (`gp`): Global Pointer.
* `x4` (`tp`): Thread Pointer.
* `x5 – x7` (`t0 – t2`): Temporary registers.
* `x8` (`s0 / fp`): Saved register / Frame pointer.
* `x9` (`s1`): Saved register.
* `x10 – x11` (`a0 – a1`): Function arguments and return values.
* `x12 – x17` (`a2 – a7`): Function arguments.
* `x18 – x27` (`s2 – s11`): Saved registers.
* `x28 – x31` (`t3 – t6`): Temporary registers.

---

### 3.2 RISC-V Instruction Formats (All 32 bits long)
RISC-V organizes instructions into standard formats so that register specifiers always sit in the same bit positions:

```
R-type: | funct7 (7b) | rs2 (5b) | rs1 (5b) | funct3 (3b) | rd (5b) | opcode (7b) |
I-type: | immediate (12b)        | rs1 (5b) | funct3 (3b) | rd (5b) | opcode (7b) |
S-type: | imm[11:5]   | rs2 (5b) | rs1 (5b) | funct3 (3b) | imm[4:0]| opcode (7b) |
B-type: | imm split   | rs2 (5b) | rs1 (5b) | funct3 (3b) | imm split | opcode (7b) |
U-type: | immediate (20b)                                 | rd (5b) | opcode (7b) |
J-type: | immediate split (20b)                           | rd (5b) | opcode (7b) |
```

* **R-type (Register):** For pure arithmetic between registers (`add rd, rs1, rs2`).
* **I-type (Immediate / Loads):** For arithmetic with constants (`addi`) and memory loads (`lw`, `ld`).
* **S-type (Stores):** For writing registers into memory (`sw`, `sd`). Has separate format to keep `rs1` and `rs2` at fixed positions.
* **B-type (Branches):** For conditional branches (`beq`, `bne`).
* **U-type (Upper Immediate):** For large 20-bit constants (`lui`, `auipc`).
* **J-type (Jumps):** For unconditional jump-and-link (`jal`).

---

## 4. Functional Requirements Architects Must Consider

When computer architects design a machine, they must satisfy critical functional requirements across four areas:

1. **Application Area Needs:**
   * *Personal Mobile Devices (PMD):* Real-time media performance, strict battery energy limits.
   * *Desktop:* Balanced interactive compute and graphics performance.
   * *Servers:* High availability, database transaction throughput, scalability.
   * *Clusters / WSCs:* Throughput for millions of requests, error-correcting memory, power proportionality.
   * *IoT / Embedded:* Minimal cost and power, real-time deadlines.
2. **Software Compatibility:**
   * *Language-level compatibility:* Most flexible for the designer, but needs new compilers.
   * *Binary / Object-code compatibility:* Processor must implement the exact existing ISA (like x86) so existing software runs without modification.
3. **Operating System Support:**
   * Proper virtual memory mechanisms (paging or segmentation).
   * Memory protection between user applications and the OS kernel.
4. **Industry Standards:**
   * IEEE 754 floating-point standard.
   * Standard I/O interfaces (PCI Express, SATA).
   * Standard network protocols (Ethernet, InfiniBand).

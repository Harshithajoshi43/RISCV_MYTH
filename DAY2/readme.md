# Day 2: Introduction to Application Binary Interface (ABI) & Basic Verification Flow

## 📖 Theory Overview

### 1. What is an Application Binary Interface (ABI)?
An **Application Binary Interface (ABI)** acts as a bridge between high-level software (like a C program) and the underlying hardware (RISC-V CPU). It defines standard rules for how functions call each other, how registers are used, and how data is stored in memory so that compiled code runs smoothly on the processor.

---

### 2. RISC-V Registers & ABI Naming Conventions
In RV64 (64-bit RISC-V), `XLEN = 64`, meaning there are **32 general-purpose registers** (`x0` to `x31`), each 64 bits wide. Instead of just using hardware numbers (`x0–x31`), the ABI assigns easy-to-understand roles to each register:

| Register | ABI Name | Usage / Role | Saver |
| :--- | :--- | :--- | :--- |
| **x0** | `zero` | Hard-wired zero (always holds 0) | — |
| **x1** | `ra` | Return address for function calls | Caller |
| **x2** | `sp` | Stack pointer | Callee |
| **x3** | `gp` | Global pointer | — |
| **x4** | `tp` | Thread pointer | — |
| **x5 – x7** | `t0 - t2` | Temporary registers | Caller |
| **x8** | `s0 / fp` | Saved register / Frame pointer | Callee |
| **x9** | `s1` | Saved register | Callee |
| **x10 – x11**| `a0 - a1` | Function arguments / Return values | Caller |
| **x12 – x17**| `a2 - a7` | Function arguments | Caller |
| **x18 – x27**| `s2 - s11` | Saved registers | Callee |
| **x28 – x31**| `t3 - t6` | Temporary registers | Caller |

* **Caller-saved:** The calling function saves these values before making a call if it needs them later.
* **Callee-saved:** The called function must preserve these values and restore them before returning.

---

### 3. Memory Allocation & Data Loading (Little-Endian)
* Memory in RISC-V is byte-addressable, and data is stored in **Little-Endian** format (the Least Significant Byte is stored at the lowest memory address).
* A **doubleword** in RV64 is **64 bits (8 bytes)**, occupying 8 consecutive memory addresses (e.g., addresses `16` through `23`).
* When loading data from memory into a 64-bit register (like `x8`), bytes from higher addresses form the Most Significant Bits (MSB), and bytes from lower addresses form the Least Significant Bits (LSB).

---

### 4. Instruction Formats & Assembly Walkthrough
To understand how instructions interact with memory and registers, consider this 3-step sequence:

1. **Load Doubleword (`ld x8, 16(x23)`)** — *I-Type Format*
   * Reads a 64-bit value from memory starting at address `x23 + 16` and loads it into destination register `x8`.
   * **Fields:** `immediate[11:0]` | `rs1` | `funct3` | `rd` | `opcode`

2. **Add (`add x8, x24, x8`)** — *R-Type Format*
   * Adds the values inside source registers `x24` (`rs1`) and `x8` (`rs2`), then stores the result back into `x8` (`rd`).
   * **Fields:** `funct7` | `rs2` | `rs1` | `funct3` | `rd` | `opcode`

3. **Store Doubleword (`sd x8, 8(x23)`)** — *S-Type Format*
   * Takes the updated 64-bit value from data register `x8` (`rs2`) and writes it back into memory at address `x23 + 8` (using base register `x23` as `rs1`).
   * **Fields:** `immediate[11:5]` | `rs2` | `rs1` | `funct3` | `immediate[4:0]` | `opcode`

---

### 5. Basic Verification Flow (Running C on RISC-V CPU)
To verify that our Verilog RISC-V CPU works properly:
1. **Compile C to Hex:** A C program (such as calculating the sum of numbers from `1 to n`) is compiled into RISC-V machine code and converted into a **Hex file**.
2. **Load into Memory:** The Hex file is loaded into the instruction/data memory module of the testbench.
3. **Simulate in Verilog:** The **RISC-V CPU (written in Verilog)** fetches and executes the instructions from memory and outputs the final result (e.g., displaying the sum).








<img width="1076" height="607" alt="image" src="https://github.com/user-attachments/assets/a2c73b00-e8f9-4392-ae16-0e44b818b1b9" />


# Day 2: Application Binary Interface (ABI) & Basic Verification Flow


### 1. What is an Application Binary Interface (ABI)?
When you write a program in a high-level language like C, the compiler translates it into assembly and machine code so the processor can execute it. The **Application Binary Interface (ABI)** is the set of rules that governs how this compiled software communicates with the underlying RISC-V hardware. 

Specifically, the ABI defines:
* How functions pass arguments and return results.
* Which registers are preserved across function calls and which can be overwritten.
* How data types are sized, aligned, and laid out in memory.

---

### 2. RISC-V Registers & ABI Naming Conventions
In RV64 (64-bit RISC-V), the register width is **`XLEN = 64` bits** (spanning from bit `0` to `XLEN - 1`, which is bit `63`). The architecture has **32 general-purpose registers** (`x0` through `x31`).

Instead of referring to registers only by their hardware numbers (`x0–x31`), the ABI assigns symbolic names and dedicated roles to keep code organized and modular:

| Register | ABI Name | Usage / Role | Saver | Detailed Explanation |
| :--- | :--- | :--- | :--- | :--- |
| **x0** | `zero` | Hard-wired zero | — | Always reads as `0`. Writes to this register are ignored. |
| **x1** | `ra` | Return address | Caller | Stores the memory address to jump back to when a function finishes. |
| **x2** | `sp` | Stack pointer | Callee | Points to the top of the current function's stack frame in memory. |
| **x3** | `gp` | Global pointer | — | Points to the middle of the global data section for fast variable access. |
| **x4** | `tp` | Thread pointer | — | Points to thread-local storage in multi-threaded programs. |
| **x5 – x7** | `t0 - t2` | Temporaries | Caller | Scratchpad registers for quick math; functions don't need to preserve them. |
| **x8** | `s0 / fp` | Saved register / Frame pointer | Callee | Holds variables that must survive across function calls, or tracks the base of the stack frame. |
| **x9** | `s1` | Saved register | Callee | Holds persistent local variables across function calls. |
| **x10 – x11**| `a0 - a1` | Function arguments / Return values | Caller | Passes the first two inputs into a function and returns the output back. |
| **x12 – x17**| `a2 - a7` | Function arguments | Caller | Passes up to 6 additional arguments into a function. |
| **x18 – x27**| `s2 - s11` | Saved registers | Callee | Additional registers for long-lived variables that must be preserved. |
| **x28 – x31**| `t3 - t6` | Temporaries | Caller | Extra scratchpad registers for intermediate calculations. |

#### Caller-Saved vs. Callee-Saved Registers
* **Caller-Saved (`ra`, `t0–t6`, `a0–a7`):** If Function A calls Function B, Function B is free to overwrite these registers. If Function A still needs the data inside them after Function B finishes, **Function A (the caller)** must save them to the stack before making the call.
* **Callee-Saved (`sp`, `s0–s11`):** If **Function B (the callee)** wants to use any of these registers, it must first save their original values onto the stack and restore them before returning to Function A.

---

### 3. Memory Allocation & Little-Endian Format
In RISC-V, memory is **byte-addressable**, meaning every individual byte (8 bits) has its own unique address.
* **Doubleword Size:** In RV64, a **doubleword** is **64 bits = 8 bytes**. For example, an array `M` of 3 doublewords takes up 3 × 8 = 24 bytes of memory.
* **Little-Endian Ordering:** RISC-V stores multi-byte data in **Little-Endian** format. This means the **Least Significant Byte (LSB)** is stored at the lowest memory address, and the **Most Significant Byte (MSB)** is stored at the highest memory address.

#### Visual Walkthrough (From Lecture Example):
Suppose memory addresses `16` through `23` hold an 8-byte doubleword:
* **Address `23` (Highest Address / MSB):** `01110010`
* **Address `22`:** `00011111`
* **Address `21`:** `01001001`
* **Address `20`:** `01001100`
* **Address `19`:** `01011000`
* **Address `18`:** `10001100`
* **Address `17`:** `10111101`
* **Address `16` (Lowest Address / LSB):** `10111111`

When loaded into 64-bit register **`x8`**, the bytes are assembled from MSB (left) to LSB (right) as:
`(01110010 00011111 01001001 01001100 01011000 10001100 10111101 10111111)_bin`

---

### 4. Step-by-Step Instruction & Encoding Breakdown
Every RISC-V instruction is **32 bits wide** (bits `31` down to `0`). Depending on what the instruction does, those 32 bits are split into different fields:

#### Step 1: Load Doubleword — `ld x8, 16(x23)` *(I-Type Format)*
* **What it does:** Takes the base address stored in `x23` (e.g., `0`), adds the offset `16` to get memory address `16`, reads 8 bytes (64 bits), and loads them into destination register `x8`.
* **32-Bit Instruction Layout:**
  ```text

  | immediate [31:20] | rs1 [19:15] | funct3 [14:12] | rd [11:7] | opcode [6:0] |
  ```
  * `immediate` (`12 bits`): The byte offset (`16`).
  * `rs1` (`5 bits`): Base address register (`x23`).
  * `funct3` (`3 bits`): Identifies the data width (`011` for doubleword).
  * `rd` (`5 bits`): Destination register (`x8`).
  * `opcode` (`7 bits`): Identifies this as a Load instruction.

#### Step 2: Add — `add x8, x24, x8` *(R-Type Format)*
* **What it does:** Adds the 64-bit value in `x24` (`rs1`) to the 64-bit value we just loaded into `x8` (`rs2`), and writes the sum into `x8` (`rd`).
* **32-Bit Instruction Layout:**
  ```text

  | funct7 [31:25] | rs2 [24:20] | rs1 [19:15] | funct3 [14:12] | rd [11:7] | opcode [6:0] |
  ```
  * `funct7` + `funct3`: Together specify the exact ALU operation (`ADD`).
  * `rs2` (`5 bits`): Second source operand (`x8`).
  * `rs1` (`5 bits`): First source operand (`x24`).
  * `rd` (`5 bits`): Destination register (`x8`).
  * `opcode` (`7 bits`): Identifies this as a register-to-register ALU instruction.

#### Step 3: Store Doubleword — `sd x8, 8(x23)` *(S-Type Format)*
* **What it does:** Takes the updated 64-bit result from data register `x8` (`rs2`) and writes it back into memory starting at address `x23 + 8` (`0 + 8 = 8`).
* **32-Bit Instruction Layout:**
  ```text

  | imm[11:5] [31:25] | rs2 [24:20] | rs1 [19:15] | funct3 [14:12] | imm[4:0] [11:7] | opcode [6:0] |
  ```
  * Notice that `rs1` (`x23`) and `rs2` (`x8`) stay in the exact same bit positions (`19:15` and `24:20`) as in R-type instructions to keep hardware decoding simple! Because there is no destination register `rd`, the 12-bit offset (`8`) is split into two pieces: `imm[11:5]` and `imm[4:0]`.

---

### 5. Basic Verification Flow (Running a C Program on a RISC-V CPU)
Before building our own RISC-V core from scratch, we verify how a complete C program runs on an existing **Verilog RISC-V CPU model**:

```text
[ C Program (e.g., Sum 1 to N) ]
               │
               ▼  (RISC-V GCC Compiler + Objcopy)
[ Hex Format Machine Code File ]
               │
               ▼  (Loaded into Instruction/Data Memory)
[ RISC-V CPU Written in Verilog ] ──► Executes instructions ──► [ Display: "Sum of numbers from 1 to n is __" ]
```

1. **Software Compilation:** A C program (calculating the sum of numbers from `1` to `n`) is compiled using the RISC-V GCC toolchain into machine instructions, then converted into a **`.hex` file**.
2. **Memory Initialization:** During Verilog simulation, the testbench loads this `.hex` file directly into the simulated **Memory** block.
3. **Hardware Execution & Output:** The **RISC-V CPU (written in Verilog)** fetches instructions one by one from memory, executes the loops and additions using the ABI register conventions, and prints the final result on the simulator console.









<img width="1076" height="607" alt="image" src="https://github.com/user-attachments/assets/a2c73b00-e8f9-4392-ae16-0e44b818b1b9" />


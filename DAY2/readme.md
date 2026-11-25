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




# LAB WORK

### Part 1: Calling an Assembly Function from C (`1to9_custom.c` & `load.S`)

In this lab, we write a C program that delegates the sum calculation (from `1` to `9`) to an external RISC-V assembly function (`load`). This demonstrates how C and Assembly communicate directly through **ABI registers**.

#### 1. C Source Code (`1to9_custom.c`)
The `main()` function declares `load` as an `extern` function and passes two arguments: `0x0` (initial sum) and `count + 1` (`10`). According to RISC-V ABI rules:
* First argument (`0x0`) is placed in register **`a0`**.
* Second argument (`10`) is placed in register **`a1`**.
* The return value from `load()` is expected back in register **`a0`**.


#### 2. Assembly Function (`load.S`)
The assembly function receives the inputs in `a0` and `a1`, runs a loop to add numbers from `0` up to `9`, and places the final sum (`45`) into **`a0`** before returning (`ret`) to `main`.

<img width="1076" height="607" alt="image" src="https://github.com/user-attachments/assets/a2c73b00-e8f9-4392-ae16-0e44b818b1b9" />
<img width="1600" height="900" alt="image" src="https://github.com/user-attachments/assets/3e4a3bb0-f31f-4c43-a5e2-2618c2440845" />
<img width="1600" height="900" alt="image" src="https://github.com/user-attachments/assets/981aee9e-af6a-4936-99fe-bed3b51ff6dc" />


#### 3. Compilation, Simulation & Disassembly Commands
Run the following commands to compile both files together, simulate them on the Spike RISC-V simulator, and inspect the generated assembly instructions:

```bash
# Compile C and Assembly files together using the RISC-V GCC toolchain
riscv64-unknown-elf-gcc -Ofast -mabi=lp64d -march=rv64imafdc -o 1to9_custom.o 1to9_custom.c load.S

# Run the executable on the Spike simulator via Proxy Kernel (pk)
spike pk 1to9_custom.o

# View the disassembled object file to verify ABI register assignments in main
riscv64-unknown-elf-objdump -d 1to9_custom.o | less
```

**Expected Output:**
```text
bbl loader
Sum of number from 1 to 9 is 45
```

#### 4. Key Observations from `objdump` Disassembly (`<main>`)
Looking at the `<main>` disassembly confirms how the compiler follows the ABI rules:
* `li a1, 10` (`45a9`): Loads the second argument (`count + 1 = 10`) into **`a1`**.
* `li a0, 0` (`4501`): Loads the first argument (`0x0`) into **`a0`**.
* `sd ra, 8(sp)` (`e406`): Saves the return address (`ra`) onto the stack before jumping.
* `jal 1017c <load>`: Jumps to the `load` assembly function.
* `mv a2, a0` (`862a`): Moves the returned sum (`45`) from **`a0`** into `a2` so `printf` can display it.

---

#### 3. Execution Commands & Hardware Simulation Output

To navigate into the lab directory, give execution permissions to the script, and run the Verilog hardware simulation, use the following commands:

```bash
# Navigate to the lab directory
cd ~/riscv_workshop_collaterals/labs

# Grant execution permission to the shell script
chmod 777 rv32im.sh

# Run the compilation and Verilog simulation script
./rv32im.sh
```

<img width="1076" height="607" alt="image" src="https://github.com/user-attachments/assets/248f72e7-9046-4b2a-b906-fe6d90d6c0af" />
<img width="1600" height="900" alt="image" src="https://github.com/user-attachments/assets/d7b0ab0f-bceb-4ff7-8869-28f30b978e6e" />

<img width="1600" height="900" alt="image" src="https://github.com/user-attachments/assets/a5a4c70e-3a34-488b-a24b-e8c8d62ae0b0" />


##### Command Breakdown:
* **`cd ~/riscv_workshop_collaterals/labs`**: Moves into the working directory containing the C source files (`1to9_custom.c`), assembly code (`load.S`, `start.S`), Verilog CPU (`picorv32.v`), and testbench (`testbench.v`).
* **`chmod 777 rv32im.sh`**: Grants read, write, and execute permissions to `rv32im.sh` so the terminal can run it as a program.
* **`./rv32im.sh`**: Executes the entire pipeline—compiling the C/Assembly code for `RV32IM`, generating `firmware32.hex`, compiling the Verilog design with `iverilog`, and simulating it with `vvp`.

##### Simulation Output:
```text
Sum of number from 1 to 2 is 3
TRAP
testbench.v:111: \$finish called at 162950000 (1ps)
```

* **`Sum of number from 1 to 2 is 3`**: Printed directly by the simulated **PicoRV32** hardware core executing the compiled C program from memory (where `count` in the local lab file was set to `2`).
* **`TRAP`**: Signals that the processor finished executing `main()` and reached the end-of-program halt/trap condition defined in `start.S`.
* **`$finish called at 162950000 (1ps)`**: Indicates that the Verilog testbench (`testbench.v` at line `111`) cleanly terminated the simulation once the `TRAP` signal was detected.

---

#### 4. Inspecting the Generated Memory File (`firmware.hex`)

You can inspect the generated machine-code hex file using `vim firmware.hex` or any text editor:

```bash
vim firmware.hex
```

##### What is inside `firmware.hex`?
* **`@00000000` (Boot / Startup Section):** The `@` symbol specifies the starting memory address in hexadecimal. Address `0x00000000` contains the machine code from `start.S` (initializing registers `x1` through `x15` to zero with `addi` instructions like `93 00 00 00`, `93 01 00 00`, etc.) before jumping to `main`.
* **`@00010000` (Main Firmware Section):** Starting at memory address `0x00010000`, this block stores the compiled 32-bit instructions for `main()`, `load()`, and `printf()` in **Little-Endian byte order** (least significant byte first), which the Verilog testbench loads into memory to drive the CPU.









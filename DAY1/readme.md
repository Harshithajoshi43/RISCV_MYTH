# Day 01: Getting Started with RISC-V and the GNU Toolchain

> Compile a simple C program for a RISC-V processor, look at the assembly the compiler produces, and run it on a software model of a RISC-V CPU (the Spike simulator).

---

## Table of Contents

1. [What You Will Learn](#1-what-you-will-learn)
2. [The Big Picture: From C Code to Silicon](#2-the-big-picture-from-c-code-to-silicon)
3. [Key Concepts](#3-key-concepts)
4. [Tools Used](#4-tools-used)
5. [Lab: Sum of the First N Integers](#5-lab-sum-of-the-first-n-integers)
6. [Understanding the Output](#6-understanding-the-output)
7. [Troubleshooting](#7-troubleshooting)
8. [Summary](#8-summary)

---

## 1. What You Will Learn

By the end of this lab you will be able to:

- Explain how a C program turns into instructions a processor can run.
- Describe what an ISA is and why RISC-V is a useful one to study.
- Cross-compile a C program for RISC-V with different optimisation levels.
- Read the generated assembly using `objdump`.
- Run the program on Spike and inspect CPU registers while it executes.

---

## 2. The Big Picture: From C Code to Silicon

### 2.1 Three layers of a computer system

| Layer | What lives here | Example |
|-------|-----------------|---------|
| Application software | Programs you write | C, C++, Java programs |
| System software | Tools that prepare and manage programs | OS, compiler, assembler |
| Hardware | The chip that actually runs instructions | A RISC-V processor |

### 2.2 How your code reaches the hardware

```
 C program  ──►  Compiler  ──►  Assembly  ──►  Assembler  ──►  Machine code (0s and 1s)
                                                                        │
                                                                        ▼
                                                      CPU: fetch → decode → execute
```

1. You write a program in C.
2. The **compiler** translates it into assembly instructions.
3. The **assembler** converts each assembly instruction into binary machine code.
4. The **processor** fetches an instruction, decodes it, and executes it, over and over.

### 2.3 Where the ISA fits in

The **Instruction Set Architecture (ISA)** is the contract between software and hardware. It spells out:

- which instructions exist,
- which registers the CPU has,
- which data types it understands,
- what each operation does.

Compilers target the ISA, and chip designers build hardware that obeys the ISA. As long as both sides respect it, software runs on any compatible chip. **RISC-V** is an open ISA, meaning anyone can use it without licence fees.

### 2.4 How an ISA becomes a physical chip (VLSI flow)

```
Specification ► RTL design (Verilog/SystemVerilog) ► Synthesis (gate-level netlist)
              ► Physical design (floorplan, placement, clock tree, routing) ► Final layout
```

| Stage | What happens |
|-------|--------------|
| Specification | Decide what the processor must do |
| RTL design | Describe the processor in a hardware language |
| Synthesis | Turn RTL into a netlist of logic gates |
| Physical design | Floorplanning, placement, clock tree synthesis, routing |
| Layout | The final geometry sent for fabrication |

This lab stays at the software and ISA level. You will not touch the hardware flow here, but it helps to know where this fits.

---

## 3. Key Concepts

These topics are introduced on Day 01 and revisited in later labs.

### 3.1 64-bit integer representation

| Type | Range |
|------|-------|
| Unsigned 64-bit | `0` to `18,446,744,073,709,551,615` (2⁶⁴ − 1) |
| Signed 64-bit (two's complement) | `-9,223,372,036,854,775,808` to `9,223,372,036,854,775,807` (−2⁶³ to 2⁶³ − 1) |

In signed (two's complement) form, the most significant bit acts as the sign bit: `0` means positive, `1` means negative.

### 3.2 Application Binary Interface (ABI)

The ABI is a set of rules for how compiled code behaves, for example which registers carry function arguments and return values. In RISC-V the 32 integer registers have friendly names:

| Register | ABI name | Typical role |
|----------|----------|--------------|
| x0 | `zero` | Always holds 0 |
| x1 | `ra` | Return address |
| x2 | `sp` | Stack pointer |
| x3 | `gp` | Global pointer |
| x4 | `tp` | Thread pointer |
| x5 – x7, x28 – x31 | `t0`–`t6` | Temporaries |
| x8 | `s0` / `fp` | Saved register / frame pointer |
| x9, x18 – x27 | `s1`, `s2`–`s11` | Saved registers |
| x10 – x11 | `a0`, `a1` | Function arguments and return values |
| x12 – x17 | `a2`–`a7` | More function arguments |

The `-mabi=lp64` flag used below means: `long` and pointers are 64 bits wide, `int` is 32 bits, and no hardware floating-point registers are used for passing arguments.

### 3.3 Memory and the stack pointer

- Programs get a region of memory for the **stack**, which holds local variables, saved registers and return addresses.
- The register `sp` always points to the top of the stack.
- On RISC-V the stack **grows downward**: making room means *subtracting* from `sp`, and releasing space means *adding* to it.

### 3.4 Pseudo-instructions

Pseudo-instructions are friendly shortcuts that the assembler expands into real base instructions.

| You write | Assembler turns it into | Meaning |
|-----------|-------------------------|---------|
| `li a0, 5` | `addi a0, zero, 5` | Load an immediate value |
| `mv a1, a0` | `addi a1, a0, 0` | Copy a register |
| `nop` | `addi zero, zero, 0` | Do nothing |
| `ret` | `jalr zero, ra, 0` | Return from function |
| `j label` | `jal zero, label` | Unconditional jump |
| `beqz a0, label` | `beq a0, zero, label` | Branch if equal to zero |

### 3.5 The RISC-V instruction set modules

RISC-V is modular. You pick a base set and add extensions as needed.

| Name | Full name | What it adds |
|------|-----------|--------------|
| **RV64I** | Base Integer | Add, subtract, logic, shifts, loads, stores, branches, jumps |
| **RV64M** | Multiply / Divide | `mul`, `div`, `rem` and related instructions |
| **RV64F** | Single-precision float | 32-bit floating point (`fadd.s`, `flw`, `fsw`, ...) |
| **RV64D** | Double-precision float | 64-bit floating point (`fadd.d`, ...) |

In the lab below we use only **RV64I** (`-march=rv64i`), the smallest 64-bit base set.

---

## 4. Tools Used

| Tool | What it does |
|------|--------------|
| `gcc` | Regular compiler for your own computer (x86/ARM) |
| `riscv64-unknown-elf-gcc` | Cross-compiler: runs on your PC, produces RISC-V programs |
| `riscv64-unknown-elf-objdump` | Disassembler: shows the assembly inside a compiled file |
| `spike` | Software simulator of a RISC-V processor |
| `pk` (Proxy Kernel) | Tiny runtime layer that lets a program use things like `printf` on Spike |

> **Why cross-compile?** Your laptop does not run RISC-V natively, so you compile on your machine but *target* a different CPU. A native program cannot run on RISC-V, and a RISC-V program cannot run directly on your laptop. That is why we need Spike.

---

## 5. Lab: Sum of the First N Integers

**Goal:** add the numbers 5 + 4 + 3 + 2 + 1 using a loop, and see how the same C code behaves natively and on RISC-V.

### Step 1: Write the C program

Create a file named `sum1ton.c` with any editor (`nano`, `vim`, `gedit`, `leafpad`, VS Code, ...):

```bash
nano sum1ton.c
```

Paste this code:

```c
#include <stdio.h>

int main()
{
    int sum = 0;
    int n = 5;

    while (n)
    {
        sum += n;
        n--;
    }

    printf("Sum of integers,%d \n", sum);

    return 0;
}
```

**How it works:** `sum` starts at 0. Each loop pass adds the current `n` to `sum`, then lowers `n` by one. When `n` reaches 0 the loop condition becomes false and the loop stops.

| Pass | n before | sum after |
|------|----------|-----------|
| 1 | 5 | 5 |
| 2 | 4 | 9 |
| 3 | 3 | 12 |
| 4 | 2 | 14 |
| 5 | 1 | 15 |

### Step 2: Compile and run natively (sanity check)

First confirm the program is correct using the normal compiler on your own machine:

```bash
gcc sum1ton.c -o a.out
./a.out
```

Expected output:

```
Sum of integers,15
```

### Step 3: Cross-compile for RISC-V

Compile with optimisation level **-O1**:

```bash
riscv64-unknown-elf-gcc -O1 -mabi=lp64 -march=rv64i -o sum1ton9.elf sum1ton.c
```

Compile again with **-Ofast**:

```bash
riscv64-unknown-elf-gcc -Ofast -mabi=lp64 -march=rv64i -o sum1ton9.elf sum1ton.c
```

**What each flag means**

| Flag | Meaning |
|------|---------|
| `-O1` | Basic optimisations: smaller and faster code without long compile times |
| `-Ofast` | Aggressive optimisation for maximum speed (can bend some strict standards rules) |
| `-march=rv64i` | Target the 64-bit base integer ISA only |
| `-mabi=lp64` | Use the 64-bit integer ABI (long and pointers are 64-bit) |
| `-o <name>` | Name of the output file |

You now have two `.elf` executables built from the *same* C code but optimised differently.

### Step 4: Look at the assembly

Disassemble the `-O1` build:

```bash
riscv64-unknown-elf-objdump -d sum1ton.elf | less
```

Disassemble the `-Ofast` build:

```bash
riscv64-unknown-elf-objdump -d sum1tonfast.elf | less
```

Tips for `less`: press `/main` then Enter to search for `main`, press `n` for the next match, and press `q` to quit.

Each line of output shows three things: the **memory address**, the **machine code in hex**, and the **assembly instruction**.

To save the output in a file instead:

```bash
riscv64-unknown-elf-objdump -d sum1ton9.elf > sum1ton.txt
riscv64-unknown-elf-objdump -d sum1ton9fast.elf > sum1ton.txt
```

To compare how many instructions `main` takes in each build, open both files and look at the `<main>` section. The `-Ofast` version is often shorter or tighter, because the compiler can simplify the loop.

### Step 5: Run on the Spike simulator

Run the program normally:

```bash
spike pk sum1ton.elf
```

Expected output:

```
Sum of integers,15
```

Run the `-Ofast` build too and confirm that the result is the same:

```bash
spike pk sum1ton.o
```

> **Note:** Use the `.elf` file name that you created in Step 3. Spike takes the program as an argument after `pk`.

### Step 6: Debug mode and register inspection

Start Spike in interactive debug mode:

```bash
spike -d pk sum1ton.o
```

You will get a `:` prompt. Type the following commands one at a time:

```
until pc 0 100b0
```

```
reg 0 a0
```

| Command | What it does |
|---------|--------------|
| `until pc 0 100b0` | Run core 0 until its program counter (`pc`) reaches address `0x100b0` |
| `reg 0 a0` | Print the current value of register `a0` on core 0 |

Other useful debug commands:

| Command | What it does |
|---------|--------------|
| (just press Enter) | Execute one instruction |
| `reg 0` | Show all registers of core 0 |
| `pc 0` | Show the current program counter |
| `mem 0 <address>` | Read memory at an address |
| `q` | Quit Spike |

> **Important:** The address `100b0` came from the author's own disassembly. Your addresses may differ. Open your `objdump` output, pick an address inside `main` (for example just after the loop ends), and use that address in the `until` command.

---

## 6. Understanding the Output

### Native and RISC-V results match

| Run | Output |
|-----|--------|
| Native `gcc` on your PC | `Sum of integers,15` |
| RISC-V `-O1` on Spike | `Sum of integers,15` |
| RISC-V `-Ofast` on Spike | `Sum of integers,15` |

The answer is identical everywhere. That is the whole point of an ISA and a toolchain: the *same* C source gives the *same* result, whichever processor it targets.

### What an inspected register shows

When you pause just after the loop and run `reg 0 a0`, you are looking at the value held in `a0` at that moment. Since `a0` is also the register used for arguments and return values, checking it around the `printf` call helps you follow how the sum travels through the program.

### Why -O1 and -Ofast look different

The compiler may keep variables in registers instead of memory, remove unnecessary instructions, or restructure the loop. The behaviour stays the same while the instruction sequence changes.

---

---

## 7. Summary

- The ISA is the bridge between software and hardware, and RISC-V is an open, modular ISA.
- The GNU RISC-V toolchain lets you compile C for RISC-V from an ordinary PC.
- `objdump` reveals the real instructions behind your C code.
- Spike runs and debugs RISC-V programs without any physical hardware.
- Different optimisation levels change the instructions, not the result.

### Quick command cheat sheet

```bash
# 1. Native build and run
gcc sum1ton.c -o a.out && ./a.out

# 2. RISC-V builds
riscv64-unknown-elf-gcc -O1    -mabi=lp64 -march=rv64i -o sum1ton.o   sum1ton.c
riscv64-unknown-elf-gcc -Ofast -mabi=lp64 -march=rv64i -o sum1tonOfast.o sum1ton.c

# 3. Disassemble
riscv64-unknown-elf-objdump -d sum1ton.o | less

# 4. Run on Spike
spike pk sum1ton.o

# 5. Debug on Spike
spike -d pk sum1ton.o
```

---

**Next:** Day 02, where we go deeper into RISC-V instruction formats and the generated assembly.

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/f82b06d2-c9fc-43c7-a135-b1fd0bb0a20c" />


<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/22418557-993f-47b1-abd1-f432e723d626" />
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/9f6666b5-4055-447d-90c0-70248b4ea247" />

<img width="1920" height="459" alt="image" src="https://github.com/user-attachments/assets/c2f92002-0159-4f0f-82ef-9698a7035264" />
<img width="1920" height="459" alt="image" src="https://github.com/user-attachments/assets/f64ef339-d8d7-4443-bef4-a989c3b8f980" />

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

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/f82b06d2-c9fc-43c7-a135-b1fd0bb0a20c" />


<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/22418557-993f-47b1-abd1-f432e723d626" />
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/9f6666b5-4055-447d-90c0-70248b4ea247" />




## Lab 02: How 64-bit numbers are stored

RV64 means registers are **64 bits** wide, so each register holds one 64-bit integer.

### Data units

| Unit | Size | Bytes |
|------|------|-------|
| Bit | 1 bit | n/a |
| Byte | 8 bits | 1 |
| Halfword | 16 bits | 2 |
| Word | 32 bits | 4 |
| Doubleword | 64 bits | 8 |

A 64-bit **doubleword** is 8 bytes.

- **LSB (Least Significant Bit):** the rightmost bit, worth the least.
- **MSB (Most Significant Bit):** the leftmost bit (bit 63), worth the most.

```
bit 63                                        bit 0
  │                                             │
  ▼                                             ▼
 [ MSB ][ . . . . . . . . . . . . . . . . . ][ LSB ]
```

### Unsigned integers (only zero and positive)

For **n bits**:

```
range = 0 to 2^n − 1
```

**Example with 3 bits:** 2³ = 8 combinations, so the values are 0 to 7.

| Binary | Value |
|--------|-------|
| 000 | 0 |
| 001 | 1 |
| 010 | 2 |
| 011 | 3 |
| 100 | 4 |
| 101 | 5 |
| 110 | 6 |
| 111 | 7 |

**For 64 bits:**

```
0 to 2^64 − 1  =  0 to 18,446,744,073,709,551,615
```

**Overflow (wrap-around):** if a result is too big, it wraps around, like a car odometer rolling over. In 64-bit unsigned maths, the result is taken **modulo 2⁶⁴**:

```
18,446,744,073,709,551,615 + 1  →  0
```

(With 3 bits: `7 + 1 = 0`.)

### Signed integers (two's complement)

Signed numbers can be negative. RISC-V stores them in **two's complement**.

- **MSB = 0** → non-negative
- **MSB = 1** → negative

For **n bits**:

```
range = −2^(n−1) to 2^(n−1) − 1
```

**For 64 bits:**

```
−2^63 to 2^63 − 1  =  −9,223,372,036,854,775,808 to 9,223,372,036,854,775,807
```

**3-bit example:**

| Binary | Signed value | Unsigned value |
|--------|--------------|----------------|
| 000 | 0 | 0 |
| 001 | 1 | 1 |
| 010 | 2 | 2 |
| 011 | 3 | 3 |
| 100 | −4 | 4 |
| 101 | −3 | 5 |
| 110 | −2 | 6 |
| 111 | −1 | 7 |

###  How to write a negative number

1. Write the positive number in binary.
2. Flip every bit (1's complement).
3. Add 1 (2's complement).

**Example: −5 in 8 bits**

```
+5            = 0000 0101
flip all bits = 1111 1010
add 1         = 1111 1011   ← this is −5 (hex 0xFB)
```

In 64 bits, −5 is `0xFFFFFFFFFFFFFFFB`.

### Same bits, different meaning

The hardware only stores bits. Whether they mean a signed or unsigned number depends on how the program uses them.

| 64-bit pattern | As unsigned | As signed |
|----------------|-------------|-----------|
| `0x0000000000000005` | 5 | 5 |
| `0x7FFFFFFFFFFFFFFF` | 9,223,372,036,854,775,807 | 9,223,372,036,854,775,807 |
| `0x8000000000000000` | 9,223,372,036,854,775,808 | −9,223,372,036,854,775,808 |
| `0xFFFFFFFFFFFFFFFB` | 18,446,744,073,709,551,611 | −5 |
| `0xFFFFFFFFFFFFFFFF` | 18,446,744,073,709,551,615 | −1 |

---

## Lab 02 hands-on: largest and smallest 64-bit values

Here we use `pow()` from `<math.h>` to calculate the limits. `pow()` works with `double`, so we compile with floating-point support:

- `-march=rv64imafdc` turns on the integer, multiply/divide, atomic, float, double and compressed instructions.
- `-mabi=lp64d` passes doubles in floating-point registers.
- `-lm` links the math library.

### Signed: `signedHighest.c`

```c
#include <stdio.h>
#include <math.h>

int main()
{
    long long int max = (long long int)(pow(2, 63) - 1);
    long long int min = (long long int)(pow(2, 63) * -1);

    printf("Highest number represented by long long int is %lld\n", max);
    printf("Lowest number represented by long long int is %lld\n", min);
    return 0;
}
```

Compile and run:

```bash
riscv64-unknown-elf-gcc -Ofast -mabi=lp64d -march=rv64imafdc -o signedHighest.o signedHighest.c -lm
spike pk signedHighest.0
```

Output:

```
bbl loader
Highest number represented by long long int is 9223372036854775807
Lowest number represented by long long int is -9223372036854775808
```

### Unsigned: `unsignedHighest.c`

```c
#include <stdio.h>
#include <math.h>

int main()
{
    unsigned long long int max = (unsigned long long int)(pow(2, 64) - 1);

    printf("Highest number represented by unsigned long long int is %llu\n", max);
    return 0;
}
```

Compile and run:

```bash
riscv64-unknown-elf-gcc -Ofast -mabi=lp64d -march=rv64imafdc -o unsignedHighest.o unsignedHighest.c -lm
spike pk unsignedHighest.o
```

Output:

```
bbl loader
Highest number represented by unsigned long long int is 18446744073709551615
```

> `bbl loader` is just a start-up banner printed by the proxy kernel. It is normal.

### 5.3 These match the theory

| Type | Theory | Program output |
|------|--------|----------------|
| Signed max | 2⁶³ − 1 | 9223372036854775807 |
| Signed min | −2⁶³ | -9223372036854775808 |
| Unsigned max | 2⁶⁴ − 1 | 18446744073709551615 |

### A note on accuracy

A `double` only has 53 bits of precision, so `pow(2,63) - 1` can't be stored exactly. It rounds up to 2⁶³, which is one larger than the biggest `long long`. In C, converting an out-of-range `double` to an integer is *undefined behaviour*. RISC-V happens to clamp the value to the maximum, so the output looks correct, but other processors may give a different result.

A safe way to get the same numbers uses `<limits.h>`:

```c
#include <stdio.h>
#include <limits.h>

int main()
{
    printf("Max long long          : %lld\n", LLONG_MAX);
    printf("Min long long          : %lld\n", LLONG_MIN);
    printf("Max unsigned long long : %llu\n", ULLONG_MAX);
    return 0;
}
```

```bash
riscv64-unknown-elf-gcc -O1 -mabi=lp64 -march=rv64i -o limits.elf limits.c
spike pk limits.elf
```

### Seeing wrap-around and −5 in action

```c
#include <stdio.h>
#include <stdint.h>
#include <inttypes.h>

int main()
{
    int64_t  neg  = -5;
    uint64_t wrap = UINT64_MAX;

    wrap = wrap + 1;   /* unsigned overflow wraps to 0 */

    printf("-5 as signed   : %" PRId64 "\n", neg);
    printf("-5 as unsigned : %" PRIu64 "\n", (uint64_t)neg);
    printf("-5 in hex      : 0x%016" PRIx64 "\n", (uint64_t)neg);
    printf("UINT64_MAX + 1 : %" PRIu64 "\n", wrap);
    return 0;
}
```

```bash
riscv64-unknown-elf-gcc -O1 -mabi=lp64 -march=rv64i -o datatypes.elf datatypes.c
spike pk datatypes.elf
```

Expected output:

```
-5 as signed   : -5
-5 as unsigned : 18446744073709551611
-5 in hex      : 0xfffffffffffffffb
UINT64_MAX + 1 : 0
```

This shows that the **same bits** (`0xFFFFFFFFFFFFFFFB`) are −5 when read as signed and 18446744073709551611 when read as unsigned.

---

## RV64 extensions

RISC-V is modular: a small base plus optional add-ons.

| Extension | What it adds |
|-----------|-------------|
| **RV64I** | Base 64-bit integer instructions: add, subtract, logic, shifts, loads, stores, branches, jumps |
| **RV64M** | Integer multiply and divide |
| **RV64F** | Single-precision (32-bit) floating point |
| **RV64D** | Double-precision (64-bit) floating point |

`-march=rv64i` uses only the base. `-march=rv64imafdc` adds M, A (atomics), F, D and C (compressed 16-bit instructions).

---

## Common instructions explained

These appear again and again in `objdump` output.

| Instruction | Name | What it does | Example |
|-------------|------|-------------|---------|
| `addi` | Add immediate | Adds a small constant to a register | `addi sp,sp,-16` |
| `lui` | Load upper immediate | Loads a 20-bit value into the upper bits of a register | `lui a5,0x12` |
| `sd` | Store doubleword | Writes 64 bits from a register to memory | `sd ra,8(sp)` |
| `ld` | Load doubleword | Reads 64 bits from memory into a register | `ld ra,8(sp)` |
| `jal` | Jump and link | Jumps to a function and saves the return address | `jal ra,<printf>` |
| `ret` | Return | Jumps back to the caller (pseudo-instruction) | `ret` |
| `li` | Load immediate | Puts a constant in a register (pseudo-instruction) | `li a0,5` |

**In plain words:**

- `addi sp,sp,-16` subtracts 16 from the stack pointer, which reserves 16 bytes of stack space (the stack grows downward).
- `sd ra,8(sp)` saves `ra` at the memory address `sp + 8`. The format is `offset(base register)`.
- `ld ra,8(sp)` loads it back.
- `li a0,5` is shorthand for `addi a0,zero,5`.
- `ret` is shorthand for `jalr zero,0(ra)`.

**Pseudo-instructions** are friendly shortcuts that the assembler expands into real instructions.

**Loading a big constant:** one `addi` can only handle −2048 to 2047, so larger values use two instructions. For example, `li a5,0x12345` becomes:

```asm
lui  a5,0x12        # a5 = 0x12000  (0x12 shifted left by 12 bits)
addi a5,a5,0x345    # a5 = 0x12345
```

**Typical function start and end:**

```asm
addi sp,sp,-16      # make room on the stack
sd   ra,8(sp)       # save the return address

# ... function body, may call other functions with jal ...

ld   ra,8(sp)       # restore the return address
addi sp,sp,16       # free the stack space
ret                 # go back to the caller
```

### Important registers

| Register | ABI name | Role |
|----------|----------|------|
| x0 | `zero` | Always 0 |
| x1 | `ra` | Return address |
| x2 | `sp` | Stack pointer |
| x10 – x11 | `a0`, `a1` | Arguments and return values |
| x12 – x17 | `a2`–`a7` | More arguments |
| x5 – x7, x28 – x31 | `t0`–`t6` | Temporaries |
| x8, x9, x18 – x27 | `s0`–`s11` | Saved registers |

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `riscv64-unknown-elf-gcc: command not found` | Add the toolchain's `bin` folder to your `PATH` |
| `spike: command not found` | Install Spike or add it to your `PATH` |
| Spike can't find `pk` | Give the full path, e.g. `spike /path/to/riscv64-unknown-elf/bin/pk file.elf` |
| `undefined reference to pow` | Add `-lm` at the end of the compile command |
| Float-ABI mismatch error | Use `-march=rv64imafdc` with `-mabi=lp64d`, or `-march=rv64i` with `-mabi=lp64` |
| `until pc 0 <addr>` never stops | The address isn't in your code. Check your own `objdump` output |

---

## Summary

- The **ISA** is the contract between software and hardware, and RISC-V is open and modular.
- A **cross-compiler** builds RISC-V programs on your PC, and **Spike** runs them.
- A 64-bit **doubleword** is 8 bytes.
- **Unsigned** 64-bit range is `0` to `2⁶⁴ − 1`, and overflow wraps around.
- **Signed** 64-bit range is `−2⁶³` to `2⁶³ − 1`, stored as two's complement (flip bits, add 1).
- The **same bits** can mean different numbers depending on signed or unsigned use.
- Common instructions: `addi`, `lui`, `ld`, `sd`, `jal`, `ret`, `li`.



#### Lab 02
riscv64-unknown-elf-gcc -Ofast -mabi=lp64d -march=rv64imafdc -o signedHighest.o signedHighest.c -lm
spike pk signedHighest.o
riscv64-unknown-elf-gcc -Ofast -mabi=lp64d -march=rv64imafdc -o unsignedHighest.o unsignedHighest.c -lm
spike pk unsignedHighest.o
```

<img width="1920" height="459" alt="image" src="https://github.com/user-attachments/assets/c2f92002-0159-4f0f-82ef-9698a7035264" />
<img width="1920" height="459" alt="image" src="https://github.com/user-attachments/assets/f64ef339-d8d7-4443-bef4-a989c3b8f980" />

# Task 2: RISC-V Program Execution and Instruction-Level Debugging using SPIKE Simulator


# Introduction

After successfully compiling C programs for the RISC-V architecture, the next step is to validate and analyze the generated executable using a RISC-V Instruction Set Simulator (ISA Simulator). This task focuses on executing a RISC-V binary on the SPIKE simulator and performing instruction-level debugging to understand how the processor executes instructions internally.

The experiment demonstrates the complete flow from C source code to RISC-V machine instructions while examining processor state transitions, register updates, stack operations, and compiler optimizations. Particular attention is given to comparing binaries generated using **-O1** and **-Ofast** optimization levels to evaluate their impact on instruction count and execution efficiency.

---

# Learning Objectives

By completing this task, the following concepts are explored:

* Cross-compilation of C programs for RV64I architecture.
* Execution of RISC-V binaries using SPIKE.
* Inspection of generated assembly code.
* Understanding function prologue and stack allocation.
* Analysis of instruction encoding and register behavior.
* Debugging using SPIKE interactive mode.
* Comparison of compiler optimization techniques.
* Verification of functional equivalence between native and RISC-V execution.

---

# Development Environment

| Tool                 | Purpose                              |
| -------------------- | ------------------------------------ |
| GCC                  | Native compilation and verification  |
| RISC-V GCC Toolchain | Cross-compilation for RV64I          |
| SPIKE                | RISC-V ISA Simulator                 |
| Proxy Kernel (pk)    | Runtime support for system calls     |
| Objdump              | Disassembly and instruction analysis |

---

# Application Under Test

### Source Program: `sum1ton.c`

The program calculates the summation of integers from 1 to 100.

Mathematically:

\sum_{i=1}^{100} i = 5050

Expected Output:

```text
Sum from 1 to 100 is 5050
```

Although the application is simple, it serves as an excellent example for studying compiler behavior and processor execution.

---

# Step 1: Native Program Verification

Before generating the RISC-V executable, the program is compiled and executed on the host machine.

```bash
gcc sum1ton.c
./a.out
```

Output:

```text
5050
```

This output serves as the reference result against which the RISC-V implementation is validated.

### Screenshot

<img src="Screenshot 2026-06-02 144117.png"  width="800"/>

---

# Step 2: RISC-V Cross Compilation

The application is compiled for the RV64I instruction set architecture.

```bash
riscv64-unknown-elf-gcc -Ofast \
-mabi=lp64 \
-march=rv64i \
-o sum1ton.o \
sum1ton.c
```

### Compilation Parameters

| Parameter      | Description                      |
| -------------- | -------------------------------- |
| `-march=rv64i` | Generates RV64I instructions     |
| `-mabi=lp64`   | Uses LP64 ABI                    |
| `-Ofast`       | Enables aggressive optimizations |
| `-o`           | Defines output executable        |

The output is an ELF executable containing RISC-V machine instructions.


# Step 3: Executing the Program on SPIKE

The generated executable is executed using the SPIKE simulator and Proxy Kernel.

```bash
spike pk sum1ton.o
```

Output:

```text
Sum from 1 to 100 is 5050
```

The result matches the native implementation, confirming that the generated RISC-V binary is functionally correct.


# Step 4: Assembly-Level Inspection

To analyze the generated machine code, the executable is disassembled using:

```bash
riscv64-unknown-elf-objdump -d sum1ton.o | less
```

Navigate directly to the `main()` function:

```bash
/main
```

## Observation

The compiler recognizes that the sum of integers from 1 to 100 is a constant value.

Instead of generating instructions for a loop, the compiler evaluates the result during compilation itself and embeds the constant directly into the executable.

This optimization technique is called **Constant Folding**.

### Benefits

* Reduced instruction count
* Faster execution
* Smaller executable size
* Lower runtime overhead

### Screenshot

<img src="Screenshot 2026-06-02 144710.png" alt="Disassembly Screenshot" width="800"/>

---

# Step 5: Launching the SPIKE Debugger

SPIKE provides an interactive debugging mode that allows instruction-by-instruction execution.

```bash
spike -d pk sum1ton.o
```

Useful commands:

| Command             | Description                       |
| ------------------- | --------------------------------- |
| `until pc 0 <addr>` | Continue until PC reaches address |
| `reg 0 <reg>`       | Display register contents         |
| `step`              | Execute next instruction          |
| `mem 0 <addr>`      | Inspect memory                    |
| `q`                 | Exit debugger                     |


# Step 6: Navigating to main()

For the optimized executable:

```bash
until pc 0 100b0
```

This skips startup routines and stops execution at the first instruction of the user program.

### Screenshot

<img src="Images/Screenshot 2026-06-02 144448.png" alt="Main Entry Screenshot" width="800"/>

---

# Step 7: Register Analysis – LUI Instruction

One of the first instructions observed is:

```assembly
lui a2, 0x1
```

## Purpose

LUI (Load Upper Immediate) loads a 20-bit immediate value into the upper portion of a register.

Before execution:

```text
a2 = 0x0000000000000000
```

After execution:

```text
a2 = 0x0000000000001000
```

The immediate value is shifted left by 12 bits before being stored.

### Significance

LUI is frequently used to construct:

* Large constants
* Global addresses
* String literal addresses

Typically paired with:

```assembly
lui
addi
```

to create complete 32-bit or 64-bit values.



# Step 8: Stack Frame Creation

At function entry:

```assembly
addi sp, sp, -16
```

This instruction reserves stack memory for local storage and saved registers.

### Stack Pointer Transition

| State  | Value              |
| ------ | ------------------ |
| Before | 0x000000007f7e9b50 |
| After  | 0x000000007f7e9b40 |

Verification:

```text
0x7f7e9b50 - 0x10 = 0x7f7e9b40
```

This confirms allocation of exactly 16 bytes.


# Optimization Study: O1 vs Ofast

To understand compiler behavior, the same program was compiled with two optimization levels.

| Optimization | Instructions in main() |
| ------------ | ---------------------- |
| O1           | 15                     |
| Ofast        | 12                     |

## O1 Build

Characteristics:

* Conservative optimization
* More explicit instruction sequence
* Easier debugging
* Higher instruction count

## Ofast Build

Characteristics:

* Aggressive optimization
* Constant folding
* Reduced instruction count
* Better performance

### Instruction Reduction

```text
15 Instructions  →  12 Instructions
Reduction        → 20%
```

This demonstrates how compiler optimization can improve both performance and code density.



# Key Findings

* Native GCC and RISC-V binaries produce identical results.
* SPIKE accurately simulates RV64I instruction execution.
* Register values can be traced instruction-by-instruction.
* LUI efficiently constructs larger constants and addresses.
* Function prologue allocates stack space using ADDI.
* Compiler optimization significantly reduces instruction count.
* Ofast generates more compact and efficient machine code.

---

# Conclusion

This task provided a detailed understanding of the software execution flow within the RISC-V ecosystem. Starting from C source code, the program was cross-compiled, executed on SPIKE, disassembled, and debugged at the instruction level.

Through register inspection and stack analysis, the internal behavior of the processor became visible, revealing how individual instructions modify machine state. The comparison between O1 and Ofast further demonstrated the importance of compiler optimizations in reducing instruction count and improving execution efficiency.

The knowledge gained in this exercise forms a critical foundation for subsequent stages of the RISC-V design flow, including RTL simulation, functional verification, physical implementation, and eventual silicon realization.
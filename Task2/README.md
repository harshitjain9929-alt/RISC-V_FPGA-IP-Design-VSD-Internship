# Task 2 & Application: RISC-V Program Execution, Instruction-Level Debugging and Digital Design Concepts using SPIKE Simulator

---

# Introduction

After successfully compiling C programs for the RISC-V architecture, the next step is to validate and analyze the generated executable using a RISC-V Instruction Set Simulator (ISA Simulator). This task focuses on executing a RISC-V binary on the SPIKE simulator and performing instruction-level debugging to understand how the processor executes instructions internally.

The experiment demonstrates the complete flow from C source code to RISC-V machine instructions while examining processor state transitions, register updates, stack operations, and compiler optimizations. Particular attention is given to comparing binaries generated using **-O1** and **-Ofast** optimization levels to evaluate their impact on instruction count and execution efficiency.

---

# Learning Objectives

* Cross-compilation of C programs for RV64I architecture.
* Execution of RISC-V binaries using SPIKE.
* Inspection of generated assembly code.
* Understanding function prologue and stack allocation.
* Analysis of instruction encoding and register behavior.
* Debugging using SPIKE interactive mode.
* Comparison of compiler optimization techniques.
* Verification of functional equivalence between native and RISC-V execution.
* Understand the operation of a 4-bit ripple counter.
* Implement binary counting logic in C.
* Analyze assembly code generated under different optimization levels.

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

# Application 1: Sum of Integers (sum1ton.c)

### Source Program: `sum1ton.c`

The program calculates the summation of integers from 1 to 100.

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

<img src="Images/Screenshot 2026-06-02 144117.png" width="800"/>

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

---

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

---

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

<img src="Images/Screenshot 2026-06-02 144710.png" alt="Disassembly Screenshot" width="800"/>

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

---

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

---

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

---

# Optimization Study: O1 vs Ofast (sum1ton)

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

---

# Key Findings (Application 1)

* Native GCC and RISC-V binaries produce identical results.
* SPIKE accurately simulates RV64I instruction execution.
* Register values can be traced instruction-by-instruction.
* LUI efficiently constructs larger constants and addresses.
* Function prologue allocates stack space using ADDI.
* Compiler optimization significantly reduces instruction count.
* Ofast generates more compact and efficient machine code.

---

---

# Application 2: Digital Design Concepts – 4-Bit Ripple Counter (ripple_counter.c)

## Introduction

Digital counters are among the most fundamental sequential circuits used in digital electronics. A ripple counter, also known as an asynchronous counter, consists of cascaded flip-flops where the output of one flip-flop acts as the clock input of the next stage. Each stage divides the input frequency by two, allowing the counter to generate binary count sequences.

To relate software programming with digital design concepts, a C program was developed that emulates the behavior of a 4-bit ripple counter. The program counts from 0 to 15 and displays both the decimal count and its corresponding binary representation.

---

## Digital Design Background

A 4-bit ripple counter consists of four flip-flops connected in cascade. Each flip-flop toggles when triggered by the previous stage.

The binary sequence generated is:

| Decimal | Binary |
| ------- | ------ |
| 0       | 0000   |
| 1       | 0001   |
| 2       | 0010   |
| 3       | 0011   |
| ...     | ...    |
| 15      | 1111   |

Since four flip-flops are used, the counter can represent:

```text
2⁴ = 16 states
```

ranging from 0000 to 1111.

The C implementation mimics this hardware behavior by incrementing a variable and displaying its binary equivalent through bitwise operations.

---

## Source Code Development

The source file was created using:

```bash
gedit ripple_counter.c
```

### Source Code

```c
#include <stdio.h>

int main()
{
    unsigned int counter = 0;

    while(counter < 16)
    {
        printf("Count = %u\tBinary = ", counter);

        for(int i = 3; i >= 0; i--)
        {
            printf("%d", (counter >> i) & 1);
        }

        printf("\n");
        counter++;
    }

    return 0;
}
```

### Program Explanation

* The variable `counter` acts as the counter register.
* The `while` loop generates counts from 0 to 15.
* Right-shift operations extract individual bits.
* Bitwise AND (`& 1`) isolates the current bit.
* The binary output emulates the state transitions of a hardware ripple counter.

<img src="Images/Screenshot 2026-06-08 162821.png" alt="Native GCC Execution" width="800"/>

---

## Step 1: Native GCC Compilation and Execution

The program was first compiled and executed on the host machine.

### Commands

```bash
gcc ripple_counter.c
./a.out
```

### Output Observation

The program successfully generated all sixteen states of a 4-bit binary counter, beginning from 0000 and ending at 1111.

Each increment in the decimal count produced the corresponding binary transition exactly as expected from a hardware ripple counter.

### Screenshot

<img src="Images/Screenshot 2026-06-08 162929.png" alt="Native GCC Execution" width="800"/>

---

## Step 2: RISC-V Compilation – O1

The source code was compiled for the RV64I architecture using the RISC-V GCC toolchain with O1 optimization.

```bash
riscv64-unknown-elf-gcc -O1 -mabi=lp64 -march=rv64i -o ripple_counter.o ripple_counter.c
```
<img src="Images/Screenshot 2026-06-08 171534.png" alt="Native GCC Execution" width="800"/>

### O1 Assembly Analysis

The executable was disassembled using:

```bash
riscv64-unknown-elf-objdump -d ripple_counter.o | less
```

### Instruction Count Calculation (O1)

The `main()` function spans from address `0x10184` to `0x10228`.

$$\text{Instructions} = \frac{0x10228 - 0x10184}{4} + 1 = \frac{0xA4}{4} + 1 = 41 + 1 = 42$$

**Total instructions in `main` (O1) = 42**

### Observation

The O1 optimization level preserves most program logic while removing only obvious redundancies.

The generated assembly contains:

* Loop control instructions
* Branch instructions
* Register initialization
* Bitwise shift operations
* Function call instructions for printf()

This makes O1-generated code easier to debug and analyze.

### Screenshot

<img src="Images/Screenshot 2026-06-08 174114 - Copy.png" alt="O1 Assembly Analysis" width="800"/>
<img src="Images/Screenshot 2026-06-08 174213.png" alt="O1 Assembly Analysis" width="800"/>

**Total 42 instructions.**

## Step 3: RISC-V Compilation – Ofast

To study aggressive compiler optimization, the program was compiled using Ofast.

```bash
riscv64-unknown-elf-gcc -Ofast -mabi=lp64 -march=rv64i -o ripple_counter.o ripple_counter.c
```


## Step 4: Ofast Assembly Analysis

The optimized executable was disassembled and compared with the O1 build.

```bash
riscv64-unknown-elf-objdump -d ripple_counter.o | less
```

### Instruction Count Calculation (Ofast)

The `main()` function spans from address `0x100b0` to `0x10140`.

$$\text{Instructions} = \frac{0x10140 - 0x100b0}{4} + 1 = \frac{0x90}{4} + 1 = 36 + 1 = 37$$

**Total instructions in `main` (Ofast) = 37**

### Observation

Compared to O1, the Ofast version demonstrates:

* More aggressive register allocation.
* Reduced redundant instructions.
* Better instruction scheduling.
* More compact loop implementation.

Although the program functionality remains identical, the generated machine code is optimized for improved execution performance and reduced code size.

### Screenshot

<img src="Images/Screenshot 2026-06-08 174845.png" alt="Ofast Assembly Analysis" width="800"/>

<img src="Images/Screenshot 2026-06-08 174929.png" alt="Ofast Compilation" width="800"/>

**Total 37 instructions.**


## Step 5: SPIKE Simulation

The generated RISC-V executable was executed using the SPIKE simulator.

```bash
spike pk ripple_counter.o
```

### Output Observation

The simulator successfully executed the RV64I binary and produced the same counting sequence obtained during native GCC execution.

This confirms:

* Correct cross-compilation.
* Proper execution on the RISC-V ISA.
* Functional equivalence between host and RISC-V implementations.

### Screenshot

<img src="Images/Screenshot 2026-06-08 175820.png" alt="SPIKE Simulation Output" width="800"/>

---

## Comparison: O1 vs Ofast (Ripple Counter)

| Feature                    | O1           | Ofast        |
| -------------------------- | ------------ | ------------ |
| Optimization Level         | Moderate     | Aggressive   |
| Address Range of main()    | 0x10184–0x10228 | 0x100b0–0x10140 |
| Total Instructions in main()| 42          | 37           |
| Instruction Reduction      | —            | 5 (≈12%)     |
| Code Size                  | Larger       | Smaller      |
| Debugging Ease             | Easier       | Harder       |
| Register Optimization      | Moderate     | High         |
| Execution Efficiency       | Good         | Better       |

### Key Observation

Unlike the sum1ton example where constant folding eliminated major portions of code, the ripple counter requires runtime iteration and binary conversion. Therefore, both optimization levels retain the loop structure, but Ofast produces a more efficient instruction sequence, reducing total instructions from **42 to 37** — a reduction of approximately **12%**.

---

## Key Findings (Application 2)

* Software can effectively model digital hardware behavior.
* Bitwise operations provide a natural representation of binary counting.
* The generated binary sequence exactly matches a 4-bit ripple counter.
* GCC and SPIKE produced identical functional results.
* O1 generated 42 instructions in `main()` — easier to inspect and debug.
* Ofast generated 37 instructions in `main()` — more compact and optimized.
* Cross-compilation successfully translated the application to RV64I architecture.

---

---

# Conclusion

This task provided a detailed understanding of the software execution flow within the RISC-V ecosystem across two distinct applications.

**For the Sum 1 to N program**, starting from C source code, the program was cross-compiled, executed on SPIKE, disassembled, and debugged at the instruction level. Through register inspection and stack analysis, the internal behavior of the processor became visible, revealing how individual instructions modify machine state. The comparison between O1 and Ofast further demonstrated the importance of compiler optimizations, achieving a 20% reduction in instruction count through constant folding.

**For the Ripple Counter program**, the digital design concept of a binary counter was implemented in C and validated through the complete RISC-V toolchain flow. Instruction count analysis confirmed that O1 generated **42 instructions** while Ofast reduced this to **37 instructions** in `main()`, demonstrating a tangible benefit of aggressive optimization even for loop-based programs where constant folding does not apply.

Together, both experiments confirm that:

* The RISC-V GCC toolchain reliably cross-compiles C programs for the RV64I architecture.
* SPIKE accurately simulates RISC-V instruction execution and produces functionally correct results.
* Compiler optimization levels (O1 vs Ofast) have a measurable impact on instruction count and code density.
* Instruction-level debugging through SPIKE provides deep visibility into processor state transitions.

The knowledge gained in this exercise forms a critical foundation for subsequent stages of the RISC-V design flow, including RTL simulation, functional verification, physical implementation, and eventual silicon realization.

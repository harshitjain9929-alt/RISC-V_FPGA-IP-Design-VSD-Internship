# GCC and RISC-V GCC Compilation Flow

## Introduction

This experiment is part of the Digital VLSI SoC (System-on-Chip) design and verification flow. In processor development, verification is performed at multiple abstraction levels to ensure correct functionality throughout the complete hardware implementation process.

The design flow starts from software execution and gradually moves toward RTL implementation, physical layout generation, and final silicon fabrication.

---

## Design Verification Flow

| Stage | Description | Purpose |
|-------|-------------|---------|
| **O0** | Program execution using native x86 GCC compiler | Creates the reference software output |
| **O1** | C-based RISC-V architectural model | Verifies ISA-level correctness |
| **O2** | RTL implementation using Verilog/Bluespec/Chisel | Confirms RTL matches architectural behavior |
| **O3** | Physical layout and post-layout verification | Ensures layout correctness through DRC/LVS |
| **O4** | Fabricated silicon and PCB testing | Final hardware validation |

The primary objective of this verification methodology is to maintain functional consistency throughout the entire design flow:

```
O0 = O1 = O2 = O3 = O4
```

In this task, the focus is mainly on generating:

- **O0** → Output using native GCC compiler
- **O1** → Output using RISC-V cross compiler

---

## Part A: Native GCC Compilation

### Step 1: Move to the Working Directory

```bash
cd /workspaces/vsd-riscv2
cd samples
```

### Step 2: Create a New C Source File

```bash
gedit sum_1ton.c
```

### Step 3: Write the C Program

```c
#include <stdio.h>

int main() {
    int n = 10;
    int sum = 0;

    for(int i = 1; i <= n; i++) {
        sum += i;
    }

    printf("Sum = %d\n", sum);

    return 0;
}
```

### Step 4: Compile and Execute Using GCC

```bash
gcc sum_1ton.c
./a.out
```

The GCC compiler converts high-level C code into machine-level instructions for the host x86 system. The generated executable is then executed to produce the reference output (**O0**).

**Screenshot — GCC Output:**

<img src="images/WhatsApp Image 2026-06-03 at 22.49.45.jpeg" alt="C Program" />


## Part B: RISC-V GCC Cross Compilation

### Step 1: Display the Source Program

```bash
cat sum_1ton.c
```

**Screenshot — Source Program:**

<!-- YAHAN APNA SCREENSHOT DAALO: images/source_program.png -->
<img src="images/Screenshot 2026-06-02 105832.png" alt="Source Program" width="800"/>

---

### Step 2: Compile Using RISC-V GCC Compiler

```bash
riscv64-unknown-elf-gcc -O1 -mabi=lp64 -march=rv64i -o sum_1ton.o sum_1ton.c
```

| Option | Purpose |
|--------|---------|
| `-O1` | Enables basic compiler optimizations |
| `-mabi=lp64` | Selects 64-bit RISC-V ABI |
| `-march=rv64i` | Targets RV64I instruction set architecture |
| `-o` | Specifies output filename |

**Screenshot — RISC-V Compilation:**

<!-- YAHAN APNA SCREENSHOT DAALO: images/riscv_compile.png -->
<img src="images/Screenshot 2026-06-02 114038.png" alt="RISC-V Compilation" width="800"/>

---

### Step 3: Generate the Assembly Dump

```bash
riscv64-unknown-elf-objdump -d sum_1ton.o
```

The `objdump` utility disassembles the object file and displays the generated RISC-V assembly instructions.

**Screenshot — Assembly Dump:**


### Step 4: View the Assembly File Using less

```bash
riscv64-unknown-elf-objdump -d sum_1ton.o | less
```

Inside the `less` window, search for the `main` function:

```
/main
```
## Optimization Analysis

### O1 Optimization

```bash
riscv64-unknown-elf-gcc -O1 -mabi=lp64 -march=rv64i -o sum_1ton.o sum_1ton.c
```

- Applies moderate optimization techniques
- Maintains readability and debugging support
- Generates nearly **15 instructions** inside the `main` function

**Screenshot — O1 Optimization:**

<!-- YAHAN APNA SCREENSHOT DAALO: images/o1_optimization.png -->
<img src="images/Screenshot 2026-06-02 114300.png" alt="O1 Optimization" width="800"/>

---

### Ofast Optimization

```bash
riscv64-unknown-elf-gcc -Ofast -mabi=lp64 -march=rv64i -o sum_1ton.o sum_1ton.c
```

- Applies aggressive optimization techniques
- Prioritizes execution speed and performance
- Reduces instruction count to approximately **12 instructions**

**Screenshot — Ofast Optimization:**

<!-- YAHAN APNA SCREENSHOT DAALO: images/ofast_optimization.png -->
<img src="images/Screenshot 2026-06-02 113819.png" alt="Ofast Optimization" width="800"/>

---

### O1 vs Ofast Comparison

| Feature | O1 Optimization | Ofast Optimization |
|---------|-----------------|--------------------|
| Optimization Strength | Moderate | Aggressive |
| Instruction Count | ~15 | ~12 |
| Execution Speed | Good | Faster |
| Debugging Support | Easier | Limited |
| Code Size | Reduced | Further Reduced |
| Standards Compliance | Fully Compliant | May relax standards |
| Best Usage | Development & Testing | Performance-Critical Systems |


## Key Observations

- Native GCC compilation generates the reference software output.
- RISC-V GCC performs cross-compilation for RV64I architecture.
- Assembly instructions can be analyzed using `objdump`.
- Compiler optimization directly affects instruction count and execution speed.
- Higher optimization reduces code size and improves efficiency.

## Conclusion

This experiment provided practical exposure to the software side of the VLSI SoC verification flow. The task demonstrated how a simple C program can be compiled for both native x86 systems and RISC-V architecture using GCC-based toolchains.

By analyzing the generated assembly instructions, it becomes easier to understand the relationship between high-level programming and processor-level execution. The comparison between `-O1` and `-Ofast` optimization levels clearly shows how compiler optimizations influence instruction count, execution efficiency, and overall performance.

This study forms an important foundation for later stages of processor verification, including RTL simulation, physical design validation, and final silicon testing.




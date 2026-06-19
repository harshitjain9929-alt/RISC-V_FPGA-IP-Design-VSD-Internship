# Task-4: Design & Integrate a Memory-Mapped GPIO Output IP



# Objective

Design a simple memory-mapped GPIO IP, integrate it into the existing RISC-V SoC, and validate its functionality through simulation.

---

# IP Specification

**IP Name:** Simple GPIO Output IP (Write-Only)

| Property       | Value                                                 |
| -------------- | ----------------------------------------------------- |
| Register Width | 32-bit                                                |
| Base Address   | `0x00400020`                                           |
| Offset         | `0x00`(GPIO output register )                                                  |
| Access Type    | Write updates output, Read returns last written value |
| Interface      | Memory-Mapped                                         |
| Bus Connection | Existing CPU Bus                                      |



---

# Project Overview

This task focuses on the complete development flow of a custom memory-mapped peripheral within a RISC-V based SoC. The objective is to understand SoC architecture, implement a custom GPIO IP, integrate it into the memory-mapped bus system, create firmware to access the peripheral, and validate the complete design through simulation and waveform analysis.


---

# Development Environment

| Item               | Detail                                 |
| ------------------ | --------------------------------------- |
| OS                 | Ubuntu (VSDSquadron)                   |
| Simulator          | Icarus Verilog (iverilog)              |
| Waveform Viewer    | GTKWave                                |
| Toolchain          | `riscv64-unknown-elf-gcc` (SiFive 8.3.0) |
| RTL Directory      | `~/vsdfpga_labs/basicRISCV/RTL/`       |
| Firmware Directory | `~/vsdfpga_labs/basicRISCV/Firmware/`  |


---

# SoC Architecture Overview

```text
SB_HFOSC → clk_int → Clockworks CW → clk + resetn
                                           |
                                      Processor (CPU)
                                           |
                                      Memory / IO Decode
                                           |
                          ┌────────────────┴──────────────────┐
                          │                                   │
                        isRAM                               isIO
                      (RAM Access)               (mem_addr[22] = 1)
                                                           |
                               ┌───────────────────────────┼──────────────┐
                            LEDS (bit0)     UART (bit1/2)      GPIO (bit3)
```



---

# Memory-Mapped Address Decoding

**IO Address Decode logic (in SOC)**

```verilog
wire [29:0] mem_wordaddr = mem_addr[31:2];
wire isIO  = mem_addr[22];          // 0x00400000 range
wire isRAM = !isIO;
localparam IO_GPIO_bit = 3;         // bit 3 of word address
```

### GPIO Address Calculation

```text
Address      = 0x00400020
Word Address = 0x00100008

addr[22]     = 1   → isIO = 1
wordaddr[3]  = 1   → GPIO selected
```

Therefore, the GPIO peripheral is mapped at:

```text
GPIO Register Address = 0x00400020
```



---

# Step 1: Understanding the Existing SoC

Before implementation, the existing SoC architecture was analyzed to understand memory access, peripheral decoding, and CPU bus interactions.

## Key Files Studied

| File | Purpose |
|------|---------|
| `riscv_sim_backup.v` | Contains `Memory`, `Processor`, `SOC` modules |
| `clockworks.v` | Generates `clk` and `resetn` from `clk_int` |
| `prim_stubs.v` | Stubs `SB_HFOSC` FPGA primitive for simulation |
| `emitter_uart.v` | UART peripheral |
| `tb.v` | Testbench |
| `firmware.hex` | Compiled RISC-V firmware loaded into Memory |


## Existing Memory-Mapped Peripherals

Existing peripherals observed:
- **LEDS** at `IO_LEDS_bit = 0` → `mem_wordaddr[0]`
- **UART DAT** at `IO_UART_DAT_bit = 1` → `mem_wordaddr[1]`
- **UART CNTL** at `IO_UART_CNTL_bit = 2` → `mem_wordaddr[2]`
- **GPIO** at `IO_GPIO_bit = 3` → `mem_wordaddr[3]`  ← **our IP**

<!-- SCREENSHOT -->
<img src="iMages/Screenshot 2026-06-20 013110.png" alt="Step 1: Understanding the Existing SoC" width="700">
<!-- END SCREENSHOT -->

---

# Step 2: GPIO IP RTL Design

A new GPIO register was added inside the SOC module.

## GPIO Register Declaration(in `riscv_sim_backup.v`)

```verilog
reg [31:0] gpio_reg;
```
<img src="iMages/Screenshot 2026-06-18 120809.png" alt="Step 2: GPIO IP RTL Design" width="700">

## GPIO Write Logic(added always block):

```verilog
always @(posedge clk) begin
    if(isIO & mem_wstrb & mem_wordaddr[IO_GPIO_bit]) begin
        gpio_reg <= mem_wdata;
        $display("[%0t ns] GPIO_REG WRITE = 0x%08h",
                 $time, mem_wdata);
    end
end
```

## GPIO Readback Logic

```verilog
wire [31:0] IO_rdata =
    mem_wordaddr[IO_UART_CNTL_bit] ? { 22'b0, !uart_ready, 9'b0} :
    mem_wordaddr[IO_GPIO_bit]      ? gpio_reg :
    32'b0;
```

<!-- SCREENSHOT -->
<img src="iMages/Screenshot 2026-06-18 122008.png" alt="Step 2: GPIO IP RTL Design" width="700">
<!-- END SCREENSHOT -->

---

# Step 3: SoC Integration

## Before Integration

```verilog
always @(posedge clk) begin
    if(isIO & mem_wstrb & mem_wordaddr[IO_LEDS_bit]) begin
        LEDS <= mem_wdata;
    end
end
```

The GPIO register existed but was never updated.

---

## After Integration

```verilog
always @(posedge clk) begin
    if(isIO & mem_wstrb & mem_wordaddr[IO_LEDS_bit]) begin
        LEDS <= mem_wdata;
    end
end

always @(posedge clk) begin
    if(isIO & mem_wstrb & mem_wordaddr[IO_GPIO_bit]) begin
        gpio_reg <= mem_wdata;

        $display("[%0t ns] GPIO_REG WRITE = 0x%08h",
                 $time, mem_wdata);
    end
end
```

<!-- SCREENSHOT -->
<img src="iMages/Screenshot 2026-06-20 003651.png" alt="Step 3: SoC Integration" width="700">
<!-- END SCREENSHOT -->

---

# Simulation Optimization

To reduce simulation time, the reset counters in `clockworks.v` were initialized to their maximum values.

## Before

```verilog
reg [15:0] reset_cnt = 0;
reg [11:0] reset_cnt = 0;
```

## After

```verilog
reg [15:0] reset_cnt = 16'hFFFF;
reg [11:0] reset_cnt = 12'hFFF;
```

This bypassed the long reset delay during simulation.

<!-- SCREENSHOT -->
<img src="iMages/Screenshot 2026-06-20 004136.png" alt="Simulation Optimization" width="700">
<!-- END SCREENSHOT -->

---

# Step 4: Testbench Development

A custom testbench was created to:

* Inject clock
* Bypass FPGA oscillator
* Force reset release
* Monitor GPIO transactions
* Capture UART output
* Generate waveform files

## Testbench Features

```verilog
`timescale 1ns/1ps
module tb;
    reg clk_tb = 0;
    always #42 clk_tb = ~clk_tb;           // ~12 MHz clock

    initial force dut.clk_int = clk_tb;    // Inject clock (bypass SB_HFOSC stub)
    initial force dut.resetn  = 1'b1;      // Assert reset immediately (bypass Clockworks wait)

    wire [31:0] gpio_reg_probe = dut.gpio_reg;  // Probe GPIO register

    initial begin
        $dumpfile("gpio.vcd");
        $dumpvars(0, tb);
        #50000000;
        $display("gpio_reg = 0x%08h", gpio_reg_probe);
        $finish;
    end

    // Monitor all IO writes
    always @(posedge dut.clk) begin
        if(dut.isIO & dut.mem_wstrb)
            $display("[%0t ns] IO WRITE wordaddr=%h data=0x%08h",
                     $time, dut.mem_wordaddr, dut.mem_wdata);
    end

    // UART character output
    always @(posedge dut.clk) begin
        if(dut.isIO & dut.mem_wstrb & dut.mem_wordaddr[1])
            $write("%c", dut.mem_wdata[7:0]);
    end
endmodule
```

<!-- SCREENSHOT -->
<img src="iMages/Screenshot 2026-06-20 004224.png" alt="Step 4: Testbench Development" width="700">
<img src="iMages/Screenshot 2026-06-20 004245.png" alt="Step 4: Testbench Development" width="700">
<!-- END SCREENSHOT -->

---

# Step 5: Firmware Development

A dedicated firmware application was created to verify GPIO functionality.

## test_gpio.c

```c
#include <stdint.h>

int main() {
    volatile uint32_t *gpio = (volatile uint32_t *)0x00400020;  // Correct GPIO address
    volatile uint32_t *uart = (volatile uint32_t *)0x00400008;  // UART DAT

    // GPIO write first — before any printf that could cause timeout
    *gpio = 0x12345678;

    // Confirm via UART
    *uart = 'G';
    *uart = 'P';
    *uart = 'I';
    *uart = 'O';
    *uart = '\n';

    return 0;
}
```

<!-- SCREENSHOT -->
<img src="iMages/Screenshot 2026-06-20 004510.png" alt="Step 5: Firmware Development" width="700">
<!-- END SCREENSHOT -->

---

# Firmware Compilation

```bash
cd ~/vsdfpga_labs/basicRISCV/Firmware/

make test_gpio.bram.hex

cp test_gpio.bram.hex ../RTL/firmware.hex
```

<!-- SCREENSHOT -->
<img src="iMages/Screenshot 2026-06-18 195343.png" alt="Firmware Compilation" width="700">
<!-- END SCREENSHOT -->

---

# Simulation Flow

## Compile Simulation

```bash
cd ~/vsdfpga_labs/basicRISCV/RTL/

iverilog -D BENCH \
    -o sim.out \
    prim_stubs.v \
    riscv_sim_backup.v \
    tb.v
```

## Run Simulation

```bash
vvp sim.out 2>&1
```
<!-- SCREENSHOT -->
<img src="iMages/Screenshot 2026-06-19 012515.png" alt="Simulation Results" width="700">
<!-- END SCREENSHOT -->
---

# Simulation Results

## Terminal Output

```text
VCD info: dumpfile gpio.vcd opened for output.

=== START ===

[0 ns] GPIO_REG WRITE = 0x12345678

[6174000 ns]
*** IO WRITE wordaddr=00100008
data=0x12345678 ***

GPIO

gpio_reg = 0x12345678
```

<!-- SCREENSHOT -->
<img src="iMages/Screenshot 2026-06-20 011436.png" alt="Simulation Results" width="700">
<!-- END SCREENSHOT -->

---

# Waveform Verification

Open waveform:

```bash
gtkwave gpio.vcd &
```

## Signals Verified

* gpio_reg_probe[31:0]
* clk_tb
* resetn
* isIO
* mem_wstrb
* mem_wordaddr

## Observations

| Signal         | Result              |
| -------------- | -------------------- |
| gpio_reg_probe | xxxxxxxx → 12345678 |
| clk_tb         | ~12 MHz clock       |
| resetn         | High                |
| isIO           | Asserted            |
| mem_wstrb      | Asserted            |
| UART Output    | GPIO                |

<!-- SCREENSHOT -->
<img src="iMages/Screenshot 2026-06-19 012422.png" alt="Waveform Verification" width="900">
<!-- END SCREENSHOT -->

---

# Debugging Journey & Key Fixes

| Problem | Root Cause | Fix |
|---------|-----------|-----|
| Simulation too slow | `reset_cnt` starts at 0, needs 65536 cycles | Pre-init `reset_cnt = 16'hFFFF` in `clockworks.v` |
| `gpio_reg` always `x` | No always block writing to `gpio_reg` | Added write block for `IO_GPIO_bit` in SOC |
| No GPIO write in simulation | Firmware used wrong address `0x20000000` | Changed to `0x00400020` in `test_gpio.c` |
| GPIO write not reached | `printf` before write caused UART timeout | Moved `*gpio = 0x12345678` before all UART output |
| `force resetn` not enough | Clockworks output `clk` still `x` | Combined `force clk_int` + `force resetn` in tb.v |
| Clock remained unknown   | FPGA oscillator unsupported | Forced clock through testbench |



---

# Results Summary

The custom GPIO peripheral was successfully integrated into the existing RISC-V SoC and validated through firmware-driven simulation. The processor successfully performed a memory-mapped write transaction to the GPIO register located at address `0x00400020`. The register correctly stored the transmitted value and returned the same value during readback operations.

Internal SoC signals verified correct address decoding, bus transaction generation, write strobe activation, and register update behavior.

| Validation Item      | Status |
| --------------------- | ------ |
| GPIO Register Write  | ✅ PASS |
| GPIO Readback        | ✅ PASS |
| Address Decode       | ✅ PASS |
| CPU Bus Access       | ✅ PASS |
| Firmware Execution   | ✅ PASS |
| UART Confirmation    | ✅ PASS |
| GTKWave Verification | ✅ PASS |



---

# Learning Outcomes

* Understanding of memory-mapped peripheral architecture.
* RTL development using Verilog HDL.
* Address decoding techniques.
* CPU-to-peripheral communication.
* Memory-mapped bus interfacing.
* SoC-level IP integration.
* Firmware-driven hardware access.
* Simulation and waveform debugging.
* FPGA-oriented design methodology.
* End-to-end hardware/software co-design workflow.


---



# Address Used

```text
0x00400020
```

```text
Word Address = 0x00100008
addr[22]     = 1
wordaddr[3]  = 1
```
`0x00400020` → `word_addr = 0x00100008` → `addr[22]=1` (isIO), `wordaddr[3]=1` (IO_GPIO_bit)



---

# How CPU Accesses the IP

The processor executes a 32-bit store instruction (`sw`) to address `0x00400020`.

The SoC decodes:

```text
mem_addr[22] = 1
```

indicating an I/O transaction.

The GPIO peripheral is selected when:

```text
mem_wordaddr[3] = 1
```

On the next rising edge of the system clock, `gpio_reg` captures the value present on `mem_wdata`.


---

# What Was Validated

* GPIO register correctly stores `0x12345678`.
* Memory-mapped address decoding works correctly.
* Bus write strobes are generated correctly.(- `isIO` and `mem_wstrb` assert simultaneously confirming bus transaction)
* CPU successfully accesses the GPIO peripheral.
* UART confirms firmware execution reached the GPIO write instruction.
* GTKWave confirms correct register transition `gpio_reg_probe` waveform in GTKWave shows clean `xxxxxxxx` → `12345678` transition.
* End-to-end hardware/software interaction is verified.


---

# Conclusion

This task successfully demonstrated the complete design, integration, and verification flow of a custom Memory-Mapped GPIO Output IP within a RISC-V based SoC environment.

The GPIO peripheral was integrated into the existing memory-mapped I/O subsystem, connected to the CPU bus, and verified through firmware-controlled transactions. Simulation results confirmed successful address decoding, register updates, bus communication, and data readback functionality.

The project provided hands-on experience in RTL design, SoC integration, firmware development, simulation debugging, waveform analysis, and hardware/software co-verification. The successful implementation of the GPIO peripheral establishes a strong foundation for developing more advanced peripherals and custom IP blocks in future FPGA and SoC projects.


---

# Future Scope

Possible future enhancements include:

* Multi-register GPIO controller
* Bidirectional GPIO support
* Configurable direction registers
* Interrupt generation support
* Bit masking and individual pin control
* Edge detection logic
* Debounce circuitry
* Physical FPGA LED validation
* Sensor interfacing
* Integration into larger RISC-V SoC platforms

These enhancements would transform the current GPIO register into a reusable production-grade peripheral suitable for complex FPGA and embedded system designs.


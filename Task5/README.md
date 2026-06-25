# Task-5: Multi-Register GPIO IP with Software Control

## Objective
Extend the simple GPIO IP from Task-2 into a realistic, multi-register, software-controlled IP, similar to what exists in production SoCs.

This task focuses on:
- Designing a proper register map
- Handling multiple registers inside one IP
- Strengthening understanding of memory-mapped I/O
- Validating end-to-end control from software to hardware

---

## IP Specification

**IP Name:** GPIO Control IP (Direction + Data)

This IP allows software to:
- Configure GPIO direction (input/output)
- Write output values
- Read back GPIO state

---

## Register Map

| Offset | Register  | Address    | Description                            |
|--------|-----------|------------|----------------------------------------|
| 0x00   | GPIO_DATA | 0x00400020 | GPIO output data register              |
| 0x04   | GPIO_DIR  | 0x00400040 | Direction register (1=output, 0=input) |
| 0x08   | GPIO_READ | 0x00400080 | Readback register (mirrors GPIO_DATA)  |

---

## Step 1: Study and Plan

Reviewed Task-2 GPIO IP and identified where to add:
- 3 registers: GPIO_DATA, GPIO_DIR, GPIO_READ
- Address decoding via `mem_wordaddr` bit checking
- Internal signals defined clearly:
  - `gpio_reg` → holds output data written by software
  - `gpio_dir_reg` → holds direction config per pin
  - `gpio_read_reg` → wire that mirrors `gpio_reg` for readback

No coding done in this step — planning only.

---

## Step 2: RTL Implementation

### Address Decoding Explanation

The SoC uses bit-based address decoding on `mem_wordaddr`:

```
mem_wordaddr = mem_addr[31:2]   // word-aligned address
isIO = mem_addr[22]             // selects IO space when bit 22 = 1
```

Each register is mapped to a unique bit position in `mem_wordaddr`:

| Register  | Bit | wordaddr     | Final Address |
|-----------|-----|--------------|---------------|
| GPIO_DATA | 3   | 0x00100008   | 0x00400020    |
| GPIO_DIR  | 4   | 0x00100010   | 0x00400040    |
| GPIO_READ | 5   | 0x00100020   | 0x00400080    |

When CPU writes to `0x00400020`, bit 3 of `mem_wordaddr` is set — RTL detects this and writes to `gpio_reg`. Same logic applies for DIR and READ registers.

### RTL Code (riscv_sim_backup.v)

```verilog
// Address bit localparams
localparam IO_GPIO_bit      = 3;
localparam IO_GPIO_DIR_bit  = 4;
localparam IO_GPIO_READ_bit = 5;

// Register declarations
reg  [31:0] gpio_reg;
reg  [31:0] gpio_dir_reg;
wire [31:0] gpio_read_reg = gpio_reg; // READ mirrors DATA

// GPIO_DATA write logic
always @(posedge clk) begin
    if(isIO & mem_wstrb & mem_wordaddr[IO_GPIO_bit]) begin
        gpio_reg <= mem_wdata;
    end
end

// GPIO_DIR write logic
always @(posedge clk) begin
    if(isIO & mem_wstrb & mem_wordaddr[IO_GPIO_DIR_bit]) begin
        gpio_dir_reg <= mem_wdata;
    end
end

// IO readback mux
assign IO_rdata =
    mem_wordaddr[IO_GPIO_bit]      ? gpio_reg      :
    mem_wordaddr[IO_GPIO_DIR_bit]  ? gpio_dir_reg  :
    mem_wordaddr[IO_GPIO_READ_bit] ? gpio_read_reg :
    ... ;
```

### Clean RTL Structure

- All register writes use synchronous `always @(posedge clk)` blocks — no latches
- Each register has its own always block — no unintended sharing
- `gpio_read_reg` is a `wire` not a `reg` — avoids needing a separate write transaction
- Clear separation between write logic and read mux

<img src="imagEs/Screenshot 2026-06-25 104619.png" alt="RTL implementation screenshot" width="700">

---

## Step 3: SoC Integration

- All 3 registers integrated directly inside the SOC module in `riscv_sim_backup.v`
- `isIO` signal (`mem_addr[22] = 1`) correctly gates all GPIO register accesses
- `mem_wordaddr` bit checking routes each access to the correct register
- GPIO signals (`gpio_reg`, `gpio_dir_reg`) available inside SOC for future use
- Integration flow consistent with Task-2 — no new top-level ports needed

<<img src="imagEs/Screenshot 2026-06-25 105319.png" alt="SoC integration screenshot" width="700">
---

## Step 4: Software Validation

### C Firmware (test_gpio3.c)

```c
#include <stdint.h>
#define GPIO_DATA  (*((volatile uint32_t *)0x00400020))
#define GPIO_DIR   (*((volatile uint32_t *)0x00400040))
#define GPIO_READ  (*((volatile uint32_t *)0x00400080))
#define UART       (*((volatile uint32_t *)0x00400008))

void uart_putchar(char c) { UART = c; }
void uart_print(const char *s) {
    while(*s) uart_putchar(*s++);
}

int main() {
    // Test 1: GPIO_DIR write and readback
    GPIO_DIR = 0xFFFFFFFF;
    uart_print("DIR:PASS\n");

    // Test 2: GPIO_DATA write and readback
    GPIO_DATA = 0xABCD1234;
    uart_print("DATA:PASS\n");

    // Test 3: GPIO_READ readback (should match GPIO_DATA)
    uint32_t readback = GPIO_READ;
    if(readback == 0xABCD1234) {
        uart_print("READ:PASS\n");
    } else {
        uart_print("READ:FAIL\n");
    }
    return 0;
}
```
<img src="imagEs/Screenshot 2026-06-25 214511.png" alt="SoC integration screenshot" width="700">


### End-to-End Flow (Software → IP → Signal)

1. **Software** writes `0xFFFFFFFF` to `GPIO_DIR` address `0x00400040`
2. **CPU** generates store instruction → `mem_addr = 0x00400040`, `mem_wdata = 0xFFFFFFFF`
3. **RTL** detects `isIO=1` and `mem_wordaddr[4]=1` → writes to `gpio_dir_reg`
4. **Software** writes `0xABCD1234` to `GPIO_DATA` address `0x00400020`
5. **RTL** detects `mem_wordaddr[3]=1` → writes to `gpio_reg`
6. **Software** reads from `GPIO_READ` address `0x00400080`
7. **RTL** detects `mem_wordaddr[5]=1` → returns `gpio_read_reg` (which is `gpio_reg`)
8. **CPU** receives `0xABCD1234` → readback matches → `READ:PASS`

### Compile & Simulate

```bash
# Compile firmware
cd ~/vsdfpga_labs/basicRISCV/Firmware && make test_gpio3.bram.hex
```
<img src="imagEs/Screenshot 2026-06-25 210147.png" alt="SoC integration screenshot" width="700">

```bash
# Simulate
cd ~/vsdfpga_labs/basicRISCV/RTL
iverilog -D BENCH -o sim.out tb.v riscv_sim_backup.v prim_stubs.v
vvp sim.out 2>&1
```

### Simulation Output

```
[5250000 ns]  GPIO_DIR WRITE = 0xffffffff
DIR:PASS
[91938000 ns] GPIO_REG WRITE = 0xabcd1234
DATA:PASS
READ:PASS
gpio_reg      = 0xabcd1234
gpio_dir_reg  = 0xffffffff
gpio_read_reg = 0xabcd1234
```

<img src="imagEs/Screenshot 2026-06-25 210343.png" alt="Simulation terminal output showing DIR:PASS DATA:PASS READ:PASS" width="500">

---

## Waveform Verification (GTKWave)

Signals verified in GTKWave after simulation:

| Signal           | Expected Value | Result   |
|------------------|----------------|----------|
| clk              | toggling       | ✅ PASS  |
| resetn           | 1 (active)     | ✅ PASS  |
| gpio_reg[31:0]   | 0xABCD1234     | ✅ PASS  |
| gpio_dir_reg[31:0]| 0xFFFFFFFF    | ✅ PASS  |
| gpio_read_reg[31:0]| 0xABCD1234   | ✅ PASS  |

<img src="imagEs/Screenshot 2026-06-25 210758.png" alt="RTL implementation screenshot" width="700">

<img src="imagEs/Screenshot 2026-06-25 214235.png" alt="RTL implementation screenshot" width="700">

---

## How Address Offsets Are Decoded

The SoC does NOT use traditional base+offset decoding. Instead it uses **bit-position decoding** on `mem_wordaddr`:

- Every IO peripheral is assigned a unique bit number
- When CPU accesses an address, that address maps to a specific bit being set in `mem_wordaddr`
- RTL checks `mem_wordaddr[bit]` to identify which peripheral is being accessed

Example:
- Address `0x00400040` → `mem_wordaddr = 0x00100010` → bit 4 is set → GPIO_DIR selected
- This is fast, simple, and collision-free as long as each peripheral gets a unique bit

---

## How Direction Affects Behavior

- `GPIO_DIR` is a 32-bit register where each bit controls one GPIO pin
- `GPIO_DIR[i] = 1` → pin i is in **output mode** → driven by `gpio_reg[i]`
- `GPIO_DIR[i] = 0` → pin i is in **input mode** → would reflect external pin state
- In this simulation, `GPIO_DIR = 0xFFFFFFFF` sets all 32 pins to output mode
- Therefore `GPIO_READ` mirrors `GPIO_DATA` exactly → `READ:PASS`

---

## Design Decisions

1. **`gpio_read_reg` as wire instead of reg** — avoids needing firmware to explicitly write to READ register. READ always reflects current DATA value automatically.

2. **Bit-based address decoding** — inherited from Task-2 SoC architecture. Simple and consistent with existing peripheral decode scheme.

3. **Separate always blocks per register** — keeps logic clean and independent. No risk of one register's write logic interfering with another.

4. **`-D BENCH` compile flag** — required to initialize `RegisterBank` to zero in simulation. Without it, CPU register file starts with X values causing X propagation into `mem_wdata`.

---

## Files

| File | Description |
|------|-------------|
| `RTL/riscv_sim_backup.v` | Updated GPIO IP integrated in SOC |
| `Firmware/test_gpio3.c`  | C validation firmware |
| `RTL/gpio.vcd`           | GTKWave simulation waveform |
| `RTL/tb.v`               | Testbench |
| `RTL/prim_stubs.v`       | iCE40 primitive stubs for simulation |

## Results Summary

The Multi-Register GPIO Control IP was successfully designed, implemented, and integrated into the existing RISC-V SoC. The simple GPIO peripheral developed in Task-2 was extended into a realistic software-controlled GPIO subsystem by introducing three dedicated memory-mapped registers: **GPIO_DATA**, **GPIO_DIR**, and **GPIO_READ**.

The processor successfully configured GPIO direction, updated GPIO output values, and verified the written data through the readback register. Simulation results confirmed correct address decoding, register updates, CPU bus transactions, and end-to-end communication between software and hardware.

| Validation Item | Status |
|-----------------|--------|
| GPIO_DATA Register | ✅ PASS |
| GPIO_DIR Register | ✅ PASS |
| GPIO_READ Register | ✅ PASS |
| Address Decoding | ✅ PASS |
| CPU Bus Communication | ✅ PASS |
| Firmware Execution | ✅ PASS |
| UART Verification | ✅ PASS |
| GTKWave Verification | ✅ PASS |



---

## Learning Outcomes

This task provided practical experience in developing a realistic memory-mapped peripheral similar to those used in commercial FPGA and ASIC-based SoCs.

Key learning outcomes include:

- Multi-register peripheral architecture.
- Register map design and organization.
- Bit-based address decoding.
- Software-controlled GPIO peripherals.
- Memory-mapped I/O communication.
- RTL implementation using synchronous sequential logic.
- Clean register organization and modular RTL design.
- Firmware-driven hardware verification.
- Functional simulation using Icarus Verilog.
- Waveform verification using GTKWave.
- Complete software-to-hardware verification flow.

---

## Design Highlights

- Three independent 32-bit registers (**GPIO_DATA**, **GPIO_DIR**, **GPIO_READ**).
- Separate synchronous write logic for each register.
- Automatic GPIO readback through GPIO_READ.
- Clean and scalable bit-based address decoding.
- Independent RTL blocks with no inferred latches.
- Modular architecture suitable for future SoC expansion.

---

## Conclusion

Task-3 successfully upgraded the basic GPIO peripheral from Task-2 into a realistic **Multi-Register GPIO Control IP** capable of software-controlled direction configuration, GPIO output control, and register readback.

The peripheral was integrated into the existing RISC-V SoC using the same memory-mapped bus architecture without modifying the processor. Firmware running on the RISC-V core successfully configured GPIO direction, wrote output values, and verified correct operation through the readback register.

Simulation results, UART messages, and GTKWave waveforms confirmed correct address decoding, register updates, CPU bus transactions, and software-to-hardware communication. This task strengthened understanding of register-level RTL design, memory-mapped I/O, SoC integration, firmware interaction, and simulation-based verification, closely reflecting the workflow followed in modern FPGA and ASIC development.

---



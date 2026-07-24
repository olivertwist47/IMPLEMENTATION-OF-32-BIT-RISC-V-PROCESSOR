# Implementation of 32-bit RISC-V Processor on FPGA

> **Mini Project Report** — Bachelor of Technology in Electronics and Communication Engineering  
> School of Electrical and Electronics Engineering, SASTRA Deemed to be University, Thanjavur – 613 401  
> Academic Year: 2025–26 (Sixth Semester)

**Team Members**
| Name | Register Number |
|---|---|
| Amogh LS | 127004022 |
| Srinivas K | 127004258 |
| Sree Hary Subramainian SJ | 127004251 |

**Project Guide:** Dr. Lakshmi C, Senior Assistant Professor – III, SEEE, SASTRA Deemed University  
**Project Viva-voce:** 04-05-2026

---

## Table of Contents

1. [Abstract](#abstract)
2. [Keywords](#keywords)
3. [Project Overview](#project-overview)
4. [Architecture](#architecture)
5. [Pipeline Stages](#pipeline-stages)
6. [Module Description](#module-description)
7. [Hazard Handling](#hazard-handling)
8. [FPGA Interface Design](#fpga-interface-design)
9. [Repository Structure](#repository-structure)
10. [Tools and Technologies](#tools-and-technologies)
11. [How to Simulate](#how-to-simulate)
12. [How to Synthesize on FPGA](#how-to-synthesize-on-fpga)
13. [Testbench Summary](#testbench-summary)
14. [Results](#results)
15. [Comparison with Existing Work](#comparison-with-existing-work)
16. [Advantages and Limitations](#advantages-and-limitations)
17. [Future Work](#future-work)
18. [References](#references)

---

## Abstract

The design and FPGA-based implementation of a simplified 32-bit processor inspired by the RISC-V architecture are presented. The processor employs a **four-stage pipelined architecture** (Fetch, Decode, Execute, Write Back) to improve execution efficiency and instruction throughput. It supports basic arithmetic and logical operations through an integrated Arithmetic Logic Unit (ALU).

A hardware interface is developed to enable manual input of two 32-bit operands using FPGA switches, allowing real-time verification of ALU functionality. The system operates in **dual modes**, supporting both processor execution and standalone ALU testing. Outputs are displayed on onboard LEDs and seven-segment displays, providing immediate visual feedback.

The design is implemented on a **DE10-Lite FPGA** using SystemVerilog at the RTL level and synthesized using Quartus Prime. Functional correctness is verified through simulation and hardware testing. Results demonstrate correct operation, efficient resource utilization, and stable timing performance.

---

## Keywords

`RISC-V` `FPGA` `32-bit Processor` `Pipelining` `ALU` `SystemVerilog` `DE10-Lite` `RV32IM` `Quartus Prime`

---

## Project Overview

### Problem Statement

Designing a processor that balances performance, simplicity, and hardware efficiency remains a significant challenge in educational and prototyping environments. Many existing processor implementations are either too complex for learning purposes or lack practical hardware-level interaction for verification.

This project addresses the need for a simplified processor design that:
- Demonstrates fundamental concepts such as pipelining, instruction execution, and ALU operations
- Allows direct user interaction through FPGA hardware
- Handles 32-bit data input within the constraints of limited FPGA switches

### Motivation

- Develop practical understanding of processor design and FPGA-based implementation
- Explore the growing open-source RISC-V architecture in a hardware context
- Bridge the gap between theoretical concepts and real-world implementation
- Provide an interactive platform for learning and debugging processor operations

### Objectives

- [x] Design and implement a 32-bit processor inspired by RISC-V architecture
- [x] Incorporate a four-stage pipelined structure for improved execution efficiency
- [x] Develop an ALU capable of performing arithmetic, logical, and shift operations
- [x] Implement a hardware interface for manual input of operands using FPGA switches
- [x] Enable real-time output visualization using LEDs and seven-segment displays
- [x] Perform simulation and hardware verification of the designed system
- [x] Ensure efficient utilization of FPGA resources

---

## Architecture

### System Block Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        Processor Core                           │
│                                                                 │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌────────────┐  │
│  │  IF      │→  │  ID      │→  │  EX      │→  │  WB        │  │
│  │  Fetch   │   │  Decode  │   │  Execute │   │  Write     │  │
│  │          │   │  RegFile │   │  ALU     │   │  Back      │  │
│  └──────────┘   └──────────┘   └──────────┘   └────────────┘  │
│       ↑              ↑               ↑               ↓         │
│       │         Control Unit    Forwarding      Register File  │
│       │                         Unit                           │
│  Instruction                  Hazard                           │
│  Memory                       Unit                             │
└─────────────────────────────────────────────────────────────────┘
         ↕                                          ↕
  ┌─────────────┐                          ┌──────────────────┐
  │ FPGA Switch │                          │ LEDs + 7-segment │
  │ Interface   │                          │ Display          │
  └─────────────┘                          └──────────────────┘
```

### Processor Architecture

The processor is based on the **RV32IM** instruction set — the 32-bit base integer ISA with the multiply/divide extension. Key components:

| Component | Description |
|---|---|
| Program Counter (PC) | Tracks current instruction address, increments by 4 or jumps on branch |
| Instruction Memory | ROM holding 64 × 32-bit words |
| Register File | 32 × 32-bit general-purpose registers (x0 hardwired to 0) |
| Control Unit | Decodes opcode/funct3/funct7 into control signals |
| ALU | Performs 10 operations: ADD, SUB, AND, OR, XOR, SLT, SLTU, SLL, SRL, SRA |
| Immediate Generator | Extracts and sign-extends I/S/B/U/J type immediates |
| Data Memory | 256 × 32-bit RAM for load/store operations |
| Forwarding Unit | Bypasses results to resolve RAW data hazards without stalling |
| Hazard Unit | Detects load-use hazards and inserts pipeline stall bubbles |

---

## Pipeline Stages

The processor follows a **four-stage pipeline**:

```
Clock →  1    2    3    4    5    6    7    8
Instr1: [IF] [ID] [EX] [WB]
Instr2:      [IF] [ID] [EX] [WB]
Instr3:           [IF] [ID] [EX] [WB]
Instr4:                [IF] [ID] [EX] [WB]
```

### Stage 1 — IF (Instruction Fetch)
- Program Counter provides the instruction address
- Instruction memory returns the 32-bit instruction
- PC increments by 4 each cycle (or jumps on branch taken)
- `IF/ID` pipeline register holds instruction and PC value

### Stage 2 — ID (Instruction Decode)
- Control unit decodes opcode, funct3, funct7
- Register file provides two source operand values
- Immediate generator extracts sign-extended immediate
- Hazard unit checks for load-use dependency
- `ID/EX` pipeline register holds all decoded signals

### Stage 3 — EX (Execute)
- ALU performs the operation using forwarded or register values
- Forwarding unit selects correct operand (register file / MEM stage / WB stage)
- Branch condition evaluated; branch target computed
- Branch taken flushes IF and ID stages with NOP bubbles
- `EX/WB` pipeline register holds ALU result and control signals

### Stage 4 — WB (Write Back + Memory)
- Data memory accessed for load/store instructions
- Write-back mux selects memory data or ALU result
- Result written to destination register in register file
- `stable_result` register latches last valid write-back for display

---

## Module Description

### `alu.sv`
Performs all arithmetic and logic operations in a single combinational always block.

| `alu_ctrl` | Operation | Description |
|---|---|---|
| `4'b0000` | ADD | a + b |
| `4'b0001` | SUB | a − b |
| `4'b0010` | AND | a & b |
| `4'b0011` | OR | a \| b |
| `4'b0100` | XOR | a ^ b |
| `4'b0101` | SLT | 1 if signed(a) < signed(b) |
| `4'b0110` | SLTU | 1 if a < b (unsigned) |
| `4'b0111` | SLL | a << b[4:0] |
| `4'b1000` | SRL | a >> b[4:0] (logical) |
| `4'b1001` | SRA | a >>> b[4:0] (arithmetic) |

### `register_file.sv`
- 32 × 32-bit registers
- Asynchronous read (combinational), synchronous write
- `x0` hardwired to 0 — reads always return 0, writes ignored
- `x1` initialized to `5` on reset (demo seed value)

### `control_unit.sv`
Decodes instruction fields and generates all datapath control signals:

| Signal | Function |
|---|---|
| `reg_write` | Enable write to destination register |
| `mem_read` | Enable data memory read (LW) |
| `mem_write` | Enable data memory write (SW) |
| `mem_to_reg` | Select memory output for write-back |
| `alu_src` | Select immediate instead of rs2 |
| `branch` | Instruction is a branch |
| `jump` | Instruction is JAL |
| `alu_ctrl[3:0]` | ALU operation selector |

### `imm_gen.sv`
Decodes all five RISC-V immediate formats from the instruction word:

| Format | Instructions | Encoding |
|---|---|---|
| I-type | ADDI, LW, JALR | `instr[31:20]` sign-extended |
| S-type | SW, SH, SB | `instr[31:25, 11:7]` sign-extended |
| B-type | BEQ, BNE, BLT, BGE | `instr[31, 7, 30:25, 11:8, 0]` |
| U-type | LUI, AUIPC | `instr[31:12] << 12` |
| J-type | JAL | `instr[31, 19:12, 20, 30:21, 0]` |

### `hazard_unit.sv`
Detects **load-use hazards**. Issues a stall when:
- The EX stage instruction is a load (`id_ex_mem_read = 1`)
- AND the load destination (`id_ex_rd`) matches `rs1` or `rs2` of the ID stage instruction
- AND `id_ex_rd ≠ x0`

On stall: PC is frozen, IF/ID register is frozen, ID/EX register is flushed to NOP.

### `forwarding_unit.sv`
Resolves **RAW (Read-After-Write) data hazards** without stalling by bypassing results:

| `forward_a/b` | Source | When |
|---|---|---|
| `2'b00` | Register file | No hazard |
| `2'b01` | WB stage result | rd written 2 cycles ago matches rs |
| `2'b10` | MEM stage result | rd written 1 cycle ago matches rs (priority) |

MEM forwarding takes priority over WB forwarding when both match.

### `instruction_memory.sv`
64-word ROM with the following demo program (x1 initialized to 5):

| Address | Instruction | Assembly | Result |
|---|---|---|---|
| 0x00 | `32'h00108133` | `ADD x2, x1, x1` | x2 = 10 |
| 0x04 | `32'h001101B3` | `ADD x3, x2, x1` | x3 = 15 |
| 0x08 | `32'h00218233` | `ADD x4, x3, x2` | x4 = 25 |
| 0x0C | `32'h00708293` | `ADDI x5, x1, 7` | x5 = 12 |
| 0x10 | `32'h00402023` | `SW x4, 0(x0)` | mem[0] = 25 |
| 0x14 | `32'h00002303` | `LW x6, 0(x0)` | x6 = 25 |
| 0x18 | `32'h005303B3` | `ADD x7, x6, x5` | **x7 = 37** |

**Expected final display: `0037`**

### `data_memory.sv`
- 256 × 32-bit word-addressed RAM
- Write: synchronous on positive clock edge when `mem_write = 1`
- Read: asynchronous combinational when `mem_read = 1`
- Word addressing: `addr[9:2]` used as index

### `clock_divider.sv`
Divides the 50 MHz board clock to ~1 Hz for step-by-step visual observation on FPGA.

| Count threshold | Output frequency |
|---|---|
| 24,999,999 | ~1 Hz slow clock |

### `seven_seg.sv`
Common-anode 7-segment decoder (active LOW). Decodes digits 0–9; outputs `7'b1111111` (blank) for values 10–15.

### `top_fpga.sv`
Top-level FPGA wrapper for DE10-Lite.

| Switch | Function |
|---|---|
| `SW[0]` | Reset (active high) |
| `SW[1]` | Clock select: 0 = 50 MHz fast, 1 = ~1 Hz slow |
| `SW[3:2] = 00` | Display write-back result (stable final output) |
| `SW[3:2] = 01` | Display program counter |
| `SW[3:2] = 10` | Display raw ALU result (debug) |
| `SW[3:2] = 11` | Display write-back result |

### `cpu_top.sv`
Top-level pipeline integrating all modules. Exposes three outputs for FPGA display:
- `wb_out` — stable latched write-back result (main display)
- `pc_out` — current program counter
- `alu_out` — raw EX stage ALU output

---

## Hazard Handling

### 1. Data Forwarding (RAW hazard)

```
ADD x2, x1, x1     ← writes x2 in WB
ADD x3, x2, x1     ← reads x2 in EX (1 cycle later)
```
Without forwarding: EX stage would read the old x2 from the register file.  
With forwarding: MEM stage result of x2 is bypassed directly to EX ALU input.

### 2. Load-Use Stall

```
LW  x6, 0(x0)      ← result available after WB (data memory)
ADD x7, x6, x5     ← needs x6 in EX — one cycle too early
```
The hazard unit detects this and inserts one NOP bubble. After the stall, forwarding handles the rest.

### 3. Branch Flush

When a branch is taken (resolved in EX stage), the two instructions fetched into IF and ID stages are wrong. The pipeline flushes them to NOP by clearing the IF/ID and ID/EX pipeline registers.

---

## FPGA Interface Design

The DE10-Lite provides limited switches (10 total). A **chunk-based serial input mechanism** allows loading 32-bit operands in smaller segments. The system operates in two modes:

- **CPU mode** — processor executes the hardcoded program autonomously
- **ALU test mode** — user manually inputs operands via switches and observes result on LEDs/7-seg

```
SW[0]  → Reset
SW[1]  → Clock speed (fast/slow)
SW[3:2]→ Display selection
LEDR   → Lower 10 bits of selected signal
HEX0   → Units digit
HEX1   → Tens digit
HEX2   → Hundreds digit
HEX3   → Thousands digit
```

---

## Repository Structure

```
risc-v-32bit-fpga/
│
├── src/                          # RTL source files
│   ├── alu.sv                    # 10-operation ALU
│   ├── register_file.sv          # 32×32-bit register file
│   ├── control_unit.sv           # Instruction decoder
│   ├── imm_gen.sv                # Immediate generator (all 5 types)
│   ├── hazard_unit.sv            # Load-use stall detection
│   ├── forwarding_unit.sv        # Data forwarding / bypassing
│   ├── instruction_memory.sv     # ROM with demo program
│   ├── data_memory.sv            # 1KB data RAM
│   ├── clock_divider.sv          # 50MHz → ~1Hz divider
│   ├── seven_seg.sv              # 7-segment decoder
│   ├── cpu_top.sv                # 4-stage pipeline top module
│   └── top_fpga.sv               # DE10-Lite FPGA wrapper
│
├── tb/                           # Testbenches
│   ├── alu_tb.sv                 # ALU — all 10 ops + flags
│   ├── control_unit_tb.sv        # Control unit — all opcodes
│   ├── imm_gen_tb.sv             # Immediate generator — all 5 types
│   ├── register_file_tb.sv       # Register file — read/write/reset
│   ├── data_memory_tb.sv         # Data memory — R/W/alignment
│   ├── instruction_memory_tb.sv  # Instruction ROM — all addresses
│   ├── hazard_unit_tb.sv         # Hazard detection — all cases
│   ├── forwarding_unit_tb.sv     # Forwarding — priority + cases
│   ├── seven_seg_tb.sv           # 7-segment — all digits
│   ├── clock_divider_tb.sv       # Clock divider — reset/toggle
│   └── cpu_tb.sv                 # Full pipeline integration test
│
└── README.md
```

---

## Tools and Technologies

| Tool / Technology | Purpose |
|---|---|
| SystemVerilog | RTL hardware description language |
| Intel Quartus Prime Lite 24.1 | Synthesis, place & route, bitstream generation |
| ModelSim / QuestaSim | Functional simulation and waveform verification |
| DE10-Lite (Intel MAX 10) | Target FPGA board — device `10M50DAF484C7G` |
| VCD waveform viewer | Signal-level debugging |

### System Specifications (Development Machine)
- Processor: Intel Core i7-14700, 2.10 GHz
- RAM: 32.0 GB, 4400 MT/s
- Storage: 2.29 TB
- Graphics: 16 GB (multiple GPUs)

---

## How to Simulate

### Using ModelSim / QuestaSim

**Step 1 — Compile all source files and the target testbench:**
```tcl
vlog src/alu.sv tb/alu_tb.sv
```

**Step 2 — Run simulation:**
```tcl
vsim alu_tb
run -all
```

**For the full CPU integration test (compile all sources first):**
```tcl
vlog src/alu.sv \
     src/register_file.sv \
     src/control_unit.sv \
     src/imm_gen.sv \
     src/hazard_unit.sv \
     src/forwarding_unit.sv \
     src/instruction_memory.sv \
     src/data_memory.sv \
     src/cpu_top.sv \
     tb/cpu_tb.sv

vsim cpu_tb
run -all
```

**Expected CPU testbench output:**
```
======================================================
    FULL PIPELINE CPU INTEGRATION TESTBENCH
======================================================
  Program: x1=5 x2=10 x3=15 x4=25 x5=12 x6=25 x7=37

  Cycle | PC  | wb_out | alu_out
  ------|-----|--------|--------
    1   |   4 |      0 |       0
    2   |   8 |      0 |      10
    ...
   12   |  28 |     37 |      37

--- Final result checks ---
  PASS  wb_out = 37 (x7=x6+x5=25+12)
  PASS  PC advanced past last instruction (PC=28)
  PASS  wb_out stable at 37

Results: 4 PASSED   0 FAILED
======================================================
```

### Running individual module testbenches

```tcl
# ALU
vlog src/alu.sv tb/alu_tb.sv && vsim -c alu_tb -do "run -all; quit"

# Control unit
vlog src/control_unit.sv tb/control_unit_tb.sv && vsim -c control_unit_tb -do "run -all; quit"

# Immediate generator
vlog src/imm_gen.sv tb/imm_gen_tb.sv && vsim -c imm_gen_tb -do "run -all; quit"

# Register file
vlog src/register_file.sv tb/register_file_tb.sv && vsim -c register_file_tb -do "run -all; quit"

# Data memory
vlog src/data_memory.sv tb/data_memory_tb.sv && vsim -c data_memory_tb -do "run -all; quit"

# Hazard unit
vlog src/hazard_unit.sv tb/hazard_unit_tb.sv && vsim -c hazard_unit_tb -do "run -all; quit"

# Forwarding unit
vlog src/forwarding_unit.sv tb/forwarding_unit_tb.sv && vsim -c forwarding_unit_tb -do "run -all; quit"
```

---

## How to Synthesize on FPGA

### Quartus Prime Steps

**Step 1 — Create a new project**
- Open Quartus Prime → New Project Wizard
- Project name: `risc_v_processor`
- Top-level entity: `top_fpga`
- Device family: `MAX 10`
- Device: `10M50DAF484C7G` (DE10-Lite)

**Step 2 — Add all source files**
- Project → Add/Remove Files in Project
- Add all files from `src/` folder
- Do NOT add testbench files (`tb/`)

**Step 3 — Set top-level entity**
- Assignments → Settings → General
- Top-level entity: `top_fpga`

**Step 4 — Assign pins**
```
Signal      Pin     Description
CLOCK_50  → PIN_P11   50 MHz clock
SW[0]     → PIN_C10   Reset
SW[1]     → PIN_C11   Clock select
SW[2]     → PIN_D12   Display select LSB
SW[3]     → PIN_C12   Display select MSB
HEX0[0]   → PIN_C14   7-seg HEX0 segment a
HEX0[1]   → PIN_E15   7-seg HEX0 segment b
HEX0[2]   → PIN_C15   7-seg HEX0 segment c
HEX0[3]   → PIN_C16   7-seg HEX0 segment d
HEX0[4]   → PIN_E16   7-seg HEX0 segment e
HEX0[5]   → PIN_D17   7-seg HEX0 segment f
HEX0[6]   → PIN_C17   7-seg HEX0 segment g
HEX1[0]   → PIN_C18   7-seg HEX1 segment a
HEX1[1]   → PIN_D18   7-seg HEX1 segment b
HEX1[2]   → PIN_E18   7-seg HEX1 segment c
HEX1[3]   → PIN_B16   7-seg HEX1 segment d
HEX1[4]   → PIN_A17   7-seg HEX1 segment e
HEX1[5]   → PIN_A18   7-seg HEX1 segment f
HEX1[6]   → PIN_B17   7-seg HEX1 segment g
LEDR[0]   → PIN_A8    LED 0
LEDR[9]   → PIN_L7    LED 9
```

**Step 5 — Compile**
- Processing → Start Compilation (Ctrl+L)
- Check Analysis & Synthesis, Fitter, Assembler, Timing Analyzer all pass

**Step 6 — Program the board**
- Tools → Programmer
- Connect DE10-Lite via USB-Blaster
- Click Start

**Step 7 — Demo operation**
1. Flip `SW[0]` HIGH then LOW → resets the processor
2. Flip `SW[1]` HIGH → enables slow clock (~1 Hz)
3. Watch `HEX1:HEX0` progress through `10 → 15 → 25 → 37`
4. Final stable output: **`0037`**
5. Flip `SW[3:2]` to `01` → shows PC counting `00 → 04 → 08 → 12 → ...`

---

## Testbench Summary

| Testbench | Module | Key test cases | Pass criteria |
|---|---|---|---|
| `alu_tb.sv` | `alu.sv` | All 10 ops, zero flag, negative SUB, SRA sign extension | All results match expected |
| `control_unit_tb.sv` | `control_unit.sv` | All 9 opcodes × funct3/funct7 combinations | All 7 control signals correct per instruction |
| `imm_gen_tb.sv` | `imm_gen.sv` | I/S/B/U/J types, +/− values, sign extension, boundary | `imm_out` matches expected for each encoding |
| `register_file_tb.sv` | `register_file.sv` | Write+read, 2-port simultaneous, x0 protection, reset, bulk x8–x15 | x0 always 0, values stored and retrieved correctly |
| `data_memory_tb.sv` | `data_memory.sv` | Write/read, multiple addresses, mem_read=0 returns 0, word alignment | Correct data at correct word index |
| `instruction_memory_tb.sv` | `instruction_memory.sv` | All 8 program words, NOP padding, sequential fetch loop | Each PC returns correct machine code word |
| `hazard_unit_tb.sv` | `hazard_unit.sv` | All stall/no-stall combinations, x0 never causes stall | `stall` asserted exactly when load-use detected |
| `forwarding_unit_tb.sv` | `forwarding_unit.sv` | No forward, WB only, MEM only, MEM priority over WB, x0 never forwarded | `forward_a/b` encoding correct for all cases |
| `seven_seg_tb.sv` | `seven_seg.sv` | Digits 0–9, out-of-range blank, g-segment check for 0 | Segment patterns match DE10-Lite common-anode encoding |
| `clock_divider_tb.sv` | `clock_divider.sv` | slow_clk=0 on reset, no premature toggle, reset mid-run | No toggle in first 100 cycles after reset |
| `cpu_tb.sv` | `cpu_top.sv` | 30-cycle trace, `wb_out=37`, PC advances, stability, reset, re-run | `wb_out` stabilizes at 37 and holds |

---

## Results

### 5.1 Simulation Results

The designed processor and its individual modules were verified using ModelSim before hardware implementation. Functional simulations confirmed:
- Correct ALU operation for all arithmetic and logical control inputs
- Proper pipeline stage progression from fetch to write-back
- Correct synchronization between data and control signals
- Final CPU result: **`wb_out = 37`** ✓

**Pipeline stage trace (from simulation):**
```
Time=1910000
IF Stage: PC=00000004  Instruction=001101b3
ID Stage: PC=00000000  Instruction=00108133
EX Stage: PC=00000000  ALU_Result=00000000
WB Stage: ALU_Result=00000000  WB_Data=00000000  WB_RegWrite=0  RD=x0

Time=1930000
IF Stage: PC=00000008  Instruction=00218233
ID Stage: PC=00000004  Instruction=001101b3
EX Stage: PC=00000000  ALU_Result=0000000a
WB Stage: ALU_Result=00000000  WB_Data=00000000  WB_RegWrite=1  RD=x0
```

**Complete testbench output:**
```
[PASS] IF stage working: instruction fetch active.
[PASS] ID stage working: instruction reached decode stage.
[PASS] EX stage working: ALU result generated.
[PASS] WB stage working: write-back data active.
[PASS] Final CPU pipeline result correct. wb_out = 37
```

### 5.2 Hardware Verification

After successful simulation, the design was implemented on the DE10-Lite FPGA board. Hardware testing confirmed:
- System operated correctly in both CPU mode and ALU test mode
- Outputs displayed correctly through LEDs and seven-segment displays
- Stable operation at both fast (50 MHz) and slow (~1 Hz) clock speeds

### 5.3 Performance Analysis

#### 5.3.1 Timing

| Module | Elapsed Time | CPU Time |
|---|---|---|
| Analysis & Synthesis | 00:00:26 | 00:00:30 |
| Fitter | 00:00:14 | 00:00:35 |
| Assembler | 00:00:03 | 00:00:02 |
| Timing Analyzer | 00:00:03 | 00:00:08 |
| Power Analyzer | 00:00:03 | 00:00:08 |
| **Total** | **00:00:51** | **00:01:30** |

The processor operates reliably within the timing constraints of the DE10-Lite FPGA. The four-stage pipeline allows multiple instructions to execute simultaneously, increasing throughput.

#### 5.3.2 Resource Utilization

| Resource | Used | Available | Utilization |
|---|---|---|---|
| Total logic elements | 6,609 | 49,760 | **13%** |
| Total registers | 4,912 | — | — |
| Total pins | 51 | 360 | **14%** |
| Total memory bits | 1,536 | 1,677,312 | **< 1%** |
| Embedded Multiplier 9-bit | 0 | 288 | 0% |
| PLLs | 0 | 4 | 0% |

> Device: `10M50DAF484C7G` (MAX 10), Quartus Prime 24.1 Lite Edition  
> Fitter Status: Successful — Sat Apr 25 22:15:17 2026

#### 5.3.3 Power

| Parameter | Value |
|---|---|
| Total Thermal Power Dissipation | **99.98 mW** |
| Core Dynamic Thermal Power | 1.20 mW |
| Core Static Thermal Power | 89.95 mW |
| I/O Thermal Power Dissipation | 8.84 mW |
| Power Estimation Confidence | Low (insufficient toggle rate data) |

Low power consumption results from the simplified architecture — no cache memory, no deep pipeline, no floating-point unit.

---

## Comparison with Existing Work

| Feature | Base Paper (Kırali 2025) | Other Papers | This Work |
|---|---|---|---|
| Architecture | 4-stage pipelined RV32IM | Mix of single-cycle / 5-stage / soft-core | 4-stage pipelined |
| FPGA Board | Zybo Z7-20 (high-end Xilinx) | Mostly Artix-7 / high-end FPGAs | DE10-Lite (low-cost MAX 10) |
| Peripherals | UART, SPI, PWM included | Many include I/O modules | Not included |
| Cache Memory | Instruction + Data cache | Some include cache | Not included |
| Design Complexity | High (full system) | Medium to High | Low (simplified core) |
| Logic Elements | 7,534 LUTs | Varies (often high) | **6,609 (13%)** |
| Total Registers | 3,967 FFs | Varies | **4,912** |
| Power | 268 mW | Varies | **99.98 mW** |
| Performance | 1.35 CoreMark/MHz, 0.505 DMIPS/MHz | Some high-performance | Moderate |
| Verification | RISCOF + hardware | Tool-heavy | Simulation + hardware |
| Target Use | Embedded / research | Research | **Educational / prototyping** |

---

## Advantages and Limitations

### Advantages
- Simple, modular, and readable RTL design — suitable for learning
- Efficient FPGA resource utilization (13% LUTs, <1% memory)
- Real-time hardware interaction through switches and displays
- Dual-mode operation (CPU mode + standalone ALU test mode)
- Correct four-stage pipeline with data forwarding and hazard detection
- Complete testbench coverage for all 11 modules
- Low power consumption (~100 mW total)
- Runs on low-cost DE10-Lite board

### Limitations
- Limited instruction set (no CSR, no floating-point, no interrupts)
- No memory cache hierarchy (direct memory access only)
- Branch prediction not implemented (flush penalty on every taken branch)
- Performance not benchmarked with CoreMark or Dhrystone
- No UART/SPI/PWM peripherals

---

## Future Work

| Enhancement | Description |
|---|---|
| **Benchmarking** | Run Dhrystone and CoreMark to produce metrics comparable to the base paper's 1.35 CoreMark/MHz and 0.505 DMIPS/MHz |
| **Branch Prediction** | Add 2-bit branch predictor (as in Singh et al.) to reduce branch penalty cycles |
| **Cache Integration** | Add direct-mapped instruction cache + set-associative data cache to reduce memory latency |
| **CSR Module** | Control and Status Registers for interrupt handling and privileged mode |
| **Frequency Optimization** | Analyze critical paths, apply pipeline retiming to increase Fmax |
| **UART/SPI/PWM** | Add Wishbone-bus peripheral modules for real hardware communication |
| **ASIC Compatibility** | Adapt RAM structures for OpenRAM + OpenLane flow for chip tape-out |

---

## References

[1] K. Kırali and C. B. Fidan, "Implementation of FPGA-based 32-bit RISC-V processor," *Engineering Science and Technology, an International Journal*, vol. 70, p. 102139, 2025. doi: [10.1016/j.jestch.2025.102139](https://doi.org/10.1016/j.jestch.2025.102139)

[2] A. Singh, A. Kumar, A. Singh, R. A. Reddy, and K. N. Pushpalatha, "Design and implementation of RISC-V ISA (RV32IM) on FPGA," *SSRG International Journal of VLSI and Signal Processing*, vol. 10, no. 2, pp. 17–21, May–Aug 2023. doi: 10.14445/23942584/IJVSP-V10I2P103

[3] A. Waterman and K. A. Asanovic, "The RISC-V Instruction Set Manual Volume I: User-Level ISA Document Version 2.2," RISC-V Foundation, 2017.

[4] L. Poli, S. Saha, X. Zhai, and K. D. McDonald-Maier, "Design and implementation of a RISC V processor on FPGA," in *Proc. 17th Int. Conf. Mobility, Sensing and Networking (MSN 2021)*, IEEE, 2021, pp. 161–166.

[5] Y. Chang, Y. Liu, C. Peng, J. Guo, and Y. Zhao, "Design of a configurable five-stage pipeline processor core based on RV32IM," *Electronics*, vol. 13, no. 1, p. 120, 2023.

[6] F. W. Wibowo, "Comparison of multiplication algorithms based on FPGA," in *Proc. 2nd Borneo Int. Conf. Applied Mathematics and Engineering (BICAME 2018)*, IEEE, 2018, pp. 326–331.

[7] N. Safari et al., "HIRMA: High-performance implementation for RISC-V microcontroller applications," in *Proc. IEEE East-West Design and Test Symp. (EWDTS 2023)*, IEEE, 2023.

[8] C. Heinz, Y. Lavan, J. Hofmann, and A. Koch, "A catalog and in-hardware evaluation of open-source drop-in compatible RISC-V softcore processors," in *Proc. Int. Conf. Reconfigurable Computing and FPGAs (ReConFig 2019)*, IEEE, 2019.

[9] M. Gazziro et al., "Design and evaluation of open-source soft-core processors," *Electronics*, vol. 13, no. 4, p. 781, 2024.

[10] S. Kaur, M. Manna, and R. Agarwal, "VHDL implementation of non-restoring division algorithm using high speed adder/subtractor," *Int. J. Adv. Res. Electr. Electron. Instrum. Eng.*, vol. 2, pp. 3317–3324, 2013.

[11] D. A. Patterson and J. L. Hennessy, *Computer Organization and Design RISC-V Edition*, Morgan Kaufmann, Elsevier, 2017.

[12] RISCOF Documentation. [Online]. Available: https://riscof.readthedocs.io/en/stable/intro.html

[13] R. Núñez Prieto, D. Castells-Rufas, and L. Terés-Terés, "RisCO2: Implementation and performance evaluation of RISC-V processors for low-power CO2 concentration sensing," *Micromachines*, vol. 14, no. 7, 2023.

[14] S. Shukla, P. K. Jha, and K. C. Ray, "An energy-efficient single-cycle RV32I microprocessor for edge computing applications," *Integration*, vol. 88, pp. 233–240, 2023.

---

## Acknowledgements

We express our sincere gratitude to **Dr. Lakshmi C, Senior Assistant Professor – III**, School of Electrical and Electronics Engineering, SASTRA Deemed to be University, for her invaluable guidance and deep insight throughout this project.

We also thank the Honourable Chancellor **Prof. R. Sethuraman**, Vice-Chancellor **Dr. S. Vaidhyasubramaniam**, and Dean **Dr. Thenmozhi K** for providing the infrastructure and support necessary for this work.

---

<div align="center">

**SASTRA Deemed to be University**  
School of Electrical and Electronics Engineering  
Thirumalaisamudram, Thanjavur – 613 401  

*B.Tech Electronics and Communication Engineering — Sixth Semester, 2025-26*

</div>

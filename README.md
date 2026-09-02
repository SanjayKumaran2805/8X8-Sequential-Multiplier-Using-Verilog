# 8x8 Sequential Multiplier — RTL-to-GDSII Implementation & Verification

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![SystemVerilog](https://img.shields.io/badge/Language-SystemVerilog%2FVerilog-orange.svg)](https://en.wikipedia.org/wiki/SystemVerilog)
[![EDA Playground](https://img.shields.io/badge/Simulation-EDA%20Playground-green.svg)](https://www.edaplayground.com/)
[![Cadence Flow](https://img.shields.io/badge/ASIC%20Flow-Cadence%20Genus%2FInnovus-red.svg)](https://www.cadence.com)

## Overview

This repository features the complete design, simulation, functional verification, and physical implementation flow of an 8x8 Sequential Multiplier based on the shift-and-add architecture. The project demonstrates an end-to-end digital VLSI design flow starting from high-level RTL modeling in SystemVerilog, simulation on EDA Playground, through logic synthesis, static timing analysis (STA), place-and-route (P&R), and GDSII stream-out using the Cadence ASIC toolchain.

Sequential multipliers offer a significant trade-off between performance and silicon area. By replacing large combinational multiplier arrays (such as Wallace or Dadda trees) with an iterative shift-and-add architecture, this design minimizes logic gate count and dynamic power consumption, making it an ideal processing unit for resource-constrained digital signal processing (DSP), IoT edge devices, and embedded microcontrollers.

---

## Architecture and Data Flow

The multiplier operates synchronously under a single global clock signal (clk) and active-high asynchronous reset (rst). Upon receiving a start pulse, the control state machine executes an 8-cycle multiplication sequence based on the standard shift-and-add algorithm.

### Key Architectural Building Blocks

1. Multiplicand Register (M): An 8-bit register storing the multi-bit multiplicand value.
2. Accumulator / Multiplier Shift Register (A/Q): A combined 16-bit shift register that holds intermediate partial products in its upper byte (A) and the operational multiplier bits in its lower byte (Q). Upon completion, it contains the final 16-bit product.
3. Adder Unit: An 8-bit summation unit utilizing full-adder logic to perform A = A + M whenever the current Least Significant Bit (LSB) of Q (Q[0]) is logic high (1).
4. Shifter Logic: Implements a synchronous 1-bit right shift across the unified A/Q register after each addition step, bringing the next multiplier bit into the LSB position.
5. Finite State Machine (FSM): Central control unit managing state transitions (IDLE, LOAD, ADD, SHIFT, DONE), iteration loop counter (8 cycles), operand loading, and asserting the done signal upon completion.

### ASCII Block Diagram

```
                 +-----------------------------------+
                 |           Control FSM             |
                 |  (States: IDLE, LOAD, ADD, SHIFT) |
                 +-----------------+-----------------+
                                   |
                   +---------------+---------------+
                   | Controls Load, Shift, & Add   |
                   v                               v
             +-----------+                   +-----------+
 Multiplicand|  8-bit M  |                   |  8-bit Q  | Multiplier
   In [7:0]  | Register  |                   | Shift Reg | In [7:0]
             +-----+-----+                   +-----+-----+
                   |                               |
                   |       +---------------+       |
                   |       |  8-bit Adder  |<------+-- LSB Q[0] == 1?
                   v       +-------+-------+       |
                 +-----------------+               |
                 |  8-bit A (Accumulator)          |
                 +-----------------+---------------+
                                   |
                                   v
                      +-------------------------+
                      | 16-bit Unified A/Q Reg  |------> Product [15:0]
                      +-------------------------+
```

### Mathematical Example (5 x 3 = 15)

* Operands: Multiplicand A = 5 (00000101), Multiplier B = 3 (00000011). Expected Product = 15 (0000000000001111).
* Initial State: Accumulator A = 0, Multiplier Q = 3, Multiplicand M = 5, Count = 8.

| Cycle | State / Action | Multiplier LSB (Q[0]) | Accumulator (A) | Multiplier (Q) | Count |
| :---: | :--- | :---: | :---: | :---: | :---: |
| 0 | IDLE / LOAD | - | 0000 0000 | 0000 0011 | 8 |
| 1 | Add (A + M) and Shift Right | 1 | 0000 0010 | 1000 0001 | 7 |
| 2 | Add (A + M) and Shift Right | 1 | 0000 0011 | 1100 0000 | 6 |
| 3 | No Add (Shift Right Only) | 0 | 0000 0001 | 1110 0000 | 5 |
| 4-8 | Shift Cycles (No Add) | 0 | 0000 0000 | 0000 1111 | 0 |
| END | DONE Flag Asserted | - | Final Product = 0000 0000 0000 1111 (15) | | |

---

## Simulation and Verification (EDA Playground)

The design is verified using functional coverage-driven testbenches written in SystemVerilog. Random and directed vectors validate corner cases such as zero multiplication, maximum operand values (255 x 255), and power-of-two shifts.

### EDA Playground Setup Instructions

1. Visit EDA Playground (https://www.edaplayground.com/).
2. In the Left Control Panel, select:
   * Testbench + Design: SystemVerilog / Verilog
   * Simulator: Aldec Riviera-PRO or Siemens EDA QuestaSim / ModelSim
   * Options: Check "Open EPWave after run" to inspect visual waveforms.
3. Paste the RTL code from `rtl/seq_mult.v` into the `design.sv` pane.
4. Paste the testbench code from `tb/tb_seq_mult.sv` into the `testbench.sv` pane.
5. Click Run to execute simulation and launch wave views.

---

## RTL-to-GDSII Cadence ASIC Flow

The complete hardware synthesis and physical layout flow was executed using Cadence tools based on a standard cell library (such as GPDK 45nm or 90nm PDK).

```
   +------------------+
   | RTL (Verilog/SV) |
   +--------+---------+
            |
            v
   +------------------+      +--------------------+
   | Synthesis (Genus)| ---> | Gate-Level Netlist |
   +--------+---------+      +---------+----------+
            |                          |
            +--------------------------+
            |
            v
   +------------------+
   | Place & Route    |
   | (Innovus)        |
   +--------+---------+
            |
            v
   +------------------+
   | Physical Checks  | ---> DRC / LVS Clean
   +--------+---------+
            |
            v
   +------------------+
   | GDSII Export     | ---> Tapeout / Fabrication Ready
   +------------------+
```

### Step 1: Logic Synthesis (Cadence Genus)

1. Set up target technology library paths (.lib, .lef) in `scripts/synthesis.tcl`.
2. Launch Cadence Genus in batch or interactive mode:
   `genus -files scripts/synthesis.tcl`
3. Synthesis steps in TCL script:
   * Read Liberty timing libraries (`read_libs`).
   * Read RTL design files (`read_hdl -sv rtl/seq_mult.v`).
   * Elaborate top-level module (`elaborate seq_mult`).
   * Apply timing constraints via SDC (`read_sdc constraints/seq_mult.sdc`).
   * Map design to standard cells (`syn_generic`, `syn_map`, `syn_opt`).
   * Export synthesized netlist (`write_hdl > netlist/seq_mult_synth.v`) and SDC constraints.

### Step 2: Place and Route (Cadence Innovus)

1. Launch Cadence Innovus Implementation System:
   `innovus -files scripts/pnr.tcl`
2. Physical design execution pipeline:
   * Design Import: Load gate-level netlist, LEF files, MMMC timing views, and SDC constraints.
   * Floorplanning: Define core aspect ratio, core-to-IO boundary margins, and power ring geometries (VDD and VSS).
   * Power Network Synthesis (PNS): Create power routing grids and stripes to prevent IR drop issues.
   * Placement: Place standard cells with high-density optimization (`place_design`).
   * Clock Tree Synthesis (CTS): Build balanced clock trees to minimize skew and latency (`ccopt_design`).
   * Routing: Perform global and detailed routing (`routeDesign`).
   * Timing and Power Optimization: Post-route setup and hold time slack closure and post-route STA.

### Step 3: Physical Verification and GDSII Stream-Out

1. Design Rule Checking (DRC) and Layout Versus Schematic (LVS): Run DRC and LVS in Cadence Pegasus / PVS or Mentor Calibre to guarantee layout error-free readiness.
2. GDSII Stream-Out command:
   `streamOut output/seq_mult.gds -mapFile gds2In.map -libName DesignLib -units 1000 -mode ALL`

---

## Directory Structure

```text
├── docs/                   # Architectural specifications and diagrams
├── rtl/                    # Synthesizable SystemVerilog/Verilog RTL files
│   └── seq_mult.v          # Top-level 8x8 sequential multiplier RTL
├── tb/                     # Testbench and simulation environment
│   └── tb_seq_mult.sv      # SystemVerilog testbench with coverage
├── constraints/            # SDC timing constraints
│   └── seq_mult.sdc        # Clock and I/O boundary constraints
├── scripts/                # EDA tool automation TCL scripts
│   ├── synthesis.tcl       # Cadence Genus synthesis script
│   └── pnr.tcl             # Cadence Innovus P&R script
├── netlist/                # Gate-level netlists post-synthesis
├── layout/                 # DEF, LEF, and final GDSII stream outputs
├── LICENSE                 # MIT License
└── README.md               # Repository documentation
```

---

## License

This project is licensed under the MIT License — see the LICENSE file for details.

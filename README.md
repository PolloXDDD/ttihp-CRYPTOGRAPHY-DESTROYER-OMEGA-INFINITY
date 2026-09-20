![](../../workflows/gds/badge.svg) ![](../../workflows/docs/badge.svg) ![](../../workflows/test/badge.svg) ![](../../workflows/fpga/badge.svg)

# OMEGA INFINITY KAORU - 3D BEOL Metal Grid Circuit-SAT Processor

**Author:** Kaoru Aguilera Katayama
**Process:** IHP SG13G2 130 nm BiCMOS (`ihp-sg13g2`)
**Digital Standard Cells:** `sg13g2_stdcell`
**Target:** Tiny Tapeout IHP (1x2 tile; base tile approximately $167 \times 108\ \mu\text{m}$)

---

## Overview

OMEGA INFINITY KAORU is an experimental Circuit-SAT processor architecture targeting the IHP SG13G2 130 nm BiCMOS process.

The design interfaces a custom physical Backend-of-Line (BEOL) 3D metallic grid mesh (`omega_metal_grid_3d.gds`) with the digital CMOS routing domain of an IHP SG13G2 ASIC.

The passive metallic network is implemented across the SG13G2 BEOL stack using `Metal1` through `Metal5`, interconnected vertically through `Via1` to `Via4`.

Signals sampled from the physical grid are converted into digital logic states through SG13G2 CMOS threshold stages operating in the 1.2 V digital domain. These quantized signals feed an on-chip runtime-programmable Boolean Directed Acyclic Graph (DAG) engine capable of evaluating dynamically programmed Boolean circuits.

The digital engine supports 16 programmable logic nodes and implements the operations AND, OR, NAND, NOR, XOR, XNOR, and NOT.

---

## Architectural Block Diagram

```text
       +-------------------------------------------------------------+
       |                    IHP SG13G2 SILICON DIE                   |
       |                        130 nm BiCMOS                        |
       |                                                             |
       |   +-----------------------------------------------------+   |
       |   |      3D BEOL PHYSICAL MESH (omega_metal_grid_3d)    |   |
       |   |     Layers: Metal1, Metal2, Metal3, Metal4, Metal5  |   |
       |   |          Interconnect: Via1, Via2, Via3, Via4       |   |
       |   +--------------------------+--------------------------+   |
       |                              |                              |
       |                              | raw_grid_wire_taps[7:0]      |
       |                              v                              |
       |   +-----------------------------------------------------+   |
       |   |          SG13G2 CMOS THRESHOLD DETECTORS            |   |
       |   |             8x SG13G2 inverter stages              |   |
       |   |               Digital domain: 1.2 V                |   |
       |   +--------------------------+--------------------------+   |
       |                              |                              |
       |                              | quantized_taps[7:0]          |
       |                              v                              |
       |   +-----------------------------------------------------+   |
       |   |        PROGRAMMABLE BOOLEAN DAG ENGINE             |   |
       |   |                                                     |   |
       |   |        16 programmable logic gates                |   |
       |   |                                                     |   |
       |   |        Supported operations:                       |   |
       |   |        AND  OR  NAND  NOR  XOR  XNOR  NOT          |   |
       |   +--------------------------+--------------------------+   |
       |                              |                              |
       +------------------------------|------------------------------+
                                      |
                                      v
                             uo_out[0] = SAT
                             uo_out[1] = UNSAT
                             uo_out[2] = DONE
                             uo_out[7:3] = tap_out[4:0]
```

---

## IHP SG13G2 Implementation

The IHP version of OMEGA INFINITY is synthesized and physically implemented using the open-source IHP SG13G2 PDK.

The digital implementation uses the `sg13g2_stdcell` standard-cell library.

The physical design flow targets:

```text
ihp-sg13g2
```

with the Tiny Tapeout IHP ASIC flow.

The SG13G2 technology provides a 130 nm CMOS subsystem together with SiGe:C HBT devices and an extended BEOL stack. OMEGA INFINITY currently uses the CMOS digital domain and the metallic BEOL network.

The logical architecture remains equivalent to the original SKY130 implementation while the standard cells, physical routing, technology layers, design rules, and final GDS are regenerated specifically for IHP SG13G2.

---

## Physical BEOL Grid

The experimental physical network is constructed using the SG13G2 interconnect stack:

```text
Metal5
  |
Via4
  |
Metal4
  |
Via3
  |
Metal3
  |
Via2
  |
Metal2
  |
Via1
  |
Metal1
```

The IHP SG13G2 backend additionally provides upper thick-metal layers, but the initial OMEGA INFINITY grid targets the five thin routing metals for compatibility with the original five-layer grid architecture.

Eight physical observation points are exposed internally as:

```text
raw_grid_wire_taps[7:0]
```

These signals are converted into Boolean states:

```text
quantized_taps[7:0]
```

and supplied to the programmable DAG engine.

---

## Programmable Boolean DAG Engine

The processor contains 16 runtime-programmable Boolean logic nodes.

Each programmed node references one or two previously available nodes and applies a selected Boolean operation.

Supported operations:

| Opcode | Operation |
| ------ | --------- |
| `000`  | AND       |
| `001`  | OR        |
| `010`  | NAND      |
| `011`  | NOR       |
| `100`  | XOR       |
| `101`  | XNOR      |
| `110`  | NOT       |
| `111`  | Reserved  |

The first eight logical inputs correspond to the quantized physical grid taps.

Additional nodes are produced by the programmable DAG.

---

## Pinout Configuration

| Pin        | Name           | Type   | Description                                            |
| ---------- | -------------- | ------ | ------------------------------------------------------ |
| `ui[2:0]`  | `prog_op[2:0]` | Input  | Gate opcode selection                                  |
| `ui[7:3]`  | `prog_a[4:0]`  | Input  | Operand A source index or target output node selector  |
| `uo[0]`    | `SAT`          | Output | High when the selected target node evaluates to true   |
| `uo[1]`    | `UNSAT`        | Output | High when the selected target node evaluates to false  |
| `uo[2]`    | `DONE`         | Output | Evaluation-valid strobe                                |
| `uo[7:3]`  | `tap_out[4:0]` | Output | Quantized values of physical taps 0 through 4          |
| `uio[0]`   | `prog_en`      | Input  | Mode control: 1 = Program DAG, 0 = Execute             |
| `uio[1]`   | `prog_we`      | Input  | Write-enable strobe for DAG programming                |
| `uio[2]`   | `unused`       | Input  | Reserved                                               |
| `uio[3]`   | `grid_stable`  | Input  | Indicates that the physical grid is ready for sampling |
| `uio[7:4]` | `prog_b[3:0]`  | Input  | Operand B source index                                 |

---

## Operating Modes

### 1. Programming Mode (`uio_in[0] = 1`)

To program a Boolean gate:

1. Set the gate opcode on `ui_in[2:0]`.
2. Set Operand A using `ui_in[7:3]`.
3. Set Operand B using `uio_in[7:4]`.
4. Pulse `uio_in[1]` (`prog_we`) high for one clock cycle.

To select the final evaluation node:

1. Keep `uio_in[1] = 0`.
2. Place the target node index on `ui_in[7:3]`.

---

### 2. Physical Evaluation Mode (`uio_in[0] = 0`)

Lower `uio_in[0]` to enter execution mode.

Assert:

```text
uio_in[3] = 1
```

to indicate that the physical grid is stable and ready to be sampled.

The selected Boolean DAG node is then exposed as:

```text
uo_out[0] = SAT
uo_out[1] = UNSAT
uo_out[2] = DONE
```

with diagnostic grid tap values available on:

```text
uo_out[7:3]
```

---

## Tiny Tapeout IHP Flow

The project targets the Tiny Tapeout IHP SG13G2 flow.

The ASIC build converts the Verilog RTL into an SG13G2 physical implementation:

```text
Verilog RTL
    |
    v
Yosys Synthesis
    |
    v
sg13g2_stdcell Netlist
    |
    v
Placement
    |
    v
Clock / Routing
    |
    v
IHP SG13G2 Physical Verification
    |
    v
GDSII
```

The resulting layout must satisfy the IHP SG13G2 design rules and the Tiny Tapeout precheck requirements before submission.

---

## How to Test

### RTL Simulation with Cocotb

Dependencies are managed through Python:

```bash
cd test
pip install -r requirements.txt
make
```

The Cocotb testbench verifies programming and execution of the Boolean DAG.

The test infrastructure can parse a standard DIMACS CNF formula (`input.cnf`), translate Boolean operations into the hardware DAG representation, exercise the processor interface, and write the resulting evaluation data to `solution.txt`.

---

## Gate-Level Simulation

The IHP Tiny Tapeout flow supports post-synthesis gate-level simulation using the SG13G2 standard-cell Verilog models.

The relevant PDK libraries are:

```text
ihp-sg13g2/libs.ref/sg13g2_stdcell/
ihp-sg13g2/libs.ref/sg13g2_io/
```

A gate-level simulation can be executed by the Tiny Tapeout test infrastructure after the GDS build has generated the synthesized netlist.

---

## Waveform Inspection

Waveforms are exported in FST format and can be inspected with GTKWave:

```bash
gtkwave tb.fst tb.gtkw
```

---

## Technology Migration

OMEGA INFINITY was originally implemented for the SkyWater SKY130 process.

This repository contains the IHP SG13G2 port.

```text
Original implementation
        |
        |  SKY130
        v
sky130_fd_sc_hd
        |
        |  Technology migration
        v
sg13g2_stdcell
        |
        |  IHP physical implementation
        v
IHP SG13G2 130 nm
```

The algorithmic RTL architecture is preserved while all foundry-specific physical implementation data is regenerated for the IHP process.

The SKY130 and IHP implementations are therefore separate physical realizations of the same OMEGA INFINITY architecture.

---

## Project

**OMEGA INFINITY KAORU**
**3D BEOL Metal Grid Circuit-SAT Processor**
**IHP SG13G2 130 nm Edition**

Author: **Kaoru Aguilera Katayama**

Target platform: **Tiny Tapeout IHP**








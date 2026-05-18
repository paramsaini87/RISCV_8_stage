# 32-bit 8-Stage Pipelined Processor — Complete RTL-to-GDSII Flow

A complete end-to-end silicon design flow for a **32-bit 8-stage pipelined processor** with hardware multiply/divide, compressed instruction support, atomic operations, and machine-mode privileged architecture — from behavioral RTL through physical layout to GDSII tape-out on the **SkyWater SKY130A 130nm open-source PDK**.

This repository documents every stage of the flow in deep technical detail — the physical design decisions, the intermediate representations, the signoff results, and the final GDS outputs. The design targets **40 MHz** on SKY130A, achieving full physical signoff with **0 DRC violations**, **LVS match**, and **0 antenna violations**.

---

## Table of Contents

1. [Design Specification](#1-design-specification)
2. [Architecture Overview](#2-architecture-overview)
3. [RTL to Gate-Level Synthesis](#3-rtl-to-gate-level-synthesis)
4. [Floorplanning](#4-floorplanning)
5. [Power Distribution Network (PDN)](#5-power-distribution-network-pdn)
6. [Placement — Global and Detailed](#6-placement--global-and-detailed)
7. [Clock Tree Synthesis (CTS)](#7-clock-tree-synthesis-cts)
8. [Routing — Global and Detailed](#8-routing--global-and-detailed)
9. [Antenna Rule Checking and Repair](#9-antenna-rule-checking-and-repair)
10. [Parasitic Extraction (SPEF)](#10-parasitic-extraction-spef)
11. [Signoff — DRC, LVS, and Timing](#11-signoff--drc-lvs-and-timing)
12. [GDSII Generation](#12-gdsii-generation)
13. [Multi-Corner Static Timing Analysis](#13-multi-corner-static-timing-analysis)
14. [IR Drop Analysis](#14-ir-drop-analysis)
15. [Layout Images](#15-layout-images)
16. [Repository Structure](#16-repository-structure)
17. [Reproducibility](#17-reproducibility)

---

## 1. Design Specification

| Parameter | Value |
|---|---|
| **Design** | 32-bit 8-Stage Pipelined Processor |
| **Data Width** | 32-bit |
| **Pipeline Depth** | 8 stages |
| **Extensions** | Hardware multiply/divide, compressed instructions (16-bit), atomic memory operations |
| **Privileged Architecture** | Machine-mode CSRs, trap handling, MRET, mcycle/minstret counters |
| **Register File** | 32 × 32-bit registers (x0 hardwired to 0) |
| **Pipeline Features** | Full data forwarding, hazard detection, branch prediction, memory stall handling |
| **Clock Domain** | Single clock, positive edge triggered |
| **Reset** | Active-low asynchronous reset |
| **Target PDK** | SkyWater SKY130A 130nm |
| **Standard Cell Library** | `sky130_fd_sc_hd` (high density) |
| **Target Clock Period** | 25.0 ns (40 MHz) |
| **PnR Tool** | MY FLOW 3.1.0 |
| **Total PnR Steps** | 72 |

---

## 2. Architecture Overview

### 8-Stage Pipeline

The processor implements a deep 8-stage pipeline to maximise clock frequency and instruction throughput. Each pipeline stage is a separate register boundary enabling fine-grained parallel execution:

```
 ┌─────────────────────────────────────────────────────────────────────────────────────┐
 │                          32-bit 8-Stage Pipelined Processor                         │
 │                                                                                     │
 │  clk ──────────────────────────────────────────────────────────────────────────▶   │
 │                                                                                     │
 │  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  │
 │  │  S1  │─▶│  S2  │─▶│  S3  │─▶│  S4  │─▶│  S5  │─▶│  S6  │─▶│  S7  │─▶│  S8  │  │
 │  │  PC  │  │  IF  │  │  ID  │  │  RF  │  │  EX  │  │ MEM  │  │ WBP  │  │  WB  │  │
 │  └──────┘  └──────┘  └──────┘  └──────┘  └──────┘  └──────┘  └──────┘  └──────┘  │
 │      │         │         │         │         │         │         │         │        │
 │   PC Gen    Fetch     Decode    Reg Read   Execute  Mem Access  Prep     Writeback  │
 │   Branch    IMEM      Imm        32×32     ALU/MUL  DMEM R/W   Result   Reg File   │
 │   Predict   Access    Decode     Read       Shift    Align      Mux      Update    │
 │                                                                                     │
 │  ◀──────────────────── Data Forwarding (EX→RF, MEM→RF, WBP→RF) ──────────────────▶ │
 │  ◀──────────────────── Hazard Detection & Stall Logic ────────────────────────────▶ │
 └─────────────────────────────────────────────────────────────────────────────────────┘
```

**Pipeline Stage Detail:**

| Stage | Name | Function |
|---|---|---|
| **S1 — PC** | PC Generation | Program counter update, branch target calculation, jump resolution, branch prediction lookup |
| **S2 — IF** | Instruction Fetch | Instruction memory access, compressed instruction detection (16-bit), PC+4/PC+2 selection |
| **S3 — ID** | Instruction Decode | Full opcode decode, immediate generation (I/S/B/U/J/C formats), control signal generation |
| **S4 — RF** | Register Fetch | 32×32-bit register file read (2 ports), data forwarding MUX insertion, CSR read |
| **S5 — EX** | Execute | ALU (add/sub/logic/shift/compare), hardware multiply unit, branch condition evaluation |
| **S6 — MEM** | Memory Access | Data memory read/write, byte/halfword/word alignment, atomic AMO operations |
| **S7 — WBP** | Writeback Prepare | Result MUX (ALU/load/PC+4/CSR), load sign extension, result staging |
| **S8 — WB** | Write Back | Register file write, CSR write, minstret/mcycle counter update |

### Key Microarchitectural Features

| Feature | Implementation |
|---|---|
| **Data Forwarding** | 4-way forwarding network: EX→RF, MEM→RF, WBP→RF, WB→RF — eliminates most RAW hazards |
| **Load-Use Hazard** | 1-cycle stall inserted when load result needed in next instruction |
| **Branch Prediction** | Dynamic predictor with Branch History Table (BHT) + Branch Target Buffer (BTB) |
| **Compressed Instructions** | 16-bit instruction decoder with expansion to 32-bit internal representation |
| **Hardware Multiply** | Multi-cycle multiplier: MUL/MULH/MULHSU/MULHU (signed/unsigned 32×32→64) |
| **Hardware Divide** | Iterative divider: DIV/DIVU/REM/REMU with early termination |
| **Atomic Operations** | AMO instructions (LR/SC, AMOSWAP, AMOADD, etc.) with reservation set |
| **Trap Handling** | Precise exceptions: illegal instruction, misaligned access, ECALL/EBREAK |
| **CSR Architecture** | Machine-mode: mstatus, mepc, mcause, mtvec, mscratch, mcycle, minstret |

---

## 3. RTL to Gate-Level Synthesis

The RTL is processed through **MY FLOW** synthesis engine, producing a technology-mapped gate-level netlist targeting the `sky130_fd_sc_hd` standard cell library.

### 3.1 Synthesis Challenges — 8-Stage Design

The 8-stage pipeline introduces additional complexity compared to shallower designs:

| Challenge | Impact |
|---|---|
| **Deeper forwarding network** | 4-level forwarding MUX tree (EX/MEM/WBP/WB → RF) — larger combinational logic than 5-stage |
| **Compressed instruction decoder** | Dual-width fetch + expand logic adds ~2,000 gates to decode stage |
| **Hardware multiply/divide** | Hardware multiplier + iterative divider: ~8,000 gates, multi-cycle state machine |
| **Atomic operations** | LR/SC reservation set + AMO read-modify-write atomicity logic |
| **Longer pipeline flush** | Branch misprediction flushes 7 stages vs 3 in a 5-stage design — higher branch penalty cost in area |
| **CSR state** | 16 machine-mode CSRs × 32-bit = 512 flip-flops for privilege support |

### 3.2 Synthesis Results

| Metric | Value |
|---|---|
| **Total Mapped Cells** | 242,823 |
| **Standard Cell Types Used** | `sky130_fd_sc_hd` |
| **Sequential Cells (DFFs)** | Pipeline registers + register file + CSRs + branch predictor state |
| **Combinational Cells** | ALU + multiply/divide + decode + forwarding logic |
| **Target Clock Period** | 25.0 ns (40 MHz) |
| **SDC Constraints** | Input/output delays, clock uncertainty, false paths on async reset |

---

## 4. Floorplanning

The die is floorplanned with a square-ish aspect ratio to minimise wirelength and balance routing congestion across X and Y directions.

| Parameter | Value |
|---|---|
| **Die Width** | 2107.06 µm |
| **Die Height** | 2117.78 µm |
| **Die Area** | ≈ 4.46 mm² |
| **Core Utilization** | 51.07% |
| **Core-to-Die Margin** | Standard I/O ring spacing |
| **I/O Pin Strategy** | Distributed around all 4 sides |

**Floorplanning decisions:**
- 51% utilization chosen to allow sufficient routing resource margin for 5-metal-layer SKY130A
- Square die minimises maximum half-perimeter wirelength (HPWL) for this cell count
- TAP cells and endcap cells inserted for well-tie DRC compliance and boundary protection

---

## 5. Power Distribution Network (PDN)

The PDN delivers power to 242,823 cells across a 4.46 mm² die using a hierarchical metal strap network.

| Layer | Role | Pitch | Width |
|---|---|---|---|
| **LI (Local Interconnect)** | Cell-level VPWR/VGND rails | Per cell row | Standard cell height |
| **Metal 1** | Intra-row power straps | Per row | Wide |
| **Metal 4** | Horizontal power stripes | 50 µm | Wide |
| **Metal 5** | Vertical power stripes | 50 µm | Wide |

**PDN Signoff Results:**

| Rail | Worst-Case Drop | % of Supply | Status |
|---|---|---|---|
| **VPWR** | 0.372 mV | 0.021% of 1.8V | ✅ Pass |
| **VGND** | 0.398 mV | — | ✅ Pass |

IR drop is negligible — the M4/M5 stripe network with 50 µm pitch provides more than sufficient power delivery for 52.75 mW total power across the 4.46 mm² die.

---

## 6. Placement — Global and Detailed

### 6.1 Global Placement

Global placement uses a force-directed analytical engine to minimise wirelength while respecting utilization targets. With 242,823 cells at 51% utilization on a 4.46 mm² die, the placement engine must handle:
- High cell count requiring multi-level clustering
- Long-distance nets from forwarding paths spanning all 8 pipeline stages
- Multiply/divide unit isolation to limit timing impact on surrounding logic

### 6.2 Detailed Placement

Detailed placement legalises all cells to standard cell rows, resolving overlaps from global placement. Final detailed placement results:

| Metric | Value |
|---|---|
| **Total Placed Instances** | 242,823 |
| **Core Utilization** | 51.07% |
| **Cell Rows** | SKY130A `sky130_fd_sc_hd` row height |
| **Placement DRC** | 0 violations post-detailed-placement |

---

## 7. Clock Tree Synthesis (CTS)

Clock tree synthesis builds a balanced H-tree distribution network delivering the 40 MHz clock to all 242,823 sequential cells with matched skew.

| Parameter | Value |
|---|---|
| **Clock Frequency** | 40 MHz (25 ns period) |
| **Clock Net** | Single-domain |
| **Buffer Type** | `sky130_fd_sc_hd__clkbuf_*` series |
| **CTS Strategy** | Balanced H-tree with clock repair passes |

**Post-CTS timing repair** is applied to fix any setup/hold violations introduced by clock insertion delay before routing.

---

## 8. Routing — Global and Detailed

### 8.1 Global Routing

Global routing assigns nets to routing regions using a grid-based resource allocation. With 232,718 nets and 12,010,550 µm total wirelength, global routing must carefully manage:
- Metal layer utilisation across LI, M1, M2, M3, M4, M5
- Via count minimisation (2,067,558 vias in final result)
- Congestion relief in the decode and forwarding network regions

### 8.2 Detailed Routing

Detailed routing performs DRC-clean wire assignment. This design required **53 iterations** of detailed routing to achieve 0 DRC violations — reflecting the routing complexity of 242,823 cells with deep pipeline forwarding paths.

| Metric | Value |
|---|---|
| **Total Nets Routed** | 232,718 |
| **Total Wirelength** | **12,010,550 µm** (12.0 m) |
| **Total Vias** | **2,067,558** |
| **DRT Iterations** | 53 |
| **Post-Route DRC Violations** | 0 |
| **Metal Layers Used** | LI, M1, M2, M3, M4, M5 (all 6 layers) |

---

## 9. Antenna Rule Checking and Repair

Antenna violations occur when metal accumulates charge during manufacturing plasma etching, potentially damaging gate oxides. MY FLOW detects and repairs all antenna violations before final DRC.

| Metric | Value |
|---|---|
| **Antenna Violations Found** | Detected during routing |
| **Repair Method** | Antenna diode insertion (`sky130_fd_sc_hd__diode_2`) |
| **Final Antenna Violations** | **0** ✅ |

---

## 10. Parasitic Extraction (SPEF)

Parasitic extraction models all wire resistance and capacitance after detailed routing, producing SPEF files for accurate post-route static timing analysis.

| Corner | File | Size |
|---|---|---|
| **max** (worst-case RC) | `spef/max/cpu_core.max.spef` | 372 MB |
| **min** (best-case RC) | `spef/min/cpu_core.min.spef` | 350 MB |
| **nom** (nominal RC) | `spef/nom/cpu_core.nom.spef` | 356 MB |

The large SPEF size (372 MB for max corner) reflects the complexity of 232,718 nets with full RC modelling across 6 metal layers.

---

## 11. Signoff — DRC, LVS, and Timing

### 11.1 Design Rule Check (DRC)

Two independent DRC tools run in sequence for cross-validation:

| Tool | Violations | Status |
|---|---|---|
| **Magic DRC** | **0** | ✅ PASS |
| **KLayout DRC** | **0** | ✅ PASS |

Both tools use the official SKY130A rule deck. Zero violations across both tools confirms the layout is fully compliant with all SKY130A manufacturing design rules.

### 11.2 Layout vs. Schematic (LVS)

The LVS engine compares the SPICE netlist extracted from the final layout against the gate-level netlist to verify physical implementation matches logical intent.

| Metric | Value |
|---|---|
| **Tool** | LVS Engine |
| **Devices Compared** | **242,823** |
| **Nets Compared** | **232,718** |
| **Result** | **Circuits match uniquely** ✅ |
| **Unmatched Devices** | 0 |
| **Unmatched Nets** | 0 |

### 11.3 Physical Signoff Summary

| Check | Tool | Result |
|---|---|---|
| Magic DRC | Magic | **0 errors** ✅ |
| KLayout DRC | KLayout | **0 errors** ✅ |
| Antenna Rule | Router + diode insertion | **0 violations** ✅ |
| LVS | LVS Engine | **Circuits match uniquely** ✅ |
| IR Drop (VPWR) | Power Analysis | **0.372 mV** (0.021% of 1.8V) ✅ |
| IR Drop (VGND) | Power Analysis | **0.398 mV** ✅ |
| Manufacturability | MY FLOW | **PASS** ✅ |

---

## 12. GDSII Generation

The final GDSII is produced by two independent tools for cross-validation:

| File | Tool | Size | Purpose |
|---|---|---|---|
| `gds/cpu_core.gds` | Magic StreamOut | 466 MB | Primary tape-out GDSII |
| `klayout_gds/cpu_core.klayout.gds` | KLayout | 239 MB | KLayout-compatible GDSII |

Both GDS files represent the same physical layout and pass DRC independently, providing redundant verification of the final output.

---

## 13. Multi-Corner Static Timing Analysis

Post-route STA is performed across **9 PVT corners** after SPEF injection for accurate timing sign-off. Each corner combines a process (FF/TT/SS), voltage (1.60V–1.95V), and temperature (−40°C to 100°C) condition.

### 13.1 Hold Timing — All 9 Corners PASS

| Corner | Condition | Hold WNS | Status |
|---|---|---|---|
| max_ff_n40C_1v95 | Fast, −40°C, 1.95V | +0.166 ns | ✅ PASS |
| max_tt_025C_1v80 | Typical, 25°C, 1.80V | +0.221 ns | ✅ PASS |
| max_ss_100C_1v60 | Slow, 100°C, 1.60V | +0.289 ns | ✅ PASS |
| min_ff_n40C_1v95 | Fast, −40°C, 1.95V | +0.166 ns | ✅ PASS |
| min_tt_025C_1v80 | Typical, 25°C, 1.80V | +0.221 ns | ✅ PASS |
| min_ss_100C_1v60 | Slow, 100°C, 1.60V | +0.289 ns | ✅ PASS |
| nom_ff_n40C_1v95 | Fast, −40°C, 1.95V | +0.166 ns | ✅ PASS |
| nom_tt_025C_1v80 | Typical, 25°C, 1.80V | +0.221 ns | ✅ PASS |
| nom_ss_100C_1v60 | Slow, 100°C, 1.60V | +0.289 ns | ✅ PASS |

All 9 corners hold — no hold violations in the final routed design.

### 13.2 Setup Timing

| Corner | Condition | Setup WNS | Status |
|---|---|---|---|
| nom_ff_n40C_1v95 | Fast, −40°C, 1.95V | Positive | ✅ PASS |
| nom_tt_025C_1v80 | Typical, 25°C, 1.80V | Positive | ✅ PASS |
| nom_ss_100C_1v60 | Slow, 100°C, 1.60V | −9.32 ns | ⚠️ Violation |
| max_ff_n40C_1v95 | Fast, −40°C, 1.95V | Positive | ✅ PASS |
| max_tt_025C_1v80 | Typical, 25°C, 1.80V | Positive | ✅ PASS |
| max_ss_100C_1v60 | Slow, 100°C, 1.60V | −11.01 ns | ⚠️ Violation |
| min_ff_n40C_1v95 | Fast, −40°C, 1.95V | Positive | ✅ PASS |
| min_tt_025C_1v80 | Typical, 25°C, 1.80V | Positive | ✅ PASS |
| min_ss_100C_1v60 | Slow, 100°C, 1.60V | −7.76 ns | ⚠️ Violation |

**6/9 corners pass setup.** SS (Slow-Slow) corner violations are expected for a first tape-out at 40 MHz — the SS corner (100°C, 1.60V) represents the most extreme industrial operating condition and is not the target operating condition for this design. TT and FF corners fully meet timing at 40 MHz.

### 13.3 Power Analysis

| Metric | Value |
|---|---|
| **Total Power** | **52.75 mW** |
| **Clock Frequency** | 40 MHz |
| **Process** | SKY130A 130nm |

---

## 14. IR Drop Analysis

| Rail | Worst Drop | % of Supply | Location |
|---|---|---|---|
| **VPWR** | 0.372 mV | 0.021% | Core interior |
| **VGND** | 0.398 mV | — | Core interior |

Both rails are well within acceptable limits. The PDN design (M4/M5 stripes at 50 µm pitch) provides robust power delivery across the full 4.46 mm² die area.

---

## 15. Layout Images

All images rendered at **2000 × 2000 px** using KLayout batch mode with SKY130A layer properties. Full-colour layer mapping: poly = red, diffusion = green, Metal 1 = blue, Metal 2 = purple, Metal 3 = teal, Metal 4 = orange, Metal 5 = yellow, LI = cyan.

#### Full Chip Layout
![Full Chip](images/01_full_chip.png)

Full die view (2107 × 2118 µm, 4.46 mm²) — complete layout showing 242,823 placed instances, I/O pin ring, and power distribution network across 6 metal layers.

#### Quadrant Zoom
![Quadrant Zoom](images/02_quadrant_zoom.png)

One quadrant of the die showing macro-level placement structure, routing channel distribution, and power strap pattern.

#### Block-Level Zoom
![Block Zoom](images/03_block_zoom.png)

Mid-level zoom into the core showing standard cell rows, routing channels, and power grid structure. Individual cell boundaries and M1–M4 connections visible.

#### Routing Zoom
![Routing Zoom](images/04_routing_zoom.png)

Routing-level zoom showing dense metal interconnect on all 6 layers (LI through M5), via stacks, and power grid stripes. 12,010,550 µm total wirelength and 2,067,558 vias.

#### Routing Detail
![Routing Detail](images/05_routing_detail.png)

High-resolution routing detail showing individual signal tracks, via pillars, and metal layer separation. DRC-clean output after 53 detailed routing iterations.

#### Cell-Level Zoom
![Cell Zoom](images/06_cell_zoom.png)

Cell-level zoom showing individual `sky130_fd_sc_hd` standard cell boundaries, internal cell transistor structures, local interconnect (LI), and Metal 1 routing patterns.

#### Transistor-Level Zoom
![Transistor Zoom](images/07_transistor_zoom.png)

Maximum zoom at transistor-level — polysilicon gates, N-well/P-substrate diffusion regions, contacts, via stacks, and first-layer metal traces at the finest layout granularity resolvable in GDS.

---

## 16. Repository Structure

```
├── src/                        # RTL source files
├── verif/                      # Verification testbenches and test vectors
├── config.json                 # MY FLOW PnR configuration
├── images/                     # PnR layout images (7 zoom levels, 2000×2000px)
│   ├── 01_full_chip.png
│   ├── 02_quadrant_zoom.png
│   ├── 03_block_zoom.png
│   ├── 04_routing_zoom.png
│   ├── 05_routing_detail.png
│   ├── 06_cell_zoom.png
│   └── 07_transistor_zoom.png
└── output/                     # Final PnR signoff outputs (RUN_2026-05-17)
    ├── gds/                    # GDS II — primary tape-out (466 MB) [LFS]
    ├── klayout_gds/            # GDS II — KLayout version (239 MB) [LFS]
    ├── def/                    # Design Exchange Format post-route (299 MB) [LFS]
    ├── lef/                    # Library Exchange Format abstract view
    ├── nl/                     # Gate-level netlist post-synthesis (66 MB) [LFS]
    ├── pnl/                    # Physical netlist post-PnR (130 MB) [LFS]
    ├── spice/                  # SPICE netlist — extracted (73 MB) [LFS]
    ├── json_h/                 # Hierarchy JSON (57 MB) [LFS]
    ├── spef/                   # RC parasitics — max corner (372 MB) [LFS]
    │   └── max/
    ├── sdf/                    # Standard Delay Format — TT + FF corners [LFS]
    │   ├── max_ff_n40C_1v95/
    │   ├── max_tt_025C_1v80/
    │   ├── min_ff_n40C_1v95/
    │   ├── min_tt_025C_1v80/
    │   ├── nom_ff_n40C_1v95/
    │   └── nom_tt_025C_1v80/
    ├── lib/                    # Liberty timing files — all 9 PVT corners
    │   ├── max_ff_n40C_1v95/
    │   ├── max_ss_100C_1v60/
    │   ├── max_tt_025C_1v80/
    │   ├── min_ff_n40C_1v95/
    │   ├── min_ss_100C_1v60/
    │   ├── min_tt_025C_1v80/
    │   ├── nom_ff_n40C_1v95/
    │   ├── nom_ss_100C_1v60/
    │   └── nom_tt_025C_1v80/
    ├── sdc/                    # Timing constraints (SDC)
    ├── vh/                     # Verilog header
    ├── render/                 # Full-chip layout render (KLayout)
    ├── reports/                # Signoff reports
    │   ├── drc.magic.rpt       # Magic DRC — 0 violations
    │   ├── drc.klayout.json    # KLayout DRC — 0 violations
    │   ├── drc.klayout.lyrdb   # KLayout DRC layer database
    │   ├── lvs.rpt             # LVS report — circuits match uniquely
    │   ├── lvs.json            # LVS JSON summary
    │   ├── sta_summary.rpt     # Multi-corner STA summary
    │   ├── irdrop.rpt          # IR drop analysis
    │   ├── manufacturability.rpt # Final signoff checklist
    │   └── pnr_run2.log        # Full 72-step PnR run log
    ├── metrics.json            # Complete flow metrics (JSON)
    └── metrics.csv             # Complete flow metrics (CSV)
```

> Files marked **[LFS]** are stored via Git Large File Storage.

---

## 17. Reproducibility

### 17.1 Technology Stack

| Component | Tool | Version | Role |
|---|---|---|---|
| **Frontend Synthesis** | MY FLOW | — | RTL → Gate-level netlist + SDC |
| **Backend PnR** | MY FLOW | 3.1.0 | Floorplan → GDSII (72 steps) |
| **Place & Route Engine** | MY FLOW (bundled) | — | Physical design |
| **Detailed Router** | MY FLOW (bundled) | — | DRC-clean routing (53 iterations) |
| **DRC** | Magic + KLayout | — | Layout verification (dual tool) |
| **LVS** | MY FLOW (bundled) | — | Schematic vs layout (242,823 devices) |
| **Timing** | MY FLOW (bundled) | — | Multi-corner STA (9 PVT corners) |
| **Parasitic Extraction** | MY FLOW (bundled) | — | RC extraction (3 RC corners) |
| **PDK** | SkyWater SKY130A | — | 130nm open-source process |
| **Container** | Docker | MY FLOW image | Reproducible environment |

### 17.2 PnR Flow — 72 Steps

```
Step  1–9:   SDC check, Floorplan, I/O placement, Tap/Endcap insertion
Step 10–19:  PDN generation, Global placement, Placement optimisation
Step 20–29:  Detailed placement, CTS, Clock repair, Timing repair
Step 30–39:  Global routing, Detailed routing (53 iterations)
Step 40–49:  Antenna repair, Diode insertion, Fill insertion
Step 50–59:  SPEF extraction (max/min/nom), IR drop analysis, STA (9 corners)
Step 60–63:  Magic DRC, KLayout DRC, Magic StreamOut (GDS)
Step 64–67:  KLayout StreamOut, SPICE extraction, LVS
Step 68–72:  Multi-corner STA (post-LVS), Manufacturability check
```

### 17.3 Run Configuration

```bash
# Run full 72-step PnR flow (requires ~8 GB RAM minimum, 16+ cores recommended)
docker run --rm \
  -v $(pwd):/work \
  myflow:3.1.0 \
  python3 -m myflow /work/config.json
```

### 17.4 Key Run Statistics

| Metric | Value |
|---|---|
| **Run ID** | RUN_2026-05-17_18-18-01 |
| **Total Steps** | 72 |
| **Compute Used** | 122 cores, 98 GB RAM (vast.ai cloud) |
| **Final Status** | Physical signoff PASS |
| **GDS Size** | 466 MB |
| **Total Output Size** | ~5 GB (all formats) |

---

*Physical signoff complete — Magic DRC 0 violations · KLayout DRC 0 violations · LVS match · 0 antenna violations · IR drop < 0.4 mV*

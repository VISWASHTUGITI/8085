# Standard Cell Placement — OpenROAD / OpenLane Physical Design Flow

A complete, hands-on implementation and analysis of the **Placement** stage of the RTL-to-GDSII flow for a hierarchical sub-block (`block_standalone`) using OpenLane + OpenROAD on the SkyWater 130nm open-source PDK.

## Table of Contents
- [Design Overview](#design-overview)
- [Flow Status](#flow-status)
- [Placement Sub-Flow](#placement-sub-flow)
  - [Step 1 — Pre-Placement Audit](#step-1--pre-placement-audit)
  - [Step 2 — Global Placement](#step-2--global-placement)
  - [Step 3 — Post-Global DRC & Timing](#step-3--post-global-drc--timing)
  - [Step 4 — Legalization](#step-4--legalization)
  - [Step 5 — Timing-Driven Optimization](#step-5--timing-driven-optimization)
  - [Step 6 — Congestion Analysis](#step-6--congestion-analysis)
  - [Step 7 — High Fanout Net Synthesis](#step-7--high-fanout-net-synthesis)
- [Final Signoff](#final-signoff)
- [Key Concepts Learned](#key-concepts-learned)
- [Interview Q&A](#interview-qa)
- [File Structure](#file-structure)
- [Reproducing the Flow](#reproducing-the-flow)
- [Tool Version Notes](#tool-version-notes)

---

## Design Overview

| Parameter | Value |
|---|---|
| Design Name | block_standalone |
| PDK | SkyWater SKY130 (sky130A) |
| Standard Cell Library | sky130_fd_sc_hd |
| Core Area | 60 µm × 60 µm = 3,600 µm² |
| Clock Frequency | 100 MHz (10.0 ns period) |
| Tools | OpenLane v1.0.2, OpenROAD |
| Signal Routing Layers | met2 – met5 |
| Clock Routing Layers | met3 – met5 |
| Power Layers | met1 (rails), met4 (V-stripes), met5 (H-stripes) |

### Netlist Composition

```
Library masters available (LEF)  : 441 cell templates
Total instantiated cells         : 236
  ├── Movable logic cells        : 144
  │     ├── Sequential (DFFs)    : 18
  │     └── Combinational        : 218
  └── Fixed cells (tap/endcap)   : 92
Total signal nets                : 156
Boundary I/O ports               : 30
```

## Flow Status

```
✅ Synthesis                    →  netlist + SDC generated
✅ Floorplan + PDN              →  60×60 µm core, met1/met4/met5 grid
✅ PLACEMENT (this document)    →  fully legalized, DRC clean, routable
⬜ Clock Tree Synthesis (CTS)   →  next
⬜ Detailed Routing + SPEF
⬜ Macro Hardening (LEF/LIB/GDS)
```

## Placement Sub-Flow

Placement is not a single command. It is a sequence of physically distinct engines:

```
      PRE-PLACEMENT AUDIT
   (structural + zero-wireload STA)
                ↓
      GLOBAL PLACEMENT  (RePlAce / Nesterov)
   cells spread out, overlapping, off-grid
                ↓
      DRC + STA CHECK   (wire RC now real)
   max slew / max cap / max fanout / WNS
                ↓
      PASS-1 RESIZER    (repair_design)
   HFNS buffer trees, driver upsizing
                ↓
      LEGALIZATION      (OpenDP)
   snap to rows, 0 overlaps, VDD/VSS flip
                ↓
      PASS-2 RESIZER    (repair_timing -setup)
   critical-path upsize, non-critical downsize
                ↓
      FINAL LEGALIZATION + GLOBAL ROUTE CHECK
   congestion / overflow verification
```

**Why two resizer passes?** Pass-1 fixes electrical DRCs (slew/cap/fanout) and adds cells. Pass-2 fixes timing slack and resizes cells. Legalization must run after each because both change cell count/width and break row legality.

---

## Step 1 — Pre-Placement Audit

**Purpose:** Verify the floorplan is placeable before wasting time on placement. Catch bad constraints, over-utilization, or structural HFNs early.

### Commands
```tcl
read_lef      .../merged.nom.lef
read_liberty  $PDK_ROOT/$PDK/libs.ref/sky130_fd_sc_hd/lib/sky130_fd_sc_hd__tt_025C_1v80.lib
read_db       .../results/floorplan/block.odb

report_design_area
report_check_types -max_slew        -violators
report_check_types -max_capacitance -violators
report_check_types -max_fanout      -violators
```

### Results
```
Design area 1551 u^2   43% utilization
```

| Metric | Value | Interpretation |
|---|---|---|
| Core area | 3,600 µm² | 60 × 60 µm |
| Cell area | 1,551 µm² | Sum of all standard cell areas |
| Utilization | 43 % | Ideal — 57 % whitespace for routing/buffers |
| Max slew | 0.51 ns (limit 1.50) | MET, slack +0.99 ns |
| Max cap | 0.01 pF (limit 0.09) | MET, slack +0.08 pF |

### High Fanout Nets Discovered

| Net | Fanout | Driver | Strategy |
|---|---|---|---|
| VPWR | 424 | Power rail | met1 rails (not signal) |
| VGND | 424 | Ground rail | met1 rails (not signal) |
| clk | 18 | Input port | Deferred to CTS |
| _076_ | 11 | _126_/X | HFNS buffer tree |
| _077_ | 11 | _127_/X | HFNS buffer tree |
| _079_ | 11 | _130_/X | HFNS buffer tree |
| inp_north[1] | 10 | Input port | HFNS buffer tree |
| rst | 9 | Input port | HFNS buffer tree |

**Key insight:** clk fanout = 18 exactly matches the 18 flip-flops — perfect 1:1 confirmation of the sequential element count.

### Why Were There No Slew/Cap Violations Here?

```
C_total = C_wire + Σ C_pin
```

At pre-placement, cells are unplaced → wire length = 0 → C_wire = 0 fF. Each input pin contributes only ~2–4 fF. So for an 11-pin net:

```
C_total = 11 × 3 fF + 0 fF = 33 fF  ≪  90 fF limit   → MET
```

Slew follows `Slew ≈ 2.2 × R_driver × C_total` — with tiny C_total, slew is fast. Violations only appear once real wires exist.

---

## Step 2 — Global Placement

**Purpose:** Distribute all 144 movable cells across the core to minimize HPWL (Half-Perimeter Wirelength) while keeping bin density below a target.

### The Optimization Problem
```
minimize   HPWL(x, y)
subject to Density(bin) ≤ target_density   for all bins
```

RePlAce models this as an electrostatic system:
- Nets = springs → pull connected cells together (reduce wirelength)
- Cells = like charges → repel each other (eliminate overlap)
- Nesterov's accelerated gradient descent finds the equilibrium.

### Command
```tcl
global_placement -density 0.70 -pad_left 2 -pad_right 2
```

### The Density Error We Hit First
```
[ERROR GPL-0302] Use a higher -density or re-floorplan with a larger core area.
Given target density: 0.60
Suggested target density: 0.65
```

Why? Utilization jumped from 43 % → 64.90 %:

| Contributor | Δ Utilization |
|---|---|
| Raw synthesized logic | 43.0 % |
| Fixed tap/endcap cells | + 6.2 % |
| Virtual cell padding (-pad 2/2) | + 15.7 % |
| **Effective total** | **64.90 %** |

Padding adds 2 × 0.46 µm = 0.92 µm of virtual whitespace on each side of every cell — this reserves room for pin access and reduces routing congestion, but inflates apparent area.

Since 64.90 % > 60 %, the constraint was mathematically unsatisfiable → raised density to 0.70.

### Convergence Log
```
[INFO GPL-0019] Util(%): 64.90
[INFO GPL-0028] BinCnt: 8 8            (64 bins, 7.59 × 7.48 µm each)
[INFO GPL-0023] TargetDensity: 0.70

--- Phase 1: Conjugate Gradient (initial rough placement) ---
[InitialPlace] Iter: 1   HPWL: 3,181,480
[InitialPlace] Iter: 5   HPWL: 2,767,142      ← 13 % HPWL reduction

--- Phase 2: Nesterov (electrostatic spreading) ---
[NesterovSolve] Iter:   1  overflow: 0.721  HPWL: 1,948,351
[NesterovSolve] Iter: 100  overflow: 0.597  HPWL: 2,093,654
[NesterovSolve] Iter: 200  overflow: 0.465  HPWL: 2,271,898
[NesterovSolve] Iter: 300  overflow: 0.116  HPWL: 2,491,867
[NesterovSolve] Finished with Overflow: 0.097933   ← < 10 % = CONVERGED ✅
```

### Understanding the HPWL Increase
```
Iter 1   : overflow 72 %, HPWL 1.95 M   ← cells stacked → artificially short wires
Iter 310 : overflow  9.8 %, HPWL 2.50 M ← cells spread  → realistic wire lengths
```

HPWL increases during Nesterov because eliminating overlap forces cells apart. This is the fundamental placement tradeoff:

```
minimize wirelength  →  clump cells  →  overlap
minimize overlap     →  spread cells →  longer wires
```

The solver finds the optimal balance where overflow < 10 %.

---

## Step 3 — Post-Global DRC & Timing

**Purpose:** Now that cells occupy real coordinates, model wire RC parasitics and re-check all electrical constraints.

### Commands
```tcl
set_wire_rc -layer met2
read_sdc    .../tmp/floorplan/3-initial_fp.sdc
estimate_parasitics -placement

report_check_types -max_slew
report_check_types -max_capacitance
report_check_types -max_fanout
report_checks -path_delay max -format full_clock_expanded
report_worst_slack -max
```

**Common pitfall:** Running `estimate_parasitics` without `set_wire_rc` gives
`[WARNING RSZ-0014] wire capacitance for corner default is zero`
→ all wire caps become 0 and DRC/timing reports are meaningless.

### DRC Results

| Check | Pin | Limit | Actual | Slack | Status |
|---|---|---|---|---|---|
| Max Slew | _251_/A | 0.75 ns | 0.61 ns | +0.14 ns | MET |
| Max Cap | _126_/X | 0.13 pF | 0.05 pF | +0.08 pF | MET |
| Max Fanout | _126_/X | 10 | 10 | 0 | MET (at limit) |

### Pre-Place vs Post-Global Comparison

| Metric | Pre-Placement | Post-Global | Δ | Physical Cause |
|---|---|---|---|---|
| Max slew | 0.51 ns | 0.61 ns | +0.10 ns | Wire RC delay added |
| Max cap | 0.01 pF | 0.05 pF | 5× | Wire parasitic capacitance |
| Slack margin | +0.99 ns | +0.14 ns | −0.85 ns | The "wire penalty" |

### Worst Setup Path
```
Startpoint : inp_west[0]  (input port, clocked by clk)
Endpoint   : _269_        (rising-edge DFF, clocked by clk)
Logic depth: 12 gates

 Delay   Time   Description
------------------------------------------------------------------
  0.00   0.00   clock clk (rise edge)
  2.00   2.00 v input external delay
  0.01   2.01 v inp_west[0] (in)
  0.22   2.24 v _127_/X  sky130_fd_sc_hd__buf_1
  0.26   2.49 v _173_/X  sky130_fd_sc_hd__a21o_2
  0.27   2.77 v _174_/X  sky130_fd_sc_hd__and4b_2
  0.29   3.05 v _178_/X  sky130_fd_sc_hd__o211a_2
  0.45   3.50 v _179_/X  sky130_fd_sc_hd__or3_2     ← slow cell
  0.24   3.74 v _181_/X  sky130_fd_sc_hd__and3_2
  0.49   4.23 v _183_/X  sky130_fd_sc_hd__or3_2     ← slowest cell
  0.27   4.49 v _186_/X  sky130_fd_sc_hd__and3_2
  0.36   4.85 v _221_/X  sky130_fd_sc_hd__a211o_2
  0.30   5.16 v _246_/X  sky130_fd_sc_hd__a41o_2
  0.23   5.38 v _249_/X  sky130_fd_sc_hd__and3_2
  0.14   5.52 ^ _251_/Y  sky130_fd_sc_hd__nor3_2
  0.00   5.52 ^ _269_/D  sky130_fd_sc_hd__dfxtp_2
------------------------------------------------------------------
         5.52   DATA ARRIVAL TIME

 10.00  10.00   clock clk (rise edge)
  0.00  10.00   clock network delay (IDEAL — no CTS yet)
 -0.25   9.75   clock uncertainty
 -0.06   9.69   library setup time
------------------------------------------------------------------
         9.69   DATA REQUIRED TIME
        -5.52   DATA ARRIVAL TIME
------------------------------------------------------------------
         4.17   SLACK (MET) ✅
```

> **Note:** Clock network delay is 0.00 ns because CTS has not run. After CTS, real buffer delays and skew will be inserted and this slack will change.

---

## Step 4 — Legalization

**Purpose:** After global placement, cells overlap and sit off-grid. Legalization (OpenDP) snaps every cell to a legal site.

### What Legalization Does
1. **Site snapping** — align cell origins to the 0.46 µm site grid
2. **Overlap removal** — slide cells along rows until zero overlap
3. **Orientation flipping** — flip cells (R0 ↔ MX) so VDD pins meet VDD rails and VSS pins meet VSS rails
4. **Padding preservation** — maintain whitespace for pin access

```
BEFORE (illegal)                      AFTER (legal)
┌──────────────────────┐              ┌──────────────────────┐
│ [A]░░[B]             │              │ ══════ Row 3 ══════  │
│   ░░[C]░░            │    ──────►   │ [A] [B] [C]          │
│ [D]░░ [E]            │              │ ══════ Row 2 ══════  │
│ overlapping/off-grid │              │ [D]     [E]          │
└──────────────────────┘              └──────────────────────┘
```

### Command
```tcl
detailed_placement
check_placement -verbose
```

> **Syntax note:** `detailed_placement` takes no `-pad_left`/`-pad_right` flags. Padding is set during `global_placement` or via `set_placement_padding`.

### Results
```
Placement Analysis
---------------------------------
total displacement        390.9 u
average displacement        1.7 u
max displacement            9.0 u
original HPWL            2514.9 u
legalized HPWL           2981.1 u
delta HPWL                   19 %
```

| Metric | Value | Meaning |
|---|---|---|
| Cells before | 236 | — |
| Cells after | 236 | Legalization adds ZERO cells |
| Avg displacement | 1.7 µm | ≈ 3.7 site widths per cell |
| Max displacement | 9.0 µm | Worst cell had to slide far to find a slot |
| ΔHPWL | +19 % | Wires stretch as overlaps are removed |
| check_placement | 0 errors | 100 % legal ✅ |

**Critical distinction:**
- Resizer / CTS / Filler insertion → add or resize cells
- Legalization → only moves and flips existing cells

---

## Step 5 — Timing-Driven Optimization

**Purpose:** With exact legal positions known, close setup timing and recover power/area.

### Theory → Command Mapping

| Theory Concept | OpenROAD Command |
|---|---|
| Parasitic estimation | `estimate_parasitics -placement` |
| Critical-path tracing | `repair_timing -setup` (internal) |
| Gate upsizing | `repair_timing -setup` (internal) |
| Non-critical downsizing | `repair_timing -setup` (internal) |
| Re-legalize resized cells | `detailed_placement` |
| Legality signoff | `check_placement -verbose` |
| Slack signoff | `report_worst_slack -max` |

### What Upsizing / Downsizing Does Physically

**Upsizing (fix critical path):**
```
BEFORE:  ...──► [ or3_1  ] (weak, W=small)  →  delay 0.45 ns
AFTER :  ...──► [ or3_2  ] (strong, W=2×)   →  delay 0.22 ns   (2× faster)
```
Larger transistor width → higher drive current I_on → faster charge/discharge of load cap.

**Downsizing (recover power on non-critical path):**
```
BEFORE:  [ and2_4 ] (1.84 µm wide)  slack +4.17 ns  ← wasteful
AFTER :  [ and2_1 ] (0.92 µm wide)  slack +2.10 ns  ← still MET, 50 % less area/leakage
```

### Why Re-Legalize After Resizing?
```
Width of or3_1 = 1.38 µm  →  Width of or3_2 = 1.84 µm
```
The upsized cell physically expands and collides with its row neighbor → illegal overlap → must re-run `detailed_placement`.

### Command
```tcl
estimate_parasitics -placement
repair_timing -setup
detailed_placement
check_placement -verbose
write_db  .../results/placement/block_placed.odb
write_def .../results/placement/block_placed.def
```

---

## Step 6 — Congestion Analysis

**Purpose:** Run global routing on the placed design to verify wires can physically fit — before committing to CTS and detailed routing.

### GCell Edge Capacity Theory

The core is diced into a grid of GCells. Congestion is measured at the **boundary (edge)** between adjacent GCells, not inside them.

```
    GCell (1,1)              GCell (1,2)
  ┌───────────────┬ ◄─ EDGE ─► ┬───────────────┐
  │  [Driver] ────┼────────────┼───► [Sink]    │
  │               │  wire must │               │
  │               │ cross here │               │
  └───────────────┴────────────┴───────────────┘
```

**Room & doorway analogy:** Two large rooms connected by one narrow doorway. Room size doesn't matter — the doorway width is the bottleneck.

**Capacity math:**
```
Edge Capacity = Edge Length / Track Pitch × (1 − derate)

Example (met2, 7.5 µm edge, 0.46 µm pitch, 20 % derate):
  = 7.5 / 0.46 × 0.8
  = 16 × 0.8
  ≈ 12 usable tracks
```

**Overflow:**
```
Overflow = max(0, Demand − Capacity)

Overflow = 0  →  routable ✅
Overflow > 0  →  hotspot, detailed routing will DRC ❌
```

### Commands
```tcl
set_routing_layers -signal met2-met5 -clock met3-met5

set_global_routing_layer_adjustment met1 1.0   # 100 % blocked (cell rails)
set_global_routing_layer_adjustment met2 0.2   # 20 % safety derate
set_global_routing_layer_adjustment met3 0.2
set_global_routing_layer_adjustment met4 0.2
set_global_routing_layer_adjustment met5 0.2

global_route -congestion_report_file /tmp/congestion.rpt -verbose
```

### Why Clock Uses met3–met5 but Signals Use met2–met5

| Aspect | Signal Nets | Clock Nets |
|---|---|---|
| Count | Hundreds | 1–2 |
| Switching rate | Occasional | Every cycle (100 M/s) |
| Sensitivity | Tolerates wire delay | Extremely skew-sensitive |
| Allowed layers | met2 – met5 | met3 – met5 only |

Clock delay ∝ R_wire × C_wire. Upper layers (met3–met5) are thicker (lower R) and further from substrate (lower C) → faster, lower-skew clock distribution.

### Layer Track Pitches (SKY130)
```
[INFO GRT-0088] Layer li1    Track-Pitch = 0.4600 µm
[INFO GRT-0088] Layer met1   Track-Pitch = 0.3400 µm
[INFO GRT-0088] Layer met2   Track-Pitch = 0.4600 µm
[INFO GRT-0088] Layer met3   Track-Pitch = 0.6800 µm
[INFO GRT-0088] Layer met4   Track-Pitch = 0.9200 µm
[INFO GRT-0088] Layer met5   Track-Pitch = 3.4000 µm
```

Lower layers = fine pitch, thin metal, local interconnect.
Upper layers = wide pitch, thick metal, power + global clock.

### Routing Resource Analysis
```
          Routing      Original      Derated      Resource
Layer     Direction    Resources     Resources    Reduction (%)
---------------------------------------------------------------
li1        Vertical            0             0          0.00%
met1       Horizontal          0             0          0.00%
met2       Vertical         1800          1395          22.50%
met3       Horizontal       1200           826          31.17%
met4       Vertical          720           389          45.97%   ← PDN met4 stripes
met5       Horizontal        240            99          58.75%   ← PDN met5 stripes
---------------------------------------------------------------
```

This directly proves the PDN impact. The vertical met4 power stripes eat 46 % of met4 tracks; the horizontal met5 stripes eat 59 % of met5 tracks. FastRoute correctly subtracts them before routing signals.

### Final Congestion Report
```
Layer         Resource        Demand        Usage (%)    Max H / Max V / Total Overflow
---------------------------------------------------------------------------------------
li1                  0             0            0.00%             0 /  0 /  0
met1                 0             0            0.00%             0 /  0 /  0
met2              1395           202           14.48%             0 /  0 /  0
met3               826           246           29.78%             0 /  0 /  0
met4               389             6            1.54%             0 /  0 /  0
met5                99             6            6.06%             0 /  0 /  0
---------------------------------------------------------------------------------------
Total             2709           460           16.98%             0 /  0 /  0

[INFO GRT-0018] Total wirelength: 6451 um
[INFO GRT-0111] Final number of vias: 1178
[INFO GRT-0014] Routed nets: 154
```

### Verdict

| Metric | Value | Status |
|---|---|---|
| Total Overflow | 0 / 0 / 0 | 100 % ROUTABLE ✅ |
| Overall usage | 16.98 % | Very comfortable |
| Total wirelength | 6,451 µm (6.45 mm) | — |
| Total vias | 1,178 (1,123 pin + 15 Steiner) | — |

### Reading the GUI Heatmap

The GUI heatmap showed red regions in the center/top — but overflow was 0. Why?

The heatmap normalizes color to the **highest local usage** on the chip, not to 100 % capacity. A tile at 30 % usage appears red relative to 0 % blue tiles. Red ≠ overflow. Always confirm with the numeric report.

| Color | Relative usage | Meaning |
|---|---|---|
| Blue / Teal | 0–20 % | Wide-open routing space |
| Green | 30–60 % | Healthy |
| Yellow | 70–90 % | Tight but legal |
| Red | > 100 % demand | True hotspot (only if overflow > 0) |

### Causes of Local Density in Our Design
1. **PDN blockages** — met4/met5 straps physically remove tracks
2. **Pin density clustering** — multi-input cells (a41o, o211a) packed together
3. **Pass-through nets** — long nets crossing the middle of the die

---

## Step 7 — High Fanout Net Synthesis

### What Is an HFN?

A non-clock net driving a large number of input pins:
- Reset (`rst`) → all async reset pins
- Scan enable → all scan flops
- Control / mux-select → wide datapaths

The clock net is also high-fanout but is excluded — it goes to CTS with dedicated low-skew clkbuf cells.

### The Electrical Problem
```
                       ┌──► pin 1  (3 fF)
                       ├──► pin 2  (3 fF)
Driver ────────────────┼──► ...
(weak, R_drv high)     ├──► pin 10 (3 fF)
                       └──► pin 11 (3 fF)

C_total  = C_wire + Σ C_pin
Slew     ≈ 2.2 × R_driver × C_total
```
High C_total on a weak driver → slow slew ramp → high short-circuit current, high dynamic power, setup timing failure.

### The Solution: Buffer Tree
```
BEFORE (fanout 11)                    AFTER HFNS
                                      
        ┌──► s1                       Driver ──┬──► s1
        ├──► s2                       (fanout   ├──► s2
        ├──► s3                        = 4)     ├──► s3
Driver ─┼──► s4                                 │
        ├──► s5                                 ├──► [BUF1] ──┬──► s4
        ├──► ...                                │             ├──► s5
        └──► s11                                │             └──► s6
                                                │
                                                └──► [BUF2] ──┬──► s7
                                                              ├──► ...
                                                              └──► s11
```
Driver now sees only 2 buffer inputs + 3 pins instead of 11 pins. Each buffer easily drives its small local group → sharp slew restored.

### Attempt 1 — Default `repair_design`
```tcl
repair_design -max_wire_length 300
```

Result:
```
[INFO RSZ-0039] Resized 130 instances.
Total Instances Before : 236
Total Instances After  : 236
New Buffers Inserted   : 0
```

**Zero buffers added — why?**
- The engine chose driver upsizing over buffer insertion.
- It swapped weak drivers in-place for stronger variants (nand2_1 → nand2_2/nand2_4).
- Instance count stays 236 because upsizing is a master swap, not an addition.
- In a tiny 60×60 µm block, wires are < 50 µm — upsizing is more area-efficient than adding repeaters.
- Default `max_wire_length` is 3176 µm (sized for mm-scale SoCs) — no wire in our block comes close.

Legalization impact was negligible:
```
average displacement  0.0 u
max displacement      0.9 u
delta HPWL            0 %
```

### Attempt 2 — Forcing Buffer Trees
```tcl
set_max_fanout 8 [current_design]
repair_design
detailed_placement
```

Result:
```
[INFO RSZ-0035] Found 8 fanout violations.
[INFO RSZ-0038] Inserted 8 buffers in 8 nets.
[INFO RSZ-0039] Resized 138 instances.

Placement Analysis
---------------------------------
total displacement         22.9 u
average displacement        0.1 u
max displacement            4.6 u
original HPWL            3304.7 u
legalized HPWL           3308.2 u
delta HPWL                    0 %

Total Instances Before HFNS : 236
Total Instances After HFNS  : 244
New HFNS Buffers Inserted   : 8
```

### The 8 Nets Repaired

| # | Net | Fanout Before | Driver | After |
|---|---|---|---|---|
| 1 | _076_ | 11 | _126_/X | ≤ 8 |
| 2 | _077_ | 11 | _127_/X | ≤ 8 |
| 3 | _079_ | 11 | _130_/X | ≤ 8 |
| 4 | inp_north[1] | 10 | Port | ≤ 8 |
| 5 | inp_north[3] | 10 | Port | ≤ 8 |
| 6 | inp_north[2] | 9 | Port | ≤ 8 |
| 7 | inp_west[2] | 9 | Port | ≤ 8 |
| 8 | rst | 9 | Port | ≤ 8 |

### Post-HFNS Fanout Report
```
max fanout

Pin                                   Limit Fanout  Slack
---------------------------------------------------------
fanout1/X                                 8      8      0 (MET)
```
Worst fanout on the entire chip is now ≤ 8 pins. Zero violations. ✅

### Manual ECO Buffer Insertion (GUI)

To understand net splitting visually, net `_076_` was highlighted in the OpenROAD GUI:
```
Find Object → Type: Instance → Find: _126_ → ☑ Add Set To Highlight
```
This revealed 10 green flightlines radiating from `_126_` across the entire die (some spanning 50 µm) — the physical reality of high fanout: one small gate charging 10 distant pin capacitances plus all the wire capacitance in between.

Useful GUI/TCL commands discovered:
```tcl
# Inspect what net a pin drives
set eco_net [get_nets -of_objects [get_pins _126_/X]]
puts [get_full_name $eco_net]        # → _076_

# List all resizer commands available in your build
info commands rsz::*

# Physical DB object lookup (not STA handle)
set db_net [[ord::get_db_block] findNet "_076_"]
```

> **Gotcha:** OpenROAD has two object systems — OpenSTA (`get_nets`) returns timing handles, OpenDB (`findNet`) returns physical handles. GUI commands need OpenDB handles. Passing an STA handle gives `[ERROR GUI-0035] Unable to find descriptor.`

---

## Final Signoff

| Metric | Target / Limit | Achieved | Status |
|---|---|---|---|
| Core area | 3,600 µm² | 3,600 µm² | ✅ |
| Cell area | — | 1,551 µm² | ✅ |
| Utilization | 40–60 % | 43 % | ✅ |
| Effective util (with padding) | ≤ 70 % | 64.90 % | ✅ |
| Placed instances | — | 244 (236 + 8 HFNS buf) | ✅ |
| Global placement overflow | < 10 % | 9.79 % | ✅ |
| Placement legality | 0 overlaps | 0 errors | ✅ |
| Avg cell displacement | — | 0.1 µm | ✅ |
| Max cell displacement | < 15 µm | 4.6 µm | ✅ |
| Max slew | ≤ 0.75 ns | 0.61 ns | ✅ |
| Max capacitance | ≤ 0.13 pF | 0.05 pF | ✅ |
| Max fanout | ≤ 8 | 8 (slack 0) | ✅ |
| Worst setup slack (WNS) | ≥ 0 ns | +4.17 ns | ✅ |
| Routing overflow | 0 | 0 / 0 / 0 | ✅ |
| Total wirelength | — | 6,451 µm | — |
| Total vias | — | 1,178 | — |

Placement phase is fully signed off. Database `block_hfns_buffered.odb` is locked and ready for CTS.

---

## Key Concepts Learned

**1. Utilization ≠ Effective Utilization**
Raw cell area (43 %) is not what the placer sees. Fixed taps + virtual padding pushed it to 64.9 %. Always set target density above the effective number.

**2. HPWL Increases Are Normal**
Both Nesterov spreading and legalization increase wirelength. That's the price of eliminating overlap. Judge quality by overflow and slack, not HPWL alone.

**3. Wire Parasitics Are the Whole Story**
Pre-placement: C_wire = 0 → everything MET, meaningless.
Post-placement: C_wire real → slew ×1.2, cap ×5. Only now are DRCs trustworthy.

**4. Two Resizer Passes, Two Purposes**
- Pass 1 (`repair_design`, before legalization) → fix DRC (slew/cap/fanout), adds cells
- Pass 2 (`repair_timing`, after legalization) → fix timing, resizes cells
- Legalize after both, because both break row legality

**5. Upsizing vs Buffering**
The resizer prefers in-place driver upsizing when wires are short (small blocks). It only inserts buffer trees when fanout/wire-length constraints are explicitly tightened or the block is large.

**6. Congestion Lives at GCell Edges**
Not inside tiles. Capacity = edge_length / track_pitch × (1 − derate). Red heatmap tiles ≠ overflow — always read the numeric report.

**7. PDN Steals Routing Resources**
met4 lost 46 % and met5 lost 59 % of tracks to power stripes. Every power strap you add is routing capacity you give up.

**8. Clock Is Never an HFNS Target**
Logic buffers create uncontrolled skew. `clk` (18 fanout) is deliberately deferred to CTS with dedicated clkbuf cells.

---

## Interview Q&A

<details>
<summary><b>Q: Walk me through your placement flow.</b></summary>

"I ran placement as seven distinct stages rather than a single command.

First a pre-placement audit — I checked utilization (43 % of a 3,600 µm² core), counted instances (236 cells, 18 flops, 218 combinational), and audited high-fanout nets structurally. Timing at this stage is zero-wireload so all DRCs pass trivially.

Then global placement with RePlAce. My first attempt at `-density 0.60` failed with GPL-0302 because effective utilization was 64.9 % — raw logic 43 % plus 6 % fixed taps plus 16 % from `-pad_left 2 -pad_right 2` padding. I raised density to 0.70 and it converged in 310 Nesterov iterations to 9.79 % overflow.

Next post-global DRC and STA. I set `set_wire_rc -layer met2` and ran `estimate_parasitics -placement`. This is where real violations appear — max slew went from 0.51 to 0.61 ns and max cap went 5× from 0.01 to 0.05 pF purely from wire parasitics. Worst setup slack was +4.17 ns on a 12-gate path from `inp_west[0]` to a flop.

Then HFNS as Pass-1 DRC repair, legalization with OpenDP, Pass-2 timing repair, and finally global routing to verify congestion."
</details>

<details>
<summary><b>Q: How did you handle high fanout nets?</b></summary>

"I audited all 156 nets and found eight with fanout above 8 — three internal control nets at 11 pins (`_076_`, `_077_`, `_079_`), plus input ports `inp_north[1..3]`, `inp_west[2]`, and `rst` at 9–10 pins.

I explicitly excluded `clk` (fanout 18, matching the 18 flops) because logic buffers introduce uncontrolled skew — that goes to CTS.

My first `repair_design` run inserted zero buffers. Investigating, I found it had upsized 130 drivers in-place instead. That's correct behavior: in a 60×60 µm block wires are under 50 µm, and the default `max_wire_length` is 3176 µm, so no wire triggered buffering. In-place upsizing is more area-efficient than adding repeaters at that scale.

To demonstrate genuine buffer-tree synthesis, I tightened the constraint with `set_max_fanout 8 [current_design]`. That found 8 violations and inserted 8 `buf_2` repeaters, growing the design from 236 to 244 cells. Post-repair, worst fanout was ≤ 8 with zero violations. I re-legalized immediately — displacement was only 0.1 µm average, 4.6 µm max, with 0 % HPWL delta."
</details>

<details>
<summary><b>Q: Why run global routing before CTS?</b></summary>

"To validate that the placement is physically routable before investing in clock tree construction. If congestion is unfixable, you'd have to redo placement anyway — and redoing placement after CTS means throwing away the entire clock tree.

I set signals to met2–met5 and clock to met3–met5, applied a 20 % derate on met2–met5 for via and pin-access headroom, and fully blocked met1 since it carries cell rails.

The result was zero overflow — max horizontal 0, max vertical 0, total 0 — at only 16.98 % overall utilization. Notably, the resource analysis showed met4 lost 46 % and met5 lost 59 % of their tracks to the PDN power stripes, which the router correctly accounted for."
</details>

<details>
<summary><b>Q: Your heatmap showed red regions but overflow was zero. Explain.</b></summary>

"The GUI heatmap normalizes color to the highest local usage on the die, not to absolute 100 % capacity. My met3 layer peaked around 30 % usage; relative to the 0 % blue periphery, those tiles render red.

Red on the heatmap means 'relatively dense', not 'overflowing'. The authoritative check is the numeric congestion report, which showed 0/0/0 overflow on every layer. I always cross-check the heatmap against `global_route -congestion_report_file`."
</details>

<details>
<summary><b>Q: Why does legalization increase wirelength?</b></summary>

"Global placement is an analytical solver — it places connected cells at their mathematically ideal positions, which often means overlapping. Legalization forces them apart onto discrete legal sites, so wires stretch.

In my run, HPWL went from 2,514.9 µm to 2,981.1 µm — a 19 % increase — with 1.7 µm average and 9.0 µm max displacement. Importantly, legalization added zero cells; it only moves and flips existing ones. Cell count stayed at 236."
</details>

<details>
<summary><b>Q: Why two resizer passes?</b></summary>

"They solve different problems and must be ordered correctly relative to legalization.

Pass-1 (`repair_design`) fixes electrical DRCs — max slew, max cap, max fanout. It adds buffer cells. Running it before legalization means the new buffers get legalized along with everything else in one pass.

Pass-2 (`repair_timing -setup`) fixes setup slack. It upsizes gates on critical paths and downsizes over-sized gates on paths with excess slack to recover power and area. It runs after legalization because you need exact wire lengths for accurate STA.

Both require a `detailed_placement` afterward — Pass-1 because it adds cells, Pass-2 because upsizing widens cells (e.g. `or3_1` at 1.38 µm becomes `or3_2` at 1.84 µm) and collides with row neighbors."
</details>

<details>
<summary><b>Q: Why is congestion measured at GCell edges?</b></summary>

"Because wires must physically cross the boundary between tiles. Think of two large rooms joined by a narrow doorway — room area is irrelevant; the doorway width is the bottleneck.

Capacity at an edge is edge_length / track_pitch × (1 − derate). For met2 with a 7.5 µm edge and 0.46 µm pitch at 20 % derate, that's about 12 usable tracks. Overflow is max(0, demand − capacity). Any positive overflow means detailed routing will produce shorts or spacing violations."
</details>

---

## File Structure

```
designs/block_standalone/
├── config.json
├── src/
│   └── block.v
└── runs/
    ├── block_run/
    │   ├── results/
    │   │   ├── synthesis/  block.v, block.sdc
    │   │   └── floorplan/  block.def, block.odb
    │   └── tmp/
    │       ├── merged.nom.lef
    │       ├── synthesis/  synthesis.sdc
    │       └── floorplan/  3-initial_fp.sdc      ← golden SDC used
    │
    └── block_run_pnr/
        ├── tmp/
        │   └── merged.nom.lef
        └── results/
            ├── placement/
            │   ├── block_global_placed.odb/.def   ← Step 2
            │   ├── block_legalized.odb/.def       ← Step 4
            │   ├── block_placed.odb/.def          ← Step 5
            │   └── block_hfns_buffered.odb/.def   ← Step 7 (FINAL)
            └── routing/
                └── block_grouted.odb              ← Step 6
```

## Reproducing the Flow

```bash
# Enter container
cd ~/OpenLane && make mount
```

```tcl
# Each stage is an independent OpenROAD script
openroad -no_init /tmp/run_preplace_clean.tcl        # Step 1
openroad -no_init /tmp/run_step2_global_placement.tcl # Step 2
openroad -no_init /tmp/run_step3_fixed.tcl            # Step 3
openroad -no_init /tmp/run_step4_legalization.tcl     # Step 4
openroad -no_init /tmp/run_step5_resizer_timing.tcl   # Step 5
openroad -no_init /tmp/check_congestion.tcl           # Step 6
openroad -no_init /tmp/run_hfns_buffers.tcl           # Step 7

# Visual inspection
openroad -gui
```

```tcl
# In GUI console
read_lef .../merged.nom.lef
read_db  .../results/placement/block_hfns_buffered.odb
# Then: View → Heat Maps → Routing Congestion
```

## Tool Version Notes

Commands vary across OpenROAD builds. Verified working on `41a51eaf`:

| Works | Does NOT work in this build |
|---|---|
| `set_wire_rc -layer met2` | `set_wire_rc -signal met2 -clock met3` |
| `detailed_placement` | `detailed_placement -pad_left 2` |
| `repair_design -max_wire_length N` | `repair_design -max_fanout N` |
| `set_max_fanout 8 [current_design]` | `report_congestion` |
| `report_checks -path_delay max` | `report_checks -max_paths 5` |
| `global_route -congestion_report_file F` | `insert_buffer`, `repair_net` |
| `$inst getLocation` | `$inst getX` / `$inst getY` |
| `[ord::get_db_block] findNet "n"` | `ord::clear` then re-`read_db` (segfaults) |

Always run `info commands <pkg>::*` to discover what your build supports.

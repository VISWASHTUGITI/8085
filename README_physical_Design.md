# Physical Design (PD) Flow — Steps, Internal Steps, and Input/Output Files

*Extracted and organized from PD_Q.pdf (handwritten VLSI physical design notes).*

## 1. Overall PD Flow (High Level)

```
Synthesis → Physical Design (Floorplan → Powerplan → Placement → CTS →
Post-CTS → Routing → Post-Route) → Signoff (STA, Physical Verification)
```

### Overall Inputs to Physical Design (given by front-end/synthesis teams)
| # | Input | Purpose |
|---|-------|---------|
| 1 | Verilog **Netlist** | Gate-level connectivity from synthesis |
| 2 | **SDC** (constraint file) | Timing/design constraints (Tcl-based, SDC format) |
| 3 | **Technology LEF** | Layer, site, via, and rule definitions of the process |
| 4 | **Physical/Macro LEF** | Abstract physical view of macros/std cells |
| 5 | **Floorplan/Physical DEF** | Macro locations, physical placement info |
| 6 | **Timing Library (.lib)** | Cell timing, power, function (PVT-based) |
| 7 | **Scan DEF** | Scan-chain info from DFT team |
| 8 | **MMMC file** | Multi-mode multi-corner setup (PVT corners) |
| 9 | **UPF/CPF** | Power intent — voltage domains, retention, level shifters |
| 10 | **Don't-touch file** | Cells/nets to preserve as-is |
| 11 | **Don't-use file** | Cells forbidden for use in PD |
| 12 | **Port DEF** (optional) | Port location/spacing |
| 13 | **Floorplan DEF** (optional) | Pre-defined floorplan from chip-top |

### Overall Output of PD (fed to Signoff / fabrication)
- Final routed **DEF** and **GDSII**
- **SPEF** (parasitics)
- STA reports, ECO files
- Verified netlist (post physical verification)

---

## 2. Synthesis (Pre-PD Stage — for context)

**Three main steps:** Translation → Optimization → Technology Mapping

| Step | What happens |
|------|---------------|
| a. Translation | Tool (Genus/DC) reads Verilog, builds an un-optimized internal (generic) representation |
| b. Optimization | Removes redundant logic |
| c. Technology Mapping | Maps generic gates to cells in the target technology library |

**Detailed internal steps:**
1. **Inputs** – RTL + `.lib` (lib used at technology-mapping stage)
2. **Read RTL** – tool checks for syntax errors
3. **Elaboration** – builds one unified netlist from multiple RTL files; builds data structures, infers registers, performs HDL optimizations (e.g., dead-code removal, if-else → case/mux inference)
4. **Apply Constraints** – timing constraints + design-rule constraints (e.g. `create_clock`)
5. **Generic Synthesis** – maps to a technology-independent generic library (unoptimized gate netlist)
6. **Logic Optimization** – removes redundant logic (constant flops, unused cells/wires)
7. **Technology Mapping** – maps generic cells to the actual target-technology library cells
8. **Scan Chain Insertion** – done by synthesis team (one-pass flow) or DFT team (two-pass flow)
9. **Incremental Optimization** – one more optimization pass after scan insertion

**Outputs of Synthesis (→ inputs to PD):**
- Synthesized (gate-level) netlist
- Output SDC
- Scan DEF

---

## 3. Floorplanning

**Purpose:** Decide chip/core dimensions, arrangement of macros/IPs, and I/O placement.

### Internal Steps
1. Decide core width/height (die-size estimation)
2. Create I/O pad sites (I/O pad placement)
3. Macro placement
4. Create standard-cell rows (for std-cell placement)
5. Add physical-only cells (endcaps, well-taps, decaps, tie cells — placed at pre-placement/floorplan stage)

### Two Approaches
- **Top-down:** chip-top engineer partitions floorplan into blocks (square/rectangle/rectilinear); block owner completes design, generates DEF for integration.
- **Bottom-up:** block-level engineer decides **utilization ratio** and **aspect ratio** independently.

### Macro Placement Guidelines (priority order)
1. Place macros per connectivity/timing analysis; preference: macro–port > macro–macro > macro–standard cell
2. Group macros per hierarchical naming convention (related macros placed near each other)
3. Place macros near the core boundary (not center) — avoids detouring/timing failure and IR-drop-related voltage issues for surrounding standard cells

### Key Commands (Innovus-style)
- `read_def` / `read_sdc` – read netlist/def/constraints
- `create_rows` – create standard cell rows
- `editPin` – place/edit port pins
- `check_pin_assignment`, `check_place`, `check_welltap` – sanity checks

### Sanity Checks
- Minimum width violation
- Port-affinity buffer presence
- Ports aligned to tracks
- Unplaced ports / multiple ports on same track (short risk)
- Macro-to-macro overlap
- Well-tap minimum distance

### Inputs / Outputs
| Inputs | Outputs |
|--------|---------|
| Netlist, SDC, Tech LEF, macro LEF, Physical/Floorplan DEF, lib | Floorplan LEF & DEF |

Floorplan LEF/DEF is used for: physical-aware synthesis, Caliber base-DRC checks, and given to the IR team for IR/EM checks.

---

## 4. Powerplanning

**Purpose:** Build the power delivery network (PDN) to distribute VDD/VSS with minimal IR drop.

### Internal Steps
1. Create power **rings** around the core/blocks
2. Create power **stripes/straps** across upper metal layers (increase capacitance, reduce IR drop)
3. Create power **rails/staples** at standard-cell rows (lower metal layers)
4. Connect stripes with **vias** (single-cut or multi-cut — multi-cut lowers resistance via parallel paths)
5. Run sanity checks: missing vias, opens, shorts, base-level DRC
6. Add UPF-driven physical cells relevant to power: **well-tap cells**, **tie-cells**, **decap cells**, **endcap/boundary cells**, **filler cells**

### Physical Cells Added (mostly during/around powerplan-placement stage)
| Cell | Purpose | When Added |
|------|---------|------------|
| Well-tap cells | Prevent latch-up | Pre-placement |
| Tie-high/Tie-low cells | Provide constant logic 1/0 without direct connection to gate (protects thin gate oxide from voltage fluctuation) | During placement (or post-route if missed) |
| Decap cells | Supply instantaneous current, reduce power droop/ground bounce | Pre-placement |
| Endcap/boundary cells | Protect boundary cells from manufacturing/base-layer DRC issues | Floorplan stage |
| Filler cells (standard-cell) | Maintain N-well continuity, fix base-layer DRC gaps | Post-placement |
| Metal fill | Fix metal density DRC | Post-route |

### Checks
- Missing vias, opens, shorts on the power mesh
- Base-level DRC (via Caliber)
- **CLP (Conformal Low Power) check** – validates UPF syntax, exactly one power + one ground supply defined, values assigned correctly

### Inputs / Outputs
| Inputs | Outputs |
|--------|---------|
| Floorplan DEF, UPF/CPF, Tech LEF | Power-planned DEF, power-mesh DRC (`.db`) report given to Caliber |

---

## 5. Placement

**Purpose:** Place all standard cells within the core boundary optimally for timing, congestion, power.

### Internal Steps (as listed in notes)
1. **Pre-placement** – place physical-only cells (endcap, well-tap, I/O buffers, antenna diodes, spare cells) before actual std-cell placement
2. **Initial/Global (Coarse) Placement** – tool finds approximate location for each cell based on timing/congestion/connectivity (cells may not sit on grid, may overlap); requires optimization settings (bounds, path groups) before running `place_opt`/`place_design`
3. **Legalization** – moves cells onto the placement grid and removes overlaps (can shift net timing slightly)
4. **Iterations for Congestion, Timing, DRV, and Power Optimization**
5. **Multi-bit flop conversion**
6. **Scan-chain reordering**
7. **Tie-cell insertion**

### Timing Optimization Techniques Used at Placement
- **Vt swapping** (HVT/SVT/LVT/ULVT trade-off between leakage and delay)
- **Upsizing** (increase drive strength)
- **Buffer/inverter insertion** (break long nets to reduce RC delay)
- **Cell cloning**
- **Pin swapping**

### Key Metrics
- **WNS** (Worst Negative Slack) – single worst-violating path
- **TNS** (Total Negative Slack) – sum of all negative slacks
- Tool fixes WNS first, which generally reduces TNS

### Checks After Placement
- Legalization check
- PG connections for all cells
- Congestion report
- Timing (no WNS violation)
- DRVs (design rule violations)
- Total utilization

### Inputs / Outputs
| Inputs | Outputs |
|--------|---------|
| Floorplan+Powerplan DEF, netlist, SDC, lib | Placed DEF (legalized), placement-stage timing/congestion reports |

---

## 6. Clock Tree Synthesis (CTS)

**Definition:** Process of building the clock distribution network to achieve **minimal skew** and **minimal latency**.

### Internal Steps (`ccopt_design` flow)
1. **Building** – tool builds a DRV-aware clock tree only (clocks not yet balanced)
2. **Balancing** – balances the tree per skew-group constraints
3. **Routing the clock trees** – clock-tree nets are routed using the NanoRoute engine
4. **Post-conditioning** – cleans up minor degradation from signal-integrity effects after clock routing

### Key Concepts
- **Latency / Insertion delay** – time for clock to travel from the clock definition point to a flop's clock pin
  - **Min insertion delay** – to the nearest flop
  - **Max insertion delay** – to the farthest flop
  - **Skew** = difference in insertion delay between two flops
- **Skew group** – set of flops sharing the same clock, balanced together
- After CTS, real clock skew is introduced (previously assumed ideal during placement), so a data-path re-optimization step is required next (**Post-CTS**).

### Checks After Post-CTS
- Setup/hold reports on user-defined path groups
- DRVs

### Setup/Hold Fix Techniques (used at Post-CTS)
| For Setup | For Hold |
|-----------|----------|
| Vt swap (HVT→LVT) | Vt swap (LVT→HVT) |
| Upsizing | Downsizing |
| Buffer insertion (break net) | Buffer insertion near capture flop |
| Cloning | Cloning |

---

## KEY INTERVIEW CONCEPTS (PD Fundamentals)

A grab-bag of concepts that come up constantly in PD interviews, beyond the stage-by-stage flow above. Grouped by theme.

### A. Placement & Routing Blockages, and Types of Regions

**Routing Blockage**
- Prevents **routing** (not placement) in a particular area, for a specific metal layer, for signal/PG nets.
- Used mainly **over macros** — a macro's internal top metal layer could otherwise short with the block's routing layers, so a routing blockage is placed above the macro to prevent this.
- Also used generally wherever two metal layers are shorting due to routing — apply a routing blockage on one of those layers in that area.

**Placement Blockage** — locations where cell placement is prevented/restricted. Three types:

| Type | What it allows |
|------|-----------------|
| **Hard placement blockage** | No standard cell allowed at all. Used only in leftover space after well-tap/endcap cells are placed (to avoid DRC violations from unfilled gaps). |
| **Soft placement blockage** | Blocks combinational/sequential cells, but **buffers/inverters are still allowed** to be placed. |
| **Partial blockage** | Partially blocks a region — allows placement in *some* of the area, blocks the rest. |

Effect on congestion (as noted):
- Soft blockage → does **not** resolve congestion
- Hard blockage → does **not** resolve congestion
- Partial blockage → **improves congestion to some extent** in certain cases

**Halo**
- A halo is essentially a placement blockage tied to a macro — it prevents standard cells from being placed within a set distance of the macro's edge (to avoid bad-DRC/congestion at the macro boundary).
- **Halo vs. soft blockage**: functionally similar (both block standard-cell placement), but the **halo moves along with the macro** if the macro is shifted, while a soft blockage stays fixed in place.

**Padding** (a related, "better form" of blockage)
- Restricts placement of other standard cells near a specific cell, rather than fully blocking a region.
- **Instance padding** — applied across an entire hierarchy of cells.
- **Cell padding** — applied to a specific cell type (e.g., all AOI cells get the same padding constraint).
- Can be specified per side: top / bottom / left / right.

Three techniques used to reduce congestion overall: **padding, partial placement blockage, and spreading cells.**

**Types of Regions**

Used when a hierarchical split in the design is contributing to a timing failure with a large slack.

| Type | Definition |
|------|--------------------------|
| **Region** | Places one specific hierarchy's cells within a particular area — but *other* hierarchies' cells are still allowed in that same region, as long as there's enough space. |
| **Fence** | Places one specific hierarchy in a particular region **while restricting all other hierarchies from entering** that region. |
| **Guide** | Places cells of one hierarchy in a specific region, but **still allows the tool to relocate those cells to a different region** if it improves timing. |

Note: when using a **fence**, the allotted area must be *sufficient* for the standard cells being placed — not more, not less.

### B. Timing Exceptions
- **False path** — a path that logically exists in the netlist but functionally will never be sensitized/used (e.g., paths through test/config muxes that only switch during reset) → excluded from timing analysis.
- **Multicycle path** — a path allowed more than one clock cycle to complete (common for slow datapath logic) → relaxes the setup constraint on that path.
- **Half-cycle path** — path expected to complete in half a clock cycle (common with DDR-style logic).
- **Case analysis** — tells the tool a certain input pin is tied to a constant value (0/1) in the current mode, letting it prune impossible logic paths.
- **Disable timing arc** — tells STA to ignore a specific timing arc inside a cell (e.g., a non-functional path in a mux).

### C. On-Chip Variation (OCV) / Derating
- **OCV** — accounts for the fact that process, voltage, and temperature (PVT) can vary slightly *across the same chip*, so two supposedly-identical cells may not behave identically.
- **Derating** — applying a margin (multiplier) to cell delays to account for this: **early derate** (<1, makes fast paths look faster — worst case for hold) and **late derate** (>1, makes slow paths look slower — worst case for setup).
- **AOCV (Advanced OCV)** — derate value depends on the *depth* (number of stages) and *distance* of the path, since variation statistically averages out over longer paths — less pessimistic than flat OCV.
- **POCV (Parametric/Statistical OCV)** — derate is applied statistically per-cell using standard-deviation-based delay modeling — most accurate, least pessimistic, but heavier compute.

### D. GBA vs PBA (Timing Analysis Modes)
- **GBA (Graph-Based Analysis)** — worst-case delay is picked *independently* for every arc in the timing graph, even if those worst cases can't physically occur together on the same path → pessimistic but fast. Used through most of the P&R flow.
- **PBA (Path-Based Analysis)** — timing is calculated *per actual path*, using consistent conditions along that one path → far more accurate, less pessimistic, but much slower (used only at signoff, and only re-analyzing the failing paths GBA flagged).

### E. Clock-Related Concepts
- **Useful skew** — deliberately introducing clock skew to "borrow" time from a path with slack and give it to a critical path (discussed under CTS above).
- **Clock gating** — inserting an **ICG (Integrated Clock Gating cell)** to stop the clock toggling into unused logic when not needed → saves dynamic power. Enable signal comes from control logic; ICG cell sits close to the leaf flops it feeds.
- **CGC (Clock Gating Check)** — STA check ensuring the enable signal to an ICG is stable and doesn't cause a glitch that could corrupt the gated clock.
- **Clock domain crossing (CDC)** — when a signal moves from one clock domain to another (asynchronous clocks); needs synchronizers (2-flop, handshake, or FIFO-based) to avoid metastability.
- **Metastability** — an unstable state a flop can enter if data changes too close to the clock edge (violates setup/hold) — output hovers between logic 0/1 before eventually settling, potentially at the wrong value or after unpredictable delay.

### F. Signal Integrity (SI)
- **Crosstalk** — coupling capacitance between adjacent parallel metal wires causes one net's switching to influence a neighboring net.
  - **Crosstalk delay (delta delay)** — victim net's delay increases/decreases depending on whether the aggressor switches in the same or opposite direction.
  - **Crosstalk glitch/noise** — aggressor switching can inject a spurious voltage bump on a quiet victim net, which is dangerous if the victim is a clock or an asynchronous set/reset.
- **NDR (Non-Default Rule) routing** — wider spacing/width rules applied to specific nets (clock, sensitive analog, high-current) to reduce crosstalk and electromigration risk.

### G. Power Concepts
- **IR drop** — voltage drop across the resistive power grid before it reaches a cell's VDD pin.
  - **Static IR drop** — average/DC drop under typical switching activity.
  - **Dynamic IR drop** — instantaneous drop during high simultaneous switching activity (worse-case, localized, short-duration).
- **Electromigration (EM)** — metal atom displacement due to sustained high current density, leading to opens (voids) or shorts (hillocks) over time.
- **Power gating** — shutting off power to unused blocks using **header/footer switch cells (MTCMOS)** to cut leakage power.
- **Isolation cells** — placed at the boundary of a power-gated (or different voltage) domain to clamp outputs to a known value when that domain is powered off, preventing floating/invalid signals from propagating.
- **Level shifter cells** — convert signal voltage swing when crossing between two different voltage domains (e.g., 0.9V domain → 1.2V domain).
- **Retention cells** — special flops that retain their state even when the surrounding logic's power is switched off (used with always-on power rail), so the block can resume instantly on power-up.
- **Always-on (AON) buffers/cells** — cells that stay powered even when the rest of the block is power-gated, to keep certain always-needed control/enable signals alive.
- **Multi-Vt design** — mixing HVT/SVT/LVT/ULVT cells across the chip to balance speed vs. leakage power (critical paths use low-Vt, non-critical paths use high-Vt to save power).

### H. Physical Verification / DFM

#### Sign-off Checks in IC Design: DRC, DRV, LVS, ERC

**1. DRC (Design Rule Checks)**
*What it verifies:* Ensures the layout follows foundry/process manufacturing rules (geometric/manufacturability check).

| Check Type | Description |
|---|---|
| Minimum Width | Metal/poly/diffusion layers meet min width rules |
| Minimum Spacing | Distance between same-layer features |
| Minimum Area | Enclosed area of shapes meets minimum |
| Enclosure Rules | Via enclosure by metal layers |
| Overlap Rules | Layer overlap requirements |
| Density Rules | Metal/poly density (min/max) for CMP uniformity |
| Antenna Rules | Metal area ratio to gate oxide |
| Notch Rules | Internal spacing within same net |
| Extension Rules | Layer extension beyond another layer |
| Grid Rules | Features must align to manufacturing grid |

**2. DRV (Design Rule Violations) / Electrical DRC**
*What it verifies:* Electrical characteristics of the layout (sometimes called eDRC).

| Check Type | Description |
|---|---|
| Max Current Density | Metal/via current handling capacity |
| EM (Electromigration) | Wire/via current limits over lifetime |
| IR Drop | Voltage drop across power/ground networks |
| Thermal Violations | Heat dissipation limits |
| Slew Rate Violations | Signal transition time limits |
| Capacitance Violations | Net capacitance exceeding limits |
| Min/Max Voltage | Operating voltage range checks |
| Power Density | Power per unit area limits |

**3. LVS (Layout vs. Schematic)**
*What it verifies:* Layout matches the schematic (netlist equivalence).

| Check Type | Description |
|---|---|
| Device Matching | Same devices in layout vs schematic |
| Net Connectivity | All connections match between layout & schematic |
| Device Count | Number of transistors/resistors/caps match |
| Device Parameters | W/L ratios, resistance values, cap values match |
| Port Matching | I/O pins match between layout & schematic |
| Short Circuits | Unintended connections between nets |
| Open Circuits | Missing/broken connections |
| Property Check | Device property values match |
| Floating Nets | Nets not connected to any device |
| Extra/Missing Devices | Device count mismatch detection |

**LVS Flow:**
```
Layout (GDSII) → Extraction → Extracted Netlist ─┐
                                                   ├→ Compare → PASS/FAIL
Schematic (CDL/SPICE) → Reference Netlist ────────┘
```

**4. ERC (Electrical Rule Checks)**
*What it verifies:* Electrical correctness and reliability of the design.

| Check Type | Description |
|---|---|
| Floating Gates | Unconnected transistor gates |
| Floating Inputs | Undriven input pins |
| VDD/VSS Shorts | Power to ground shorts |
| Multiple Drivers | Multiple outputs driving same net |
| Unconnected Pins | Pins with no connections |
| Latch-up Susceptibility | N-well/P-well tap spacing |
| ESD Violations | Missing/improper ESD protection |
| Antenna Violations | Charge accumulation on floating gates |
| Well Proximity Effect | Transistor near well edge |
| Diode Checks | ESD diode orientation/placement |
| Resistor/Cap Values | Passive component value verification |
| Bulk Connections | Proper body/bulk connections |

**Summary Comparison**

| Check | Focus Area | Tools Used |
|---|---|---|
| DRC | Manufacturability (Geometric) | Calibre, PVS, Assura |
| DRV | Electrical Physical Rules | Voltus, Redhawk |
| LVS | Layout = Schematic Equivalence | Calibre, PVS, Assura |
| ERC | Electrical Correctness/Safety | Calibre, Virtuoso ERC |

**Sign-off Criteria**
- ✅ DRC → Zero violations
- ✅ LVS → Clean (layout matches schematic)
- ✅ ERC → No critical violations
- ✅ DRV → All within limits

*Note: All these checks must PASS before sending the design for tape-out (GDSII submission to foundry).*

#### As Described in the Source Notes (PD_Q.pdf)
- **DRC** — physical check of metal width, pitch, and spacing requirements for different layers. Split into front-end-of-line layer checks (OD/oxide-diffusion, PO/polyoxide, PODE, CPODE — checked mainly at floorplan/macro stage) and metal-layer checks (shorts, opens, min cut/step, cut enclosure, via loops — checked mainly post-route). Also includes ESD violations, latch-up violations, and Go violations (metal-loop formation from non-straight pin shapes).
- **LVS** — compares extracted layout netlist vs. spice/schematic netlist: number of devices, type of devices, number of nets. Typical errors: shorts/opens, component mismatch (e.g. LVT used instead of SVT), missing component, parameter mismatch, and cell overlap causing misread device type. Guided by a foundry-supplied **rule deck** file.
- **ERC** — checks unconnected inputs/shorted outputs, gates connected directly to supply without going through tie-high/tie-low cells, and incomplete PG (VDD/VSS) connections (e.g. "N-WELL not connected to VDD").
- Overall physical verification flow (via Caliber): **DRC → ERC → Antenna violations → Metal Go violations → LVS**.

- **Double/Multi-patterning (DPT/MPT)** — at advanced nodes, some layers can't be printed in a single lithography pass, so features are split across two or more masks; DPT-aware routing avoids "same-mask" conflicts on adjacent tightly-spaced wires.
- **CMP (Chemical Mechanical Polishing) / Metal fill** — dummy metal fill is added post-route to keep metal density uniform across the die, since uneven density causes polishing (CMP) variation and thickness inconsistency during fabrication.

### I. DFT-Related (touches PD)
- **Scan chain** — flops are re-wired in test mode into a shift-register chain to load/unload test patterns (covered in scan reordering above).
- **MBIST (Memory Built-In Self-Test)** — dedicated logic to self-test embedded memories using a separate test clock, independent of the functional test flow.
- **ATPG (Automatic Test Pattern Generation)** — generates test vectors post-layout to detect manufacturing defects via the scan chain (not a PD step itself, but PD must preserve scan connectivity for it to work).

### J. Floorplan/Utilization Concepts
- **Utilization** — ratio of total standard-cell area to total core area. Typical target ~50-70%; too high → congestion/routing failure, too low → wasted die area (cost).
- **Aspect ratio** — ratio of core height to width. 1 = square; deviating from 1 stretches the floorplan, which can be needed for I/O or macro constraints but tends to increase wirelength.
- **Density screen/window** — checks local (not just global) cell density in smaller windows across the chip, since global utilization can look fine while local hot-spots are badly congested.

---
| Logic restructuring (⚠ can add congestion) | — |
| Pin swapping | Pin swapping |
| Clock pushing/pulling (useful skew) | Clock pushing/pulling |

### Routing status at CTS stage
| Net type | Route type |
|----------|-----------|
| Data nets | Early global route (trial route) |
| Power nets | Global + detailed route |
| Clock tree nets | Global + detailed route |

### Inputs / Outputs
| Inputs | Outputs |
|--------|---------|
| Placed & legalized DEF, SDC (clock definitions), skew-group constraints | CTS'd DEF, latency/skew log reports |

---

## 7. Post-CTS

Data-path optimization to fix setup/hold violations introduced by real clock skew from CTS (same optimization techniques as above table). Ends with setup/hold slack met assuming clock nets are routed but data nets are not yet finalized.

---

## 8. Routing

**Definition:** Creating physical metal connections based on logical (netlist) connectivity.

### Internal Steps (Routing Operation Stages)
1. **Global Routing** – identifies shortest routable paths at a coarse "gcell" grid level; assigns nets to layers; is congestion-aware and blockage-aware; **not DRC-aware** (does not fix design-rule violations)
2. **Track Assignment (TA)** – assigns each net to a specific track; lays down actual metal traces across the whole design at once; does **not** check physical DRC rules
3. **Detailed Routing (DR)** – fixes DRC violations after track assignment, working box-by-box using a fixed-size search window (bbox); traverses the entire design until complete
4. **Search and Repair** – fixes remaining DRC violations through multiple loops using progressively larger boxes (loop1 → loop2 → loop3 → loop4) to find more routing resources

*(Global routing, track assignment, and detailed routing can be thought of as happening in this order, with search-and-repair as cleanup.)*

### Challenges During Routing
- Clock/PG nets already routed → fewer tracks available → possible shorts
- Tool may not find legal location for a net → opens
- Avoiding DRC can cause net detouring → timing failures

### Checks After Routing
- Setup/hold timing (compared against post-CTS)
- DRVs: max fanout, max transition, max capacitance
- Noise/glitch report

### Inputs / Outputs
| Inputs | Outputs |
|--------|---------|
| CTS'd DEF, SDC, tech LEF (routing rules) | Fully routed DEF, routing DRC/timing reports |

---

## 9. Post-Route

Detailed routing introduces real RC delay on data nets (previously only estimated), which again degrades timing → requires final data-path optimization.

### Optimization Techniques
| Setup | Hold |
|-------|------|
| Vt swapping | Vt swapping (lower→higher Vt) |
| Upsizing | Downsizing |
| Buffer addition | Buffer addition (near capture flop) |
| Cloning | — |
| Logic restructuring | — |
| Pin swapping | — |

### Post-Route Also Includes
- Filler cell addition (base-layer DRC)
- Metal fill (density DRC)
- SPEF generation (extracted RC parasitics for accurate STA)

### Inputs / Outputs
| Inputs | Outputs |
|--------|---------|
| Routed DEF | Post-route DEF, SPEF, updated netlist |

---

## 10. Signoff

Signoff = final set of checks/closures before tapeout. Sub-flows:

### A. Static Timing Analysis (STA)
- Why needed even though post-route timing looked clean: SPEF gives accurate R/C values the P&R tool couldn't fully determine → real timing may still degrade
- **Inputs:** Post-route DB, SPEF, STA reports (from prior stage) for ECO generation
- **Outputs:** Setup ECO files, Hold ECO files, DRV ECO files (noise/glitch), reports (in STA/reporting tool, e.g. Tempus)

**ECO Flow (Engineering Change Order):**
1. Generate tool-based ECOs (from STA reports)
2. Source ECO files into the P&R tool (Innovus/IC Compiler) — restore post-route DB
3. Delete filler cells (needed to source ECO cleanly)
4. Legalize (place newly introduced cells legally)
5. ECO-route (re-route disturbed nets)
6. Add filler cells back
7. Save new post-route DB
8. Generate new SPEF from updated DB
9. If tool ECO can't fully fix timing → manual ECO (usually after ~1-2 tool-ECO iterations)

### B. Physical Verification
- **DRC (Design Rule Check):** metal DRC (shorts, opens, min cut/step, via enclosure), ESD violations (spacing/coverage of ESD cells), latch-up violations (missing well-tap coverage), Go/no-go metal-loop violations on standard-cell pins
- **LVS (Layout vs. Schematic):** ensures the layout matches the intended schematic/netlist

### C. Logic Equivalence Check (LEC)
- **Inputs:** Golden netlist (originally synthesized netlist; updates as ECOs are introduced) vs. Revised netlist (latest netlist after each stage)
- **Outputs:** Mismatch instance report (LEC fail) / Passing instance report (LEC pass)
- **Performed after:** Post-synthesis (timing opt.), Placement (data-path opt.), Post-CTS (data+clock opt.), Post-route (data-path opt.), and after ECO (manual ECOs)

### D. Power Signoff (IR Drop / EM)
- **Inputs:** DEF, TWF (timing window file), SPEF, Apache power library
- **Outputs:** Vector-based IR report, vectorless IR report (via tools like Redhawk — EM/IR checks)
- **IR drop:** voltage drop across power-grid metal before reaching cell VDD pins
- **Electromigration (EM):** current-driven atom movement in metal → depletion (voids/opens) or deposition (hillocks/shorts); mitigated with NDR (non-default routing) rules that widen metal to lower current density

### E. Antenna Effect Check
- Caused by charge accumulation on long metal interconnects during plasma etching, which discharges through a thin gate oxide and can permanently damage it. Fixed with antenna diodes / jumpers (bridging to a higher metal layer).

---

## Quick Reference Table — Stage vs. Internal Steps vs. I/O

| Stage | Internal Steps | Key Inputs | Key Outputs |
|-------|----------------|-----------|--------------|
| **Synthesis** | Translation → Optimization → Tech Mapping (Read RTL → Elaborate → Apply Constraints → Generic Synth → Logic Opt → Tech Map → Scan Insertion → Incremental Opt) | RTL, .lib | Netlist, SDC, Scan DEF |
| **Floorplan** | Core sizing → I/O placement → Macro placement → Row creation → Physical cells | Netlist, SDC, LEF, lib, Floorplan/Physical DEF | Floorplan DEF/LEF |
| **Powerplan** | Rings → Stripes/straps → Rails → Vias → Sanity/DRC checks | Floorplan DEF, UPF/CPF | Power-planned DEF |
| **Placement** | Pre-placement → Global/coarse placement → Legalization → Congestion/Timing/DRV/Power iterations → MBFF conversion → Scan reorder → Tie-cell insertion | Power-planned DEF, netlist, SDC | Placed & legalized DEF |
| **CTS** | Build (DRV-aware) → Balance → Route (NanoRoute) → Post-conditioning | Placed DEF, clock SDC | CTS'd DEF |
| **Post-CTS** | Data-path setup/hold optimization | CTS'd DEF | Optimized DEF |
| **Routing** | Global routing → Track assignment → Detailed routing → Search & repair | CTS'd/Post-CTS DEF | Routed DEF |
| **Post-Route** | Data-path setup/hold optimization, filler/metal fill, SPEF generation | Routed DEF | Post-route DEF, SPEF |
| **Signoff** | STA → ECO flow → Physical verification (DRC/LVS) → LEC → Power signoff (IR/EM) → Antenna check | Post-route DB, SPEF, STA reports | ECO files, final GDSII, signoff reports |

---

*Note: This document was reconstructed from a heavily OCR'd/handwritten source PDF (214 pages); wording has been cleaned up and organized, but the technical content (steps, input/output files) is drawn directly from the source notes.*

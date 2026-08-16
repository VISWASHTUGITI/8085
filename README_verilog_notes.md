# Verilog — Complete Revision Notes

---

## 1. Introduction to Verilog

### Why we need Verilog (Problem vs HDL Solution)

| Problem with Manual Design | HDL Solution |
|---|---|
| **Too Slow & Error-Prone** — manual circuit design is slow; one mistake breaks the whole system | **High-Level Description** — write code to describe hardware function, faster & fewer errors |
| **Hard to Test** — can't test a large physical circuit without building it first | **Easy to Simulate** — simulate behavior, find/fix bugs before building anything |
| **Lack of Automation** — designers manually convert designs into gates (tedious) | **Automated Synthesis** — EDA tools auto-convert code into physical gates |
| **Not Reusable** — redesigning the same block for every project wastes effort | **Reusability & Flexibility** — reuse code blocks, change parameters (e.g. bus width) without rewriting |

### Verilog HDL — Key Features
1. **Hardware Description Language (HDL)** — describes structure & behavior of a physical circuit (not step-by-step instructions like software).
2. **Industry Standard** — governed by **IEEE 1364-2001**, ensuring consistency across tools/designers.
3. **Supports Both Real-World & Test-World Design**
   - **Synthesizable constructs** → convert to actual gates/wires on a chip.
   - **Simulation-only constructs** → used for testing (e.g., `$display`, test signal generation) but never become hardware.
4. **Handles Both Digital and Analog**
   - Primarily digital, but **Verilog-AMS** (Analog & Mixed-Signal) supports designs with both digital and analog components (e.g., a phone's sensor).

### Application Areas of Verilog
Flow: **System Specification → HW/SW Partition → Hardware Spec / Software Spec → Boards & Systems**

- **System Specification** — defines overall functionality: performance, power, area, target technology.
- **HW/SW Partition** — divides system into hardware and software (crucial for embedded design).
- **Hardware Specification** — implemented as:
  - **ASIC** — custom chip for one specific purpose.
  - **FPGA** — reconfigurable chip.
  - **PLD** — simple programmable logic chip.
  - **Std Parts** — off-the-shelf components.
- **Software Specification** — code running on processors/microcontrollers.
- **Boards and Systems** — final product combining hardware + software.

### Abstraction Levels (lowest → highest)

| Level | Description |
|---|---|
| **1. Circuit Level** | Lowest level — built using fundamental MOS switches (`pmos`, `nmos`), not gates. Rarely used — too tedious for large designs. |
| **2. Gate Level** | Design using basic logic **primitives** (`and`, `or`, `not`) — creates a netlist mapping directly to physical structure. |
| **3. Data Flow Level** | Uses **continuous assignments** (`assign`) to describe how data flows between signals via Boolean equations/operators — creates combinational logic. |
| **4. Behavioral Level** | Highest level — describes circuit **behavior** using procedural blocks (`always`, `initial`) and constructs (`if/else`, `case`, loops). Ideal for high-level modeling & testbenches. |

### Language Concepts
- **Concurrent statements** — run at the same time (e.g., multiple `assign`/`always` blocks).
- **Procedural statements** — executed **sequentially**, one after another, inside `always`/`initial` blocks; modeled with `if...else`, `case`, loops.

### Hierarchical Design
- **Design distribution** → multiple designers can work in parallel.
- **Smaller code pieces** → easier to debug.
- **Design reuse** → sub-modules reused across the hierarchy.
- Structure: `Top Level (Chip) → Sub-System IPs → Basic Modules`.

### Verilog Syntax Basics
- Keywords are **lowercase** (`module`, `endmodule`, `input`, `output`, `wire`, `reg`, etc.)
- **Identifiers are case-sensitive.**
- Identifiers: alphanumeric, underscore `_`, and `$` allowed — **must not start with a digit**.
- Comments:
  - Single line: `// comment`
  - Multi-line: `/* comment */`
  - **Nesting comments is NOT allowed.**

### Verilog Module — Structure
```verilog
module module_name(a, b, y, z);
  input a, b;
  output y, z;
  statement1;
  statement2;
endmodule
```
- `module` / `endmodule` — mandatory keywords.
- **Port declarations**: `input`, `output`, `inout` — default type is `wire`.
- Functionality defined using `assign` (dataflow) or procedural blocks.

**Example — Half Adder:**
```verilog
module half_adder(a, b, sum, carry);
  input a, b;
  output sum, carry;
  assign sum = a ^ b;
  assign carry = a & b;
endmodule
```

**Example — Full Adder (structural, using instantiation):**
```verilog
module full_adder(a, b, c, sum, carry);
  input a, b, c;
  output sum, carry;
  wire w1, w2, w3;
  half_adder HA1(.a(a), .b(b), .sum(w1), .carry(w2));
  half_adder HA2(.a(w1), .b(c), .sum(sum), .carry(w3));
  or or1(carry, w2, w3);
endmodule
```
- Named/instance names (e.g., `HA1`, `HA2`, `or1`) are used when referencing instances.

---

## 2. Data Types

### Signal Values
- `0` → logic zero / false
- `1` → logic one / true
- `x` → unknown logic value
- `z` → high-impedance state

### Strength Levels (highest → lowest)
`Supply → Strong → Pull → Large → Weak → Medium → Small → High-Z`
- **Strongest signal prevails** when drivers conflict.
- Two signals of **opposite value, same strength** → result is `x`.
- Used mainly for **switch-level modeling**, not RTL.
- Strength is a **simulation-only concept** — doesn't translate into hardware.
- **Default wire strength**: `strong1`, `strong0`.

### Numbers
Format: `<size>'<base><number>`
- Base: `b`/`B` (binary), `o`/`O` (octal), `d`/`D` (decimal), `h`/`H` (hex)
- Example: `12'h43xz` → 12-bit hex number containing digits, `x`, `z`.
- Underscores `_` improve readability (e.g., `16'b0000_0000_0000_0001`).
- **Unsized numbers** default to **32-bit**; missing upper bits are zero-extended (with x/z extension rules for x/z digits).

### Vectors (Buses)
```verilog
reg [3:0] z_bus;      // MSB = bit3, LSB = bit0
wire [1:0] a_bus;
a_bus = z_bus[2:1];   // slice — direction must match declared range
```
- Range direction of a slice must match the vector's declared direction.
- **Vector assignment is positional**, not by bit-name — direction (`[3:0]` vs `[0:3]`) must be handled consistently across both sides, or bits get swapped unintentionally.

### Nets (`wire` and related types)
- Continuously driven by combinational logic (represent physical connections).
- **Must be driven by something** — cannot store a value on their own.
- Legal **only on the LHS of a continuous `assign`** statement — **cannot** be assigned inside `always`/`initial` (procedural blocks).
- Common net types: `wire`, `wand`, `wor`, `supply0`, `supply1`, `trireg`.

#### Multiple-Driver Conflict Resolution
- **`wand` (wired-AND):** if **any** driver is 0 → net is 0.
- **`wor` (wired-OR):** if **any** driver is 1 → net is 1.

**Example logic (`wand`):**
- If `a=0, b=1` → `y = 0` (wired AND: any 0 wins)
- If both drivers go to `z` → net holds high impedance behavior per type.

#### `trireg`
- Stores a value — models **charge storage nodes** (capacitive nodes).
- **Driven state** — when ≥1 driver outputs 0/1/x, that value propagates and becomes the trireg's value.
- **Capacitive state** — when **all** drivers go to `z`, the trireg **retains its last driven value** (like a `reg`).

### `reg` Data Type
- **Does NOT automatically create a hardware register/flip-flop** — it's just a variable that retains its assigned value until updated.
- Legal **only** on LHS of procedural blocks (`always`/`initial`) — **cannot** be used on LHS of a continuous `assign`.
- `reg` is **unsigned by default**; can be made signed: `reg signed a;`
- Actually creates real hardware registers **only when used with clocked `always @(posedge clk)`**.
- In pure simulation (no clocked always), it just holds its value because the simulator remembers it — no hardware implied.
- Common `reg`-family types: `integer`, `real`, `time`.

### Integer Data Type
- Stores **signed** values (e.g., `0, 1, 256, -43`).
- Good for counting / signed arithmetic.
- **Mostly synthesizable** ✗ (not fully — check tool support).
- Arithmetic on integers gives 2's complement results.
- Assigned via procedural assignments.

### Real Data Type
- Allows decimal & scientific notation (`0.125`, `1.37e-2`).
- **Not synthesizable.**
- Assigning `real` to `integer` **rounds off**:
  - fractional part ≥ 0.5 → rounds **up**
  - fractional part ≤ −0.5 → rounds **down** (toward previous integer)

### Time Data Type
- Stores **simulation time** — used in testbenches.
- **Not synthesizable.**
- At least 64 bits, **unsigned**.
- Assign current simulation time using system function: `sim_time = $time;`

### Arrays
- Arrays of `reg`, `integer`, `time`, `real`, `realtime` are supported, and also net arrays.
```verilog
reg my_reg [0:5];          // array of 6 one-bit registers
my_reg[2] = 1'b1;

reg [1:n] rega;            // n-bit single register — NOT the same as below
reg rega [1:n];            // array of n, 1-bit registers
```

### Memories
- A **1-D array** of `reg` elements — models ROM, RAM, register files.
```verilog
reg [7:0] mema [0:255]; // 256 registers, each 8 bits wide
mema = 0;      // ILLEGAL — cannot write entire memory in one assignment
mema[1] = 0;   // Legal — assigns one word
```
- **An n-bit `reg` can be assigned in one shot; a complete memory array cannot.**

### Parameters
- **Constants, not variables** — cannot be assigned during simulation.
```verilog
module ram(clk, rst, din, radd, wadd, dout);
  parameter dt_width = 8;
  parameter ad_depth = 256;
  reg [dt_width-1:0] mem [ad_depth-1:0];
endmodule
```
- **Overriding parameters at instantiation:**
  - By **ordered list**: `ram #(16) MEM1 (...);` — override values must be **contiguous** (can't skip one).
  - By **name**: `ram #(.dt_width(16), .ad_depth(512)) MEM2 (...);`
  - Using **`defparam`**: `defparam U0.COIN = 2'b10;`
- **Limitations:**
  - Cannot be declared outside a module → **not global**.
  - Cannot hold text values → can't alias data types.

### Strings
- Sequence of characters in double quotes `"..."`, single line only.
- Each character = **8-bit ASCII** → needs `8 × (num_chars)` bits of storage.
```verilog
reg [8*13:1] string_reg;  // 104 bits for 13 characters
string_reg = "Radha_Krishna";
$display("Company Name -> %s", string_reg);
```
- Escape characters: `\n` (newline), `\t` (tab), `\\` (backslash), `\"` (quote).

### Default Values of Data Types

| Data Type | Default Value |
|---|---|
| `wire` (net) | High impedance `z` |
| `reg` | Unknown `x` |
| `integer` | Unknown `x` |
| `real` | `0.0` |
| `time` | Unknown `x` |

---

## 3. Operators

### Logical Operators — `&&` (AND), `\|\|` (OR), `!` (NOT)
- Evaluate to a **single bit**: 1 (true), 0 (false), x (unknown).

### Bitwise Operators — `&`, `\|`, `^` (XOR), `~` (NOT), `~^` or `^~` (XNOR)
- Operate **bit-by-bit** on operands of equal width.

### Reduction Operators — `&`, `\|`, `^`, `~&` (NAND), `~^`/`^~` (XNOR), `~\|` (NOR)
- Applied to a **single vector operand**, collapsing all bits into **one result bit**.
- e.g. `&4'b0110` → ANDs all 4 bits together → `0`.

### Shift Operators
- **Logical shift**: `<<` (left), `>>` (right) — vacated bits filled with `0`.
- **Arithmetic shift**: `<<<`, `>>>` — for `>>>`, vacated MSB bits are filled with the **sign bit** if the operand is signed (else 0). Arithmetic left shift `<<<` behaves same as logical left shift.
- Result width = same as operand width.

### Concatenation Operator — `{ }`
- Joins bits from two or more **sized** expressions.
```verilog
z = {x[7:0], b[2:1], c};
```
- **All operands must be sized** (no unsized numbers allowed in concatenation).

### Replication Operator — `{n{expr}}`
- Repeats (replicates) an expression `n` times.
```verilog
x = {2{a}, b, {2{c}}};
```

### Relational Operators — `>`, `<`, `>=`, `<=`
- Evaluate to 0 (false) or 1 (true); result is `x` if any operand bit is `x`/`z` and comparison is ambiguous.

### Equality Operators
- **Logical**: `==`, `!=` — result can be `x` if operands contain `x`/`z`.
- **Case**: `===`, `!==` — compares `x`/`z` bits literally too; result is always 0 or 1, **never x**.
- **Case equality (`===`/`!==`) is for simulation only — not synthesizable.**

### Conditional (Ternary) Operator
```verilog
result = condition ? true_expr : false_expr;
```

### Arithmetic Operators — `*`, `+`, `-`, `/`, `%` (modulus)
- Multiplication is **expensive on gates** (hardware cost).

#### Arithmetic Pitfalls (IMPORTANT for exams)
- **Negative register/integer values are stored in 2's complement**, but Verilog arithmetic operators by default treat operands as **unsigned** unless explicitly declared `signed`.
- **Rule**: 
  - If **ALL** operands in an expression are `signed` → Verilog does **signed arithmetic**.
  - If **ANY** operand is unsigned → Verilog does **unsigned arithmetic** on the whole expression.
- **Fix**: explicitly declare `reg`/`wire` as `signed` when signed arithmetic is needed.

### Operator Precedence (highest → lowest)
```
+ - ! ~ (unary)          highest
* / %
+ - (binary)
<< >> <<< >>>
< <= > >=
== != === !==
& ~&
^ ~^ / ^~
| ~|
&&
||
?:                          lowest
```

---

## 4. Compiler Directives and System Tasks

### Display Tasks
| Task | Behavior |
|---|---|
| `$display` | Prints once, immediately; auto-adds newline |
| `$write` | Same as `$display` but **no auto newline** |
| `$strobe` | Prints at the **end of the current simulation time step** (after all events at that time have settled) |
| `$monitor` | Prints **whenever any variable in its argument list changes**, at end of time step |

- `$monitoroff` / `$monitoron` — disable/enable monitoring.

**Format specifiers**: `%h`/`%H` hex, `%d`/`%D` decimal, `%o`/`%O` octal, `%b`/`%B` binary, `%m`/`%M` hierarchical name, `%s`/`%S` string, `%t` time.

### File I/O System Tasks
- `$fopen("filename")` → returns a 32-bit **channel/file descriptor** (only one bit set, e.g. channel1 = `32'h0000_0002`).
- `$fclose(channel)` → closes the file.
- `$fmonitor(channel, ...)` / `$fdisplay(channel, ...)` → write formatted output to a file.
- `$readmemb("file.txt", mem_array)` → loads binary data into a memory array from a text file (uninitialized addresses remain `x`).
- Combining channels: `comb = chanel_1 | chanel_2;` writes to both files simultaneously.

### Simulation Control Tasks
- `$stop` — **suspends** simulation (can be resumed).
- `$finish` — **terminates** simulation and returns control to the OS.

### Simulation Time Function
- `$time` — returns simulation time as a 64-bit unsigned integer, scaled to the invoking module's timescale.

### Random Number Generation
- `$random` — returns a new signed 32-bit random number each call (can be +ve or −ve).
- `$random(seed)` — seed controls the reproducible sequence.
- `$random % b` → range `[-(b-1) : (b-1)]`
- `{$random} % b` → range `[0 : (b-1)]` (unsigned trick using concatenation)

### Compiler Directives
| Directive | Purpose |
|---|---|
| `` `define `` | Creates a **macro** for text substitution; used with `` ` `` prefix (e.g., `` `dt_width ``). **Global** across the project; **cannot be overridden**. |
| `` `include `` | Inserts the entire content of another source file at compile time (for shared definitions/tasks). |
| `` `timescale `` | Sets **time unit** and **time precision**: `` `timescale 1ns/10ps `` → time unit = 1ns, precision = 10ps. `time_precision` cannot be coarser than `time_unit`. Mixing modules with/without `timescale` causes errors. |

**`parameter` vs `` `define ``:**
| | `parameter` | `` `define `` |
|---|---|---|
| Scope | Local to module | Global (whole project) |
| Overridable | Yes | No |

---

## 5. Verilog — Assignments

### Continuous Assignments (`assign`)
- Drives **nets (`wire`)** only — cannot assign to `reg`.
- Re-evaluates **whenever the RHS changes** (like combinational hardware).
- Executes in **parallel**, order-independent.
- Can be written as an **implicit continuous assignment** in the declaration: `wire y = a & b;`

**With delay:**
```verilog
assign #2 c = a & b;
```
- RHS is computed **after a 2-time-unit delay** before assignment to LHS — models gate propagation delay.
- Rapid/glitchy input changes narrower than the delay may not appear at the output (inertial delay behavior).

### Procedural Assignments
- Update variables (`reg`) **under control of procedural flow** (`initial`/`always`).
- Each procedural block is a separate parallel activity; **order across multiple blocks is indeterministic** unless synchronized.

#### `initial` Block
- Executes **once**, starting at simulation time 0.
- Multiple `initial` blocks run **in parallel**.
- Simulation ends when `$finish` executes (or all blocks complete).

#### `always` Block
- Repeats **continuously** throughout simulation.
- Runs in parallel with other blocks.
- **Without a timing/event control, `always` creates a simulation deadlock** (infinite loop with no time advance).

### Event Controls — `@`
```verilog
always @(posedge clk)     // rising edge trigger
always @(negedge clk)     // falling edge trigger
always @(a or b or sel)   // sensitivity list — triggers on change of a, b, or sel
```
- **Posedge**: 0→1, 0→x/z, x/z→1.
- **Negedge**: 1→0, 1→x/z, x/z→0.
- **Missing a signal from the sensitivity list** → output won't update when that signal changes (simulation-synthesis mismatch bug).
- **Implicit sensitivity list**: `always @(*)` automatically includes all RHS signals used in the block — avoids missing-signal bugs.

### Level-Sensitive Event Control — `wait`
```verilog
wait(enable) #10 a = b;
```
- Delays evaluation until `enable` becomes true (level-sensitive, not edge-sensitive).
- **Not synthesizable** — simulation only.

### Procedural Assignment Delays
- **Regular delay** (before RHS is evaluated): `#5 b = 1'b1;` — RHS evaluated & assigned after 5 time units.
- **Intra-assignment delay** (RHS evaluated *now*, assignment delayed): `a = #10 b;` — `b` is captured at current time, assigned to `a` 10 units later. Equivalent to using a temp variable + regular delay.
- Applicable to both blocking and non-blocking assignments. **Not synthesizable** — simulation only.

### Block Statements

| Type | Keyword | Execution |
|---|---|---|
| **Sequential block** | `begin...end` | Statements execute **in order**; delays are relative to **previous statement's** completion time |
| **Parallel block** | `fork...join` | Statements execute **concurrently**; delays are relative to the **time the block was entered**; block exits after the **last (max-delay) statement** finishes |

- Both types can be **named**: `begin: block_name` / `fork: block_name` — enables local variable declaration and `disable` (to terminate the named block early).

**Improper vs Proper Encapsulation:**
- Declaring a loop variable (e.g., `integer i;`) **globally** and using it in two separate `always` blocks causes **interference** between blocks.
- **Fix**: declare `i` **locally inside a named block** (`begin: block1 integer i; ... end`) so each block gets its own copy.

---

## 6. Structural Procedures

### Blocking Assignment (`=`)
- Executes **sequentially** — each statement **blocks** the next from executing until it finishes.
- Delay on one statement pushes back all subsequent statements.
- **Best practice: use for combinational logic** (to avoid accidentally inferring sequential behavior).

### Non-Blocking Assignment (`<=`)
- Execution is **concurrent** with the next statement — does **not** block.
- **All RHS values are evaluated first**, then **all assignments happen simultaneously** at the end of the time step.
- **Illegal** to use in continuous `assign` or net declarations.
- Order of execution among distinct non-blocking assignments **to the same variable** is preserved.
- **Best practice: use for sequential logic** (flip-flops / `always @(posedge clk)`).

**Classic example — shift register:**
```verilog
// Non-blocking → correct 2-stage pipeline (Q1 gets old D, Q2 gets old Q1)
always @(posedge Clock) begin
  Q1 <= D;
  Q2 <= Q1;
end

// Blocking → Q2 gets the NEW Q1 value in the same cycle (collapses to 1 stage — usually wrong!)
always @(posedge Clock) begin
  Q1 = D;
  Q2 = Q1;
end
```

### Coding Guidelines (IMPORTANT — commonly asked)
1. Use **blocking (`=`)** for **combinational** circuits.
2. Use **non-blocking (`<=`)** for **sequential** circuits.
3. **Never mix** blocking and non-blocking assignments in the **same** `always` block.
4. When combining combinational + sequential logic in **one** `always` block, code it as a **sequential block using non-blocking assignments**.

### `if...else`
- Implies **priority** — first matching condition wins; synthesizes to a **priority-encoded cascading mux**.

### Case Statement
```verilog
case (expr)
  item1: statement1;
  item2, item3: statement2;   // multiple values, one action
  default: default_statement;
endcase
```
- Synthesizes to a **flat ("tall and skinny"), non-priority mux** — **preferred style for multiplexers**.
- `casex` — treats `x` **and** `z` in case items as **don't-cares** (matches anything).
- `casez` — treats only `z` as don't-care.
- `casex`/`casez` are synthesizable but **not recommended** (can hide bugs by over-matching).

### Looping Statements

| Loop | Synthesizable? | Notes |
|---|---|---|
| `for` | ✅ Yes | Standard 3-step: init, condition check, step |
| `while` | ❌ No | Executes while condition is true; simulation/testbench only |
| `repeat(n)` | ❌ No | Executes statement a fixed `n` times |
| `forever` | ❌ No | Executes continuously (e.g., clock generation) |

> **`if-else`, `case`, and `for` loop ARE synthesizable. `while`, `repeat`, `forever` are NOT synthesizable.**

### Races in Simulation
- **Race condition**: occurs when two expressions are scheduled at the same simulation time and execution order is **not determined**.
- **Write-Write Race**: same register written by two different `always` blocks in the same time step.
- **Read-Write Race**: one block reads a signal while another writes to it in the same time step.
- **Fix**: use **non-blocking assignments** — they resolve races by scheduling all RHS reads before any LHS updates.

### Multi-Source Errors
- Occurs when **different logical outputs are driven onto the same `reg` variable** by two concurrent procedural blocks — an **error** with both blocking and non-blocking assignments.

### Tasks vs Functions

| | Function | Task |
|---|---|---|
| Can call | Other functions only | Other tasks AND functions |
| Simulation time | Always **zero** time | Can consume **non-zero** time |
| Delay/timing control | **Not allowed** | **Allowed** |
| Arguments | ≥1 input required; **no output/inout** | Zero or more; can have `input`, `output`, `inout` |
| Return value | **Single value** returned | No return value (passes via output/inout args) |
| Usable in | RTL (synthesis) only | Both RTL and testbench |

```verilog
function [31:0] parity_cal;
  input [31:0] data;
  parity_cal = ^data;   // XOR reduction
endfunction

task parity_cal;
  input [31:0] data;
  output parity;
  #1 parity = ^data;
endtask
```

---

## 7. Verilog — Synthesis

### Specifying Registers
- A hardware register (flip-flop) is inferred from `always @(posedge clock)` with non-blocking assignments to a `reg`.

### Asynchronous vs Synchronous Reset

```verilog
// Asynchronous reset — reset is in the sensitivity list
always @(posedge clock or posedge reset)
  if (reset) q <= 1'b0;
  else       q <= d;

// Synchronous reset — reset only checked on clock edge
always @(posedge clock)
  if (reset) q <= 1'b0;
  else       q <= d;
```
- **Asynchronous**: reset acts immediately regardless of clock.
- **Synchronous**: reset only takes effect on the active clock edge.

### Unwanted (Unintended) Latches
- Caused by **incomplete `if`/`case`** in a **combinational** `always` block (no `else`, or missing `default`, or a signal not assigned in every branch).
- Verilog infers a **latch** to "hold" the previous value when no branch matches — usually **unwanted** in combinational design.

**Avoiding Latches:**
1. Always include a **default/else** branch.
2. **Assign every output in every branch.**
3. Use a **default assignment at the top** of the always block before the case/if, so every signal has a value regardless of which branch executes.

```verilog
always @(a or b or sel) begin
  out1 = a;      // default assignment
  out2 = a;
  case (sel)
    2'b00: begin out1 = a; out2 = b; end
    default: begin out1 = a; out2 = a; end
  endcase
end
```

### Synthesis of Operators
- Most operators are synthesizable, but the **exact hardware architecture** (e.g., ripple-carry vs carry-look-ahead adder — small vs fast) depends on the **synthesis tool**.
- **Parentheses can optimize the logic structure** — grouping operations changes the resulting circuit depth/structure:
  ```verilog
  y = a + b + c + d;      // may synthesize as a long chain
  y = (a + b) + (c + d);  // may synthesize as a balanced tree — often faster
  ```

### Resource Sharing
- If the same operation (e.g., an adder) is used in mutually-exclusive branches (`if`/`else`), synthesis tools can **share one physical adder** with a mux selecting the inputs, instead of instantiating two separate adders.
- Some tools do this automatically; otherwise, code can be manually restructured to force sharing.

### `if-else` vs `case` for Multiplexers
- `if...else if` → infers a **priority-encoded cascading mux** (first condition has highest priority).
- `case` → infers a **flat, non-priority mux** ("tall and skinny") — **preferred style for designing multiplexers**.

---

## 8. Finite State Machines (FSM)

### Mealy vs Moore Architecture

| | Mealy FSM | Moore FSM |
|---|---|---|
| Output depends on | **Present state AND inputs** | **Present state only** |
| Structure | Comb. logic (next state) → Seq. logic (state reg) → Comb. logic (output, driven by state + input) | Comb. logic (next state) → Seq. logic (state reg) → Comb. logic (output, driven by state only) |
| Response | Faster (can react within same clock cycle) | Slower but more stable/glitch-resistant |

### Three-Block FSM Structure
1. **Present-State Logic (Sequential)** — updates `STATE <= NEXT_STATE` on clock edge (with reset).
2. **Next-State Logic (Combinational)** — a `case` statement over `STATE` determines `NEXT_STATE` based on inputs.
3. **Output Logic (Combinational or Registered)** — determines outputs based on state (Moore) or state+input (Mealy).

```verilog
parameter IDLE = 2'b00, RW_CYCLE = 2'b01, INT_CYCLE = 2'b10, DMA_CYCLE = 2'b11;
reg [1:0] state, next_state;

// 1. Present state logic
always @(posedge clock or posedge reset)
  if (reset) state <= IDLE;
  else       state <= next_state;

// 2. Next state logic
always @(state or RW or INT_REQ or DMA_REQ) begin
  case (state)
    IDLE:      next_state = ...;
    RW_CYCLE:  next_state = ...;
    // ...
  endcase
end

// 3. Output logic (Moore-style, combinational)
assign ds = (state == DONE);
```

### FSM Coding Styles (commonly tested)
1. **Style 1** — 2 always blocks: sequential state register + combinational next-state; outputs via simple continuous `assign` from state.
2. **Style 2** — Next-state AND outputs combined in a **single combinational always block**.
3. **Style 3** — 3 always blocks: sequential state register + combinational next-state + a **separate sequential always block for registered (clocked) outputs**.
4. **Style 4 (Output-Encoded)** — outputs are **directly assigned from state register bits** (`assign {ds, rd} = state[1:0];`) — very efficient, no extra output logic needed.

### FSM Design Guidelines (Summary — important for viva/exam)
- Partitioning FSM so there's **no combinational logic directly on FSM outputs** simplifies synthesizing multi-module designs.
- **Registered outputs** (Style 3/4) eliminate combinational output logic and guarantee **glitch-free** outputs.
- **Output-Encoded style** (Style 4) is an efficient technique to drive registered outputs directly from state bits.

---

## 9. Design Examples & Common Patterns

### Half Adder / Full Adder
```verilog
module half_adder(a, b, sum, carry);
  input a, b;
  output sum, carry;
  assign sum = a ^ b;
  assign carry = a & b;
endmodule

module full_adder(a, b, c, sum, carry);
  input a, b, c;
  output sum, carry;
  wire w1, w2, w3;
  half_adder HA1(.a(a), .b(b), .sum(w1), .carry(w2));
  half_adder HA2(.a(w1), .b(c), .sum(sum), .carry(w3));
  assign carry = w2 | w3;
endmodule
```

### 2:1 Multiplexer (behavioral)
```verilog
module mux_2_1(data_in0, data_in1, sel_in, y_out);
  input data_in0, data_in1, sel_in;
  output reg y_out;
  always @(data_in0, data_in1, sel_in)
    if (sel_in) y_out = data_in1;
    else        y_out = data_in0;
endmodule
```

### RAM Design Concepts

**Single-Port vs Dual-Port RAM**

| Feature | Single-Port RAM | Dual-Port RAM |
|---|---|---|
| Ports | 1 | 2 |
| Access | One operation (Read OR Write) at a time | Two operations simultaneously |
| Use case | Simple systems, fewer memory accesses | High-speed systems needing parallel access |
| Control signals | One set (addr, data, control) | Two independent sets |
| Conflict handling | Not applicable | Needed if both ports write same address |
| Hardware | Simple, area-efficient | More complex, larger area |

**Synchronous vs Asynchronous RAM**

| Feature | Synchronous RAM | Asynchronous RAM |
|---|---|---|
| Clock dependency | Operations depend on clock edges | Operations happen immediately (combinational) |
| Timing | Deterministic, easy to time | Less predictable, depends on signal changes |
| Latency | May have 1-cycle delay | Usually faster but less controlled |
| Used in | FPGAs, ASICs, modern digital systems | Small/simple/legacy systems |
| Design | Easier to integrate with synchronous systems | Trickier to manage with rest of system |

**Synchronous Dual-Port RAM (single shared clock) vs Asynchronous Dual-Port RAM (separate `wr_clk`/`rd_clk`)**

| Feature | Synchronous Dual-Port | Asynchronous Dual-Port |
|---|---|---|
| Clock domains | One shared clock | Separate read/write clocks |
| Timing sensitivity | Deterministic | Must manage carefully across domains (CDC) |
| Use case | Internal caches, same-clock-domain access | Cross-clock-domain buffering (e.g., UART↔CPU), needs FIFOs |
| Complexity | Simpler synchronization | Needs CDC (clock-domain-crossing) handling |

**Synchronous Dual-Port RAM — RTL skeleton:**
```verilog
module dual_port(clk, rst, we, re, d_in, wr_addr, rd_addr, d_out);
  input clk, rst, we, re;
  input [7:0] d_in;
  input [3:0] wr_addr, rd_addr;
  output reg [7:0] d_out;
  reg [7:0] mem [15:0];
  integer i;
  always @(posedge clk) begin
    if (rst) begin
      for (i = 0; i < 16; i = i + 1) mem[i] <= 0;
      d_out <= 0;
    end else begin
      if (we) mem[wr_addr] <= d_in;
      if (re) d_out <= mem[rd_addr];
    end
  end
endmodule
```

**Asynchronous Dual-Port RAM — key idea:** separate `always` blocks for write (`posedge clk_wr`) and read (`posedge clk_rd`), each with its own reset handling — enables cross-clock-domain operation.

---

## 10. Quick-Revision Cheat Sheet

- **Nets (`wire`)** → driven by `assign`, cannot hold value without a driver, default value `z`.
- **`reg`** → holds value until reassigned, only in procedural blocks, default value `x`.
- **`assign`** → continuous, drives nets, parallel/order-independent.
- **`initial`** → runs once from time 0.
- **`always`** → runs repeatedly; needs a sensitivity list/timing control or it deadlocks.
- **Blocking (`=`)** → sequential execution → use for **combinational** logic.
- **Non-blocking (`<=`)** → concurrent scheduling → use for **sequential** logic (flip-flops).
- **Never mix `=` and `<=`** in the same `always` block.
- **`if-else`** → priority mux. **`case`** → flat mux (preferred for muxes).
- **Synthesizable loops:** `for`. **Non-synthesizable:** `while`, `repeat`, `forever`.
- **Latches** are formed from incomplete combinational assignments — always cover all branches / use default assignment.
- **Function**: zero time, returns 1 value, no delays, RTL only.
- **Task**: can take time, has output/inout args, usable in RTL + testbench.
- **Race conditions** (write-write / read-write) → fixed using non-blocking assignments.
- **Async reset**: `posedge reset` in sensitivity list. **Sync reset**: reset checked only on clock edge.
- **Mealy** = output depends on state + input (faster). **Moore** = output depends on state only (glitch-free, stable).
- **`` `define ``** = global, not overridable. **`parameter`** = local to module, overridable (ordered list, by name, or `defparam`).
- **`===`/`!==`** (case equality) → simulation only, treats x/z literally, never synthesizable.
- Signed arithmetic requires **ALL** operands to be `signed`; otherwise Verilog does unsigned arithmetic.

---

*Compiled from your "Verilog" course slide deck (Introduction → Data Types → Operators → System Tasks → Assignments → Structural Procedures → Synthesis → FSMs → RAM/Design Examples).*

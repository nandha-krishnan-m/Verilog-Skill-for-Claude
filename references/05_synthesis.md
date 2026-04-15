# Verilog HDL — Synthesis Constructs

## Overview

Synthesis tools translate RTL Verilog into a gate-level netlist. Not all Verilog
constructs are synthesizable. Understanding this boundary is critical for writing
correct RTL.

**Key principle:** Synthesis tools ignore timing/delays — they only infer logic
structure from the connectivity and conditions in your code.

---

## 1. Fully Supported Constructs

These work identically in simulation and synthesis:

```verilog
// Module instantiation (named and positional)
dff u1 (.q(q), .d(d), .clk(clk));

// Integer data types (all bases)
reg [7:0] count;
parameter WIDTH = 8;

// Continuous assignments
assign y = a & b;

// Operators: >>, <<, ?:, {}
assign shifted = data >> 2;
assign mux_out = sel ? a : b;
assign bus     = {a, b, c};

// Procedural constructs
begin, end
case, casex, casez, endcase
default
assign (procedural), deassign — with restrictions
if, else, else if
input, output, inout
wire, wand, wor, tri
integer, reg
macromodule, module
parameter
supply0, supply1
task, endtask
function, endfunction
disable
```

---

## 2. Partially Supported Constructs

Work in synthesis but with restrictions:

| Construct | Restriction |
|-----------|-------------|
| `*`, `/`, `%` | Only when both operands are constants, OR 2nd operand is power of 2 |
| `always` | Only edge-triggered (`posedge`/`negedge`) events — no level-sensitive loops |
| `for` | Must be bounded by static (compile-time constant) variables; use only `+` or `-` to index |
| `posedge`, `negedge` | Only with `always @` — not in other contexts |
| `primitive` / `endprimitive` | Combinational and edge-sensitive UDPs often supported |
| `<=` (non-blocking) | Limitations on usage with blocking assignment (never mix) |
| Gate types | `and`, `nand`, `or`, `nor`, `xor`, `xnor`, `buf`, `not`, `bufif0/1`, `notif0/1` — without X/Z constructs |

---

## 3. Ignored Constructs

Synthesis tools silently ignore these — no error, no effect on netlist:

```verilog
// All timing/delay specs are ignored
#10 q = d;           // delay stripped
assign #5 y = a;     // delay stripped

scalared, vectored   // net attributes
small, medium, large // charge strengths
specify              // timing specs (for STA tools, not synthesis)
time                 // some tools treat as integer

// Strength keywords
weak1, weak0, highz0, highz1, pull0, pull1

// Some system tasks used for synthesis constraints
$keyword             // tool-specific

wait                 // some tools support if bounded
```

---

## 4. Unsupported Constructs

These will either cause synthesis **errors** or **simulation/synthesis mismatches**:

```verilog
// Cause synthesis errors or unexpected results:
=== , !==           // 4-value comparison — simulation only
cmos, nmos, pmos,
rcmos, rnmos, rpmos // switch-level primitives

deassign            // procedural continuous — sim only
defparam            // parameter override — avoid
event               // event data type — sim only
force               // forced assignment — sim only
fork, join          // parallel block — sim only
forever, while      // unbounded loops — sim only
initial             // time-zero block — sim only (except FPGA RAM init)
pullup, pulldown    // net primitives — sim only
release             // paired with force — sim only
repeat              // loop — sim only
rtran, tran,        // bidirectional switches
tranif0, tranif1,
rtranif0, rtranif1
table, endtable,    // UDP tables
primitive, endprimitive (sequential)
```

---

## 5. Inference Rules — What Gets Synthesized

### Flip-Flop Inference
```verilog
// This ALWAYS infers a flip-flop:
always @(posedge clk) begin
    q <= d;         // edge-triggered → D flip-flop
end

// With async reset → DFF with async reset:
always @(posedge clk or posedge rst) begin
    if (rst) q <= 0;
    else     q <= d;
end
```

### Latch Inference (usually unintentional)
```verilog
// This infers a LATCH (no else):
always @(*) begin
    if (en) q = d;      // missing else → latch
end

// Fix: add else
always @(*) begin
    if (en) q = d;
    else    q = q;      // explicit hold (still a latch — intentional)
end

// OR: assign default at top
always @(*) begin
    q = q;              // default
    if (en) q = d;
end
```

### Combinational Logic Inference
```verilog
// always @(*) with complete assignments → combinational logic
always @(*) begin
    out = 1'b0;         // default at top → no latch
    if (sel) out = in;
end
```

### Memory Inference
```verilog
// RAM inference (tool-dependent):
reg [7:0] mem [0:255];

always @(posedge clk) begin
    if (wr_en)
        mem[addr] <= data_in;
end
assign data_out = mem[addr];    // async read
```

---

## 6. Reset Style Comparison

| Style | Pros | Cons |
|-------|------|------|
| Async reset | Works even if clock fails; common in ASIC | Requires synchronous de-assertion to avoid metastability |
| Sync reset | Simpler timing analysis | Reset must be active during clock edge; needs wider pulse |
| Active-high | Intuitive naming | — |
| Active-low (`rst_n`) | Common in standard cells | Inversion in code can be confusing |

```verilog
// Best practice: async assert, sync de-assert (ASIC)
always @(posedge clk or negedge rst_n) begin
    if (!rst_n) q <= 0;     // async reset on negedge rst_n
    else        q <= d;
end
```

---

## 7. Synthesis Coding Checklist

- [ ] All `always @(*)` blocks have default assignments to prevent latches
- [ ] All `case` statements have a `default` branch
- [ ] No `initial` blocks in RTL (only in testbenches)
- [ ] No `#delay` in RTL code
- [ ] No `force`/`release` in RTL
- [ ] No `===`/`!==` in RTL
- [ ] No `forever`/`while` loops with dynamic termination in RTL
- [ ] `for` loops have static bounds
- [ ] Non-blocking `<=` used for all flip-flop assignments
- [ ] Blocking `=` used for combinational always blocks
- [ ] Never mix `=` and `<=` in the same always block
- [ ] Each `reg` driven from exactly ONE always block

---

## Common Synthesis Pitfalls

- **Simulation passes, synthesis fails**: Almost always due to using simulation-only constructs (`===`, `initial`, `#delay`, `fork`) in RTL
- **Simulation matches synthesis but wrong behavior**: Latch inferred where FF was intended — check sensitivity list and complete assignment
- **Tool-specific gotchas**: Some tools synthesize `while` if bounded; some treat `time` as integer — always consult tool reference
- **`defparam` deprecation**: Many modern tools warn or error on `defparam`; use `#()` parameter override at instantiation
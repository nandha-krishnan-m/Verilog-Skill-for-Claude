---
name: verilog-hdl
description: >
  Comprehensive Verilog HDL skill covering syntax, semantics, RTL design, simulation, synthesis,
  digital design fundamentals, and verification constructs. Use this skill whenever the user
  asks about Verilog HDL code, modules, always/initial blocks, nets, registers, gate-level
  modeling, RTL coding, behavioral modeling, structural modeling, synthesis constructs,
  testbenches, simulation tasks, FSM design, combinational or sequential logic in Verilog,
  timing, delays, operators, compiler directives, or debugging Verilog. Also use for any
  question involving Verilog-based digital design such as adders, multiplexers, decoders,
  flip-flops, memories, FIFOs, or counters. Trigger this skill even for basic or review-level
  Verilog questions.
---

# Verilog HDL Skill

## Overview

This skill covers Verilog HDL from fundamentals through RTL synthesis, structured as a
teaching and reference resource. Content is drawn from:
- *Digital Design, 5th Ed* — Mano & Ciletti (digital logic foundations)
- *Verilog HDL* — Samir Palnitkar (comprehensive HDL coverage)
- *Quick Reference for Verilog HDL* — Rajeev Madhavan (syntax cheatsheet)
- *Verilog HDL* — supplementary HDL reference

**Teaching format (default for all Verilog topics):**
1. **Theory** — Concept explanation with digital design context
2. **Code** — Synthesizable or simulation Verilog with inline comments
3. **Common Mistakes & Interview Traps** — What trips people up
4. **Practice Exercise** — Hands-on coding challenge

---

## Reference Files

Load the relevant reference file based on the topic:

| Topic Area | Reference File |
|---|---|
| Data types, nets, registers, operators, expressions | `references/01_lexical_and_datatypes.md` |
| Module structure, hierarchy, port connections | `references/02_modules_and_hierarchy.md` |
| Behavioral modeling: always, initial, tasks, functions | `references/03_behavioral_modeling.md` |
| Combinational and sequential RTL coding patterns | `references/04_rtl_coding_patterns.md` |
| Synthesis constructs — what's supported/ignored | `references/05_synthesis.md` |
| Testbench writing, simulation tasks, verification | `references/06_testbench_and_simulation.md` |
| Gate-level modeling, UDP, structural design | `references/07_gate_level_and_udp.md` |
| Timing, delays, specify blocks | `references/08_timing_and_delays.md` |
| Digital design foundations (combinational logic) | `references/09_digital_design_combinational.md` |
| Digital design foundations (sequential logic, FSMs) | `references/10_digital_design_sequential.md` |

**When to load multiple files:** For questions spanning RTL + synthesis, load both
`04_rtl_coding_patterns.md` and `05_synthesis.md`. For testbench debugging, load
`03_behavioral_modeling.md` and `06_testbench_and_simulation.md`.

---

## Quick Reference — Most-Used Constructs

### Module Template (RTL)
```verilog
module module_name #(parameter WIDTH = 8) (
    input  wire             clk,
    input  wire             rst_n,      // active-low async reset
    input  wire [WIDTH-1:0] data_in,
    output reg  [WIDTH-1:0] data_out
);
    // Internal signals
    wire [WIDTH-1:0] internal_sig;

    // Combinational logic
    assign internal_sig = data_in & {WIDTH{1'b1}};

    // Sequential logic
    always @(posedge clk or negedge rst_n) begin
        if (!rst_n)
            data_out <= {WIDTH{1'b0}};
        else
            data_out <= internal_sig;
    end

endmodule
```

### Key Rules
- **Blocking (`=`)** → Use in combinational `always` blocks
- **Non-blocking (`<=`)** → Use in sequential (clocked) `always` blocks — NEVER mix
- `wire` → driven by `assign` or module output; no storage
- `reg` → driven inside `always`/`initial`; does NOT always mean a physical register
- `===` / `!==` → 4-value comparison (includes X and Z); use in testbenches only
- `==` / `!=` → logic equality; X/Z operand returns X (unknown)

---

## Interview Traps — Top 10

1. **`reg` ≠ register**: A `reg` driven in a combinational `always` block infers combinational logic, not a flip-flop.
2. **Incomplete sensitivity list**: `always @(a)` missing `b` causes simulation/synthesis mismatch. Use `always @(*)` for combinational.
3. **Blocking vs Non-blocking in sequential**: Using `=` in a clocked `always` block creates race conditions.
4. **Latch inference**: An `if` without `else` in a combinational block infers a latch. Always have a default.
5. **Full case vs. parallel case**: Incomplete `case` without `default` → latch; `casex` with overlapping entries → simulation/synthesis mismatch.
6. **Integer overflow**: `reg [3:0] a = 4'd15; a = a + 1;` wraps to 0, not 16.
7. **`===` in synthesis**: `===` is unsupported in synthesis — simulation only.
8. **`initial` in synthesis**: `initial` blocks are ignored by most synthesis tools (except for FPGA RAM init).
9. **Task vs Function**: A function executes in zero simulation time and returns one value. A task can have delays and multiple outputs.
10. **Non-blocking scheduling**: `a <= b; b <= a;` swaps values correctly. `a = b; b = a;` does NOT swap.

---

## Coding Guidelines (Synthesis-Safe RTL)

- Always use `always @(*)` for combinational logic
- Always use `always @(posedge clk or posedge rst)` for sequential
- Never drive the same `reg` from two separate `always` blocks
- Use `parameter` and `localparam` for constants — never use magic numbers
- Prefer `assign` for simple combinational; use `always @(*)` for complex combinational
- Use non-blocking assignments (`<=`) for all flip-flop outputs
- Add `default:` to every `case` statement
- Declare port directions explicitly: `input wire`, `output reg`, `inout wire`

---

## How to Use This Skill

When a user asks a Verilog HDL question:

1. Identify the topic and load the relevant reference file(s) from the table above
2. Teach using the **four-section format**: Theory → Code → Mistakes & Traps → Exercise
3. For debugging questions: identify the bug category (synthesis mismatch, race condition, latch, etc.) and explain why it occurs before offering the fix
4. For code review: check against the synthesis guidelines and interview traps above
5. Always highlight synthesis implications when writing RTL code
6. For testbench questions: reference `06_testbench_and_simulation.md`
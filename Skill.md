---
name: verilog-hdl
description: >
  Verilog HDL skill focused on RTL design, synthesis-aware coding, debugging,
  and verification workflows. Use this skill for generating, reviewing, and
  debugging Verilog code, including modules, always blocks, combinational and
  sequential logic, FSMs, testbenches, and synthesis constructs. This skill
  prioritizes hardware-correct, synthesizable RTL and highlights design issues
  such as race conditions, latch inference, and simulation-synthesis mismatches.
---

# Verilog HDL Skill

## Overview

This skill provides a **design-oriented Verilog HDL environment** for RTL engineers,
covering the complete workflow from **code generation to debugging and synthesis validation**.

It is built as a **reference-backed engineering tool**, not just a learning resource,
ensuring all outputs are aligned with **real-world RTL design practices**.

---

## Response Framework (Default Behavior)

For any Verilog-related request, responses are structured as:

1. **Design Context** — Problem understanding and hardware intent  
2. **RTL Implementation** — Synthesizable Verilog with best practices  
3. **Design Checks & Pitfalls** — Issues like race conditions, latches, or mismatches  
4. **Optimization Notes (Optional)** — Improvements or alternative approaches  

---

## Reference Files

Load the relevant reference file based on the design context:

| Topic Area | Reference File |
|---|---|
| Data types, nets, registers, operators | `references/01_lexical_and_datatypes.md` |
| Module structure, hierarchy, ports | `references/02_modules_and_hierarchy.md` |
| Behavioral modeling: always, tasks, functions | `references/03_behavioral_modeling.md` |
| RTL coding patterns (combinational/sequential) | `references/04_rtl_coding_patterns.md` |
| Synthesis constraints and rules | `references/05_synthesis.md` |
| Testbench and simulation | `references/06_testbench_and_simulation.md` |
| Gate-level and structural design | `references/07_gate_level_and_udp.md` |
| Timing and delays | `references/08_timing_and_delays.md` |
| Combinational design fundamentals | `references/09_digital_design_combinational.md` |
| Sequential design and FSMs | `references/10_digital_design_sequential.md` |

**Multi-file usage:**
- RTL + synthesis → `04_rtl_coding_patterns.md` + `05_synthesis.md`
- Debugging testbench → `03_behavioral_modeling.md` + `06_testbench_and_simulation.md`

---

## Quick Reference — Core RTL Template

```verilog
module module_name #(parameter WIDTH = 8) (
    input  wire             clk,
    input  wire             rst_n,
    input  wire [WIDTH-1:0] data_in,
    output reg  [WIDTH-1:0] data_out
);

    wire [WIDTH-1:0] internal_sig;

    assign internal_sig = data_in;

    always @(posedge clk or negedge rst_n) begin
        if (!rst_n)
            data_out <= {WIDTH{1'b0}};
        else
            data_out <= internal_sig;
    end

endmodule
```
---

### Key Rules

- **Blocking (`=`)** → Use in combinational `always` blocks
- **Non-blocking (`<=`)** → Use in sequential (clocked) `always` blocks — NEVER mix
- `wire` → driven by `assign` or module output; no storage
- `reg` → driven inside `always`/`initial`; does NOT always mean a physical register
- `===` / `!==` → 4-value comparison (includes X and Z); use in testbenches only
- `==` / `!=` → logic equality; X/Z operand returns X (unknown)
---

## Design Constraints & Coding Standards (Synthesis-Safe RTL)

All generated RTL must comply with synthesis-safe and industry-standard practices:

- Use `always @(*)` for combinational logic to avoid sensitivity list issues
- Use `always @(posedge clk or posedge rst)` (or async reset as required) for sequential logic
- Use non-blocking assignments (`<=`) for all clocked logic
- Avoid mixing blocking and non-blocking assignments in the same design context
- Do not drive the same signal from multiple `always` blocks
- Ensure full condition coverage to prevent latch inference
- Include `default` in all `case` statements
- Prefer `assign` for simple combinational paths; use `always` for complex logic
- Use `parameter` / `localparam` for configurability
- Avoid non-synthesizable constructs unless explicitly writing testbench code

**Design Requirement:**  
All RTL must be **deterministic, synthesizable, and hardware-accurate**.

---

## RTL Design Workflow

This skill operates as an **RTL design and verification assistant**.

For any Verilog-related request:

1. **Identify Design Intent**
   - Combinational / Sequential / FSM / Testbench / Debugging

2. **Select Relevant Design Context**
   - Load appropriate reference file(s) based on the problem

3. **Generate RTL Implementation**
   - Produce synthesizable, parameterized Verilog code
   - Follow coding standards and best practices

4. **Perform Design Validation**
   - Check for:
     - Race conditions
     - Latch inference
     - Simulation vs synthesis mismatch
     - Timing-related concerns

5. **Provide Debug or Optimization Insights**
   - Suggest improvements in:
     - Area
     - Timing
     - Readability
     - Modularity

6. **Highlight Hardware Implications**
   - Explain how the RTL maps to actual hardware behavior

**Key Principle:**  
Always prioritize **correct hardware behavior over syntactic correctness**.

---

# Verilog HDL — Timing and Specify Blocks

## 1. `timescale` Directive

Defines the time unit and precision for the module:

```verilog
`timescale 1ns/1ps    // 1ns unit, 1ps precision
`timescale 1ns/100ps  // 1ns unit, 100ps precision
`timescale 1us/1ns    // microsecond-level simulation
```

- **Time unit**: Unit for all `#` delays in the module
- **Time precision**: Smallest resolvable time step (affects rounding)

```verilog
`timescale 1ns/1ps
#10;         // waits 10ns
#0.5;        // waits 500ps
#1.234;      // waits 1.234ns (rounded to 1ps precision)
```

**Important:** `timescale` must be declared before the module. Different modules can have different timescales — the simulator normalizes them.

---

## 2. Delay in RTL

**Note: All delays are IGNORED by synthesis tools.**

```verilog
// Intra-assignment delay — RHS evaluated at T, assigned at T+5
always @(posedge clk)
    q = #5 d;       // q gets d's value from time T, assigned at T+5

// Continuous assignment delay
assign #10 y = a & b;   // 10ns propagation delay (simulation only)

// Gate delay
and #(5, 3) g1 (out, a, b);   // rise=5ns, fall=3ns
```

---

## 3. Specify Blocks

Specify blocks are used for **path delay** and **timing check** specifications, primarily consumed by:
- Gate-level simulation (back-annotation with SDF)
- Static Timing Analysis (STA) tools
- NOT used by synthesis

```verilog
specify
    specparam tRISE = 2.5, tFALL = 2.0;    // like parameters, but not overridable

    // Simple path delay (input to output)
    (in1 => out1) = (tRISE, tFALL);         // rise delay, fall delay

    // Multiple inputs to one output
    (in1, in2 *> out1) = (2.5, 2.0);       // *> = multiple source path

    // Edge-sensitive path delay
    (posedge clk => (out1 +: in1)) = (tRISE, tFALL);
    // +: positive polarity; -: negative polarity

    // Conditional path delay
    if (OPCODE == 3'h4)
        (in1, in2 *> out1) = (3.0, 2.5);

    // Level-sensitive path
    if (clk)
        (in1 *> out1, out2) = 3.0;

    // Timing checks
    $setup(datain, posedge clk, 2.0);          // setup time check
    $hold(posedge clk, datain, 1.5);           // hold time check
    $setuphold(posedge clk &&& rst, datain &&& rst, 3:5:6, 2:3:6);
    $width(posedge clk, 5.0);                  // minimum pulse width

endspecify
```

---

## 4. Timing Check System Tasks

Used inside specify blocks for design rule checking:

| System Task | Purpose |
|-------------|---------|
| `$setup(data, edge, limit)` | Check setup time |
| `$hold(edge, data, limit)` | Check hold time |
| `$setuphold(edge, data, tsu, th)` | Combined setup + hold |
| `$width(edge, limit)` | Minimum pulse width |
| `$period(edge, limit)` | Minimum clock period |
| `$skew(ref_edge, data_edge, limit)` | Clock skew check |
| `$recovery(ref_edge, data, limit)` | Async reset recovery time |

---

## 5. `specparam` vs `parameter`

```verilog
specify
    specparam tPD = 5.0;    // path delay constant
    specparam tSU = 2.0;    // setup time
endspecify

// vs.
parameter WIDTH = 8;        // design parameter — overridable

// Key difference: specparam CANNOT be overridden via defparam or #()
// specparam only exists within specify block scope
```

---

## 6. SDF Back-Annotation

In gate-level simulation, actual timing from layout is back-annotated using Standard Delay Format (SDF) files:

```verilog
// In testbench
initial $sdf_annotate("timing.sdf", uut);
```

This overrides the specify block values with actual post-layout delays — used for timing simulation and sign-off.

---

## 7. Min:Typ:Max Delays

Three operating corners for timing analysis:

```verilog
// Gate delay: min:typ:max
nand #(4:5:6, 3:4:5) g1 (out, a, b);
// min=fast corner, typ=typical, max=slow corner

// Continuous assign
assign #(3:5:8) y = a & b;

// Specifying which corner to simulate (tool-specific)
// ModelSim: +maxdelays / +mindelays / +typdelays
```

---

## Common Timing Mistakes

- **Forgetting `timescale`**: Delays interpreted as simulation ticks (often 1 unit = 1 unspecified time)
- **Mismatch between modules**: `module A` has `timescale 1ns` and `module B` has `timescale 1ps` — potential precision issues; always declare consistently
- **Including delays in RTL**: `#5 q = d;` is ignored by synthesis but changes simulation behavior — can mask RTL bugs; keep delays only in testbenches
- **specparam scope**: `specparam` only valid inside `specify ... endspecify`; cannot be used in logic
- **SDF not applied**: Running gate-level sim without `$sdf_annotate` → zero-delay sim, which doesn't catch timing violations
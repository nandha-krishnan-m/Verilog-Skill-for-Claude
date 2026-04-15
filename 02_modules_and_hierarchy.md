# Verilog HDL — Modules and Hierarchy

## 1. Module Structure

A module is the fundamental unit of Verilog design. Nested module definitions are NOT allowed.

```verilog
module module_name #(
    parameter PARAM1 = default_val,
    parameter PARAM2 = PARAM1 + 1   // parameters can depend on each other
) (
    // Port declarations (Verilog-2001 ANSI style — preferred)
    input  wire             clk,
    input  wire             rst_n,
    input  wire [WIDTH-1:0] data_in,
    output reg  [WIDTH-1:0] data_out,
    inout  wire             bidir
);
    // Internal signal declarations
    wire [7:0] internal_wire;
    reg  [7:0] internal_reg;

    // Logic here

endmodule
```

**Old-style (pre-2001) port declaration — still common in legacy code:**
```verilog
module dff (q, qb, clk, d, rst);   // port list only
    input  clk, d, rst;
    output q, qb;
    reg    q, qb;
    // ...
endmodule
```

---

## 2. Module Instantiation

### Named Port Connection (Preferred — Order Independent)
```verilog
dff u_dff (
    .q   (out),
    .qb  (out_b),
    .clk (clk),
    .d   (data),
    .rst (reset)
);
```

### Positional Port Connection (Order Dependent — Legacy)
```verilog
dff u_dff (out, out_b, clk, data, reset);
```

### Overriding Parameters at Instantiation
```verilog
// Method 1: Using #() — ANSI style (preferred)
dff #(.DELAY(8), .WIDTH(16)) u_dff (...);

// Method 2: defparam — Old style (avoid in new designs)
defparam
    top.u_dff.DELAY = 8,
    top.u_dff.WIDTH = 16;
```

---

## 3. Parameters and Localparams

```verilog
parameter  DATA_WIDTH = 8;          // overridable from outside
localparam STATES     = 4;          // internal constant — NOT overridable

// Derived parameter
parameter ADDR_WIDTH = $clog2(DEPTH);   // Verilog-2005 system function
```

---

## 4. Hierarchy and Path References

```verilog
// Downward reference (from parent to child)
force top.u_dff.rst = 1;
release top.u_dff.rst;

// Upward path referencing
// generally avoided — tightly couples hierarchy
```

**`force` and `release`**: Procedural override for debugging — NOT synthesis-safe.

---

## 5. Generate Blocks (Verilog-2001)

Used to create parameterized structural repetition:

```verilog
genvar i;
generate
    for (i = 0; i < WIDTH; i = i + 1) begin : gen_ff
        dff u_ff (
            .d   (data_in[i]),
            .q   (data_out[i]),
            .clk (clk),
            .rst (rst)
        );
    end
endgenerate
```

---

## 6. Top-Level Module

A module with no ports or an empty port list is typically a top-level testbench:

```verilog
module tb_top;   // no port list — top of simulation hierarchy
    // Clock generation, DUT instantiation, stimulus
endmodule
```

---

## 7. Port Types Summary

| Direction | Used For | Inside module driven by |
|-----------|----------|------------------------|
| `input`   | Data/control in | External logic (wire inside) |
| `output`  | Data/control out | `assign` or `always` inside |
| `inout`   | Bidirectional bus | `assign` with tristate control |

**Inout/tristate example:**
```verilog
inout wire [7:0] bus;
reg [7:0] drive_data;
reg       bus_enable;

assign bus = bus_enable ? drive_data : 8'bz;
```

---

## Common Mistakes

- **Two modules with same name**: Causes linker error — every module name must be unique
- **Connecting wrong widths**: `wire [7:0]` connected to a `wire [3:0]` port — Verilog silently truncates; always check port width match
- **`defparam` in synthesis**: Avoid — many tools flag it; prefer `#()` instantiation override
- **Missing port connections**: Unconnected `input` ports default to Z; unconnected `output` ports are silently ignored (but create warnings)
- **Upward path references in synthesis**: `force`/`release` with hierarchy paths are simulation-only constructs
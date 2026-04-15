# Verilog HDL — Lexical Elements, Data Types, Nets, Registers, Operators

## 1. Lexical Elements

- **Case sensitive** — all keywords are lowercase
- **Comments**: `// single line` and `/* multi-line */`
- **Identifiers**: start with letter or `_`, followed by alphanumeric or `_`
- **Escaped identifiers**: start with `\`, end at first whitespace — allow special chars

### Integer Literals

| Format | Example | Meaning |
|--------|---------|---------|
| `n'bXXX` | `4'b1010` | 4-bit binary |
| `n'oXXX` | `3'o7` | 3-bit octal |
| `n'dXXX` | `8'd255` | 8-bit decimal |
| `n'hXXX` | `8'hFF` | 8-bit hexadecimal |
| Unsized | `9` or `'d9` | Integer |
| With underscore | `24_000` | Readability separator |

**Logic values:** `0`, `1`, `x`/`X` (unknown/uninitialized), `z`/`Z` (high impedance)

### Real and Special Types
```verilog
real    a, b;          // floating point
integer i, j;          // signed 32-bit
time    t;             // unsigned 64-bit (like integer but for time)
reg [8*14:1] str;      // string stored in wide vector
```

---

## 2. Net Types

Nets represent physical wires — **no storage**, must be continuously driven.

| Net Type | Description |
|----------|-------------|
| `wire` | Standard interconnect — most commonly used |
| `tri` | Same as wire, used for multiple drivers (tristate) |
| `wand` / `triand` | Wired-AND |
| `wor` / `trior` | Wired-OR |
| `tri0` | Undriven → pulls to 0 |
| `tri1` | Undriven → pulls to 1 |
| `trireg` | Retains last driven value when all drivers are Z (capacitive) |
| `supply0` | Tied to GND |
| `supply1` | Tied to VCC |

```verilog
wire net1;                  // 1-bit wire
wire [7:0] databus;         // 8-bit bus, MSB=7 LSB=0
tri [3:0] addr;             // 4-bit tristate
wand net2;                  // wired-AND
trireg (medium) cap;        // medium-strength capacitive net
supply0 gnd;
supply1 vcc;
```

**Wire resolution (multiple drivers):**
- Same value → resolves to that value
- One driver Z, others same → resolves to non-Z
- Conflicting strengths → stronger wins
- Equal strength, different values → `x`

---

## 3. Register Types

Registers store values between assignments. Used in `always`, `initial`, `task`, `function`.

```verilog
reg [5:0]  din;             // 6-bit vector register
reg [15:0] mem [0:511];     // 16-bit × 512 memory array
reg        flag;            // 1-bit register
integer    count;           // signed 32-bit (synthesis: use with care)
```

**`scalared` vs `vectored`:**
```verilog
wire scalared [5:0] neta;   // allows bit/part-select access
wire vectored [5:0] netb;   // only collective access
```

---

## 4. Operators

### Precedence (Highest to Lowest)
```
+, -, !, ~         (unary)
*, /, %
+, -               (binary)
<<, >>
<, <=, >, >=
==, !=
===, !==
&, ~&
^, ~^
|, ~|
&&
||
?:                 (ternary, right-associative)
```

### Arithmetic
```verilog
c = a * b;       // multiply
c = a / b;       // integer divide (truncates toward zero)
c = a % b;       // modulo
c = a + b;       // add
c = a - b;       // subtract
```

### Relational (return 1-bit true/false)
```verilog
a < b;  a > b;  a <= b;  a >= b;
```

### Equality
```verilog
a == b;    // logic equality — X/Z operand yields X
a != b;    // logic inequality
a === b;   // 4-value identity (includes X, Z) — SIMULATION ONLY
a !== b;   // 4-value non-identity — SIMULATION ONLY
```

### Logical (return 1-bit)
```verilog
a && b;   // AND
a || b;   // OR
!a;       // NOT
```

### Bitwise (per-bit operation)
```verilog
~a;       // bitwise NOT
a & b;    // bitwise AND
a | b;    // bitwise OR
a ^ b;    // bitwise XOR
a ~^ b;   // bitwise XNOR (also a ^~ b)
a ~& b;   // bitwise NAND
a ~| b;   // bitwise NOR
```

### Reduction (single-bit result from vector)
```verilog
&a;       // AND all bits
|a;       // OR all bits
^a;       // XOR all bits (odd-parity check)
~&a;      // NAND all bits
~|a;      // NOR all bits
~^a;      // XNOR all bits (even-parity check)
```

### Shift
```verilog
a << 1;   // logical shift left (fills with 0)
a >> 1;   // logical shift right (fills with 0)
// Note: Verilog-2001 adds <<< and >>> (arithmetic shift)
```

### Concatenation and Replication
```verilog
{co, sum} = a + b + ci;     // concatenation
b = {3{a}};                 // replicate a 3 times → {a, a, a}
{4{2'b01}}                  // → 8'b01010101
```

### Ternary
```verilog
c = sel ? a : b;    // if sel=1: c=a, else c=b
```

---

## 5. Compiler Directives

```verilog
`define OPCODEADD 5'b00010    // text macro substitution
`timescale 1ns/10ps           // time unit / precision
`include "myfile.v"           // file inclusion
`ifdef SYNTH                  // conditional compilation
  // synthesis-specific code
`endif
`resetall                     // reset all directives to default
`default_nettype wire         // default net type when undeclared
```

---

## Common Mistakes

- **Unsized literals in wide buses**: `a = 1` in a 32-bit context is fine, but `a = 'bx` is only 1 bit — use `{WIDTH{1'bx}}` instead
- **`===` in synthesis**: Will cause synthesis error or be flagged — always limit to testbench
- **Forgetting `'timescale`**: Simulation delays won't behave as expected without it
- **Part-select out of range**: `a[9:0]` on a 4-bit reg gives X in simulation, not an error
- **`x` propagation**: Uninitialized regs start as X — always initialize in reset logic
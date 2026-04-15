# Verilog HDL — Gate-Level Modeling and UDP

## 1. Built-in Gate Primitives

Verilog provides built-in gate primitives for structural modeling.

### Basic Gates
```verilog
// Syntax: gate_type [drive_strength] [#delay] instance_name (out, in1, in2, ...);

and  g1 (out, a, b);            // 2-input AND
and  g2 (out, a, b, c);         // 3-input AND (any number of inputs)
nand g3 (out, a, b);
or   g4 (out, a, b);
nor  g5 (out, a, b);
xor  g6 (out, a, b);
xnor g7 (out, a, b);
buf  g8 (out, in);              // buffer (can have multiple outputs)
not  g9 (out, in);              // inverter

// buf/not with multiple outputs
buf g10 (out1, out2, in);       // one input, two outputs
```

### Three-State (Tristate) Gates
```verilog
// bufif1: passes input when control=1, else Z
bufif1 ts1 (out, in, ctrl);     // enable-high tristate buffer
bufif0 ts2 (out, in, ctrl);     // enable-low tristate buffer
notif1 ts3 (out, in, ctrl);     // inverted, enable-high
notif0 ts4 (out, in, ctrl);     // inverted, enable-low
```

### MOS Switches (Switch-Level — Rarely Used in RTL)
```verilog
nmos  sw1 (out, data, ncontrol);    // NMOS pass gate
pmos  sw2 (out, data, pcontrol);    // PMOS pass gate
cmos  sw3 (out, data, nctrl, pctrl); // CMOS pass gate
```

### Bidirectional Switches
```verilog
tran     t1 (inout1, inout2);
tranif1  t2 (inout1, inout2, control);
tranif0  t3 (inout1, inout2, control);
```

### Pull Devices
```verilog
pullup   (net1);     // pulls net to logic 1
pulldown (net2);     // pulls net to logic 0
```

---

## 2. Gate Delays

```verilog
// Single delay (all transitions)
and #10 g1 (out, a, b);

// Two delays: rise, fall
and #(6, 8) g2 (out, a, b);

// Three delays: rise, fall, turn-off (for tristate)
bufif1 #(5, 7, 3) ts1 (out, in, ctrl);

// Min:Typ:Max delays
nand #(6:7:8, 5:6:7) g3 (out, a, b);
nand #(6:7:8, 5:6:7, 12:16:19) g4 (out, a, b);  // rise:fall:turnoff
```

### trireg Decay Delay
```verilog
trireg (large) #(0, 1, 9) cap;  // tr=0, tf=1, tdecay=9
```

---

## 3. Drive Strength

Strength levels (highest to lowest):

| Strength | Logic 1 | Logic 0 |
|----------|---------|---------|
| 7 | `supply1` | `supply0` |
| 6 | `strong1` | `strong0` |
| 5 | `pull1` | `pull0` |
| 4 | `large1` | `large0` |
| 3 | `weak1` | `weak0` |
| 2 | `medium1` | `medium0` |
| 1 | `small1` | `small0` |
| 0 | `highz1` | `highz0` |

```verilog
// Gate with drive strength and delay
nor (highz1, strong0) #(2:3:5) g1 (out, a, b);

// Continuous assign with strength
assign (weak1, strong0) y = a & b;
```

---

## 4. Gate-Level Instantiation Example

```verilog
// Full adder using gate primitives
module full_adder (sum, cout, a, b, cin);
    output sum, cout;
    input  a, b, cin;

    wire w1, w2, w3;

    xor g1 (w1,  a,   b);
    xor g2 (sum, w1,  cin);
    and g3 (w2,  a,   b);
    and g4 (w3,  w1,  cin);
    or  g5 (cout, w2, w3);
endmodule
```

---

## 5. User-Defined Primitives (UDP)

UDPs extend gate primitives using truth tables. Limited to:
- 1 output (must be listed first in port list)
- Any number of inputs
- Sequential UDPs: output is `reg` type, can be initialized

### Combinational UDP
```verilog
// 2:1 Multiplexer
primitive mux2 (out, sel, a, b);
    output out;
    input  sel, a, b;
    table
    //  sel  a   b  : out
        0    0   ?  : 0;
        0    1   ?  : 1;
        1    ?   0  : 0;
        1    ?   1  : 1;
        x    0   0  : 0;
        x    1   1  : 1;
    endtable
endprimitive
```

### Sequential Level-Sensitive UDP (Latch)
```verilog
primitive latch (q, clk, rst, d);
    output q;
    input  clk, rst, d;
    reg    q;
    initial q = 1'b1;       // initialization (simulation only)
    table
    //  clk  rst  d  :  q  : q+
        ?    1    ?  :  ?  : 0;     // reset overrides
        1    0    0  :  ?  : 0;     // transparent: d→q when clk=1
        1    0    1  :  ?  : 1;
        0    0    ?  :  ?  : -;     // hold when clk=0 (- = unchanged)
    endtable
endprimitive
```

### Sequential Edge-Sensitive UDP (D Flip-Flop)
```verilog
primitive dff (q, d, clk, rst);
    output q;
    input  d, clk, rst;
    reg    q;
    table
    //  d    clk    rst  :  q  : q+
        1   (01)    0    :  ?  : 1;     // clocked D=1
        0   (01)    0    :  ?  : 0;     // clocked D=0
        ?    ?      1    :  ?  : 0;     // async reset
        ?   (x1)    0    :  1  : 1;     // clock glitch — hold if consistent
        ?   (x1)    0    :  0  : 0;
        ?    n      0    :  ?  : -;     // negedge — hold
        *    ?      ?    :  ?  : -;     // input change when no posedge — hold
    endtable
endprimitive
```

### UDP Table Abbreviations
| Abbreviation | Meaning |
|---|---|
| `?` | Don't care (0, 1, or X) |
| `b` | Binary (0 or 1 only, no X) |
| `-` | Output unchanged |
| `(01)` or `r`/`R` | Rising edge (0→1) |
| `(10)` or `f`/`F` | Falling edge (1→0) |
| `p`/`P` | Positive edge (0→1, 0→X, X→1) |
| `n`/`N` | Negative edge (1→0, 1→X, X→0) |
| `*` or `(??)` | Any transition |

---

## Common Gate-Level Mistakes

- **Output must be first in UDP port list**: `primitive foo (output, input1, input2)` — output always first
- **UDP output is reg for sequential**: Forgetting `reg q;` in sequential UDP → compile error
- **Gate output count**: `buf` and `not` can have multiple outputs; other gates have exactly one
- **Drive strength + delay syntax**: Strength comes before delay: `nor (highz1, strong0) #5 g1 (...)`
- **Z in UDP input**: Z is treated as X inside UDPs — cannot match Z explicitly in table
- **`===` vs table matching**: UDP tables use 4-value matching differently — understand before mixing with behavioral
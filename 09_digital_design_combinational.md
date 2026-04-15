# Digital Design — Combinational Logic Foundations

*Based on: Digital Design, 5th Ed — Mano & Ciletti*

## 1. Boolean Algebra Essentials

### Fundamental Laws
```
Identity:      A + 0 = A        A · 1 = A
Null:          A + 1 = 1        A · 0 = 0
Idempotent:    A + A = A        A · A = A
Complement:    A + A' = 1       A · A' = 0
Involution:    (A')' = A
Commutative:   A + B = B + A    A · B = B · A
Associative:   A+(B+C) = (A+B)+C
Distributive:  A(B+C) = AB + AC
               A+BC = (A+B)(A+C)
De Morgan's:   (A+B)' = A'·B'
               (A·B)' = A'+B'
Absorption:    A + AB = A       A(A+B) = A
Consensus:     AB + A'C + BC = AB + A'C  (BC is redundant)
```

### SOP and POS Forms
```verilog
// Sum of Products (SOP) — OR of AND terms
// F = A'B + AB' + AB
assign F_SOP = (~A & B) | (A & ~B) | (A & B);

// Product of Sums (POS) — AND of OR terms
// F = (A+B)(A+B')
assign F_POS = (A | B) & (A | ~B);
```

---

## 2. Karnaugh Maps (K-Map) Simplification

**Rules:**
- Group cells containing 1s in powers of 2 (1, 2, 4, 8)
- Groups can wrap around edges
- Larger groups → simpler expression
- Cover all 1s with minimum groups

**2-variable K-Map:**
```
     B=0  B=1
A=0 |  0 |  1 |     F = A'B + AB = B (B is the simplified result)
A=1 |  1 |  1 |
```

**4-variable K-Map (16 cells):**
```
        CD=00  CD=01  CD=11  CD=10
AB=00 |   0  |   0  |   1  |   1  |
AB=01 |   0  |   0  |   1  |   1  |
AB=11 |   1  |   1  |   1  |   1  |
AB=10 |   1  |   1  |   1  |   1  |
Group of 8: F = A (A covers the entire bottom half)
```

---

## 3. Combinational Building Blocks

### Half Adder
```verilog
// Truth table: A B | Sum Cout
//              0 0 |  0   0
//              0 1 |  1   0
//              1 0 |  1   0
//              1 1 |  0   1
module half_adder (input a, b, output sum, cout);
    assign sum  = a ^ b;
    assign cout = a & b;
endmodule
```

### Full Adder
```verilog
module full_adder (input a, b, cin, output sum, cout);
    assign sum  = a ^ b ^ cin;
    assign cout = (a & b) | (b & cin) | (a & cin);
endmodule
```

### Ripple Carry Adder (4-bit)
```verilog
module rca4 (input [3:0] a, b, input cin, output [3:0] sum, output cout);
    wire c1, c2, c3;
    full_adder fa0 (.a(a[0]), .b(b[0]), .cin(cin),  .sum(sum[0]), .cout(c1));
    full_adder fa1 (.a(a[1]), .b(b[1]), .cin(c1),   .sum(sum[1]), .cout(c2));
    full_adder fa2 (.a(a[2]), .b(b[2]), .cin(c2),   .sum(sum[2]), .cout(c3));
    full_adder fa3 (.a(a[3]), .b(b[3]), .cin(c3),   .sum(sum[3]), .cout(cout));
endmodule
```

### Carry Lookahead Adder (CLA)
Key insight: Propagate (P) and Generate (G):
```verilog
// P_i = A_i XOR B_i  (propagates carry from below)
// G_i = A_i AND B_i  (generates carry regardless)
// C_i = G_i + P_i * C_{i-1}
assign P = A ^ B;
assign G = A & B;
assign C1 = G[0] | (P[0] & Cin);
assign C2 = G[1] | (P[1] & G[0]) | (P[1] & P[0] & Cin);
// ... and so on — all carries computed in parallel
```

### Comparator
```verilog
module comparator #(parameter N=4) (
    input  [N-1:0] a, b,
    output         eq, gt, lt
);
    assign eq = (a == b);
    assign gt = (a > b);
    assign lt = (a < b);
endmodule
```

### Encoder / Decoder

**2-to-4 Decoder:**
```verilog
module dec2to4 (input [1:0] sel, input en, output reg [3:0] y);
    always @(*) begin
        y = 4'b0000;
        if (en)
            y = 4'b0001 << sel;     // shift: clean one-hot output
    end
endmodule
```

**8-to-3 Priority Encoder:**
```verilog
module enc8to3 (input [7:0] in, output reg [2:0] out, output valid);
    always @(*) begin
        casez (in)
            8'b1???????: out = 3'd7;
            8'b01??????: out = 3'd6;
            8'b001?????: out = 3'd5;
            8'b0001????: out = 3'd4;
            8'b00001???: out = 3'd3;
            8'b000001??: out = 3'd2;
            8'b0000001?: out = 3'd1;
            8'b00000001: out = 3'd0;
            default:     out = 3'd0;
        endcase
    end
    assign valid = |in;     // valid if any input is 1
endmodule
```

### Multiplexer
```verilog
// Parameterized MUX using generate
module mux4 #(parameter W=8) (
    input  [W-1:0] a, b, c, d,
    input  [1:0]   sel,
    output reg [W-1:0] out
);
    always @(*) begin
        case (sel)
            2'b00: out = a;
            2'b01: out = b;
            2'b10: out = c;
            2'b11: out = d;
        endcase
    end
endmodule
```

---

## 4. Hazards in Combinational Logic

**Static-1 hazard**: Output should stay 1 but briefly glitches to 0 during input transition.
**Static-0 hazard**: Output should stay 0 but briefly glitches to 1.
**Dynamic hazard**: Multiple transitions on an output that should switch only once.

```verilog
// Example: F = AB + A'C — has hazard when A transitions with B=C=1
// When A: 1→0 with B=1, C=1: AB goes 1→0, A'C goes 0→1 — glitch possible
// Fix: Add consensus term BC: F = AB + A'C + BC
assign F_hazard_free = (A & B) | (~A & C) | (B & C);
```

**In synchronous design:** Combinational hazards are generally not a problem because registers only capture stable values at clock edges.

---

## 5. Parity Generator/Checker

```verilog
// Even parity generator (XOR of all bits → parity bit)
assign parity = ^data_bus;      // reduction XOR

// Odd parity
assign parity_odd = ~(^data_bus);

// Parity checker (even parity check — should be 0)
assign error = ^{received_data, received_parity};
```

---

## Common Combinational Design Mistakes

- **Incomplete case** without default → latch in synthesis, X in simulation
- **Inconsistent signal width**: Adding 4-bit + 4-bit without capturing carry → 1-bit silent loss
- **Priority encoding with `case`**: Regular `case` doesn't imply priority — use `if-else` chain or `casez` for priority
- **Glitch-sensitive outputs**: Combinational outputs directly connected to async inputs — register them
- **K-map errors**: Missing wrap-around groups → non-minimal SOP expression
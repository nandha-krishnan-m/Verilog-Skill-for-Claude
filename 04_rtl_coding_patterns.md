# Verilog HDL — RTL Coding Patterns

## 1. Combinational Logic

### Basic Gates
```verilog
// Using assign (preferred for simple logic)
assign y_and  = a & b;
assign y_or   = a | b;
assign y_xor  = a ^ b;
assign y_not  = ~a;
assign y_nand = ~(a & b);

// Using always @(*) for complex combinational
always @(*) begin
    case ({a, b, cin})
        3'b000: {cout, sum} = 2'b00;
        3'b001: {cout, sum} = 2'b01;
        3'b010: {cout, sum} = 2'b01;
        3'b011: {cout, sum} = 2'b10;
        3'b100: {cout, sum} = 2'b01;
        3'b101: {cout, sum} = 2'b10;
        3'b110: {cout, sum} = 2'b10;
        3'b111: {cout, sum} = 2'b11;
        default: {cout, sum} = 2'bxx;
    endcase
end
```

### Multiplexer
```verilog
// 2:1 MUX
assign out = sel ? b : a;

// 4:1 MUX using case
always @(*) begin
    case (sel)
        2'b00: out = a;
        2'b01: out = b;
        2'b10: out = c;
        2'b11: out = d;
        default: out = {WIDTH{1'bx}};
    endcase
end
```

### Priority Encoder
```verilog
// casez-based priority encoder (8-to-3)
always @(*) begin
    casez (in)
        8'b1???????: {valid, out} = {1'b1, 3'd7};
        8'b01??????: {valid, out} = {1'b1, 3'd6};
        8'b001?????: {valid, out} = {1'b1, 3'd5};
        8'b0001????: {valid, out} = {1'b1, 3'd4};
        8'b00001???: {valid, out} = {1'b1, 3'd3};
        8'b000001??: {valid, out} = {1'b1, 3'd2};
        8'b0000001?: {valid, out} = {1'b1, 3'd1};
        8'b00000001: {valid, out} = {1'b1, 3'd0};
        default:     {valid, out} = {1'b0, 3'd0};
    endcase
end
```

### Decoder
```verilog
// 2-to-4 decoder
always @(*) begin
    out = 4'b0000;          // default — no latch
    if (en) begin
        case (sel)
            2'b00: out = 4'b0001;
            2'b01: out = 4'b0010;
            2'b10: out = 4'b0100;
            2'b11: out = 4'b1000;
        endcase
    end
end
```

---

## 2. Sequential Logic

### D Flip-Flop
```verilog
// Async reset (most common in ASIC)
always @(posedge clk or posedge rst) begin
    if (rst) q <= 1'b0;
    else     q <= d;
end

// Sync reset
always @(posedge clk) begin
    if (rst) q <= 1'b0;
    else     q <= d;
end

// Async reset + enable
always @(posedge clk or posedge rst) begin
    if (rst)       q <= 1'b0;
    else if (en)   q <= d;
end
```

### D Latch (Use Only When Intentional)
```verilog
always @(*) begin
    if (en) q = d;      // latch: holds value when en=0
end
```

### Register Bank
```verilog
module reg_bank #(parameter WIDTH=8, DEPTH=16) (
    input  wire             clk, rst,
    input  wire [$clog2(DEPTH)-1:0] wr_addr, rd_addr,
    input  wire [WIDTH-1:0] wr_data,
    input  wire             wr_en,
    output reg  [WIDTH-1:0] rd_data
);
    reg [WIDTH-1:0] mem [0:DEPTH-1];
    integer i;

    always @(posedge clk or posedge rst) begin
        if (rst) begin
            for (i = 0; i < DEPTH; i = i+1)
                mem[i] <= {WIDTH{1'b0}};
        end else if (wr_en) begin
            mem[wr_addr] <= wr_data;
        end
    end

    assign rd_data = mem[rd_addr];      // async read
endmodule
```

---

## 3. Counters

### Up Counter
```verilog
always @(posedge clk or posedge rst) begin
    if (rst)          count <= {WIDTH{1'b0}};
    else if (en)      count <= count + 1'b1;
end
```

### Up/Down Counter
```verilog
always @(posedge clk or posedge rst) begin
    if (rst)       count <= 0;
    else if (up)   count <= count + 1;
    else if (down) count <= count - 1;
end
```

### Modulo-N Counter
```verilog
always @(posedge clk or posedge rst) begin
    if (rst)
        count <= 0;
    else if (count == N-1)
        count <= 0;
    else
        count <= count + 1;
end
```

---

## 4. Shift Registers

```verilog
// SIPO — Serial In Parallel Out
always @(posedge clk) begin
    shift_reg <= {shift_reg[WIDTH-2:0], serial_in};
end

// PISO — Parallel In Serial Out
always @(posedge clk) begin
    if (load)   shift_reg <= parallel_in;
    else        shift_reg <= {shift_reg[WIDTH-2:0], 1'b0};  // shift out MSB
end
assign serial_out = shift_reg[WIDTH-1];
```

---

## 5. Finite State Machines (FSM)

### Two-Process FSM (Recommended Style)
```verilog
module fsm (
    input  wire clk, rst, go, in,
    output reg  out
);
    // State encoding
    localparam IDLE  = 2'b00,
               START = 2'b01,
               WORK  = 2'b10,
               DONE  = 2'b11;

    reg [1:0] state, next_state;

    // Process 1: State register (sequential)
    always @(posedge clk or posedge rst) begin
        if (rst) state <= IDLE;
        else     state <= next_state;
    end

    // Process 2: Next state + output (combinational)
    always @(*) begin
        next_state = state;     // default: stay in current state
        out = 1'b0;             // default output
        case (state)
            IDLE:  if (go)        next_state = START;
            START: begin
                   next_state = WORK;
                   out = 1'b1;
                   end
            WORK:  if (!in)       next_state = DONE;
            DONE:  next_state = IDLE;
            default: next_state = IDLE;
        endcase
    end
endmodule
```

### Three-Process FSM (Mealy Output Registered)
```verilog
// Process 1: state register
// Process 2: next state logic
// Process 3: output register (registered output — no glitch)
always @(posedge clk or posedge rst) begin
    if (rst) out_reg <= 1'b0;
    else     out_reg <= next_out;
end
```

**Moore vs. Mealy:**
| | Moore | Mealy |
|---|---|---|
| Output depends on | State only | State + Input |
| Output changes | Only at clock edge | Can change with input |
| States needed | More | Fewer |
| Glitch risk | Lower | Higher |

---

## 6. Synchronizer (CDC — Clock Domain Crossing)

```verilog
// Double-flop synchronizer (for single-bit signals)
always @(posedge clk_dest or posedge rst) begin
    if (rst) begin
        sync_ff1 <= 1'b0;
        sync_ff2 <= 1'b0;
    end else begin
        sync_ff1 <= async_signal;
        sync_ff2 <= sync_ff1;
    end
end
assign synced_signal = sync_ff2;
```

---

## 7. Adder / ALU

```verilog
// Ripple carry adder
assign {cout, sum} = a + b + cin;

// Simple ALU
always @(*) begin
    case (opcode)
        3'b000: result = a + b;
        3'b001: result = a - b;
        3'b010: result = a & b;
        3'b011: result = a | b;
        3'b100: result = a ^ b;
        3'b101: result = ~a;
        3'b110: result = a << 1;
        3'b111: result = a >> 1;
        default: result = {WIDTH{1'b0}};
    endcase
end
```

---

## Common RTL Mistakes

- **Latch inferred**: `always @(*)` with incomplete assignment (no `else`, no `default`) → latch. Fix: assign default at top of always block.
- **Reset missing signals**: Only registering some signals in the reset condition → others power up with X.
- **Width mismatch in arithmetic**: Adding `[3:0]` to `[3:0]` silently discards carry — use `[4:0]` to capture overflow.
- **FSM no default state**: Add `default: state <= IDLE;` to handle glitches/X states.
- **Registered vs unregistered output**: Mealy output can glitch; register it for clean timing.
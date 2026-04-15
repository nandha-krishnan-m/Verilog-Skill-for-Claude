# Digital Design — Sequential Logic and FSMs

*Based on: Digital Design, 5th Ed — Mano & Ciletti*

## 1. Sequential Circuit Fundamentals

**Key distinction from combinational:** Output depends on current **state** AND inputs.
State is stored in **memory elements** (flip-flops).

### SR Latch (NOR-based)
```verilog
module sr_latch_nor (input S, R, output Q, Qn);
    // NOR-based SR latch — cross-coupled
    assign Q  = ~(R | Qn);
    assign Qn = ~(S | Q);
    // State: S=0,R=0 → hold  |  S=1,R=0 → set
    //        S=0,R=1 → reset |  S=1,R=1 → FORBIDDEN
endmodule
```

### D Latch
```verilog
module d_latch (input D, en, output reg Q);
    always @(*) begin
        if (en) Q = D;   // transparent when en=1
        // implicit else: Q holds — latch behavior
    end
endmodule
```

### D Flip-Flop (Edge-Triggered)
```verilog
module dff (
    input  wire D, clk, rst_n,
    output reg  Q
);
    always @(posedge clk or negedge rst_n)
        if (!rst_n) Q <= 1'b0;
        else        Q <= D;
endmodule
```

### JK Flip-Flop
```verilog
// JK: J=0,K=0 → hold | J=1,K=0 → set
//     J=0,K=1 → reset | J=1,K=1 → toggle
always @(posedge clk)
    case ({J, K})
        2'b00: Q <= Q;
        2'b01: Q <= 1'b0;
        2'b10: Q <= 1'b1;
        2'b11: Q <= ~Q;
    endcase
```

### T Flip-Flop
```verilog
// T=0 → hold | T=1 → toggle
always @(posedge clk or posedge rst)
    if (rst) Q <= 1'b0;
    else if (T) Q <= ~Q;
```

---

## 2. Flip-Flop Timing Parameters

| Parameter | Symbol | Definition |
|-----------|--------|------------|
| Setup time | tsu | Data must be stable BEFORE clock edge |
| Hold time | th | Data must be stable AFTER clock edge |
| Propagation delay | tpd | Clock edge to output change |
| Clock-to-Q | tCQ | Time from clock edge to valid output |

**Timing constraint:**
```
Minimum clock period: T ≥ tCQ + tlogic + tsu
```

**Metastability**: Occurs when data changes within the setup/hold window → output takes unpredictable time to resolve.

---

## 3. Registers and Counters

### Parallel Load Register
```verilog
module parallel_reg #(parameter N=8) (
    input  wire         clk, rst, load,
    input  wire [N-1:0] D,
    output reg  [N-1:0] Q
);
    always @(posedge clk or posedge rst)
        if (rst)       Q <= {N{1'b0}};
        else if (load) Q <= D;
endmodule
```

### BCD Counter (0–9)
```verilog
always @(posedge clk or posedge rst) begin
    if (rst)
        count <= 4'd0;
    else if (count == 4'd9)
        count <= 4'd0;
    else
        count <= count + 1;
end
assign tc = (count == 4'd9);   // terminal count
```

### Johnson Counter (Ring)
```verilog
// Twisted ring counter — 2N states for N flip-flops
always @(posedge clk or posedge rst) begin
    if (rst) Q <= 4'b0000;
    else     Q <= {~Q[0], Q[3:1]};   // shift with inverted MSB feedback
end
```

---

## 4. Finite State Machines

### State Encoding Choices

| Encoding | States | Bits | Pros | Cons |
|----------|--------|------|------|------|
| Binary | 8 | 3 | Compact, fewer FFs | More logic for decode |
| One-hot | 8 | 8 | Fast decode, easy | Many FFs |
| Gray code | 8 | 3 | Only 1 bit changes | Complex next-state |

```verilog
// One-hot encoding (FPGA preferred)
localparam S0 = 4'b0001, S1 = 4'b0010,
           S2 = 4'b0100, S3 = 4'b1000;

// Binary encoding (ASIC preferred for large FSMs)
localparam S0 = 2'b00, S1 = 2'b01,
           S2 = 2'b10, S3 = 2'b11;
```

### Moore FSM — Traffic Light Controller
```verilog
module traffic_light (
    input  wire clk, rst,
    input  wire timer,
    output reg  red, yellow, green
);
    localparam RED    = 2'b00,
               GREEN  = 2'b01,
               YELLOW = 2'b10;

    reg [1:0] state, next;

    // State register
    always @(posedge clk or posedge rst)
        state <= rst ? RED : next;

    // Next state logic
    always @(*) begin
        case (state)
            RED:    next = timer ? GREEN  : RED;
            GREEN:  next = timer ? YELLOW : GREEN;
            YELLOW: next = timer ? RED    : YELLOW;
            default: next = RED;
        endcase
    end

    // Output logic (Moore — depends only on state)
    always @(*) begin
        {red, yellow, green} = 3'b000;
        case (state)
            RED:    red    = 1'b1;
            GREEN:  green  = 1'b1;
            YELLOW: yellow = 1'b1;
        endcase
    end
endmodule
```

### Mealy FSM — Sequence Detector (101)
```verilog
module seq_det_101 (
    input  wire clk, rst, in,
    output reg  detect
);
    localparam IDLE = 2'b00, GOT1 = 2'b01, GOT10 = 2'b10;

    reg [1:0] state, next;

    always @(posedge clk or posedge rst)
        state <= rst ? IDLE : next;

    always @(*) begin
        next   = state;
        detect = 1'b0;      // Mealy: output computed combinationally
        case (state)
            IDLE:  next = in ? GOT1 : IDLE;
            GOT1:  next = in ? GOT1 : GOT10;
            GOT10: begin
                   next   = in ? GOT1 : IDLE;
                   detect = in;    // Mealy output: detect=1 when 101 seen
                   end
            default: next = IDLE;
        endcase
    end
endmodule
```

---

## 5. RAM and ROM

### Synchronous RAM (Dual-Port)
```verilog
module dpram #(parameter W=8, D=256) (
    input  wire            clk,
    // Port A: Write
    input  wire            wr_en,
    input  wire [$clog2(D)-1:0] wr_addr,
    input  wire [W-1:0]    wr_data,
    // Port B: Read
    input  wire [$clog2(D)-1:0] rd_addr,
    output reg  [W-1:0]    rd_data
);
    reg [W-1:0] mem [0:D-1];

    always @(posedge clk)
        if (wr_en) mem[wr_addr] <= wr_data;

    always @(posedge clk)
        rd_data <= mem[rd_addr];    // registered read (1-cycle latency)
endmodule
```

### ROM (Lookup Table)
```verilog
module sine_rom #(parameter N=8) (
    input  wire [N-1:0] addr,
    output reg  [N-1:0] data
);
    always @(*) begin
        case (addr)
            8'd0:   data = 8'd128;   // sin(0°) = 0 → centered at 128
            8'd32:  data = 8'd191;   // sin(45°)
            8'd64:  data = 8'd255;   // sin(90°) = max
            // ... fill table ...
            default: data = 8'd128;
        endcase
    end
endmodule
```

---

## 6. FIFO (Synchronous)

```verilog
module sync_fifo #(parameter W=8, D=16) (
    input  wire        clk, rst,
    input  wire        wr_en, rd_en,
    input  wire [W-1:0] wr_data,
    output reg  [W-1:0] rd_data,
    output wire        full, empty
);
    reg [W-1:0]          mem [0:D-1];
    reg [$clog2(D):0]    wr_ptr, rd_ptr, count;

    assign empty = (count == 0);
    assign full  = (count == D);

    always @(posedge clk or posedge rst) begin
        if (rst) begin
            wr_ptr <= 0; rd_ptr <= 0; count <= 0;
        end else begin
            if (wr_en && !full) begin
                mem[wr_ptr] <= wr_data;
                wr_ptr <= (wr_ptr == D-1) ? 0 : wr_ptr + 1;
                count  <= count + 1;
            end
            if (rd_en && !empty) begin
                rd_data <= mem[rd_ptr];
                rd_ptr <= (rd_ptr == D-1) ? 0 : rd_ptr + 1;
                count  <= count - 1;
            end
        end
    end
endmodule
```

---

## Common Sequential Design Mistakes

- **Setup/hold violation**: Applying data too close to clock edge in RTL simulation — use `#1` after posedge in testbench
- **Missing default in FSM**: FSM can get stuck in unreachable state from power-up X → always add `default: state <= IDLE`
- **One-hot encoding not checked**: If more than one bit is high, FSM behavior is unpredictable — add assertion or glitch protection
- **FIFO write and read same cycle with count**: Simultaneous wr+rd can have race — handle both simultaneously by not incrementing/decrementing count when both valid
- **Forgetting to initialize pointers in FIFO reset**: Partial reset leaves rd_ptr/wr_ptr in unknown state
# Verilog HDL — Behavioral Modeling

## 1. Procedural Blocks

### `always` Block
Runs continuously — re-triggers based on sensitivity list.

```verilog
// Combinational logic — use @(*) or @(all inputs)
always @(*) begin
    y = a & b;
end

// Sequential logic — edge-triggered
always @(posedge clk or posedge rst) begin
    if (rst)
        q <= 1'b0;
    else
        q <= d;
end

// Level-sensitive latch (usually unintentional — avoid unless needed)
always @(en or d) begin
    if (en) q = d;          // no else → latch inferred
end
```

### `initial` Block
Executes once at time 0. **Simulation only — NOT synthesized (except FPGA RAM init).**

```verilog
initial begin
    clk = 0;
    rst = 1;
    #20 rst = 0;
    #100 $finish;
end
```

---

## 2. Assignments

### Blocking Assignment (`=`)
Executes sequentially — each line completes before the next starts.
**Use in: combinational `always` blocks.**

```verilog
always @(*) begin
    a = b + c;      // a gets value immediately
    d = a * 2;      // uses updated a
end
```

### Non-Blocking Assignment (`<=`)
All RHS expressions evaluated first, then LHS updated simultaneously.
**Use in: sequential (clocked) `always` blocks.**

```verilog
always @(posedge clk) begin
    a <= b;     // swap
    b <= a;     // works correctly — a and b swap values
end
// After clock edge: a = old_b, b = old_a ✓

// WRONG with blocking:
always @(posedge clk) begin
    a = b;      // a gets b immediately
    b = a;      // b gets new a — NOT a swap!
end
```

### Continuous Assignment (`assign`)
For combinational logic on wire types. Updates whenever RHS changes.

```verilog
assign sum = a ^ b ^ cin;
assign cout = (a & b) | (b & cin) | (a & cin);
assign #10 net15 = enable;          // with propagation delay
assign (weak1, strong0) y = a & b;  // with drive strength
```

### `assign` / `deassign` (Procedural Continuous)
Override a register's value — mainly for simulation/debug:

```verilog
always @(rst)
    if (rst) assign q = 0;    // force q to 0
    else     deassign q;      // release q to normal logic
```

---

## 3. Conditional Statements

### if-else
```verilog
always @(*) begin
    if (!write_en)
        out = old_val;
    else if (!status)
        q = new_status;
    else
        out = new_val;
end
```

### case / casex / casez
```verilog
// case — exact match (including x and z)
always @(*) begin
    case (sel)
        2'b00: out = a;
        2'b01: out = b;
        2'b10: out = c;
        default: out = d;   // ALWAYS add default to avoid latches
    endcase
end

// casez — treats z as don't care
casez (state)
    3'b1??: fsm = 0;    // MSB=1, others don't care
    3'b01?: fsm = 1;
    default: fsm = 2;
endcase

// casex — treats both x and z as don't care (use carefully)
casex (opcode)
    4'b1xxx: alu_op = ADD;
    4'b01xx: alu_op = SUB;
    default: alu_op = NOP;
endcase
```

**Interview Trap:** `casex` can mask real X propagation bugs in simulation. Prefer `casez`.

---

## 4. Loops

```verilog
// forever — runs until $finish or disable
forever @(posedge clk) begin
    {co, sum} = a + b + ci;
end

// for — bounded iteration (synthesizable if bounds are static)
for (i = 0; i < 8; i = i + 1)
    mem[i] = 0;

// repeat — fixed count
repeat (8) begin
    @(posedge clk);
    count = count + 1;
end

// while — condition-based (synthesis: only if bounded and static)
while (delay > 0) begin
    @(posedge clk);
    delay = delay - 1;
end
```

---

## 5. Tasks

- Can have multiple inputs/outputs
- Can contain timing controls (`#`, `@`, `wait`)
- Does NOT return a value directly
- Simulation and synthesis (limited)

```verilog
task send_data;
    input  [7:0] data;
    output       valid;
    begin
        @(posedge clk);
        tx_data  = data;
        tx_valid = 1;
        @(posedge clk);
        tx_valid = 0;
        valid = 1;
    end
endtask

// Calling the task
always begin
    send_data(8'hAB, flag);
end
```

---

## 6. Functions

- Must have at least one input
- Returns exactly ONE value (function name = return value)
- Executes in zero simulation time (no delays, no `@`)
- Used in expressions

```verilog
function [1:0] next_state;
    input [1:0] current_state;
    input       go;
    begin
        case (current_state)
            2'b00: next_state = go ? 2'b01 : 2'b00;
            2'b01: next_state = 2'b10;
            2'b10: next_state = 2'b00;
            default: next_state = 2'b00;
        endcase
    end
endfunction

// Using function in an expression
assign ns = next_state(cs, go_signal);
```

---

## 7. Named Blocks and `disable`

```verilog
always begin: MAIN_BLOCK
    if (!queue_full) begin
        do_work;
    end else begin
        disable MAIN_BLOCK;     // exit this block
    end
end

initial forever @(posedge reset)
    disable MAIN_BLOCK;         // can disable from outside too
```

Named blocks can have local variables and support hierarchical path references.

---

## 8. `fork`/`join`

Launches concurrent processes — all run in parallel, block until all complete.

```verilog
initial begin
    fork
        @(posedge clk);         // waits for clk
        #100 rst = 0;           // simultaneously waits 100 time units
    join
    // continues only after BOTH complete
end
```

---

## Task vs. Function Summary

| Feature | Task | Function |
|---------|------|----------|
| Simulation time | Can consume time | Zero time only |
| Return value | None (uses outputs) | Returns one value |
| Inputs | Zero or more | At least one |
| Outputs | Yes, multiple | No (only return) |
| Delays/events | Allowed | NOT allowed |
| Calls tasks | Yes | No |
| Used in expressions | No | Yes |

---

## Common Mistakes

- **`always` without sensitivity list**: `always begin` runs forever at time 0 — infinite loop in simulation
- **Mixing blocking/non-blocking**: `q <= d; count = count + 1;` in the same always block → race condition
- **Missing `default` in `case`**: Synthesis infers latch; simulation produces X
- **Calling a task inside a function**: Not allowed — functions must be zero-time
- **`initial` for reset assumption in synthesis**: Use proper async/sync reset in `always @(posedge clk)` instead
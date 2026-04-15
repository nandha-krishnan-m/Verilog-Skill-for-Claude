# Verilog HDL — Testbench Writing and Simulation

## 1. Testbench Structure

```verilog
`timescale 1ns/1ps      // time unit / precision — always declare

module tb_dut;          // no ports — top-level simulation module

    // DUT port signals
    reg         clk, rst;
    reg  [7:0]  data_in;
    wire [7:0]  data_out;
    wire        valid;

    // Instantiate DUT (Device Under Test)
    my_module uut (
        .clk      (clk),
        .rst      (rst),
        .data_in  (data_in),
        .data_out (data_out),
        .valid    (valid)
    );

    // Clock generation
    parameter CLK_PERIOD = 10;  // 10ns → 100MHz
    always #(CLK_PERIOD/2) clk = ~clk;

    // Stimulus and checking
    initial begin
        // Initialize
        clk      = 0;
        rst      = 1;
        data_in  = 0;

        // Apply reset
        @(posedge clk); #1;
        @(posedge clk); #1;
        rst = 0;

        // Apply stimulus
        @(posedge clk); #1;
        data_in = 8'hAB;

        @(posedge clk); #1;
        data_in = 8'h55;

        // Wait and check
        @(posedge clk); #1;
        if (data_out !== 8'hAB)
            $display("FAIL: Expected AB, Got %h at time %0t", data_out, $time);
        else
            $display("PASS: data_out = %h", data_out);

        #100;
        $finish;
    end

    // Waveform dump
    initial begin
        $dumpfile("tb_dut.vcd");
        $dumpvars(0, tb_dut);
    end

endmodule
```

---

## 2. System Tasks and Functions

### Display and Monitoring
```verilog
$display("a=%b, b=%h, time=%0t", a, b, $time);  // print once
$write("no newline");                            // like $display without \n
$strobe("at end of timestep: q=%b", q);         // prints at END of time step

// Monitor — triggers every time any argument changes
$monitor($time, " a=%b clk=%b sum=%h", a, clk, sum);

// File output
integer fd;
fd = $fopen("output.txt");
$fdisplay(fd, "data=%h", data);
$fclose(fd);
```

### Format Specifiers
| Specifier | Meaning |
|-----------|---------|
| `%b` | Binary |
| `%o` | Octal |
| `%d` | Decimal |
| `%h` | Hexadecimal |
| `%s` | String |
| `%t` | Time |
| `%0t` | Time without leading spaces |

### Simulation Control
```verilog
$finish;        // exit simulator
$stop;          // pause simulator (allows interactive debug)
$time;          // current simulation time (integer)
$realtime;      // current simulation time (real)
```

### Memory Load
```verilog
$readmemb("stimulus.txt", mem);         // load binary data into memory
$readmemh("data.hex", mem);             // load hex data into memory
$readmemh("data.hex", mem, 0, 255);     // with start and end addresses
```

---

## 3. Timing Controls in Testbench

```verilog
#10;                        // delay 10 time units
@(posedge clk);             // wait for posedge of clk
@(negedge clk);             // wait for negedge of clk
@(a or b);                  // wait for any change on a or b
@(posedge clk or posedge rst);

wait(done == 1);            // wait until condition is true
```

**Best practice — apply stimulus slightly after clock edge:**
```verilog
@(posedge clk); #1;         // wait for edge, then 1ns setup
data_in = 8'hAB;            // now apply — avoids race condition with DUT
```

---

## 4. Self-Checking Testbench

```verilog
// Task to apply input and check output
task apply_and_check;
    input [7:0] in_val;
    input [7:0] expected;
    begin
        @(posedge clk); #1;
        data_in = in_val;
        @(posedge clk); #1;
        if (data_out !== expected) begin
            $display("FAIL @ time %0t: in=%h, expected=%h, got=%h",
                      $time, in_val, expected, data_out);
            $stop;
        end else begin
            $display("PASS: in=%h, out=%h", in_val, data_out);
        end
    end
endtask

// Use in stimulus
initial begin
    // ... reset ...
    apply_and_check(8'hAB, 8'hAB);
    apply_and_check(8'h00, 8'h00);
    apply_and_check(8'hFF, 8'hFF);
    $display("All tests passed!");
    $finish;
end
```

---

## 5. Waveform Viewing

### VCD Dump (Portable — works with GTKWave)
```verilog
initial begin
    $dumpfile("waves.vcd");
    $dumpvars(0, tb_module);    // 0 = all levels from tb_module
    $dumpvars(1, tb_module.uut); // 1 = one level deep in uut only
end
```

### ModelSim/QuestaSim Wave Logging
```tcl
# In ModelSim Tcl console:
add wave -r /*          # add all signals recursively
run 1000ns
```

---

## 6. Random Stimulus

```verilog
// $random — returns 32-bit random signed integer
reg [7:0] rand_data;
always begin
    @(posedge clk);
    rand_data = $random;        // 32-bit random, truncated to 8 bits
    rand_data = $random % 256;  // bounded to 0-255
end

// $urandom (Verilog-2005) — unsigned 32-bit random
rand_data = $urandom_range(0, 255);
```

---

## 7. Testbench for Common DUTs

### Clock Generator Patterns
```verilog
// 50% duty cycle
always #5 clk = ~clk;     // 10ns period

// Non-50% duty cycle (60%/40%)
always begin
    #6 clk = 1;
    #4 clk = 0;
end
```

### Reset Sequence
```verilog
initial begin
    clk = 0; rst = 1;
    repeat(5) @(posedge clk);   // hold reset for 5 cycles
    @(posedge clk); #1;
    rst = 0;
end
```

### FSM Stimulus Pattern
```verilog
task apply_stimulus;
    input go;
    input [7:0] data;
    begin
        @(posedge clk); #1;
        go_signal = go;
        data_in   = data;
        @(posedge clk); #1;
        go_signal = 0;
    end
endtask
```

---

## Common Testbench Mistakes

- **Race condition**: Applying stimulus exactly at clock edge (`@(posedge clk); data=x;`) — add `#1` after edge
- **Missing `$finish`**: Simulator runs forever — always end with `$finish`
- **`$monitor` overload**: Multiple `$monitor` calls — only the last one is active; use `$display` for multiple
- **Not initializing `clk`**: `always #5 clk = ~clk` with uninitialized clk starts as X — always `clk = 0` in initial
- **Using `===` for synthesis check**: `!==` and `===` are testbench-only — correct in TB but never in RTL
- **No reset in testbench**: DUT registers start as X, causing false failures in self-checking TB
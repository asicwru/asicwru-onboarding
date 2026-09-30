# Introduction to Hardware Description Language (HDL)

### What is a hardware description language?

A **hardware description language (HDL)** is a special type of programming language created specifically for describing and simulating the structure and behavior of digital logic circuits. Popular HDLs include Verilog, SystemVerilog, and VHDL.
Verilog has a C-like syntax (similar operators, if-else structures, etc.) but their underlying behavior is completely different. Verilog operates in parallel, which means multiple blocks of code can run at the same time like an electronic circuit.
On the other hand, C code executes the code sequentially.


SystemVerilog is an extension of Verilog with extra features such as more data types (e.g logic, struct, and enum) and advanced verification features. Syntax-wise, they're still the same so you shouldn't worry about missing out on it while learning Verilog.

--- 

### How do I learn Verilog?

To learn the basics of Verilog, you're recommended to check out [HDLBits](https://hdlbits.01xz.net) which provides over 150+ Verilog exercises from learning how to create an AND gate to shift registers. However, you won't need to go through all of the exercises there.
We've highlighted some of the exercises that are going to be relevant for the onboarding projects.

You can access all the problems from [here](https://hdlbits.01xz.net/wiki/Problem_sets).

| # | Topic | Problem(s) |
| --- | --- | --- |
| 1 | Getting Started |  Getting Started, Output Zero |
| 2 |  Basics | Simple wire, Four wires, Inverter, AND Gate, NOR Gate, Declaring wires |
| 3 | Vectors | Vectors, Vectors in more detail, Bitwise operators, Vector concatenation operator|
| 4 | Modules: Hierarchy | Modules, Three modules, Modules and vectors, Adder 1|
| 5 | Procedures | All exercises besides the priority encoder exercises |
| 6 | More Verilog Features | Conditional ternary operator |
| 7 | Combinational Logic | Basic Gates, Multiplexers, Adders, Karnaugh Map exercises |
| 8 | Sequential Logic | D Flip-Flops, Registers, Counters, Shift Registers |
| 9 |  Finite State Machines | Simple FSM exercises and FSM design |

<br>

> HDLBits primarily teaches you the ``wire`` and ``reg`` signal types. For the onboarding project which is in SystemVerilog, you can use the ``logic`` data type in place of either instead.

> In SystemVerilog, use ``always_comb`` instead of ``always`` for combinational logic. Similarly, use ``always_ff`` for sequential circuits to avoid confusion.

---

### Testbenches

A **testbench** is code used to test and verify that your Verilog/SystemVerilog module works correctly. Unlike your actual design, the testbench is not synthesized into hardware.

Instead, the testbench provides inputs to your module and checks its outputs.

For example, if you created an AND gate:

```systemverilog
module and_gate (
    input  logic a,
    input  logic b,
    output logic y
);

assign y = a & b;

endmodule
```

A simple testbench could look like:

```systemverilog
module and_gate_tb;

logic a;
logic b;
logic y;

and_gate dut (
    .a(a),
    .b(b),
    .y(y)
);

initial begin
    a = 0;
    b = 0;

    #10;
    a = 0;
    b = 1;

    #10;
    a = 1;
    b = 0;

    #10;
    a = 1;
    b = 1;

    #10;
    $finish;
end

endmodule
```

The module being tested is commonly called the **DUT (Design Under Test)**. The `#10` tells the simulator to wait 10 units of simulation time before moving on to the next input. The `$finish` statement ends the simulation. For larger projects, testbenches can automatically check outputs instead of requiring you to manually look at waveforms.

---

### Non-blocking vs. blocking statements

There are two types of assignments you will see in SystemVerilog. Generally, ``=`` is a **blocking** assignment used for combinational logic (inside `always_comb`) while ``<=`` is a **non-blocking** assignment used for sequential logic (inside `always_ff`). Don't mix them for the same signal.

Blocking assignments happen in order. Each line can see the result of the line before it while non-blocking assignments calculate their new values first, then update them together.

For example:

```systemverilog
always_ff @(posedge clk) begin
    q1 <= data;
    q2 <= q1;
end
```

On the clock edge, ``q1`` receives data while ``q2`` receives the previous value of q1.

---

### Waveforms from a testbench

Add these to your testbench's `initial` block to write a waveform file you can open in GTKWave:

```systemverilog
$dumpfile("and_gate.vcd");
$dumpvars(0, and_gate_tb);
```

Also, checking outputs automatically (`if (y !== expected) $error(...)`, using `!==` so unknown X values fail too) is better than reading the waveform yourself. See [04-verification-basics.md](04-verification-basics.md).

---

## Constructs you will need for the onboarding projects

HDLBits teaches syntax, but not everything the ALU, FIFO, and UART use. Read this before starting the projects.

### Parameters and `$clog2`

A `parameter` is a compile-time constant that can be overridden when the module is instantiated; a `localparam` cannot be overridden (use it for values derived from parameters). `$clog2(N)` is the number of bits needed to count 0..N-1.

```systemverilog
module counter #(
    parameter int MAX = 10               // override with: counter #(.MAX(100)) u_cnt (...)
) (
    input  logic clk,
    input  logic rst_n,
    output logic [$clog2(MAX)-1:0] count
);
    // ...
endmodule
```

### `case` is a mux, and an incomplete `case` makes a latch

```systemverilog
always_comb begin
    case (op)
        2'd0:    result = a + b;
        2'd1:    result = a - b;
        default: result = '0;      // REQUIRED: covers every value not listed above
    endcase
end
```

If some path doesn't assign `result`, the hardware must *remember* its old value, and remembering without a clock builds a **latch**. That is almost always a bug. The fix is to assign every output on every path, usually with a `default`. Verilator's lint (`verilator --lint-only -Wall file.sv`) reports these as `LATCH` or `CASEINCOMPLETE`. `'0` means "all zeros at whatever width is needed".

### Reset styles

```systemverilog
// synchronous reset: only checked on a clock edge (the FIFO uses this)
always_ff @(posedge clk) begin
    if (!rst_n) q <= '0;
    else        q <= d;
end

// asynchronous reset: takes effect immediately (the UART uses this)
always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) q <= '0;
    else        q <= d;
end
```

Both are active-low here (`rst_n`). Every flip-flop should have a reset value, or it starts as unknown (X) in simulation.

### Finite state machines

An `enum` gives the states names, and a `case` picks the behavior. This is the shape of the UART skeleton:

```systemverilog
typedef enum logic [1:0] {IDLE, RUN, DONE} state_t;
state_t state;

always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        state <= IDLE;
    end else begin
        case (state)
            IDLE:    if (start)    state <= RUN;
            RUN:     if (finished) state <= DONE;
            DONE:    state <= IDLE;
            default: state <= IDLE;
        endcase
    end
end
```

"Stay in this state for N cycles, then move on" is a counter: count up each cycle, and when it reaches N-1, reset it and change state.

### Memories and vector tricks

```systemverilog
logic [15:0] mem [8];        // 8 words of 16 bits
mem[wptr] <= din;            // write (inside always_ff)
dout <= mem[rptr];           // read

x[3:0]                       // low 4 bits of x
{a, b}                       // concatenate (a = upper bits)
{31'b0, flag}                // widen a 1-bit flag to 32 bits
```

### Widths and signed vs. unsigned

- A literal like `8'hFF` is 8 bits wide; a bare `1` is a **32-bit** integer. Write `1'b1` when you mean one bit. Comparing a 4-bit counter to a 32-bit constant gives a Verilator `WIDTHEXPAND` warning, and the cocotb build treats warnings as errors, so make the constant the same width as the signal.
- Vectors are **unsigned** by default. Use `$signed(...)` for signed behavior:

```systemverilog
a < b                        // unsigned compare
$signed(a) < $signed(b)      // signed compare
a >> n                       // logical right shift (zero fill)
$signed(a) >>> n             // arithmetic right shift (sign fill)
```

`>>>` only sign-fills if its left operand is signed.

### Style rules we follow

- `always_comb` for combinational logic, `always_ff` for flip-flops (no plain `always`)
- Every output assigned on every path; every flip-flop has a reset
- Connect ports by name: `.a(x)`, never by position
- Lint (`verilator --lint-only -Wall file.sv`) should be clean before you commit

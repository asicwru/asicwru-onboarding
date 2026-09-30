# Onboarding Projects

You will complete three onboarding modules. Write the RTL yourself without using AI to generate it. For each module, write the testbench, run the simulation, and open the resulting waveforms in GTKWave to check the behavior.

Each module folder has a skeleton with `TODO`s and a README with the exact interface and a "done when" checklist. Those READMEs are the source of truth for what your design must do. Do the modules in this order:

---

### 1. 32-bit ALU (`alu/`)

The **arithmetic logic unit (ALU)** takes two 32-bit operands, `a` and `b`, and a 4-bit control input `op` that selects the operation. Reference the [RV32I ISA](https://docs.riscv.org/reference/isa/v20260120/unpriv/rv32.html) and implement the ten basic operations: ADD, SUB, AND, OR, XOR, SLL, SRL, SRA, SLT, SLTU. Invalid opcodes produce 0.

Your SystemVerilog testbench must check each operation, including the difference between signed and unsigned comparisons and between logical and arithmetic right shifts, and include randomized inputs. It should exit with an error if any check fails.

You practice: combinational logic, `case` as a mux, avoiding latches, signed vs. unsigned.

---

### 2. Synchronous FIFO (`sync_fifo/`)

A **first-in, first-out (FIFO)** buffer stores data and returns it in the same order it was written. Implement a synchronous FIFO with one clock, read and write enables, and `full` and `empty` status signals. Pointers carry one extra bit so all `DEPTH` slots are usable. Writes when full and reads when empty are ignored.

Your cocotb testbench must check normal reads and writes, the order of stored values, full and empty behavior, simultaneous read/write, pointer wraparound, and random traffic against a reference model.

You practice: sequential logic, memories, pointers, reset, reference-model checking.

---

### 3. UART Transmitter (`uart_transmitter/`)

A **UART transmitter** sends data one bit at a time over a serial output. Implement a transmitter that sends a start bit, eight data bits (least significant bit first), and a stop bit, each lasting `CLKS_PER_BIT` clock cycles (derived from the clock frequency and baud rate parameters). A `tx_busy` output is high during the whole frame.

Your cocotb testbench must check the value and duration of every bit, the busy signal, back-to-back bytes, a start request while busy, and reset in the middle of a frame.

You practice: finite state machines, counters, protocol timing.

---

### Running a module

See each module's README for how to run its testbench. The FIFO and UART folders have a `Makefile`:

```bash
cd sync_fifo      # or uart_transmitter
make              # build and run the testbench, writes the waveform (.vcd)
make lint         # static checks on the RTL
make waves        # open the waveform in GTKWave
```

The FIFO and UART use cocotb 2.x, which needs Verilator 5.036 or newer.

### Submission

Send the link to your cloned GitHub repository (it must be public!) containing the `.sv` files, a testbench for each module (SystemVerilog for the ALU, cocotb for the others), and the `.vcd` waveform files from your simulations.

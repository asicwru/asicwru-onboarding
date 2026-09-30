# Introduction to Design Verification & Simulation

## Why verification matters

A design that "looks right" and a design that's *correct* are different things. On a real ASIC, a bug caught in simulation costs nothing. A bug caught after tapeout costs the team the whole chip and there's no patching silicon. Verification is how you close that gap before it's expensive. You drive known stimulus into a design and check that its outputs match what they should be, automatically, without a human eyeballing waveforms every time.

This is also why every ASICWRU onboarding project and flagship project ships with a testbench alongside the RTL. The RTL isn't "done" when it compiles; it is done when its testbench passes all cases.

## What a testbench actually is

A testbench is code that sits around your design (the **DUT**, device under test) and:

1. **Drives inputs** — applies known values to the DUT's input ports
2. **Waits** — lets the DUT react (usually: waits for a clock edge)
3. **Checks outputs** — compares what the DUT produced against what you expected
4. **Reports pass/fail** — automatically, with no human reading a waveform

```
        ┌─────────────┐
 stim → │             │ → outputs
        │     DUT     │
 clk  → │             │
        └─────────────┘
              ↑
        testbench drives inputs,
        checks outputs against
        an expected/reference model
```

A **self-checking testbench** is one that asserts pass/fail on its own. You run it and get a clear "9/9 tests passed" or "FAILED: expected 0x0A, got 0x0B at cycle 42," not a waveform you have to inspect by hand. This is the standard for every project. Waveform viewing (GTKWave) is for *debugging* a failure, not for verifying correctness in the first place.

## Directed vs. random testing

- **Directed tests** — you pick specific inputs to check specific behavior: known operations, boundary values (all-zeros, all-ones, max/min), and cases you know are tricky (e.g. a FIFO exactly full, exactly empty).
- **Randomized tests** — you throw many random inputs at the DUT and check each one against a reference model. This catches bugs you didn't think to test for directly. A few hundred random cycles will usually find a bug that ten hand-picked cases miss.

Good testbenches use both: directed tests for known edge cases, randomized tests for coverage you didn't think of. The ALU testbench is structured this way: a block of directed cases (including edge cases like `32'hFFFFFFFF + 1` and shift-by-32), a loop over the invalid opcodes, and a long randomized loop. Those are three different testing *strategies*, not just three chunks of code.

**Reproducibility:** a random test that fails must fail *the same way* next time, or you can't debug it. Use a seeded generator (`random.Random(1234)` in Python) so runs are identical. Verilator's `$urandom` is also repeatable from run to run.

## Two testbench styles in this repo

- **SystemVerilog testbench (ALU):** simulation-only SystemVerilog (`initial`, `task`, `$urandom`). It counts failures and calls `$fatal` at the end if there were any, so the simulation exits with an error when the ALU is wrong.
- **cocotb testbench (FIFO, UART):** Python, described below. A failing `assert` makes the test (and `make`) fail.

Either way, a testbench that only prints values is not a test: it has to *decide* pass or fail itself.

## cocotb quickstart

ASICWRU testbenches are written in **cocotb** - a Python-based verification framework that drives a Verilog/SystemVerilog DUT running under a simulator (we use Verilator). You write ordinary Python; cocotb handles the simulator handshake.

### The timing pattern to use: drive and read on the falling edge

Registers update just after the rising clock edge. If your testbench also acts at the rising edge, it races with the DUT (do you see the old or the new value?). To avoid guessing, every ASICWRU cocotb testbench follows this convention:

1. **Drive** inputs right after a **falling** edge
2. The DUT **samples** them at the next **rising** edge
3. **Read** outputs at the following **falling** edge, when everything has settled

### A complete example

A 4-bit counter with a synchronous reset and an enable, tested against a one-line reference model:

```python
import cocotb
from cocotb.clock import Clock
from cocotb.triggers import FallingEdge, RisingEdge


@cocotb.test()
async def test_counts_when_enabled(dut):
    # start a 10 ns clock in the background
    cocotb.start_soon(Clock(dut.clk, 10, unit="ns").start())

    # reset: drive on falling edges, hold for a couple of cycles
    dut.rst_n.value = 0
    dut.en.value = 0
    await FallingEdge(dut.clk)
    await FallingEdge(dut.clk)
    dut.rst_n.value = 1

    expected = 0                            # the reference model
    for cycle in range(40):
        en = cycle % 3 != 0                 # some cycles enabled, some not
        dut.en.value = en                   # 1) drive just after a falling edge
        await RisingEdge(dut.clk)           # 2) the DUT samples at the rising edge
        await FallingEdge(dut.clk)          # 3) outputs have settled: now read
        if en:
            expected = (expected + 1) % 16  # model: 4-bit wraparound
        assert dut.count.value == expected, f"cycle {cycle}: count={dut.count.value}, expected {expected}"
```

Things to know:

- `@cocotb.test()` marks an `async` function as a test case; a testbench file can (and should) have several
- `dut.<signal>.value = x` drives a signal; `dut.<signal>.value` reads one
- `await RisingEdge(dut.clk)` / `FallingEdge` / `ClockCycles(dut.clk, n)` are how you advance simulated time — nothing happens in the DUT until you `await` something
- For a *combinational* design with no clock, use `from cocotb.triggers import Timer` and `await Timer(1, unit="ns")` after changing inputs to let the logic settle
- `cocotb.start_soon(...)` runs a coroutine in the background; that's how the clock keeps ticking while your test does other things
- This repo uses **cocotb 2.x**, where the time argument is spelled `unit="ns"` (cocotb 1.x used `units=`). Old tutorials online may use the old spelling
- Everything is Python, so build reference models in plain Python (e.g. `expected = a + b`) and assert the DUT matches — this reference model *is* your "known-correct" answer
- Assertions that fail print immediately with the values involved, which is why cocotb tests are self-checking by default — no separate pass/fail scoring step needed

## Running the tests

The FIFO and UART folders each have a `Makefile` that wires Verilator and cocotb together (the ALU README explains how to run its SystemVerilog testbench):

```bash
make        # compiles the RTL and runs every test; writes the waveform (.vcd)
make lint   # static checks on your RTL: latches, width mismatches, unused signals
make waves  # opens the waveform in GTKWave
make clean  # wipes build artifacts for a clean re-run
```

Output is a per-test pass/fail summary. If something fails, cocotb prints the assertion that tripped, which is then the starting point for debugging, not a waveform dump. Reach for GTKWave (`make waves`) only once you need to see *why* a specific cycle produced the wrong value: find the failing time, look at the DUT's inputs and internal state (pointers, FSM `state`, counters) just before it, and find the first point where they differ from what you expected.

Run `make lint` before you commit. Note that the cocotb build treats Verilator warnings as errors, so a width mismatch in your RTL will stop the build until you fix it.

## Done when (the standard to hold your own testbenches to)

A testbench for a new project should, at minimum:

- [ ] Cover every documented operation/mode at least once (directed)
- [ ] Cover boundary conditions specific to the design (overflow, full/empty, min/max, reset behavior)
- [ ] Include a randomized test with a reference model, run for enough iterations to be meaningful (1000+ where it's cheap)
- [ ] Fail loudly and specifically when something's wrong — no silent passes, no "looks fine" from eyeballing a waveform
- [ ] Pass the **sanity check**: deliberately break your design (flip a comparison, change `>>>` to `>>`, drop a `!full`, send UART bits MSB-first) and confirm at least one test fails. If nothing fails, your testbench has a hole. Then undo the change.

# Part 2: Thinking in Parallel — HDL and Dataflow

## 2.1 The Mindset Shift

Before we write a single line of Verilog or VHDL, we need to address the biggest intellectual hurdle in learning hardware design. Read the following carefully.

**When you write Python or C, you are writing a sequence of instructions for a machine to execute one at a time.** Even with threading and vectorization, there is a fundamental mental model of "line 1 runs, then line 2 runs, then line 3 runs."

**When you write Verilog or VHDL, you are not writing a program. You are writing a description of physical hardware.**

When your Verilog is synthesized, the tool does not create a CPU that "executes" your code. It creates a network of LUTs, flip-flops, wires, and hard IP blocks. Once the FPGA is configured, all of that logic operates **simultaneously and continuously**, driven by the clock. There is no program counter. There is no "next instruction." Every combinational gate is always evaluating. Every flip-flop is always listening to its data input and waiting for the clock edge.


This is both the power and the difficulty of hardware design.

---

## 2.2 Hardware Description Languages: Verilog Fundamentals

We will use Verilog (specifically SystemVerilog-compatible synthesizable RTL) in this course. VHDL follows the same concepts with different syntax; the principles are identical. If you are comfortable with VHDL you are free to use it for the assignment.

### 2.2.1 The Module: Hardware's Basic Unit

In Verilog, the fundamental building block is the **module** — analogous to a hardware component or a schematic symbol. Every module has explicit **ports** (inputs and outputs). When you instantiate a module, you are placing a component on a schematic and wiring its pins.

```verilog
// A module describing a 2-to-1 multiplexer
module mux2to1 (
    input  wire       sel,    // Port declarations are explicit
    input  wire [7:0] a,      // 8-bit bus
    input  wire [7:0] b,      // 8-bit bus
    output wire [7:0] y       // Output
);

    // Continuous assignment: this IS a wire in hardware
    assign y = sel ? b : a;

endmodule
```

There are no loops iterating over this. No function call overhead. The `assign` statement describes a **physical multiplexer circuit** — 8 pairs of AND gates, an OR gate, and an inverter (or more efficiently, 8 LUTs with `sel`, `a[i]`, and `b[i]` as inputs). The moment `sel`, `a`, or `b` changes, `y` changes — with only propagation delay.

### 2.2.2 Combinational Logic: Continuous Assignments

Combinational logic has no memory; its output is a pure function of its current inputs.

In Verilog, you describe combinational logic with:

- **`assign` statements** for simple wire-level logic
- **`always @(*)` blocks** for more complex combinational structures

```verilog
module alu_simple (
    input  wire [3:0] a, b,
    input  wire [1:0] op,
    output reg  [3:0] result
);

    // always @(*) means: re-evaluate whenever ANY input changes.
    // The synthesis tool sees this as purely combinational logic.
    always @(*) begin
        case (op)
            2'b00: result = a + b;   // ADD → maps to LUTs or DSP
            2'b01: result = a - b;   // SUB
            2'b10: result = a & b;   // AND
            2'b11: result = a | b;   // OR
            default: result = 4'bx;
        endcase
    end

endmodule
```

> **Common Mistake:** In a combinational `always` block, if your `case` or `if-else` does not cover every possible input combination, the synthesis tool will **infer a latch** to hold the previous value. Latches are not inherently bad, but unintentional latches are a major source of bugs and timing violations. Always have a `default` case.

### 2.2.3 Sequential Logic: The Clocked Always Block

Sequential logic has state. In hardware, state is stored in **flip-flops**, and flip-flops are driven by a clock. In synthesizable Verilog, you describe sequential logic with a clocked `always` block:

```verilog
module d_register #(parameter WIDTH = 8) (
    input  wire             clk,   // Clock input — every sequential element shares this
    input  wire             rst_n, // Active-low synchronous reset (common in FPGAs)
    input  wire [WIDTH-1:0] d,     // Data input
    output reg  [WIDTH-1:0] q      // Registered output
);

    // always @(posedge clk): this block is ONLY evaluated at the
    // rising edge of the clock. It synthesizes to flip-flops.
    always @(posedge clk) begin
        if (!rst_n)
            q <= {WIDTH{1'b0}};   // Non-blocking assignment (<=) is mandatory here
        else
            q <= d;
    end

endmodule
```

**Non-blocking assignments (`<=`) are the law in clocked blocks.** This is not a style preference. Non-blocking assignments model the simultaneous sampling and updating of all flip-flops at the clock edge. Using blocking assignments (`=`) in a clocked block will produce simulations that appear correct but synthesize to wrong hardware — a notoriously hard-to-debug class of errors.

> **The tips of Verilog RTL:**
> 1. Use `assign` or `always @(*)` with `=` for combinational logic.
> 2. Use `always @(posedge clk)` with `<=` for sequential logic.
> 3. Never mix the two within the same `always` block.

### 2.2.4 Structural Verilog: Wiring Modules Together

Just as you draw a schematic by placing components and wiring their pins, you build hierarchical hardware designs by **instantiating modules**:

```verilog
module top_level (
    input  wire clk, rst_n,
    input  wire [7:0] data_in,
    output wire [7:0] data_out
);

    wire [7:0] stage1_out; // An intermediate wire between two modules

    // Instantiate a register stage
    d_register #(.WIDTH(8)) reg_stage1 (
        .clk   (clk),
        .rst_n (rst_n),
        .d     (data_in),
        .q     (stage1_out)
    );

    // Wire-level logic on the intermediate signal
    assign data_out = stage1_out ^ 8'hA5; // XOR with a constant

endmodule
```

Note the named port connections (`.clk(clk)`, etc.). Always use named connections when instantiating modules; positional connections are a maintenance hazard.

---

## 2.3 Pipelining in Hardware: Trading Latency for Throughput

You have studied CPU pipelining — the idea of breaking instruction execution into stages (Fetch, Decode, Execute, Writeback) so that multiple instructions can be in-flight simultaneously. The same principle applies, even more powerfully, in custom datapath design.

### 2.3.1 Why Pipelining Matters: The Critical Path

The maximum clock frequency your design can achieve (`F_max`) is determined by the **critical path** — the longest combinational logic path between any two flip-flops (or between an input and a flip-flop, or a flip-flop and an output).

```
      FF_A                          FF_B
       │                             │
       │    ┌────┐  ┌────┐  ┌────┐  │
       └───►│LUT1├─►│LUT2├─►│LUT3├─►│
            └────┘  └────┘  └────┘
            ←─── Critical Path Delay = T_crit ───►

            F_max = 1 / (T_crit + T_setup + T_routing)
```

If your critical path passes through 10 LUTs and the associated routing, your entire design is limited to the clock period that can accommodate 10 LUT delays. Everything else runs slower than it needs to.

**Pipelining inserts registers (flip-flops) in the middle of that long combinational path**, cutting it into shorter segments. Each segment can now run at a higher frequency.

### 2.3.2 A Concrete Example: Pipelined vs. Unpipelined Multiply-Accumulate

Consider computing `result = A × B + C`, where A, B, C are 8-bit values.

**Unpipelined version:**

```
Cycle N:   A,B,C available
           │
           ▼
        ┌──────────────────────────────────────────┐
        │  A × B  (multiply delay: ~3ns)           │
        │         ↓                                │
        │  product + C  (add delay: ~1ns)          │
        └──────────────────────────────────────────┘
           │
           ▼
Cycle N+1: result captured (minimum clock period: ~4ns → F_max ~250 MHz)
```

One result per clock. Clock period constrained to ~4ns.

**Pipelined version (2 stages):**

```
Stage 1:   A × B  (multiply delay: ~3ns)  → register product
Stage 2:   product + C  (add delay: ~1ns) → register result

Clock period only needs to accommodate: max(3ns, 1ns) = 3ns
F_max ~333 MHz — an improvement of 33%

But more importantly — THROUGHPUT:
Cycle N:   A0,B0 enter multiply stage
Cycle N+1: A0×B0 exits multiply, enters add stage  |  A1,B1 enter multiply
Cycle N+2: result0 exits add stage                 |  A1×B1 exits multiply, enters add
                                                   |  A2,B2 enter multiply
```

After the initial **pipeline fill latency** of 2 cycles, you get one result every single clock cycle — at a higher frequency than the unpipelined version. For a streaming application processing thousands of values, this is an enormous win.

> **Pipeline Fill and Flush:** The first result takes N clock cycles (N = number of pipeline stages) to emerge. This initial latency is usually acceptable because after that, the throughput is one result per clock cycle. Knowing your pipeline depth is essential for control logic.

The Verilog for a 2-stage pipelined multiplier:

```verilog
module pipelined_mult_add #(parameter W = 8) (
    input  wire             clk, rst_n,
    input  wire [W-1:0]     a, b,
    input  wire [2*W-1:0]   c,
    output reg  [2*W-1:0]   result,
    output reg              result_valid   // Registered valid signal tracks data
);

    // Pipeline Stage 1: Multiplication
    reg [2*W-1:0] product_reg;
    reg           valid_s1;
    reg [2*W-1:0] c_s1;       // C must be delayed to align with product

    always @(posedge clk) begin
        if (!rst_n) begin
            product_reg <= 0;
            valid_s1    <= 0;
            c_s1        <= 0;
        end else begin
            product_reg <= a * b;        // Infers DSP slice
            valid_s1    <= 1'b1;         // Assume inputs always valid here
            c_s1        <= c;            // Delay C to align with pipeline
        end
    end

    // Pipeline Stage 2: Accumulate
    always @(posedge clk) begin
        if (!rst_n) begin
            result       <= 0;
            result_valid <= 0;
        end else begin
            result       <= product_reg + c_s1;
            result_valid <= valid_s1;
        end
    end

endmodule
```

Notice `c_s1` — a pipeline delay register for the `c` input. This is **data alignment**, one of the most important (and most commonly forgotten) aspects of pipelined design. When you pipeline one operand through extra registers, all other operands that combine with it later must be delayed by the same number of stages.

---

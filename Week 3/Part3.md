
# Part 3: Finite State Machines and Control Logic

## 3.1 The Role of FSMs: Replacing Software Control Flow with Hardware State

In software, you control program flow with `while` loops, `if/else` chains, and function calls. These constructs rely on a CPU's program counter to track "where you are" in the execution. In hardware, there is no program counter. There is no "where you are" — everything is always running simultaneously.

**Finite State Machines (FSMs)** are how hardware implements control flow. An FSM explicitly encodes a finite number of operational states and the conditions that cause transitions between them. It is the skeleton upon which your datapath hangs.

### 3.1.1 Moore vs. Mealy Machines

You have studied these in your digital logic course, but let us revisit them in the context of hardware design:

| | Moore Machine | Mealy Machine |
|---|---|---|
| **Output depends on** | Current state only | Current state AND current inputs |
| **Output timing** | Synchronized with state register | Can change immediately with inputs (combinational) |
| **FPGA preference** | Generally preferred | Use carefully; combinational outputs can cause glitches |
| **Diagram** | Outputs labeled on states | Outputs labeled on transition arcs |

In synchronous FPGA design, Moore machines are typically safer because their outputs are registered, which prevents glitches on downstream logic. A Mealy output that changes based on an input arriving mid-cycle can cause setup time violations if not carefully constrained.

### 3.1.2 The 3-Process FSM Template

The industry-standard way to write a synthesizable FSM in Verilog is the **3-process (or 3-always-block) style**:

1. **State Register Process:** A clocked `always` block that simply stores the current state.
2. **Next-State Logic Process:** A combinational `always` block that computes the next state based on current state and inputs.
3. **Output Logic Process:** A combinational (Mealy) or registered (Moore) `always` block for outputs.

```verilog
// Example: Simple 3-state FSM
// States: IDLE → RUNNING → DONE

module simple_fsm (
    input  wire clk, rst_n,
    input  wire start, data_valid,
    output reg  busy, done
);

    // --- State Encoding ---
    // One-Hot encoding (explained in Section 3.3)
    localparam IDLE    = 3'b001;
    localparam RUNNING = 3'b010;
    localparam DONE    = 3'b100;

    reg [2:0] state, next_state;

    // --- Process 1: State Register (Sequential) ---
    always @(posedge clk) begin
        if (!rst_n)
            state <= IDLE;
        else
            state <= next_state;
    end

    // --- Process 2: Next-State Logic (Combinational) ---
    always @(*) begin
        next_state = state; // Default: stay in current state
        case (state)
            IDLE:    if (start)       next_state = RUNNING;
            RUNNING: if (!data_valid) next_state = DONE;
            DONE:                     next_state = IDLE;
            default:                  next_state = IDLE;
        endcase
    end

    // --- Process 3: Output Logic (Moore — depends only on state) ---
    always @(*) begin
        busy = 1'b0;
        done = 1'b0;
        case (state)
            IDLE:    begin busy = 0; done = 0; end
            RUNNING: begin busy = 1; done = 0; end
            DONE:    begin busy = 0; done = 1; end
            default: begin busy = 0; done = 0; end
        endcase
    end

endmodule
```

The `next_state = state` default assignment at the top of the next-state logic is critical — it means "if no condition matches, stay in the current state," which avoids unintended latches.

---

## 3.2 Handshaking and Data Streaming: Valid/Ready Protocol

At this point you know how to build a pipelined datapath (Part 2) and how to sequence operations with an FSM. Now we need to connect modules together in a way that handles a real-world challenge: **what happens when one module produces data faster than the next can consume it?**

The answer is a **handshaking protocol**. The most ubiquitous in FPGA and SoC design is the **Valid/Ready (or Valid/Accept) protocol**.

### 3.2.1 The Two-Signal Contract

Every data channel carries two control signals in addition to the data payload:

- **`valid`** (driven by the **producer/source**): "I have valid data on the data bus right now."
- **`ready`** (driven by the **consumer/sink**): "I am ready to accept data right now."

**A transfer occurs if and only if both `valid` AND `ready` are HIGH on the same rising clock edge.**

```
          CLK  ─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐
                └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘

         VALID  ──────────────────────────
                      ▲         ▲
         READY  ────────────┐   │  ┌──────
                            └───┘  └
                                │
         DATA   ─── D0 ─────────┤── D1 ──
                              ↑ Transfer ↑
                              (VALID & READY both high)
```

This protocol is elegant because:

- The **producer** never needs to know the consumer's state. It just asserts valid when it has data and keeps the data stable until the transfer occurs.
- The **consumer** never needs to know the producer's state. It asserts ready when it can accept data.
- **Neither side can lose data.** If ready is low, the producer must keep its data stable and keep valid high. If valid is low, the consumer knows there is nothing to consume.

### 3.2.2 Implementing a Simple FIFO-Based Handshake

In practice, mismatches in producer/consumer rates are handled with FIFOs, which are almost always built on top of FPGA Block RAM (BRAM):

```
 Producer ──[valid/data]──► FIFO ──[valid/data]──► Consumer
          ◄──[ready]──────        ◄──[ready]───────
                            (BRAM-based)
                            Absorbs bursts &
                            rate mismatches
```

Xilinx (AMD) provides parameterizable FIFO IP cores. In your assignments, you will instantiate these directly rather than designing the FIFO from scratch — which is the standard industry practice.

---

## 3.3 State Machine Encoding

When you implement a binary-encoded FSM, an N-state machine uses `log₂(N)` flip-flops. When you use one-hot encoding, you use N flip-flops — one per state. For a 16-state FSM, that is 4 flip-flops (binary) vs. 16 flip-flops (one-hot).

At first glance, one-hot seems wasteful. But in FPGAs, this reasoning inverts for several reasons:

**1. FPGAs are flip-flop rich.** Modern FPGAs have roughly 1–2 flip-flops per LUT. A typical design uses far fewer flip-flops than LUTs. Using extra flip-flops for state encoding costs almost nothing.

**2. One-hot dramatically simplifies next-state and output logic.** With binary encoding, every next-state equation is a complex Boolean function of multiple state bits. With one-hot, the next-state condition for state `S_i` is almost always: "I was in state `S_{i-1}` AND the transition condition is true." This maps to **a single LUT** per state.

**3. One-hot is faster.** Simpler next-state logic means shorter critical paths and higher `F_max`.

```
Binary FSM for 4 states:
  next_state[1:0] = f(state[1:0], inputs)
  Each bit requires a function of 3 variables → 1 LUT each

One-Hot FSM for 4 states:
  next_state[0] = state[3] | (state[0] & !start)  → 1 LUT
  next_state[1] = state[0] &  start               → 1 LUT
  next_state[2] = state[1] &  done                → 1 LUT
  next_state[3] = state[2]                         → wire (no LUT!)
```



---

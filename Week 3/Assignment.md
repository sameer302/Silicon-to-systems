
# Assignment: Design of a Pipelined MAC Accelerator

## Overview

You are an engineer at a startup building an AI-powered wearable device — a smart glasses platform that performs real-time gesture recognition from a wrist-worn IMU array. The embedded ARM Cortex-M4 processor cannot meet the real-time requirements for the 16-element dot-product at the heart of the inference kernel. Your task is to design a custom hardware accelerator on the accompanying FPGA fabric.


**Submission:** Verilog/VHDL source files, a testbench, waveform screenshots, a written analysis.

---

## Scenario Context

The inference kernel computes:

$$\text{result} = \sum_{i=0}^{15} A_i \times B_i$$

where `A[i]` and `B[i]` are 8-bit unsigned integers, and the result is a 16-bit accumulator. This operation is a **dot product** of two 16-element vectors — the fundamental operation in fully-connected neural network layers.

The system provides data pairs `(A[i], B[i])` one pair per clock cycle via a Valid/Ready interface. When all 16 pairs have been processed, the accelerator must assert a `Done` signal and hold the final accumulated result.

---

## Task 1: RTL Datapath Design (40 points)

### Specification

Design a `mac_unit` module with the following interface: (you may define the corresponding entity if you are using VHDL)

```verilog
module mac_unit (
    input  wire        clk,
    input  wire        rst_n,
    // Data Input Interface (Valid/Ready handshake)
    input  wire [7:0]  a_in,        // 8-bit operand A
    input  wire [7:0]  b_in,        // 8-bit operand B
    input  wire        data_valid,  // Input data is valid
    output wire        data_ready,  // Accelerator can accept data
    // Result Output Interface
    output reg  [15:0] accumulator, // Running 16-bit accumulator
    output reg         result_valid // Accumulator holds valid result
);
```

### Requirements

1. **Pipeline Stage 1 (Multiply):** Compute `product = a_in × b_in` (8×8 → 16-bit result). This result must be registered in a pipeline register at the end of Stage 1.

2. **Pipeline Stage 2 (Accumulate):** Add `product` to the running `accumulator`. This result must be registered at the end of Stage 2.

3. **Data Alignment:** Since Stage 1 introduces a 1-cycle latency, the accumulation in Stage 2 must use the product from Stage 1, not the raw inputs. Ensure your control signals are properly delayed to match.

4. **Reset Behavior:** On assertion of active-low `rst_n`, the accumulator must clear to zero within one clock cycle.

### Design Guidance

Your pipeline should look like:

```
         ┌─────────┐    Stage 1 Reg    ┌────────────┐    Stage 2 Reg
 a_in ──►│         │   ┌───────────┐   │            │   ┌───────────┐
         │  a × b  ├──►│  product  ├──►│ acc + prod ├──►│  result   ├──► accumulator
 b_in ──►│         │   └───────────┘   │            │   └───────────┘
         └─────────┘                   └────────────┘
```

**Hint:** The accumulator itself is the Stage 2 register. In Stage 2, compute `accumulator <= accumulator + product_reg`.

---

## Task 2: FSM Control Logic (35 points)

### Specification

Design a `mac_controller` FSM wrapper that manages the `mac_unit`. The FSM must implement the following state diagram:

```
                  ┌─────────┐
        rst_n=0   │         │
       ──────────►│  IDLE   │◄─────────────────┐
                  │         │                  │
                  └────┬────┘                  │
                       │ start=1               │
                       ▼                       │
                  ┌─────────┐                  │
                  │  LOAD   │ ← Receive data   │
                  │         │   pairs via      │
                  │ count++ │   Valid/Ready     │
                  └────┬────┘   handshake      │
                       │                       │
                       │ count == 16            │ next_start
                       ▼                       │
                  ┌─────────┐                  │
                  │  FLUSH  │ ← Wait for       │
                  │         │   pipeline to    │
                  │         │   drain (2 cycs) │
                  └────┬────┘                  │
                       │ flush_count == 2       │
                       ▼                       │
                  ┌─────────┐                  │
                  │  DONE   ├──────────────────┘
                  │         │ Assert done=1
                  └─────────┘
```

### Requirements

1. **IDLE state:** Assert `data_ready = 0`. Wait for a `start` pulse. Transition to LOAD.

2. **LOAD state:** Assert `data_ready = 1`. For each clock cycle where `data_valid & data_ready`, increment an internal counter. Transition to FLUSH when the counter reaches 16. Assert `data_ready = 0` upon transition.

3. **FLUSH state:** Wait for **exactly 2 clock cycles** (the pipeline depth) to allow the last data pair to propagate through both pipeline stages. Use a 2-bit counter.

4. **DONE state:** Assert `done = 1` for one clock cycle. Transition back to IDLE.

5. **One-Hot Encoding:** Implement the FSM using one-hot state encoding. Comment your code to explain why one-hot was chosen over binary.

### Interface

```verilog
module mac_controller (
    input  wire        clk,
    input  wire        rst_n,
    input  wire        start,       // Begin accumulation
    input  wire        data_valid,  // Upstream data valid
    output wire        data_ready,  // We are ready to receive
    output wire        mac_enable,  // Enable signal to mac_unit
    output wire        mac_clear,   // Clear accumulator at start
    output reg         done         // Accumulation complete
);
```

---

## Task 3: Simulation and Analysis (25 points)

### Testbench Requirements

Write a testbench `tb_mac_top.v` that:

1. Instantiates and connects `mac_unit` and `mac_controller`.
2. Generates a clock with a 10 ns period (100 MHz).
3. Applies a reset for 3 clock cycles.
4. Pulses `start` for 1 clock cycle.
5. Sends 16 data pairs. Use the values: `A[i] = i+1`, `B[i] = 2` for `i = 0..15`. (So the expected result is `2 × (1+2+...+16) = 2 × 136 = 272`.)
6. After asserting `data_valid` for 16 cycles, de-asserts it and waits for `done`.
7. Verifies that the accumulator output equals **272** when `done` is asserted.



### Waveform Requirements

Provide a waveform screenshot showing:

- [ ] Clock and reset
- [ ] `start`, `data_valid`, `data_ready`
- [ ] `a_in`, `b_in` (showing the 16 input pairs)
- [ ] The internal `product_reg` signal (visible pipeline stage)
- [ ] `accumulator` (showing it grow with each clock cycle in LOAD/FLUSH)
- [ ] FSM `state` signal (showing all four state transitions)
- [ ] `done` pulse

Annotate the waveform (with arrows or labels) to show:
1. The first pipeline stage delay (a_in×b_in → product_reg)
2. The pipeline flush phase
3. The DONE state transition

### Written Analysis (Required)

 answer the following:

**Section A — Pipeline Analysis:**
Trace through your simulation. How many clock cycles elapsed from when `start` was asserted to when `done` was asserted? Break this down by: (a) LOAD phase cycles, (b) FLUSH phase cycles, (c) DONE state cycles.

**Section B — CPU Comparison:**
Assume a Cortex-M4 processor running at 168 MHz with a single-cycle multiplier (the FPU VMUL instruction takes 1 cycle, and integer multiply takes 1–2 cycles in the M4 core).

- In the best case, how many cycles does the M4 take to compute the same 16-element dot-product? (Assume no memory latency — data is in registers. Account for: 16 multiplies + 16 additions + loop overhead ~2 instructions/iteration.)
- Compare this to your FPGA MAC's cycle count. At what operating frequency would your FPGA MAC need to run to achieve the same *wall-clock time* performance as the M4? Is that realistic?
- What happens to the CPU-vs-FPGA performance ratio if the dot-product grows to 1024 elements?

**Section C — Reflection:**
Describe one design decision you made (in either the datapath or FSM) that you would change if you were optimizing for lower latency rather than higher throughput.

---

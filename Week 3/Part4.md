
# Part 4: High-Level Synthesis and Modern Co-Design (Reading material)

## 4.1 The Evolution of Hardware Design: From Gates to Algorithms

For decades, writing RTL (Register Transfer Level) Verilog or VHDL was the only way to describe digital hardware at a level above schematics. RTL design is powerful but slow — a typical RTL designer can produce perhaps 100–200 lines of verified, timing-closed RTL per day. Complex IP blocks take months of expert engineering time.

**High-Level Synthesis (HLS)** changes this equation. HLS tools accept an algorithmic description (typically in C, C++, or SystemC) and automatically generate synthesizable RTL. The designer works with algorithmic constructs (loops, arrays, function calls) and uses **pragmas** (compiler directives) to control the hardware architecture. Last week we looked at High Level Synthesis.

### 4.1.1 What is HLS

HLS  does not transform arbitrary software into efficient hardware. It works best on **compute kernels** — computationally intensive loops with well-defined data access patterns, similar to the inner loops of signal processing, linear algebra, and neural network inference.

What HLS is very good at:

- Automatically exploring pipeline and parallelism trade-offs
- Implementing loop unrolling, array partitioning, and dataflow pipelines



### 4.1.2 A Brief HLS Example: Matrix-Vector Multiply

Here is the same computation in C (what you would write for HLS) versus the RTL it implies:

```c
// Vitis HLS C++ Source
#include "ap_int.h"   // Arbitrary-precision integer types

void matvec_mult(
    ap_int<8>  matrix[4][4],
    ap_int<8>  vector[4],
    ap_int<16> result[4]
) {
    #pragma HLS PIPELINE II=1    // Initiation interval: 1 (fully pipelined)
    #pragma HLS ARRAY_PARTITION variable=matrix complete dim=2

    for (int i = 0; i < 4; i++) {
        ap_int<16> acc = 0;
        for (int j = 0; j < 4; j++) {
            #pragma HLS UNROLL    // Fully unroll inner loop → parallel multipliers
            acc += matrix[i][j] * vector[j];
        }
        result[i] = acc;
    }
}
```

The `#pragma HLS UNROLL` on the inner loop tells the tool: *do not implement this as a sequential loop. Replicate the hardware for all 4 iterations in parallel.* The `#pragma HLS PIPELINE II=1` tells the tool: *accept a new set of inputs every single clock cycle.*

The resulting hardware is a tree of four parallel multipliers whose outputs are summed, registered, and pipelined — the same structure you would laboriously write by hand in RTL, but generated automatically.

---

## 4.2 The Modern SoC FPGA: One Chip, Two Worlds

The most significant architectural development in FPGA history over the past decade is the **SoC FPGA** (System-on-Chip FPGA). These devices integrate a **hard CPU subsystem** — typically one or more ARM Cortex cores — directly onto the same silicon die as the FPGA fabric.

Representative devices:

| Device Family | CPU Cores | FPGA Fabric |
|---|---|---|
| Xilinx Zynq-7000 | ARM Cortex-A9 (dual-core, up to 1 GHz) | 7-series FPGA (28nm) |
| Xilinx Zynq UltraScale+ MPSoC | ARM Cortex-A53 (quad) + Cortex-R5 + Mali GPU | UltraScale+ FPGA (16nm) |
| Intel Cyclone V SoC | ARM Cortex-A9 (dual-core, 925 MHz) | Cyclone V FPGA |
| Intel Agilex | ARM Cortex-A53 (quad-core) | Agilex 7 FPGA (Intel 10nm) |

The key distinction from a board with a separate CPU and FPGA is that the two subsystems on an SoC FPGA are connected through **on-chip, high-bandwidth interconnects** (AXI4 buses running at hundreds of MHz), not through slow external buses. This makes the handoff of data between CPU and FPGA fabric extremely efficient.

```
┌────────────────────────────────────────────────────────────┐
│                     SoC FPGA Die                           │
│                                                            │
│  ┌─────────────────────────┐   ┌──────────────────────┐   │
│  │   Processing System     │   │    Programmable       │   │
│  │         (PS)            │   │       Logic (PL)      │   │
│  │                         │   │                       │   │
│  │  ARM Cortex-A9/A53      │   │   CLBs, DSPs, BRAMs   │   │
│  │  L1/L2 Cache            │   │                       │   │
│  │  DDR Controller         │   │   Your Custom IP      │   │
│  │  USB, Ethernet, I2C...  │   │   Accelerators        │   │
│  │                         │   │                       │   │
│  └──────────┬──────────────┘   └─────────┬────────────┘   │
│             │                             │                 │
│             └──────── AXI4 Bus ──────────┘                 │
│                    (on-chip, fast)                         │
└────────────────────────────────────────────────────────────┘
```

### 4.2.1 Hardware/Software Partitioning: The Art of the Split

When designing a system on an SoC FPGA, the most critical design decision is: **what runs on the CPU, and what gets accelerated in the FPGA fabric?**

The methodology is:

1. **Profile the algorithm first.** Run the complete algorithm on the CPU and use profiling tools (or Amdahl's Law reasoning) to identify the computational bottleneck — the "hot kernel."

2. **Characterize the bottleneck.** Is it compute-bound (lots of arithmetic, few memory accesses)? Or memory-bound? Compute-bound kernels are ideal for FPGA acceleration. Memory-bound kernels may benefit from BRAM-based caching strategies.

3. **Define the software/hardware interface.** Decide on the data format, the AXI4 interface type (AXI4-Lite for registers, AXI4-Stream for bulk data), and the synchronization mechanism (polling, interrupts, or DMA).

4. **Implement and co-simulate.** Run the CPU software model and the hardware simulation together in a co-simulation environment. Xilinx Vitis and Intel Quartus both support this.

5. **Iterate.** The first partitioning is almost never optimal. Profile the accelerated system and refine.

#### Worked Example: Sensor Fusion on an AR Headset

Consider a sensor fusion pipeline that:
- Reads 100 samples/sec from an IMU (accelerometer + gyroscope)
- Runs a complementary filter (involves `sin()`, `cos()`, matrix multiply)
- Feeds orientation data to a 60fps rendering pipeline

Profiling reveals:
- IMU interrupt handling, state management, communication: 5% of cycles → stays on CPU
- Complementary filter's 4×4 matrix multiply: 80% of cycles → **offload to FPGA**
- Output formatting and rendering commands: 15% → stays on CPU

The FPGA accelerator is given a simple register-based AXI4-Lite interface. The CPU writes the 32 input floats to memory-mapped registers, pulses a `start` signal, waits for a `done` interrupt, and reads back the 3 output angles. The heavy math runs on a custom pipelined matrix multiplier in the FPGA fabric, using 16 DSP slices, completing in ~20 clock cycles at 200 MHz.

---

We suggest that if you want to use verilog, you should setup a Verilog toolchain properly. For VHDL if you have quartus it is fine. You can find various setup guides online. Have your favourite LLM guide you!

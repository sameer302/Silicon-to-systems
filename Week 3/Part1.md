# Introduction to FPGAs


# Part 1: The "When, Why, and Where" of FPGAs

## 1.1 The Hardware Spectrum

Before we understand FPGAs, we need to understand why they exist. You are already familiar with two ends of a spectrum. Let us place them properly.
The below is a simplification of the scenario.

```
┌─────────────────────────────────────────────────────────────────────┐
│                      THE COMPUTE HARDWARE SPECTRUM                  │
│                                                                     │
│   HIGH FLEXIBILITY ◄────────────────────────────► HIGH EFFICIENCY  │
│                                                                     │
│   ┌─────────────┐      ┌─────────────┐      ┌─────────────────┐    │
│   │     CPU     │      │    FPGA     │      │      ASIC       │    │
│   │             │      │             │      │                 │    │
│   │ Fetch/Decode│      │Reconfigurable│     │ Fixed Silicon   │    │
│   │ /Execute    │      │   Fabric    │      │  Logic Gates    │    │
│   │             │      │             │      │                 │    │
│   │ Any task,   │      │ Any hardware│      │ One task,       │    │
│   │ low perf/W  │      │ medium perf │      │ peak perf/W     │    │
│   └─────────────┘      └─────────────┘      └─────────────────┘    │
│                                                                     │
│   Dev Cost:  $0            $10k–$100k           $1M–$100M+         │
│   Flexibility: ●●●●●          ●●●○○                ●○○○○           │
│   Performance: ●○○○○          ●●●○○                ●●●●●           │
│   Time to Market: Days        Weeks                Years            │
└─────────────────────────────────────────────────────────────────────┘
```

### The CPU: A Factory Assembly Line

You know this model already from week 1. A CPU is a general-purpose workhorse. Its logic gates, ALU, register file, and branch predictor are **permanently etched into silicon**. You feed it instructions and data; it processes them sequentially (or out-of-order, but still discretely). The hardware never changes — only the software does.

This is enormously flexible. The same chip can run your browser, your sensor driver, and your neural network inference code. But that flexibility has a cost: for any specific task, most of the chip is idle most of the time.

### The ASIC: A Purpose-Built Machine

At the other extreme, an ASIC is a chip designed from scratch for exactly one job. The Bitcoin mining ASICs in data centers, the H.264 video encoder in your phone's media processor, the dedicated NPU in Apple's M-series chip — these are all ASICs. They are so efficient at their specific computation that they consume orders of magnitude less energy than a CPU doing the same work.

The catch: designing an ASIC costs millions of dollars in non-recurring engineering (NRE) and takes 12–24 months to fabricate. If you discover a bug or need to change the algorithm after tape-out, you start over.

### The FPGA: A Configurable Silicon Fabric

The FPGA sits between these two extremes, and it is a genuinely different beast.

Unlike a CPU or ASIC, the FPGA has no fixed function at power-up. It downloads a **bitstream** (a configuration file, typically from on-board flash memory) that physically connects its internal programmable logic cells into any digital circuit you have described.

---

## 1.2 Where FPGAs Dominate: The "Why" Behind the Markets

Understanding where FPGAs win is as important as understanding what they are. There are three core advantages that drive their deployment: **low and deterministic latency**, **massive data parallelism**, and **algorithm flexibility without ASIC NRE costs**.

### High-Frequency Trading (HFT)

In financial markets, the difference between profit and loss can be measured in **nanoseconds**. When a market event occurs (a price tick, a news headline parsed by an algorithm), an HFT system must detect that event, run a trading strategy, and issue an order to an exchange as fast as physically possible.
Check out the following links-

https://youtu.be/RCb8PsdipHI?si=QyBmnNPtWWt8STev

https://www.imc.com/us/articles/how-are-fpgas-used-in-trading

You are encouraged to explore more about FPGA uses!

A CPU-based system, no matter how fast, must fetch and execute instructions one at a time. It passes through an operating system, a network stack, and a software framework. Latency is in the range of **~10 microseconds at best**.

An FPGA-based system can be wired so that the moment the last byte of a network packet arrives on the optical transceiver, the **hardware logic itself** has already parsed the packet header, compared the price against a threshold stored in a register, and begun driving the output serializer with the response packet. End-to-end latency drops to **~100–200 nanoseconds** — before a CPU has even finished its interrupt handler.

### Edge AI: Smart Wearables & AR Glasses

Consider the computational demands of a next-generation AR headset. It must simultaneously:

- Run computer vision (feature detection, SLAM) at 60+ frames per second.
- Process audio and run a keyword-detection neural network.
- Manage BLE/WiFi sensors and fuse inertial measurement data.
- Do all of this on a milliwatt-scale battery budget.

A powerful CPU/GPU combination could handle the workload, but would drain a battery in minutes and generate too much heat. An ASIC could be hyper-efficient, but every six months the neural network architecture improves and you would need a new chip.

An FPGA (or increasingly, a small SoC FPGA paired with a microcontroller) offers a sweet spot: you can **re-architect the hardware acceleration pipeline** as your ML model evolves, without a new chip. For a startup shipping 10,000 units, this is the difference between being viable and not.

### Cloud Datacenters: Reconfigurable Compute

Microsoft's Project Catapult (now the backbone of Azure's networking and AI inference acceleration) and AWS's F1 instances (which offer bare-metal FPGA access to cloud customers) are proof that FPGAs have entered the datacenter mainstream.

In cloud workloads, the needs are heterogeneous and changing. One customer needs custom genomic sequence matching; another needs low-latency network packet processing; another needs to accelerate a specific proprietary compression algorithm. FPGAs let datacenters offer **hardware-customized compute** without committing to purpose-built ASICs for every possible use case.

---

## 1.3 Inside the Fabric: FPGA Architecture

Now let us open the hood. Understanding the physical structure of an FPGA is critical because it directly shapes how you write good HDL code.

### 1.3.1 Configurable Logic Blocks (CLBs)

The fundamental unit of logic in an FPGA is the **Configurable Logic Block**, or CLB. Depending on the vendor (Xilinx/AMD calls them CLBs; Intel/Altera calls them Logic Elements or Adaptive Logic Modules), the naming varies, but the internal structure is essentially the same.

A CLB contains two primary components:

**Look-Up Tables (LUTs)**

A LUT is the most important concept in FPGA architecture. Forget AND gates, OR gates, NAND gates. In an FPGA, **arbitrary combinational logic is implemented by a look-up table**, not by interconnected discrete gates.

A typical modern FPGA uses a **6-input LUT (LUT6)**. Here is how it works:

```
         6-bit address (A[5:0])
              │
              ▼
  ┌───────────────────────┐
  │  64-cell SRAM table   │◄── Programmed by bitstream
  │                       │    (stores the truth table)
  │  Address → Data Out   │
  └───────────┬───────────┘
              │
              ▼
          Output (O)
```

A LUT6 is literally a **64-entry SRAM**. The 6 logic inputs form a 6-bit address that selects one of 64 stored bits. That stored bit is the output. The bitstream programs what those 64 bits are — and that determines what Boolean function the LUT computes.

Think about what this means: a LUT6 can implement **any** Boolean function of 6 variables. XOR, majority vote, complex selector, arbitrary polynomial — it doesn't matter. The synthesis tool figures out the truth table and loads it into the SRAM at configuration time.

> **This is why FPGA logic is so fast and uniform.** There is no difference in the delay of an AND gate versus a complex 6-input function — both take one LUT, and one LUT has a fixed propagation delay.

**Flip-Flops (FFs)**

Paired with each LUT is one or more **D-type Flip-Flops**. These provide the sequential (clocked) storage that makes synchronous design possible. In Xilinx 7-series FPGAs, each slice (a grouping of 4 LUTs) contains 8 flip-flops. In modern UltraScale+ devices, the ratio is even higher.

The abundance of flip-flops is a defining characteristic of FPGAs and has major implications for design style, as we will see in Part 3.

A simplified CLB looks like this:

```
              Inputs
             (up to 6)
                │
                ▼
        ┌───────────────┐
        │     LUT       │──────────────────────►  Combinational
        │  (64-b SRAM)  │                          Output
        └───────┬───────┘
                │
                ▼
        ┌───────────────┐
        │  Flip-Flop    │──────────────────────►  Registered
        │    (D-FF)     │                          Output
        └───────────────┘
                ▲
            Clock (CLK)
```

You, as the designer, choose (through your HDL) whether the output of a LUT feeds directly to downstream logic (combinational path) or gets registered in the flip-flop first (sequential path). This choice is the entire basis of synchronous digital design.

### 1.3.2 Programmable Interconnect: The Routing Matrix

Logic cells alone are useless without wiring. In an ASIC, the metal routing layers connect gates with fixed copper traces determined at fabrication time. In an FPGA, the interconnect is also programmable.

The FPGA fabric is a grid of CLBs surrounded by a sea of **programmable switch matrices** and routing channels. Each switch matrix contains a grid of pass transistors controlled by SRAM bits from the bitstream. By setting these bits, the Place & Route (P&R) tool can connect the output of any CLB to the input of any other CLB through any available routing track.

```
     CLB ──┬──── Switch ──┬──── CLB
           │   Matrix     │
     CLB ──┘              └──── CLB
               (any-to-any
                connections
              set by bitstream)
```

> **Key Insight:** Routing delay dominates in FPGAs. Unlike an ASIC where you can optimize every metal trace for your specific circuit, FPGA routing uses generic switch matrices. For timing-critical paths, the P&R tool works hard to place connected CLBs physically close together to minimize routing delay. This is why "timing closure" — making sure all signals arrive at flip-flops before the clock edge — is a central challenge of FPGA development.

### 1.3.3 Beyond CLBs: Dedicated Hard IP Blocks

Modern FPGAs are not just a sea of LUTs. They integrate several **hard (fixed-function) blocks** that would be extremely area- and power-inefficient to build from LUTs:

| Block | Purpose |
|---|---|
| **Block RAM (BRAM)** | Dual-port synchronous SRAM, 18 Kb or 36 Kb per tile. Used for FIFOs, LUTs for large lookup, frame buffers. |
| **DSP Slices** | Dedicated 18×18 or 27×18 multiplier-accumulator units. Critical for signal processing. Far more efficient than LUT-based multipliers. |
| **PLLs / MMCMs** | Clock generation and management. Multiply, divide, and deskew clock signals. |
| **SerDes (GTX/GTP)** | High-speed serial transceivers for PCIe, 10GbE, SATA, etc. |
| **PCIe Hard Core** | A full PCIe Gen3 endpoint controller in silicon, saving thousands of LUTs. |

When writing HDL for FPGAs, you want to **infer** these hard blocks wherever possible. A well-written multiply-accumulate structure in Verilog will be automatically mapped to DSP slices by a good synthesis tool — which is 10× more efficient than building the same multiplier from LUTs.

---

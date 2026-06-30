
# Part 2: GPU Architecture Basics

## 2.1 The Core Design Philosophy: Throughput Over Latency

Before looking at any block diagram, let us take a look at this-

| | **CPU** | **GPU** |
|---|---|---|
| **Optimizes for** | Latency — finish *one* task as fast as possible | Throughput — finish *many* tasks per unit time |
| **Core count** | Few (4–64 typical) | Thousands (a modern GPU has 1,000s of simple cores) |
| **Core complexity** | High — deep pipelines, branch prediction, out-of-order execution, large caches | Low — simple pipelines, small caches, minimal branch prediction |
| **Best at** | Sequential logic, complex control flow, unpredictable branching | Data-parallel workloads: the same operation on huge datasets |

The CPU prioritizes flexibility and per-task speed; the ASIC prioritizes per-task efficiency at the cost of all flexibility; the FPGA gives you hardware-level customizable parallelism. The **GPU's replicate a moderately flexible core thousands of times** and rely on a number of identical workers to deliver throughput.

## 2.2 Inside the Chip: Streaming Multiprocessors (SMs)

A modern GPU is not one big undifferentiated group of cores. It is organized hierarchically. The major building block is the **Streaming Multiprocessor (SM)** — NVIDIA's term (AMD calls the equivalent a **Compute Unit, or CU**; Intel calls it a **Xe-core**).

```
┌───────────────────────────────────────────────────────────────────┐
│                          ENTIRE GPU CHIP                           │
│                                                                     │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│   │   SM 0   │  │   SM 1   │  │   SM 2   │  │   SM 3   │  ... (up  │
│   │          │  │          │  │          │  │          │   to 100+│
│   └──────────┘  └──────────┘  └──────────┘  └──────────┘   on high │
│                                                              -end)  │
│   ┌─────────────────────────────────────────────────────┐          │
│   │             L2 Cache (shared by all SMs)             │          │
│   └─────────────────────────────────────────────────────┘          │
│   ┌─────────────────────────────────────────────────────┐          │
│   │          Global Memory (GDDR6X / HBM, several GB)    │          │
│   └─────────────────────────────────────────────────────┘          │
└───────────────────────────────────────────────────────────────────┘
```

Each individual SM is itself a small, self-contained parallel processor:

```
┌─────────────────────────────────────────────────────────────┐
│                    ONE STREAMING MULTIPROCESSOR (SM)         │
│                                                               │
│   ┌──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┐  │
│   │ CUDA │ CUDA │ CUDA │ CUDA │ CUDA │ CUDA │ CUDA │ CUDA │  │  ← Dozens of
│   │ Core │ Core │ Core │ Core │ Core │ Core │ Core │ Core │  │     simple
│   └──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┘  │     ALU cores
│                                                               │
│   ┌────────────────┐   ┌──────────────────┐  ┌────────────┐ │
│   │ Tensor Cores    │   │  Shared Memory /  │  │  Register   │ │
│   │ (matrix MAC)    │   │  L1 Cache         │  │  File       │ │
│   └────────────────┘   └──────────────────┘  └────────────┘ │
│                                                               │
│   ┌─────────────────────────────────────────────────────┐   │
│   │           Warp Scheduler(s) — issues instructions     │   │
│   └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

A high-end GPU might contain on the order of **100+ SMs**, each containing **64–128 individual cores**, for a total core count in the tens of thousands. Compare this to a high-end CPU with perhaps 64 cores a difference of roughly **two to three orders of magnitude** in raw core count, traded against per-core complexity.

## 2.3 The Execution Model: Threads, Blocks, Warps, and SIMT

### 2.3.1 The Programmer's View: Threads and Blocks

When you write a GPU program, you describe the work in terms of an enormous number of lightweight **threads**. Unlike a CPU thread (which is a heavyweight OS-managed entity with its own stack and context, expensive to create and switch) (please look up what a thread means in CPU, use your favorite LLM!), a **GPU thread is extremely cheap** — you are expected to launch thousands or millions of them for a single problem.

Threads are organized into a two-level hierarchy:

- **Thread:** The smallest unit of execution. Each thread typically processes one data element (e.g., one pixel, one matrix entry, one array index).
- **Block (Thread Block):** A group of threads (commonly 128–1024) that are scheduled together onto a *single SM* and can cooperate via fast on-chip shared memory.
- **Grid:** The entire collection of blocks needed to cover the whole problem.

```
                              GRID
        ┌─────────────────────────────────────────────┐
        │   Block(0,0)   Block(1,0)   Block(2,0)  ...  │
        │   Block(0,1)   Block(1,1)   Block(2,1)  ...  │
        └─────────────────────────────────────────────┘
                              │
                 each Block maps to one SM
                              ▼
        ┌─────────────────────────────────────────────┐
        │              Block (e.g., 256 threads)        │
        │   T0  T1  T2  T3  T4  T5 ... T255             │
        └─────────────────────────────────────────────┘
```

### 2.3.2 The Hardware's View: Warps

**The hardware does not schedule individual threads one at a time. It groups threads into fixed-size bundles called warps** (NVIDIA terminology; AMD calls the equivalent a **wavefront**).

A **warp is (on NVIDIA hardware) a group of 32 threads that execute the exact same instruction, in lockstep, at the same time**, each operating on its own private data. This is the literal hardware unit of scheduling and execution.

### 2.3.3 SIMD vs. SIMT

You may already know the term **SIMD (Single Instruction, Multiple Data)**(look this up!). GPUs implement a closely related but distinct model called **SIMT (Single Instruction, Multiple Threads)**.

| | **SIMD** | **SIMT** |
|---|---|---|
| Programming view | One instruction operates explicitly on a wide vector register | Programmer writes ordinary **scalar** code for one thread; hardware replicates it across the warp |
| Who manages the "width"? | Programmer/compiler explicitly packs data into vector lanes | Hardware automatically maps each thread to a lane |
| Divergence handling | Not really applicable — it's one instruction stream by construction | Hardware *can* let threads in a warp take different branches, just at a steep performance cost (see 2.3.4) |


### 2.3.4 Warp Divergence

Because all 32 threads in a warp must execute the *same instruction* at the same time, what happens when an `if/else` branch causes different threads in the same warp to want to go different ways?

```verilog-like pseudocode
if (thread_id % 2 == 0)
    result = compute_path_A(data);   // even threads want this
else
    result = compute_path_B(data);   // odd threads want this
```

The hardware cannot truly run both paths at once. Instead, it **serializes**: it first executes `compute_path_A` for the whole warp, with the odd threads masked off (idle, doing no useful work), and then executes `compute_path_B` for the whole warp, with the even threads masked off.

```
Warp (32 threads), divergent branch:

Time ──────────────────────────────────────────►

Step 1: [T0][T2][T4]...[T30] active → compute_path_A
        [T1][T3][T5]...[T31] MASKED (idle)

Step 2: [T1][T3][T5]...[T31] active → compute_path_B
        [T0][T2][T4]...[T30] MASKED (idle)

Total time = time(A) + time(B), even though only
half the threads did useful work in each step.
```

This is called **warp divergence**, and it is one of the most important performance pitfalls in GPU programming. A warp full of unpredictable branches can be several times slower than the same warp executing uniform code — this is the GPU analogue of branch misprediction penalties on a CPU, but with a different and often steeper cost structure, because it affects an entire group of 32 threads rather than just one instruction stream.

## 2.4 GPU vs. CPU vs. ASIC vs. FPGA: The Four-Way Comparison

You now have enough vocabulary to place all four major compute platforms on a single, precise comparison.
This is your Assignment Problem 1 for this week. Draw a comparision between the 4 compute units. Be as elaborate and creative as you can!, Feel free to explore Part3 and 4 before answering this.

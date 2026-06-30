# Assignment P2: Build a GPU From First Principles
### GPU Architecture
Feel free to discuss this assignment, look up resource online or ask!
## Premise
Let's make this a bit story based in the world of battlegrounds! There might be some plotholes, bonus points for the one who spots any.


You are a hardware architect in 1999. Your team has just been handed the following brief:

> *"We need to render a 1024 × 768 game frame at 60 frames per second. Each pixel requires a lighting calculation involving 4 multiplications, 4 additions, and 1 texture lookup. The CPU is too slow. Design dedicated hardware to solve this problem."*

You will make major architectural decision — core count, grouping, execution model, memory hierarchy, and scheduling. At each stage, the concept is explained, and then you are asked to decide and justify. By the end, you will have designed a simplified GPU from scratch and compared your design against a real one.

---

## Part 1 — The Problem in Numbers

Before designing anything, let's understand the workload.

### Background

At 1024 × 768 resolution and 60 fps, you must process:

$$\text{Pixels per second} = 1024 \times 768 \times 60 = 47,185,920 \approx 47 \text{ million pixels/sec}$$

Each pixel requires:
- 4 multiply operations
- 4 add operations
- 1 texture lookup (assume this costs 8 operations equivalent)

$$\text{Operations per pixel} = 4 + 4 + 8 = 16 \text{ operations}$$

$$\text{Total operations per second} = 47\text{M} \times 16 \approx 754 \text{ million ops/sec}$$

Assume each of your custom cores can execute **1 operation per clock cycle**, and you are targeting a clock frequency of **500 MHz** (achievable with late-1990s process technology).

$$\text{Operations per core per second} = 500 \times 10^6 = 500 \text{ million}$$

---

### Questions — Part 1

**Q1.1**
Using the numbers above, calculate the minimum number of cores your hardware needs to meet the 754 million ops/sec requirement. Show your working. Round up to the nearest power of 2.

**Q1.2**
A colleague suggests: "Just use a fast CPU — the Pentium III runs at 500 MHz." The Pentium III has 1 core and can issue 3 operations per clock (superscalar). Calculate its throughput in operations per second. Can it meet the requirement? By how much does it fall short?

**Q1.3**
The key insight in Q1.1 is that this workload is **parallelizable**. In one or two sentences, define what makes a workload parallel. Explain why pixel rendering fits this definition perfectly.

---

## Part 2 — Core Design: Simple or Complex?

You have determined you need a certain number of cores. Now decide what *kind* of core to build.

### Background

You have two options for your individual compute cores:

**Option A — Complex Cores (CPU-style)**
Each core has: branch predictor, out-of-order execution, large instruction cache, deep pipeline (14+ stages), hardware prefetcher.
- Can execute any code efficiently
- Each core costs approximately **200 million transistors**
- Can run 3 instructions per cycle (superscalar)

**Option B — Simple Cores (GPU-style)**
Each core has: short in-order pipeline (5 stages), small instruction buffer, no branch predictor, no out-of-order logic.
- Best at predictable, arithmetic-heavy, branch-free code
- Each core costs approximately **10 million transistors**
- Executes exactly 1 instruction per cycle

You have a transistor budget of **1 billion transistors** for the entire chip.

---

### Questions — Part 2

**Q2.1**
Calculate how many cores of each type you can fit within your 1 billion transistor budget. Show your calculation for both options. Leave some budget (20%) aside for memory, routing, and control logic before calculating.

**Q2.2**
Which option gives you more raw cores? Which option gives you more raw throughput (total operations per second) given the pixel-rendering workload? Fill in this table:

| | Complex Cores (Option A) | Simple Cores (Option B) |
|---|---|---|
| Cores within budget | | |
| Total ops/sec (at 500 MHz) | | |
| Meets 754M ops/sec target? | | |

**Q2.3**
Pixel shading code has almost no unpredictable branches (the same formula runs on every pixel). Given this, explain in 2–3 sentences why the branch predictor and out-of-order logic in Option A cores are largely wasted on this workload.

**Q2.4 — Design Decision**
Which core type do you choose? Write a one-paragraph justification that references your transistor budget, your throughput numbers, and the nature of the workload. This is your first architectural commitment — be specific.

---

## Part 3 — Grouping Cores: Streaming Multiprocessor

You now have a large number of simple cores. Running them all independently creates a control and routing nightmare. You need to group them.

### Background

Grouping cores into clusters gives you:
- **Shared instruction fetch/decode:** Instead of each core fetching its own instruction stream, a group of cores shares one instruction fetch unit. If they all run the same program (likely for pixels!), this saves enormous area and power.
- **Shared local memory:** Cores within a group can exchange data without touching slow global memory.
- **Shared scheduler:** One scheduling unit manages the group, reducing control logic overhead.

This cluster is exactly what NVIDIA calls a **Streaming Multiprocessor (SM)**.

The trade-off: cores within a group are **tightly coupled** — they often must execute the same instruction at the same time (we will revisit this in Part 4).

Common cluster sizes in real hardware: 8, 16, 32, 64, or 128 cores per SM.

---

### Questions — Part 3

**Q3.1**
You have the total number of simple cores from Q2.1 (Option B). Consider three grouping options:

| Grouping | Cores per SM | Number of SMs |
|---|---|---|
| Option X | 8 | ? |
| Option Y | 32 | ? |
| Option Z | 128 | ? |

Fill in the "Number of SMs" column using your total core count from Q2.1.

**Q3.2**
For each grouping option, state one advantage and one disadvantage. Consider: scheduling flexibility, control overhead, shared memory utilization, and the granularity of work that can be assigned to one SM.

**Q3.3 — Design Decision**
Choose a grouping size for your design. Justify your choice in 2–3 sentences. There is no single correct answer — your justification is what matters.

**Q3.4**
Inside each SM, you will add a small block of fast on-chip memory shared by all cores in that SM. If your total chip has 512 KB of on-chip memory budget and all of it is split equally across your SMs, how many KB does each SM get? Is this a meaningful amount for caching intermediate pixel data? (For reference: one pixel in RGBA float format = 16 bytes.)

---

## Part 4 — The Execution Model: Inventing SIMT

You have cores grouped into SMs. Now decide how those cores inside an SM actually execute instructions.

### Background

You notice something about pixel rendering: neighboring pixels on screen almost always run **exactly the same shader program** with different input data (their position, their texture coordinates). This is the core insight that leads to the SIMT model.

**Option 1 — Independent Execution (MIMD)**
Every core in the SM has its own program counter and fetches its own instruction independently.
- Maximum flexibility — each core can be doing something completely different
- Requires N instruction fetch/decode units per SM (one per core) — expensive

**Option 2 — Lockstep Execution (SIMD)**
All cores in the SM execute the same instruction on the same clock cycle, operating on different data.
- One instruction fetch/decode unit services the whole SM — cheap and efficient
- Problem: the programmer must explicitly pack data into wide vector registers. Writing a "scalar" program for one pixel and having it run on 32 pixels at once is not straightforward.

**Option 3 — SIMT (Single Instruction, Multiple Threads)**
Cores execute in lockstep like SIMD, but the programmer writes **scalar code for one thread**. The hardware automatically groups threads into execution bundles (which NVIDIA will later call **warps**) and runs them in lockstep.
- One instruction fetch/decode unit per warp group — efficient like SIMD
- Programmer writes simple scalar code — friendly like MIMD
- Trade-off: if threads in the same warp diverge (take different branches), the hardware must serialize, which costs performance

---

### Questions — Part 4

**Q4.1**
You have 32 cores per SM (assume you chose Option Y from Part 3). Under the SIMT model, you group all 32 cores into one **warp** that executes in lockstep.

A pixel shader has the following branch:
```
if (surface_type == METAL)
    color = compute_specular(light, normal);
else
    color = compute_diffuse(light, normal);
```

In a warp of 32 threads processing 32 neighboring pixels, 20 pixels are METAL surfaces and 12 are diffuse. Draw a timeline showing what happens in this warp. Label: which threads are active, which are masked, and how many total instruction cycles are spent compared to a warp where all 32 pixels are the same surface type.

**Q4.2**
Why is SIMT preferable to pure SIMD from a **programmer's perspective**? Why is it preferable to full MIMD from a **hardware cost perspective**? Answer in 3–4 sentences total.

**Q4.3 — Design Decision**
You choose SIMT with a warp size of 32. A colleague argues for a warp size of 64 (like AMD's wavefronts). Give one argument in favor of warp size 32 and one argument in favor of warp size 64. Which would you choose for your design, and why?

**Q4.4**
Under SIMT, all 32 threads in a warp must run the same instruction. This means your SM only needs **one instruction fetch and decode unit** for all 32 cores. If a single instruction fetch/decode unit costs 5 million transistors, how many transistors do you save per SM by using SIMT versus giving every core its own fetch/decode unit?

---

## Part 5 — Memory Hierarchy: Feeding the Cores

Your cores can compute quickly — but only if they have data. Design the memory system.

### Background

Your chip needs multiple levels of memory, each with different speed/cost/size trade-offs:

| Memory Level | Latency | Bandwidth | Cost per KB | Location |
|---|---|---|---|---|
| Registers | 1 cycle | Unlimited | Very High | Per-core |
| On-chip Shared Memory | ~20 cycles | Very High | High | Per-SM |
| On-chip L2 Cache | ~100 cycles | High | Medium | Whole chip |
| Off-chip DRAM (Framebuffer) | ~400 cycles | Medium | Low | External chip |

The fundamental challenge: your cores need data every cycle, but DRAM takes 400 cycles to respond. If every core stalls waiting for DRAM, your 754M ops/sec chip actually delivers 1/400th of that.

**Texture data** (the images applied to 3D surfaces) is a major memory consumer. In your 1024×768 game, textures total approximately 128 MB — far too large to fit on-chip.

---

### Questions — Part 5

**Q5.1**
Calculate the minimum DRAM bandwidth your chip needs if every pixel requires 1 texture lookup that fetches 16 bytes from DRAM (at 47 million pixels/sec). Express your answer in GB/s.

**Q5.2**
Your on-chip L2 cache is 2 MB. A texture used by many pixels is 512 KB. Explain qualitatively how the L2 cache can dramatically reduce DRAM traffic for this texture. What property of rendering (many pixels reusing the same texture) makes caching effective here?

**Q5.3**
Explain the concept of **latency hiding** in the context of your SM design. If SM has 4 warps resident simultaneously, and each warp stalls for 400 cycles waiting on a DRAM texture fetch, sketch a timeline showing how the warp scheduler keeps the cores busy. How many warps do you need to fully hide a 400-cycle memory latency if each warp runs for 20 cycles before issuing a memory request?

**Q5.4 — Design Decision**
You must allocate your 512 KB of total on-chip memory budget (from Q3.4). Propose a split between:
- Shared memory per SM (programmer-accessible scratchpad)
- L1 cache per SM (automatic, hardware-managed)

Justify your split. Consider: what does the programmer benefit from in terms of explicit scratchpad control vs. what is better left to automatic caching?

**Q5.5**
Your chip will connect to off-chip GDDR memory. Modern GDDR6 provides approximately 512 GB/s bandwidth on a 256-bit memory bus running at 16 Gbps. Does this meet your Q5.1 requirement? Show your working and state your conclusion.

---

## Part 6 — Putting It Together: Your Complete Design

### Questions — Part 6

**Q6.1 — Complete Block Diagram**
Draw a complete block diagram of your GPU. Your diagram must show, clearly labeled:

- The full chip boundary
- All SMs (you may show 2–3 as representative and indicate the total count)
- Inside one SM: cores, shared memory, warp scheduler, instruction fetch/decode unit
- L2 cache (chip-wide)
- Memory controller
- Off-chip GDDR memory
- Arrows showing data flow from GDDR → L2 → Shared Memory → Cores, and results flowing back

This can be hand-drawn, digitally drawn, or made with any diagramming tool. Neatness and completeness matter more than artistic quality.

**Q6.2 — Design Summary Table**
Fill in the following summary of your final design:

| Parameter | Your Design Choice | Justification (one sentence) |
|---|---|---|
| Core type | | |
| Total cores | | |
| Cores per SM | | |
| Total SMs | | |
| Warp size | | |
| Execution model | | |
| On-chip memory per SM | | |
| L2 cache size | | |
| Off-chip memory | | |

**Q6.3 — Comparison with Real Hardware**
The NVIDIA GeForce 256 (1999) had: 1 fixed-function pipeline, no programmable shaders, and was designed entirely for graphics. The NVIDIA GeForce 8800 GTX (2006) — the first unified shader GPU — had: 128 CUDA cores, 16 SMs, 8 cores per SM, 32-thread warps (SIMT), 16 KB shared memory per SM, 512 MB GDDR3 at 86.4 GB/s.

Answer the following:
- How does your core count compare to the 8800 GTX?
- Your execution model (SIMT, warp size) — does it match the 8800 GTX? If you chose differently, what might have driven NVIDIA's choice?
- The 8800 GTX has only 16 KB shared memory per SM. Is that more or less than your design? What constraint might have led NVIDIA to that number in 2006?

**Q6.4 — Forward Look**
The NVIDIA H100 (2022) has: 144 SMs, 128 CUDA cores per SM (18,432 total), 256 KB registers per SM, 228 KB shared memory per SM, 50 MB L2 cache, 3.35 TB/s memory bandwidth (HBM3), and Tensor Cores for matrix operations.

In 3–4 sentences, describe the trajectory from your 1999 design to the H100. What scaled up? What stayed architecturally similar? What was added that your 1999 design could not have anticipated?

---

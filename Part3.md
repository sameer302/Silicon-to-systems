
# Part 3: GPU Memory and Performance Concepts

## 3.1 Why Memory, Not Just Cores, Determines GPU Performance

A common beginner mistake is to assume that because a GPU has thousands of cores, any sufficiently "parallel" program will automatically run fast. In practice, **the majority of real-world GPU performance problems are memory problems, not compute problems**. A GPU's arithmetic units can typically perform far more operations per second than its memory system can supply data for — this imbalance is so common it has a name: a kernel is either **compute-bound** or **memory-bound**, and most real kernels are the latter.

Understanding the memory hierarchy, and how to use it well, is therefore not an optional advanced topic — it is the difference between using 5% and 80% of the GPU's theoretical performance.

## 3.2 The GPU Memory Hierarchy

```
┌──────────────────────────────────────────────────────────────────┐
│                     GPU MEMORY HIERARCHY                          │
│                                                                    │
│   FASTEST, SMALLEST, PER-THREAD                                   │
│   ┌────────────────────────────┐                                 │
│   │   Registers (per thread)    │  ~1 cycle latency               │
│   └────────────────────────────┘  Tens of KB per SM, split        │
│                  │                 across active threads          │
│                  ▼                                                │
│   ┌────────────────────────────┐                                 │
│   │  Shared Memory / L1 Cache    │  ~20-30 cycle latency           │
│   │  (per thread BLOCK)          │  Tens to ~100+ KB per SM,       │
│   └────────────────────────────┘  programmer-managed scratchpad   │
│                  │                                                │
│                  ▼                                                │
│   ┌────────────────────────────┐                                 │
│   │  L2 Cache (shared by all SMs)│  ~200 cycle latency             │
│   └────────────────────────────┘  Several MB, automatic           │
│                  │                                                │
│                  ▼                                                │
│   ┌────────────────────────────┐                                 │
│   │  Global Memory (GDDR6X/HBM)  │  ~400-600+ cycle latency        │
│   └────────────────────────────┘  Many GB, visible to all         │
│   SLOWEST, LARGEST, CHIP-WIDE       threads on the entire GPU      │
└──────────────────────────────────────────────────────────────────┘
```

- **Registers:** Private to each thread, fastest possible storage. Used automatically by the compiler for local variables.
- **Shared Memory:** A small, fast, programmer-managed scratchpad shared by all threads *within the same block*. This is the primary tool programmers use to avoid repeatedly hitting slow global memory — e.g., loading a tile of data once into shared memory and letting many threads reuse it.
- **L2 Cache:** Automatically managed, shared across the entire chip, helps reduce traffic to global memory.
- **Global Memory:** The large pool of GDDR6X or HBM memory (several to tens of GB) that all threads on the GPU can access. This is also where data is initially copied from the host (CPU) system, and where results are copied back.

## 3.3 Memory Bandwidth

GPUs are often discussed purely in terms of compute throughput (FLOPS), but their **memory bandwidth** is just as defining a characteristic. A high-end GPU's global memory might deliver **on the order of 1–3 TB/s** of bandwidth — compared to roughly 50–100 GB/s for a typical CPU's DRAM. This 10–30× bandwidth advantage exists specifically because GPU workloads (graphics, AI, simulation) are usually moving huge volumes of data, not just performing a few isolated calculations.

But high peak bandwidth is only achievable if the **access pattern** is favorable — which brings us to an important practical performance concept in this course.

## 3.4 Memory Coalescing

DRAM (the technology behind GPU global memory) is fastest when accessed in large, contiguous chunks — not when accessed as many scattered, small, individual reads. When a warp of 32 threads issues a memory request, the hardware checks whether those 32 requests can be **combined ("coalesced") into one or a few wide memory transactions**.

```
COALESCED ACCESS (FAST)                  UNCOALESCED / STRIDED ACCESS (SLOW)
────────────────────────                 ───────────────────────────────────
Thread:  T0  T1  T2  T3 ... T31           Thread:  T0   T1   T2   T3  ... T31
Address: A0  A1  A2  A3 ... A31           Address: A0  A64  A128 A192 ... A992
         │   │   │   │       │                     │    │    │    │        │
         └───┴───┴───┴───────┘                     └────┴────┴────┴────────┘
         Contiguous block                          Scattered across memory
              │                                            │
              ▼                                            ▼
     ONE wide memory transaction                  UP TO 32 SEPARATE
     serves the entire warp                        memory transactions
                                                    (one per thread!)

     Effective bandwidth: ~full rated speed         Effective bandwidth: a small
                                                     fraction of rated speed —
                                                     sometimes 10-30x slower
```

**The practical rule:** when consecutive threads in a warp access consecutive memory addresses, you get coalescing and near-peak bandwidth. When consecutive threads access memory addresses that are far apart (a "strided" or "scattered" access pattern — common, for instance, when accessing a 2D array by column instead of by row), the hardware must issue many separate transactions, and effective bandwidth can collapse by an order of magnitude or more. This is why **how you lay out and index your data is often more important than how clever your algorithm is.**

## 3.5 Latency Hiding and Occupancy

If global memory access takes ~400–600 clock cycles, how does a GPU achieve high performance at all?

A single SM does not run just one warp at a time. It keeps **many warps "in flight"** simultaneously — far more than its core count would suggest, because most of those warps are not actively computing at any given moment; they are *waiting* on a memory request. The warp scheduler's job is simple: the instant one warp stalls waiting for memory, **instantly switch to a different warp that has data ready**, with essentially zero context-switch overhead (unlike a CPU thread switch, which is comparatively expensive).

```
SM executing MANY warps to hide memory latency:

Warp A:  [compute][──── waiting for memory (400+ cycles) ────][compute]
Warp B:           [compute][──── waiting for memory ────][compute]
Warp C:                    [compute][──── waiting ────][compute]
Warp D:                             [compute][── waiting ──][compute]

Scheduler's actual issued work over time:
         [A][B][C][D][A][B][C][D] ...  ← cores stay busy almost continuously,
                                          by hopping to whichever warp is ready
```

This only works if there are *enough* warps available to hop between. The metric for "how many warps are actively available, relative to the hardware maximum" is called **occupancy**.

- **Occupancy** = (number of active warps on an SM) / (maximum warps the SM could support)
- High occupancy gives the scheduler many options to hide latency behind.
- Occupancy is limited by **resource usage per thread**: if each thread uses a lot of registers, or each block uses a lot of shared memory, fewer threads/blocks can be resident on the SM simultaneously, lowering occupancy.

> **Important nuance:** Higher occupancy is usually good, but it is not automatically the goal in itself — it is a means to hide latency. A kernel can sometimes achieve excellent performance at moderate occupancy if it does not need much latency-hiding (e.g., it is compute-bound with little memory traffic).

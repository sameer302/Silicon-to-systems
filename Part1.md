# Parallel by Design: An Introduction to GPUs


---

---

## Week Overview

You have already studied three points on the hardware spectrum: the CPU (general-purpose, sequential-by-default), the ASIC (fixed-function, maximum efficiency), and the FPGA (reconfigurable, hardware-level parallelism). This week introduces the fourth major compute platform — and arguably the one reshaping the entire technology industry right now: the **Graphics Processing Unit (GPU)**.

It is a fundamentally different machine, built around a fundamentally different bet: that most large computational problems are not a long chain of dependent steps, but a *wide* collection of mostly-independent steps that can be done simultaneously, if only you have enough simple cores to throw at them.

---

# Part 1: GPU Origins and Why It Exists

## 1.1 The Problem: Pixels Are Embarrassingly Parallel

To understand why the GPU looks the way it does, you have to start with the problem it was built to solve.

Consider a 1990s 3D video game rendering a single frame at 640×480 resolution. That is **307,200 pixels**. For each pixel, the system must determine: what color is this point, given the 3D geometry, the lighting, and the camera angle? Critically — **the color of pixel (100, 200) does not depend on the color of pixel (101, 200)**. They can, in principle, all be computed at exactly the same time, by completely independent hardware, and the final image would be identical.

This property is called **data parallelism**, and graphics rendering is one of the purest examples of it in all of computing. A CPU, with its handful of powerful cores, is a poor match for this workload — it would have to compute those 307,200 pixel colors largely one (or a few) at a time. What you actually want is hardware built from the ground up to do the *same simple operation*, on *different data*, *hundreds of thousands of times in parallel*. That hardware is the GPU.

```
┌──────────────────────────────────────────────────────────────┐
│             RENDERING ONE FRAME: 307,200 PIXELS               │
│                                                                │
│   CPU approach:                GPU approach:                  │
│   ┌─────┐                      ┌───┬───┬───┬───┬───┬───┐      │
│   │ Core│  pixel 1             │ C │ C │ C │ C │ C │ C │      │
│   │     │  pixel 2             ├───┼───┼───┼───┼───┼───┤      │
│   │     │  pixel 3             │ C │ C │ C │ C │ C │ C │      │
│   │     │  ...                 ├───┼───┼───┼───┼───┼───┤      │
│   │     │  pixel 307,200       │ C │ C │ C │ C │ C │ C │      │
│   └─────┘                      └───┴───┴───┴───┴───┴───┘      │
│   One at a time (or a few)     Thousands of simple cores,     │
│   on a few powerful cores      each computing a handful        │
│                                 of pixels at once               │
└──────────────────────────────────────────────────────────────┘
```

## 1.2 A Short History: From Fixed Function to General Purpose

The architecture of a modern GPU is a record of how this problem was solved. Understanding this evolution explains *why* GPUs have the specific structures they have today.

### Stage 1 — Fixed-Function Graphics Pipeline (1990s)

Early 3D accelerators (3dfx Voodoo, early NVIDIA RIVA, ATI Rage) implemented the graphics pipeline as **dedicated, non-programmable hardware**. There were physical circuits for vertex transformation, physical circuits for lighting calculations (using the fixed Phong or Gouraud shading model), and physical circuits for texture mapping. You could not change.

This is, in essence, an **ASIC built specifically for the graphics pipeline**. It was fast and efficient, but utterly inflexible: every visual effect had to be expressible as some combination of the fixed stages the hardware provided.

### Stage 2 — Programmable Shaders (early 2000s)

Game developers wanted custom visual effects — fog, water ripples, custom lighting models, fur, skin — that the fixed pipeline simply could not produce. The industry's answer was the **programmable shader**: small programs that ran on dedicated *vertex shader* and *pixel (fragment) shader* hardware units.

This was the critical turning point. For the first time, GPU hardware executed **arbitrary, programmer-supplied code**, not fixed circuits. Crucially, there were two *separate* types of programmable cores — vertex shader cores and pixel shader cores — each specialized for its stage of the pipeline.

### Stage 3 — Unified Shader Architecture (mid-2000s)

Game workloads vary: some frames are vertex-heavy (complex geometry, simple shading), others are pixel-heavy (simple geometry, complex lighting and post-processing). With separate vertex and pixel shader hardware, one set of cores would sit idle while the other was the bottleneck — a clear waste of fixed silicon.

The solution, introduced around the Xbox 360 and NVIDIA's GeForce 8800 (2006), was the **unified shader architecture**: a single pool of generic, programmable cores that could be dynamically allocated to vertex work, pixel work, or (soon after) general-purpose work, depending on what the current workload demanded.


### Stage 4 — General-Purpose GPU Computing (GPGPU, late 2000s)

Researchers noticed that this "unified pool of thousands of simple programmable cores" was exactly the kind of hardware that scientific computing — matrix algebra, fluid simulation, molecular dynamics — needed.

NVIDIA's release of **CUDA (Compute Unified Device Architecture) in 2007** changed this entirely. CUDA gave programmers a C-like language to write general-purpose parallel programs that ran directly on GPU shader core. This is the moment the GPU became a general parallel compute platform.

### Stage 5 — AI and Tensor Acceleration (2012–present)

The watershed moment came in 2012, when a deep neural network (AlexNet) trained on GPUs won the ImageNet competition by a dramatic margin, demonstrating that the matrix-multiplication-heavy workload of training neural networks was an almost perfect match for GPU parallelism. Within a few years, GPUs became the *default* hardware for training and running AI models.

NVIDIA responded by adding **Tensor Cores** (starting with the Volta architecture in 2017) — specialized hardware units dedicated to the exact mixed-precision matrix-multiply-accumulate operations that dominate deep learning.
```
TIMELINE OF GPU EVOLUTION
──────────────────────────────────────────────────────────────────►
1995              2001              2006              2007    2012  2017
  │                 │                 │                 │      │     │
Fixed-Function   Programmable     Unified Shader      CUDA   AlexNet Tensor
  Pipeline         Shaders         Architecture      Released  Moment  Cores
  (ASIC-like)    (Vertex/Pixel    (Generic core      (GPGPU            (AI-
                  split cores)      pool)             born)            specific
                                                                        HW)
```

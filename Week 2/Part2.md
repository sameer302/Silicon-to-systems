
## PART 2: The Hardware Paradigm and Data Flow Graphs

### Overview

Now that you understand *why* we build custom hardware, the next question is *how*. How do we take a mathematical function — something expressible in a few lines of code — and translate it into a network of physical processing elements connected by wires in silicon?

The central insight of hardware design is this: if two computations do not depend on each other's results, they can execute simultaneously in separate hardware units. Our task is to make those dependency relationships explicit, visualize them as a graph, and then reason about how to schedule the operations across time and hardware resources.

We develop all of these ideas using a single running example that we will carry through the rest of the course.

---

### 2.1 The Walkthrough Function: The Cubic Polynomial

Our working example is the standard-form cubic polynomial:

    Y = a·x³ + b·x² + c·x + d


All of the techniques introduced in Parts 2, 3, and 4 will be applied to this function. By the end, you will have translated it from an algebraic expression into a full physical datapath schematic.

---

### 2.2 Single Static Assignment (SSA): Making Dependencies Explicit

The first step is to decompose the polynomial into a sequence of **atomic operations** — steps so small that each one maps directly onto a single hardware functional unit. We use a representation called **Single Static Assignment (SSA)**, where each intermediate result is assigned to a unique name exactly once and never overwritten.

This matters because hardware does not work like software variables. In a CPU program, you can write `x = x + 1` and overwrite a variable. Hardware is different: a wire or register holds a specific value produced by a specific computation unit. Making each assignment unique forces us to be explicit about the identity and lifetime of every intermediate value in the circuit — which is precisely the information we need to design the datapath.

Here is the SSA decomposition of Y = a·x³ + b·x² + c·x + d:

    t1 = x  × x       ← compute x²
    t2 = t1 × x       ← compute x³  (x² times x)
    t3 = a  × t2      ← compute a·x³
    t4 = b  × t1      ← compute b·x²  (reuses x² from t1)
    t5 = c  × x       ← compute c·x
    t6 = t3 + t4      ← compute a·x³ + b·x²
    t7 = t6 + t5      ← compute a·x³ + b·x² + c·x
    t8 = t7 + d       ← compute the final output Y

We now have 8 uniquely named operations: 5 multiplications (t1 through t5) and 3 additions (t6 through t8). Pay attention to t1 — its result (x²) is consumed by both t2 and t4. This "fan-out" — one intermediate value feeding multiple downstream operations — is a characteristic feature of the dependency graph we are about to draw.

---

### 2.3 The Data Flow Graph (DFG): Visualizing Dependencies

A **Data Flow Graph (DFG)** is a directed graph that makes the dependency structure of a computation visually and formally explicit. The construction rules are simple:

- **Nodes** represent operations (functional units: adders, multipliers, etc.)
- **Directed edges** represent data dependencies. An edge from node A to node B means: *B cannot begin executing until A has produced its output.*
- **Source nodes** (inputs like x, a, b, c, d) have no incoming edges — they are available from the start
- **The sink node** (the final output Y = t8) has no outgoing edges

Drawing the DFG for our polynomial, the edges reflect the SSA dependencies:

- x connects into both inputs of t1
- t1's output connects into t2 (along with x) and into t4 (along with b)
- t2's output connects into t3 (along with a)
- t3 and t4 both connect into t6
- c and x connect into t5; t5 and t6 connect into t7
- t7 and d connect into t8 (the output)

The graph has a branching, tree-like structure converging at the output node. Notice the two distinct paths through the graph:

**The deep chain:** x → t1 → t2 → t3 → t6 → t7 → t8. This is a long sequential chain where each step depends on the previous one.

**The shallow path:** x → t5 → t7 → t8. This is a short path — t5 can start immediately since it only depends on the original inputs c and x.

**Why the DFG matters:** The edges define which operations *must* be sequential (a path connects them) and which operations *can* be parallel (no path connects them). t1 and t5, for instance, share no dependency — they can execute simultaneously in separate hardware units. This parallelism is exactly what we want to exploit.

Look up how a DFG actually looks like! Is it conventionally drawn vertically or horizontally?
---


### 2.4 Functional Unit Latencies

Before we can schedule anything, we need to know how long each type of functional unit takes to produce its output. These are the hardware latency assumptions we use throughout this course:

- **Adder:** 1 clock cycle
- **Multiplier:** 2 clock cycles

These numbers reflect the reality that multiplication is more computationally complex than addition at the gate level, requiring more transistor stages and therefore more time. In real designs, these values come from the technology library characterization — but for our purposes, 1 and 2 cycles are clean numbers that produce interesting scheduling problems.

---

### 2.5 Critical Path Analysis

The **critical path** through the DFG is the longest chain of dependent operations from any input to the final output, measured in total execution time. It establishes the **absolute minimum possible latency** of the circuit — a hard physical lower bound that cannot be improved regardless of how much hardware you add.

Tracing the longest weighted path through our DFG (adding up the latency of each operation in the chain):

    t1 (MUL, 2 cycles) → t2 (MUL, 2 cycles) → t3 (MUL, 2 cycles) → t6 (ADD, 1 cycle) → t7 (ADD, 1 cycle) → t8 (ADD, 1 cycle)

    Critical Path Length = 2 + 2 + 2 + 1 + 1 + 1 = 9 clock cycles

No schedule can produce the final output in fewer than 9 cycles. Even with unlimited hardware — every operation running on its own dedicated unit — the sequential dependency chain through t1, t2, t3, t6, t7, t8 forces a minimum of 9 cycles of elapsed time.

Importantly, operations t4 and t5 do **not** lie on this critical path. Their execution fits within the 9-cycle window created by the critical chain. This means they have scheduling flexibility — we can move them around in time to accommodate resource constraints without increasing the overall latency (up to a point). This flexibility is formally quantified as **mobility**, which we compute next.

---

### 2.6 ASAP Scheduling: The Earliest Legal Schedule

**ASAP (As Soon As Possible)** scheduling answers the question: if we have unlimited hardware resources, what is the earliest cycle in which each operation can legally begin?

An operation can begin as soon as all of its predecessor operations have completed. Working forward through the DFG:

- **t1** depends on nothing (only raw inputs). Starts at **cycle 1**. Completes end of cycle 2.
- **t5** depends on nothing. Also starts at **cycle 1**. Completes end of cycle 2.
- **t2** depends on t1. Earliest start = cycle 1 + 2 = **cycle 3**. Completes end of cycle 4.
- **t4** depends on t1. Earliest start = cycle 1 + 2 = **cycle 3**. Completes end of cycle 4.
- **t3** depends on t2. Earliest start = cycle 3 + 2 = **cycle 5**. Completes end of cycle 6.
- **t6** depends on both t3 (done end of cycle 6) and t4 (done end of cycle 4). It must wait for the *later* of the two: earliest start = **cycle 7**. Completes end of cycle 7.
- **t7** depends on t6 (done end of cycle 7) and t5 (done end of cycle 2). Latest prerequisite: cycle 7. Earliest start = **cycle 8**. Completes end of cycle 8.
- **t8** depends on t7. Earliest start = **cycle 9**. Completes end of cycle 9.

**ASAP Start Times:**

| Operation | Type | ASAP Start Cycle |
|-----------|------|-----------------|
| t1        | MUL  | 1               |
| t5        | MUL  | 1               |
| t2        | MUL  | 3               |
| t4        | MUL  | 3               |
| t3        | MUL  | 5               |
| t6        | ADD  | 7               |
| t7        | ADD  | 8               |
| t8        | ADD  | 9               |

The ASAP latency is **9 cycles** — exactly equal to the critical path length, as expected. This is the best possible latency. With unlimited hardware, t1 and t5 run simultaneously in cycle 1, t2 and t4 run simultaneously in cycle 3, and so on.

---

### 2.7 ALAP Scheduling: The Latest Legal Schedule

**ALAP (As Late As Possible)** scheduling answers the complementary question: given a fixed latency deadline, what is the *latest* cycle in which each operation can be scheduled without missing the deadline?

We set our deadline equal to the ASAP result — 9 cycles. ALAP is computed by working **backward** from the output.

- **t8** must complete by end of cycle 9. It takes 1 cycle. Latest start = **cycle 9**.
- **t7** must complete before t8 starts (before cycle 9 begins, so by end of cycle 8). It takes 1 cycle. Latest start = **cycle 8**.
- **t6** must complete before t7 starts (before cycle 8), by end of cycle 7. It takes 1 cycle. Latest start = **cycle 7**.
- **t3** must complete before t6 starts (before cycle 7), by end of cycle 6. It takes 2 cycles. Latest start = **cycle 5**.
- **t4** must complete before t6 starts (before cycle 7), by end of cycle 6. It takes 2 cycles. Latest start = **cycle 5**.
- **t5** must complete before t7 starts (before cycle 8), by end of cycle 7. It takes 2 cycles. Latest start = **cycle 6**.
- **t2** must complete before t3 starts (before cycle 5), by end of cycle 4. It takes 2 cycles. Latest start = **cycle 3**.
- **t1** must satisfy two constraints: t2 must start by cycle 3 (so t1 must finish by end of cycle 2) AND t4 must start by cycle 5 (so t1 must finish by end of cycle 4). The *tighter* constraint governs: t1 must finish by end of cycle 2. It takes 2 cycles. Latest start = **cycle 1**.

**ALAP Start Times:**

| Operation | Type | ALAP Start Cycle |
|-----------|------|-----------------|
| t1        | MUL  | 1               |
| t5        | MUL  | 6               |
| t2        | MUL  | 3               |
| t4        | MUL  | 5               |
| t3        | MUL  | 5               |
| t6        | ADD  | 7               |
| t7        | ADD  | 8               |
| t8        | ADD  | 9               |

---

### 2.8 Mobility: Measuring Scheduling Flexibility

**Mobility** (also called slack) is the difference between an operation's ALAP start and its ASAP start:

    Mobility(op) = ALAP_Start(op) − ASAP_Start(op)

Mobility tells you how much freedom you have in scheduling an operation. An operation with **zero mobility** is rigidly locked to one specific cycle — it *must* run at its ASAP time and cannot be deferred by even a single cycle without pushing the output past the deadline. An operation with **high mobility** can be scheduled anywhere within a window of cycles — it has slack that can be exploited to resolve resource conflicts.

**Mobility Table:**

| Operation | ASAP | ALAP | Mobility | On Critical Path? |
|-----------|------|------|----------|------------------|
| t1        | 1    | 1    | **0**    | Yes              |
| t2        | 3    | 3    | **0**    | Yes              |
| t3        | 5    | 5    | **0**    | Yes              |
| t4        | 3    | 5    | 2        | No               |
| t5        | 1    | 6    | 5        | No               |
| t6        | 7    | 7    | **0**    | Yes              |
| t7        | 8    | 8    | **0**    | Yes              |
| t8        | 9    | 9    | **0**    | Yes              |

**Interpretation:** Operations t1, t2, t3, t6, t7, t8 form the critical path — all with zero mobility. They are the spine of the computation and cannot move. t4 has 2 cycles of slack; it can be scheduled anywhere in the window [cycle 3, cycle 5]. t5 has 5 cycles of slack; it can be scheduled anywhere in the window [cycle 1, cycle 6]. This flexibility will be essential when we have limited hardware resources and need to defer some operations to free up functional units.

---

### 2.9 A Glimpse at Exact Solutions: Integer Linear Programming

The ASAP and ALAP analyses give us bounding information, and mobility tells us where we have scheduling flexibility. But when we add real-world constraints — say, we only have one multiplier and one adder — we need to decide *exactly* when each operation runs. This is a combinatorial optimization problem.

The mathematically exact approach is **Integer Linear Programming (ILP)**. The scheduling problem can be formulated as:

- **Decision variables:** Binary variable x(i,t) = 1 if operation i is scheduled in cycle t, else 0
- **Dependency constraints:** If operation B depends on A with latency L, then the scheduled cycle of B must be at least L cycles after A's scheduled cycle
- **Resource constraints:** In every cycle, the number of operations of each type cannot exceed the available units of that type
- **Objective:** Minimize total latency (for resource-constrained scheduling) or minimize resource usage (for time-constrained scheduling)

An ILP solver explores this formulation and returns the globally optimal schedule. The catch: for large circuits with hundreds or thousands of operations, ILP solvers can be slow or intractable.

In practice, HLS tools rely on **heuristic scheduling algorithms** — methods that trade guaranteed optimality for speed. The most widely used heuristic is **list scheduling**, which runs in polynomial time and consistently produces near-optimal results for realistic circuits. We cover it in full detail in Part 3.

---


## PART 4: Microarchitecture Binding

### Overview

Scheduling answers *when* each operation executes. Binding answers *what physical hardware* it executes on. Together, they define the complete **datapath** — the actual circuit that can be implemented in silicon.

In this final part, we take the schedule produced in Part 3 and transform it into a concrete microarchitecture: physical adders and multipliers, registers to hold intermediate values between cycles, and multiplexers to route data to the correct destination at each clock edge.

By the end of this section, you will have a complete Register Transfer Level (RTL) schematic of the cubic polynomial hardware accelerator.

---

### 4.1 Functional Unit Binding: Assigning Operations to Physical Units

**Functional unit binding** is the process of assigning every operation in the schedule to a specific, named physical unit of the appropriate type.

In our 1 MUL + 1 ADD schedule, the binding is straightforward: there is exactly one multiplier (call it MUL_1) and one adder (call it ADD_1). Every multiplication maps to MUL_1, and every addition maps to ADD_1.

**Binding Table:**

| Operation | Type | Cycle   | Bound To |
|-----------|------|---------|----------|
| t1        | MUL  | 1–2     | MUL_1    |
| t2        | MUL  | 3–4     | MUL_1    |
| t3        | MUL  | 5–6     | MUL_1    |
| t4        | MUL  | 7–8     | MUL_1    |
| t5        | MUL  | 9–10    | MUL_1    |
| t6        | ADD  | 9       | ADD_1    |
| t7        | ADD  | 11      | ADD_1    |
| t8        | ADD  | 12      | ADD_1    |

This is the maximum-sharing scenario — every operation of the same type shares a single physical unit, executing in different cycles. The hardware area is minimized at the cost of more cycles (12 instead of 9).

In systems with multiple units of the same type (e.g., 2 multipliers), binding becomes a richer decision: which specific multiplier handles which operation? The goal is to minimize the routing complexity (wire length, multiplexer count) while satisfying timing. For our single-unit case, the decision is forced.

---

### 4.2 Register Lifetime Analysis: How Long Must Each Value Be Stored?

Every intermediate value in our circuit — t1 through t7 — is produced by one operation and consumed by one or more downstream operations. Between production and consumption, the value must be held in a storage element: a **register** (a set of flip-flops that retain their value across clock edges).

**Register Lifetime** is the interval during which a value must remain available in a register. Formally:

    Lifetime(v) = [end of production cycle, start of last consumption cycle]

A value's lifetime begins the cycle after its producer finishes (when the value is written into a register) and ends when its last consumer reads it from that register.

**Lifetime Table for our 1 MUL + 1 ADD schedule:**

| Value | Produced End of Cycle | Last Consumed at Cycle | Must Stay in Register |
|-------|----------------------|----------------------|----------------------|
| t1    | 2                    | 7 (consumed by t4)   | Cycles 3 through 6  |
| t2    | 4                    | 5 (consumed by t3)   | Cycle 5 only         |
| t3    | 6                    | 9 (consumed by t6)   | Cycles 7 through 8  |
| t4    | 8                    | 9 (consumed by t6)   | Cycle 9 only         |
| t5    | 10                   | 11 (consumed by t7)  | Cycle 11 only        |
| t6    | 9                    | 11 (consumed by t7)  | Cycle 10             |
| t7    | 11                   | 12 (consumed by t8)  | Cycle 12 only        |

**Maximum simultaneous live values:**
Examining which values overlap in time:
- Cycles 3–4: t1 alive → 1 register
- Cycle 5: t1 alive, t2 alive → 2 registers
- Cycles 7–8: t1 alive (until cycle 7), t3 alive → 2 registers simultaneous
- Cycle 9: t3 alive, t4 alive → 2 registers
- Cycle 10: t6 alive → 1 register
- Cycle 11: t6 alive, t5 alive → 2 registers simultaneous

**The circuit requires a minimum of 2 storage registers** to hold intermediate values. Lifetime analysis thus tells us the minimum register count, which directly translates to flip-flop area and power consumption.

**Register sharing:** If two values have non-overlapping lifetimes, they can share a single physical register (with a multiplexer controlling which value gets written in). For example, t2 (alive in cycle 5) and t3 (alive in cycles 7–8) have non-overlapping lifetimes and could share a register. Minimizing registers through lifetime analysis and sharing is an important area optimization in real designs.

---

### 4.3 Multiplexer Insertion: Routing Data to the Right Place

When a single functional unit executes different operations in different cycles, its inputs must receive different data values depending on which cycle it is in. This dynamic routing is implemented with **multiplexers (MUX)**.

A multiplexer is a selector circuit: given N input data lines and a binary select signal, it outputs exactly one of the N data lines. The select signal is controlled by the datapath controller (which tracks the current cycle number) and changes each clock cycle to route the correct operands to each unit.

**Analyzing MUL_1 inputs:**

MUL_1 executes five different operations across 12 cycles. Tracing what data must appear at its two inputs in each active cycle:

| Cycle | Operation | MUL_1 Input A | MUL_1 Input B |
|-------|-----------|--------------|--------------|
| 1     | t1 = x×x  | x            | x            |
| 3     | t2 = t1×x | t1 (from REG)| x            |
| 5     | t3 = a×t2 | a            | t2           |
| 7     | t4 = b×t1 | b            | t1 (from REG)|
| 9     | t5 = c×x  | c            | x            |

MUL_1 Input A must receive: {x, t1, a, b, c} — **5 distinct sources → 5-to-1 MUX**
MUL_1 Input B must receive: {x, t2, t1} — 3 distinct sources (x appears in cycles 1, 3, and 9) → **3-to-1 MUX**

**Analyzing ADD_1 inputs:**

| Cycle | Operation  | ADD_1 Input A | ADD_1 Input B |
|-------|------------|--------------|--------------|
| 9     | t6 = t3+t4 | t3           | t4           |
| 11    | t7 = t6+t5 | t6           | t5           |
| 12    | t8 = t7+d  | t7           | d            |

ADD_1 Input A: {t3, t6, t7} → **3-to-1 MUX**
ADD_1 Input B: {t4, t5, d} → **3-to-1 MUX**

The multiplexer select signals are driven by the controller. In cycle 1, the controller sets MUX_A to select x and MUX_B to select x. In cycle 3, it sets MUX_A to select t1 (from its register) and MUX_B to select x. And so on for each active cycle.

---

### 4.4 The Datapath Controller

The multiplexers, registers, and functional units do not operate autonomously — they are directed by a **controller**. The controller is a finite state machine (FSM) with one state per clock cycle. At each clock edge, the controller transitions to the next state and updates all MUX select signals to configure the datapath for the operation scheduled in the upcoming cycle.

For a 12-cycle schedule, the controller has 12 states. Its outputs in each state are:
- The select values for all MUX inputs (telling each MUX which data source to forward)
- Register enable signals (telling which registers to latch new data this cycle)
- A "done" output signal that pulses high when Y is valid at the end of cycle 12


---

### 4.5 The Complete Datapath Schematic

Assembling all of the above, the final microarchitecture of the cubic polynomial accelerator contains:

**Functional Units:**
- MUL_1: one two-cycle multiplier, handling t1 through t5 sequentially
- ADD_1: one one-cycle adder, handling t6, t7, and t8 sequentially

**Registers:**
- REG_t1: a register that captures the result of t1 (x²) at end of cycle 2 and holds it until it is consumed by t4 at cycle 7 (a 5-cycle hold)
- REG_t3: a register that captures t3 (a·x³) at end of cycle 6 and holds it until consumed by t6 at cycle 9
- REG_t6: a register that captures t6 at end of cycle 9 and holds it until consumed by t7 at cycle 11
- Short-duration registers for t2, t4, t5, and t7 (one-cycle bridges between producer and consumer)

**Multiplexers:**
- At MUL_1 Input A: a 5-to-1 MUX selecting among {x, t1, a, b, c}
- At MUL_1 Input B: a 3-to-1 MUX selecting among {x, t2, t1}
- At ADD_1 Input A: a 3-to-1 MUX selecting among {t3, t6, t7}
- At ADD_1 Input B: a 3-to-1 MUX selecting among {t4, t5, d}


**Data flow cycle by cycle:**

- Cycles 1–2: MUL_1 computes t1 = x × x. Output stored in REG_t1.
- Cycles 3–4: MUL_1 computes t2 = REG_t1 × x. Output stored in a short-lived register.
- Cycles 5–6: MUL_1 computes t3 = a × t2. Output stored in REG_t3.
- Cycles 7–8: MUL_1 computes t4 = b × REG_t1. Output stored briefly.
- Cycle 9: MUL_1 computes t5 = c × x (result held until cycle 11). Simultaneously, ADD_1 computes t6 = REG_t3 + t4. Output stored in REG_t6.
- Cycle 10: No operations (waiting for t5 to complete).
- Cycle 11: ADD_1 computes t7 = REG_t6 + t5. Output stored briefly.
- Cycle 12: ADD_1 computes t8 = t7 + d. Output Y is valid.

This complete schematic is the final product of the High-Level Synthesis flow for the cubic polynomial.

---

### 4.6 From Schematic to Silicon: What Comes Next

The schematic you have just derived represents the circuit at the **Register Transfer Level (RTL)** — the abstraction level that describes data movement between registers across clock cycles. From here, the standard ASIC design flow takes over:

**RTL to Logic Synthesis:**
The RTL schematic is encoded in a hardware description language — VHDL or Verilog — and fed to a **logic synthesizer**. The synthesizer maps the RTL description onto a library of physical standard cells: NAND gates, NOR gates, flip-flops, buffers, and so on. It optimizes for area, timing, and power while meeting the clock frequency constraint.

**Logic to Physical Layout:**
A **place-and-route** tool takes the gate-level netlist produced by synthesis and physically positions each cell on the silicon die, then draws the metal interconnect wires between them. This step must satisfy tight constraints on timing (signal propagation must complete within the clock period), signal integrity (wire lengths and capacitances), and physical density (fitting within the allocated die area).

**Verification:**
At every stage, simulation and formal verification tools confirm that the circuit still computes the correct function. Gate-level simulation verifies timing. Equivalence checking confirms that the optimized gate netlist is logically identical to the original RTL.

**Tapeout:**
The final physical layout is converted to photomask data and sent to a semiconductor fabrication facility (fab). The fab manufactures the wafers, and the resulting chips implement our cubic polynomial accelerator in silicon.

You have now traced the complete journey: from a mathematical expression Y = ax³ + bx² + cx + d, through dependency analysis, scheduling, binding, and schematic extraction, to a manufacturable silicon design. This is the end-to-end flow of High-Level Synthesis.


We hope that you have now understood the High Level Synthesis of an application specific circuits. Think of all the different application specific circuits around you and question why they are like that and not a CPU. Hope you have fun solving the assignment!
---
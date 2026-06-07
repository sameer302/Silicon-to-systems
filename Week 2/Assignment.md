# HLS Capstone Assignment: $2 \times 2$ Matrix Transformer

### Context & Objective

You are designing a custom hardware accelerator for a real-time computer vision pipeline. The core operation is applying a $2 \times 2$ transformation matrix ($A$) to a $2 \times 2$ pixel block ($B$) to produce the output matrix ($C$).

$$C = A \times B$$

Because this operation runs on every pixel in a 4K video stream, you must explore two different microarchitecture designs: a low-power, area-constrained design for edge devices, and a high-performance, time-constrained design for a data center accelerator.

### The Target Function

The algorithm requires computing the dot product of rows and columns to generate the 4 output elements ($C_{11}, C_{12}, C_{21}, C_{22}$).

**Software Implementation:**

```cpp
void compute_2x2_matmul(int A11, int A12, int A21, int A22,
                        int B11, int B12, int B21, int B22,
                        int &C11, int &C12, int &C21, int &C22) {
    // Stage 1: The 8 Multiplications
    int m1 = A11 * B11;
    int m2 = A12 * B21;
    
    int m3 = A11 * B12;
    int m4 = A12 * B22;
    
    int m5 = A21 * B11;
    int m6 = A22 * B21;
    
    int m7 = A21 * B12;
    int m8 = A22 * B22;

    // Stage 2: The 4 Additions
    C11 = m1 + m2;
    C12 = m3 + m4;
    C21 = m5 + m6;
    C22 = m7 + m8;
}

```

### Hardware Constraints & Assumptions

* **Multipliers (`MULT`):** Take exactly **2 clock cycles** to complete. (They are non-pipelined; a unit starting in Cycle 1 is busy through the end of Cycle 2).
* **Adders (`ADD`):** Take exactly **1 clock cycle** to complete.
* **Dependencies:** An addition cannot begin until *both* of its operand multiplications have fully completed.

---

## Part 1: Data Flow Graph & Theoretical Bounds

Before applying constraints, we must understand the mathematical limits of the algorithm.

1. **Draw the DFG:** Draw the Data Flow Graph for this function. Clearly label the 8 multiplication nodes ($m_1 \dots m_8$) and the 4 addition nodes ($C_{11} \dots C_{22}$).
2. **ASAP Latency:** Assuming you have infinite `MULT` and `ADD` units, what is the Critical Path length (in clock cycles)? This is your ASAP absolute minimum latency.

---

## Part 2: Resource-Constrained Scheduling (RCS)

**Scenario:** You are targeting a low-power edge device. Your ASIC layout strictly limits you to **2 Multipliers** and **1 Adder**.

**Task:** Schedule the operations by hand to find the minimum possible latency under these tight hardware constraints.

* Use **List Scheduling**. If multiple operations are ready, you may prioritize them in numerical order (e.g., prioritize $m_1$ over $m_3$).
* Fill out the schedule table below. (Remember `MULT` takes 2 cycles).

| Clock Cycle | MULT 1 | MULT 2 | ADD 1 |
| --- | --- | --- | --- |
| **Cycle 1** |  |  |  |
| **Cycle 2** |  |  |  |
| **Cycle 3** |  |  |  |
| **Cycle 4** |  |  |  |
| **Cycle 5** |  |  |  |
| **Cycle 6** |  |  |  |
| *(Extend as needed)* |  |  |  |

* **Question:** What is the final total latency of your resource-constrained circuit?

---

## Part 3: Time-Constrained Scheduling (TCS)

**Scenario:** You are now targeting a data center accelerator. The system architecture dictates a strict real-time deadline: the entire matrix multiplication must complete in **exactly 5 clock cycles**.

**Task:** Determine the absolute *minimum* hardware resources (number of `MULT` units and `ADD` units) required to meet this 5-cycle deadline.

1. **ALAP Analysis:** Calculate the ALAP (As Late As Possible) start time for the additions ($C_{11} \dots C_{22}$). Then, calculate the ALAP start time for the multiplications ($m_1 \dots m_8$).
2. **Resource Overlap Calculation:** Based on your ASAP and ALAP start times, figure out how to distribute the 8 multiplications across the available start cycles to minimize the maximum number of simultaneously active `MULT` units.
3. **Final Verdict:** State exactly how many `MULT` units and how many `ADD` units are required, and prove it by providing the 5-cycle schedule table.

---

## Part 4: Microarchitecture Datapath

Using your **Time-Constrained (5-cycle)** schedule from Part 3:

1. Draw the final Datapath Schematic.
2. You must include the exact number of `MULT` and `ADD` blocks you calculated.
3. Draw the multiplexers (MUXes) on the inputs of the functional units that execute different operations on different clock cycles.

---
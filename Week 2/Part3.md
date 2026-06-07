
## PART 3: Scheduling Techniques — Preparing for Temporal and Spatial Scheduling

### Overview

In Part 2, we established the DFG, computed ASAP/ALAP bounds, and identified the critical path. Now we face the real challenge: scheduling all 8 operations under realistic hardware constraints.

There are two fundamental types of scheduling problems, and you need to be able to solve both by hand:

**Resource-Constrained Scheduling (RCS):** You are given a fixed number of functional units (for example, 1 multiplier and 1 adder). Find the minimum-latency schedule that respects those resource limits.

**Time-Constrained Scheduling (TCS):** You are given a fixed latency deadline (for example, you must complete in 11 cycles). Find the minimum number of functional units needed to meet that deadline.

This section teaches you every skill and technique required to solve these problems. The goal is that by the end of Part 3, you can sit down with any DFG, apply the methods you have learned, and construct a valid, efficient schedule by hand.

---

### 3.1 Foundational Skill: Reading Precedence from the DFG

Every scheduling algorithm starts with a correct understanding of what must come before what. This is called **precedence** and it comes directly from the DFG edges.

**Rule:** Operation B must be scheduled at least L cycles after operation A if there is a directed edge from A to B and A has latency L. In other words:

    Start_Cycle(B) ≥ Start_Cycle(A) + Latency(A)

This rule must hold for every edge in the DFG. Violating even one precedence constraint produces an incorrect circuit — B would start before A has finished, reading a stale or uninitialized result.

**Practice exercise:** For our polynomial DFG, verify that the following are independent (no edge connects them and they can run simultaneously):
- t1 and t5: correct, they share no dependency
- t3 and t4: correct, they share no dependency (both depend on earlier operations, but neither depends on the other)
- t2 and t5: correct, independent

And verify that these cannot overlap:
- t1 and t2: cannot — t2 depends on t1 with 2-cycle latency; t2 cannot start until cycle 3 at the earliest if t1 starts at cycle 1

Internalizing these relationships before running any algorithm is essential.

---

### 3.2 Foundational Skill: Computing the Earliest Legal Start Time

Given a partial schedule (some operations already assigned to cycles), you must be able to compute the earliest cycle in which any unscheduled operation can legally begin. The rule is:

    Earliest_Start(op) = max over all predecessors P of (Scheduled_Cycle(P) + Latency(P))

If an operation has no predecessors, it can start at cycle 1 (or the current cycle, if we are mid-schedule).

**Example:** Suppose t1 is scheduled in cycle 1 (and takes 2 cycles). When can t2 start?

    Earliest_Start(t2) = Scheduled_Cycle(t1) + Latency(t1) = 1 + 2 = 3

t2 cannot begin before cycle 3. If t2 also depended on some other operation Q that was scheduled in cycle 4 and had 1-cycle latency, then:

    Earliest_Start(t2) = max(1 + 2, 4 + 1) = max(3, 5) = 5

The bottleneck predecessor governs the start time. Always take the maximum.

---

### 3.3 Foundational Skill: Detecting and Resolving Resource Conflicts

A **resource conflict** occurs when two or more operations that require the same type of functional unit are assigned to the same clock cycle, but you have fewer units of that type than the operations require.

**Detecting conflicts:** After tentatively assigning operations to a cycle, count how many operations of each type you have placed in that cycle. If the count exceeds the available units, you have a conflict.

    Conflict if: (# of MUL operations in cycle C) > (# of available multipliers)
    Conflict if: (# of ADD operations in cycle C) > (# of available adders)

**Resolving conflicts:** When a conflict exists, one or more operations must be deferred to a later cycle. The key question is: which operation moves? The answer is determined by the priority function (discussed in Section 3.5). The operation with the highest priority stays; the others wait.

After deferring an operation, you must re-check whether deferring it causes any of its successors to also be delayed (because the successor depends on the deferred operation). This cascading effect must be tracked carefully.

---

### 3.4 Foundational Skill: The Schedule Table

A **schedule table** is the primary artifact you produce when solving a scheduling problem by hand. It is a two-dimensional grid:

- **Rows** = clock cycles (one row per cycle)
- **Columns** = functional unit slots (one column per available unit)
- **Cells** = the operation (if any) assigned to that unit in that cycle

Example format for a system with 1 multiplier (MUL) and 1 adder (ADD):

| Cycle | MUL  | ADD  |
|-------|------|------|
| 1     | t1   | —    |
| 2     | —    | —    |
| 3     | t2   | —    |
| 4     | —    | —    |
| ...   | ...  | ...  |

Build the table incrementally, one cycle at a time. At each cycle:
1. Determine which operations are ready (all predecessors complete)
2. Apply the priority function to rank them
3. Assign operations to free units, in priority order
4. Leave any remaining ready operations for the next cycle

**Multi-cycle operations and the "busy" convention:** When a multiplier starts an operation in cycle C and the operation takes 2 cycles, the multiplier is **busy in both cycle C and cycle C+1**. No new operation can be assigned to that multiplier until cycle C+2. In your schedule table, leave the multiplier cell empty (or mark it as "busy / t1 executing") in the second cycle.

---

### 3.5 The List Scheduling Algorithm

**List scheduling** is the dominant heuristic in industrial HLS tools. It is greedy — it makes locally optimal decisions cycle by cycle — but it consistently produces excellent results. Here is the full algorithm:

```
Initialize:
  current_cycle ← 1
  For all operations with no predecessors: mark as "ready"
  All functional units start as "free"

Repeat until all operations are scheduled:

  Step 1 — Build the Ready List:
    Collect every unscheduled operation whose predecessors
    have all finished by current_cycle:
        for each unscheduled op:
            if ALL predecessors P have been scheduled AND
               (start_cycle(P) + latency(P)) ≤ current_cycle:
               add op to ready_list

  Step 2 — Sort the Ready List by priority:
    Apply the chosen priority function (see Section 3.6).
    Place higher-priority operations at the front of the list.

  Step 3 — Assign operations to functional units:
    For each operation in the ready list (in priority order):
        If a suitable free unit of the correct type exists:
            Assign this operation to that unit.
            Mark the unit as busy from current_cycle to
            current_cycle + latency(op) - 1.
        Else:
            Leave this operation in the ready list.
            It will be reconsidered next cycle.

  Step 4 — Advance time:
    current_cycle ← current_cycle + 1
    Free any units whose assigned operation has now completed.
```

The algorithm terminates when every operation has been placed into the schedule table. The total number of cycles used is the schedule's latency.

---

### 3.6 Priority Functions: What Gets Scheduled First

The most critical design decision in list scheduling is the **priority function** — the rule that determines which ready operations get scheduled first when there are more ready operations than available units. The priority function is what separates a good heuristic from a poor one.

**Priority 1 — Minimum Mobility (Critical-First)**

Operations with zero mobility are on the critical path. If they are delayed even a single cycle, the total latency grows. They must be scheduled the instant they become ready.

Operations with high mobility have slack — they can safely wait a cycle or two without harming the overall latency.

Rule: Among ready operations, schedule those with *lower* mobility first.

For our polynomial: t1, t2, t3, t6, t7, t8 (all mobility = 0) always take priority over t4 (mobility = 2), which takes priority over t5 (mobility = 5).

This is the primary priority function you should use for hand calculations.

**Priority 2 — Earliest ALAP Start (Tightest Deadline)**

When two or more operations have the same mobility, break the tie by preferring the one with the smallest ALAP start cycle. An operation with an earlier ALAP deadline is more urgently constrained — waiting any longer risks eventually missing the deadline.

Rule: Among equal-mobility operations, schedule the one with the smaller ALAP start first.

**Priority 3 — Longest Remaining Path to Output**

Assign each operation a "criticality score" equal to the weighted length of the longest path from that operation to the output node. Operations that are the ancestors of long downstream chains are more urgent — delaying them causes maximum downstream harm.

Rule: Among operations under consideration, prefer those with the longer remaining path to the output.

**Which function to use:** In practice, Priority 1 (minimum mobility) with Priority 2 as a tiebreaker is the standard approach for hand calculations. It is effective because it directly encodes the constraint structure of the problem.

---

### 3.7 Resource-Constrained Scheduling: Full Worked Example

We now apply list scheduling to the cubic polynomial DFG with a **resource constraint of 1 multiplier and 1 adder**. We use minimum-mobility priority.

Starting state: Ready list = {t1 (m=0), t5 (m=5)}. Both units free.

**Cycle 1:**
Ready: {t1 (m=0), t5 (m=5)}
Priority order: t1 first.
- t1 → MUL free → assign. MUL busy cycles 1–2.
- t5 → MUL busy → defer.

Schedule: MUL=t1, ADD=empty.

**Cycle 2:**
t1 still executing. Ready: {t5} (t1's successors not yet ready — t1 not done).
- t5 → MUL busy → defer.

Schedule: MUL=busy(t1), ADD=empty.

**Cycle 3:**
t1 finishes at end of cycle 2. MUL now free.
Ready: {t5 (m=5), t2 (m=0, depends on t1 which finished), t4 (m=2, depends on t1 which finished)}
Priority: t2 (m=0) → t4 (m=2) → t5 (m=5)
- t2 → MUL free → assign. MUL busy cycles 3–4.
- t4 → MUL busy → defer.
- t5 → MUL busy → defer.
No ADD operations ready yet.

Schedule: MUL=t2, ADD=empty.

**Cycle 4:**
t2 still executing. Ready: {t4, t5}. MUL busy.
- t4 → MUL busy → defer.
- t5 → MUL busy → defer.

Schedule: MUL=busy(t2), ADD=empty.

**Cycle 5:**
t2 finishes. MUL free.
Ready: {t4 (m=2), t5 (m=5), t3 (m=0, depends on t2 which finished)}
Priority: t3 (m=0) → t4 → t5.
- t3 → MUL → assign. MUL busy cycles 5–6.
- t4 → MUL busy → defer.
- t5 → MUL busy → defer.

Schedule: MUL=t3, ADD=empty.

**Cycle 6:**
t3 executing. Ready: {t4, t5}. MUL busy.
Both defer.

Schedule: MUL=busy(t3), ADD=empty.

**Cycle 7:**
t3 finishes. MUL free.
Ready: {t4 (m=2), t5 (m=5)}. Note: t6 depends on t3 AND t4 — t4 is not yet done, so t6 is not ready.
Priority: t4 (m=2) before t5 (m=5).
- t4 → MUL → assign. MUL busy cycles 7–8.
- t5 → MUL busy → defer.

Schedule: MUL=t4, ADD=empty.

**Cycle 8:**
t4 executing. Ready: {t5}. MUL busy.
- t5 → MUL busy → defer.

Schedule: MUL=busy(t4), ADD=empty.

**Cycle 9:**
t4 finishes. MUL free. ADD free.
Ready: {t5 (m=5), t6 (m=0, both t3 and t4 now finished)}
Priority: t6 (m=0) → t5 (m=5).
- t6 → ADD → assign. ADD busy cycle 9.
- t5 → MUL → assign. MUL busy cycles 9–10.

Schedule: MUL=t5, ADD=t6.

**Cycle 10:**
t6 finishes (end of cycle 9). t5 still executing (finishes end of cycle 10).
Ready: t7 depends on t6 (done) AND t5 (not done yet) → NOT ready.
MUL busy, ADD free but nothing ready for it.

Schedule: MUL=busy(t5), ADD=empty.

**Cycle 11:**
t5 finishes. MUL free. ADD free.
Ready: {t7 (t6 done end of cycle 9, t5 done end of cycle 10 — both done)}
- t7 → ADD → assign. ADD busy cycle 11.

Schedule: MUL=empty, ADD=t7.

**Cycle 12:**
t7 finishes. ADD free.
Ready: {t8 (depends on t7, now done)}
- t8 → ADD → assign. ADD busy cycle 12.

Schedule: MUL=empty, ADD=t8.

Output Y is valid at end of cycle 12. **Total latency with 1 MUL + 1 ADD = 12 cycles.**

**Complete Schedule Table:**

| Cycle | MUL       | ADD  |
|-------|-----------|------|
| 1     | t1        | —    |
| 2     | (t1 busy) | —    |
| 3     | t2        | —    |
| 4     | (t2 busy) | —    |
| 5     | t3        | —    |
| 6     | (t3 busy) | —    |
| 7     | t4        | —    |
| 8     | (t4 busy) | —    |
| 9     | t5        | t6   |
| 10    | (t5 busy) | —    |
| 11    | —         | t7   |
| 12    | —         | t8   |

The resource constraint cost us 3 extra cycles compared to the ASAP bound of 9 cycles (a 33% latency penalty), but we achieved it with minimum hardware — just one unit of each type.

---

### 3.8 Time-Constrained Scheduling: Minimum Resources for a Given Deadline

In time-constrained scheduling, the problem is reversed: you are given a latency deadline and asked to use as few functional units as possible while meeting it.

**The core question at each step:** When a resource conflict occurs and an operation cannot be scheduled in its current cycle, check its ALAP start time. If deferring the operation by one cycle would push it past its ALAP time, it *must* run now — which means you must add another functional unit of the required type. If deferring it keeps it within its ALAP window, it can safely wait without a new unit.

**Algorithm for TCS:**

1. Begin list scheduling with an optimistic resource assumption (start with 1 of each unit type)
2. At each cycle, prioritize using minimum-mobility (or ALAP-based) ordering
3. When a conflict arises: check if the waiting operation can afford to wait one more cycle without violating its ALAP constraint
   - If YES: defer it (no new unit needed)
   - If NO: allocate an additional unit to run the operation now
4. The total number of units of each type that were ever simultaneously active is the minimum resource count for your deadline

**The trade-off curve:** There is a continuous relationship between latency and resources. For our polynomial:

- Latency = 9 cycles (ASAP bound) requires the most hardware: enough multipliers to run t1, t5 simultaneously in cycle 1, t2 and t4 simultaneously in cycle 3, etc.
- Latency = 12 cycles (constrained bound) requires only 1 MUL and 1 ADD
- Latency values between 9 and 12 require intermediate resource counts

Mapping out this relationship — latency on one axis, resource count on the other — produces the **design space curve**. This curve is one of the central deliverables of any HLS tool: it gives the designer a menu of options from which to choose based on their area, power, and performance requirements.

---

### 3.9 Verifying a Completed Schedule

Once you have a schedule, always verify it before submitting or moving to the binding step. Verification has two checks:

**Check 1 — Precedence correctness:**
For every dependency edge (A → B) in the DFG, confirm that:

    Start_Cycle(B) ≥ Start_Cycle(A) + Latency(A)

If this holds for every edge, no operation reads a value before it has been produced.

**Check 2 — Resource correctness:**
For every cycle C, count the number of MUL operations active in cycle C (both starting and in-progress from prior cycles) and the number of ADD operations active in cycle C. Confirm these counts do not exceed your available units.

If both checks pass, the schedule is valid.

---

### 3.10 Summary: Your Complete Scheduling Toolkit

Before the assignment, here is a full checklist of everything you need to apply:

1. Write the SSA decomposition of the function — identify all 8 operations
2. Draw the DFG — identify nodes (operations), edges (dependencies), and source inputs
3. Apply functional unit latencies — know that MUL = 2 cycles, ADD = 1 cycle
4. Compute ASAP start times — forward pass through the DFG
5. Compute ALAP start times — backward pass from the output, with a given deadline
6. Calculate mobility for every operation — ALAP minus ASAP
7. Identify the critical path — operations with zero mobility
8. Apply list scheduling — build the schedule table cycle by cycle using minimum-mobility priority
9. Detect and resolve resource conflicts — defer low-priority ready operations
10. Verify the completed schedule — check all precedence and resource constraints

If you can execute all 10 steps correctly and confidently, you are ready for the assignment.

---

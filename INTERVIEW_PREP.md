# Sift (MultiPaxos & Raft): Interview & Systems Defense Guide

This guide is designed for **Bradshaw Asher** to prepare for technical interviews and research reviews covering **Sift** ([Ma et al., USENIX ATC '22](file:///c:/Users/s-bas/Sift/Sift/sift_paper.pdf)), **MultiPaxos**, **Raft**, and **Automated Formal Verification**.

---

## Part 1: The Elevator Pitch

> **Question:** *"Can you explain the Sift project and how it proves complex protocols like MultiPaxos and Raft?"*

**Your Answer:**
> *"Sift is a formal verification methodology from USENIX ATC '22 that solves the fundamental tradeoff between automated model checking and expressive refinement proofs.*
> 
> *Historically, verifying real implementations of protocols like Raft or MultiPaxos required months of manual proof writing in tools like IronFleet or Verdi. Fully automated tools like IC3PO could only verify toy, monolithic protocols because quantifier alternations over dynamic log slots and quorums blow up the solver.*
> 
> *Sift introduces a **Two-Tier Refinement Architecture**: humans prove the abstract protocol at Layer 0 (Lamport's Safe Value Invariant in Paxos, Leader Completeness in Raft), and then split the path down to the executable C++ implementation into incremental refinement layers. Because each layer-to-layer refinement is small and localized, automated tools like IC3PO and Z3 can synthesize and check the simulation invariants completely push-button in seconds."*

---

## Part 2: High-Level Concepts (Easy)

### Q1: What is the difference between Single-Instance Paxos and MultiPaxos?
**Answer:**
* **Single-Instance Paxos:** Reaches agreement on **exactly one value** (e.g., electing a leader or choosing a configuration slate). Once decided, the instance is permanently closed.
* **MultiPaxos:** Extends Paxos to an infinite sequence of consensus instances (log slots $0, 1, 2, \dots$) to build a linearizable replicated state machine:
  * **Phase 1 (Prepare/Promise)** is amortized: run once across all slots when a leader is elected.
  * **Phase 2 (Propose/Accept)** is executed repeatedly for each client command to append to the log without running Phase 1 again.

---

### Q2: How does Raft compare to MultiPaxos architecturally?
**Answer:**
Both solve state machine replication with majority quorums, but differ in leader election and log management:
* **MultiPaxos:** Allows out-of-order log commits and log holes. If a new leader is elected, it must scan past slots in Phase 1b and propose `no_op` to fill uncommitted holes. Any node can become leader, even if its log is behind, as long as it collects prior votes.
* **Raft:** Enforces strict contiguous log matching (no holes). A candidate can only be elected leader if its log is **at least as up-to-date** as a majority quorum (`lastLogTerm` and `lastLogIndex`). This guarantees that a newly elected Raft leader already possesses all committed entries without needing to backfill.

---

## Part 3: Deep Protocol Mechanics (Medium)

### Q3: Why does MultiPaxos need 5 refinement layers in Sift?
**Answer:**
Going directly from abstract MultiPaxos to an executable C++ implementation in one step is impossible for automated SMT solvers. Sift decomposes it into 5 manageable layers:
1. **Layer 0 ([`paxos_protocol0.ivy`](file:///c:/Users/s-bas/Sift/Sift/multipaxos/paxos_protocol0.ivy)):** Monolithic protocol with atomic quorum joins and Lamport's Safe Value Invariant.
2. **Layer 1 ([`paxos_protocol1.ivy`](file:///c:/Users/s-bas/Sift/Sift/multipaxos/paxos_protocol1.ivy)):** Asynchronous message passing (`1a`, `1b`, `2a`, `2b`).
3. **Layer 2 ([`paxos_protocol2.ivy`](file:///c:/Users/s-bas/Sift/Sift/multipaxos/paxos_protocol2.ivy)):** Node-local ballot variables (`current_round(n)`) and vote maps (`my_votes(n)`).
4. **Layer 3 ([`paxos_protocol3.ivy`](file:///c:/Users/s-bas/Sift/Sift/multipaxos/paxos_protocol3.ivy)):** Iterative promise accumulation (`joined(n)`), slot pointwise maximum merge (`zip_max`), and `no_op` hole filling.
5. **System Layer ([`paxos_system.ivy`](file:///c:/Users/s-bas/Sift/Sift/multipaxos/paxos_system.ivy)):** Matches the exact state of DSLabs `PaxosServer` (`my_log`, `first_undecided`, `inst_status`) and serializes to network sockets via `paxos_trans.ivy`.

---

### Q4: How does Raft enforce the Log Matching Property?
**Answer:**
In Raft, if two logs contain an entry with the same index and term:
1. They store the same command.
2. Their logs are identical in all preceding entries.

In Layer 0 ([`raft_protocol0.ivy`](file:///c:/Users/s-bas/Sift/Sift/raft/raft_protocol0.ivy)), Sift proves this by induction:
* In `append_new_entry`, the leader only appends monotonically to its own log.
* In `replicate_entry`, the leader includes the term and index of the immediately preceding entry (`preix`, `pret`).
* In `recv_append`, the follower verifies that its log matches `pret` at `preix`. If a conflict exists, the follower truncates its conflicting tail and accepts the leader's entries.

---

### Q5: What is Lamport's Safe Value Invariant in Paxos?
**Answer:**
It is the central inductive invariant that guarantees Paxos Agreement across rounds:
```ivy
invariant nset.majority(Q) & R1 < R2 & ((proposal(I, R2, V2) & V1 ~= V2) | available(R2, I)) -> (
    exists N:node. nset.member(N, Q) & left_round(N, R1) & ~vote(N, I, R1, V1)
)
```
**Plain English:** If a leader in round $R_2$ proposes value $V_2 \neq V_1$ in slot $I$, then value $V_1$ could *not* have been decided in any earlier round $R_1$, because every majority quorum $Q$ contains at least one node that abandoned round $R_1$ before casting a vote for $V_1$.

---

## Part 4: Formal Logic & Verification (Hard)

### Q6: Was IC3PO used to discover all the invariants in Sift?
**Answer:**
**No.** As documented in Table 1 of the Sift paper:
* **Layer 0 (Spec $\to$ Layer 0):** Proved **manually by humans**. IC3PO failed on Layer 0 because reasoning about un-factored consensus requires complex quantifier alternations ($\forall \exists$ across quorums) that cause model checkers to time out.
* **Refinement Layers (Layer 0 $\to$ System):** Discovered **automatically by IC3PO**. Because each refinement layer introduces only one incremental systems concern (e.g., adding local variables or packetizing RPCs), IC3PO successfully synthesized the simulation invariants (yielding the `# invariant [ic3po_...]` clauses in Sift's files).

---

### Q7: Why are packet retransmissions commented out in `paxos_system.ivy`?
**Answer:**
In [`paxos_system.ivy`](file:///c:/Users/s-bas/Sift/Sift/multipaxos/paxos_system.ivy#L220-L260), handlers like `ask_for_retransmission` and `missing_two_a_handler` are commented out to stay within the **Effectively Propositional (EPR)** decidable logic fragment.

Modeling explicit packet loss, retry timers, and sequence number duplicate suppression introduces temporal quantifier alternations ($\forall t_1.\ \exists t_2$ over packet drops) that cause Z3 to diverge or time out. Sift proves consensus safety assuming an abstract reliable transport shim (similar to TCP in DSLabs).

---

### Q8: Why was `maxvote` commented out in `votemap_segment.ivy`?
**Answer:**
In [`votemap_segment.ivy`](file:///c:/Users/s-bas/Sift/Sift/multipaxos/votemap_segment.ivy#L39-L55), the authors originally defined:
```ivy
# function maxvote(M:votemap_seg, I:inst) : vote_struct
# definition maxvote(M,I) = votemap_seg.value(M,I) if votemap_seg.first(M) <= I & I < votemap_seg.upper(M) else not_a_vote
```
In Ivy and Z3, uninterpreted function definitions with quantifiers over array segments trigger **E-matching loops** (quantifier instantiation cycles) during SMT solving. The authors inlined the ternary conditional directly into the `ensure` clauses to eliminate the matching loop and keep verification decidable.

---

## Part 5: Senior Engineer & Faculty Curveballs (Expert)

### Q9: "What prevents split-brain in MultiPaxos if two leaders believe they are active at the same round?"
**Answer:**
In Sift, round numbers are strictly partitioned across nodes:
```ivy
function round_leader(R: round) : node
```
A node can only attempt leadership in round $R$ if `leader_of(self, R)` holds. Because round ownership is disjoint, two nodes can never be leaders of the same round. If two leaders run competing rounds $R_1 < R_2$, majority quorum intersection ensures that the higher round $R_2$ forces acceptors to abandon $R_1$, preventing $R_1$ from committing any conflicting values.

---

### Q10: "In Raft, why can't a leader commit an entry from an older term by counting replicas?"
**Answer:**
This is the subtle Raft corner case from Section 5.4.2 of the Raft paper (Figure 8):
If a leader tries to commit an entry from a *previous* term merely because it is replicated on a majority of nodes, an older leader could wake up, be elected by a different quorum, and overwrite those entries. 

Raft solves this by requiring that a leader **must replicate at least one entry from its current term** before committing. In [`raft_protocol0.ivy`](file:///c:/Users/s-bas/Sift/Sift/raft/raft_protocol0.ivy), this is enforced in `recv_append_reply` by requiring $t = \text{current\_term}$, which guarantees the Leader Completeness inductive invariant holds.

---

### Q11: "What was your direct contribution to the Sift repository?"
**Answer:**
> *"I performed a complete systems and verification audit of Sift:
> 1. Restored and documented all historical IC3PO synthesized invariants and scratchpad code that were originally disabled across MultiPaxos and Raft.
> 2. Added comprehensive systems, DSLabs, and safety invariant comments across all layers of MultiPaxos and Raft, explaining the exact role of each ghost transition and refinement invariant.
> 3. Verified that every isolate passes cleanly (`OK`) in Ivy 1.7 with Z3.
> 4. Produced the architectural specification (`ARCHITECTURE.md`) documenting the two-tier methodology and EPR logic constraints for the team."*

---

## Key Invariants Comparison Table

| Feature | MultiPaxos ([`paxos_protocol0.ivy`](file:///c:/Users/s-bas/Sift/Sift/multipaxos/paxos_protocol0.ivy)) | Raft ([`raft_protocol0.ivy`](file:///c:/Users/s-bas/Sift/Sift/raft/raft_protocol0.ivy)) |
| :--- | :--- | :--- |
| **Primary Safety Theorem** | Agreement: At most one value decided per slot | Agreement: Committed entries match specification |
| **Core Inductive Invariant** | **Safe Value Invariant:** Abandoned rounds in quorum $Q$ | **Leader Completeness:** Committed term entries preserved in future leaders |
| **Log Structure** | Allows holes (filled with `no_op` in Layer 3) | Strictly contiguous (Log Matching Property) |
| **Leader Qualification** | Any node can lead (adopts highest Phase 1b vote) | Only candidate with up-to-date log can lead |
| **Refinement Layers** | 5 Layers (Spec $\to$ 0 $\to$ 1 $\to$ 2 $\to$ 3 $\to$ System) | 3 Layers (Spec $\to$ 0 $\to$ 1 $\to$ System) |

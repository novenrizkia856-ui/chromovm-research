# ChromoVM: An Adaptive-Assurance Execution Network Anchored to Ethereum

Technical Whitepaper v0.2.1 — September 2026

*(Revision of v0.2. This revision adds batched, self-submitted Ethereum action settlement (§10.5) and separates Ethereum gas economics from the CHR resource account (§13, §14). See accompanying Critical Review and Change Log for the full diff against v0.2 and v0.1.)*

---

## 1. Abstract

Ethereum provides a durable, economically secured state machine, but its consensus overhead makes it a poor fit as a general application runtime: every authoritative state transition inherits block-level latency, throughput, and cost, regardless of whether that particular transition needs Ethereum-grade assurance.

ChromoVM is a network of staked nodes that executes application programs as persistent, deterministic processes outside Ethereum's block production, while using Ethereum as the trust, settlement, and economic-security anchor for those processes. A program advances by consuming ordered messages and producing outputs; most transitions never touch Ethereum directly. When a transition needs stronger guarantees, it can request one of three assurance levels — a staked-committee attestation, an optimistic checkpoint with a fraud-proof-style challenge window, or an immediate validity proof — each enforceable on Ethereum through a small set of contracts.

ChromoVM is not a new blockchain: there is no global block producer or globally ordered ledger in the execution layer. Each program has its own independent ordering, state, and checkpoint history. Ethereum remains the single shared source of stake, identity, settlement, and dispute resolution.

This paper is deliberately conservative about novelty. The three assurance primitives — committee attestation, optimistic challenge, and validity proof — are not new; each has a direct analogue in existing systems (AVS/restaking committees, optimistic rollups, zk-rollups/coprocessors) and this paper names them explicitly rather than implying invention. What ChromoVM proposes is a specific composition: per-transition (not per-application, not per-chain) assurance selection, combined with program-scoped execution that never shares a global sequence. Section 19 gives a rigorous account of what is and is not novel.

---

## 2. Motivation

Applications that need continuous execution — event loops, long-running computation, frequent state updates, background jobs, low-latency interaction — either pay Ethereum's per-transition cost and latency for every state change, or move that logic to centralized infrastructure and use Ethereum only for asset custody and final settlement. The second pattern is common and functional, but it concentrates trust exactly where application behavior is most complex and least auditable.

ChromoVM asks a narrower question than "how do we build a faster chain": *can an application run continuously off-chain while individual state transitions opt into Ethereum-backed assurance only when the application actually needs it, at the transition level rather than the application level?*

---

## 3. Problem Statement

Given an application that requires (a) frequent, low-latency state updates, (b) persistent process state across many interactions, and (c) occasional high-value actions that require strong, Ethereum-enforceable correctness guarantees — design an execution and settlement architecture that:

- does not require Ethereum block inclusion for ordinary progress,
- does not require a new independent blockchain with its own global consensus,
- gives the application a way to obtain Ethereum-anchored assurance for the specific transitions that need it, and
- makes the security model of every assurance level explicit rather than asserted.

---

## 4. Design Goals

1. **General application execution** — not constrained to the EVM's contract-call/transaction programming model.
2. **Existing toolchains** — compile with mature, off-the-shelf compilers and debuggers.
3. **Determinism** — identical machine state and ordered inputs always produce identical resulting state and outputs.
4. **Persistent processes** — a program yields, waits, and resumes from exact machine state, rather than being re-invoked from scratch per call.
5. **Low-latency ordinary progress** — most transitions do not require an Ethereum transaction or block.
6. **Ethereum-backed settlement on demand** — high-value or externally-consequential transitions can reach assurance enforceable on Ethereum.
7. **Horizontal scaling** — independent programs do not contend for one global execution sequence.
8. **Minimal new consensus** — reuse Ethereum's consensus rather than build a second one.
9. **Metered, attributable resource use** — compute, storage, bandwidth, and proof work are measured and billed per transition.
10. **Replaceable implementation components** — networking, storage, and proving backends can change without changing the protocol's logical guarantees.
11. **Self-sovereign settlement** — realizing a program's own Ethereum-side actions must not require trusting, or paying, the node network. The program owner (or anyone) should be able to submit those actions directly, in ETH, independent of CHR.

---

## 5. Non-Goals

Stated explicitly, because their absence in v0.1 let scope drift:

- ChromoVM does **not** aim to be a general-purpose L1 or L2 blockchain, and does not compete on "TPS" as an undifferentiated global metric — its scaling claim is per-program, not aggregate-chain.
- ChromoVM does **not** aim to make off-chain computation free, or eliminate the cost of proof generation, storage, or bandwidth.
- ChromoVM does **not** aim to make FAST-tier state Ethereum-equivalent in trust. FAST is explicitly weaker, by design, and this must never be described as "Ethereum-secured."
- ChromoVM does **not** aim to solve the general oracle problem. External data enters as an explicit, attributable input; the protocol does not make an untrustworthy source truthful.
- ChromoVM does **not** aim to replace the EVM as a settlement environment. Ethereum's deterministic, atomic, shared-state execution remains the correct model for the applications it already serves well (most DeFi); ChromoVM targets a different constraint profile (persistent, asynchronous, computationally heavier applications), not a strictly superior one.

---

## 6. System Overview

Four layers, with different trust characteristics:

```
+--------------------------------------------------------------+
| Ethereum                                                      |
| Registry | Stake/Slashing | Balances | Checkpoints | Bridge   |
| — authoritative for membership, collateral, settled state,    |
|   and Ethereum-side asset custody                             |
+------------------------------+---------------------------------+
                                |
+-------------------------------v-------------------------------+
| Chromo Node Network                                            |
| Schedulers | Executors | Provers | Storage replicas | Challengers |
| — authoritative ONLY within the assurance level a given        |
|   transition selected                                          |
+------------------------------+---------------------------------+
                                |
+-------------------------------v-------------------------------+
| ChromoVM Execution Machine                                     |
| Deterministic RV32IM process | ECALL runtime ABI | Metering    |
| — a shared transition function every implementation must match |
+------------------------------+---------------------------------+
                                |
+-------------------------------v-------------------------------+
| Applications                                                   |
| Persistent processes: games, backends, autonomous services,    |
| computational protocols                                        |
+------------------------------------------------------------------+
```

Ethereum is authoritative for protocol membership, collateral, balances, *settled* checkpoint status, and execution of authorized on-chain actions. The node network is authoritative only within the assurance level a program's transition selected — this is the load-bearing sentence in the whole architecture, and every later section (§10–17) has to justify it precisely.

---

## 7. Protocol Architecture and Trust Boundaries

| Layer | Records / does | Does NOT record / do | Trust required |
|---|---|---|---|
| Ethereum | Node stake, program registry, code/manifest hashes, settled checkpoint roots, action-intent execution, vault custody | Program execution, program state contents, message contents | Ethereum consensus (assumed externally secure) |
| Chromo node network | Ordering, execution, state replication, receipts/certificates, proof generation, availability | Nothing final on its own — its statements only carry the weight of the assurance level requested | Depends on tier: committee honesty (FAST), challenger liveness + proof soundness (OPTIMISTIC), proof system soundness alone (PROVEN) |
| ChromoVM machine | The deterministic transition function itself | — | Correctness of the shared spec; a divergence between two conforming implementations is a protocol-breaking bug, not a policy choice |
| Applications | Their own program-scoped logic and state | Nothing about other programs by default (inter-program access is explicit, via messaging — §10) | Whatever the application's own manifest declares as its minimum acceptable assurance per input class |

This table is the answer to "what does Ethereum record and what does it not record" (Required Question 8) and should be read before the sections that follow, since §10–17 assume it.

---

## 8. Execution Model

Application code targets RV32IM (RISC-V 32-bit integer, with multiply/divide). RISC-V is a pragmatic implementation choice, not the protocol's core claim: it is chosen because (a) mature C/C++/Rust/LLVM toolchains already target it, (b) the base integer ISA is small enough to interpret, meter, and prove efficiently, and (c) existing RISC-V proving systems already share its execution model, which lowers the cost of building the PROVEN tier. A different execution backend — a different ISA, or a higher-level deterministic bytecode — would not invalidate the protocol; RV32IM is the current implementation target, not a claim about what makes general-purpose decentralized execution possible.

The machine is little-endian, single-threaded per process, and excludes floating-point, vector, privileged, and atomic instructions from the consensus-critical execution contract; deterministic software floating point may be used by applications that need it. Parallelism comes from running many independent processes, not from nondeterministic threading inside one.

**Execution slices.** A process is never run for an unbounded duration in one protocol transition. A slice is a function `T(S_n, I_n, L_n) → (S_n+1, O_n, U_n)`: given a state, an ordered input batch, and a resource limit, it produces a new state, outputs, and consumed units. A slice ends on explicit yield, on waiting for input, on a checkpoint request, on exhausting its resource limit, on program exit, or on a trap/fault. This is what makes a continuously running program representable as a sequence of bounded, replayable, and provable transitions rather than one long-lived black box.

**Runtime boundary.** All interaction with the decentralized runtime — messaging, storage, chain reads, chain actions, checkpoint requests — goes through a fixed set of ECALL syscalls with a versioned numeric ID and register-based argument convention (full enumeration in the companion ABI reference, not reproduced here). No host operating-system call, wall-clock read, filesystem access, local RNG, or environment value is exposed to application code. Any external fact — time, an HTTP result, an Ethereum observation, randomness — must arrive as an explicit, committed input, because determinism requires that replay from the same inputs always produces the same outputs. This single rule is what makes checkpointing, dispute resolution, and proof generation possible at all; it should be read as a security-model requirement, not an implementation detail.

---

## 9. State Model

**Is execution stateful?** Yes, and persistently so — this is the point of the design. Each process carries `pc`, integer registers, a `memoryRoot` (linear memory commitment), a `storeRoot` (persistent key/value store commitment), input/output sequence counters, a status (`RUNNABLE | WAITING | PAUSED | EXITED | FAULTED`), and cumulative compute units.

**Commitment.** `stateRoot = H(domain, vmVersion, codeHash, pc, registersHash, memoryRoot, storeRoot, inboxSeq, outboxSeq, status, units)`, canonically serialized and versioned; two conforming implementations must produce byte-identical input to the hash.

**Who stores state, and how is it replicated?** Assigned executor and storage-replica nodes hold the current state and enough history to reconstruct it. Linear memory is page-committed under a Merkle tree so a node can fetch only needed pages while still committing to the whole address space; durable application records live in a separate sparse-Merkle-tree-backed store, distinct from transient memory, because it needs different proof and partial-retrieval properties.

**How are versions identified and stale states prevented?** Every state transition is sequence-numbered (`inboxSeq`/`outboxSeq` plus checkpoint sequence). A receipt or checkpoint that doesn't match the expected pre-state root and sequence is a different, non-mergeable statement — there is no implicit "latest wins."

**Recovery.** A node reconstructs a program from (1) the latest accepted checkpoint, (2) the code package, (3) a state snapshot matching that checkpoint's root, and (4) ordered input batches since that checkpoint. If a snapshot is unavailable but an earlier one plus all subsequent inputs is, state is recomputed by replay. This is only as good as the availability obligations in §15.4 — a correct root with no reachable data behind it is not recoverable, and the paper should not claim otherwise.

**What happens when a node disappears?** Its role (scheduler or committee slot) is reassigned at the next epoch boundary via the same rendezvous-hash procedure used for initial assignment (§16.3); a program's state is not lost because it was never uniquely held by one node — storage replicas exist independently of the currently active executor set, subject to the replication factor the program's manifest configures.

---

## 10. Asynchronous Messaging

No path in this protocol blocks execution on a synchronous Ethereum call-and-wait. This is a deliberate architectural constraint, not an incidental property, and it applies in all four possible directions:

**User/program → process (input messages).** Signed messages (`destination, sender, senderNonce, payloadHash, payload, requestedAssurance, expiry, signature`) are ordered into batches by an assigned scheduler and independently verified by the executor committee for signature validity, nonce correctness, and batch continuity before execution. `senderNonce` gives replay protection.

**Delivery semantics, stated explicitly.** Message delivery is **at-least-once**, not exactly-once: a client may resend to multiple relay nodes for censorship resistance (§15.5), and a scheduler could in principle include a batch containing a message twice. Applications must treat message handling as idempotent with respect to `(sender, senderNonce)`; the runtime rejects a batch containing a stale or duplicate nonce for a given sender, which converts at-least-once delivery into effectively-once processing at the application boundary. This guarantee — and its limits — was previously implicit and is now a stated protocol property.

**Process → process (application-to-application).** An output message references `sourceProgram, sourceCheckpoint, sourceOutboxSeq, destinationProgram, payloadHash, assuranceLevel`. The destination declares, in its own manifest, the minimum source assurance it will accept — a game process might accept a FAST-level message from another game process, while a value-bearing process might require the source to be at PROVEN. This is the protocol's actual answer to "how does this differ from an ordinary contract call": an ordinary call is atomic and synchronous within one execution context; a Chromo inter-program message is asynchronous, assurance-tagged, and the receiver — not the protocol — decides what it trusts.

**Process → Ethereum (action intents).** A `CHAIN_ACTION` ECALL does not itself produce an Ethereum transaction. It produces an `ActionIntent{programId, checkpoint, nonce, target, value, calldataHash, calldata, minimumAssurance}` — a committed output of the process, sitting in a **pending** state until someone submits it to `ActionBridge`. `ActionBridge` executes it only if the referenced checkpoint has reached the required assurance, the nonce is unused, the vault policy permits the call, the payload matches the committed output, and value/rate limits are satisfied. The executor network never holds unilateral custody of Ethereum assets — this is enforced by construction, not by trust in node operators. Note also what is *not* on Ethereum: only the ECALLs whose semantics require an on-chain effect — `CHAIN_ACTION` (via `ActionBridge`) and `CHECKPOINT` (via `CheckpointManager`) — have any corresponding contract logic. The rest of the ECALL surface (`STORE_*`, `MESSAGE_*`, arithmetic, memory) has no on-chain representation whatsoever; Ethereum never sees those instructions execute. §10.5 below specifies how a pending `ActionIntent` actually gets submitted, and by whom.

### 10.5 Batched, Self-Submitted Action Execution

v0.2 left one thing implicit: once an `ActionIntent` is eligible (its checkpoint has reached `minimumAssurance`), *who* pays the Ethereum transaction that actually invokes `ActionBridge`? Left unspecified, the natural reading is "a node does it, and gets reimbursed somehow" — which quietly reintroduces exactly the kind of dependency on the node network for a user's own settlement that §11 and the custody property above were trying to avoid, and ties an Ethereum-gas cost to the CHR resource-accounting model without a stated mechanism for the conversion. This revision resolves it directly, based on a property the design already had without stating it.

**`ActionIntent` carries no signature from the program owner.** Its authority does not come from being signed by anyone — it comes from being a deterministic, committed output of the process's own execution, referenced by a checkpoint whose assurance is independently verifiable on Ethereum. This means submission of `ActionBridge.execute(...)` is, and always was, **permissionless**: the calldata is public (it is part of a committed output), and anyone who presents it along with a valid reference to its assured checkpoint may call it. The protocol does not need a new rule to allow self-submission — it needs to *use* this property deliberately instead of leaving it as an accident of the data model.

**Batched multicall interface.** `ActionBridge` exposes `executeBatch(ActionIntent[] intents, bytes[] inclusionProofs)`. Each intent in the batch is checked and executed independently — under a try/catch pattern that emits a per-intent success or failure event rather than reverting the whole call on one bad or expired intent. This changes nothing about *what* is checked (the same five conditions as the single-intent case, per intent), only how many intents are packaged into one Ethereum transaction. `inclusionProofs[i]` is a Merkle proof against the relevant checkpoint's `outputRoot`, required for OPTIMISTIC-settled checkpoints; it is unnecessary for PROVEN checkpoints, since the validity proof already commits to the full output set `ActionBridge` checks against.

Ordering within a batch is constrained only where it matters for correctness: intents from the *same* program must preserve their `nonce` order (across a batch or across sequential batches); intents from different programs, or different owners entirely, have no relative ordering requirement and can be freely interleaved purely for gas efficiency.

**Chrono Core — reference client.** The reference implementation for self-submission is **Chrono Core**: a client the program owner runs locally, holding the owner's own Ethereum private key. It subscribes to the pending-action outbox and checkpoint status of the programs it controls (over the topics in §16), assembles a batch according to a local policy — by count, by timer, or on manual trigger, none of which is consensus-critical — and submits `executeBatch` directly, signing with the owner's key and paying gas from the owner's own ETH balance. Chrono Core is a convenience and a self-sovereignty guarantee, not a protocol requirement: because submission is permissionless (above), any other relayer — including a Chromo node, or a third party that batches multiple owners' pending actions together for shared gas efficiency — may equally submit the same intents. An owner who never runs Chrono Core is not stuck; they are simply depending on someone else to eventually submit their already-valid, already-committed action, with no trust implication either way since the contract enforces correctness independent of who calls it. Fee-sharing or tipping incentives for third-party batch relayers are an application/market-layer concern this paper does not specify (flagged in §21).

**Economic separation, stated plainly.** Ethereum gas for `executeBatch` is paid, in ETH, by whoever submits the transaction — typically the program owner via Chrono Core. It is entirely outside the CHR-denominated resource account in §13. A program that never emits `CHAIN_ACTION` never causes an Ethereum transaction on this path at all — not one, not batched, not ever. CHR is not required to submit a program's own actions; it remains required for registration (§14.3) and for funding the resource account that pays the node network for off-chain execution, storage, bandwidth, and proof-generation service (§13) — a distinct cost, from a distinct payer relationship, for a distinct thing.

**Ethereum → process (chain reads).** A process cannot treat a local RPC read as deterministic truth. Ethereum information enters as a chain input carrying a canonical block reference plus a declared confirmation policy — observed, K-confirmed, finalized, or accompanied by a state/storage proof — chosen per input class in the program's manifest. A low-latency application can react to a recent observation while a high-value decision requires stronger finality; this tradeoff is explicit and configurable, not fixed.

**Ordering.** There is no global transaction order. Each program has its own strictly ordered input sequence; `Program A: 1→2→3→...` and `Program B: 1→2→...` share no block height. This is the mechanism, not just the claim, behind the horizontal-scaling design goal.

---

## 11. Ethereum Integration

Ethereum's role, stated precisely rather than as "the blockchain where the computation happens":

- **Trust and economic-security anchor** — node stake is locked and slashable on Ethereum (§16).
- **Settlement layer** — checkpoint roots are only *final* once accepted by Ethereum contracts under the rules of the relevant assurance tier.
- **Verification/commitment layer** — the `ProofVerifier` contract is where a validity proof is actually checked; Chromo does not implement its own verification consensus.
- **Custody layer** — Ethereum-native assets tied to a program live in a `ProgramVault`, released only through a verified `ActionIntent`.

What Ethereum explicitly does **not** do: execute application logic, store program state contents, or provide low-latency feedback for ordinary application progress. Conflating "anchored to Ethereum" with "executed on Ethereum" is the single most common source of overclaiming in this design space, and this paper treats the distinction as load-bearing.

**Congestion.** If Ethereum base fees spike or block space is scarce, OPTIMISTIC challenge windows and PROVEN response deadlines are defined in terms of **finalized epochs, not wall-clock time** — a proposer's obligation to get a proof-verification transaction included is bounded by a deadline expressed in Ethereum finalized slots from the challenge, with the contract extending the deadline if Ethereum itself is not producing finalized blocks (a liveness fault of Ethereum, not of the proposer). A proposer that fails to get inclusion despite Ethereum being live is treated as failing to prove and is slashed under the normal rule; this is a deliberate design choice, but it is now explicit rather than implicit in v0.1.

**Reorgs.** A checkpoint submission transaction is not "settled" the moment it's included in a block — it inherits Ethereum's own probabilistic-then-finalized status. `CheckpointManager` does not advance a program's authoritative settled root until the submitting block itself is finalized under Ethereum consensus; if the containing block is reorged out before finality, the checkpoint simply never became settled and must be resubmitted. This removes the previously-unstated question of "what if the checkpoint's block disappears."

---

## 12. Verification and Consensus

This is the section that answers "if a decentralized node network executes arbitrary programs, how does the system know the result is correct" — and it should be read alongside §19, because none of the three mechanisms below is novel on its own.

### 12.1 Threat model

Adversary controls up to some fraction of stake within a program's assigned committee, may withhold or reorder messages it schedules (but cannot forge signatures), may attempt to submit an invalid checkpoint, and may attempt to prevent a valid challenge or proof from reaching Ethereum. Honest majority/minority assumptions differ per tier (below) and are stated per-tier, not once for the whole system, because that is the actual security model — a blanket "secured by Ethereum" is not a claim this paper makes.

### 12.2 FAST — staked committee attestation

A FAST receipt is accepted once a signing threshold of the program's assigned executor committee signs the same receipt. This is structurally an AVS-style attestation (§19.7) and its security reduces entirely to: *the signing threshold for this specific assignment does not collude or get compromised.* It provides no protection against a colluding threshold beyond after-the-fact slashing if a later proof shows the attested transition was invalid.

**Honest disagreement (new in this revision).** If the committee genuinely splits — e.g. a network partition causes two sub-groups to observe different message orderings — two differently-signed receipts for the same sequence may both be individually well-formed. Neither is a certificate until it reaches the signing threshold. The protocol resolves this by making the batch, not the receipt, the unit of agreement: the scheduler's signed batch ordering is canonical input, and an executor that signs a receipt over a *different* input batch has signed a statement about a different (non-canonical) transition, which cannot form a certificate regardless of signature count. A partitioned minority simply fails to reach threshold and produces no FAST receipt for the affected sequence until connectivity is restored — this trades liveness for the absence of ambiguous finality, which is the correct tradeoff for a tier whose whole purpose is speed rather than strong correctness. This resolves Required Question 11 for the FAST tier; the OPTIMISTIC and PROVEN tiers resolve it structurally, since they are backstopped by Ethereum (below).

### 12.3 OPTIMISTIC — checkpoint + fraud-proof-style challenge

A proposer posts a transition commitment and bond to Ethereum. A challenge window opens; if unchallenged, it settles. If challenged, the proposer must produce a validity proof for exactly the disputed transition before a (finality-bounded, §11) deadline, or be slashed. A successful challenger is paid from the slashed bond; a challenger who forces an unnecessary proof compensates the proposer from their own bond.

This is a fraud-proof pattern, structurally the same family as optimistic rollups (§19.2), with one deliberate difference: instead of an on-chain interactive dispute game that narrows down to a single disputed instruction, a challenge triggers a single full re-execution proof of exactly the disputed transition. This avoids implementing a custom interactive RISC-V dispute interpreter on Ethereum, at the cost of making the *unlikely* dispute path itself relatively expensive — a tradeoff that is reasonable only if disputes are rare in practice, which is an empirical claim this paper does not yet have data to support (flagged in §21).

### 12.4 PROVEN — validity proof at submission

A checkpoint includes a validity proof at submission time; Ethereum's `ProofVerifier` checks it against the code commitment, pre-state root, input commitment, post-state root, VM version, and metering result before accepting it. Correctness reduces to: soundness of the proof system, correctness of the Chromo transition circuit/guest matching the reference implementation, and availability of the witness data needed to generate the proof. This is architecturally a general-purpose zkVM coprocessor call (§19.4), applied to Chromo's own transition function rather than to Ethereum's.

### 12.5 Monotonicity

Assurance only strengthens: `FAST → OPTIMISTIC → PROVEN`. An action that requires PROVEN assurance can only reference a state whose full unproven ancestry back to the last already-PROVEN state (or genesis) has been proven — proving only the newest slice does not retroactively upgrade its unproven ancestors. If an OPTIMISTIC ancestor is later rejected, dependent descendants are invalidated unless re-executed from the last valid ancestor.

---

## 13. Resource Accounting

Ethereum gas is not reused, because Chromo's cost surface is different: gas prices EVM opcodes and Ethereum state I/O under Ethereum's own contention; Chromo needs to price RV32IM instruction execution, persistent-store access, message emission, blob storage, bandwidth, and proof generation — a materially different resource mix with its own contention (executor capacity, storage replication, prover throughput), not Ethereum's block space. Reusing gas would price the wrong scarcity.

**Measurement.** Every canonical instruction and runtime operation has a deterministic compute-unit (CU) weight, defined in a versioned protocol table (illustrative example, non-normative):

| Operation | Example weight |
|---|---|
| Simple integer instruction | 1 CU |
| Memory load/store | 2 CU |
| Multiply/divide | 3–8 CU |
| Persistent store access | fixed + byte cost |
| Message emission | fixed + byte cost |
| Blob operation | byte + availability cost |
| Proof generation | separate proof fee |

A receipt commits to the exact CU count consumed, which makes billing replayable and, at the PROVEN tier, provable — a node cannot charge an amount that isn't a deterministic output of the same transition anyone could recompute.

**Who pays, and pricing.** A program funds a resource account on Ethereum. Nodes accrue deterministic claims against it and settle them through accepted checkpoints — not per instruction, per checkpoint — which converts many off-chain application actions into one Ethereum-side settlement. Pricing for CU/storage/bandwidth is a protocol parameter table, not fixed at issuance; how it should be set (fixed schedule vs. market-based) is left as an open question in §21 rather than answered with unjustified confidence.

**Knowing expected cost.** A program's manifest and the versioned CU table let a developer estimate cost before deployment; a deployment quote is required before registration (§14) so a program cannot register without a known minimum reserve.

**Abuse/DoS prevention.** Execution is bounded per slice by an explicit resource limit `L_n` (§8); a program that exceeds available balance moves to `PAUSED_OUT_OF_FUNDS` rather than accruing unbounded unpaid work; deployment itself requires a funded reserve before a registry transaction is even accepted.

**Relationship to Ethereum gas (new in v0.2.1).** The CU-based charge above prices the node network's *off-chain* service — execution, storage, bandwidth, and proof generation — and is settled from the CHR resource account. It is a separate cost, paid to a separate party, from the Ethereum gas a submitter pays out of pocket to call `ActionBridge.executeBatch` (§10.5) or to post a checkpoint (§12). Neither of these ETH costs is currently reimbursed by the protocol; a checkpoint proposer's real ETH gas cost is expected to be recovered as part of the `proofFee` / checkpoint-fee line item, priced to cover it, but this pass-through is a policy choice for the fee-table design, not something this paper derives from first principles — flagged as open in §21.

This also surfaces a gap that predates this revision and is worth stating rather than leaving implicit: CU billing settles *at checkpoints*, because `CheckpointManager` is the only place a charge is recorded against Ethereum. A program that stays entirely on the FAST tier and never requests an OPTIMISTIC or PROVEN checkpoint therefore never generates a settlement event at all — meaning today's design implicitly requires *some* minimum periodic checkpointing purely to let nodes get paid, independent of whether the application itself needs the stronger assurance a checkpoint provides. This is distinct from, but adjacent to, the action-gas question this revision resolves, and is listed as its own open item in §21.

---

## 14. Economic Security

**Functions of the native token (CHR).** Node collateral; payment/accounting unit for execution-network resources; the unit slashing and rewards are denominated in. CHR is not required to be the unit end users interact with — sponsors or gateways may convert other assets into CHR to fund a program's resource account.

**What CHR is not for (new in v0.2.1).** CHR is not a gas token for Ethereum. Submitting a program's own on-chain actions (§10.5) is paid in ETH, directly, by whoever submits the transaction — that cost never touches the resource account and is not denominated in CHR. CHR's scope is deployment/registration and the ongoing off-chain resource account in §13; it is not a universal settlement unit for everything a program does that eventually touches Ethereum.

**Staking.** `requiredStake = max(baseStake, securityFactor × assignedEconomicExposure)` — collateral scales with the actual value a node's assignment could put at risk, not a flat fee. A flat floor alone (2,000 CHR in the illustrative parameterization) exists only for spam resistance, not as the primary security bound.

**Slashing.** Reserved for objectively provable violations: signing a FAST receipt later disproved, failing a proof obligation after proposing an OPTIMISTIC checkpoint, or equivocation under protocol rules. Ordinary downtime reduces assignment eligibility and rewards, not stake — slashing is for provable dishonesty, not unavailability.

**Fees.** For a settled charge `F`: 80% to the node-service pool (split by measured role — scheduling, execution, proving, storage), 20% burned. Challenge rewards are funded primarily from slashed invalid-proposer bonds, so honest execution isn't taxed to subsidize disputes. This split is a policy parameter, changeable without touching execution semantics — stated explicitly so it is not mistaken for a consensus-critical constant.

**Yield.** Node return = execution + proof + storage fees + temporary bootstrap emissions − slashing losses − operating costs. No fixed APY is promised; emissions may be calibrated toward a target range early on, but long-run security is intended to be paid by real usage, not perpetual subsidy. This is stated as intent, not as a guarantee, because the paper does not yet have the usage data to back a stronger claim.

**Genesis allocation** (illustrative, non-normative): founder treasury 30M, foundation treasury 10M, network distribution 20M — total 60M CHR. This is economic policy, not protocol correctness, and token ownership does not confer execution authority or one-token-one-vote control over application state.

---

## 15. Failure Recovery

Addressed as concrete cases, not general assurance:

**Node crash.** A node's role (scheduler or committee seat) is reassigned at the next epoch via the standard rendezvous-hash procedure. A replacement node reconstructs state exactly as in §9 (recovery): last checkpoint + snapshot + replay of subsequent input batches. There is no ambiguity about what to resume from, because the checkpoint sequence number *is* the resume point by construction.

**Network partition.** Handled per assurance tier: FAST requires threshold agreement and simply fails to produce a certificate for a partitioned minority (§12.2) rather than producing a conflicting one; OPTIMISTIC/PROVEN settlement is anchored to Ethereum, which is assumed to remain live and correct outside the partition.

**Dropped, duplicated, or reordered messages.** Dropped messages are re-sent by the client (delivery is at-least-once, §10); duplicates are rejected via `senderNonce`; reordering within a batch is constrained by the scheduler's signed batch, which the committee verifies for continuity before execution — a batch that violates ordering rules is not valid input.

**Byzantine nodes.** Bounded by stake and slashing at the FAST/OPTIMISTIC tiers, and by proof soundness alone at PROVEN (§12).

**Execution interruption / incomplete computation.** A slice either completes and commits, or it doesn't — there is no partially-committed intermediate state, because a slice's post-state is only written once `T(S_n, I_n, L_n)` fully returns (§8). A crash mid-slice simply means the slice is re-run from `S_n` by whichever node is assigned next.

**Ethereum unavailability or reorg.** Covered in §11: checkpoint settlement waits for Ethereum finality; deadlines are finality-bounded, not wall-clock; temporary Ethereum liveness failure extends deadlines rather than penalizing proposers.

**Data availability.** A correct state root with no reachable data behind it cannot be recovered — this is stated as a real limitation, not solved by the protocol. Programs configure a replication factor; storage replicas are paid to retain code, recent snapshots, input batches, and outputs needed by dependents. Attestations of availability affect rewards but are not proof of long-term possession; high-value applications should use independent, external content-addressed persistence in addition to the active committee.

---

## 16. Node Network

**Roles.** Scheduler (orders and batches messages), Executor (replays batches, executes ChromoVM, signs receipts), Prover (generates validity proofs on request), Storage replica (retains packages/snapshots/batches), Challenger (observes OPTIMISTIC checkpoints and disputes invalid ones). No role produces blocks for a new chain — this bears repeating because it is the property that lets §19 distinguish ChromoVM from an appchain.

**Networking.** libp2p for identity, transport, discovery, request/response, and pub/sub, topic-scoped per program/role; large blobs are fetched by content identifier rather than gossiped in full.

**Assignment.** At each epoch, nodes are ranked for a program via deterministic rendezvous hashing over `(programId, assignmentEpoch, finalized Ethereum seed, nodeId)`. The top-ranked eligible node is scheduler; the next N form the executor committee, subject to stake/capacity requirements. This seed is used to *distribute work*, not as a cryptographic randomness source for applications. A failed scheduler is replaced by the next ranked eligible node after timeout.

This mechanism has an open question, not resolved here: whether a well-resourced adversary can grind stake or identity across epochs to bias future assignment toward controlling a specific high-value program's committee. This is listed as an open research problem in §21 rather than dismissed.

---

## 17. Security Model

Stated per-guarantee, with the assumptions required to obtain it — a deliberate departure from any single blanket "secured by Ethereum" claim.

| Guarantee | Requires | Fails if |
|---|---|---|
| FAST receipt correctness | Signing threshold of assigned committee is honest for this assignment | Threshold collusion; assignment successfully targeted (§16, open) |
| OPTIMISTIC settlement correctness | Ethereum live; invalid checkpoints observable; ≥1 economically motivated challenger reachable in time; proof backend sound; proposer cannot indefinitely censor their own proof tx | Challenger absence/censorship; proof-system unsoundness |
| PROVEN correctness | Ethereum contract execution correct; proof system sound; Chromo circuit/guest matches reference spec; commitment hash collision-resistant; witness data available at proof time | Circuit/spec mismatch (a determinism bug, §17.6); witness unavailability |
| Asset custody | `ActionBridge` policy logic correct; vault contract correct | Bug in bridge/vault contract — no assurance tier compensates for this |
| Censorship resistance | Users can reach ≥1 honest relay; forced-inbox path available for critical messages | All reachable nodes censoring simultaneously (mitigated, not eliminated, by forced inbox) |
| Data availability | Configured replication factor actually retains data | Replica set collectively drops data below what's needed to recompute state (§15) |

**Determinism as a security property.** A mismatch between two conforming VM implementations is consensus-critical, not a bug report. VM semantics, syscall ABI, serialization, hash domains, metering schedule, and proof adapter are all versioned together; a running program stays pinned to its declared version until it explicitly migrates, so an upgrade cannot silently change the meaning of an already-deployed program's history.

**Censorship.** A scheduler can delay but not forge messages. Users may broadcast to multiple nodes; an unresponsive scheduler is replaced (§16). A program may expose an Ethereum forced-inbox path for messages that must be censorship-resistant at Ethereum's cost/latency; this path is opt-in, not the default, because most traffic does not need it.

---

## 18. Developer / Application Model

"Feels like Web2" is a **goal for tooling**, not a protocol guarantee — v0.1 mostly avoided the stronger claim, and this version states the boundary directly. The protocol primitives that make that UX goal achievable, without the protocol itself promising it, are:

- Asynchronous execution and persistent sessions (native — §8, §10).
- Account abstraction / session keys / delegated permissions — not yet specified at the protocol level; left as an application-layer or future-protocol concern (§21).
- Application-level identity separate from raw signing keys — same status.
- A resource sponsor model, already present (§14 of v0.1 / deployment lifecycle), letting end users interact without holding CHR directly.
- Reliable messaging with explicit delivery semantics (§10) as a building block for higher-level developer tooling (retries, dedup libraries) that the protocol does not itself provide.
- **Chrono Core** (new in v0.2.1, §10.5) — a reference client pairing a local VM view with the owner's own Ethereum wallet, so a developer's default path for realizing their program's own on-chain actions never depends on a third party or on CHR.

**Example.** A persistent game-world process consumes signed movement messages, updates state without waiting on Ethereum, and uses FAST receipts for ordinary movement, periodic OPTIMISTIC checkpoints for world-state durability, and a PROVEN checkpoint only when an item is withdrawn as an Ethereum asset via an `ActionIntent`, which the player's own Chrono Core instance batches and submits directly once assurance is reached. If the same game never withdraws anything to Ethereum, it never causes a single Ethereum transaction beyond its own one-time deployment — ordinary play stays entirely on FAST, off-chain. This illustrates the adaptive-assurance idea concretely, but it is an example, not a proof that the pattern generalizes to every application class claimed in §4 — that generalization is untested (§21).

---

## 19. Prior Art and Comparisons

This section exists because a design that reuses well-known primitives without naming them invites exactly the wrong kind of scrutiny. For each category: what it solves, how it works, where ChromoVM is similar, where it differs, what tradeoff that difference introduces, and why ChromoVM should exist alongside it rather than being replaced by it.

### 19.1 Ethereum post-Merge L1

*Solves:* shared, replicated, economically secured state with strong finality.
*Similar:* ChromoVM's entire trust anchor is Ethereum consensus; it adds nothing to the base security assumption.
*Differs:* Ethereum executes and finalizes every transaction on-chain; ChromoVM executes off-chain and only anchors selected transitions.
*Tradeoff:* weaker-than-Ethereum guarantees for anything below PROVEN, in exchange for latency and cost that don't scale with Ethereum block time.
*Why not just Ethereum:* because persistent, high-frequency, non-atomic application state does not need per-update Ethereum finality, and paying for it anyway is the exact inefficiency this paper starts from (§2–3).

### 19.2 Optimistic Rollups

*Solves:* scaling execution while inheriting L1 security via fraud proofs during a challenge window.
*Similar:* ChromoVM's OPTIMISTIC tier is a fraud-proof-style challenge mechanism, structurally in the same family.
*Differs:* an optimistic rollup maintains one global, chain-wide ordered ledger and state; ChromoVM has no chain-wide ledger — each program has its own independent ordering and its own checkpoint history, and different programs can use different assurance tiers for different transitions simultaneously.
*Tradeoff:* rollups get a single coherent global state useful for atomic cross-application composition; ChromoVM trades that away for per-program independence and heterogeneous per-transition assurance.
*Why not just use a rollup:* a rollup still commits to one verification model for (essentially) everything on it; ChromoVM's premise is that a single application legitimately wants three different assurance levels for three different classes of its own state.

### 19.3 ZK Rollups

*Solves:* scaling with validity proofs instead of a challenge period, giving faster finality than optimistic designs.
*Similar:* ChromoVM's PROVEN tier is architecturally a validity-proof settlement, same category.
*Differs:* same structural point as 19.2 — global ledger vs. program-scoped state — plus ZK rollups typically prove *every* transition, where ChromoVM proves only the transitions that request PROVEN assurance, leaving the rest at cheaper, weaker tiers.
*Tradeoff:* ChromoVM avoids paying proof cost on transitions that don't need it, at the cost of those transitions carrying weaker guarantees until/unless upgraded.
*Why not just use a ZK rollup:* proving every transition is the right choice when uniform strong guarantees are the product; it is unnecessary cost when most of an application's state (chat, movement, UI updates) doesn't need it.

### 19.4 Cartesi

*Solves:* running large, complex off-chain computation (a full Linux/RISC-V environment) with Ethereum-anchored dispute resolution — the closest prior art to ChromoVM's execution model.
*Similar:* off-chain RISC-V execution, Ethereum as settlement/dispute layer, avoidance of executing the whole program on-chain.
*Differs:* Cartesi's dispute resolution is an on-chain interactive fraud-proof game that bisects down to a single disputed instruction; ChromoVM's OPTIMISTIC tier instead requires a full re-execution proof of the disputed transition on challenge (§12.3), avoiding an on-chain interactive interpreter at the cost of a more expensive (but rarer, if the design assumption holds) dispute path. ChromoVM also adds a FAST tier below Cartesi's model (for state that doesn't need dispute-grade assurance at all) and native program-scoped, assurance-tagged inter-program messaging (§10), which is not part of Cartesi's core model.
*Tradeoff:* ChromoVM's dispute path is architecturally simpler but each dispute is more expensive; Cartesi's interactive game is cheaper per dispute step but more complex to implement and verify on-chain.
*Why not just use Cartesi:* Cartesi does not offer a sub-dispute-grade fast tier or a native cross-program messaging/assurance-tagging model — an application that wants "instant local feedback, occasional disputable checkpoints, rare proven settlement" as three coexisting levels for the same program does not get that from Cartesi's single-tier model. This is the most honest and important comparison in this paper, and if further analysis shows the difference doesn't hold up in practice, that should be reported rather than hidden.

### 19.5 General-purpose zkVM coprocessors (RISC Zero, Succinct, Axiom-style)

*Solves:* verifiable off-chain computation, callable from an on-chain contract, without executing the computation on-chain.
*Similar:* ChromoVM's PROVEN tier is exactly this pattern, applied to Chromo's own transition function (§12.4).
*Differs:* a coprocessor call is typically stateless and request/response — compute once, verify, done. ChromoVM processes are long-lived, stateful, and message-driven, with the proof tier being one of three selectable assurance levels rather than the only mode of operation.
*Tradeoff:* a coprocessor is simpler and sufficient for one-shot verifiable computation; it is not designed for continuous, cheap, frequent state progress, which is what FAST/OPTIMISTIC exist to provide.
*Why not just use a coprocessor:* because most of what a persistent application does (movement, chat, incremental updates) does not need per-call proof generation, and paying proof cost for all of it defeats the latency/cost goals in §4.

### 19.6 Application-specific chains (appchains)

*Solves:* dedicated throughput and customized execution/consensus rules for one application.
*Similar:* both give an application its own execution environment separate from Ethereum's shared block space.
*Differs:* an appchain has its own independent consensus and validator set (own security budget, own liveness risk); ChromoVM has no independent consensus layer at all — security for anything above FAST comes entirely from Ethereum.
*Tradeoff:* an appchain can tune consensus fully to its application at the cost of bootstrapping and maintaining independent economic security; ChromoVM inherits Ethereum's security for its strong tiers but is constrained by Ethereum's finality and fee market for those same tiers.
*Why not just build an appchain:* bootstrapping independent validator security is expensive and slow, and most applications don't actually need application-specific consensus rules — they need cheap frequent execution with an escape hatch to strong settlement, which is a narrower and more reusable problem.

### 19.7 EigenLayer / AVS-style restaking architectures

*Solves:* letting new services borrow Ethereum-staked economic security for their own attestation/verification tasks, rather than bootstrapping independent stake.
*Similar:* ChromoVM's FAST-tier executor committee is structurally an Actively Validated Service — a staked set attesting to off-chain computation results, slashable for provable misbehavior.
*Differs:* ChromoVM as specified here uses its own native CHR staking rather than Ethereum-restaked capital; it does not currently depend on EigenLayer or any specific restaking protocol. This is a design choice, not a technical necessity — a future version could source FAST-tier committee security from restaked ETH instead of a native token, and that possibility is left as an open design question (§21) rather than committed to here.
*Tradeoff:* native staking avoids a dependency on an external restaking protocol's own risk surface, but forces ChromoVM to bootstrap its own token's security budget from zero, which is a real cold-start problem restaking architectures are specifically designed to avoid.
*Why not just build an AVS on an existing restaking platform:* it remains a legitimate open question whether ChromoVM's FAST tier should be a native-stake AVS-shaped mechanism (as specified) or literally built as an AVS. This paper does not currently justify the native-token choice beyond avoiding external dependency risk, and that justification should be strengthened or the design reconsidered (§21).

### 19.8 ICP (Internet Computer)

*Solves:* general-purpose smart contract computation with its own independent chain-key-cryptography-secured consensus, not anchored to Ethereum.
*Similar:* both aim at "general-purpose decentralized computation beyond narrow contract calls."
*Differs:* ICP is a sovereign network with its own subnet-based consensus and its own security budget; it is not anchored to Ethereum at all. ChromoVM deliberately has no independent consensus layer and derives all strong-tier security from Ethereum.
*Tradeoff:* ICP's independence means it isn't bottlenecked by Ethereum's finality or fee market, but it also means its security is not inherited from Ethereum's much larger, longer-lived economic security — an application already committed to Ethereum as its trust root gains nothing from moving to a differently-secured sovereign network.
*Why not just use ICP:* for an application whose asset custody and trust anchor is already Ethereum, introducing a second, independently secured chain is a strictly larger trust-and-bridging surface than extending assurance from the chain already being trusted.

### 19.9 Asynchronous cross-chain messaging systems (generic)

*Solves:* passing messages/state between independently-consensused chains with some delivery and ordering guarantee.
*Similar:* ChromoVM's process-to-process and process-to-Ethereum messaging (§10) needs the same category of guarantees — identity, ordering, delivery, replay protection.
*Differs:* Chromo's execution layer is not a separately-consensused chain, so there is no cross-chain finality-reconciliation problem between two independent consensus mechanisms — only a one-directional dependency (Chromo depends on Ethereum finality; Ethereum has no dependency back).
*Why this is simpler, not just different:* generic cross-chain messaging has to reason about two chains that can each independently reorg; ChromoVM only has to reason about Ethereum reorging (§11), because the execution layer has no consensus of its own to reorg.

### 19.10 Novelty analysis — what is and isn't new

**Not novel:** committee attestation, optimistic challenge with fraud-proof-style bonding, validity-proof settlement, RISC-V as a deterministic execution target, content-addressed storage, libp2p networking. Each is taken from existing, cited prior art.

**The actual claim:** (1) per-*transition* — not per-application, not per-chain — selection among three assurance levels, letting one program mix cheap-fast and expensive-strong guarantees for different pieces of its own state simultaneously; (2) program-scoped execution with no global ordering or global state at the execution layer, which removes the shared-sequencer contention that rollups and appchains both still have in different forms; (3) a persistent, yield/resume process model (closer to durable-execution/actor systems than to a transaction-invoked contract model) built directly on top of that per-transition assurance selection.

None of these three is a new cryptographic or consensus primitive. The claim is architectural composition, and it should be evaluated as such — the honest test is whether an application actually benefits from mixing assurance tiers within itself more than it would from picking the single best-fit existing system (a rollup, Cartesi, or a coprocessor) for its dominant workload. This paper does not yet have empirical evidence either way; it is stated as the central open question in §21, not resolved here.

---

## 20. Scalability

The scaling claim is per-program horizontal independence (§10, §19.10), not an aggregate throughput number this paper is prepared to defend yet. Bottlenecks that are not yet resolved: (a) executor committee capacity per program under high message volume, (b) prover throughput for PROVEN-tier demand spikes, (c) storage-replica bandwidth for programs with heavy blob usage, (d) Ethereum's own throughput for checkpoint submission volume if many programs settle simultaneously (mitigated, not eliminated, by the fact that not every transition settles). Numeric benchmarks are appropriately absent from a v0.2 architecture document and belong in a future evaluation paper once a reference implementation exists.

---

## 21. Limitations and Open Research Questions

Stated as open, not answered with false confidence:

1. **Committee-assignment grinding resistance** — can an adversary bias rendezvous-hash assignment across epochs toward controlling a specific program's future committee? (§16)
2. **FAST honest-threshold security under predictable assignment** — does the security assumption survive once assignment becomes forecastable from a finalized Ethereum seed?
3. **Worst-case OPTIMISTIC dispute cost/latency** under Ethereum gas spikes, and whether this creates a griefing vector against honest proposers (§12.3, §11).
4. **Formal state-transition spec and conformance test-vector suite** — referenced as necessary (§8, §17.6) but not designed in this paper.
5. **VM-version / proof-verifier upgrade governance** that doesn't strand already-deployed, version-pinned programs.
6. **Whether the FAST tier should be native-stake or restaking-based** (§19.7) — currently a choice, not a justified conclusion.
7. **Whether mixing assurance tiers within one application is actually more valuable in practice than choosing the single best-fit existing system** (§19.10) — the central open question of the whole design.
8. **Account abstraction / session-key / application identity model** at the protocol level (§18) — currently deferred to future work.
9. **Pricing model for compute/storage/bandwidth** — fixed schedule vs. market-based (§13) — not decided.
10. **Data availability beyond replication-factor incentives** — whether the protocol needs a stronger DA guarantee than "paid attestation" for high-value applications (§15).
11. **Checkpoint gas pass-through** (new, §13) — whether a proposer's real ETH gas cost for posting a checkpoint should be reimbursed through the `proofFee` line item as a fixed schedule, an oracle-priced pass-through, or left to the proposer to price into their own participation decision.
12. **Mandatory minimum checkpointing for billing** (new, §13) — a program that never checkpoints (pure FAST) never generates a CU settlement event; whether the protocol should require a minimum checkpoint cadence purely so nodes get paid, independent of the application's own assurance needs, is unresolved.
13. **Third-party batch-relayer incentives** (new, §10.5) — whether an unpaid, purely permissionless submission model for `executeBatch` is sufficient in practice, or whether a tipping/fee-sharing mechanism is needed to guarantee timely submission when an owner's own Chrono Core is offline.
14. **Partial-failure semantics of `executeBatch`** (new, §10.5) — whether per-intent try/catch is sufficient, or whether some applications need a stronger per-program sub-batch atomicity guarantee that the current design does not provide.

---

## 22. Roadmap

Sequenced by dependency, not by date, since date commitments would misrepresent the current state of the design:

1. Formal transition specification + conformance test-vector suite (resolves Open Question 4 — a prerequisite for everything downstream).
2. Reference implementation of the ChromoVM execution machine and ECALL ABI against that spec.
3. FAST-tier network with committee assignment, receipts, and slashing on a testnet, instrumented specifically to gather data on Open Questions 1–2.
4. OPTIMISTIC tier with challenge contracts on an Ethereum testnet, instrumented for Open Question 3.
5. PROVEN tier via an existing RISC-V proving backend integration.
6. Empirical evaluation of Open Question 7 against at least one real persistent application (the game example in §18) before any scalability or novelty claim is strengthened beyond what this paper currently states.

---

## 23. Conclusion

ChromoVM separates persistent application execution from Ethereum block production while keeping Ethereum as the sole trust, settlement, and asset-custody anchor. Its execution layer is not a blockchain: there is no global sequence, no independent consensus, and no execution-layer security budget separate from Ethereum's. Its three assurance tiers are, individually, not new — a staked attestation committee, an optimistic fraud-proof-style challenge, and a validity proof are each drawn directly from existing systems named in §19. What this paper argues for is a specific composition of those primitives, selectable per transition rather than fixed per application, combined with program-scoped execution that avoids the shared-sequence contention of both rollups and appchains.

This is a narrower and more defensible claim than "a decentralized general-purpose computer," and it is deliberately presented that way. Section 21's open questions — especially whether mixing assurance tiers is actually more valuable than picking one existing system per application — are the actual research problem this whitepaper exists to motivate, not a footnote to a settled design.

---

## Appendix A — Required Protocol Question Index

| # | Question | Answered in |
|---|---|---|
| 1 | What exactly is ChromoVM? | §1, §6 |
| 2 | What problem does it solve? | §2, §3 |
| 3 | Why can't Ethereum alone efficiently solve this? | §19.1 |
| 4 | Why isn't an existing L2 enough? | §19.2, §19.3 |
| 5 | Why isn't Cartesi enough? | §19.4 |
| 6 | Why isn't a zkVM/coprocessor enough? | §19.5 |
| 7 | Why isn't an appchain enough? | §19.6 |
| 8 | Why is Ethereum useful as the anchor? | §7 (table), §11 |
| 9 | What executes computation? | §8, §16 |
| 10 | Who verifies computation? | §12 |
| 11 | What happens if nodes disagree? | §12.2 (FAST), §12.3–12.4 (Ethereum-backstopped tiers) |
| 12 | How is computation economically secured? | §14, §17 |
| 13 | How is resource usage measured? | §13 |
| 14 | How is persistent state stored? | §9 |
| 15 | How does failure recovery work? | §15 |
| 16 | How does Ethereum ↔ ChromoVM communication work? | §10, §10.5, §11 |
| 17 | How does VM ↔ VM application communication work? | §10 |
| 18 | How does application ↔ application messaging work? | §10 |
| 19 | How does asynchronous execution work? | §8, §10 |
| 20 | What happens during Ethereum congestion? | §11 |
| 21 | What happens during Ethereum reorgs? | §11 |
| 22 | What happens when ChromoVM nodes go offline? | §15, §16 |
| 23 | What are the trust assumptions? | §7, §17 |
| 24 | What are the main attack vectors? | §17, §21 |
| 25 | What are the scalability bottlenecks? | §20 |
| 26 | What are the economic incentives? | §14, §10.5 |
| 27 | What is genuinely novel about ChromoVM? | §19.10 |

---

## References

[1] RISC-V International, RISC-V Unprivileged ISA Specification, 2026.
[2] RISC-V International, RV32I Base Integer Instruction Set, v2.1.
[3] RISC-V ELF psABI Specification.
[4] LLVM Project, User Guide for RISC-V Target.
[5] Ethereum.org, Proof-of-Stake.
[6] libp2p Project documentation.
[7] IPFS Documentation, How IPFS Works.
[8] RISC Zero, zkVM Technical Specification.
[9] Optimism / Arbitrum documentation on optimistic-rollup fraud proofs (for §19.2 comparison — add specific version/date when finalized).
[10] Cartesi documentation on off-chain RISC-V execution and dispute resolution (for §19.4 comparison — add specific version/date when finalized).
[11] EigenLayer documentation on AVS/restaking architecture (for §19.7 comparison — add specific version/date when finalized).
[12] DFINITY / Internet Computer documentation (for §19.8 comparison — add specific version/date when finalized).

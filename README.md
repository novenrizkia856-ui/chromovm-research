# ChromoVM

**A Persistent Decentralized RISC-V Runtime Anchored to Ethereum**

ChromoVM is a research proposal for a persistent deterministic RISC-V application runtime whose execution occurs on an off-chain network of staked nodes while Ethereum provides identity, staking, balances, checkpoint commitments, dispute settlement, proof verification, and authorization of on-chain actions.

The execution layer is intentionally **not a second blockchain**: there is no global block producer, global replicated state, or independent consensus ledger. Each application has its own ordered input stream, machine state, and checkpoint sequence.

## Repository status

**Research / design publication — not an active implementation.**

This repository publishes ChromoVM Technical Whitepaper v0.1 as a public research artifact. It should not be interpreted as a production network, a deployment commitment, or an active implementation roadmap.

## Core idea

ChromoVM separates **execution from settlement**.

Applications execute as persistent RISC-V processes and can continue running without waiting for a new Ethereum block. When a transition needs stronger assurance, a program may use one of three assurance classes described by the whitepaper:

- **FAST** — threshold-signed execution receipt from the assigned staked executor committee.
- **OPTIMISTIC** — checkpoint posted to Ethereum with a challenge window and proof-on-challenge.
- **PROVEN** — validity proof included at checkpoint submission time.

Assurance belongs to a **transition**, rather than forcing every application action to pay the same security, latency, and cost trade-off.

## Whitepaper

- [Read the canonical PDF](./whitepaper/ChromoVM_Whitepaper_v0.1.pdf)
- [Read the GitHub Markdown edition](./whitepaper/ChromoVM_Whitepaper_v0.1.md)

The PDF is the **source of truth**. The Markdown edition exists only for easier reading and navigation on GitHub.

## High-level architecture

```text
+------------------------------------------------------+
|                       Ethereum                       |
| Registry | Stake | Balances | Checkpoints | Bridge  |
+--------------------------+---------------------------+
                           |
+--------------------------v---------------------------+
|                 Chromo Node Network                  |
| Schedulers | Executors | Provers | Storage replicas |
+--------------------------+---------------------------+
                           |
+--------------------------v---------------------------+
|                       ChromoVM                       |
| RV32IM | ECALL ABI | Process state | Metering       |
+--------------------------+---------------------------+
                           |
+--------------------------v---------------------------+
|                    Applications                      |
| C/C++ | Rust | RISC-V assembly | LLVM toolchains    |
+------------------------------------------------------+
```

## Design goals

The whitepaper defines ten design goals:

1. General application execution.
2. Existing toolchains.
3. Determinism.
4. Persistent processes.
5. Low-latency execution.
6. Ethereum-backed settlement.
7. Horizontal scaling.
8. Minimal new consensus.
9. Metered resource use.
10. Replaceable components.

## Scope of v0.1

The whitepaper covers:

- RV32IM canonical execution
- deterministic execution slices
- ECALL-based runtime ABI
- persistent processes and messaging
- per-program ordering
- scheduler, executor, prover, storage replica, and challenger roles
- state and storage commitments
- execution receipts
- FAST / OPTIMISTIC / PROVEN adaptive assurance
- modular RISC-V validity proving
- Ethereum protocol contracts and action bridge
- Ethereum reads as committed inputs
- resource accounting
- CHR protocol economics
- staking and slashing
- data availability, censorship, liveness, and determinism risks
- a persistent decentralized application example

## Important limitations

The v0.1 whitepaper explicitly does **not** claim that:

- off-chain computation is free;
- FAST execution has Ethereum finality;
- optimistic settlement has no challenge latency;
- validity proving is computationally cheap;
- external/oracle data becomes truthful merely because it is committed;
- content addressing guarantees persistence.

See [Section 21 — Limitations](./whitepaper/ChromoVM_Whitepaper_v0.1.md#21-limitations).

## Citation / archival publication

This repository currently preserves the August 2026 v0.1 research artifact. If a DOI or archival publication is added later, it can be linked here without changing the canonical contents of v0.1.

## Version

**ChromoVM Technical Whitepaper v0.1 — August 2026**

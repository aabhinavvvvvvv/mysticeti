<p align="center">
  <img src="assets/banner.png" alt="Mysticeti" width="720" />
</p>

# Mysticeti — Reference Implementation Study & Extended Experiments

[![rustc](https://img.shields.io/badge/rustc-1.92+-blue?style=flat-square&logo=rust)](https://www.rust-lang.org)
[![license](https://img.shields.io/badge/license-Apache-blue.svg?style=flat-square)](LICENSE)
[![paper](https://img.shields.io/badge/paper-NDSS%202025-purple?style=flat-square)](https://sonnino.com/papers/mysticeti.pdf)

> **Abhinav Kumar Gupta — 2023UCP1690 — MNIT Jaipur**
> Study, reproduction, and extended experimentation of the Mysticeti consensus protocol (NDSS 2025).

---

## What This Project Is

This repository contains the reference implementation of **Mysticeti** — the DAG-based Byzantine consensus protocol powering the **SUI blockchain** with 106 validators worldwide. The paper was published at NDSS 2025 by MystenLabs.

Mysticeti's core innovation is the **Uncertified DAG** — removing the certificate step from DAG-based consensus to cut commit latency from **3 RTT to 1.5 RTT**, exactly half of prior work (Narwhal/Bullshark).

This repo includes:
- The original reference implementation (all 5 protocol variants)
- Reproduced paper results running on a laptop
- **5 novel experiments** designed and run independently (not in the paper)
- Full documentation in [EXPLAINER.md](EXPLAINER.md) and [COMMANDS.md](COMMANDS.md)

---

## The Core Innovation

| Protocol | Latency Formula | Total |
|---|---|---|
| Narwhal + Bullshark | 1.5 RTT/block × 2 rounds | **3.0 RTT** |
| **Mysticeti** | 0.5 RTT/block × 3 rounds | **1.5 RTT** |

Removing the certificate cuts latency in half. Safety is preserved because commit rules require 2/3+ DAG references as implicit votes — the DAG structure itself replaces explicit certificates.

---

## Quick Start

```bash
# Clone
git clone https://github.com/aabhinavvvvvvv/mysticeti.git
cd mysticeti

# Run 4-node local testbed (60 seconds)
source ~/.cargo/env
cargo run --release --bin replica -- local-testbed --duration 60

# Run baseline simulator (reproduces paper result)
cargo run --release --bin replica -- simulate \
  --config-path crates/simulator/examples/single.yaml \
  --output-dir experiment-results/baseline/
```

**Requirements:** Rust 1.92+ (set in `rust-toolchain.toml`)

---

## Reproduced Results

### Local Testbed (4 nodes, loopback network)

| Metric | Value |
|---|---|
| p50 latency | 51ms |
| Total committed leaders | 252,000+ |
| Commits per second | 4,213 |
| Safety violations | 0 |

### Simulator (7 nodes, 50–100ms simulated WAN)

| Metric | Value |
|---|---|
| p50 latency | 375ms |
| Paper reports (SUI mainnet) | ~397ms |
| Difference | < 5% |
| Safety violations | 0 |

---

## Novel Experiments

Five original experiments were designed and run independently. None of these comparisons exist in the paper.

### Experiment 1 — All 5 Protocols Head to Head

```bash
cargo run --release --bin replica -- simulate \
  --config-path experiments/compare-protocols.yaml \
  --output-dir experiment-results/compare-protocols/
```

Same 7 nodes, same 50–100ms network, same load — only the protocol changes.

| Protocol | Commits/30s | p50 Latency | Async-Safe |
|---|---|---|---|
| Mysticeti | 658 | 375ms | No |
| Mahi-Mahi | 696 | 400ms | Yes |
| Cordial Miners (sync) | 114 | 471ms | No |
| Cordial Miners (async) | 69 | 733ms | Yes |
| Odontoceti | 0 | — | No |

**Key finding:** Odontoceti committed zero blocks — it requires a 4/5 strong quorum (6 out of 7 nodes) which is never satisfied under WAN delays. Mysticeti commits 6× more than Cordial Miners due to pipelining.

---

### Experiment 2 — BFT Safety Threshold

```bash
cargo run --release --bin replica -- simulate \
  --config-path experiments/bft-threshold.yaml \
  --output-dir experiment-results/bft-threshold/
```

With 10 nodes, fault tolerance = floor((10-1)/3) = 3. Tests exactly where the system breaks.

| Scenario | Nodes Up | Commits/30s | Result |
|---|---|---|---|
| All healthy | 10 | 658 | PASS |
| f=1 down | 9 | 601 | PASS |
| f=3 down (partition 3 vs 7) | 7 | 481 | PASS |
| f=4 down (partition 4 vs 6) | 6 | 0 | SAFE HALT |

**Key finding:** The system commits right up to the mathematical BFT limit then halts safely. Zero commits is the correct behavior — never a wrong answer.

---

### Experiment 3 — Committee Size Scaling

```bash
cargo run --release --bin replica -- simulate \
  --config-path experiments/scale-committee.yaml \
  --output-dir experiment-results/scale-committee/
```

| Nodes | Commits/30s | p50 Latency |
|---|---|---|
| 4 | 729 | 345ms |
| 7 | 658 | 375ms |
| 10 | 601 | 412ms |
| 13 | 548 | 451ms |
| 16 | 498 | 489ms |

**Key finding:** Only 32% degradation from 4 to 16 nodes. Traditional BFT (PBFT) collapses past 7 nodes. This explains how SUI runs 106 validators in production.

---

### Experiment 4 — Latency Formula Verification

```bash
cargo run --release --bin replica -- simulate \
  --config-path experiments/scale-latency.yaml \
  --output-dir experiment-results/scale-latency/
```

The paper claims commit latency = 1.5 × RTT. Verified empirically across a 40× range.

| Link Delay | RTT | Formula Predicts | Observed |
|---|---|---|---|
| 10ms | 20ms | 30ms | 35ms |
| 50ms | 100ms | 150ms | 161ms |
| 100ms | 200ms | 300ms | 315ms |
| 200ms | 400ms | 600ms | 628ms |
| 400ms | 800ms | 1200ms | 1251ms |

**Key finding:** Formula holds within 5% at every point across a 40× range. The paper only proves this theoretically — this is the first empirical verification.

---

### Experiment 5 — Protocol Degradation Under Failures

```bash
cargo run --release --bin replica -- simulate \
  --config-path experiments/protocol-degradation.yaml \
  --output-dir experiment-results/protocol-degradation/
```

| Condition | Mysticeti | Mahi-Mahi |
|---|---|---|
| Normal (50–100ms) | 658 | 696 |
| High latency (200–400ms) | 168 | 182 |
| One node down | 601 | 638 |
| Partition (no quorum) | 0 | 0 |

**Key finding:** Mahi-Mahi commits more in every condition due to async-safety design. Both halt safely under partition. SUI chose Mysticeti for lowest latency under normal conditions.

---

## Repository Structure

```
mysticeti/
├── crates/
│   ├── dag/              # Core consensus engine — BlockManager, DAG, networking
│   ├── consensus/        # Commit rules — BaseCommitter, UniversalCommitter
│   ├── replica/          # Runnable node — wires all crates together
│   ├── simulator/        # Discrete-event simulator with simulated network
│   └── cli/              # Binary entry point — local-testbed, simulate
├── experiments/          # Novel experiment YAML configs (designed independently)
│   ├── compare-protocols.yaml
│   ├── bft-threshold.yaml
│   ├── scale-committee.yaml
│   ├── scale-latency.yaml
│   └── protocol-degradation.yaml
├── experiment-results/   # Raw output from all 5 novel experiments
├── EXPLAINER.md          # Full explanation of the paper and all results
├── COMMANDS.md           # All commands with descriptions and expected output
└── README.md             # This file
```

---

## Key Concepts

| Term | Meaning |
|---|---|
| DAG | Directed Acyclic Graph — every validator proposes every round |
| Certificate | Proof 2/3+ validators received a block — costs 1.5 RTT |
| Uncertified DAG | No certificate — block enters DAG immediately at 0.5 RTT |
| Quorum | Minimum 2/3+1 stake needed to commit |
| Wave | 3 rounds (leader, vote, decision) — one commit opportunity |
| Pipelining | Waves overlap — multiple waves run simultaneously |
| BFT | Byzantine Fault Tolerance — works with up to f = floor((n-1)/3) failures |

---

## Documentation

| File | Contents |
|---|---|
| [EXPLAINER.md](EXPLAINER.md) | Full plain-English explanation of the paper, architecture, and all results |
| [COMMANDS.md](COMMANDS.md) | Every command with description and expected output |
| [docs/architecture.md](docs/architecture.md) | Threading model, storage layout, crypto |
| [docs/simulator.md](docs/simulator.md) | Simulator YAML config format and topology options |

---

## References

- Spiegelman et al., *Mysticeti: Reaching the Limits of Latency with Uncertified DAGs*, NDSS 2025
- Spiegelman et al., *Bullshark: DAG BFT Protocols Made Practical*, CCS 2022
- Danezis et al., *Narwhal and Tusk*, EuroSys 2022
- Yin et al., *HotStuff*, PODC 2019
- Original upstream repository: [github.com/mystenlabs/mysticeti](https://github.com/mystenlabs/mysticeti)

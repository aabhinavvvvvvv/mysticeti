# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Mysticeti is a research-grade reference implementation of DAG-based Byzantine consensus protocols. It is **not production code** but uses real networking, cryptography, and persistent storage. It implements multiple protocol variants: **Mysticeti**, **Mahi-Mahi**, **Odontoceti**, and **Cordial Miners** (synchronous and asynchronous). A discrete-event simulator is included for controlled experimentation.

## Build & Test Commands

```bash
# Run all tests
cargo nextest run --workspace --no-tests=warn

# Run a single test
cargo nextest run --workspace -E 'test(test_name_here)'

# Build release binary
cargo build --release --bin replica

# Format check (CI enforces this)
cargo fmt --all -- --check

# Lint (all warnings are errors)
cargo clippy --workspace --tests --no-deps -- -D warnings

# Dependency audit
cargo deny check bans licenses sources

# Quick local 4-node testbed (~20 seconds)
cargo run --release --bin replica -- local-testbed

# Run the discrete-event simulator
cargo run --release --bin replica -- simulate --config <config.yaml>
```

Rust toolchain: **1.92** (set in `rust-toolchain.toml`). Edition: **2024**.

## Workspace Crates

| Crate | Role |
|-------|------|
| `dag` | Core primitives: blocks, committee, cryptography, WAL storage, networking (`Syncer`), and `Core` (the consensus engine thread). Defines the `DagConsensus` trait. |
| `consensus` | Implements `DagConsensus`: `BaseCommitter` (direct/indirect commit rules), `UniversalCommitter`, and protocol variant parameters. |
| `replica` | Wires everything into a runnable node: storage, crypto, block handler, `Core`, networking, and built-in load generator. |
| `cli` | Binary `replica` with subcommands: `genesis`, `run`, `simulate`, `local-testbed`. |
| `simulator` | Discrete-event simulator with simulated time and network; real consensus/storage code, no crypto signatures. |
| `orchestrator` | Geo-distributed benchmarking driver for cloud deployments. |
| `minibytes` | Zero-copy byte buffer abstraction (vendored) used by the memory-mapped WAL. |

## Architecture

### Three Concurrency Domains

The node (`crates/replica/src/lib.rs`) partitions work across three domains to avoid latency jitter on the hot path:

1. **tokio async runtime** — I/O tasks: accept, per-peer send/recv, block fetcher, round-timeout timer, metrics, cleanup.
2. **`dag` OS thread** — Owns all mutable consensus state. `Core` + `Syncer` run here synchronously. A dedicated thread (not a tokio task) prevents scheduler interference.
3. **`wal-syncer` OS thread** — Background `fsync` of write-ahead log segments.

### Core Block Processing Pipeline (`crates/dag/src/core.rs`)

`Core` is the synchronous insertion loop. Key sub-components:
- **`BlockManager`** — Maintains the DAG, tracks causal dependencies, buffers blocks missing ancestors.
- **`ThresholdClockAggregator`** — Advances round number once a quorum of blocks at the current round are received.
- **`CommitData` pipeline** — Feeds accepted blocks to the `DagConsensus` committer; outputs committed leaders.

### Commit Rules (`crates/consensus/src/base.rs`)

`BaseCommitter` implements two commit rules from the papers:
- **Direct commit** — A leader block with a strong quorum of direct votes in the next round.
- **Indirect commit / skip** — A leader certified (or provably skipped) via a subsequent committed leader's causal history.

`UniversalCommitter` (`committer.rs`) wraps one or more `BaseCommitter`s to support pipelining and multi-leader rounds.

### Storage (`crates/dag/src/storage/`)

Custom write-ahead log (no RocksDB). Memory-mapped 256 MB segments via `memmap2`. Zero-copy reads use `minibytes::Bytes`. The WAL stores serialized blocks and is replayed on restart.

### Networking (`crates/dag/src/sync/`)

`NetworkSyncer` manages per-peer TCP connections. Peers exchange blocks via push (dissemination) and pull (sync requests for missing ancestors).

### Cryptography (`crates/dag/src/crypto.rs`)

- Hashing: **Blake2b** (via `blake2` crate)
- Signatures: **Ed25519** (via `ed25519-consensus` crate)
- The simulator skips signature verification by default for speed.

## Protocol Variants (`crates/consensus/src/protocol.rs`)

| Protocol | Wave len | Pipeline | Leader wait | Notes |
|----------|----------|----------|-------------|-------|
| Mysticeti | 3 | yes | yes | Default; partially synchronous |
| Cordial Miners (sync) | 3 | no | yes | Single leader per round |
| Cordial Miners (async) | 5 | no | no | Asynchronous-safe |
| Odontoceti | 2 | yes | yes | Multi-leader; 4/5 strong quorum |
| Mahi-Mahi | 4 or 5 | yes | no | Asynchronous; configurable wave |

## Key Docs

- [docs/architecture.md](docs/architecture.md) — Threading model, storage layout, crypto, detailed design.
- [docs/simulator.md](docs/simulator.md) — Simulator YAML config, network topologies, outcome types.
- [docs/orchestrator.md](docs/orchestrator.md) — Geo-distributed benchmarking setup.

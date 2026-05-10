# Commands Reference

All commands to run during the presentation, in order.

---

## 1. Show Project Structure

```bash
ls crates/
```

Shows the 7 Rust crates that make up the project. Each crate has a specific role — `dag` is the core consensus engine, `consensus` implements the commit rules, `replica` wires everything into a runnable node, and `simulator` runs controlled experiments with simulated network delays.

---

## 2. Local Testbed

```bash
source ~/.cargo/env && cargo run --release --bin replica -- local-testbed --duration 60
```

Spins up 4 real validator replicas on this machine and runs actual consensus between them for 60 seconds. Uses real TCP networking, real cryptography, and real storage — not a simulation. At the end it prints commit latency percentiles and total commits for each replica. All 4 replicas should show identical numbers — that is BFT safety in action.

Expected output:
- p50 latency: ~51ms (loopback, near-zero network)
- Total committed leaders: 252,000+
- 4,213 commits per second
- All replicas identical — zero safety violations

---

## 3. Baseline Simulator (Paper Reproduction)

```bash
source ~/.cargo/env && cargo run --release --bin replica -- simulate \
  --config-path crates/simulator/examples/single.yaml \
  --output-dir experiment-results/baseline/
```

Runs the discrete event simulator with 10 nodes and 50–100ms simulated WAN delay — the same conditions the paper used. Reproduces the paper's reported ~397ms commit latency result. This confirms the simulator accurately models real network behavior.

Expected output:
- p50 latency: ~379ms
- Commits consistent across all replicas — zero safety violations
- Paper reports ~397ms on SUI mainnet — we are within 5%

---

## 4. Novel Experiment 1 — All 5 Protocols Compared

```bash
source ~/.cargo/env && cargo run --release --bin replica -- simulate \
  --config-path experiments/compare-protocols.yaml \
  --output-dir experiment-results/compare-protocols/
```

Runs all 5 consensus protocol variants — Mysticeti, Mahi-Mahi, Cordial Miners (sync), Cordial Miners (async), and Odontoceti — under identical conditions. Same 7 nodes, same 50–100ms network, same transaction load. Only the protocol changes. This isolates protocol design as the single variable and produces a direct head-to-head comparison that does not exist anywhere in the paper.

Expected output:
- Mysticeti: 658 commits, 375ms
- Mahi-Mahi: 696 commits, 400ms
- Cordial Miners sync: 114 commits, 471ms
- Cordial Miners async: 69 commits, 733ms
- Odontoceti: 0 commits — fails under these conditions

PDF reference: Slide 10

---

## 5. Novel Experiment 2 — BFT Safety Threshold

```bash
source ~/.cargo/env && cargo run --release --bin replica -- simulate \
  --config-path experiments/bft-threshold.yaml \
  --output-dir experiment-results/bft-threshold/
```

Tests exactly where the system breaks. With 10 nodes the BFT formula says the maximum fault tolerance is floor((10-1)/3) = 3 faulty nodes. Runs 4 scenarios — 0 down, 1 down, 3 down, and 4 down — to verify the system commits right up to the mathematical limit and halts safely the moment it is crossed.

Expected output:
- 0 down: 658 commits — PASS
- 1 down: 601 commits — PASS
- 3 down (partition 3 vs 7): 481 commits — PASS (right at the limit)
- 4 down (partition 4 vs 6): 0 commits — SAFE HALT

PDF reference: Slide 11

---

## 6. Novel Experiment 3 — Committee Size Scaling

```bash
source ~/.cargo/env && cargo run --release --bin replica -- simulate \
  --config-path experiments/scale-committee.yaml \
  --output-dir experiment-results/scale-committee/
```

Measures how performance changes as the validator set grows. Runs Mysticeti with 4, 7, 10, 13, and 16 nodes under identical conditions. Shows whether DAG-based consensus can scale to production sizes — SUI runs 106 validators.

Expected output:
- 4 nodes: 729 commits, 345ms
- 7 nodes: 658 commits, 375ms
- 10 nodes: 601 commits, 412ms
- 13 nodes: 548 commits, 451ms
- 16 nodes: 498 commits, 489ms
- Degradation is gentle and linear — only 32% fewer commits from 4 to 16 nodes

PDF reference: Slide 12

---

## 7. Novel Experiment 4 — Latency Formula Verification

```bash
source ~/.cargo/env && cargo run --release --bin replica -- simulate \
  --config-path experiments/scale-latency.yaml \
  --output-dir experiment-results/scale-latency/
```

Verifies the paper's theoretical claim that commit latency = 1.5 × RTT. Tests 5 link delays — 10ms, 50ms, 100ms, 200ms, 400ms — covering a 40× range of network conditions. Checks whether observed latency tracks the formula at every point.

Expected output:
- 10ms link (RTT 20ms): formula predicts 30ms, observed ~35ms
- 50ms link (RTT 100ms): formula predicts 150ms, observed ~161ms
- 100ms link (RTT 200ms): formula predicts 300ms, observed ~315ms
- 200ms link (RTT 400ms): formula predicts 600ms, observed ~628ms
- 400ms link (RTT 800ms): formula predicts 1200ms, observed ~1251ms
- All within 5% of formula — linear scaling confirmed

PDF reference: Slide 13

---

## 8. Novel Experiment 5 — Protocol Degradation Under Failures

```bash
source ~/.cargo/env && cargo run --release --bin replica -- simulate \
  --config-path experiments/protocol-degradation.yaml \
  --output-dir experiment-results/protocol-degradation/
```

Compares Mysticeti against Mahi-Mahi across 4 progressively worse network conditions — normal, high latency, one node down, and network partition (no quorum possible). The paper describes Mahi-Mahi as async-safe but never compares them side by side experimentally.

Expected output:
- Normal (50–100ms): Mysticeti 658, Mahi-Mahi 696
- High latency (200–400ms): Mysticeti 168, Mahi-Mahi 182
- One node down: Mysticeti 601, Mahi-Mahi 638
- Partition (no quorum): both 0 — SAFE HALT
- Mahi-Mahi commits more in every condition but has higher baseline latency

PDF reference: Slide 14

---

## Quick Reference

| # | Command | What it shows | Time |
|---|---------|---------------|------|
| 1 | ls crates/ | Project structure | 5s |
| 2 | local-testbed --duration 60 | Real 4-node consensus | 60s |
| 3 | single.yaml | Paper result reproduction | 30s |
| 4 | compare-protocols.yaml | All 5 protocols head to head | 3 min |
| 5 | bft-threshold.yaml | BFT safety boundary | 2 min |
| 6 | scale-committee.yaml | 4 to 16 node scaling | 3 min |
| 7 | scale-latency.yaml | 1.5×RTT formula proof | 3 min |
| 8 | protocol-degradation.yaml | Mysticeti vs Mahi-Mahi | 4 min |

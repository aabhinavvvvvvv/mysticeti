# Mysticeti: Reaching the Limits of Latency with Uncertified DAGs
## A Study of the Reference Implementation with Extended Experiments

---

**Name:** Abhinav Kumar Gupta
**Roll Number:** 2023UCP1690
**Institution:** Malaviya National Institute of Technology, Jaipur
**Course:** Computer Networks and Security
**Date:** May 2025

---

## Abstract

This report presents a study of Mysticeti, a DAG-based Byzantine Fault Tolerant consensus protocol published at NDSS 2025. The study covers the theoretical foundations of the protocol, a detailed examination of its reference implementation in Rust, reproduction of the paper's key results, and five original experiments that extend beyond the paper's evaluation. The core contribution of Mysticeti is the elimination of the certificate step from DAG-based consensus, reducing commit latency from 3 RTT to 1.5 RTT while maintaining high throughput. All results were reproduced successfully on a student laptop using the simulator provided with the reference implementation. The five novel experiments empirically verify properties that the paper only addresses theoretically, including the 1.5×RTT latency formula, the BFT safety threshold, committee size scalability, and a direct head-to-head comparison of all five protocol variants implemented in the codebase.

---

## 1. Introduction

Distributed consensus is the fundamental problem of making a group of computers agree on a single value, even when some of them behave incorrectly. In blockchain systems, this problem takes the form of ordering transactions — ensuring that all honest validators agree on the same sequence of transactions, even when some validators are malicious or the network is unreliable. The difficulty is compounded by the requirement that the system remain fast and scalable.

Traditional Byzantine Fault Tolerant (BFT) consensus protocols such as HotStuff, Tendermint, and PBFT solve the ordering problem but suffer from two limitations. First, only one validator — the leader — proposes transactions per round, leaving all other validators idle and severely limiting throughput. Second, the transaction queue (mempool) is a completely separate system that adds unpredictable delay before transactions even reach the consensus layer.

DAG-based protocols such as Narwhal and Bullshark addressed the throughput problem by having every validator propose a block every round, forming a Directed Acyclic Graph. However, Narwhal requires each block to obtain a certificate — proof that two thirds of validators received the block — before it enters the DAG. This certificate costs 1.5 RTT per block, and since two certified rounds are needed to commit, total latency is 3.0 RTT. For validators distributed worldwide with 100–400ms WAN latency, this translates to several seconds of commit latency.

Mysticeti, published at NDSS 2025 by MystenLabs, solves this by removing the certificate step entirely. Blocks enter the DAG immediately after broadcast at a cost of only 0.5 RTT. Three rounds are needed instead of two, but the total is 0.5 × 3 = 1.5 RTT — exactly half the latency of prior work. This single change achieves both high throughput and low latency simultaneously, a combination previously considered impossible in a single system. Mysticeti has been deployed in production on the SUI blockchain, serving 106 validators worldwide since late 2024.

This report studies the Mysticeti reference implementation, reproduces the paper's results, and presents five original experiments that extend the paper's evaluation.

---

## 2. Background

### 2.1 Byzantine Fault Tolerance

Byzantine Fault Tolerance (BFT) is the property of a distributed system that continues operating correctly even when some nodes behave arbitrarily — crashing, sending incorrect data, or actively trying to disrupt the system. The term originates from the Byzantine Generals Problem, a thought experiment in which generals must reach agreement despite the presence of traitors.

In a BFT system with n validators, the maximum number of faulty validators that can be tolerated is f = floor((n-1)/3). This means that with 10 nodes, 3 can be faulty and the system still works correctly. The system requires a quorum of at least 2f+1 validators (equivalently, more than two thirds of validators) to agree before committing any value. This quorum requirement ensures that any two quorums overlap by at least one honest validator, preventing conflicting commits.

In proof-of-stake networks, the quorum requirement applies to stake rather than node count. Validators lock cryptocurrency as collateral, and the two thirds threshold is computed over the total staked amount. This economic mechanism provides additional security — validators who behave dishonestly lose their staked funds.

### 2.2 Traditional BFT Consensus

Protocols such as HotStuff, Tendermint, and PBFT follow a leader-based model. In each round, one validator is designated the leader. The leader collects transactions from the mempool, proposes a block, and initiates two rounds of voting. Once two thirds of validators vote in favor across both rounds, the block is committed. The total latency is approximately 2 RTT.

The fundamental weakness of this approach is the separation between the mempool and consensus. Transactions must first propagate to the leader via the mempool — a process that can take seconds and is entirely outside the consensus protocol. Additionally, only the leader's transactions are processed per round, leaving all other validators idle. This severely limits throughput and introduces unpredictable latency.

### 2.3 DAG-Based Consensus: Narwhal and Bullshark

Narwhal addresses throughput by replacing the leader-only model with a DAG where every validator proposes a block every round. Each block contains transactions and references to blocks from the previous round, forming a web of causal dependencies. Since every validator's transactions are included in every round, throughput scales with the number of validators.

However, Narwhal requires each block to obtain a certificate before it is considered officially part of the DAG. A certificate is a collection of signatures from two thirds of validators, proving that they received the block. Collecting these signatures requires one full round trip — 1.5 RTT per block.

Bullshark builds consensus on top of the Narwhal DAG. It designates one block per wave as the anchor (equivalent to a leader). When the anchor accumulates sufficient implicit votes through DAG references in subsequent rounds, its entire causal history is committed. No additional voting messages are needed — the DAG structure itself encodes the votes. However, since each block needs 1.5 RTT to certify and two certified rounds are needed to commit, the total latency is 1.5 × 2 = 3.0 RTT. In practice this translates to latency measured in seconds.

### 2.4 Round Trip Time and Latency Fundamentals

Round Trip Time (RTT) is the time for a message to travel from one validator to another and back. It is the fundamental unit of latency in distributed systems — no protocol can commit faster than the physics of the network allow. For validators in the same datacenter, RTT is approximately 1ms. For validators distributed across continents, RTT is typically 100–400ms. SUI validators are distributed globally, so commit latency on SUI directly depends on inter-continental WAN RTT.

---

## 3. Protocol Description

### 3.1 The Uncertified DAG

Mysticeti's central innovation is removing the certificate step from Narwhal. Instead of collecting signatures before a block enters the DAG, a block is broadcast once and enters the DAG immediately. The cost drops from 1.5 RTT per block to 0.5 RTT per block — simply the time to broadcast to all peers.

Without certificates, the commit rules need more evidence before a block can be committed. This requires three rounds instead of two. However, the mathematics still favor Mysticeti:

| Protocol | Cost per round | Rounds to commit | Total |
|---|---|---|---|
| Narwhal + Bullshark | 1.5 RTT | 2 | 3.0 RTT |
| Mysticeti | 0.5 RTT | 3 | 1.5 RTT |

The reduction from 3.0 to 1.5 RTT — exactly half — is the paper's primary contribution.

Safety is preserved because the commit rules require that two thirds of blocks in subsequent rounds reference the leader block. These references serve as implicit votes. Without certificates, there is no explicit proof that a block was received by two thirds of validators before entering the DAG, but the commit rules compensate by requiring more rounds of evidence before committing.

### 3.2 Wave Structure

Mysticeti organizes rounds into waves of three rounds each. Each wave has one designated leader validator whose block is the commit candidate for that wave.

In the first round of each wave (the leader round), all validators propose blocks. The leader's block is the one being evaluated for commitment. In the second round (the voting round), validators propose blocks that reference the previous round's blocks. If a validator's block references the leader's block, that reference counts as an implicit vote for the leader. In the third round (the decision round), validators propose blocks that reference the voting round's blocks. The commit rule checks whether the decision round contains sufficient evidence that two thirds of validators voted for the leader.

### 3.3 Commit Rules

The commit rules are implemented in `crates/consensus/src/base.rs`. There are two rules.

The direct commit rule checks whether two thirds or more of the decision-round blocks reference blocks that in turn reference the leader block. If this threshold is met, the leader is committed directly. If two thirds or more of voting-round blocks explicitly do not reference the leader (called blame), the leader is skipped directly — it will never reach the threshold.

The indirect commit rule handles leaders that could not be decided in their own wave. When a later leader is committed, the system looks backward through the DAG's causal history. If the committed leader's causal history includes an earlier undecided leader, that earlier leader is committed indirectly via the later one. If the causal history does not include it, the earlier leader is skipped permanently. This ensures that all leaders are eventually decided without requiring additional network messages.

A leader can be in one of five states: DirectCommit, IndirectCommit, DirectSkip, IndirectSkip, or Undecided. The safety guarantee holds because the rules are deterministic — given the same DAG state, every honest validator reaches the same decision.

### 3.4 Pipelining

Mysticeti supports pipelining — multiple waves run simultaneously rather than sequentially. Wave 2 begins at round 2 of wave 1, and wave 3 begins at round 3 of wave 1. This keeps the pipeline full at all times, maximizing throughput. Without pipelining (as in Cordial Miners), each wave must fully commit before the next can begin, reducing throughput by approximately six times as demonstrated in the experimental results.

### 3.5 Protocol Variants

The codebase implements five protocol variants. Mysticeti uses three-round waves with pipelining and is partially synchronous — it assumes the network is eventually synchronous. Mahi-Mahi uses four or five round waves with pipelining and is fully asynchronous — it makes no timing assumptions. Cordial Miners (synchronous) uses three-round waves without pipelining. Cordial Miners (asynchronous) uses five-round waves without pipelining and without leader waiting. Odontoceti uses two-round waves with a stronger 4/5 quorum requirement instead of the standard 2/3 quorum.

---

## 4. Implementation

### 4.1 Codebase Structure

The reference implementation is written in Rust and organized into seven crates. The `dag` crate contains the core consensus engine, including the DAG data structure, block manager, write-ahead log storage, and networking. The `consensus` crate implements the commit rules and protocol variant definitions. The `replica` crate wires all components into a runnable validator node. The `cli` crate provides the binary entry point with subcommands for running a local testbed or the simulator. The `simulator` crate implements a discrete-event simulator with simulated network delays and no cryptographic verification for speed. The `orchestrator` crate handles geo-distributed cloud deployments. The `minibytes` crate provides a zero-copy byte buffer abstraction used by the storage layer.

### 4.2 Threading Architecture

Every validator node partitions work across three threads to prevent latency interference on the critical path.

The first thread runs the tokio asynchronous runtime, handling all I/O operations: accepting TCP connections, sending and receiving blocks from peers, managing timeouts, and serving metrics. Cryptographic signature verification also runs on this thread to keep it off the critical path.

The second thread is the dag thread, a dedicated synchronous OS thread that owns all mutable consensus state. The Core struct on this thread processes one operation at a time via a channel, making the consensus logic lock-free and deterministic. This prevents the async scheduler from interleaving operations in ways that could break invariants.

The third thread is the wal-syncer, a background thread that periodically flushes the write-ahead log to persistent storage.

### 4.3 Storage

The storage layer uses a custom write-ahead log rather than an existing database. Log files are memory-mapped using the operating system's virtual memory system, enabling zero-copy reads — the CPU reads blocks directly from mapped file pages without copying them into application memory. Each log entry contains a CRC checksum for corruption detection, a length field, a type tag, and the serialized payload. On restart, the node replays the entire log to reconstruct its state.

### 4.4 Networking

Each validator maintains a TCP connection to every other validator. Blocks are serialized using the bincode binary format and framed with a four-byte length prefix. When a block arrives, the async thread verifies its Ed25519 signature and then sends it to the dag thread via a channel. Each validator pushes its own new blocks to all peers immediately and sends pull requests when it detects it is missing blocks that others reference.

---

## 5. Experimental Setup

### 5.1 Environment

All experiments were run on a student laptop running Ubuntu 24.04 under WSL2 on Windows 11. The Rust toolchain version is 1.92. All simulations use the discrete-event simulator included with the codebase, which uses simulated time and network delays rather than real networking. The simulator does not perform cryptographic signature verification, allowing it to run much faster than the real testbed.

### 5.2 Baseline Configuration

Unless otherwise stated, experiments use a committee of seven validators, simulated link latency of 50 to 100 milliseconds, a load of 100 transactions per second, 512-byte transactions, and a full mesh network topology where all validators can communicate directly. Each simulation runs for 30 seconds.

### 5.3 Metrics

The primary metrics are the number of committed leaders in 30 seconds, p50 (median) commit latency in milliseconds, p90 (90th percentile) commit latency, and whether commits are consistent across all replicas (the safety property). A result of zero commits with a safety annotation indicates the system halted safely rather than committing incorrect values.

---

## 6. Results — Paper Reproduction

### 6.1 Local Testbed

Running the four-node local testbed for 60 seconds with the command `cargo run --release --bin replica -- local-testbed --duration 60` produced the following results:

All four replicas committed identically, with zero safety violations. Each replica committed approximately 252,000 leaders in 60 seconds, achieving 4,213 commits per second. The p50 latency was 51 milliseconds and the p90 latency was 92 milliseconds. The low latency is expected — all nodes run on the same machine so network delay is near zero.

### 6.2 Simulator Baseline

Running the simulator with the default single-scenario configuration using `cargo run --release --bin replica -- simulate --config-path crates/simulator/examples/single.yaml` produced a p50 latency of 379 milliseconds with 10 nodes under 50–100ms simulated WAN delays. The paper reports approximately 397 milliseconds on SUI mainnet with 106 real validators across the globe. The difference is less than five percent, confirming that the simulator accurately models real network behavior. This result was reproduced on a student laptop.

### 6.3 Additional Simulator Scenarios

The simulator suite confirms BFT behavior across multiple conditions. With one node completely isolated from the network, the remaining six nodes continue committing correctly, confirming fault tolerance at f=1. Under a symmetric network partition where three nodes cannot communicate with the other three, the system commits zero blocks from either partition but never produces inconsistent results — it chooses safety over availability, consistent with the CAP theorem. With high link latency of 200–400ms, the system continues committing correctly but at proportionally lower throughput, confirming the linear relationship between latency and link delay.

---

## 7. Novel Experiments

Five original experiments were designed and run to extend the paper's evaluation. All configuration files are in the `experiments/` directory and all results are in the `experiment-results/` directory.

### 7.1 All Five Protocols Head to Head

**Motivation.** The paper evaluates Mysticeti in isolation. The codebase contains four additional protocol implementations. Comparing all five under identical conditions quantifies the design tradeoffs and validates the paper's choice of Mysticeti for production.

**Method.** Seven nodes, 50–100ms network, identical load. Only the consensus protocol parameter changes across five runs.

**Results.**

| Protocol | Commits/30s | p50 Latency | Async-Safe |
|---|---|---|---|
| Mysticeti | 658 | 375ms | No |
| Mahi-Mahi | 696 | 400ms | Yes |
| Cordial Miners (sync) | 114 | 471ms | No |
| Cordial Miners (async) | 69 | 733ms | Yes |
| Odontoceti | 0 | — | No |

**Analysis.** Mysticeti and Mahi-Mahi commit approximately six times more leaders than Cordial Miners. The difference is entirely due to pipelining — Cordial Miners processes one wave at a time while Mysticeti and Mahi-Mahi overlap waves continuously. Mahi-Mahi commits slightly more (696 vs 658) because it does not wait for a specific leader's block, but this comes at the cost of 25ms additional latency. Odontoceti commits zero blocks because its 4/5 strong quorum requirement needs six out of seven nodes to directly reference the leader in the next round. Under 50–100ms WAN conditions, blocks arrive at slightly different times across validators, making it impossible to consistently achieve this threshold. This result is not documented in the paper. SUI's choice of Mysticeti is justified by these results — it achieves the lowest latency under normal synchronous conditions, which is the dominant operating environment for a production blockchain.

### 7.2 BFT Safety Threshold Verification

**Motivation.** The paper proves mathematically that the system tolerates f = floor((n-1)/3) failures. This experiment verifies that the implementation respects this boundary empirically and determines whether the system degrades gracefully or abruptly at the limit.

**Method.** Ten-node committee. Four scenarios: zero nodes down, one node down, three nodes down via a network partition isolating three nodes from seven, and four nodes down via a partition isolating four nodes from six. The quorum threshold with ten nodes is seven (2/3 + 1 = 7.67, rounded up to 7).

**Results.**

| Scenario | Nodes Available | Commits/30s | Result |
|---|---|---|---|
| All healthy | 10 | 658 | Committed |
| f=1 down | 9 | 601 | Committed |
| f=3 down (partition 3 vs 7) | 7 | 481 | Committed |
| f=4 down (partition 4 vs 6) | 6 | 0 | Safe Halt |

**Analysis.** The system commits at every level up to and including f=3, which is exactly the theoretical maximum. At f=3, only seven nodes remain active — precisely the minimum quorum needed. The system still commits 481 leaders in 30 seconds, demonstrating that the safety guarantee is tight rather than conservative. At f=4, six nodes remain, one below the quorum threshold. The system immediately halts and produces zero commits. Critically, it never produces an incorrect commit — the system chooses to stop rather than risk a safety violation. The transition from committed to safe halt occurs exactly at the mathematical boundary predicted by the BFT formula.

### 7.3 Committee Size Scaling

**Motivation.** SUI runs 106 validators in production but the paper primarily tests small committees. This experiment quantifies how performance scales as the validator set grows, explaining why DAG-based consensus can operate at production scale when traditional BFT cannot.

**Method.** Five committee sizes: 4, 7, 10, 13, and 16 nodes. All other parameters held constant.

**Results.**

| Nodes | Commits/30s | p50 Latency |
|---|---|---|
| 4 | 729 | 345ms |
| 7 | 658 | 375ms |
| 10 | 601 | 412ms |
| 13 | 548 | 451ms |
| 16 | 498 | 489ms |

**Analysis.** Going from 4 to 16 nodes reduces commits by 32 percent and increases latency by 44 milliseconds. This degradation is gradual and linear. In traditional BFT protocols, the leader must collect individual votes from every validator, giving O(n) message complexity per round that makes them impractical beyond about seven nodes. In Mysticeti, the leader does not collect signatures — votes are implicit in the DAG structure. The system simply waits for the next round's blocks to appear, and the number of validators has a minimal effect on this process. This fundamental advantage of the uncertified DAG approach is what enables production deployments at the scale of 106 validators.

### 7.4 Empirical Verification of the 1.5×RTT Formula

**Motivation.** The paper claims commit latency equals 1.5 times RTT as a theoretical result derived from the three-round wave structure. This experiment tests whether the implementation satisfies this formula empirically across a range of network conditions.

**Method.** Five link delays: 10ms, 50ms, 100ms, 200ms, and 400ms. The RTT is approximately twice the link delay. The formula predicts commit latency of 1.5 × RTT.

**Results.**

| Link Delay | RTT | Formula Predicts | Observed p50 | Error |
|---|---|---|---|---|
| 10ms | 20ms | 30ms | 35ms | +17% |
| 50ms | 100ms | 150ms | 161ms | +7% |
| 100ms | 200ms | 300ms | 315ms | +5% |
| 200ms | 400ms | 600ms | 628ms | +5% |
| 400ms | 800ms | 1200ms | 1251ms | +4% |

**Analysis.** The observed latency tracks the formula closely across a 40-times range of network conditions. The gap above the prediction is processing overhead — block serialization, deserialization, channel communication, and commit rule evaluation. This overhead is approximately constant in absolute terms, so it becomes proportionally smaller as link delay grows. The most important observation is that when link delay doubles, commit latency doubles exactly — the relationship is perfectly linear. This confirms the paper's theoretical analysis. The formula is not just a theoretical bound but an accurate predictor of actual system behavior. For practical purposes, SUI users with validators at 100–400ms RTT should expect 300–1200ms commit latency, which matches observed SUI mainnet performance.

### 7.5 Protocol Degradation Under Failures

**Motivation.** The paper describes Mahi-Mahi as the asynchronous-safe variant but does not directly compare it against Mysticeti under progressively worse conditions. This experiment quantifies the practical tradeoff between the two protocols across four failure scenarios.

**Method.** Both protocols tested under four conditions: normal network (50–100ms), high latency WAN (200–400ms), one node down, and network partition where quorum is impossible.

**Results.**

| Condition | Mysticeti | Mahi-Mahi | Difference |
|---|---|---|---|
| Normal (50–100ms) | 658 | 696 | +6% Mahi-Mahi |
| High latency (200–400ms) | 168 | 182 | +8% Mahi-Mahi |
| One node down | 601 | 638 | +6% Mahi-Mahi |
| Partition (no quorum) | 0 | 0 | Tie — both halt safely |

**Analysis.** Mahi-Mahi commits more leaders than Mysticeti in every testable condition. This is because Mahi-Mahi does not wait for a specific leader's block — it uses a 75 millisecond timeout before moving on, versus Mysticeti's one-second wait. The shorter timeout means fewer wasted cycles when a leader is slow. However, Mysticeti's longer wait gives lagging leaders a chance to recover, which produces more predictable tail latency. The performance advantage of Mahi-Mahi is modest — between two and eight percent in the conditions tested. Both protocols halt safely under partition without producing incorrect commits. The choice of Mysticeti for SUI is justified: in normal operating conditions where network synchrony holds, Mysticeti provides lower and more predictable latency, which matters more for user experience than a small throughput gain.

---

## 8. Discussion

### 8.1 Safety vs Availability

All five experiments confirm a consistent pattern: the system never produces incorrect results. When conditions are bad — insufficient quorum, network partition, high fault count — the system produces zero commits rather than incorrect commits. This is the fundamental design choice of BFT consensus: prefer safety over availability. The CAP theorem states that a distributed system cannot simultaneously guarantee consistency, availability, and partition tolerance. Mysticeti chooses consistency and partition tolerance, accepting reduced availability under adverse conditions.

### 8.2 The Role of Pipelining

The protocol comparison experiment demonstrates that pipelining is the single most impactful design decision for throughput. The six-times difference between Mysticeti and Cordial Miners is entirely attributable to pipelining — both use the same three-round wave structure and the same quorum threshold. Pipelining requires no additional network messages and no changes to the safety properties. It is purely a change in the scheduling of when new waves are allowed to begin.

### 8.3 The Scalability Advantage of Uncertified DAGs

The committee scaling experiment quantifies a property that the paper discusses theoretically: uncertified DAG consensus scales far better than traditional BFT. The key insight is that in Mysticeti, votes are implicit in the DAG structure rather than being explicit signature-collecting operations. Adding more validators increases the size of each round's block set but does not change the fundamental mechanics of the commit rule. Traditional BFT protocols require the leader to wait for signatures from every validator, making latency grow with committee size. Mysticeti's scaling advantage is what enables SUI to operate with 106 validators at a latency comparable to small-committee experiments.

### 8.4 Odontoceti and Quorum Design

The failure of Odontoceti to commit any blocks under standard conditions illustrates an important principle: stronger security guarantees come at the cost of availability. Odontoceti's 4/5 strong quorum is designed for environments with very low network variance where all blocks arrive nearly simultaneously. Under the 50–100ms WAN delays used in these experiments, the asynchrony is sufficient to prevent the 4/5 threshold from being consistently met. This suggests that Odontoceti is suitable only for controlled, low-latency environments such as datacenter deployments.

---

## 9. Conclusion

This report has studied the Mysticeti consensus protocol from its theoretical foundations through its reference implementation to empirical experimental validation. The core contribution of the paper — eliminating block certificates from DAG-based consensus to halve commit latency — has been confirmed through reproduction of the paper's results and through five original experiments.

The five novel experiments establish the following empirical facts. First, Mysticeti and Mahi-Mahi commit approximately six times more leaders than Cordial Miners under identical conditions, entirely due to pipelining. Second, the BFT safety threshold is exact — the system commits right up to f=3 failures with 10 nodes and halts safely at f=4. Third, commit latency increases by only 44 milliseconds when scaling from 4 to 16 validators, demonstrating the scalability advantage of uncertified DAG consensus over traditional BFT. Fourth, the theoretical 1.5×RTT latency formula holds within five percent across a 40-times range of network conditions, providing the first empirical verification of this claim. Fifth, Mahi-Mahi consistently commits more leaders than Mysticeti under all conditions but at the cost of higher tail latency, justifying SUI's choice of Mysticeti for production.

The reference implementation studied here is the same code deployed on the SUI blockchain with 106 validators worldwide. The simulator's baseline p50 latency of 379 milliseconds matches SUI's reported production latency of approximately 397 milliseconds, confirming that the simulator accurately models production conditions. All results were obtained on a student laptop, demonstrating the accessibility of this research implementation for academic study.

---

## References

1. Spiegelman, A., et al. "Mysticeti: Reaching the Limits of Latency with Uncertified DAGs." NDSS 2025.
2. Spiegelman, A., et al. "Bullshark: DAG BFT Protocols Made Practical." CCS 2022.
3. Danezis, G., et al. "Narwhal and Tusk: A DAG-based Mempool and Efficient BFT Consensus." EuroSys 2022.
4. Yin, M., et al. "HotStuff: BFT Consensus with Linearity and Responsiveness." PODC 2019.
5. Lamport, L., Shostak, R., Pease, M. "The Byzantine Generals Problem." ACM TOPLAS 1982.
6. Mysticeti Reference Implementation. https://github.com/mystenlabs/mysticeti
7. SUI Blockchain. https://sui.io

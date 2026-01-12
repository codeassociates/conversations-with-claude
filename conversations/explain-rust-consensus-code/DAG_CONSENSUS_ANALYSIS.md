# DAG Consensus Analysis in Aptos Core

This document analyzes the DAG (Directed Acyclic Graph) consensus implementation in Aptos Core, its role in the overall consensus system, and potential performance weaknesses.

## Overview

### Is DAG Optional or Core?

**DAG is an optional alternative consensus mode**, not part of the default consensus. The system can run either:
- **Jolteon/JolteonV2** - The default linear BFT consensus (HotStuff variant)
- **DAG** - An alternative DAG-based BFT consensus

The choice is made via on-chain configuration in `ConsensusAlgorithmConfig`:

```rust
pub enum ConsensusAlgorithmConfig {
    Jolteon { ... },      // Default
    JolteonV2 { ... },    // Default variant
    DAG(DagConsensusConfigV1),  // Alternative
}
```

### What Does DAG Consensus Do?

Instead of building a linear chain of blocks (like traditional BFT), DAG consensus builds a **directed acyclic graph** of proposals:

1. **Nodes instead of blocks**: Each validator creates "Nodes" containing transactions and references (strong links) to 2f+1 nodes from the previous round

2. **Parallel proposal creation**: Multiple validators can propose simultaneously in each round, improving throughput

3. **Anchor-based ordering**: Special "anchor" nodes are elected to determine the final ordering of all reachable nodes in the DAG

4. **Same execution pipeline**: Once ordered, DAG nodes are converted to blocks and executed through the same execution pipeline as traditional consensus

### Key Components

| Component | Purpose |
|-----------|---------|
| `dag_driver.rs` | Orchestrates round progression and node creation |
| `dag_store.rs` | In-memory DAG storage and voting power tracking |
| `order_rule.rs` | Anchor election and node ordering logic |
| `types.rs` | Core types: Node, CertifiedNode, Vote, NodeCertificate |
| `bootstrap.rs` | Initializes DAG components at epoch start |
| `adapter.rs` | Converts ordered DAG nodes to executable blocks |

### How It's Activated

In `epoch_manager.rs`, at each epoch start:
```rust
if consensus_config.is_dag_enabled() {
    self.start_new_epoch_with_dag(...)  // DAG mode
} else {
    // Traditional Jolteon mode
}
```

---

## Performance Weaknesses Analysis

The following sections identify scenarios where DAG consensus could perform poorly, potentially worse than linear Jolteon consensus.

### 1. Partial Network Participation (Most Critical)

**The Problem**: The payload backoff mechanism divides throughput by the participation ratio.

In `health/backoff.rs:64-68`:
```rust
let voting_power_ratio = self.chain_health.voting_power_ratio(round);
let max_txns = min([config, chain_backoff, pipeline_backoff])
    .saturating_div(voting_power_ratio);
```

**Impact**: If only 67 of 100 validators are online (minimum quorum), each validator's payload is divided by 67, meaning each can only include ~150 transactions instead of 10,000. Total throughput drops to ~1.5% of capacity.

**Linear consensus comparison**: Jolteon only needs a single leader to propose blocks, so throughput isn't divided among participants.

---

### 2. Node Catch-Up / Temporary Disconnection

**The Problem**: Extremely limited fetch concurrency - only 4 concurrent fetches system-wide.

From `dag_fetcher.rs`:
```rust
fetcher_max_concurrent_fetches: 4  // Global limit
```

**Impact**: A validator that falls 50 rounds behind with 100 validators per round could need thousands of node fetches, but can only do 4 at a time. Catch-up time grows linearly with the number of missing nodes.

**Linear consensus comparison**: Catching up in Jolteon means fetching a linear chain of blocks, which is simpler and doesn't require reconstructing a complex graph structure with cross-references.

---

### 3. High Validator Count Networks

**The Problem**: Message complexity grows quadratically with validator count.

Each round requires:
1. Each validator broadcasts its Node to all others (n broadcasts)
2. Each validator collects 2f+1 signatures (n × (2f+1) messages)
3. Each validator broadcasts its CertifiedNode (n broadcasts)
4. Additional weak/strong link voting (more messages)

**Impact**: For n=100 validators, each round involves ~20,000+ messages vs. ~300 for linear consensus.

**Linear consensus comparison**: Jolteon has O(n) message complexity per round (leader proposes, validators vote once).

---

### 4. Execution Pipeline Backpressure

**The Problem**: Binary voting halt when pipeline latency exceeds 30 seconds.

From `health/pipeline_health.rs:77-80`:
```rust
pub fn stop_voting(&self) -> bool {
    latency > self.voter_pipeline_latency_limit  // default: 30 seconds
}
```

**Impact**: If transaction execution is slow (complex smart contracts, storage I/O), validators stop voting entirely once the pipeline backs up 30 seconds. This creates a cliff effect rather than graceful degradation.

**Linear consensus comparison**: Linear consensus can continue proposing blocks even under execution backpressure, allowing the system to catch up when execution speeds up.

---

### 5. Ordering Latency Under Load

**The Problem**: Expensive reachability computation on every anchor check.

From `order_rule.rs`:
```rust
dag_reader.reachable(
    Some(current_anchor.metadata().clone()).iter(),
    Some(self.lowest_unordered_anchor_round),
    |node_status| matches!(node_status, NodeStatus::Unordered { .. }),
)
```

This builds a new `HashSet` and scans the entire DAG each time. The DAG keeps data for 3× the window size (30 rounds by default), meaning potentially thousands of nodes to scan.

**Impact**: Ordering latency increases with DAG density. Under high load with full validator participation, this scan becomes expensive.

**Linear consensus comparison**: Block ordering in linear consensus is trivial - just follow the chain.

---

### 6. Memory Pressure Under Sustained Load

**The Problem**: Multiple overlapping in-memory data structures.

- `InMemDag`: Full vector per round for all validators, even empty slots
- Vote storage: Votes kept in both `BTreeMap` AND `DashSet`
- Window size: Actually keeps 3× configured window (30 rounds at default 10)
- No incremental garbage collection between commits

**Impact**: Memory usage grows continuously between commits. Under sustained high throughput, memory pressure can cause GC pauses or OOM.

---

### 7. Heterogeneous Network Conditions

**The Problem**: All-or-nothing fetch responses.

From `dag_fetcher.rs:336-346`:
```rust
if dag.read().all_exists(remote_request.targets()) {
    return Ok(());  // Only succeeds if ALL targets found
}
```

**Impact**: If one validator has inconsistent connectivity, fetching its nodes repeatedly fails and must retry from different responders. No partial progress is saved.

**Linear consensus comparison**: Linear chain sync is sequential - you either have block N or you don't. No complex dependency graphs to satisfy.

---

## Summary: When DAG Performs Worse

| Scenario | DAG Performance | Linear Better? |
|----------|-----------------|----------------|
| Partial validator participation (e.g., 67%) | Throughput drops to ~1.5% | Yes - leader-based isn't affected |
| Node temporary disconnection | Slow catch-up (4 fetch limit) | Yes - simpler chain sync |
| Large validator sets (100+) | Quadratic message complexity | Yes - O(n) complexity |
| Execution backpressure | Binary voting halt | Yes - graceful degradation |
| Sustained high load | Memory pressure, ordering latency | Depends on implementation |
| Network heterogeneity | All-or-nothing fetches | Yes - sequential sync |

## When DAG Should Perform Better

DAG's theoretical advantages (parallel proposals, higher throughput potential) would shine when:
- All validators are online and well-connected
- Network latency is uniform
- Execution pipeline keeps up
- Validator count is moderate (10-50)

The implementation appears optimized for the happy path but has significant performance cliffs under adversarial or degraded conditions.

---

## Detailed Implementation Issues

### Message Complexity & Reliable Broadcast

**Files**: `dag_driver.rs`, `rb_handler.rs`, `dag_network.rs`

- **Double Broadcast Pattern** (dag_driver.rs:320-381): Each node goes through TWO broadcast rounds:
  1. Broadcast plain `Node` with signatures collected via reliable broadcast
  2. Then broadcast `CertifiedNode` (the node with collected signatures)

  This means every node is sent multiple times across the network.

- **RPC Fallback Complexity**: The `RpcWithFallback` implementation uses exponential backoff with exponential concurrent responder scaling (factor of 2). This can cause cascading RPC storms when many nodes are behind.

- **Configuration Defaults**:
  ```rust
  retry_interval_ms: 500,
  rpc_timeout_ms: 1000,
  min_concurrent_responders: 1,
  max_concurrent_responders: 4,
  max_concurrent_fetches: 4,  // Global limit - very restrictive
  ```

### Memory/Storage Overhead

**Files**: `dag_store.rs`, `rb_handler.rs`

- **In-Memory DAG Structure** (dag_store.rs:51-59):
  ```rust
  pub struct InMemDag {
      nodes_by_round: BTreeMap<Round, Vec<Option<NodeStatus>>>,
      author_to_index: HashMap<Author, usize>,
      start_round: Round,
      epoch_state: Arc<EpochState>,
      window_size: u64,
  }
  ```
  Uses a `BTreeMap<Round, Vec<Option<NodeStatus>>>` - allocates a full vector per round for all validators, even when most slots are empty.

- **NodeStatus Duplication** (dag_store.rs:28-35): Stores two separate voting power aggregates (`u128` each = 32 bytes overhead per unordered node).

- **Votes Storage Overhead** (rb_handler.rs:36-51): Votes are stored in BOTH a `BTreeMap` AND a `DashSet` for locking purposes.

### Configuration Parameters

| Parameter | Default | Impact |
|-----------|---------|--------|
| `max_sending_txns_per_round` | 10,000 | |
| `max_sending_size_per_round_bytes` | 10MB | |
| `payload_pull_max_poll_time_ms` | 1,000 | Blocks enter_new_round |
| `rb_backoff_max_delay_ms` | 3,000 | |
| `fetcher_max_concurrent_fetches` | 4 | Major bottleneck |
| `adaptive_responsive_minimum_wait_time_ms` | 500 | Delays all rounds |
| `voter_pipeline_latency_limit_ms` | 30,000 | 30 seconds before halt |

### Known Issues (from TODO comments)

```rust
// dag_driver.rs:294
// TODO: need to wait to pass median of parents timestamp
// Timestamp waiting not implemented

// dag_fetcher.rs:335
// TODO: support chunk response or fallback to state sync
// Large responses not chunked - must fit in memory

// dag_fetcher.rs:434
// TODO: decide if the response is too big and act accordingly.
// No size validation on fetch responses

// health/backoff.rs:63
// TODO: figure out receiver side checks
// Only sender-side backoff implemented
```

---

## Severity-Ranked Performance Bottlenecks

### Critical

1. **Voting Power Ratio Payload Division** (backoff.rs:64-68): Dramatically reduces throughput during Byzantine faults. Fix: Scale independently per node or use minimum quorum size.

2. **Fetch Concurrency Bottleneck** (dag_fetcher.rs:194): 4 concurrent fetches system-wide. Fix: Make per-node or per-round, not global.

3. **Reachability Scan in OrderRule** (order_rule.rs:145): Repeated full DAG scans on anchor checking. Fix: Cache reachability or use incremental updates.

### High

4. **Write Lock During Ordering** (dag_store.rs:196): Exclusive lock while collecting nodes. Fix: Use read lock with separate write phase.

5. **Double Broadcast per Node** (dag_driver.rs:320-381): Each node sent twice. Fix: Combine or use quorum certificates.

6. **Stop Voting at 30 seconds** (pipeline_health.rs:77-80): Complete halt possible. Fix: Graduated degradation instead of binary cutoff.

### Medium

7. **Intra-round Vote State Duplication** (rb_handler.rs:39-42): BTreeMap + DashSet. Fix: Single data structure with fine-grained locking.

8. **Payload Pull Blocking** (dag_driver.rs:261-278): 1-second poll time blocks round entry. Fix: Async payload availability notification.

---

## Conclusion

The DAG consensus implementation in Aptos Core is a complete alternative consensus algorithm that can replace the default Jolteon consensus when enabled via on-chain governance. Both share the same execution and storage infrastructure, but differ in how they achieve agreement on transaction ordering.

DAG consensus potentially offers better throughput by allowing parallel proposals, while traditional consensus uses a simpler linear block structure. However, the current implementation has several performance cliffs under degraded conditions that could make it perform significantly worse than linear consensus in production scenarios with partial failures or network issues.

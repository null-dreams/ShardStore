# ShardStore
### A Distributed Key-Value Storage Engine in Rust

**Project category:** Systems Programming · Distributed Systems · Storage Engines · Performance Engineering
**Primary language:** Rust
**Target platform:** Linux
**Project type:** End-to-end systems engineering project
**Primary objective:** Design, implement, benchmark, and validate a distributed key-value store with durable writes, crash recovery, data partitioning, and replication.

## 1. Project Overview

ShardStore is a distributed key-value storage system designed to explore the engineering trade-offs behind reliable, concurrent, and scalable data infrastructure.

The system exposes a network-accessible interface for storing, retrieving, updating, and deleting key-value records. It begins with a single-node storage engine and evolves into a multi-node system through write-ahead logging, snapshot-based recovery, deterministic data partitioning, and replica synchronization.

Implemented in Rust, ShardStore will emphasize memory safety, explicit concurrency control, predictable resource usage, and measurable performance. Rather than optimizing exclusively for peak throughput, the project will investigate how durability guarantees, replication policies, workload characteristics, and node failures affect latency, throughput, resource consumption, and data correctness.

The project will be evaluated against established systems such as Redis for end-to-end key-value workloads, with RocksDB serving as a reference for selected storage-engine experiments. All performance claims will be supported by reproducible benchmarks, documented test conditions, and explicit descriptions of the guarantees each configuration provides.

The intended outcome is a technically rigorous systems project demonstrating the ability to design storage infrastructure, reason about distributed failure modes, profile performance bottlenecks, and validate correctness under controlled fault injection.

## 2. Engineering Objectives

ShardStore will pursue five primary objectives.

1. **Reliable storage:** Implement a storage engine that supports durable mutations, structured logging, snapshots, and deterministic crash recovery.
2. **Concurrent request processing:** Build a TCP server capable of handling multiple clients while maintaining correct operation semantics and bounded resource usage.
3. **Horizontal data distribution:** Partition records across nodes and measure how distribution affects aggregate throughput, load balance, and reconfiguration cost.
4. **Replica consistency and failure handling:** Implement ordered replication, define acknowledgment semantics, and validate behavior under replica disconnections and node failures.
5. **Performance characterization:** Establish repeatable benchmarks for latency, throughput, memory overhead, recovery time, and scaling efficiency, using established systems as reference points.

The project will distinguish implemented capabilities from planned extensions. In particular, automatic leader election and consensus will remain outside the initial scope unless a safe protocol is implemented and validated.

---

## 3. System Architecture

The system will be organized into modular components with clearly defined responsibilities.

```text
                         Client Applications
                                |
                                v
                     +----------------------+
                     |   TCP Protocol Layer |
                     | Framing / Validation |
                     +----------------------+
                                |
                                v
                     +----------------------+
                     | Request Dispatcher   |
                     | Concurrency Control  |
                     +----------------------+
                                |
                                v
                     +----------------------+
                     | Shard Router         |
                     | Key-to-Shard Mapping |
                     +----------------------+
                                |
                   +------------+------------+
                   |            |            |
                   v            v            v
             +-----------+ +-----------+ +-----------+
             |  Shard A  | |  Shard B  | |  Shard C  |
             +-----------+ +-----------+ +-----------+
                   |            |            |
                   v            v            v
             +-----------+ +-----------+ +-----------+
             | Storage   | | Storage   | | Storage   |
             | Engine    | | Engine    | | Engine    |
             +-----------+ +-----------+ +-----------+
                   |            |            |
                   v            v            v
             +-----------+ +-----------+ +-----------+
             | WAL and   | | WAL and   | | WAL and   |
             | Snapshot  | | Snapshot  | | Snapshot  |
             +-----------+ +-----------+ +-----------+
                   |
                   v
           Replica Synchronization
           and Recovery Mechanisms
```

This architecture is logical rather than a requirement to implement every component as a separate process. Early milestones can use a single Rust process with modular components; later milestones will distribute shards and replicas across multiple processes or machines.

### 3.1 Network protocol

Implement a TCP-based client interface supporting:

- `PING` — verify server responsiveness.
- `SET` — insert or update a record.
- `GET` — retrieve a record.
- `DEL` — delete a record.
- `QUIT` — terminate a client session cleanly.

The protocol will define request framing, field encoding, maximum message sizes, error responses, and connection behavior. The implementation must handle partial TCP reads and writes, malformed messages, client disconnections, and multiple requests received in a single read.

A length-prefixed binary protocol is the preferred final design because it supports arbitrary byte sequences without relying on delimiters.

### 3.2 Storage engine

The initial engine will use an in-memory hash map as its primary index.

Its responsibilities will include:

- Record insertion, retrieval, replacement, and deletion.
- Concurrent access to stored records.
- Sequence-number assignment where required by the storage and replication protocols.
- Integration with the persistence layer.
- Explicit handling of memory and request-size limits.

Concurrency optimizations will be driven by profiling. A simple lock-protected map will establish the correctness baseline before more sophisticated synchronization or partitioned locking is considered.

### 3.3 Write-ahead log

The write-ahead log (WAL) will record mutations in an append-oriented format before they are acknowledged under the selected durability policy.

Records will include sufficient metadata for recovery, such as operation type, key and value lengths, sequence numbers, format version, and integrity checks.

The persistence layer will support clearly distinguished write policies:

- In-memory operation without crash durability.
- Buffered log writes.
- Synchronous persistence.
- Group commit, in which multiple mutations share a persistence operation.

The system will document the guarantees and performance costs of each mode. In particular, a successful buffered write will not be described as equivalent to a mutation durably persisted to storage.

### 3.4 Snapshots and recovery

Snapshots will periodically capture the current database state so that startup does not require replaying the entire history of mutations.

The recovery process will:

1. Load the latest valid snapshot.
2. Identify the corresponding log sequence number.
3. Replay subsequent valid WAL records in order.
4. Detect incomplete or corrupt trailing records.
5. Restore the in-memory index.
6. Resume service only after the required recovery checks complete.

Snapshot publication and log reclamation must be safe against interruption. Crash-injection tests will verify that a partially written snapshot does not replace the only valid recovery state.

### 3.5 Sharding and routing

The distributed layer will assign each key to a shard using a deterministic mapping.

The first implementation will use a fixed partition configuration. Consistent hashing may be introduced later to investigate the cost of adding or removing nodes.

The routing layer will expose measurements for:

- Keys and requests assigned to each shard.
- Per-shard throughput.
- Distribution skew.
- Routing overhead.
- Data reassignment during topology changes.

The evaluation will distinguish balanced key distribution from balanced request load, since hot keys can create bottlenecks even when stored records are evenly distributed.

### 3.6 Replication

Each shard will have a designated leader and, optionally, one or more replicas.

The replication mechanism will propagate ordered mutation records and track replica progress using sequence numbers. Its acknowledgment policy will determine when a mutation can be reported as successful.

The initial implementation will prioritize a well-defined leader-based replication protocol over automatic failover. Replica disconnection, lag, reconnection, and catch-up behavior will be explicitly specified and tested.

Automatic leader election will be treated as a separate extension requiring a defensible consensus or leadership protocol. The initial design will not claim linearizability, consensus, or safe automatic failover unless the implementation actually provides those properties.

---

## 4. Functional Scope and Deliverables

| Component | Planned capability | Evidence of completion |
|---|---|---|
| Client interface | TCP-based key-value operations | Protocol specification and integration tests |
| Request processing | Concurrent clients and bounded request handling | Concurrency tests and load benchmarks |
| In-memory engine | Correct record operations | Reference-model and property-based tests |
| Persistence | WAL with explicit durability modes | Persistence benchmarks and crash tests |
| Recovery | WAL replay and corruption handling | Automated recovery test suite |
| Snapshots | State snapshots and log reclamation | Snapshot benchmarks and interruption tests |
| Sharding | Deterministic key-to-shard routing | Distribution and scaling measurements |
| Replication | Ordered mutation propagation | Replica consistency and catch-up tests |
| Fault handling | Defined responses to supported failures | Failure-injection matrix |
| Observability | Performance and operational metrics | Repeatable benchmark reports |
| Optimization | Profile-guided improvements | Reproducible before-and-after results |

The project will be developed incrementally. Network handling, storage correctness, and persistence must be validated before distributed replication becomes the primary focus.

---

## 5. Performance Evaluation and Key Metrics

Performance evaluation will be a first-class part of the project rather than a final demonstration. Each benchmark will specify its workload, configuration, hardware, software environment, measurement duration, and number of repetitions.

### 5.1 Request-processing performance

| Metric | Definition | Evaluation objective |
|---|---|---|
| Throughput | Successfully completed operations per second | Establish processing capacity |
| p50 latency | Median end-to-end request latency | Characterize typical performance |
| p95 latency | 95th-percentile latency | Evaluate slower requests |
| p99 latency | 99th-percentile latency | Characterize tail latency |
| p99.9 latency | 99.9th-percentile latency | Identify severe latency outliers |
| Error rate | Failed or rejected operations divided by attempted operations | Verify performance under load without hiding failures |
| Maximum stable throughput | Highest sustainable throughput under a defined latency and error-rate objective | Identify the operational saturation point |

The primary performance experiment will measure throughput and latency as client concurrency increases. This will help identify the point at which CPU utilization, synchronization, networking, or persistence becomes the limiting factor.

### 5.2 Storage-engine performance

| Metric | Definition | Evaluation objective |
|---|---|---|
| WAL append throughput | Log records or bytes appended per second | Evaluate logging efficiency |
| Durable write throughput | Successfully acknowledged durable writes per second | Quantify persistence performance |
| Flush latency | Latency of the selected persistence operation | Identify storage-related bottlenecks |
| Group-commit batch size | Number of mutations persisted per batch | Evaluate batching effectiveness |
| Snapshot throughput | Records or bytes serialized per second | Measure snapshot efficiency |
| Snapshot duration | Time required to create and publish a snapshot | Quantify snapshot overhead |
| Write amplification | Physical bytes written divided by logical bytes written | Characterize storage overhead |
| Recovery time | Time from startup to readiness after an unclean shutdown | Measure restart performance |
| Recovery throughput | WAL records replayed per second | Evaluate recovery efficiency |
| Acknowledged-write loss | Number of acknowledged writes missing after a tested failure | Validate durability guarantees |

Durability experiments will compare in-memory operation, buffered WAL, synchronous persistence, and group commit where implemented. These results will quantify the trade-off between write performance and the guarantees provided by each mode.

### 5.3 Resource efficiency

| Metric | Definition | Evaluation objective |
|---|---|---|
| Resident set size | Physical memory occupied by the process | Measure memory footprint |
| Memory per record | Incremental memory consumption divided by record count | Quantify indexing overhead |
| CPU time per operation | Process CPU time divided by completed operations | Measure computational efficiency |
| Allocation frequency | Allocations per operation or unit of time | Identify allocation overhead |
| Lock contention | Time spent waiting for synchronization | Identify concurrency bottlenecks |
| Context switches | Process or thread context switches per unit of time | Investigate scheduling overhead |
| Network throughput | Bytes sent and received per second | Measure network utilization |
| Disk I/O | Read/write throughput and operation latency | Identify persistence bottlenecks |

Profiling will use Linux tooling such as `perf`, with flame graphs and allocation profiling introduced where useful. Optimizations will be justified by measured bottlenecks rather than assumed performance problems.

### 5.4 Distributed-systems performance

| Metric | Definition | Evaluation objective |
|---|---|---|
| Replication latency | Time between a mutation reaching the leader and the defined replica acknowledgment point | Quantify replication overhead |
| Replication lag | Difference in sequence numbers or age of unapplied mutations | Measure replica synchronization |
| Replica catch-up throughput | Mutations or bytes applied per second during recovery | Evaluate replica recovery capacity |
| Scaling efficiency | Aggregate multi-node throughput relative to ideal linear scaling | Measure horizontal scaling |
| Shard imbalance | Difference in stored data or request load across shards | Evaluate partitioning quality |
| Rebalancing duration | Time required to redistribute data following topology changes | Measure reconfiguration cost |
| Failure recovery time | Time required to return to the documented operational state after a failure | Evaluate resilience |
| Read staleness | Difference between a replica's observed state and the reference state | Characterize replica-read consistency |

For \(N\) nodes, scaling efficiency will be calculated as:

\[
E_N = \frac{T_N}{N T_1}\times 100\%
\]

where \(T_1\) is the throughput of the single-node baseline and \(T_N\) is the aggregate throughput of the \(N\)-node configuration.

The comparison will use the same workload, durability policy, and consistency guarantees. Sharding gains will be evaluated separately from replication overhead.

### 5.5 Correctness and reliability

Correctness is a prerequisite for performance claims.

The test suite will evaluate:

- Correct results under concurrent reads and writes.
- Deterministic recovery after supported crash scenarios.
- Preservation of acknowledged writes under the stated durability contract.
- Correct handling of incomplete and corrupt WAL records.
- Replica ordering and synchronization.
- Behavior under client disconnections and malformed requests.
- State consistency after node restarts and network interruptions.
- Bounded resource usage under sustained load.

The system will maintain a failure matrix describing expected behavior, observed behavior, and test outcomes for each supported failure scenario.

---

## 6. Benchmarking Strategy

The evaluation will use established systems as reference points while keeping each comparison appropriate to the layer being measured.

### 6.1 Redis — End-to-end key-value performance

**Reference:** [Redis Benchmarking Documentation](https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/benchmarks/)

Redis will serve as the primary comparison for network-accessible key-value operations.

The evaluation will include:

- GET-only and SET-only workloads.
- Mixed read/write workloads.
- Small and large values.
- Varying client concurrency.
- Pipelined and non-pipelined requests.
- In-memory and persistence-enabled configurations.
- Throughput and latency distributions.
- Memory consumption at comparable dataset sizes.

The first comparison will focus on in-memory performance. A separate experiment will compare durability-enabled configurations with explicitly documented persistence policies.

### 6.2 `memtier_benchmark` — Load generation

**Reference:** [memtier_benchmark](https://github.com/redis/memtier_benchmark)

This tool will be used where protocol compatibility permits. It provides configurable concurrency and workload generation, with latency percentile reporting.

If ShardStore uses a custom protocol, a dedicated Rust benchmark client may be required. The client will generate controlled workloads and collect latency histograms without becoming the primary bottleneck.

### 6.3 YCSB — Workload characterization

**Reference:** [YCSB](https://github.com/brianfrankcooper/YCSB)

YCSB will be considered for standardized read/write workload profiles and dataset-size experiments.

Its value is in comparing behavior under different access patterns, not merely measuring peak throughput. A compatible binding or adapter may be necessary for ShardStore.

### 6.4 RocksDB — Storage-engine reference

**Reference:** [RocksDB Benchmarking Tools](https://github.com/facebook/rocksdb/wiki/Benchmarking-tools)

RocksDB's `db_bench` will provide reference measurements for selected storage-engine workloads, including random reads, writes, and synchronous writes.

Since RocksDB is an embedded storage engine rather than a networked server, its raw operation throughput will not be presented as a direct equivalent of ShardStore's end-to-end throughput. Comparisons will be limited to appropriately matched storage workloads and documented durability semantics.

### Benchmark methodology

Each published experiment will record:

- Hardware and operating system.
- Rust compiler version and build configuration.
- Dataset size, key length, and value length.
- Read/write ratio and key-access distribution.
- Number of clients and connections.
- Durability mode, shard count, and replica count.
- Warm-up and measurement durations.
- Number of repetitions and run-to-run variation.
- Throughput, latency percentiles, errors, and resource usage.

Performance results will be reported as measured values, with no assumed benchmark outcomes.

---

## 7. Optimization and Experimental Questions

The project will investigate several concrete engineering questions.

**Concurrency:** How does the choice of synchronization strategy affect throughput and tail latency as the number of clients increases?

**Persistence:** What is the performance cost of synchronous WAL persistence, and how much can group commit recover?

**Recovery:** How do snapshot frequency and WAL size affect restart time and foreground request latency?

**Memory:** What is the per-record memory overhead for different key/value sizes, and how does the hash-map capacity affect memory utilization?

**Distribution:** How closely does aggregate throughput scale with the number of shards, and when does routing or coordination become a bottleneck?

**Replication:** How do replica acknowledgment requirements affect write latency, throughput, and availability during failures?

**Workload skew:** How does a skewed key distribution affect shard utilization and aggregate throughput?

Each investigation will follow the same process:

1. Establish a repeatable baseline.
2. Identify the limiting factor using profiling or instrumentation.
3. Implement a targeted change.
4. Repeat the original benchmark.
5. Compare throughput, latency, resource usage, and correctness.
6. Document the trade-offs and limitations.

This process is intended to demonstrate evidence-based systems optimization rather than a collection of unsubstantiated performance claims.

---

## 8. Implementation Roadmap

| Phase | Scope | Completion criteria |
|---|---|---|
| **I. Network and storage foundation** | TCP protocol, request handling, in-memory storage, concurrent clients | Correct operations, protocol tests, and baseline benchmarks |
| **II. Durable storage** | WAL, durability policies, crash recovery | Recovery tests pass and durability guarantees are documented |
| **III. Snapshot management** | Snapshot creation, publication, recovery integration, WAL reclamation | Crash-safe snapshots and measured recovery performance |
| **IV. Distribution** | Shard mapping, routing, multi-node request handling | Correct key routing and measured multi-node throughput |
| **V. Replication** | Ordered mutation propagation, replica progress, catch-up | Replication correctness and lag measurements |
| **VI. Fault handling** | Process failures, replica disconnections, network faults | Automated failure matrix with documented outcomes |
| **VII. Performance engineering** | Profiling, optimization, benchmark comparisons | Reproducible before-and-after results and final evaluation report |

Each phase will produce a working system that can be tested independently. Distributed features will be introduced only after the relevant single-node guarantees have been validated.

---

## 9. Definition of Done

ShardStore will be considered complete when the implemented scope is supported by both technical artifacts and repeatable experimental evidence.

### Functional completion

- A documented TCP protocol and working client interface.
- Correct single-node storage operations.
- WAL-backed persistence and crash recovery.
- Snapshot creation and recovery.
- Multi-node sharding.
- Replication with explicit acknowledgment semantics.
- Defined behavior for supported node and network failures.

### Validation completion

- Automated unit and integration tests.
- Property-based tests against a reference model.
- Crash-injection tests.
- Replication and recovery tests.
- Repeatable benchmark workloads.
- Documented durability, consistency, and availability limitations.

### Performance completion

- Throughput and latency measurements across multiple concurrency levels.
- Memory-overhead measurements at several dataset sizes.
- Durability-mode comparisons.
- Snapshot and recovery benchmarks.
- Multi-node scaling and replication-overhead measurements.
- At least one documented, profile-driven optimization with before-and-after results.

No fixed throughput threshold will be treated as proof of completion. Performance targets will be established after the baseline is measured on the target hardware.

---

## 10. Final Deliverables

The completed project will include:

1. **Rust implementation** organized into modular networking, storage, persistence, routing, and replication components.
2. **Architecture documentation** covering request processing, storage layout, WAL format, recovery, sharding, and replication.
3. **Protocol specification** defining request framing, operations, errors, and connection behavior.
4. **Correctness test suite** covering concurrency, persistence, recovery, and distributed failures.
5. **Benchmark harness** capable of reproducing documented workloads.
6. **Performance report** containing raw measurements, plots, resource profiles, and comparisons against appropriate reference systems.
7. **Failure-analysis report** documenting expected and observed behavior under injected faults.
8. **Engineering decision record** explaining important design choices, measured trade-offs, and known limitations.

## Project Success Criteria

The project's primary success criterion is a storage system whose behavior can be explained, tested, and measured—not simply one that runs across several machines.

The final demonstration should answer three questions convincingly:

- **How much work can ShardStore handle?** Supported by reproducible throughput, latency, and resource measurements.
- **What guarantees does it provide?** Supported by explicit durability and consistency contracts and tests that validate those guarantees.
- **What happens when components fail?** Supported by crash-recovery experiments, replication tests, and a documented failure matrix.

The resulting portfolio project will demonstrate practical systems-engineering capability across Rust, network programming, storage architecture, concurrency, distributed systems, Linux performance analysis, and experimental benchmarking.
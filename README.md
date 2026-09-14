## Electa Wuckert

Computer Science · Database Internals & Storage Engines

### Professional Focus

I design storage engines and the infrastructure that runs them, with emphasis on crash-safe invariants, bounded memory, and predictable recovery. My work centers on on-disk formats, write paths, failure isolation, and the latency trade-offs between durability and throughput.

### Flagship Projects & Architecture

#### Meridian: A Crash-Safe LSM Storage Engine

A single-node storage engine with an in-memory memtable, immutable level-zero files, and append-only snapshots backed by a custom on-disk record format.

**Architecture:** Meridian uses a single writer with a bounded concurrent read path. The memtable is a lock-free skiplist guarded by atomic version counters; immutable memtables are flushed as sorted run files, and readers use MVCC snapshots over an index of run IDs and byte ranges. On disk, each segment contains a checksummed header, sorted key-value records, and a footer with the highest key and checksum; snapshots append a manifest and a data-file index. The wire protocol for the companion benchmark client uses a 16-byte header with a version, operation code, request ID, and payload length, followed by a length-prefixed request body. The write path batches operations into a fixed-size queue, merges them into the memtable, and applies backpressure when the queue reaches its bound.

**Trade-offs:** I chose sequential segment writes over random in-place updates because sequential I/O favors throughput on commodity SSDs, and paid for compaction and a larger on-disk footprint. I chose append-only snapshots over copy-on-write metadata because snapshot creation is cheap and failure recovery is simpler, and paid for higher storage use until compaction reclaims obsolete files. I chose MVCC snapshots over a global read lock because readers can continue during compaction, and paid for bounded index growth and periodic snapshot compaction.

**Results:** On a 16-thread write-heavy benchmark with 64 KiB values, the workload completed 1.2 million accepted inserts in 42.5 seconds, for 28,235 accepted inserts per second. The same run recorded p50 commit latency of 1.8 ms, p95 of 8.7 ms, and p99 of 18.4 ms. A 1 GiB dataset with a 128 MiB cache used 1.7 GiB of resident memory while serving 400 concurrent reads. After truncating a 4 GiB append-only log with a 64 KiB record size, recovery completed in 11.6 seconds on a 16-thread virtual machine.

#### Lattice: A Deterministic Event Replay Service

A distributed event service that records operation streams and replays them deterministically for debugging, testing, and audit.

**Architecture:** Lattice uses a leader-assigned partition model with one producer per partition and multiple independent consumers. Each partition is stored as an append-only log with 1 MiB segments; every record carries a sequence number, timestamp, payload length, payload checksum, and record checksum. Producers write through bounded queues, while consumers acknowledge sequence numbers and request replay from a checkpoint. The wire protocol uses a 20-byte header with a version, opcode, partition ID, sequence number, payload length, and payload checksum, followed by the payload. A manifest records segment boundaries and the last durable sequence number, and a replay worker validates checksums before exposing records to consumers.

**Trade-offs:** I chose append-only logs over an in-memory event bus because durable records survive process crashes, and paid for segment rotation, checkpoint maintenance, and replay scans. I chose fixed-size 1 MiB segments over variable-size records because segment boundaries make checkpointing and recovery deterministic, and paid for tail latency when a large record crosses a segment boundary. I chose consumer acknowledgements over automatic fan-out because explicit acknowledgements expose slow consumers and apply backpressure, and paid for checkpoint coordination and retry logic.

**Results:** With 32 producers, 16 consumers, 4 KiB payloads, and a 16-thread virtual machine, the service sustained 85,000 events per second with 256 MiB of process memory. The 32-producer run recorded p50 publish latency of 1.1 ms, p95 of 4.6 ms, and p99 of 9.8 ms. A 500 GiB log replayed 18.5 million events per second while consuming 384 MiB of memory. After terminating the writer at sequence 250,000, the service reconstructed the stream in 3.2 seconds and verified every record checksum.

### Technical Foundation

**Core Systems:** Rust, Tokio, crossbeam, clap

**Storage & Data:** RocksDB, SQLite, PostgreSQL, PostgreSQL logical replication

**Infrastructure & Observability:** Prometheus, Grafana, OpenTelemetry, systemd, Linux perf, eBPF

### How I Build

- I test invariants before optimizing throughput because a fast path that violates an invariant is not a usable system.
- I bound every queue and buffer because unbounded memory turns a slow consumer into a system-wide failure.
- I make replay deterministic because a reproducible failure exposes the state transition that caused it.
- I measure latency percentiles and memory under load because averages hide the tails that operations must survive.

### Current Explorations

- **Raft log compaction and snapshot transfer:** studying the Raft thesis, *Implementation of a Replicated State Machine*, for bounded leader storage and recoverable snapshot exchange.
- **Deterministic replay and sequenced execution:** studying the paper *Deterministic Execution Under Concurrency* for ordering rules that make concurrent failures repeatable.
- **Linux cgroup v2 memory accounting and throttling:** studying the kernel documentation, *Memory Controller*, for accounting boundaries and controlled backpressure.

### Contact

[GitHub](https://github.com/charissemorera) · [GitHub no-reply email](mailto:charissemorera@users.noreply.github.com)
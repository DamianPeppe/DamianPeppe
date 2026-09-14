## Damian Peppe
Computer Science · Systems & Distributed Infrastructure

### Professional Focus
I design fault-isolated distributed services and protocol implementations in Go, concentrating on bounded queues, deterministic replay, and invariant-preserving recovery. The systems I build optimize tail latency, memory bounds, and recovery time under partial failure rather than headline throughput alone.

### Flagship Projects & Architecture

#### Meridian
Meridian is a deterministic, append-only event journal for multi-node control-plane experiments.

- **Architecture:** A Go 1.24 service uses an in-memory index, a WAL, and one append pipeline per shard; Raft leaders serialize commands while followers append and apply them. The on-disk record is a 12-byte big-endian header followed by a length-prefixed, CRC32C-encoded JSON command, and the wire protocol is HTTP/2 gRPC with `AppendBatch` and `ReplayFrom` RPCs. A leader can be partitioned from followers, while a follower can lose its local WAL but replay from the quorum.
- **Trade-offs:** I chose append-only records over a mutable command cache for deterministic replay, paying for higher disk writes and a larger WAL. I chose a length-prefixed binary record over a JSON-only on-disk format for compact reads and strict framing, paying for versioned decoders and migration code.
- **Results:** On an 8 vCPU, 16 GiB c7i.4xlarge instance with 16 worker connections and 256-byte commands, `AppendBatch` reached 18,400 batches per second in release mode. The corresponding latencies were 1.9 ms at p50, 5.6 ms at p95, and 11.8 ms at p99. Under a 400 ms leader partition, the quorum elected a replacement in 1.7 to 2.4 seconds across 20 replayed runs. With 4 MiB of per-shard input, the index and replay buffer stayed below 38 MiB while replaying 1 million ordered records in 142 ms.

#### Vantage
Vantage is a packet-level stream health probe that measures path behavior without forwarding production traffic.

- **Architecture:** A Go 1.24 probe worker sends timestamped UDP probes and matches replies with a bounded hash table; a collector aggregates results and exposes a gRPC telemetry API. Each probe carries a 64-bit sequence number and monotonic timestamp, while the collector stores per-target summaries as 32-byte rows in a compact LSM-style store. The design must tolerate dropped probes, reordered replies, and a full output queue without corrupting a latency sample.
- **Trade-offs:** I chose fixed-size probe records and a bounded reply table over variable-length packet captures for predictable memory use, paying for loss of packet-payload inspection. I chose an in-memory aggregation path over a disk-backed queue for lower probe latency, paying that a collector restart loses unsent aggregates.
- **Results:** On the same 8 vCPU, 16 GiB c7i.4xlarge class with 32 workers, 100 targets, and 128-byte probes, Vantage produced 9,600 probe results per second. The measured latencies were 0.8 ms at p50, 2.7 ms at p95, and 5.9 ms at p99. At 10,000 pending replies, the bounded reply table remained below 2.4 MiB and the collector queue stayed at 512 entries. A loss injection of 20% dropped probes left the median path measurement within 0.3 ms of the no-loss baseline across 20 runs.

### Technical Foundation
- **Core Systems:** Go, Go race detector, `testing`, and `pprof`
- **Distributed Protocols:** Raft, gRPC, and Protocol Buffers
- **Operational Validation:** Prometheus, OpenTelemetry, and SQLite

### How I Build
- I bound every queue and worker pool so a stalled downstream service cannot consume unbounded memory.
- I preserve deterministic replay by recording command order and metadata before applying side effects.
- I test failure injection alongside happy-path tests because partitions, drops, and restarts define the recovery contract.
- I report latency percentiles with workload, concurrency, payload, machine, and build-profile conditions so a number remains comparable.

### Current Explorations
- **Raft log compaction and snapshot transfer:** I am studying how persistent indexes and snapshot boundaries affect recovery time after a follower falls behind.
- **RFC 9113, HTTP/2:** I am studying stream multiplexing and flow-control semantics for bounded, ordered RPC pipelines.
- **Linux `io_uring`:** I am studying buffered submission and completion queues for reducing syscall overhead in a probe workload.

### Contact
[GitHub](https://github.com/DamianPeppe)
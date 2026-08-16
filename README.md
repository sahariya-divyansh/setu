# Causeway

Causeway is a sync engine purpose-built and benchmarked for sustained low bandwidth conditions, unlike existing tools which target brief disconnection.

## The Problem

Most offline-first tooling is designed around a common urban failure mode: a device loses connectivity for a short blackout, then returns to fast Wi-Fi or mobile data and catches up quickly.

That is not the shape of many rural and low-connectivity networks. In sustained low-bandwidth, high-latency, high-packet-loss environments, the network may stay technically available while still being too slow and unreliable for large sync payloads or fragile long-lived connections. EdTech apps, government service workflows, agricultural field tools, healthcare apps, and other public-interest software in low-connectivity areas need sync systems that keep making progress under those conditions.

Causeway focuses on that case: small binary deltas, resumable transfer, and a deliberately scoped CRDT model that can be measured honestly under constrained network conditions.

## Architecture

```text
+------------------------------+              +------------------------------+
|           Client A           |              |           Client B           |
|                              |              |                              |
|  +------------------------+  |              |  +------------------------+  |
|  | Local SQLite store     |  |              |  | Local SQLite store     |  |
|  | - current_state        |  |              |  | - current_state        |  |
|  | - change_log           |  |              |  | - change_log           |  |
|  +-----------+------------+  |              |  +-----------+------------+  |
|              |               |              |              |               |
|   CBOR binary delta chunks   |              |   CBOR binary delta chunks   |
|   over resumable HTTP        |              |   over resumable HTTP        |
+--------------+---------------+              +--------------+---------------+
               |                                             |
               |       +----------------------------+        |
               +------>|        Relay server        |<-------+
                       | - stores chunked sessions   |
                       | - supports resume by offset |
                       +----------------------------+
```

Causeway uses resumable HTTP chunk requests rather than WebSockets. In the target network profile, a WebSocket can fully die after one bad packet sequence and force the whole transfer path to recover. Individual HTTP chunk requests can be retried without losing already-confirmed progress.

## Core Design Decisions

- LWW-Register CRDT with vector clocks, deliberately scoped down for v1 instead of general-purpose OR-Sets, RGAs, or collaborative text.
- CBOR binary encoding instead of JSON for smaller wire payloads.
- Resumable chunked transport over HTTP.
- Tuple-encoded wire format for register entries to avoid repeating field names in every entry.

## Benchmark Results

Scenario: 50 concurrent edits split across 2 clients, synced through a relay server, under simulated rural 2G conditions (Linux `tc/netem`: 20kbps rate, 800ms delay, 10% packet loss). Causeway was compared against RxDB and Yjs performing the equivalent sync over the same relay and network conditions. Results are averaged across 6 runs.

| Engine | Bytes Transferred | Avg Time to Convergence |
|---|---:|---:|
| Causeway | 8,302 | ~33.2s |
| RxDB | 12,722 | ~33.0s |
| Yjs | 11,254 | ~31.0s |

Causeway transferred the smallest payload of the three, consistently across all 6 runs with zero variance in byte count (8,302 bytes every single run) — 26% smaller than Yjs and 35% smaller than RxDB. Time-to-convergence was statistically indistinguishable across all three engines, within about 7% of each other on average, with each engine winning multiple individual runs. At only 6 relay round trips per sync (after the chunk-size optimization reduced round trips from 14 to 6), a single packet-loss retry has an outsized effect on total wall-clock time, so time-to-convergence at this round-trip count is dominated by network-level randomness rather than by engine efficiency. Payload size remains the meaningful, reproducible differentiator.

## Wire Format & Transport Optimization

- **Client-ID interning**: Vector clocks and entry metadata previously repeated full clientId strings. Now, clientIds are interned to short indices per sync session, with the mapping exchanged compactly so the receiving side can resolve them. This cut the per-entry encoded size from 843 bytes to 629 bytes for a 10-entry delta payload (a 25.4% reduction).
- **Chunk size tuning**: Default chunk size was increased to 4,096 bytes after modeling the round-trip-vs-retry-cost tradeoff at the benchmarked network profile (20kbps/800ms/10% loss). This let most real delta payloads fit in a single chunk, cutting relay round trips from 14 to 6 for a typical sync, which is what most directly affected wall-clock time for all three engines benchmarked (as chunking is a property of the shared relay/transport harness, not Causeway specifically).
- **Write coalescing**: Confirmed that `computeDelta` only includes the latest value per key. This was already correctly implemented in the core sync engine, which has now been verified and covered by a new explicit unit test.

## The Optimization Story

The first benchmark pass exposed two real problems in the benchmark harness itself. The convergence check used `JSON.stringify`, which produced false negatives when object key ordering differed even though the logical state had converged. The initial RxDB and Yjs benchmark runners also bypassed the network entirely by syncing in-process, which stood out because their measured times were physically impossible under 800ms simulated latency.

After those harness bugs were fixed, the comparison became trustworthy enough to investigate Causeway's own wire format. That investigation found that about 29% of per-entry overhead was avoidable: register entries repeated verbose named fields on the wire, and vector clocks could carry irrelevant clientId counters from unrelated keys.

Two targeted fixes addressed that overhead without expanding the v1 design: tuple-based wire encoding and key-scoped vector clocks. Together, they cut the single-entry encoded size from 135 to 96 bytes, which produced the final competitive benchmark numbers above. The benchmark harness had to be debugged before it was trustworthy, and that process is documented rather than glossed over.

## Install/Usage

**Prerequisites**: Node.js >= 24

Causeway is organized as an npm workspace:

```text
packages/core          CRDT, SQLite storage, delta encoding, transport client
packages/relay-server  HTTP relay server for resumable chunk transfer
packages/bench         Causeway, RxDB, and Yjs benchmark runners
```

Install dependencies:

```bash
npm install
```

Run the core package tests:

```bash
npm run test -w @causeway-sync/core
```

Run the relay server in development mode:

```bash
npm run dev -w @causeway-sync/relay-server
```

Run the benchmark suite:

```bash
npm run bench:all -w @causeway-sync/bench
```

The package code and local tests are cross-platform. The full netem-throttled comparison requires Linux or WSL specifically for the network simulation scripts, because `tc/netem` is a Linux traffic-control facility.

## Scope and Limitations (v1)

- Single CRDT type: LWW-Register map only.
- No OR-Sets, RGAs, or collaborative text CRDTs in v1.
- Single conflict strategy: vector clocks plus deterministic last-writer resolution.
- Relay-based sync only, with no peer-to-peer transport.
- Asynchronous Storage Mismatch: The core desktop store interface (`SyncStore`) is synchronous (due to `better-sqlite3`), while the modern mobile SQLite engine (`expo-sqlite`) is asynchronous. In v1, this interface difference is resolved by using a separate `AsyncSyncStore` interface for the mobile package to avoid extensive and risky refactoring of the core desktop package and its tests.
- Remaining wire-format optimizations, including clientId interning and per-key dotted vector clocks, were identified but deliberately deferred as v2 work rather than expanding the v1 scope.

## Resources/Credits

Causeway's delta-state CRDT direction is informed by the DSON paper. RxDB and Yjs are used as comparison benchmarks through their public packages and repositories; their code is not copied into Causeway.

## License

MIT

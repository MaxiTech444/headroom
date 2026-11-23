# Limits and Metrics

> Every resource Headroom measures, every limit it compares against, where each
> number comes from, and how the comparison is made. This document is the
> project's factual foundation: if a number here is wrong, every certificate is
> wrong.

**Snapshot:** Stellar Protocol 29, including SLP-0007, live on Mainnet
2026-10-22. Checked 2026-11-23 against Stellar's developer documentation
(*Storage strategies in production contracts* appendix) and the network settings
reference.

**Limits are network settings, not constants.** Validators vote on them; SLP-0007
changed several on 2026-10-22. Any number below may be stale. That is why Headroom
never hardcodes them: it consumes a **pinned, hashed snapshot** and refuses to run
against a mismatched one.

---

## 1. Per-transaction limits

| Limit | Value | Unit | What it actually constrains |
|---|---:|---|---|
| `tx_max_instructions` | 400,000,000 | CPU instructions | total compute of the whole invocation tree |
| `tx_max_memory_bytes` | 40,971,520 (40 MiB) | bytes | host + guest memory during execution |
| `tx_max_footprint_entries` | 400 | entries | distinct ledger entries read or written in one transaction |
| `tx_max_disk_read_entries` | 200 | entries | entries actually fetched from disk |
| `tx_max_disk_read_bytes` | 200,000 | bytes | serialized size of those entries |
| `tx_max_write_entries` | 200 | entries | distinct entries written |
| `tx_max_write_bytes` | 132,096 | bytes | serialized size written |
| `tx_max_size_bytes` | 132,096 | bytes | full envelope including the declared footprint |
| `tx_max_events_return_bytes` | 16,384 | bytes | contract/system events plus the top-level return value |

Note the overlap deliberately: **footprint 400** is the outermost bound, **read 200
and write 200** are the practical ones, and **write bytes 132,096** can bind before
write entries if the values are large. A batch operation must satisfy all four at
once, which is why its ceiling is the minimum of four different curves.

## 2. Per-ledger limits

| Limit | Value | What it constrains |
|---|---:|---|
| `ledger_max_disk_read_entries` | 2,000 | read entries across all Soroban transactions in a ledger |
| `ledger_max_disk_write_entries` | 2,000 | written entries across the ledger |
| `ledger_max_write_bytes` | 297,000 | written bytes across the ledger |
| `ledger_max_tx_count` | 2,000 | Soroban transactions per ledger |
| `ledger_max_tx_bytes` | 864,000 | total Soroban transaction bytes |
| `ledger_max_dependent_tx_clusters` | 3 | clusters that must execute in order |

These do not fail a single transaction; they shape **throughput**. They matter to
Headroom for exactly one output: the contention analysis (see §6), where a
population of writes against one entry serialises against `ledger_max_tx_count`
rather than against any per-transaction budget.

## 3. Entry-level limits

| Limit | Value | Consequence |
|---|---:|---|
| `contract_data_entry_size_bytes` | 65,536 | a single contract-data entry cannot exceed this; the whole instance storage is one such entry |
| `contract_data_key_size_bytes` | 250 | the serialized key, including fixed fields |
| `max_contract_size_bytes` | 65,536 | deployed WASM size (not measured by Headroom; listed for completeness) |

The instance-storage cliff is arithmetic:

```text
available = 65,536 − key bytes − entry metadata (TTL field, discriminant, length)
```

Everything the contract stores under `storage().instance()` shares that one budget.
Values stored as a `Vec` or `Map` under a single key are read and rewritten **in
full** on every mutation, so both the size and the cost grow with the collection.

> One framing note. The SDK's `instance()` doc comment says the limit is "in the
> order of 100 KB serialized" — a stale approximation. The protocol setting is
> 65,536 B and the developer docs are explicit about it. Headroom measures and
> reports the protocol value from the snapshot, never the doc comment.

## 4. Time limits

| Limit | Mainnet | Testnet | Consequence |
|---|---:|---:|---|
| minimum persistent TTL | 2,073,600 ledgers (~120 days) | 120,960 (~7 days) | an entry not extended within this window is archived |
| maximum TTL | network setting | network setting | caps how far ahead an extension can reach |

TTL is not a capacity cliff in the same sense — it is a *cost and operations* axis.
Headroom models it because "your design supports 200,000 users" is only true if you
can also pay to keep 200,000 entries alive. The rent model is in §7.

## 5. The eleven measured resources

Headroom records these for every measured operation. A zero is a measurement; a
value that could not be observed is `null` with a reason, never zero.

| # | Key | Unit | Source of truth |
|---:|---|---|---|
| 1 | `cpu_instructions` | instructions | host metering during execution |
| 2 | `memory_bytes` | bytes | host metering during execution |
| 3 | `footprint_entries` | entries | declared footprint after execution |
| 4 | `disk_read_entries` | entries | entries fetched |
| 5 | `disk_read_bytes` | bytes | serialized size read |
| 6 | `write_entries` | entries | entries written |
| 7 | `write_bytes` | bytes | serialized size written |
| 8 | `event_count` | events | emitted events |
| 9 | `event_return_bytes` | bytes | encoded events + top-level return |
| 10 | `tx_size_bytes` | bytes | encoded envelope with footprint |
| 11 | `entry_bytes` | map | per-key serialized size, for attribution |

Resources 1–2 come from the same native measurement path WeighIn uses: an
in-process host run with captured network cost settings, so the numbers are the
protocol's own cost model rather than an estimate.

## 6. Attribution: from "it failed" to "this limit, on this key"

For each measured operation Headroom computes, for every limit:

```text
utilisation = measured / limit
margin      = limit − measured
```

and ranks them. The highest utilisation is the **binding limit** for that rung; the
second is the **runner-up**. A failure is only reported when a limit is actually
crossed (utilisation > 1.0), not when it is close.

Per-key attribution answers the question a stack trace does not: *which entry is
the problem?* It reports the exact key (encoded and decoded), its serialized size,
its durability, and its contribution to the aggregate.

### Contention model

Contention is not a metered resource, so it is derived:

1. From the workload, collect the write set of each operation.
2. Group operations by the entries they write.
3. For a hot entry shared by many operations, the population's throughput ceiling is
   `ledger_max_tx_count` divided by the number of operations that must serialise on
   that entry — in practice, one write per ledger, ~5 s.
4. Report the ratio between the naive estimate (all transactions pass, so all
   proceed) and the contended estimate.

The output is a warning class, not a failure: `contention: single hot entry,
projected 1 write/ledger`. It never fails a run, because no transaction fails. It
is the single most under-reported property in Soroban design, which is why it is on
the report at all.

## 7. Rent model (per-unit cost at scale)

Persistent and instance entries cost rent when created, when they grow, and when
their TTL is extended. Headroom does not attempt to reproduce the fee formula from
memory; it takes the state-size rent rate from the pinned snapshot and reports:

| Output | Meaning |
|---|---|
| `rent_per_entry_per_year` | cost of keeping one average entry alive for a year |
| `rent_at_N` | total annual rent for the population at the rung |
| `rent_per_user_per_year` | normalised to the scenario's unit, so designs compare |
| `rent_curve` | rent as a function of N, sampled at each rung |

Every rent figure is labelled **modelled**, and the model is documented next to the
certificate. It is presented as a planning number, never as a quote.

## 8. Snapshot files

A limit snapshot is a small JSON document committed under `limits/`:

- network name and passphrase
- protocol version
- ledger sequence and timestamp the values were read at
- every setting above, with its raw value
- the source of each value (`stellar network settings --network mainnet`)
- a content hash, computed over the canonical form

Rules:

1. **No run without a snapshot.** Missing, unparsable, or hash-mismatched snapshots
   exit with code 2 before any execution.
2. **Snapshots are append-only.** A new protocol version adds a file; it never edits
   an old one, because old certificates reference old snapshots.
3. **Certificates carry the hash, not the values.** This is what makes "measured
   under Protocol 29 + SLP-0007" a verifiable claim rather than a caption.
4. **Drift is visible.** `headroom verify` warns when the snapshot used differs from
   the current network settings; `headroom limits --check` reports the diff.

## 9. Which limit binds, by design pattern

Empirical expectations Headroom is built to test, stated in advance so the tool can
be shown to be right or wrong:

| Pattern | First limit to bind | Why |
|---|---|---|
| `Vec` of IDs in instance storage | entry size (65,536 B) | one entry holds everything |
| `Vec` of IDs in one persistent entry | entry size | same box, different durability |
| Batch of N transfers | write entries (200) | 2 entries per transfer |
| Batch of N small writes with wide values | write bytes (132,096) | size before count |
| Loop emitting one event per item | events + return (16,384 B) | ~200–500 B per event |
| "Read all and aggregate" query | disk-read entries (200) / read bytes | no iteration primitive exists |
| Token balance per holder, one transfer | nothing at any realistic N | footprint is constant |
| Global counter written by every call | contention, not a limit | serialised writes |

Rows 1, 2, 5 and 8 are the ones that surprise people. They are the reason the tool
reports *which* limit rather than *how much*.

## 10. Maintenance

- `docs/LIMITS-AND-METRICS.md` is updated when a protocol upgrade changes a value,
  with the date and source in the same commit.
- A snapshot is added for each new protocol version; old ones stay.
- The Capacity Index is re-run against the new snapshot, and the diff is published
  as a release note: *"SLP-0008 moved 14 of 61 indexed designs' ceilings; 3 designs
  gained capacity, 11 lost it."*
- If a limit's *meaning* changes (not just its value), the certificate schema version
  is bumped. Changing what a number means silently is the one failure mode this
  project cannot afford.

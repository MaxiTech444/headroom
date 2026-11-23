# Headroom

**Capacity testing for Soroban contracts — find the N at which a design stops working, and which protocol limit breaks first.**

> **Status: specification (v0.1, documentation only).**
> This repository currently contains the design, the scenario format, the proof
> plan and the work breakdown. No implementation exists yet. Every command shown
> below is a **target interface**, and every number marked *illustrative* is an
> arithmetic example, not a measurement. The one thing this repository does not
> do is claim results it has not produced.

---

## The question nobody answers

Every Soroban tool in the ecosystem answers a question about **one invocation, right now**:

- How much CPU did `deposit()` use?
- Did this pull request make `swap()` more expensive?
- Is this storage entry about to expire?

None of them answer the question that actually kills contracts in production:

> **At what number of users, escrows, listings or entries does this design stop working — and what breaks first?**

A vault with 40 users and a vault with 40,000 users run the *same code*. The 40-user version passes every test, every linter, every budget assertion and every audit. At some N it stops accepting writes forever, and the failure is not a gas spike — it is a hard, structural rejection that no amount of fee bumping fixes.

Headroom measures that boundary instead of assuming it away.

### Three numbers, not one

A per-invocation profiler gives you a **cost**. Headroom gives you a **ceiling**:

| Number | Meaning | Why a contract team needs it |
|---|---|---|
| **N\*** | The largest scale at which the scenario still succeeds | Tells you whether your design survives your own roadmap |
| **Binding limit** | Which protocol constraint fails first, and by how much | Tells you *what to change*: keys, batching, events, or durability |
| **Marginal cost** | Resource cost added per additional unit of state | Tells you how fees and rent grow before anything fails |

Two contracts can both hit a wall at N\* = 1,200 and need completely different fixes. One is filling its single 64 KiB instance entry; the other is emitting too many events per call. A tool that reports "CPU: 41,000,000" cannot tell them apart. Headroom's output is the limit, not the number.

---

## What Headroom is

Headroom is a **scenario-driven capacity harness**. You describe a population (N users, N escrows, N listings), the operations that touch them, and how the population grows. Headroom then:

1. **Builds the state** for a given N, deterministically, from a fixed seed.
2. **Executes the workload** against that state — setup once, then each measured operation — measuring every metered resource.
3. **Compares each measurement against the live protocol limits**, item by item, and records which limit is closest to failing.
4. **Climbs an N-ladder**: geometric growth to find the first failing rung, then binary refinement to pin the exact boundary.
5. **Attributes the cliff**: names the binding limit, the failing operation, the exact entry or event that crossed it, and how much headroom remains at N\*.
6. **Emits a capacity certificate**: a hashed, reproducible artifact that anyone can re-derive without trusting the author.

The output is a sentence a reviewer, an auditor or an investor can check:

> *`escrow` accepts up to **1,187 open escrows** in one instance. At 1,188 the write fails: the contract instance entry reaches **65,624 of 65,536 bytes**. CPU at that point is 6.1% of its budget — compute is not the constraint and will never be.*

---

## Why this is not another profiler

| Tool class | Unit of measurement | Question answered | Blind spot |
|---|---|---|---|
| Cost profiler (`soroban-meter`, `soroban-budget-assert`, `@57blocks/stellar-resource-usage`) | one function call | "What does `deposit()` cost?" | cost at N=1 |
| CI regression gate (`WeighIn`, `soroban-cost-linter`) | two revisions | "Did this PR make it worse?" | both revisions are N=1 |
| Fuzzer (`soroban-fuzzer`, `CrashLab`) | random call sequences | "Can I break correctness?" | correctness, not scale; state size is incidental |
| TTL monitor (`soroban-sentinel`) | live ledger entries | "What is about to expire?" | the entries that exist, not the ones that cannot |
| Scaffold (`soroban-forge`) | templates and contracts | "How do I start?" | nothing about the contract's limits |
| **Capacity harness (Headroom)** | **a population of size N** | **"Where does this design break, and why?"** | — |

All of the above are good tools. They measure a point. Headroom measures a **curve, its knee, and the wall behind it**.

The distinction that matters most: a load test pushes traffic at a *network* and measures *throughput*. Headroom pushes **state** at a *contract* and measures a *ceiling*. One finds congestion; the other finds a design limit.

---

## The failure modes Headroom exists to find

These are the cliffs that pass tests and audits because nobody ever ran the contract at scale. Limits are Protocol 29 including the SLP-0007 network settings change live on Mainnet on 2026-10-22 (see [`docs/LIMITS-AND-METRICS.md`](docs/LIMITS-AND-METRICS.md) for sources).

| # | Failure mode | Hard limit | Typical design that trips it | What the developer sees |
|---|---|---|---|---|
| 1 | **Instance entry overflow** | 65,536 B per contract-data entry | A registry, factory or index that pushes IDs into one instance `Vec` | `storage write` fails; the contract can no longer be written to at all |
| 2 | **Write-entry cap** | 200 distinct entries written per transaction | A batch payout, airdrop or settlement loop | `ExceededLimit` on writes while CPU sits under 20% |
| 3 | **Footprint / disk-read cap** | 400 footprint entries; 200 read entries / 200,000 B | A "load all positions and compute" query | Rejected before execution; the caller cannot even estimate |
| 4 | **Event + return budget** | 16,384 B per transaction | Anything emitting one event per item in a loop | Fails at a batch size far below every other limit — the least intuitive wall |
| 5 | **Write-byte cap** | 132,096 B per tx; 297,000 B per ledger | Wide structs written in bulk | A transaction that passes the entry-count check and fails on size |
| 6 | **Hot-entry contention** | 1 write-set per entry per ledger | A global counter, a single config entry, one pool state | Every transaction passes; throughput collapses to one per ledger |

Failure mode 6 is the one no profiler can see, because each individual transaction is fine. It is a **property of the population**, which is exactly the axis Headroom measures.

---

## How it works

```text
scenario.toml  (contract + actors + workload + ladder + pinned limits)
      │
      ├─ validate ──────────────► schema check, unknown-key rejection, limit snapshot hash
      │
      ├─ generate(N) ───────────► deterministic actor population + setup steps
      │
      ├─ execute ───────────────► native Soroban host, captured network settings,
      │                           ledger snapshot, seeded, no wall clock
      │
      ├─ meter ─────────────────► per-operation: CPU, memory, footprint entries,
      │                           read bytes, write entries, write bytes, event bytes,
      │                           return bytes, entry sizes, TTL deltas
      │
      ├─ attribute ─────────────► closest limit, margin, and the exact entry/event
      │                           responsible
      │
      ├─ ladder ────────────────► geometric probe ─► first failure ─► binary refine
      │                           ─► N found, interval reported
      │
      └─ certificate ───────────► JSON + Markdown, hashed and re-derivable
```

Two properties make the result trustworthy rather than anecdotal:

- **Determinism.** Same scenario, same contract bytes, same limit snapshot ⇒ byte-identical certificate. No timing, no concurrency, no network.
- **Attribution.** A failure is never reported as "transaction failed". It is reported as *which* limit, *which* operation, *which* key, and *how far* past the line.

---

## Target interface

Five commands, deliberately few:

| Command | Purpose |
|---|---|
| `headroom limits` | Fetch live network settings and pin them to a versioned snapshot file |
| `headroom run <scenario>` | Execute a scenario, climb the ladder, write a report and certificate |
| `headroom report <certificate>` | Render a certificate as Markdown, JSON or a single-file HTML page |
| `headroom verify <certificate>` | Re-derive every hash and re-run the boundary rung; fail if anything moved |
| `headroom index` | Validate and rebuild the public Capacity Index from committed certificates |

A `headroom run` is one command, one config file, no Docker, no funded account, no network:

```bash
headroom run scenarios/escrow-open-escrows.toml \
  --limits limits/mainnet-2026-10-22.json \
  --out evidence/
```

Exit codes are contractual, because this belongs in CI:

| Code | Meaning |
|---|---|
| `0` | Scenario completed; N\* above the configured floor |
| `1` | N\* below the configured floor (a capacity regression) |
| `2` | Scenario could not execute (invalid config, missing artifact, limit snapshot mismatch) |
| `3` | Verification failed (certificate does not re-derive) |

---

## A worked example (illustrative arithmetic, not a measurement)

A registry contract stores every depositor in one instance `Vec<Address>`. Serialized, an `Address` in a vector is roughly 44–48 bytes including element framing; the entry itself carries a key, a TTL field and type metadata of roughly 100 bytes.

```text
65,536 B  entry cap
   −120 B  entry overhead (key + TTL + discriminant)
   ───────
 65,416 B  available for the vector
     ÷ 48 B per address
   ───────
   ≈1,362  addresses
```

So the arithmetic predicts a cliff in the low four figures. The **actual** number depends on the SDK's exact XDR encoding, the key length and the contract's own fields — which is precisely why Headroom measures instead of computing from a blog post. The ladder would report something like:

| N | Setup | Write at measure | Entry size | Binding limit | Headroom |
|---:|---|---|---:|---|---:|
| 800 | ok | ok | 38,522 B | entry size | 58.8% |
| 1,280 | ok | ok | 61,564 B | entry size | 6.1% |
| 1,360 | ok | **fail** | 65,712 B | **entry size** | −0.3% |
| 1,320 | ok | ok | 63,492 B | entry size | 3.1% |
| **1,318** | ok | **ok** | 63,398 B | entry size | **3.2%** |

Result: `N* = 1,318`, binding limit `contract_data_entry_size_bytes`, next-binding limit `write_bytes` (irrelevant — it would not bind until N ≈ 9,000). The fix is therefore not "batch the writes" but "stop putting the registry in instance storage".

That last sentence is the product. The numbers are the evidence for it.

---

## Capacity certificate

Every run produces a certificate: a small JSON artifact plus a human-readable report. It records enough that a third party can reproduce the result and enough that the result cannot be quietly edited later.

- schema version, tool version, scenario hash
- contract WASM SHA-256 and size
- network, protocol version, and the **hash of the pinned limit snapshot**
- every rung: N, per-operation measurements, entry sizes, outcome, error code if any
- N\*, the refinement interval, the binding limit, the runner-up limit
- marginal cost model (cost per additional unit) with its fit quality
- environment: host OS, Rust version, SDK version
- a hash chain: certificate hash, parent scenario hash, limit snapshot hash

`headroom verify` recomputes all hashes and re-runs the rungs around N\*. If the contract bytes, the limits, or the environment changed, verification fails loudly. This is the difference between a benchmark and a **claim**.

Details: [`docs/EVIDENCE.md`](docs/EVIDENCE.md).

---

## Capacity Index

A tool that only runs on the author's laptop is a demo. Headroom ships a second artifact: a public index of **real contracts measured at scale**, each with a committed certificate and a one-line reproduction command.

The first five entries are chosen to cover all six failure modes, not to flatter the tool:

| Contract | Design under test | Expected binding limit |
|---|---|---|
| `escrow` | one escrow per key, three-party flow | write entries (batch release only) |
| `registry` | all IDs in one instance `Vec` | instance entry size |
| `token` | balance per holder, transfer | footprint on batch transfer |
| `nft-collection` | token per owner, enumeration | read bytes on enumeration |
| `orderbook` | sorted book in persistent entries | hot-entry contention |

Some entries will report "no cliff found below N = 100,000". Those are as valuable as the cliffs, and they are published with the same rigour.

Details: [`docs/CAPACITY-INDEX.md`](docs/CAPACITY-INDEX.md).

---

## What Headroom deliberately does not do

Stated up front, because scope discipline is what makes the results credible:

- **Not a throughput or load tester.** It does not generate concurrent traffic, measure TPS, or touch a live network for measurement.
- **Not a fuzzer.** It does not explore random call sequences; it executes a declared population.
- **Not a fee estimator.** It reports resources; converting them to stroops is a separate, documented model.
- **Not an audit.** Passing at every rung says nothing about correctness or security.
- **Not a replacement for `simulateTransaction`.** It complements it: Headroom finds the design limit, simulation tells you what today's transaction will cost.

Full list: [`docs/LIMITATIONS.md`](docs/LIMITATIONS.md).

---

## Where it sits next to other tools

Headroom answers a question the existing Soroban tools do not: **how much state can this design hold before a protocol limit stops it?** It sits beside those tools rather than replacing them — no shared crate, no shared release cadence, and none of them would need Headroom in order to build.

| Tool | Its own question | The question Headroom adds | How they combine |
|---|---|---|---|
| **WeighIn** | did this change make one call more expensive? | how many users does the design have left? | the certificate names the binding limit, so a CI gate can guard the metric that actually binds instead of defaulting to CPU |
| **Soroban Sentinel** | what is expiring, and what will rent cost now? | what does rent cost at 10,000 users? | Sentinel reads a certificate's rent block; live entry sizes check Headroom's predictions |
| **Soroban Forge** | what do I start from? | what ceiling does this template already have? | a template can ship a scenario and a documented `N*` |
| **`soroban-meter`, `soroban-budget-assert`** | what does this function cost in `cargo test`? | what population does that cost permit? | run one, then write the assertion the other implies |

None of these handoffs is shipped: this repository is specification only, so the table describes the boundary Headroom intends to respect, not integrations that exist. Each one, with its interface and its refusal cases, is specified in [`docs/INTEGRATIONS.md`](docs/INTEGRATIONS.md); the full comparison against fifteen adjacent tools is in [`docs/LANDSCAPE.md`](docs/LANDSCAPE.md).

---

## Repository map (planned)

```text
headroom/
├── README.md                  this document
├── CONTRIBUTING.md            how to claim work, review standards, merge bar
├── docs/                      the full specification (see index below)
├── scenarios/                 example scenarios, one per fixture design
├── contracts/fixtures/        contracts written to contain a known cliff
├── crates/
│   ├── headroom-core/         deterministic host runner + measurement
│   ├── headroom-scenario/     schema, validation, population generator
│   └── headroom-cli/          limits | run | report | verify | index
├── limits/                    pinned network-settings snapshots, hashed
├── index/                     the public Capacity Index
└── evidence/                  committed reports and certificates
```

The layout, and the rule for what may live where, is specified in [`docs/REPOSITORY-LAYOUT.md`](docs/REPOSITORY-LAYOUT.md).

---

## Documentation index

**The idea**
- [`docs/PROBLEM.md`](docs/PROBLEM.md) — why capacity is the missing axis, with the four real incidents-style failure modes
- [`docs/LANDSCAPE.md`](docs/LANDSCAPE.md) — every adjacent tool, what it measures, and the exact gap this fills
- [`docs/DECISIONS.md`](docs/DECISIONS.md) — the decisions behind the design, including what was rejected and why

**The design**
- [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) — components, data flow, determinism model, failure attribution
- [`docs/SCENARIO-FORMAT.md`](docs/SCENARIO-FORMAT.md) — the versioned scenario schema, in full
- [`docs/LIMITS-AND-METRICS.md`](docs/LIMITS-AND-METRICS.md) — all eleven measured resources, all limits, all sources
- [`docs/REPOSITORY-LAYOUT.md`](docs/REPOSITORY-LAYOUT.md) — project structure and placement rules

**The proof**
- [`docs/EVIDENCE.md`](docs/EVIDENCE.md) — what counts as proof, the five artifacts, and the publication rules
- [`docs/CAPACITY-INDEX.md`](docs/CAPACITY-INDEX.md) — the public index, entry criteria, and review process

**The work**
- [`docs/ROADMAP.md`](docs/ROADMAP.md) — four milestones from empty repo to credible v0.1
- [`docs/INTEGRATIONS.md`](docs/INTEGRATIONS.md) — WeighIn, Soroban Sentinel, Soroban Forge, CI

**Reference**
- [`docs/LIMITATIONS.md`](docs/LIMITATIONS.md) — known limits of the method itself
- [`docs/FAQ.md`](docs/FAQ.md) — the questions a reviewer will ask
- [`docs/GLOSSARY.md`](docs/GLOSSARY.md) — N, N\*, cliff, binding limit, headroom ratio

---

## A note on numbers

Every resource limit in this repository carries its source and the date it was checked. Limits are network settings: validators change them. A benchmark that quotes a limit without a snapshot is quoting folklore.

Where a number is arithmetic rather than measurement, it says **illustrative**. Where a number is a measurement, the certificate that produced it is committed next to it. There is no third category.

---

## Contributing

The work is split into small, reviewable issues, each with explicit acceptance criteria and a verification command, so a change can land in a week without a meeting. Start with [`CONTRIBUTING.md`](CONTRIBUTING.md), then [`docs/ROADMAP.md`](docs/ROADMAP.md) for what is in scope.

## License

Dual-licensed under [MIT](LICENSE-MIT) or [Apache-2.0](LICENSE-APACHE), at your option — the Rust ecosystem convention, so the crates can be used under whichever fits.

Unless you state otherwise, any contribution you intentionally submit for inclusion in this project is dual-licensed on the same terms, with no additional conditions.

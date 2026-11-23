# Architecture

> Three crates, one measurement core, and a determinism contract that every other
> design decision serves.

---

## 1. Design constraints that drive the whole architecture

| Constraint | Consequence |
|---|---|
| A single run must finish in seconds, offline | in-process host execution, no network, no funded accounts |
| A certificate must be re-derivable by a stranger | everything that affects a number goes into the certificate |
| Attribution is the product, not the aggregate | measurement is per operation, per limit, per key — not one total |
| Thousands of executions per ladder | the runner must be reusable without re-validating or re-reading the scenario |
| Limits move | no constant is compiled in; the snapshot is an input |
| Contributors arrive with small PRs | strict crate boundaries; no PR should need to understand the whole system |

---

## 2. Component map

```text
                 ┌──────────────────────────────────────────────┐
                 │              headroom-cli                    │
                 │  limits | run | report | verify | index      │
                 └───────────────┬──────────────────────────────┘
                                 │ orchestration only
        ┌────────────────────────┼─────────────────────────┐
        ▼                        ▼                         ▼
┌────────────────┐     ┌────────────────────┐     ┌──────────────────┐
│headroom-scenario│     │  headroom-core     │     │  certificate I/O │
│ schema, validate│────▶│  runner, meter,    │────▶│  hash chain,     │
│ population gen  │     │  attribution,      │     │  render, verify  │
│ ladder control  │◀────│  ladder executor   │     └──────────────────┘
└────────────────┘     └─────────┬──────────┘
                                 │ embeds
                                 ▼
                       ┌──────────────────────┐
                       │ soroban host + ledger│
                       │ snapshot + cost cfg  │
                       └──────────────────────┘
```

Dependency rule: `headroom-cli` depends on both; neither core nor scenario depends on
the CLI. Rendering and hashing live in a small module shared by CLI and verify, so a
certificate can be checked without the orchestration layer.

---

## 3. `headroom-scenario`

**Responsibility:** turn a declarative file into a deterministic, executable plan.

| Module | Job |
|---|---|
| schema | deserialize and validate the versioned scenario; reject unknown keys |
| population | build actor sets and per-unit identity from N and the seed |
| steps | resolve each operation's parameters (literals, actor refs, derived ids) |
| ladder | produce the sequence of rungs from the growth and refinement config |
| limits | load a snapshot, verify its hash, expose values by name |
| errors | one error type with actionable messages, each naming the offending field |

Rules:

- **Unknown keys are errors**, never warnings. A silently ignored `stop_after` is a
  wrong certificate.
- **No I/O beyond reading the scenario and the limit snapshot.** No clock, no
  environment-dependent defaults, no network.
- **Identity is derived, never random.** A unit's ID is a function of
  `(scenario_seed, role, index)`. Two runs produce the same keys, so the same
  entries, so the same sizes.

---

## 4. `headroom-core`

**Responsibility:** execute a plan and measure it. The only crate that knows what a
host is.

### 4.1 Runner

- Builds a fresh, empty ledger for each rung, seeded from the snapshot's network
  passphrase and protocol version.
- Registers the contract from WASM bytes (or, for index entries, binds to a snapshot
  of live state).
- Executes the scenario's `setup` steps to construct the population, then each
  `measure` step in declared order.
- Discards the ledger at the end of the rung. Rungs never share state; a rung is not
  contaminated by the rung before it.

Cost settings come from the snapshot, not from the host's defaults. This is the
single most important implementation detail: a host run against default settings can
differ from the network's cost model, and a capacity number computed from the wrong
cost model is worse than no number.

### 4.2 Meter

Wraps each measured invocation and reads the host's own accounting:

- CPU and memory from the host's metering before and after.
- Footprint, reads, writes, events and return bytes from the recorded invocation
  result.
- Entry sizes by serializing each touched entry's key and value independently.
- Pre/post ledger diff, which is the ground truth for "what did this call actually
  write", as opposed to what it declared.

The declared footprint and the actual ledger diff are compared. A contract that
declares more than it touches wastes transaction size; one that touches more than it
declares fails. Both are reported.

### 4.3 Attribution engine

For a measured operation and a limit snapshot, produces a ranked list of
`(limit, measured, limit_value, utilisation, margin)` plus the per-key breakdown.
Pure function over measurements — no host access — which is why it is trivially
unit-testable and why contributors can work on it without touching the runner.

### 4.4 Ladder executor

A controller that asks the scenario for rungs, executes them, and decides what to do
next. Deliberately not a loop with a fixed step:

1. **Probe.** Start at `start`, multiply by `growth` each rung until the first
   failing rung or `max`.
2. **Refine.** Binary search between the last passing rung and the first failing rung
   until `resolution` (absolute N) or `max_steps` is reached.
3. **Confirm.** Re-run the boundary rung once, from scratch, to prove the result is
   not an artefact of ordering.
4. **Stop early** when configured, and say so in the certificate. "No cliff found
   below 100,000" is a valid, publishable result.

Each rung records: N, per-step measurements, outcome, error class, elapsed host time,
and the utilisation table. The controller is the only stateful component, and its
entire state is serialisable — which is what allows a run to be resumed and what
makes the certificate a complete description of what happened.

### 4.5 Error taxonomy

Failures are classified, because "transaction failed" is not a finding:

| Class | Meaning | Example |
|---|---|---|
| `limit_exceeded` | a metered limit was crossed | write entries 201 / 200 |
| `entry_size_exceeded` | a single entry exceeded 65,536 B | instance vector overflow |
| `footprint_violation` | touched entries outside the declared footprint | missing key in footprint |
| `contract_error` | the contract returned an error | `InsufficientBalance` |
| `host_error` | the host rejected the invocation | resource limit during execution |
| `setup_failure` | the population could not be built | auth failure while seeding |
| `scenario_invalid` | config could not produce a plan | unknown key, bad reference |

Only `limit_exceeded`, `entry_size_exceeded` and `footprint_violation` count as a
capacity cliff. A contract error at scale is a correctness bug and is reported as
such — attaching it to a capacity number would be dishonest.

---

## 5. Certificates

### 5.1 What goes in

| Field | Why it must be there |
|---|---|
| `schema_version` | the contract with consumers |
| `tool_version` + commit | reproducibility |
| `scenario` (canonical form + hash) | defines the question |
| `contract.wasm_sha256`, `size` | the bytes measured |
| `network`, `protocol_version` | the rules |
| `limits.snapshot_hash` | the limits used, verifiable independently |
| `environment` | OS, arch, Rust, SDK, host build — differences must be visible |
| `rungs[]` | every N, every measurement, every outcome |
| `result.n_star`, `interval`, `binding_limit`, `runner_up` | the answer |
| `model` | marginal cost fit and its residuals |
| `hash` | over the canonical form, with the parent hashes included |

### 5.2 Hash chain

```text
certificate.hash = H( canonical(certificate without hash) )
```

The canonical form is sorted-key JSON with fixed number formatting. `verify`
recomputes it and compares. A certificate whose numbers were edited by hand fails
verification even if the edit looks plausible.

The chain also links:

- the **scenario hash**, so a certificate cannot be paired with a different question;
- the **limit snapshot hash**, so it cannot be re-labelled with different rules;
- the **parent certificate hash** for index entries that supersede an earlier run.

### 5.3 Verification

`headroom verify <certificate>` performs three checks, each of which can fail
independently:

1. **Integrity** — recompute the hash chain.
2. **Reproduction** — re-run the boundary rung and its neighbours, compare measured
   values exactly (integers, no tolerance: the host is deterministic).
3. **Drift** — report whether the pinned snapshot still matches current network
   settings, and whether the WASM hash still matches the artifact.

`verify` is the only command allowed to touch the network, and only for check 3, and
only when explicitly asked (`--check-network`).

---

## 6. Determinism contract

| Source of nondeterminism | How it is removed |
|---|---|
| randomness | seeded PRNG; seed in the scenario and certificate |
| wall clock | absent; TTL math uses ledgers |
| hash-map iteration order | ordered maps everywhere |
| address allocation | derived from seed + role + index |
| host cost settings | taken from the snapshot, recorded |
| environment | recorded; a difference is reported, not hidden |
| concurrency | none; single-threaded ladder |

Consequence: two people on two machines, with the same crates and the same snapshot,
produce byte-identical measurements. When they don't, the certificate names the
difference. Determinism is not a nice property here — it is what lets a reviewer
trust a number they did not compute.

---

## 7. Performance budget

| Operation | Target |
|---|---|
| One rung at N = 1,000 with 5 operations | < 250 ms |
| Full ladder, 12 rungs, N up to 50,000 | < 20 s |
| `verify` of a 12-rung certificate | < 5 s |
| Peak memory at N = 50,000 | < 1 GiB |

The budget exists because a capacity tool that takes an hour will not be run before
a design decision. If a scenario exceeds it, the answer is smaller rungs and
extrapolation with a stated model — not a longer wait.

---

## 8. What is deliberately not built

- a plugin system (adapters are declarative first; native code only where a workload
  genuinely cannot be expressed)
- a database (certificates are files in git; the index is built from them)
- a web service (the report is a single self-contained HTML file)
- a scheduler (a ladder is fixed; adaptive search is a later, measurable feature)
- a chain client inside the runner (live access is a separate, optional module used
  only by `verify --check-network` and index snapshots)

Each omission is a place where a contributor could add complexity without adding
evidence. The project's credibility depends on the opposite.

---

## 9. Testing strategy

| Layer | What is tested | How |
|---|---|---|
| schema | every valid and invalid field, unknown keys, version handling | table-driven cases with golden errors |
| population | determinism across runs and machines | golden hashes of generated plans |
| ladder | boundary discovery against synthetic pass/fail curves | injectable executor |
| attribution | ranking, tie-breaking, zero-baseline behaviour | pure-function unit tests |
| certificate | canonicalisation stability, hash chain, tamper detection | golden certificates + deliberate edits |
| runner | known fixtures with hand-computed expectations | integration tests, `#[ignore]` offline suite |
| end-to-end | a fixture with a known cliff at a known N | the escrow and registry fixtures |

The end-to-end test is the one that matters: a fixture whose cliff is known from
first principles, asserted to within ±1. If that test drifts, the tool is wrong.

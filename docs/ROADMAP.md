# Roadmap

> Four weeks to a credible v0.1, then two releases that turn a tool into a public
> dataset. Every milestone ends in a committed artifact, not a demo.

---

## 0. Sequencing principle

The order is dictated by one rule: **each milestone must make the previous one
harder to fake.** Evidence before features, fixtures before adapters, index before
dashboards.

| Week | Milestone | Artifact at the end |
|---|---|---|
| 1 | Deterministic runner + one scenario | a registry scenario that runs to a known N, measured |
| 2 | Ladder + attribution + certificate | `N*` with a binding limit and a verifiable hash chain |
| 3 | Fixtures, adapters, CI | five scenarios, one of them a negative result, all green in CI |
| 4 | Index seed + docs + release | v0.1 tagged, five index entries, public reproduction |

---

## Week 1 — Make one run real

**Goal:** execute a population and measure it, with no ladder and no reporting.

- Workspace skeleton, pinned toolchain, CI (fmt, clippy, tests, doc links).
- Limit snapshot loader with hash verification, plus `limits/mainnet-p29-slp0007.json`
  generated from the live network settings.
- Scenario schema v1 for a single rung: contract, population, setup, one measured
  step. No ladder yet.
- Runner: fresh ledger per run, contract deployment from WASM, seeded actors.
- Meter: the eleven resources, entry sizes, ledger diff.

**Definition of done:** a registry fixture with 500 registrants executes, and the
measured instance-entry size is within 1% of the arithmetic prediction. That single
number is the first moment the project is real.

**Risk to watch:** cost settings. If the host is configured from defaults rather than
the snapshot, everything downstream is subtly wrong. This is checked first, before
any feature work.

---

## Week 2 — Find the boundary and name the cause

**Goal:** `N*`, attributed, with a certificate that a stranger can verify.

- Ladder: geometric probe, binary refine, confirm, interval reporting.
- Attribution: ranked utilisations, binding limit, runner-up, per-key breakdown.
- Certificate: canonical form, hash chain, `verify` with reproduction of boundary
  rungs.
- Report renderer: Markdown with the utilisation table and the failing operation.
- CI policy: `fail_if.n_star_below` and exit codes wired to the shell.

**Definition of done:** `headroom run` on the registry scenario reports
`N* = 1,3xx`, binding limit `contract_data_entry_size_bytes`, and `headroom verify`
passes on a different machine. The report's first line is the reproduction command.

**Risk to watch:** off-by-one at the boundary. The confirm step exists for this;
without it, "N\*" is whatever the last loop iteration happened to be.

---

## Week 3 — Broaden the evidence

**Goal:** prove the tool responds to design, not just to size.

- Five fixtures: registry (instance and persistent variants), escrow, token, payout
  batch. Each with a README stating its predicted cliff.
- Five scenarios, one deliberately expecting **no** cliff.
- Adapters: `registry`, `escrow`, `token`, `raw`.
- Event-budget attribution (failure mode 4) and write-byte accounting (failure mode 5).
- Contention analysis as a warning, not a failure.
- `capacity.yml`: re-runs every scenario on push and diffs certificates.

**Definition of done:** the token scenario reports "no cliff below 100,000" and the
payout scenario fails at ~100 recipients with CPU under 20% — both published, both
reproduced in CI.

**Risk to watch:** adapter creep. Three adapters is enough for v0.1. Every extra
adapter is a fixture that is not yet trusted.

---

## Week 4 — Publish and release

**Goal:** an outsider can go from clone to a verified certificate in ten minutes.

- `index/` with the first five entries and a generated `index.json`.
- `headroom report --html` as a single self-contained file.
- Documentation complete: every doc in `docs/` matches the code, and
  `scripts/check-links` passes.
- README leads with one real number and one reproduction command.
- v0.1.0 tagged; crates published as `headroom-scenario` and `headroom-core` if a
  stable enough API exists (otherwise binary only).

**Definition of done:** a fresh clone, `cargo build --release`, one command, one
certificate, `verify` green. Recorded as a screen capture and as the README's first
paragraph.

---

## v0.2 — Usable by other people

| Feature | Why it earns its place |
|---|---|
| `headroom snapshot` | measure a deployed contract against its real state, not a synthetic one |
| adapter: `nft`, `orderbook` | the two most-requested Wave designs |
| HTML report with an N-vs-utilisation chart | the one visual that makes a cliff obvious |
| GitHub Action | a team should add capacity gating in one line |
| resume/checkpoint for long ladders | N = 200,000 runs should be interruptible |
| model fitting and extrapolation with residuals | "no cliff below 100,000" needs a stated extrapolation, not a shrug |

Not in v0.2: a web dashboard, a hosted service, a plugin system, mainnet load
generation. None of these make a number more true.

---

## v1.0 — The dataset is the product

| Feature | Why |
|---|---|
| 20+ index entries across eight contract families | comparability becomes meaningful |
| re-run on every protocol upgrade, with a published diff | the index stays alive |
| per-design guidance generated from the index | "for registries, expect ~1.3k in instance storage" |
| second native measurement path cross-checked against `simulateTransaction` on testnet | proves the offline method agrees with the chain |
| stable certificate schema v1 with a published compatibility statement | certificates outlive the tool |

---

## What would make us stop

Honest kill criteria, because a project without them drifts:

- If the measured cliff never matches first-principles arithmetic for the fixtures,
  the method is wrong and should be abandoned rather than patched.
- If the SDK ships an equivalent capacity macro, Headroom's scope collapses to the
  index and the certificates — and the README should say so within a day.
- If native host measurement cannot be kept within a few percent of testnet
  `simulateTransaction` across the fixtures, the numbers are not network numbers and
  the project must say that in every report header, or stop.

---

## Effort, honestly

| Area | Rough share of v0.1 effort |
|---|---|
| runner + meter (host, snapshots, cost settings) | 35% |
| scenario schema + population generation | 20% |
| ladder + attribution | 15% |
| certificate + verify | 10% |
| fixtures (contracts written to break) | 10% |
| CLI, reports, docs, CI | 10% |

The hard part is not the ladder; it is making the host run reproducible and making
the fixtures fail for the reason they are supposed to fail. Everything else is
plumbing.

---

## What a Wave contributor can pick up in week one

- a limit snapshot loader with hash verification (Medium)
- the utilisation ranking function with tie-breaking rules (Medium)
- the Markdown report renderer with a golden-file test (Trivial–Medium)
- one declarative adapter (Medium)
- docs, examples, and the reproduction walkthrough (Trivial)

None of these require the host to be working, which is deliberate: it keeps the
project contributable while the hardest part is being built.

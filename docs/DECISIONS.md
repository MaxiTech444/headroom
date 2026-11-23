# Decisions

> A log of the choices that shaped the project, in the order they were made,
> including the ones that were rejected. Each entry names the alternative and why
> it lost. New decisions are appended; old ones are never rewritten.

---

## D1 — Measure capacity, not cost

**Context.** The ecosystem already has cost profilers, regression gates and fuzzers.
Adding a fourth cost tool would be redundant.

**Decision.** Headroom's primary output is a **boundary**: the largest population at
which a scenario still succeeds, plus the limit that fails first.

**Alternatives rejected.**
- *A better profiler* — marginal improvement over `soroban-meter` and
  `soroban-budget-assert`; no new claim.
- *A cost visualization tool* — a dashboard over numbers nobody acts on.
- *A general benchmark runner* — no thesis, and benchmarks without a question are
  noise.

**Consequence.** The tool is only interesting if `N*` is discoverable and attributable.
If attribution proves impossible, the project has no product.

---

## D2 — Deterministic and offline

**Context.** Network-based measurement is more "real" but unreproducible: state
changes, fees move, and a reviewer cannot re-run yesterday's result.

**Decision.** Execute against an in-process Soroban host with captured network cost
settings, seeded population, fixed ledger sequence, no clock, no RPC.

**Alternatives rejected.**
- *Testnet measurement* — needs funded accounts, is slow, and is not reproducible.
- *A hybrid that measures on-chain and caches* — combines the worst properties:
  slow, expensive, and still not re-derivable.

**Consequence.** Every claim is "under the pinned protocol model". Cross-checking
against testnet simulation is a planned validation, not the measurement path.

---

## D3 — Attribution is the product

**Context.** A single number ("max 1,318 users") invites the wrong fix. A developer
told "you hit a limit" will optimise compute, because compute is what they can see.

**Decision.** Every failure names the limit, the operation, the key, the exact
overflow, and the margin to the runner-up limit.

**Alternative rejected.** *Report utilisation percentages and let the reader infer the
cause* — measurement without interpretation is what the ecosystem already has.

**Consequence.** The measurement layer must capture per-key sizes and declared-vs-actual
footprints, not just aggregate costs. That is more work than a profiler, and it is the
whole value.

---

## D4 — Certificates, hashed, with a verify command

**Context.** Benchmarks in READMEs are unverifiable. The temptation to tweak a number
after a code change is real and universal.

**Decision.** Every run emits a canonical JSON certificate with a hash chain covering
scenario, WASM bytes, limit snapshot, environment and measurements. `verify`
recomputes and re-runs.

**Alternatives rejected.**
- *A report only* — pretty, uncheckable.
- *Signed certificates with a project key* — implies an authority the project does not
  have, and adds key management for no additional guarantee (anyone can build a tool).

**Consequence.** The certificate format is a public contract from v0.1, which is why
its schema version bumps on any semantic change.

---

## D5 — Pinned limit snapshots, never constants

**Context.** Limits are network settings. SLP-0007 changed several on 2026-10-22.

**Decision.** No limit is compiled in. A snapshot file is an input; its hash is in the
certificate; a mismatch is an error.

**Alternative rejected.** *A `limits.rs` with well-known values* — simple, and wrong
within one protocol upgrade.

**Consequence.** More setup friction for a first run, and the ability to say
"measured under Protocol 29 + SLP-0007" as a verifiable statement.

---

## D6 — Fixtures are contracts that fail on purpose

**Context.** Mocks prove a harness runs; they do not prove a number is right.

**Decision.** Every fixture is a small real contract with a documented, independently
predictable cliff. The first one's cliff is derived from XDR arithmetic before the
tool runs.

**Alternative rejected.** *Synthetic scenarios against existing example contracts* —
fast to build, and impossible to validate: nobody knows the true answer.

**Consequence.** Roughly a fifth of the v0.1 effort goes into contracts that will
never be used by anyone except this project. That is the cost of a checkable number.

---

## D7 — Three crates, strict boundaries

**Context.** A single crate is fastest to start and becomes unnavigable at 5k lines,
especially for Wave contributors with one week.

**Decision.** `headroom-scenario` (no host), `headroom-core` (no file parsing), and
`headroom-cli` (all I/O). A PR should touch one crate.

**Alternatives rejected.**
- *One crate* — simpler today, hostile to contributors.
- *Six crates* — architecture in search of a problem; every boundary is a versioning
  decision.

**Consequence.** Some duplication of small types across the boundary, which is
cheaper than a shared "common" crate.

---

## D8 — Separate org for the Wave

**Context.** Drips Wave applies budgets per repo and per org.

**Decision.** Create a dedicated org with this single repository.

**Alternatives rejected.**
- *Add as a fifth repo to the existing org* — competes with siblings for a shared
  point budget, and reads as a portfolio rather than a project.

**Consequence.** A new org, a new approval application, and a cleaner claim.

---

## D9 — Name: Headroom

**Context.** The concept is the distance between current state and the first limit.

**Alternatives rejected.**
- *Freeboard* / *Plimsoll* — a Wave repo already uses the nautical load-line metaphor.
- *Capacity* — unsearchable and already overloaded.
- *Ceiling* — sounds like a governance tool.

**Decision.** "Headroom" — the word engineers already use for this exact margin, with
no collision in the Stellar Wave list.

---

## D10 — No web UI in v0.1

**Context.** A chart of utilisation versus N would be genuinely useful for
communication.

**Decision.** Markdown first. A single-file HTML report in v0.2. No hosted service ever.

**Alternatives rejected.**
- *A Next.js dashboard* — a week of work that adds a hosting dependency to a claim
  that should be checkable offline.
- *JSON only* — accurate and unreadable by the people who make the decision.

**Consequence.** The first impression is a table, not a graph. Acceptable: the table
names the binding limit, which a graph does not.

---

## D11 — Contention as a warning, not a failure

**Context.** Hot-entry serialisation is a real capacity ceiling, but no transaction
fails.

**Decision.** Derive it from write-set overlap, report it as a warning class with its
assumption stated, and never fail a run on it.

**Alternative rejected.** *Model throughput and report TPS* — would require assumptions
about fee markets and account sequencing that this project cannot justify.

**Consequence.** The tool's most interesting insight is its least quantitative output,
and it is labelled accordingly.

---

## D12 — Publish failures, including "no cliff"

**Context.** Every incentive pushes a new tool toward impressive results.

**Decision.** The evidence set includes a negative result by design (a token with no
cliff below 100,000), and CI re-runs every published certificate.

**Alternative rejected.** *Publish only the dramatic cliffs* — the fastest way to become
a tool that nobody believes.

**Consequence.** Some reports say "nothing interesting happened", and that is a
feature: it is what makes the interesting ones credible.

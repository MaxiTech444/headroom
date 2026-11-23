# Landscape

> Every tool this project could be confused with, what it actually measures, and
> the precise gap Headroom occupies. Re-checked against the Stellar Wave repo list
> (965 repos, checked 2026-11-23) and the wider ecosystem.

The honest summary: **the ecosystem is crowded at "how expensive is one call" and
empty at "how much state can this design hold".** Everything below is a good tool.
None of them varies the population.

---

## 1. The one-axis test

Give each tool a single question: *does it change the amount of state and measure
what happens?*

| Repository | What it measures | Axis | Varies N? | Collision risk |
|---|---|---|---|---|
| `soroban-meter/soroban-meter` | CPU/memory per function from `cargo test`, regression baselines | cost | no | low |
| `Tollcraft/soroban-budget-assert` | cost assertions as macros + testnet simulation report | cost | no | low |
| `@57blocks/stellar-resource-usage` | resource usage per function | cost | no | low |
| `soroban-cost-linter` (Wave) | cost anti-patterns in source | cost | no | low |
| `WeighInNG/WeighIn` (this author) | BASE vs HEAD resource deltas, CI gate | cost, over revisions | no | none — composes |
| `soroban-fuzzer` | correctness under random calls | correctness | incidental | low |
| `CrashLab` (Wave) | crash/failure exploration | correctness | incidental | low |
| `soroban-sentinel` (this author) | live TTL remaining, rent forecast | time | no | none — composes |
| `soroban-forge` (this author) | scaffolds, contract foundations, CLI, testnet proof | developer setup | no | none — composes |
| `Forge-soroban/Forge-soroban` | interactive lab, blueprints, gas estimator UI | cost | no | low |
| `CheesecakeLabs/soroban-dapps` load-test | network throughput under parallel accounts | throughput | traffic, not state | low |
| `StellarRoute` gas-benchmark docs | per-function CPU table across hops | cost | no | low |
| `soroban-testkit`, `soroscope` | harness/reporting utilities | cost | no | low |
| `soroban-fork` / `stellar snapshot create` | fork and snapshot chain state | environment | n/a | none |
| **Headroom (this project)** | **largest N that succeeds, binding limit, marginal cost** | **capacity** | **yes** | — |

---

## 2. Nearest neighbours, in detail

### 2.1 `soroban-meter`

Resource profiling inside the test suite and CI. Per-function CPU and memory,
regression detection against a baseline, JSON output.

**Why it is not the same:** its unit is a function call in a test whose state the
author wrote by hand. It answers "how much does `deposit()` cost?" and does it
well. It does not answer "at what number of deposits does it stop working", and it
cannot, because that requires generating and executing a population.

**How they compose:** `soroban-meter` is the per-call regression gate; Headroom is
the population ceiling. A team can run both in one CI job and read them as a pair.

### 2.2 `soroban-budget-assert`

Attribute macros that fail a test when a measured cost crosses a pinned value,
plus a CLI that simulates every contract in a workspace on testnet and reports the
resource amounts.

**Why it is not the same:** pinned thresholds are constants chosen by the author.
It catches *drift* against a number, not the *location of the wall*. Its own scope
note is honest that rent, transaction size, footprint counts and inclusion fees are
outside what it measures — three of which are exactly what binds a batch operation.

**Relationship:** a team could use `soroban-budget-assert` for the per-call budget
and Headroom's certificate to choose the budget in the first place.

### 2.3 `WeighIn` (same author)

Measures a configured function in BASE and HEAD, pairs logical benchmarks across
runtime addresses, enforces percentage thresholds and absolute caps, fails the
Action on regression.

**Why it is not the same, and why that matters for credibility:** WeighIn's own
README is explicit that it measures a benchmark case, not a population. Two
revisions of the same contract, both at a handful of entries, tell you about
*change*, not about *capacity*. A 0% regression on a contract that dies at 900
users is still a dead contract.

**Relationship:** WeighIn answers "did this PR regress?", Headroom answers "will
this design survive growth?". They share the native measurement approach and the
evidence culture, and the two repos should cross-link rather than merge. Keeping
them separate keeps each one's claim narrow and checkable.

### 2.4 `soroban-sentinel` (same author)

Scans live contracts for TTL remaining, warns before archival, forecasts rent,
bundles TTL extensions and restores.

**Why it is not the same:** it observes entries that **exist**. The entries that
kill a contract are the ones it can never create. Sentinel is the post-deployment
half of the same story; Headroom is the pre-deployment half.

**Relationship:** Headroom's rent model projects annual cost at N; Sentinel can
consume that projection to alert before a fleet's rent outruns its budget.

### 2.5 `soroban-forge` (same author)

Scaffolding plus reusable contract foundations — escrow, vesting, multi-sig, DAO,
subscriptions, marketplace royalties — a CLI, TypeScript bindings, and a live
testnet-verified escrow with conservation properties.

**Why it is not the same:** Forge answers "how do I start, and what is a correct
implementation?" It has no capacity axis, no ladder, no certificate. Reading its
docs confirms the scope: build, test, deploy, verify, inspect — not measure at
scale.

**Relationship, and the strongest one available:** Forge's templates are the ideal
*fixtures* for Headroom. Each Forge contract has a natural population — escrows per
instance, vesting schedules per beneficiary, proposals per DAO, subscriptions per
provider. Shipping `headroom.toml` with each template, and a capacity certificate
in the template's docs, turns two projects into one story: **Forge gives you a
foundation with a known ceiling.** That is a materially stronger claim than either
repo makes alone, and it costs no shared code.

### 2.6 `Forge-soroban` (different project, similar name)

An interactive lab: 40+ blueprints, a Solidity↔Soroban diff viewer, and a gas cost
estimator in a web app.

**Why it is not the same:** the estimator is per-call, in a browser, for education.
It is a learning tool. Headroom is a measurement instrument with committed
evidence. The name overlap is unfortunate; Headroom is unambiguous in the README
about what it is, and there is no functional overlap.

### 2.7 `CheesecakeLabs/soroban-dapps` load test

A shell-script load test for a liquidity pool: parallel accounts, N requests,
testnet, timings.

**Why it is not the same:** this is the closest thing in the ecosystem to "test at
scale" and it is still a different question. It measures a *network's* behaviour
under concurrent traffic with funded accounts on a live chain. It does not tell you
at what population the contract's own storage design fails, and it cannot run in CI
in seconds, offline, deterministically.

Worth stating plainly: load testing is not wrong, it is orthogonal. Headroom's
result is an input to a load test (you know which N is interesting), not a
replacement.

### 2.8 `StellarRoute` gas benchmarks, `FlowStar` bench suite, `Uzima` limit docs

Per-function CPU tables, a few batch sizes, documented limits.

**Why they are not the same:** these are *project documentation* — a team measuring
its own contract once and writing it down. That is the manual version of one rung
of Headroom's ladder, and it is exactly the evidence that Headroom automates,
standardises and makes comparable. Their existence is demand validation: teams are
already doing this by hand, badly, in prose.

### 2.9 `soroban-fork`, `stellar snapshot create`

Provide forked or snapshotted chain state to execute against.

**Why they are not the same:** they supply an environment. Headroom supplies a
question, a ladder, an attribution and a certificate. Headroom *may* use a snapshot
as an input for replaying a live contract; it does not reimplement forking.

---

## 3. The gap, stated precisely

Every tool above reports a **scalar at a point**: cost, TTL, pass/fail, throughput.

Headroom reports a **boundary and its cause**:

- a population size (`N*`),
- the identity of the limit that fails there (one of eleven),
- the margin to the runner-up limit,
- and the marginal cost per additional unit.

No tool in the Stellar Wave list produces that tuple. That is the whole claim. It
is narrow, it is checkable, and it is not a feature any existing tool could add in a
day — the population generator, the ladder, the attribution and the certificate are
each a body of work.

---

## 4. What would make this project redundant

Stated so the claim can be falsified:

- The Soroban SDK ships a `capacity!` test macro that builds a population and
  reports the binding limit.
- `stellar contract` gains a `scale` subcommand with a scenario file and a
  certificate format.
- A Wave repo publishes a generalised, scenario-driven capacity harness with a
  public index — not a one-off load script, not a per-project benchmark doc.

If any of those appear, Headroom's value drops to its index and its evidence, and
the README should say so. As of 2026-11-23, none exist.

---

## 5. Positioning in one line

> `soroban-meter` tells you what a call costs. `WeighIn` tells you if it got worse.
> `soroban-sentinel` tells you what is expiring. **Headroom tells you how many users
> you have left, and what breaks when they arrive.**

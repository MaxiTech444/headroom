# Evidence

> What this project will accept as proof, the five artifacts it will publish, and
> the rules that keep a certificate honest. Written before any measurement exists,
> so the standard cannot be softened after the fact.

---

## 1. Why this document exists first

Capacity claims are easy to fake. A screenshot of a number, a table in a README, a
"benchmarked at 10,000 users" line in a grant application — all of it is unverifiable
by default, and all of it is believed anyway.

So Headroom defines proof as something a stranger can **recompute**. If a claim
cannot be recomputed from committed inputs, it does not belong in this repository,
however impressive it looks.

---

## 2. The standard

A capacity claim is proven when all five of these exist:

| # | Component | Why it is required |
|---|---|---|
| 1 | **The question**, in a scenario file | a number without a scenario is not an answer |
| 2 | **The bytes**, by SHA-256 and size | the measurement is of specific code, not of a repository |
| 3 | **The rules**, as a pinned limit snapshot with a hash | a limit quoted from memory is folklore |
| 4 | **The measurements**, per rung, per operation | aggregates hide the binding limit |
| 5 | **A reproduction command** that runs offline in seconds | "trust me" is not evidence |

Missing any one of these turns a certificate into a screenshot.

---

## 3. The five artifacts

### A1 — A contract with a cliff you can predict in advance

**What:** `contracts/fixtures/registry` stores every registrant in one instance
`Vec`. From the encoding, the cliff is computable to within a few units.

**The claim:** the ladder's measured `N*` matches the arithmetic prediction, and the
failing write is rejected for entry size — not for CPU, not for events.

**Why it convinces:** the number is falsifiable *before* the run. If the tool
disagrees with arithmetic, the tool is wrong. A benchmark that only produces numbers
nobody predicted proves nothing.

**What is published:** the scenario, the certificate, the report, a four-line
calculation, and the last passing / first failing rungs.

### A2 — The same contract, redesigned, with the ceiling moved

**What:** the same workload against a variant that stores one persistent entry per
registrant.

**The claim:** `N*` moves from four figures to the point where a different limit
binds, and the certificate names that new limit.

**Why it convinces:** it shows the tool responds to design, not just to size. It is
also the first real finding of the project: *for this design, instance storage is the
wrong choice, and here is the number that proves it.*

### A3 — A limit that binds before the obvious one: events

**What:** a loop that emits one event per processed item.

**The claim:** the run fails at a batch size where CPU utilisation is under 10% and
the footprint is a fraction of its cap, because events and the return value share
16,384 bytes.

**Why it convinces:** it is the counter-intuitive result, and counter-intuitive
results are the ones that justify a new tool. The report shows all eleven
utilisations side by side, so the reader can see the wall that was not being watched.

### A4 — A published failure: no cliff found

**What:** the token fixture, transferring between holders at increasing N.

**The claim:** *no cliff below N = 100,000.* Footprint is constant; cost per transfer
is flat; only rent grows.

**Why it convinces:** a tool that finds a cliff everywhere is measuring its own
assumptions. Publishing the negative result is the cheapest available proof of
honesty, and it also documents what "scales" actually means for a token.

### A5 — Reproduction by a stranger

**What:** a CI job that starts from a clean checkout, builds the fixtures, re-runs
every committed scenario, and **diffs the recomputed certificates against the
committed ones**.

**The claim:** the project's own numbers are checked on every push, on a machine the
author does not control.

**Why it convinces:** it removes the author entirely. If the CI job is green, the
numbers in `evidence/` are true today, not true once.

---

## 4. Rules for a publishable certificate

1. **Deterministic.** No seed left to chance, no clock, no network.
2. **Self-describing.** Contract hash, limit snapshot hash, scenario hash, tool
   version, environment. A reader should not need the repository to understand an
   artifact.
3. **Boundary-explicit.** `N*` is reported with its interval. "1,318" means "passes
   at 1,318, fails at 1,319" only when the interval says so.
4. **Attributed.** A failure names a limit. An unattributed failure is a bug report,
   not a result.
5. **Reproducible.** The report's first section is the exact command.
6. **Honest about overrides.** Any run with hypothetical limits is marked
   `hypothetical` and is never indexed or quoted as a capacity figure.
7. **Dated.** Protocol version and check date are in the header, because limits move.

---

## 5. What will not be published

- Numbers from a machine whose environment is not recorded.
- Numbers whose certificate fails `verify`.
- Benchmarks of contracts the project has no permission to describe, with identifying
  details removed only if the contract's author asks.
- Any claim of the form "supports X users" without the scenario that defines a user.
- Comparisons between designs measured under different limit snapshots.

The last rule is the one most benchmark suites break. Under SLP-0007, per-ledger caps
doubled and fee rates halved; a comparison across that boundary is meaningless.

---

## 6. The public proof page

`docs/CAPACITY-INDEX.md` describes the index; this section describes what a reader
sees when they arrive.

For each indexed design, one page. The values below are the README's illustrative
registry example and show the format only; no run has produced them.

```text
Design:      registry — every registrant in one instance Vec
Measured:    <date>, Protocol 29 + SLP-0007
Contract:    <contract id>, WASM sha256 <hash>
Scenario:    registry-instance-overflow.toml (hash <hash>)
N*:          1,362   (interval 1,362–1,363)
Binding:     contract_data_entry_size_bytes  (65,498 / 65,536 = 99.9%)
Runner-up:   tx_write_bytes (65,498 / 132,096 = 49.6%)
Marginal:    +48 bytes of state per registrant
Reproduce:   headroom verify evidence/registry-instance-overflow.cert.json
```

That block is the unit of the whole project. If the format is boring and complete,
the project works.

---

## 7. Milestones of proof

| Stage | What is proven | What is not |
|---|---|---|
| M1: harness runs | the runner executes a scenario and records measurements | that the numbers mean anything |
| M2: A1 | measured cliff equals predicted cliff | that other designs behave |
| M3: A2 + A3 | the tool responds to design and attributes correctly | that it generalises |
| M4: A4 + A5 | negative results, CI reproduction | ecosystem relevance |
| v0.2: index (5 entries) | five real designs, five certificates | statistical claims |
| v1.0: index (20+ entries) | comparable designs across the ecosystem | anything about mainnet load |

Each stage has a public artifact. There is no stage where the claim outruns the
evidence.

---

## 8. On failing loudly

If the harness disagrees with the arithmetic in A1, the response is not to adjust
the arithmetic. It is to publish the discrepancy, because a disagreement between
first-principles encoding math and host measurement is either a bug in the tool, a
misunderstanding of the XDR layout, or a genuine surprise in the protocol — and all
three are worth knowing.

The same applies to the Capacity Index: an entry that contradicts its author's
published claim is published with both numbers and an invitation to respond. A
measurement instrument that only reports what people want to hear is a marketing
instrument.

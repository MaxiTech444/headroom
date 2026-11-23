# Scenario Format v1

> The scenario file is the whole input to a capacity run: what contract, which
> population, which operations, how far to climb, and under which limits. This
> document specifies it completely, so that a scenario written against v1 keeps
> meaning the same thing forever.

Status: **specified, not yet implemented.** The schema version is `1` and is
declared in every file. A breaking change increments it; additive optional fields
do not.

---

## 1. Format choice

TOML. Reasons: comments survive review, it is the format the Stellar CLI already
uses for configuration (`stellar.toml`), and hand-edited capacity scenarios are more
common than machine-generated ones. JSON is accepted as an alternative input
(`.json` extension) with identical semantics; the canonical form used for hashing is
always JSON.

---

## 2. Complete example

```toml
schema = 1
name = "escrow-open-escrows"
summary = "Concurrent open escrows held by one contract instance"

[contract]
adapter = "escrow"
wasm    = "contracts/fixtures/escrow/target/wasm32v1-none/release/escrow.wasm"
# `id` is required only for snapshot replay of a deployed contract
# id    = "CC227UDF6WBLRTOKKVRIJN7BGSBK67ZGV6IDARJ2AMATGSQ7UZNBZHSB"

[population]
unit      = "escrow"
seed      = "escrow-v1"
max_units = 20000

[actors]
buyer   = { kind = "contract", per_unit = true }
seller  = { kind = "account",  per_unit = true }
arbiter = { kind = "account",  per_unit = false }

[workload.setup]
steps = [
  { op = "create",  id = "open",   args = { escrow = "$unit.id", buyer = "$buyer", seller = "$seller", arbiter = "$arbiter", amount = 100 } },
  { op = "deposit", id = "funded", args = { escrow = "$unit.id", from = "$buyer" } },
]

[workload.measure]
steps = [
  { op = "release", args = { escrow = "$unit.id", to = "$seller" } },
]

[ladder]
start      = 100
growth     = 1.6
max        = 20000
resolution = 1
max_steps  = 24
confirm    = true

[limits]
snapshot = "limits/mainnet-p29-slp0007.json"

[report]
markdown    = "evidence/escrow-open-escrows.md"
certificate = "evidence/escrow-open-escrows.cert.json"

[fail_if]
n_star_below = 1000
binding_limit_in = ["contract_data_entry_size_bytes", "tx_write_entries"]
```

---

## 3. Top-level fields

| Field | Type | Required | Meaning |
|---|---|---|---|
| `schema` | integer | yes | must be `1`; any other value is rejected with an upgrade hint |
| `name` | string | yes | kebab-case, unique within a repository; used in the certificate |
| `summary` | string | no | one sentence shown in the report header and the index |
| `contract` | table | yes | what is measured |
| `population` | table | yes | what grows |
| `actors` | table | no | named actor sets referenced by steps |
| `workload` | table | yes | setup and measured steps |
| `ladder` | table | yes | how N is searched |
| `limits` | table | yes | which pinned snapshot applies |
| `report` | table | no | where artifacts go; defaults are documented |
| `fail_if` | table | no | CI policy; absent means report-only |

Unknown keys **anywhere** are errors. This is not strictness for its own sake: a
typo in `growth` that silently defaults produces a plausible-looking wrong answer,
which is the worst possible failure for a measurement tool.

---

## 4. `[contract]`

| Field | Type | Required | Meaning |
|---|---|---|---|
| `adapter` | string | yes | workload family: `escrow`, `registry`, `token`, `nft`, `orderbook`, `vault`, or `raw` |
| `wasm` | path | yes* | WASM artifact to deploy at each rung |
| `id` | StrKey | no | existing contract, for snapshot replay instead of fresh deployment |
| `snapshot` | path | no | ledger snapshot used to initialize the rung |
| `entry_points` | table | no | overrides for adapter method names when a contract uses different names |

\* `wasm` or `id` — exactly one.

The `raw` adapter exists so that a contract with no adapter can still be measured by
declaring operations as `(method, args)` pairs. It is deliberately lower-level and
has no generated population semantics beyond `$unit`.

Adapters are **declarative**: an adapter maps generic intents (`create`, `fund`,
`release`, `mint`, `list`) onto a contract's real methods and argument shapes. Only a
workload that genuinely cannot be expressed declaratively justifies native code, and
that code lives with the adapter's data, not in `headroom-core`.

---

## 5. `[population]`

| Field | Type | Default | Meaning |
|---|---|---|---|
| `unit` | string | `"unit"` | the noun in reports ("1,318 escrows") |
| `seed` | string | required | input to every derived identity; changing it changes the run |
| `max_units` | integer | 100000 | hard ceiling; the ladder never exceeds it |
| `preallocate` | boolean | `true` | build the full population before measuring; `false` measures during growth, which is a different question and must be declared |

`preallocate = false` is included because "can I create the 10,000th item when 9,999
exist" and "how expensive is creating items as the population grows" are different
questions. The certificate states which was asked.

---

## 6. `[actors]`

Actors are the identities a workload acts as.

| Field | Type | Meaning |
|---|---|---|
| `kind` | `account` \| `contract` | Stellar account or deployed contract used as an address |
| `per_unit` | boolean | `true` creates one actor per unit; `false` creates one shared actor |
| `count` | integer | for `per_unit = false`, how many (default 1) |

`per_unit = true` actors are derived from `(seed, role, index)`, so actor *i* is the
same address in every run and at every rung. This is what makes entry keys stable and
therefore comparable across rungs.

Referencing an actor in a step uses `$<role>` (shared) or `$<role>[i]` (indexed,
where `i` is the unit index). A reference to an undefined role, or an out-of-range
index, is rejected at validation time, before any execution.

---

## 7. `[workload]`

Two ordered lists of steps.

- **`setup`** runs once per rung, before any measurement, to build the population of
  size N. Setup failures are classified `setup_failure` and never counted as a cliff.
- **`measure`** runs after setup, in order, each measured independently.

A step has:

| Field | Type | Required | Meaning |
|---|---|---|---|
| `op` | string | yes | adapter-defined intent (`create`, `deposit`, `release`, `mint`, `list`, `query`, `transfer`) |
| `args` | table | no | intent parameters; values may be literals or references |
| `id` | string | no | label used in reports; defaults to `op` plus an index |
| `expect` | table | no | expected outcome: `success` (default) or a named error |
| `batch` | integer | no | repeat the step N times inside one invocation, for batch-shaped workloads |

`batch` deserves emphasis. A batch is not "run 200 times"; it is **one transaction
that performs N operations**, which is the shape that hits the write-entry and event
caps. Declaring `batch = 200` with `per_unit` actors is how a payout loop is modelled
faithfully.

### References

| Form | Resolves to |
|---|---|
| `$unit.id` | deterministic ID of the current unit |
| `$unit.index` | the current unit's index within the rung |
| `$<role>` | a shared actor |
| `$<role>[i]` | an indexed actor, `i` relative to the unit |
| `literal:...` | forced literal, avoiding ambiguity with a role named like a value |

There is no expression language. If a workload needs arithmetic, it needs an adapter
field, not a calculator in a config file. This keeps scenarios auditable by a
reviewer who has never seen the tool.

---

## 8. `[ladder]`

| Field | Type | Default | Meaning |
|---|---|---|---|
| `start` | integer | 10 | first N |
| `growth` | float | 1.6 | multiplier per probe rung; must be > 1.0 |
| `max` | integer | from `population.max_units` | last N attempted |
| `resolution` | integer | 1 | stop refining when the interval is this small |
| `max_steps` | integer | 40 | hard cap on rungs, so a pathological scenario terminates |
| `confirm` | boolean | true | re-run the boundary rung once from scratch |
| `stop_after_failures` | integer | 1 | consecutive failing rungs before refinement begins |

The resulting certificate always states the interval, even when `resolution = 1`,
because a boundary discovered by bisection is only known to be within the last
untested gap.

If `start` already fails, the run reports `N* = 0` and names the binding limit at the
smallest population tested. That is a legitimate and useful result — a contract that
cannot hold ten items has no capacity to discuss.

---

## 9. `[limits]`

| Field | Type | Meaning |
|---|---|---|
| `snapshot` | path | pinned network-settings JSON under `limits/` |
| `overrides` | table | explicit per-limit overrides, for experiments against hypothetical settings |

Overrides exist because protocol changes should be testable *before* they ship: an
interval SLP can be modelled by overriding a value, and the resulting certificate is
labelled `hypothetical` in its header. A certificate with overrides is never
published in the Capacity Index.

---

## 10. `[report]`

| Field | Default | Meaning |
|---|---|---|
| `markdown` | `evidence/<name>.md` | human report |
| `certificate` | `evidence/<name>.cert.json` | machine artifact |
| `html` | none | optional single-file HTML with charts |
| `title` | from `summary` | report heading |

Paths are relative to the scenario file. A report is written atomically (temp file,
then rename) so an interrupted run never leaves a half-written certificate that
looks valid.

---

## 11. `[fail_if]` — CI policy

| Field | Type | Meaning |
|---|---|---|
| `n_star_below` | integer | exit 1 when the measured N\* is lower |
| `binding_limit_in` | list | exit 1 when the binding limit is not among these |
| `headroom_at_below` | float | exit 1 when the utilisation at N\* exceeds this fraction |

The third is the subtle one. "It works up to 1,200" is a different risk depending on
whether 1,200 sits at 99% of a limit or 40%. A team that wants 2× growth room can
require `headroom_at_below = 0.5`.

Policy is evaluated after the run, from the certificate, and is itself recorded in
the certificate. A CI failure therefore always names the rule that failed.

---

## 12. Validation rules

Anything that could silently change the answer is an error:

1. Unknown keys at any level.
2. `growth <= 1.0`, `resolution < 1`, `max < start`, `max_steps < 2`.
3. A reference to an undefined actor, role, or unit index.
4. A step whose `op` is not implemented by the adapter.
5. Both or neither of `wasm` and `id`.
6. A limit snapshot with a hash that does not match its content.
7. Two scenarios in one index with the same `name`.
8. A `batch` value above `max_units`.
9. `preallocate = false` combined with `batch` (batch implies a pre-built
   population).
10. A `measure` section with no steps.

Each error names the file, the field path, the offending value, and the fix.

---

## 13. Versioning

| Change | Version |
|---|---|
| new optional field with a safe default | 1 (additive) |
| new adapter | 1 (additive) |
| changed meaning of an existing field | 2 |
| changed limit semantics | 2 |
| changed certificate canonicalisation | 2 |

Old scenarios are not migrated automatically. A `schema = 1` file run by a v2 tool
either runs with v1 semantics and says so in the certificate, or refuses with a
migration guide. It never guesses.

---

## 14. Authoring guidance

- **One question per scenario.** "How many escrows" and "how fast does release get"
  are two scenarios. A combined ladder answers neither cleanly.
- **Name the unit in the population.** Reports read better as "1,318 escrows" than
  "N = 1,318".
- **Start below the expected cliff.** The first rung is also a control: it proves the
  scenario can succeed at all.
- **Prefer `batch` over repeated steps** when modelling a loop, and say so in the
  summary — the binding limit is different.
- **Set `fail_if` only after the first run.** A gate you have not seen pass is a gate
  you cannot trust.

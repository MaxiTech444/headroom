# Integrations

> Headroom is one instrument in a set. This document specifies exactly how it hands
> off to the tools around it, so the ecosystem gains a chain of evidence rather than
> another silo.
>
> None of these handoffs exists yet. The repository is specification only; what
> follows is the boundary Headroom intends to respect, written down so it can be
> checked against the implementation once there is one.

---

## 1. The evidence chain

```text
design                build                  per-call                per-population         live
──────                ─────                  ────────                ──────────────         ────
Soroban Forge   ───▶  CI build + hash  ───▶  WeighIn /        ───▶  Headroom        ───▶  Sentinel
template              (provenance)           soroban-meter         capacity               TTL/rent
                                             (regression)          certificate            forecast
```

Each step answers a different question and produces a different artifact. None
duplicates another's claim.

---

## 2. Soroban Forge (`Meet-hybrid/soroban-forge`)

**Relationship:** fixtures and distribution, not shared code.

Forge ships reusable contract foundations — escrow, vesting, multi-signature, DAO,
subscriptions, marketplace royalties — each with a natural population. That makes
each one a ready-made capacity case.

What Headroom adds to Forge:

| Forge template | Headroom scenario | Question it answers |
|---|---|---|
| `escrow` | `escrow-open-escrows.toml` | how many escrows can one instance hold? |
| `vesting` | `vesting-beneficiaries.toml` | does claim cost grow with schedule count? |
| `multi-sig-wallet` | `multisig-pending-txs.toml` | where does the approval loop break? |
| `dao-governance` | `dao-votes-per-proposal.toml` | do votes hit the event budget first? |
| `subscription-payments` | `subscriptions-charge-batch.toml` | does billing batch beyond 100? |
| `marketplace-royalties` | `marketplace-royalty-splits.toml` | do split payouts hit write caps? |

Concrete integration, three small pieces:

1. A `headroom.toml` in each Forge template, with `start`, `growth` and `max` chosen
   for that design.
2. A line in the template README: *"Known ceiling: `<N*>` `<unit>` (certificate)."*,
   filled in from a committed certificate, never by hand.
   One number, linked. That is a stronger template than one that says "production
   ready".
3. A CI job in Forge that runs the template's scenario and fails if `N*` drops below
   the documented value — a capacity regression gate alongside the existing cost one.

**Why it is worth doing for Forge:** it converts "we have six contracts" into "we
have six contracts with measured ceilings". For adoption, that is the difference
between a library and a foundation.

**Boundary:** no shared crate. Forge must not depend on Headroom to build, and
Headroom must not vendor Forge contracts. The coupling is two files and one CI job.

---

## 3. WeighIn (`WeighInNG/WeighIn`)

**Relationship:** same measurement philosophy, different axis. Complementary, and
deliberately separate repositories.

| | WeighIn | Headroom |
|---|---|---|
| Compares | BASE vs HEAD | N vs N+1 |
| Unit | a benchmark case | a population |
| Fails on | a percentage regression | a limit crossing |
| Runs | in a PR | before a design decision, and in CI |
| Artifact | diff report + PR comment | capacity certificate |

The handoff runs both ways:

- **Headroom → WeighIn:** a certificate's `binding_limit` tells a team which metric
  deserves a strict WeighIn threshold. Today, policies are often set to
  `strict_zero_tolerance` on CPU because CPU is the only number people watch. The
  certificate says the real constraint is write entries or event bytes — so the
  policy should guard *that*.
- **WeighIn → Headroom:** when WeighIn flags a regression, Headroom answers the
  follow-up question: did that change move the ceiling?

A shared CI job is one YAML file with two steps. No shared dependency is needed, and
neither tool should acquire one: WeighIn is npm/TypeScript, Headroom is Rust, and the
overlap is a JSON file.

**Reuse, not duplication:** Headroom borrows WeighIn's hard-won lessons about native
host measurement — captured cost settings, deterministic ledger construction, logical
identity independent of runtime addresses, and the discipline of publishing failed
controls alongside successes. That is technique, not code.

---

## 4. Soroban Sentinel (`ekwe7/soroban-sentinel` equivalent)

**Relationship:** before and after deployment.

| Sentinel observes | Headroom predicts |
|---|---|
| entries that exist and their TTL | entries a design can create |
| rent for current state | rent at N, modelled |
| archived keys needing restore | keys that will never be creatable at N |

Two concrete handoffs:

1. **Rent forecast input.** Sentinel forecasts rent for a fleet. Headroom's
   `rent_per_user_per_year` at a projected population is the missing input: *"at
   10,000 users this contract costs X per year to keep alive"*. Sentinel can consume a
   certificate's rent block directly — it is a small JSON object, not a service call.
2. **Entry-size validation.** Sentinel measures live entry sizes. Headroom predicts
   them. A persistent gap between the two is a modelling error worth investigating,
   and publishing that comparison is a cheap correctness check for both projects.

**Boundary:** Sentinel reads certificates. It does not run scenarios, and Headroom
does not talk to RPC for measurement.

---

## 5. `soroban-meter` and `soroban-budget-assert`

**Relationship:** overlapping measurement, different assertions.

- `soroban-meter` is excellent at per-function cost in `cargo test` and regression
  baselines. Headroom's fixtures can be measured by it too, and the two numbers
  should agree at low N — a useful cross-check that both tools are reading the host
  correctly.
- `soroban-budget-assert` lets a test assert a cost ceiling. Headroom tells you what
  the ceiling should be. That is a natural workflow: run `headroom run`, read the
  binding limit, then write the assertion.

Neither needs to change. The integration is documentation: a short section in each
tool's docs saying which question the other answers.

---

## 6. CI

Three jobs, in increasing cost:

| Job | Trigger | Duration | Artifact |
|---|---|---|---|
| `capacity` (fixture scenarios) | every push | ~30 s | certificates, diffed against committed |
| `capacity-index` (indexed contracts) | nightly, and on protocol upgrade | minutes | updated entries, migration note |
| `capacity-gate` (a user's own scenario) | a PR that changes a contract | ~30 s | pass/fail against `fail_if` |

The gate is the one that changes behaviour. A worked example for a consuming
repository:

```yaml
- uses: MaxiTech444/headroom@<pinned-sha>
  with:
    scenario: scenarios/escrow.toml
    limits: limits/mainnet-p29-slp0007.json
    fail-if-n-star-below: "1200"
    report-path: capacity-report.md
```

Rules for the Action, matching WeighIn's precedent:

- pinned by commit SHA, never a floating tag
- the report uploads with `if: always()` so a failed gate stays inspectable
- the Action never fetches network settings at run time; the snapshot is committed
- a missing or hash-mismatched snapshot fails the job rather than falling back

---

## 7. What is deliberately not integrated

- **A hosted dashboard.** Certificates are files; a dashboard centralises a claim
  that should be checkable locally.
- **An RPC dependency in the measurement path.** It would make runs non-reproducible.
- **A shared crate with any other project.** Every shared crate is a shared release
  cadence and a shared failure mode. JSON files are enough.
- **Wallet or key handling.** The tool needs no keys, and that property is worth more
  than any integration it would enable.

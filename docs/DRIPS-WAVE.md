# Drips Wave Submission Plan

> How this repository is set up to be accepted into the Stellar Wave Program, how
> the work is split into claimable issues, and what a reviewer will check.

---

## 1. Submission mechanics (checked 2026-11-23)

| Item | Detail |
|---|---|
| Program | Stellar Wave, a recurring monthly contribution cycle |
| Scale | ~965 approved repos, 610 orgs, 384k+ issues in the program |
| Reward unit | Points per merged issue: 100 (Trivial), 150 (Medium), 200 (High) |
| Pool distribution | quadratic scaling since Wave 3; points convert to a share of the Wave pool |
| Multiplier | featured repos show as "2x/4x Points" |
| Budget | per-repo and per-org point budgets exist (Wave 4+); an org's approved repos share a cap |
| Requirement | the repo must be applied to the program and **approved** before issues can be added |

Re-check the current rules on `drips.network` before submitting; they have changed
every Wave so far.

---

## 2. The org decision

**Recommendation: create a new GitHub org for Headroom rather than adding a fifth
repository to the existing one.**

Reasons:

1. **Per-org point budgets.** The org that already carries several Wave repos shares
   one budget across them. A new repo under a saturated org competes with its
   siblings for the same points.
2. **The 2x/4x featured multiplier is per repo application.** A tool repo with a
   narrow, checkable claim is a plausible featured candidate in a way that "one more
   repo in a portfolio" is not.
3. **Reviewer signal.** An org whose single purpose is capacity measurement reads as
   a project. An org with twelve repos reads as a portfolio, and reviewers apply
   different standards.
4. **Independence of evidence.** The index is meant to be a neutral public dataset.
   That is easier to argue from an org that is not also publishing contracts,
   frontends and SDKs.

This is not a criticism of the existing repos — it is the same argument as the one
for keeping Headroom's claim narrow: **one org, one claim.**

Suggested names: `headroom-labs`, `stellar-headroom`, `capacity-labs`.

---

## 3. Why this repo should be accepted

The acceptance question is not "is this useful" but "can strangers contribute
reviewable work here for a month". The answers, in the reviewer's order:

| Criterion | How this repo satisfies it |
|---|---|
| Real gap | no tool varies the population; profilers all hold N constant |
| Verifiable output | certificates with hash chains, reproduced in CI on every push |
| Scoped work | 24 issues, each touching one crate plus tests plus a doc |
| Onboarding | docs written before code; every doc names its audience |
| Maintenance | a named reviewer, a merge standard, an issue template |
| Stellar-native | measures Soroban host behaviour against live protocol limits |
| Longevity | the index is a dataset, not a demo; it has value after the code stops changing |

The last row is the one most Wave submissions cannot offer. Tools are easy to
abandon; datasets accumulate.

---

## 4. Issue plan

24 issues across the three tiers, ordered by dependency. Each has: description,
scope, files, acceptance criteria, verification command, and the doc it must update.

### High (200 points) — 5 issues

| # | Title | Why it is High |
|---|---|---|
| H1 | Deterministic host runner with snapshot-driven cost settings | the foundation; subtle, affects every number |
| H2 | N-ladder with probe, refine, confirm, and interval reporting | the core algorithm and its edge cases |
| H3 | Limit attribution engine with per-key breakdown and contention model | the product differentiator |
| H4 | Certificate canonicalisation, hash chain, and `verify` | the credibility mechanism |
| H5 | Instance-storage overflow detector with boundary confirmation | the headline finding, needs exactness |

### Medium (150 points) — 11 issues

| # | Title |
|---|---|
| M1 | Scenario schema v1 with unknown-key rejection and version gate |
| M2 | Deterministic population generator (seeded actors, stable keys) |
| M3 | Limit snapshot loader with content hash and drift reporting |
| M4 | Adapter: `registry` (instance and persistent variants) |
| M5 | Adapter: `escrow` (three-party flow, release and refund) |
| M6 | Adapter: `token` (balances, transfers, holder growth) |
| M7 | Event-budget attribution (events + return bytes) |
| M8 | Write-byte and write-entry accounting for batch workloads |
| M9 | Rent model: per-entry, per-population, per-year, labelled modelled |
| M10 | Markdown report renderer with utilisation table and failing-operation detail |
| M11 | `capacity.yml` CI: run all scenarios, diff certificates, upload artifacts |

### Trivial (100 points) — 8 issues

| # | Title |
|---|---|
| T1 | `headroom limits --check` output formatting and diff table |
| T2 | Exit-code documentation and a shell test for each code |
| T3 | Registry fixture README with the arithmetic prediction |
| T4 | Reproduction walkthrough: clone to verified certificate in ten minutes |
| T5 | Error-message audit: every validation error names file, field, value, fix |
| T6 | Golden-file tests for the Markdown renderer |
| T7 | Docs link checker in CI |
| T8 | CONTRIBUTING: issue template, merge standard, review checklist |

### Deliberately excluded from the first Wave

- Adapters for NFT, orderbook, DAO (they need the first three adapters to be trusted)
- HTML report (Markdown is enough to prove the idea)
- Snapshot capture of live contracts (needs a stable certificate format first)
- Any web UI

Excluded work is listed in the issue tracker as `future`, not left implicit, so a
contributor does not "helpfully" build a dashboard in week one.

---

## 5. What a merged PR must contain

The review standard is the product's credibility, so it is stated in every issue:

1. **Tests that fail before the change** and pass after.
2. **A doc change** in the same PR if behaviour or a number changed.
3. **No new dependency** without a sentence justifying it against the alternative of
   writing twenty lines.
4. **No silent defaults.** If a value can be omitted, the default is documented and
   tested.
5. **No measurement claim** in a doc without a certificate beside it.
6. **`cargo fmt`, `cargo clippy -D warnings`**, and the fixture scenarios green.

Reviewers should reject, not fix. A first-time contributor whose PR is rewritten by
a maintainer learns nothing and comes back with the same mistake; a contributor who
gets a list of specific problems either fixes them or leaves, and both outcomes are
fine.

---

## 6. Timeline inside a Wave

| Day | Activity |
|---|---|
| before | repo applied and approved; 24 issues created with labels and acceptance criteria |
| day 1 | `good first issue` set (T1–T8) claimed within hours; maintainer answers questions in the issue, not in DMs |
| day 2–4 | Medium claims; maintainer reviews within 24h, always with a verification command run locally |
| day 5–6 | High issues land; integration conflicts resolved by the maintainer, not by contributors |
| day 7 | final merges, CI green, a short "what changed in this Wave" note in the repo |
| after | unmerged but sound work is kept as a draft PR with a note on what remains — never silently closed |

---

## 7. Point economics, stated plainly

| Tier | Points | Realistic count in one Wave |
|---|---:|---:|
| High | 200 | 2–3 |
| Medium | 150 | 5–8 |
| Trivial | 100 | 4–8 |

With quadratic scaling, spreading work across many small merged PRs is worth less
than a few substantial ones — which matches the project's actual needs. The plan
concentrates High issues on the four components that determine whether the numbers
are true, and leaves breadth (adapters, docs, CI) to Medium and Trivial.

A reviewer can see the intent: **the points follow the evidence, not the other way
round.**

---

## 8. What the maintainer owes contributors

- Every issue is executable without asking a question. If it is not, the issue is
  the bug.
- Every review says *why*, and names the command that proves it.
- No issue is closed as "not planned" after work has started without an explanation
  and a replacement path.
- Credit in the release notes for every merged contributor, including docs-only work.
- A public statement of what the project will do with its index, so contributors know
  what they are building toward.

---

## 9. Risks to the submission

| Risk | Mitigation |
|---|---|
| "Another profiling tool" rejection | README opens with the gap table and the one-axis distinction; the index is the differentiator |
| Per-org budget cap | separate org, single-repo scope |
| Too much docs, no code at submission | the first four issues are code; docs exist so contributors do not need the maintainer |
| Scope creep from contributors | explicit `future` list, and issue templates that require acceptance criteria |
| Numbers that do not reproduce | CI diffs certificates on every push from day one |
| Being confused with `soroban-forge` / `Forge-soroban` | distinct org, distinct name, and an explicit non-affiliation note in the README's landscape section |

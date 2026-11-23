# Contributing

> How work is scoped, claimed, reviewed and merged. Written so a contributor can
> land a change in a week without a meeting.

---

## 1. Before you start

Read, in this order:

1. [`README.md`](README.md) — what the project claims
2. [`docs/PROBLEM.md`](docs/PROBLEM.md) — why the claim matters
3. [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) — where your change goes
4. [`docs/DRIPS-WAVE.md`](docs/DRIPS-WAVE.md) — how the work is split

If an issue is ambiguous, **the issue is the bug**. Comment on it, and the maintainer
will make it executable. Do not guess at scope and then discover it in review.

---

## 2. Picking up an issue

Every issue states:

- **Description** — the problem, in two or three sentences
- **Scope** — what is in, and explicitly what is out
- **Files** — the crate and module it touches
- **Acceptance criteria** — bullet points, each checkable
- **Verification** — the exact command that proves it
- **Docs** — which document must change in the same PR

Claim it by commenting. One issue at a time until your first merge; after that, two at
most. Parallel claims from one person turn a review queue into a merge conflict.

### Labels

| Label | Meaning |
|---|---|
| `complexity: trivial` | one file, one behaviour, hours |
| `complexity: medium` | one module, new tests, a doc update |
| `complexity: high` | a subsystem, design judgment, a certificate impact |
| `good first issue` | deliberately self-contained; no host knowledge needed |
| `needs-decision` | do not start until the maintainer resolves the question |
| `future` | listed to prevent scope creep; not claimable this Wave |

---

## 3. The merge standard

A PR is merged when all six are true:

1. **Tests fail before the change and pass after.** A test that passes on the old code
   is not a test of the change.
2. **The verification command from the issue is run and pasted** in the PR
   description, with its output.
3. **Docs change with behaviour.** A number, a default, or an error message is a
   documented interface.
4. **No new dependency** without one sentence justifying it against writing twenty
   lines.
5. **No silent defaults.** Every optional value has a documented, tested default.
6. **No measurement claim** in a document without a certificate next to it.

Formatting and lints are not a review topic: `cargo fmt --check` and
`cargo clippy -- -D warnings` are CI-enforced, and a red job is fixed by the
contributor, not waived.

---

## 4. What reviewers do and do not do

Reviewers:

- run the verification command themselves, on a clean checkout
- ask "what would make this number wrong?" and check that case
- request a smaller PR if one arrives with two unrelated changes
- say *why* in every comment, and name the standard being applied

Reviewers do **not**:

- rewrite a contributor's work silently — a maintainer patch is a last resort, and it
  is disclosed in the PR
- merge a PR whose numbers they have not reproduced
- approve a change that adds a claim without evidence, however small the code

---

## 5. Commit messages

`<area>: <what changed, in the imperative>`

Examples:

```text
ladder: report the refinement interval even at resolution 1
meter: size entries from the ledger diff, not the declared footprint
docs: state the 64 KiB instance limit with its protocol source
```

A commit that changes a measurement or a limit must say so in the message body, because
those diffs are the ones a reviewer will need to find again in six months.

---

## 6. Environment

| Tool | Version | Why pinned |
|---|---|---|
| Rust | pinned in `rust-toolchain.toml` | host behaviour and cost accounting can shift between releases |
| target | `wasm32v1-none` | the target the network accepts |
| Stellar CLI | pinned in CI | fixture deployment parity |
| OS | Linux x64 in CI | the supported measurement platform for v0.1 |

A result produced on an unlisted platform is welcome as a report, but it is not
indexed until it reproduces on the CI platform.

---

## 7. Reporting a wrong number

This is the most valuable contribution the project can receive, and it has its own
process:

1. Open an issue titled `wrong number: <design> at N=<n>`.
2. Include the certificate, the scenario, and the expected value with its derivation.
3. The maintainer reproduces within 48 hours.
4. If the tool is wrong: the certificate is marked `retracted` (not deleted), the bug
   is fixed, and the case becomes a regression test.
5. If the certificate is right: the expectation is wrong, and the issue is closed with
   the derivation, which is then added to the docs.

Retractions are published, not hidden. A tool that has never retracted a result is
either perfect or unmeasured.

---

## 8. Documentation contributions

Docs are not second-class. A documentation PR is merged on the same standard as code:
checkable claims, links that resolve, and no number without a source.

Useful doc contributions:

- a reproduction walkthrough verified on your own machine
- a limit update with the protocol version and date
- a rewrite of a section you had to read twice
- a new FAQ entry from a question you actually asked

---

## 9. Code of conduct

Be specific, be kind, argue with evidence. Disagreement about a number is welcome;
disagreement about a person is not. Review comments address the change, never the
contributor.

---

## 10. Licensing

Contributions are accepted under MIT OR Apache-2.0, matching the repository. By
opening a PR you confirm you have the right to license the work.

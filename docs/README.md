# Documentation

> The full specification, in reading order. Everything here is meant to be read
> before the code exists, and kept true after.

## Start here

| Document | Read it if you want to know |
|---|---|
| [../README.md](../README.md) | what the project is, in one page |
| [PROBLEM.md](PROBLEM.md) | why capacity is the missing measurement |
| [LANDSCAPE.md](LANDSCAPE.md) | how it differs from every adjacent tool |
| [EVIDENCE.md](EVIDENCE.md) | what will count as proof |

## Design

| Document | Contents |
|---|---|
| [ARCHITECTURE.md](ARCHITECTURE.md) | crates, runner, meter, attribution, ladder, certificates, determinism |
| [SCENARIO-FORMAT.md](SCENARIO-FORMAT.md) | the complete v1 scenario schema, field by field |
| [LIMITS-AND-METRICS.md](LIMITS-AND-METRICS.md) | all limits, all eleven metrics, sources and dates |
| [REPOSITORY-LAYOUT.md](REPOSITORY-LAYOUT.md) | structure, placement rules, crate boundaries |

## Output

| Document | Contents |
|---|---|
| [CAPACITY-INDEX.md](CAPACITY-INDEX.md) | the public dataset: entry criteria, review, maintenance |
| [EVIDENCE.md](EVIDENCE.md) | the five proof artifacts and the publication rules |

## Work

| Document | Contents |
|---|---|
| [ROADMAP.md](ROADMAP.md) | four weeks to v0.1, then v0.2 and v1.0, with kill criteria |
| [INTEGRATIONS.md](INTEGRATIONS.md) | WeighIn, Soroban Sentinel, Soroban Forge, CI |

## Reference

| Document | Contents |
|---|---|
| [LIMITATIONS.md](LIMITATIONS.md) | what the method cannot tell you |
| [FAQ.md](FAQ.md) | the questions a reviewer will ask |
| [DECISIONS.md](DECISIONS.md) | twelve decisions, with the alternatives that lost |
| [GLOSSARY.md](GLOSSARY.md) | the vocabulary, defined once |

---

## Conventions used throughout

- **Measured** means read from the host during execution.
- **Modelled** means derived from measurements with a stated formula.
- **Illustrative** means arithmetic or reasoning, not a result.
- Every limit carries its protocol version and check date.
- Every claim links to the artifact that proves it.
- A document that cannot be checked is deleted, not softened.

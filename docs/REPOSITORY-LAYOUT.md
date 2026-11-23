# Repository Layout

> What lives where, why the boundaries are drawn there, and the rules a pull
> request must respect. This is the structure the project will have when the first
> line of implementation lands.

---

## 1. Tree

```text
headroom/
├── README.md
├── CONTRIBUTING.md
├── LICENSE                      MIT OR Apache-2.0
├── Cargo.toml                   workspace: three crates, pinned toolchain
├── rust-toolchain.toml           pinned, because measurements depend on it
├── .github/
│   └── workflows/
│       ├── ci.yml               format, clippy, tests, docs links
│       ├── capacity.yml         runs the fixture scenarios, uploads certificates
│       └── index.yml            validates the Capacity Index on every PR
├── docs/
│   ├── README.md                documentation index
│   ├── PROBLEM.md
│   ├── LANDSCAPE.md
│   ├── ARCHITECTURE.md
│   ├── SCENARIO-FORMAT.md
│   ├── LIMITS-AND-METRICS.md
│   ├── REPOSITORY-LAYOUT.md     (this file)
│   ├── EVIDENCE.md
│   ├── CAPACITY-INDEX.md
│   ├── ROADMAP.md
│   ├── DRIPS-WAVE.md
│   ├── INTEGRATIONS.md
│   ├── LIMITATIONS.md
│   ├── FAQ.md
│   ├── DECISIONS.md
│   └── GLOSSARY.md
├── crates/
│   ├── headroom-core/
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── runner/          host setup, ledger seeding, invocation
│   │       ├── meter/           resource capture, entry sizing, ledger diff
│   │       ├── attribute/       limit ranking, per-key attribution, contention
│   │       ├── ladder/          probe, refine, confirm, resume
│   │       ├── certificate/     canonical form, hashing, verification
│   │       └── rent/            modelled per-entry and per-population rent
│   ├── headroom-scenario/
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── schema/          serde types, unknown-key rejection, version gate
│   │       ├── population/      actor and unit generation from seed
│   │       ├── steps/           reference resolution, batch expansion
│   │       ├── limits/          snapshot loading and hash verification
│   │       └── adapters/        declarative adapter data (escrow, registry, ...)
│   └── headroom-cli/
│       └── src/
│           ├── main.rs
│           ├── cmd_limits.rs
│           ├── cmd_run.rs
│           ├── cmd_report.rs
│           ├── cmd_verify.rs
│           └── cmd_index.rs
├── contracts/
│   └── fixtures/
│       ├── registry/            deliberately overflows instance storage
│       ├── escrow/              three-party flow, one entry per escrow
│       ├── token/               balance per holder
│       ├── nft-collection/      enumeration and ownership
│       ├── orderbook/           sorted book in persistent entries
│       └── payout/              batch loop that breaks the write cap
├── scenarios/
│   ├── registry-instance-overflow.toml
│   ├── escrow-open-escrows.toml
│   ├── token-holder-balances.toml
│   ├── nft-enumeration.toml
│   ├── orderbook-depth.toml
│   └── payout-batch-size.toml
├── limits/
│   ├── mainnet-p29-slp0007.json
│   ├── testnet-p29.json
│   └── README.md                how to produce a new snapshot
├── index/
│   ├── index.json               the public list of measured designs
│   ├── README.md                entry criteria and how to add one
│   └── entries/                 one folder per indexed contract
└── evidence/
    ├── registry-instance-overflow.md
    ├── registry-instance-overflow.cert.json
    └── ...
```

---

## 2. Placement rules

| If you are adding... | It goes in... | It must not... |
|---|---|---|
| a new limit or resource | `headroom-scenario/src/limits` + `docs/LIMITS-AND-METRICS.md` | be hardcoded in core |
| a new workload family | a new declarative adapter | import `headroom-core` |
| a report format | `headroom-cli` or a renderer beside `certificate` | change the certificate |
| a fixture contract | `contracts/fixtures/<name>/` | contain production code |
| a measurement of a real contract | `index/entries/<contract>/` | be published without a certificate |
| a design note | `docs/` | live only in a PR description |

### Why fixtures are contracts, not mocks

Every fixture is a real Soroban contract with a known, documented cliff. A mock
would prove the harness works; a contract proves the *number* is right, because the
cliff can be checked independently with `stellar contract invoke` on testnet. The
fixtures are also small enough to read in one sitting, which makes them usable as
teaching material.

### Why `index/` is separate from `evidence/`

`evidence/` is the output of this repository's own scenarios: the project proving
itself. `index/` is measurements of **other people's contracts**, contributed by
their authors or by reviewers, each with a certificate and a reproduction command.
Keeping them apart means the project's self-proof cannot be confused with its
ecosystem claims.

---

## 3. Crate boundaries

```text
headroom-cli  ──▶ headroom-core
      │                ▲
      └──▶ headroom-scenario ──┘
```

- `headroom-scenario` knows about schemas, seeds, and file formats. It must not
  import the host or the runner; a validation test should never execute code.
- `headroom-core` knows about the host, measurement, and certificates. It must not
  parse TOML or read environment variables; a measurement function should be a pure
  function of a plan.
- `headroom-cli` is the only place that touches the filesystem broadly, parses
  arguments, prints, and sets exit codes.

The rule of thumb: **a contributor's PR should touch one crate plus its tests plus
one document.** If a change needs all three, the boundary is wrong and the PR
description should say so before the code is written.

---

## 4. Naming

| Thing | Convention | Example |
|---|---|---|
| scenarios | `<design>-<question>.toml` | `escrow-open-escrows.toml` |
| certificates | same base name, `.cert.json` | `escrow-open-escrows.cert.json` |
| reports | same base name, `.md` | `escrow-open-escrows.md` |
| fixtures | the design noun | `registry`, `orderbook` |
| limits | `<network>-p<protocol>-<slp>.json` | `mainnet-p29-slp0007.json` |
| index entries | `<org>-<contract>` | `acme-escrow` |

The pattern is boring on purpose: a reader should be able to find the certificate for
a report without opening anything.

---

## 5. What must never be committed

- private keys, seeds, or funded account credentials (the tool needs none)
- certificates produced with `[limits].overrides` (they model hypothetical settings)
- certificates whose `verify` fails
- generated WASM artifacts larger than the repository's policy allows (build in CI;
  commit the SHA-256, not the bytes)
- reports without their certificate

The last two are the ones that erode trust quietly: a report whose certificate is
missing cannot be checked, and a certificate without the bytes it measured cannot be
reproduced.

---

## 6. Growth path

The layout is deliberately flat today. Three later additions have reserved homes:

| Later | Where it will live | Why it is not there now |
|---|---|---|
| live-contract snapshot capture | `headroom-cli snapshot` + `core/runner` | needs the snapshot format to stabilise first |
| HTML report with charts | `headroom-cli/report` | Markdown is enough to prove the idea |
| GitHub Action wrapper | `.github/actions/headroom` | the CLI with a pinned version is sufficient |

Nothing else is planned. A capacity tool that grows a plugin ecosystem before it has
a correct number has optimised the wrong thing.

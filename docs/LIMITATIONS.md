# Known Limitations

> What the method cannot tell you, stated before anyone asks. A tool's scope is
> part of its result; pretending otherwise is how benchmarks become folklore.

---

## 1. Capacity is a property of a scenario, not of a contract

The single most important limitation. `N* = 1,318` is only meaningful as
"`N* = 1,318` **for this population, these operations, this ordering, under these
limits**". Change "release" to "dispute and resolve" and the number changes, because
the event and write profile changes.

Headroom mitigates this by making the scenario explicit and hashing it into the
certificate. It cannot mitigate it by producing a universal number, because no such
number exists.

**Consequence for readers:** never quote an `N*` without the scenario. A certificate
without its scenario is an anecdote.

---

## 2. Offline host measurement is not mainnet

The runner uses an in-process Soroban host with captured network cost settings. That
gives the protocol's own accounting — but not a real ledger, real fee competition, or
real congestion.

What can differ:

| Effect | Impact on numbers | Handling |
|---|---|---|
| host version vs network protocol version | small cost differences across releases | protocol version recorded; snapshots pinned per version |
| ledger entry size changes across protocol versions | entry-size cliff moves | snapshot hash in every certificate |
| archival / TTL behaviour | rent estimates, not failures | rent labelled *modelled* |
| real fee market | none — fees are not measured | out of scope, stated |
| live state vs synthetic state | a real contract may carry history the scenario omits | `snapshot` input for live state; limitation noted in the entry |

Cross-checking against testnet `simulateTransaction` is a planned v1.0 feature. Until
it exists, the claim is "protocol-model capacity", not "network capacity".

---

## 3. Single-transaction view of contention

Contention is derived from write-set overlap, not measured. The real world has
account-sequence ordering, fee bidding, and validator scheduling. Headroom's
contention output is a **warning class** with a stated assumption (one serialised
write per ledger), not a throughput prediction.

Anyone who quotes the contention number as TPS is misusing the tool.

---

## 4. Adapters cover common shapes, not all shapes

Declarative adapters exist for registry, escrow, token, and a raw escape hatch, with
NFT and orderbook planned. A contract whose population semantics are unusual — a
graph of dependencies, a contract that deploys contracts, a cross-contract call chain
where the callee's state grows — may need a `raw` scenario that understands less and
therefore proves less.

`raw` scenarios are accepted in `evidence/` and rejected in `index/`, because an
indexed entry must demonstrate that its scenario models the design.

---

## 5. Correctness is out of scope

A contract can have enormous headroom and be completely broken. Headroom does not
fuzz, does not check access control, does not prove invariants. It measures whether
operations succeed at a population size, not whether they should.

An entry in the Capacity Index is never an endorsement, and the index README says so
at the top.

---

## 6. Auth and signatures are simplified

Setup typically uses mocked or pre-authorized invocation to build the population
efficiently. That is fine for measuring storage and events; it is wrong if the
question is "what does the auth path cost at scale". Scenarios that need real auth
must say so, and the certificate records the auth mode.

---

## 7. Randomness and time

A contract that uses a PRNG or a timestamp produces different state per run. The
runner seeds what it can and fixes the ledger sequence; it cannot make a contract
deterministic that is not. Such scenarios are flagged `nondeterministic` and are not
indexed.

---

## 8. Limits will move

Every number is tied to a snapshot, and snapshots go stale. `verify` warns; a
certificate does not become false, it becomes **dated**. The index re-runs after
upgrades precisely because a stale capacity claim is worse than no claim.

---

## 9. Ladder cost grows with resolution

Finding `N*` exactly costs a bisection. At `resolution = 1` over a range up to
100,000, a run is 20–25 rungs. That is fast for a fixture and slow for a large
population with expensive setup. The mitigation is a coarser resolution plus an
explicit interval, which is why the interval is always reported — but a user who
wants the exact unit pays for it.

---

## 10. The index is not statistically representative

Five, twenty, or fifty contracts is not a survey. Entries are chosen to cover
failure modes and contract families, and the README says so. Any claim of the form
"most Soroban contracts die at N" is unsupported by this dataset and will not be
made by the project.

---

## 11. No mainnet execution

The tool never sends a transaction. It cannot measure anything that only manifests on
a live network, including fee inclusion, ledger congestion, and real archival timing.
This is a deliberate safety property: no keys, no funds, no possibility of an
incident caused by a capacity test.

---

## 12. What this leaves for other tools

Deliberately, the following remain someone else's job:

| Question | Tool |
|---|---|
| Did this PR regress cost? | WeighIn, `soroban-meter` |
| Is my state about to expire? | Soroban Sentinel |
| Does it cost more on network than locally? | `simulateTransaction` parity (planned cross-check) |
| Is the logic correct? | tests, fuzzing, audit |
| How much traffic can the network take? | load testing |
| How do I build it at all? | Soroban Forge |

The boundaries are not modesty. They are what makes a single number trustworthy.

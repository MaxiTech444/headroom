# Glossary

> The vocabulary this project uses, defined once so a report, a doc and an issue
> all mean the same thing.

---

**Binding limit** — the limit with the highest utilisation for a measured operation.
The one that will fail first if the population grows. A certificate names exactly one
binding limit per failing rung.

**Capacity** — the largest population at which every operation in a scenario still
succeeds under a given limit snapshot. Always relative to a scenario; never a property
of a contract alone.

**Capacity certificate** — the machine artifact of a run: scenario hash, contract
hash, limit snapshot hash, every rung's measurements, `N*`, binding limit, marginal
cost model, and a hash over the whole thing. Verifiable without trusting the author.

**Capacity Index** — the public, append-only catalogue of certificates for real
designs, each with a reproduction command.

**Cliff** — the transition point where an operation goes from succeeding to failing
because a limit was crossed. Distinct from a *cost increase*, which is continuous.

**Confirm step** — the re-run of the boundary rung from scratch, to prove `N*` is not
an artefact of iteration order or accumulating state.

**Contention** — a throughput ceiling caused by multiple transactions writing the same
ledger entry, which serialises them. Not a resource limit; reported as a warning.

**Entry size limit** — `contract_data_entry_size_bytes` = 65,536 B. The whole
instance storage is one entry, so all instance keys share this budget.

**Footprint** — the set of ledger entries a transaction declares it may read or write.
Capped at 400 entries per transaction.

**Headroom ratio** — `1 − utilisation` at `N*`. How much of the binding limit is still
unused when the population is at its maximum. A design that "works up to 1,200" at 99%
utilisation has almost no headroom and is one field away from failing.

**Illustrative** — a number produced by arithmetic or reasoning, clearly not a
measurement. Every such number in this repository is labelled, and never appears in a
certificate.

**Instance storage** — the single ledger entry attached to a contract, loaded on every
invocation. Bounded by the entry size limit; unsuitable for unbounded collections.

**Interval** — the range within which `N*` is known, from the last passing and first
failing rung after refinement. Reported even when resolution is 1, because bisection
only proves a bound.

**Ladder** — the sequence of population sizes a run executes: geometric probe, binary
refinement, confirmation.

**Limit snapshot** — a committed JSON file of network settings with a content hash.
The only source of limits; nothing is hardcoded.

**Marginal cost** — resource cost added per additional unit of the population. The
slope of the cost curve, not its value. Tells you how fees and rent grow before
anything breaks.

**Measured** — a value read from the host during execution. Distinct from *modelled*
(derived, e.g. rent) and *illustrative* (arithmetic).

**Modelled** — derived from measurements plus a documented formula, e.g. annual rent at
a population size. Always labelled, always with the formula stated.

**N** — the population size of a rung: the number of units the scenario builds.

**N\*** — the largest N at which every measured operation succeeds. The project's
headline output, always reported with its interval and its binding limit.

**Population** — the set of units and actors a scenario builds at a given N. The axis
nobody else measures.

**Protocol model** — the cost settings and limit values of a specific protocol version
plus network-settings change (e.g. "Protocol 29 + SLP-0007"). What an offline
measurement is actually measuring.

**Rung** — one execution of the scenario at one N. One row in the ladder.

**Runner-up limit** — the second-highest utilisation at a rung. Determines what binds
after the first limit is fixed, which is why it is reported.

**Scenario** — the versioned declarative file that defines contract, population,
workload, ladder, limits and policy. The question, in full.

**SLP** — Stellar Ledger Proposal. Changes to network settings, such as SLP-0007
(live 2026-10-22) which raised per-ledger caps and halved resource fee rates.

**Unit** — one element of the population: a user, an escrow, a listing, a proposal. The
noun in the report ("1,318 escrows").

**Utilisation** — `measured / limit` for one resource on one operation. A fraction;
above 1.0 is a failure.

**Verification** — `headroom verify`: recompute the hash chain, re-run the boundary
rungs, and report drift. The difference between a benchmark and a claim.

**Write bytes** — serialized bytes written by a transaction, capped at 132,096 per
transaction and 297,000 per ledger. Binds before write-entry count when values are
large.

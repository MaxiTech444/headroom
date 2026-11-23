# The Problem

> **Summary.** Soroban charges for and limits *one transaction at a time*. Almost
> all tooling inherits that frame. But the failures that take contracts down are
> functions of **how much state exists**, not of how expensive one call is. Nobody
> measures that axis, so nobody knows where their design ends until it ends.

---

## 1. Soroban's cost model is per invocation. Its failure model is per population.

A Soroban transaction declares a footprint, executes, and is metered against a
fixed set of per-transaction limits. Every limit is expressed as a property of
*that transaction*:

| Per-transaction limit | Value (Protocol 29) |
|---|---:|
| CPU instructions | 400,000,000 |
| Memory | 40 MiB |
| Footprint entries (read + write) | 400 |
| Disk-read entries / bytes | 200 / 200,000 |
| Written entries / bytes | 200 / 132,096 |
| Transaction size | 132,096 B |
| Events + return value | 16,384 B |

A developer reads this table and concludes: *keep each function under the limits.*
That conclusion is correct for a single call and wrong for a contract.

Because the *state* a call touches is not a constant. It is a function of how many
users exist. A function that writes one entry per user is fine at 40 users and
impossible at 40,000. The code is identical. The tests are identical. The audit is
identical. Only the population changed.

**No tool in the ecosystem varies that population.**

---

## 2. The four cliffs, concretely

### 2.1 The instance entry is a single 64 KiB box

Instance storage is one ledger entry. Every contract invocation loads all of it.
The entire serialized entry — key, value, TTL field, all of it — is capped at
`contract_data_entry_size_bytes` = **65,536 B**.

So this pattern is a time bomb:

> *"Store a `Vec` of all registered users in instance storage."*

It works. It passes tests. It passes review — the docs even warn about it, and it
still ships, because the warning is abstract and the failure is far away.

At some N the write fails. Not "costs more" — fails. The contract instance cannot
be updated, so *every* mutating function on that contract is now dead until
someone ships a migration. Reads still work, which makes the incident worse: the
dashboard looks fine while nothing can be created.

**The wall is at a number nobody knows, and the number is discoverable in about
four seconds of measurement.**

### 2.2 Write caps arrive long before CPU

A batch operation — an airdrop, a settlement loop, a multi-recipient payout — is
the natural answer to "one transaction per recipient is expensive". It works until
the batch hits one of:

- **200 written entries** per transaction
- **200 disk-read entries** and **200,000 read bytes**
- **132,096 written bytes**
- **400 total footprint entries**

CPU is almost never the binding constraint for writes. A batch of 150 transfers
touches 300 entries read and 300 written: the write cap is exceeded at around 100
recipients while CPU sits near 15% of its budget. The developer optimizes compute,
which was never the problem.

### 2.3 Events bind first, and nobody expects it

Events plus the top-level return value share **16,384 B** per transaction. One
event per item in a loop makes the budget a function of item count — and a single
event with a few addresses and amounts is a few hundred bytes encoded.

Published resource notes from real projects show this binding *early*. AXIS, for
example, documents its fill path failing on events at roughly twenty fills, long
before footprint or CPU become relevant. Twenty. A loop you would never think of
as heavy.

The general lesson: **the limit that binds is not the limit you are watching.**
Any tool that reports one number hides the one that matters.

### 2.4 The invisible cliff: contention

Every per-transaction measurement can pass while the system's throughput collapses.

If every transaction writes the same ledger entry — a global counter, a protocol
config, a single pool's reserves — those transactions cannot be parallelised. The
ledger can include 2,000 Soroban transactions, but if they all touch one entry,
they execute sequentially. Throughput becomes one per ledger: roughly one every
five seconds, regardless of how many users are waiting.

There is no failing transaction to profile. There is no error message. There is
just a queue. And the only way to see it is to model the **population's write
pattern**, which no profiler does.

---

## 3. Why existing tools structurally miss this

| Tool | Measures | Why it cannot see the cliff |
|---|---|---|
| `cargo test` with `env.budget()` | CPU/memory of one call | test state is whatever the test built — usually one or two entries |
| `soroban-meter`, `soroban-budget-assert` | per-function cost, per-revision regression | a fixed, small fixture; no population axis |
| `simulateTransaction` | one transaction against live state | you must already have created the state you want to test |
| Fuzzers (`soroban-fuzzer`, `CrashLab`) | correctness across random sequences | state size is incidental, not the variable under test |
| Load testing scripts | network throughput under concurrency | measures the network, not the contract; needs funded accounts and time |
| `soroban-sentinel`, TTL monitors | entries that exist today | cannot report the entries a design can never create |
| Audits | logic and known vulnerability classes | "unbounded instance storage" is a checklist item, not a measured number |

Each one is answering a real question. The gap is not quality — it is **the axis**.
They all hold N constant. Headroom varies N.

---

## 4. Who feels this, and when

**Contract teams before launch.** "Will this survive our first 10,000 users?" is
currently answered with opinion. A capacity certificate answers it with a number
and a reproduction command.

**Teams choosing a storage layout.** The choice between instance storage, one
persistent entry per user, and sharded buckets is made on vibes. The three layouts
have different cliffs at different N with different costs. Headroom measures all
three from the same scenario, so the choice becomes evidence-based.

**Auditors and reviewers.** "This is unbounded" is a comment. "This fails at
N = 1,318, here is the certificate and the four-second reproduction" is a finding.

**Protocol maintainers and SLP authors.** When limits move — as they did with
SLP-0007 on 2026-10-22, doubling per-ledger entry caps and halving resource fee
rates — someone should be able to say what that did to real contracts' capacities.
The Capacity Index makes that a diff instead of a guess.

**Grants and ecosystem reviewers.** A project claiming "scales to millions" can be
asked for a number. Headroom is how that number gets produced.

---

## 5. Why now

Three things changed recently enough that this is buildable today and wasn't two
years ago:

1. **Protocol 29 and SLP-0007 (live 2026-10-22)** moved every per-ledger limit.
   Capacity assumptions written into contracts and docs are now stale, and nobody
   has a way to re-measure them. The first tool to do so owns the conversation.
2. **Native host simulation is now practical.** Stellar's host can be exercised
   in-process with captured network cost settings and a ledger snapshot. That is
   what makes a ladder of thousands of runs feasible in minutes, offline, with no
   funded accounts — and it is the same technique WeighIn already uses for
   single-invocation measurement.
3. **The ecosystem is past "does it work" and into "does it hold".** Contracts in
   Stellar today are shipping escrows, AMMs, registries and NFT collections. The
   next question for all of them is capacity, and the tooling has not caught up.

---

## 6. The thesis, in one paragraph

A contract's capacity is a measurable property: given a scenario, there is a
largest population N\* at which every operation still succeeds, and there is a
specific protocol limit that fails first. That number is deterministic, cheap to
compute offline, and changes what teams build. Nobody measures it today. Headroom
measures it, attributes it to a limit, proves it with a re-derivable certificate,
and publishes a public index of real contracts' ceilings so the ecosystem can
compare designs instead of arguing about them.

---

## 7. What success looks like

- A developer runs one command before choosing a storage layout, and the choice is
  no longer an opinion.
- A reviewer rejects a design because the certificate shows 900 users of headroom
  against a roadmap of 50,000 — with a link, not a hunch.
- A grant applicant answers "does it scale?" with a certificate instead of an
  adjective.
- After the next SLP changes the limits, the Capacity Index shows exactly which
  designs lost capacity and by how much, within a day of the upgrade.
- The phrase "we found the cliff at 1,318" becomes normal in Soroban code review.

None of that requires new protocol features, a network, or a funded account. It
requires measuring the axis nobody measured.

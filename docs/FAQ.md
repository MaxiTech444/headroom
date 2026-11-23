# FAQ

> The questions a reviewer, a contributor, or a skeptical contract developer will
> ask — answered without hedging.

---

### Isn't this just a load test?

No, and the difference is the axis. A load test generates **traffic** against a
network and measures throughput, latency and error rate under concurrency. Headroom
generates **state** and measures at what population a contract's own resource usage
crosses a protocol limit.

A load test at 500 requests/second will not tell you that your registry dies at 1,318
registrants, because you would have to register 1,318 accounts to find out, and even
then you would see failures without knowing which limit caused them.

They are complementary: Headroom tells you which N is interesting, a load test tells
you what the network does at that N.

---

### Why not just call `simulateTransaction` at increasing N?

You can, and people do — that is the manual version of this tool. Problems:

1. **You must create the state first.** Simulating at N = 10,000 means 10,000 real
   transactions on a live network with funded accounts.
2. **You get a cost, not an attribution.** Simulation reports resources; it does not
   tell you which limit is closest or which key is responsible.
3. **It is not reproducible.** Live state changes between runs; the result is a
   snapshot of a moving target.
4. **You cannot do it in CI**, offline, in seconds, for free.

Headroom's offline result should agree with simulation — and cross-checking that
agreement is a planned v1.0 feature precisely because the claim deserves testing.

---

### Isn't N\* just the entry-size limit divided by the record size?

For one design, yes — and the fixture proves the tool reproduces that arithmetic.
That is the point of the first proof artifact: if simple arithmetic predicts the
cliff, the measurement must match it.

But most designs are not one division. A batch payout hits the minimum of four curves
(write entries, write bytes, footprint, transaction size) and no hand calculation
tells you which one binds first. A loop emitting events hits a limit that has nothing
to do with the data size. And contention has no arithmetic answer at all.

The arithmetic works for the easy case, which is exactly why the tool should be
trusted on the hard ones.

---

### Which limit actually binds in practice?

Across the designs in this project's fixture set, the expected order is:

1. **entry size** — anything holding a growing collection in one entry
2. **write entries** — any batch that touches more than 100 distinct entries
3. **events + return bytes** — any loop emitting one event per item
4. **write bytes** — wide values written in bulk
5. **disk-read entries/bytes** — enumeration-style queries
6. **CPU** — rarely, and almost never for storage-bound workloads

CPU is last. That inversion — the one resource everyone measures is the one that
almost never binds first — is the project's core argument.

---

### How is this different from WeighIn, which the same author wrote?

WeighIn compares two revisions at the same small scale and fails a PR on a percentage
regression. It answers *"did this change make it worse?"* — a question about change.

Headroom compares two scales of the same revision and reports where the design ends.
It answers *"how much room is left?"* — a question about capacity.

A contract can pass WeighIn perfectly (0% regression) while having 900 users of
headroom against a roadmap of 50,000. Both tools are needed, and keeping them in
separate repositories keeps each claim narrow enough to check.

---

### Is this a fuzzer?

No. A fuzzer explores sequences to find correctness failures; state size is
incidental. Headroom executes a **declared** population in a **declared** order, so
the result is one number per design rather than a corpus of crashes. Different
outputs: a fuzzer produces counterexamples, Headroom produces a ceiling.

---

### Why Rust and not TypeScript, like WeighIn?

Because the measurement must live in-process with the Soroban host. Calling out to a
native helper per rung would dominate a run that executes dozens of rungs at
increasing N — a ladder is thousands of invocations, not two.

WeighIn can afford a TypeScript orchestration layer with a Rust measurement helper
because it measures two revisions. Headroom's loop is the hot path.

---

### Do I need a funded account, Docker, or a network?

No. A run needs: the contract's WASM, a scenario file, and a limit snapshot. That is
it. No keys, no funds, no Docker, no RPC. This is both a safety property and what
makes the tool usable in CI.

Only `verify --check-network` and live-state snapshot capture touch a network, and
both are optional.

---

### How do you know the limits are right?

Every limit comes from a pinned snapshot generated from the network's own settings,
with its hash in the certificate. `headroom limits --check` reports when the snapshot
differs from the live network.

And because limits are network settings that validators can change — as SLP-0007 did
on 2026-10-22, doubling per-ledger entry caps and halving resource fee rates — the
project treats a stale snapshot as a normal, detected condition rather than an
embarrassment. Old certificates are dated, not deleted.

---

### What stops someone from faking a certificate?

The hash chain. A certificate's hash covers its canonical form, which includes the
scenario hash, the limit snapshot hash, the WASM hash, and every measurement.
`headroom verify` recomputes it and re-runs the boundary rung.

Someone can still publish a certificate produced by a modified tool — no scheme
prevents that. What they cannot do is quietly edit a number and have it verify, and
the reproduction command makes independent checking cheap. That is the standard the
project holds itself to; it is also the standard the index applies to others.

---

### What if my contract has no cliff — is the tool useless to me?

No, and the project publishes that result. The token fixture is designed to have no
cliff below 100,000, and its certificate says so.

"No cliff found below N = 100,000, cost per operation flat at 41,200 CPU" is a
marketable fact. It is also the result that keeps the tool honest: a benchmark that
finds a wall everywhere is measuring its own assumptions.

---

### Why does the report show a "runner-up" limit?

Because the second-nearest limit determines what happens *after* you fix the first.
A design that moves from "entry size at 1,318" to "write bytes at 1,400" has not been
fixed; it has been moved. Reporting the margin to the runner-up is what lets a team
choose a fix that buys real room instead of buying a week.

---

### Does this work for contracts I cannot open-source?

Yes. A certificate contains the WASM hash and the measurements, not the source. The
index is opt-in and requires the author's blessing; private measurements can stay
private while still being verifiable by whoever holds the certificate.

---

### How long does a run take?

Targets: ~250 ms per rung at N = 1,000 with five operations, under 20 seconds for a
12-rung ladder up to N = 50,000, offline.

If a scenario is slower than that, the answer is fewer rungs plus a stated
extrapolation — not a longer wait. A capacity tool that takes an hour will not be run
before the design decision it is meant to inform.

---

### What is the biggest risk to this project?

That the measurement is subtly wrong in a way that produces plausible numbers — wrong
cost settings, an undeclared entry, a setup step that secretly satisfies the
measurement. That is why the first artifact is a fixture whose cliff is predicted from
arithmetic before any code runs, and why the CI job re-runs and diffs certificates on
every push.

A wrong number that reproduces is worse than a missing number, because it will be
quoted.

---

### Can I use it before v0.1?

Not for a claim. The repository is a specification until the fixture runs. But the
scenario format and the certificate format are stable early, and issues are scoped so
that documentation, renderers and schema work can land before the runner is finished.

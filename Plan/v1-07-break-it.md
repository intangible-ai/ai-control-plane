# Slice v1-07 — Break it

**Version:** v1 · **Slice:** 07 of 08 · **Produces a number?** Several — and **deliverable #3**.

> **Build-sequence note (added after the portfolio's `LATEST EDIT` revision).** This project is
> **position 0 — `#30-thin`, the control plane spine** — in the only build sequence now in force.
> That section supersedes Parts 10, 11 and 12 of the portfolio file. Position 0's scope is stated
> there as: *"OTel GenAI spans, token and cost accounting, a trace store, and the provider adapter
> interface... The telemetry half of backpressure lives here too — a bounded queue that drops spans
> before user traffic is ever dropped."* Two consequences run through every slice below:
> **(a)** the bounded span queue is now **required scope**, not a discovered fix; and
> **(b)** what comes next is **position 1, `#7` (prefix-cache-aware context assembler)**, which is
> where the *reusable* mock-LLM server and load generator belong. See `v1-05` §0.

> Read `..\..\Shared Context\control_plane_thin_plan.md` §7 (*"The failure mode we expect to find"*)
> and the corrected prediction recorded during `v1-03` before answering anything.
>
> This slice is an **experiment**, not a build. Every fault gets a written hypothesis *before* it is
> injected. A result recorded without a prior prediction is an observation; a result recorded
> against a prediction is a finding — and only one of those is worth writing up.
>
> **Large slice — planned split point in §7.**

### What the `LATEST EDIT` sequence changed about this slice

The bounded span queue **is no longer a fix this slice might discover — it is position 0's stated
scope**: *"The telemetry half of backpressure lives here too — a bounded queue that drops spans
before user traffic is ever dropped — because that is the only half of the old Pass 2 spike work that
can be built and proved against a single system."*

Three consequences, and the first is the one that could go wrong:

1. **The measure → break → fix ordering does not change.** It is tempting to read "required scope"
   as "so build it in `v1-03` and skip the experiment." Do not. The requirement is that the queue
   **exists and is proved**; the proof *is* the before-number. Plan §7's instruction still stands
   verbatim — *"Do not let a later session 'helpfully' skip ahead and implement the queue in slice
   03."* The sequence made the destination mandatory, not the route shorter.
2. **The queue can no longer be dropped if the faults turn out benign.** Previously (§2) the honest
   move if faults 1 and 7 caused no user-visible damage was to write that up and not build a fix.
   That option is gone: the queue ships either way. What *does* stay data-driven is **which fault is
   the headline**, and what else gets fixed. If the queue turns out to prevent nothing measurable,
   **that is the finding** — write it up as "position 0 required this and here is what it actually
   bought, measured" rather than manufacturing a disaster it saved you from.
3. **The word "proved" is doing work.** Position 0 asks for a queue *proved against a single system*.
   A queue with no fault run behind it is asserted, not proved. This slice is that proof, and it is
   the reason the slice survives the resequencing intact.

---

# Concepts for this slice

Plan §8: *"`v1-07` opens with backpressure, load shedding, and fail-open vs fail-closed."*
`v1-00` §5.3 and §5.4 gave the shapes. This is the layer that turns them into decisions with
numbers attached.

### 1. Fault, error, failure — three words, used precisely

- A **fault** is the defect or the injected condition. Postgres is down.
- An **error** is the internal deviation it causes. The insert returns a connection error.
- A **failure** is the externally visible deviation from the service's contract. A user gets a 500.

The whole discipline of this slice lives in the gap between the second and the third: **a resilient
system is one where faults produce errors that never become failures.** Saying "it broke" collapses
three distinct things and hides where the intervention belongs.

For a control plane specifically, the contract is asymmetric and it is worth stating as a design
principle before measuring anything: a telemetry fault becoming a telemetry failure is acceptable.
A telemetry fault becoming a *user-traffic* failure is not. That is plan §6's availability row —
*"telemetry is dropped before user traffic ever is"* — expressed as a rule you can test.

### 2. Why latency explodes rather than degrades — the queueing result worth knowing

The single most useful piece of queueing theory for a backend engineer: for a simple queue at
utilisation ρ, average wait time scales as **ρ / (1 − ρ)** times service time.

| Utilisation | Wait ≈ |
|---|---|
| 50% | 1× service time |
| 80% | 4× |
| 90% | 9× |
| 95% | 19× |
| 99% | 99× |

Utilisation went up 19 points from 80% to 99%; wait time went up 25×. **This is why the knee
measured at `v1-05` is a cliff and not a slope**, and why "we have plenty of headroom, we're only at
85%" is a sentence that precedes an incident.

Two consequences that shape this slice:
- Adding a fault that slows one component does not slow the system proportionally — it pushes
  utilisation toward 1 and the response is non-linear. Expect surprises to be large.
- Capacity planning targets ~70% and not 95%, and now you can say why with arithmetic rather than
  folklore.

### 3. Slow is worse than dead

The most counter-intuitive rule in reliability, and the one this slice should demonstrate:

**A dead dependency fails fast.** The connection is refused in microseconds, the error propagates,
the caller moves on. Capacity is barely consumed.

**A slow dependency holds resources.** Every in-flight request keeps a connection, a goroutine, a
buffer and a pool slot. Requests arrive faster than they drain. The pool exhausts. Now requests that
have nothing to do with the slow dependency are blocked waiting for a connection — and the blast
radius has spread to unrelated code paths.

So the fault list must include **slow Postgres**, not only dead Postgres — and the prediction should
be that slow is the worse of the two. This is also why timeouts are non-negotiable: a timeout is
what converts a slow dependency into a fast-failing one, at a moment of your choosing.

### 4. Timeouts, and the budget that has to shrink down the chain

**Every network call gets a deadline.** Go makes this pleasant — `context.WithTimeout`, and the
context threads through the call chain you already built.

The part people miss: **a timeout that is longer than your caller's timeout is useless**. If the
gateway gives the client 30 seconds and waits 60 for the provider, the client has already gone at
30 and the gateway spends 30 more seconds working on an answer nobody will read — while holding a
connection and, on a streaming call, **paying for tokens** (`v1-05` Concepts §8).

The rule is a shrinking budget: each hop takes the remaining budget, subtracts what it needs for
itself, and passes less down. `context.Context` propagates deadlines automatically when you pass it
through, which is another concrete answer to "why does every Go function take a `ctx`."

### 5. Backpressure vs load shedding — related, not the same

- **Backpressure** — telling upstream to slow down. Bounded queues, blocking sends, 429s with
  `retry-after`. Requires an upstream that will listen.
- **Load shedding** — rejecting work at the door so that accepted work still completes. Requires no
  cooperation and is the only option when upstream is the open world.

A control plane needs both, in different places: backpressure inside the process (bounded channel
between span production and the writer), shedding at the ingest edge (the OTLP endpoint under
overload).

**Which end of a full queue do you drop from?** Not a detail — a policy:

- **Drop newest** (reject the incoming span): simple, and it preserves a coherent older window.
- **Drop oldest** (evict to make room): keeps recency, which usually matters more for telemetry.
- **Drop by priority:** and this is the right answer for us, because plan §6 commits to
  *always-keep-errors*. A span with `status = Error` is worth more than a successful one, so the
  drop policy should shed successful spans first and keep error spans until the queue is truly full.

That is a genuinely good design detail: the drop policy is derived from a stated product
requirement rather than from convenience. Whichever is chosen, **the counter is not optional** —
an uncounted drop is data loss you cannot even report.

**Do not confuse this with sampling — the new sequence separates them deliberately.** Two different
mechanisms that both discard spans:

| | **Queue drop policy — here, position 0** | **Tail-based sampling — position 7 (`#21`)** |
|---|---|---|
| When it decides | at enqueue, when the queue is already full | after the trace completes, on every trace |
| Why it discards | it physically cannot keep up right now | keeping everything is structurally impossible at this volume |
| Normal-load behaviour | **never fires** — a drop is an incident signal | fires constantly, by design |
| Scope of the decision | one span, in one process | the whole trace, centrally |

The new sequence puts *"tail-based sampling, always-keep-errors and hard label limits"* at position
7 because voice is the first system emitting fast enough to force them. Position 0 keeps everything
(`AlwaysSample`, `v1-02`) and drops only under duress — and **`cp_spans_dropped_total` reading
nonzero at normal load is a bug, not a policy working.** Say that in the metric's help text, because
the two mechanisms are easy to conflate later when both exist.

### 6. Fail-open or fail-closed, decided per component and written down

`v1-00` §5.4 gave the definitions; here every component gets an explicit answer, and the answer goes
in a comment next to the code:

| Component | Choice | Why |
|---|---|---|
| Span write to Postgres | **fail-open** | nobody's request should fail so we can log it |
| OTLP ingest under overload | **shed, with a counted 503** | protects the spans already accepted |
| Provider call | **fail-closed** | there is no meaningful degraded answer to a chat request |
| Cost calculation for an unknown model | **fail-visible** | `NULL` + status, never a plausible-looking 0 (`v1-04`) |

The senior position is not "fail-open is better." It is **knowing it is a per-policy decision and
being able to justify each one** — which is the sentence to have ready for an interview.

### 7. Recovery is a measurement too

Everyone tests the fault. Fewer test the *end* of the fault, and that is where a second class of bug
lives:

- **Does it recover at all**, or does a pool stay poisoned and require a restart?
- **How long** does it take to return to baseline p99 after the dependency comes back?
- **Is there a thundering herd on recovery** — every retry timer firing at once, knocking the
  just-recovered dependency straight back over? (This is jitter's second job.)
- **What happened to the data** during the outage: dropped, buffered, or duplicated on retry?

So each fault run has three phases and all three are recorded: **steady → fault → recovery.**

---

# 1. State going in

From `v1-06`, committed:

- Two real provider adapters (`mock` over HTTP, `anthropic`), runtime-selected
- `cmd/mockllm` with `/_control` for runtime fault knobs; `cmd/loadgen` open-loop
- Cost engine validated against a real provider
- `/metrics` on gateway and collector, including `cp_spans_dropped_total` **reading 0**
- **The v1 baseline in `bench/results/`**, with SLO verdicts
- The **corrected prediction** written at `v1-03`, which this slice now tests

**Everything in this slice runs against `mockllm`. Cost: ₹0.** Faults against a paid provider would
spend budget to learn nothing that injection cannot teach.

---

# 2. What this slice adds

Two phases: find the failures, then fix the worst one.

```
  bench/scenarios/
    fault-db-dead.yaml        <- postgres stopped mid-run
    fault-db-slow.yaml        <- postgres reachable but slow
    fault-stream-stall.yaml   <- provider stalls at ~90% of the stream
    fault-garbage.yaml        <- provider returns malformed bodies
    fault-spike.yaml          <- 10x step in arrival rate
    fault-disconnect.yaml     <- client aborts mid-stream
    fault-collector-down.yaml <- collector stopped; the gateway exporter is on its own
  bench/results/
    fault-*.json              <- steady / fault / recovery for each
  docs/failure-modes.md       <- hypothesis, method, result, fix, re-measurement
  internal/telemetry/
    queue.go                  <- THE FIX: bounded queue, priority drop, counters
```

Faults to run, each with a written hypothesis first:

| # | Fault | How | Watch |
|---|---|---|---|
| 1 | **Postgres dead** | `docker compose stop db` mid-run | gateway error rate, collector behaviour, spans lost, `cp_spans_*` counters |
| 2 | **Postgres slow** | pause the container, or a `pg_sleep` trigger, or a tiny `max_connections` | pool exhaustion, whether it is worse than #1 (Concepts §3) |
| 3 | **Stream stalls at 90%** | `mockllm /_control` | does the span ever close? does the goroutine leak? does the deadline fire? |
| 4 | **Malformed response** | `mockllm` returns truncated/garbage JSON | panic vs handled error; does one bad body take out a handler? |
| 5 | **10× spike** | `loadgen` step function | the knee from `v1-05`, GC, queue growth, whether it recovers |
| 6 | **Client disconnects mid-stream** | `loadgen` aborts a fraction of streams | **does upstream generation stop?** this is a cost bug, not a latency bug |
| 7 | **Collector down entirely** | stop the collector | gateway's batch exporter: retries, queue fill, **silent drops** |

Fault 7 is the one most likely to produce the real finding, per the corrected `v1-03` prediction.
Fault 6 is the most unusual and therefore the most interesting in a write-up — very few people
measure whether their proxy keeps paying for tokens after the reader has gone.

**The fix** (expected, but let the data choose it): a bounded in-memory queue between span production
and the writer, with an explicit priority drop policy (Concepts §5), `cp_spans_dropped_total`
incremented on every drop, and a `/metrics`-visible queue depth. Plan §7 names this as the planned
fix — *"drop telemetry before ever dropping user traffic."*

**Do not fix more than the data justifies** — with the one exception the sequence now imposes. The
bounded queue ships regardless (see the note under the header): position 0 requires it. Everything
*else* stays data-driven. If faults 1 and 7 turn out to be benign because the batch exporter was
already async, then the queue is not the *headline* finding, and inventing a disaster it rescued you
from is the same dishonesty as hiding one that happened. Build the queue, measure what it actually
bought, and report that honestly — including "less than expected, and here is why."

---

# 3. Why now

- **Because there is finally a before-number.** Plan §7 arranged the synchronous writer at `v1-03`
  and the baseline at `v1-05` specifically so this slice could produce a delta. Doing it earlier
  would have been guessing; doing it later would mean optimising past the evidence.
- **Because deliverable #3 is a hard requirement.** Plan §7 / CLAUDE.md §4a: *"At least one failure
  mode discovered and written up honestly."* Without this slice the project is not done, regardless
  of how well it runs.
- **Because it is the half of deliverable #2 that is still open.** `v1-05` measured a baseline; a
  baseline that is never beaten is half a deliverable. The fix in this slice is what beats it —
  under fault conditions, which is where it matters.
- **Because `v1-08` writes the README.** A README that claims resilience without a fault run is the
  kind of claim a skeptical reviewer tests in thirty seconds.
- **Because everything needed to do it already exists.** `mockllm`'s control endpoint, `loadgen`,
  `/metrics`, Docker. Nothing new has to be built to run the experiments — which is exactly why
  those tools were built at `v1-05` instead of here.

---

# 4. Done means

- [ ] **every fault has a written hypothesis, recorded before the run**
- [ ] all seven faults executed, each with steady / fault / recovery phases captured
- [ ] `bench/results/fault-*.json` committed with machine state and commit SHA
- [ ] `docs/failure-modes.md` written: hypothesis, method, what happened, why, what changed
- [ ] **the `v1-03` corrected prediction is explicitly evaluated** — right, wrong, or partly, in
      writing. Both plan §7's original prediction and the `v1-03` correction get a verdict.
- [ ] the worst *user-visible* failure is identified and named — not the most interesting one
- [ ] that failure is fixed
- [ ] `cp_spans_dropped_total` moves off zero under load and is visible on `/metrics`
- [ ] the drop policy is explicit, commented, and matches plan §6's always-keep-errors commitment
- [ ] fail-open / fail-closed is decided and commented per component (Concepts §6 table)
- [ ] every outbound network call has a deadline, and the deadlines shrink down the chain
- [ ] **the fault run is repeated after the fix** and the before/after numbers sit side by side
- [ ] plan §7's availability SLO — *"gateway error rate 0%, p99 rise < 10 ms, with Postgres killed"* —
      has a verdict
- [ ] no fault causes a panic, a goroutine leak, or a permanently poisoned pool
- [ ] plan §15 has a `v1-07` row; deliverable tracker items **2** and **3** ticked

---

# 5. What to measure

Per fault, the same five things, so results are comparable:

1. **Gateway error rate** — steady vs during-fault. The number that decides whether a fault became a
   *user* failure.
2. **p50 / p95 / p99 latency** in all three phases. A fault that raises p99 by 3 ms is very different
   from one that raises it by 3 seconds, and "it still worked" hides that.
3. **Telemetry integrity** — spans produced vs received vs written vs rows in Postgres. Four numbers
   that should agree. **The gap is the finding**, and if the gap is invisible in `/metrics`, *that
   is a more serious finding than the gap itself.*
4. **Resource behaviour** — goroutine count, memory, pool waits. Unbounded growth in any of them
   during a fault is a collapse-in-waiting even if the error rate looks fine.
5. **Time to recover** to baseline p99 after the fault is removed, and whether recovery is smooth or
   a herd.

Then the headline comparison, in exactly this shape:

> **Fault:** Postgres killed at 200 RPS.
> **Before:** *X* spans lost with no counter, error rate *Y*%, p99 *Z* ms, recovery *T* s.
> **After (bounded queue + priority drop):** error rate *Y′*%, p99 *Z′* ms,
> `cp_spans_dropped_total` = *N* — **losses now counted rather than silent**, recovery *T′* s.
> **Cost of the fix:** *+d* ms p99 in steady state.

That last line is the one that separates an engineer from an enthusiast: **the fix has a cost, and
you measured it.** A resilience change that claims zero overhead has not been measured.

**Sizing the queue is itself a measurement, not a guess.** Use Little's Law (`v1-05` Concepts §6)
plus a staleness bound: a span that is five minutes late is worthless, so the queue should be capped
by *time* as well as by count. Write down the reasoning next to the constant.

**Also — this is the strongest blog post in the project.** AGENT_INSTRUCTIONS: say so at the moment
the graph is on screen. Measure → break → fix → measure, with real numbers on both sides, is the
narrative the entire portfolio is built to produce. Draft it now.

---

# 6. Out of scope

- **Fixing every fault found.** Fix the worst user-visible one, plus anything that is a two-line
  correctness bug (a missing deadline, an unrecovered panic). Everything else is written up and
  is written up and handed to whichever position owns it (§8a of `v1-08`). A slice that tries to fix
  seven faults finishes none.
- **Faults against the real provider.** ₹0 slice. All injection is local.
- **The user-traffic half of backpressure.** The new sequence is explicit: *"Backpressure's
  user-traffic half belongs here"* — at **position 11 (`#35`, the router)** — *"shed with deadlines,
  admission control, and degrade to a cheaper model, because degrading to a cheaper model is
  structurally impossible before a router exists."* So: shedding *user requests*, admission control,
  and any degrade-to-cheaper-model path are **out**. What stays in is the telemetry half (the span
  queue), shedding on the *OTLP ingest* endpoint (that is telemetry traffic, not user traffic), and
  plain deadlines on outbound calls, which are hygiene rather than admission control.
- **Circuit breakers.** Tempting and probably premature: with two providers and no fan-out, a
  breaker adds a state machine and a new failure mode of its own. Note it as a candidate for
  position 11 (`#35`) **if** the data shows repeated hammering of a dead dependency; do not add it
  on principle.
- **Tail-based sampling and cardinality limits.** Position 7 (`#21`) — see Concepts §5.
- **ClickHouse / columnar storage.** Now **position 8 (`#30-store`)**, so it has a home and is not
  this slice's problem. What *is* this slice's job is handing that project its justification: if the
  fault runs expose an ingest wall, record the number, the hardware and the commit beside it.
  Position 8 opens with *"measured the wall at N spans/sec"* — `v1-03`, `v1-05` and this slice are
  collectively where N comes from. If the wall never materialises, say so plainly; position 8's own
  entry allows for being cancelled by the measurement.
- **Chaos-engineering frameworks, service meshes, fault-injection proxies.** `docker compose stop`
  and an HTTP control endpoint are sufficient and comprehensible. A framework here would add a
  dependency and obscure the mechanism.
- **Multi-node, replication, failover.** One machine. v1 measures a single instance honestly.
- **Rewriting the collector for throughput.** If ingest is the bottleneck, that is position 8's
  problem (`#30-store`), armed with the before-number produced here — not an in-flight rewrite inside
  the slice that produced the number.

---

# 7. The split point

**`v1-07a` — find the failures.** Write the hypotheses, run all seven faults, capture steady /
fault / recovery, fill in `docs/failure-modes.md`. **No fixes.** Ends with a ranked list of what
actually broke and how badly.

**`v1-07b` — fix and re-measure.** Implement the fix the data chose, re-run the relevant faults,
produce the before/after table, measure the fix's steady-state cost, tick the deliverables.

The split is genuinely useful here beyond session length: **running experiments and fixing bugs are
different modes of attention.** Fixing while measuring is how a "finding" turns into a vague memory
of something that was broken for a while and then wasn't. Finish the observations, write them down,
*then* change code.

---

# Commit plan

1. `feat(bench): fault injection scenarios`
2. `docs: failure mode hypotheses` — **committed before the runs, so the predictions are timestamped**
3. `docs: failure mode results`
4. `feat(telemetry): bounded span queue with priority drop policy`
5. `feat(metrics): spans dropped and queue depth counters`
6. `fix: deadlines on all outbound calls` *(if the runs justify it)*
7. `docs: before/after fault comparison and the cost of the fix`

Commit 2 before the experiments is not ceremony — it is what makes the prediction checkable by
someone reading the git history later, and that is a genuinely unusual thing to find in a portfolio
repo.

---

# Hand-off — what to record in the plan file

- §15 row for `v1-07`: commit SHA, the headline before/after, the cost of the fix, the surprise.
- **§7 availability SLO:** verdict.
- **§7's stated failure-mode prediction and the `v1-03` correction:** both evaluated, in writing,
  including whichever was wrong. Plan §14's whole point is that a stated-then-missed prediction is
  an asset.
- **Deliverable tracker:** tick 2 (baseline beaten) and 3 (failure mode found and written up).
- **Hand-offs:** any deferred item this slice's data now *justifies* — tail sampling (position 7),
  the columnar store (position 8), circuit breakers (position 11) — each with the number that
  justifies it. There is no v2 of this project to collect them (`v1-08` §8); they belong to the
  positions that own them, and they travel as numbers, never as opinions.

---

# Next

`v1-08` — onboard and write up. A ~50-line Python agent through Door A with the stock OTel SDK, one
trace spanning two languages, the README, the final SLO scorecard, and the done gate.

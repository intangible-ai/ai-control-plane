# Slice v1-05 — Mock LLM + load generator: **THE BASELINE**

**Version:** v1 · **Slice:** 05 of 08 · **Produces a number?** This slice *is* the numbers.

> Read `..\..\Shared Context\control_plane_thin_plan.md` §7 (the SLO table) and §14 (which of those
> numbers are guesses) before answering anything.
>
> This slice produces **deliverable #1 (a benchmark harness)** and **deliverable #2 (a measured
> baseline)** of the four in plan §7 / CLAUDE.md §4a. Everything after it is compared against what
> lands here, so a sloppy measurement now poisons every later claim.
>
> **Large slice — planned split point in §7.**

---

# Concepts for this slice

Plan §8: *"`v1-05` opens with why the mean is a lie and percentiles are the only honest summary."*
`v1-00` §4.1 made that case. This slice needs the next layer: the four ways a benchmark lies to you
even when you are already using percentiles.

### 1. Coordinated omission — the one that invalidates most homemade benchmarks

Two ways to generate load:

- **Closed loop.** N workers; each sends a request, waits for the response, sends the next. This is
  what almost every hand-written benchmark does, because it is the obvious loop to write.
- **Open loop.** Requests are sent at a fixed arrival rate regardless of whether earlier ones have
  come back.

Closed loop has a fatal property: **when the system gets slow, the load generator stops sending.**
The requests that *would* have arrived during the slow period are never issued, so their latency is
never recorded. The system stalls for 2 seconds, you record one 2-second sample instead of the
hundreds of requests that would have queued behind it, and your p99 comes out beautiful.

That is **coordinated omission** (Gil Tene's term), and it systematically under-reports exactly the
tail you built the benchmark to find. It is not a small correction — it can hide an order of
magnitude.

Real users are an open loop. They do not wait politely for your slow request to finish before
deciding to visit your site. So:

> **`loadgen` is open-loop. Requests are scheduled by a clock, not by responses.**

And the corollary that must be built in: **if the generator cannot keep up with the target rate, the
run is invalid and must say so.** A loadgen that quietly falls behind produces confident, wrong
graphs (plan §4). Record achieved-rate vs target-rate in every result file and fail the run loudly
when it drifts.

### 2. You cannot average percentiles

`p99(run A) = 10ms`, `p99(run B) = 12ms`. The p99 of A and B combined is **not** 11ms, and there is
no arithmetic on those two numbers that produces it. Percentiles are order statistics — they need
the underlying samples, or a structure that can merge them (histogram, t-digest).

Consequence for this slice: **`loadgen` keeps samples, not summaries**, and any per-worker
aggregation merges raw data before computing percentiles. At our volumes (tens of thousands of
samples) keeping every sample is exact and costs a few megabytes — do that, and note in the README
that HDR histograms or t-digest are the answer above roughly a million samples. Choosing the simple
exact thing *and knowing when it stops working* is the senior move; reaching for t-digest at 10k
samples is cargo cult.

### 3. A p99 needs enough samples to exist

p99 from 100 samples is "the worst one", which is noise. As a working rule: you want on the order of
**1,000+ samples for a stable p99** and 10,000+ if you are going to quote p99.9.

So a scenario's duration is derived, not chosen by feel: `duration = samples_needed / target_rate`.
At 200 RPS, 30 seconds is 6,000 samples — enough for p99, thin for p99.9. Write that arithmetic into
the scenario file as a comment. Quoting a p99.9 from a 20-second run is the kind of thing a skeptical
interviewer catches.

### 4. Warmup, and why the first seconds are garbage

Go has no JIT, which removes one class of warmup problem, but three remain: connection pools are
cold, the Postgres buffer cache is empty, and the first GC cycles have not settled. Every result
file must record a **warmup period that is excluded from statistics** — typically 5–10 seconds —
and the fact that it was excluded.

Related and worth stating: run **long enough for GC to happen several times**. A 5-second run of a Go
service may complete without a single GC pause, which makes the tail look impossibly clean.

### 5. Real latency is right-skewed, so the mock must be too

If `mockllm` returns a constant delay, the gateway's measured overhead is misleadingly clean —
there is no queueing behaviour to interact with. If it returns a *normal* distribution, the tail is
symmetric, which real service latency never is.

Service latency is approximately **lognormal**: most requests are near the mode, with a long right
tail and no left tail at all (nothing takes negative time, and nothing is much faster than the fast
path). Model that, and the p99 the gateway reports will be a p99 shaped like the ones it will meet
at `v1-06` against the real provider.

Support constant as an option too — it is the right choice when isolating gateway overhead, because
a constant provider time makes the subtraction unambiguous. **Different distributions answer
different questions:** constant to measure *our* overhead, lognormal to see how the system behaves
under realistic arrival and service patterns.

### 6. Little's Law, as a free sanity check

`L = λ × W` — average concurrency = arrival rate × average time in system.

200 RPS × 250 ms average = 50 requests in flight on average. That single multiplication tells you
whether the connection pool size, goroutine count, and Postgres `max_connections` are in the right
neighbourhood *before* you run anything, and it catches a whole class of "why did it fall over"
confusion. If measured concurrency is far from λ×W, one of the three numbers is wrong or something
is queueing where you did not expect it.

### 7. The measurement is running on the machine it is measuring

`loadgen`, the gateway, the collector, `mockllm` and Postgres will all be on one Windows laptop,
competing for the same cores. This is the single largest caveat on every number this project
produces, and it must be **written next to the numbers**, not discovered by a reader.

Practical handling — do all three:

1. **Record machine state in every result file:** CPU model, core count, total CPU utilisation
   during the run, OS, Go version, Docker running or not, commit SHA.
2. **Prefer rates where total CPU stays under ~50%.** Above that you are measuring scheduler
   contention as much as the gateway.
3. **Say it out loud in the README.** "These numbers were produced with the load generator on the
   same machine as the system under test; treat cross-machine comparisons as invalid" is an honest
   sentence that costs nothing and protects everything else you claim.

### 8. TTFT is a different measurement and needs different plumbing

`v1-00` §4.3: time-to-first-token is what users perceive. Measuring it requires the client to
timestamp the **first byte of the first SSE event**, not the completion of the response. In
`loadgen` that means reading the body incrementally rather than calling something that buffers it
all — an easy thing to get wrong in a way that silently reports total latency as TTFT.

On the gateway side, streaming introduces two obligations that are pure Go-mechanics and worth
teaching properly:

- **Flush per event.** Without `http.Flusher.Flush()`, the response buffers and every client sees
  one big blob at the end. TTFT becomes total latency and streaming is a lie.
- **Watch `r.Context().Done()`.** When a client disconnects mid-stream, the gateway must cancel the
  upstream call. Otherwise the provider keeps generating tokens **that we pay for and nobody reads**.
  That is a cost bug hiding inside a plumbing detail, and it is a specific fault we inject at
  `v1-07`.

---

# 1. State going in

From `v1-04`, committed:

- Gateway with SERVER/CLIENT spans, `traceparent` extraction, `CP_TELEMETRY=off` supported
- `cmd/collector` — OTLP receiver, **naive synchronous single-row writer**
- Postgres with the partitioned `spans` table; `cost_usd`, `pricing_version`, `pricing_status` filled
- `cmd/cpq` — `trace` and `cost` commands
- In-process mock adapter with configurable four-field usage
- Plan §15 rows for 01–04; ingest SLO re-baselined at `v1-03`

**Not yet existing:** `cmd/mockllm`, `cmd/loadgen`, `bench/`, `/metrics`, any streaming anywhere.

---

# 2. What this slice adds

```
  cmd/
    mockllm/main.go          <- standalone fake provider: latency dist, error/timeout/stall rates, SSE
    loadgen/main.go          <- open-loop generator, exact percentiles, JSON results
  internal/
    mockllm/
      profile.go             <- distribution + failure-rate config, seeded RNG
      sse.go                 <- streamed token emission
    provider/
      provider.go            <- + streaming method on the interface
      mockhttp/              <- adapter that talks to cmd/mockllm over HTTP
    gateway/
      stream.go              <- SSE passthrough, Flush per event, client-disconnect cancellation
    metrics/
      metrics.go             <- counters + histograms, /metrics in Prometheus text format
  bench/
    scenarios/*.yaml         <- committed
    results/*.json           <- committed, with machine state and commit SHA
    README.md                <- how to reproduce a run
```

Decisions, with recommendations:

- **`mockllm` is a separate process, not an in-process fake.** The in-process mock from `v1-01`
  stays (it is what unit tests use). The new one exists so there is a real socket, real HTTP, real
  serialisation and real latency on the provider leg — which is the shape `v1-06` will have with
  Anthropic. Measuring against an in-process fake would flatter the gateway and invalidate the
  comparison the very next slice makes.
- **`mockllm` must be seeded and deterministic.** `--seed` produces the same latency draws for the
  same scenario. A benchmark you cannot reproduce is an anecdote.
- **`mockllm` gets a runtime control endpoint** (`POST /_control`) to change error rate, latency and
  stall behaviour **without a restart**. Build it here; it is what makes `v1-07` fault injection
  possible mid-run, and mid-run is the only interesting time to inject a fault.
- **Extending the `Provider` interface for streaming** — this is the moment foreshadowed in `v1-01`,
  where an interface grows in front of a real requirement instead of being divined. Recommended
  shape: a **callback**, e.g. `ChatStream(ctx, req, func(ev Event) error) (Usage, error)`, rather
  than returning a channel. Reason: a channel-returning API leaks a goroutine whenever the consumer
  stops reading early — which is exactly the client-disconnect case we care about — whereas a
  callback returning an error makes early termination explicit and synchronous. Name the channel
  alternative and why it was rejected; it is a good Go-specific interview answer.
- **`/metrics` in Prometheus text format** — plan §14 identified its absence as a gap and put it
  here, because this is the first slice that needs to read counters under load. Recommendation: use
  `prometheus/client_golang`. The exposition format's histogram semantics (cumulative buckets, `_sum`,
  `_count`, `+Inf`) are fiddly enough that hand-rolling them teaches the wrong lesson, and the
  library is what every employer's stack actually runs. Hand-rolled counters from `sync/atomic` are
  a legitimate zero-dependency alternative for the five counters we have — say which you chose.
  - Minimum counters: `cp_requests_total{route,status}`, `cp_spans_received_total`,
    `cp_spans_written_total`, **`cp_spans_dropped_total`** (will read 0 until `v1-07` — that is the
    point), `cp_pricing_unknown_model_total`.
  - Minimum histograms: gateway request duration, provider call duration, span insert duration.
  - **Measure the cost of the metrics themselves.** Instrumentation on the hot path is not free, and
    a control plane that cannot quantify its own overhead is not making its own argument. It should
    be ~100ns per observation; confirm rather than assume.
- **Scenario files are committed; result files are committed** (plan §7, deliverable 2: *"with the
  commit SHA and machine state beside every number"*).

---

# 3. Why now

- **Because everything after this is a comparison.** `v1-06` compares real provider latency to
  mock. `v1-07` compares error rates before and after a fault and before and after a fix. Neither
  sentence is sayable without a baseline, and a baseline retrofitted after an optimisation is not a
  baseline (plan §4a, AGENT_INSTRUCTIONS: *"a step that changes performance without a before-number
  is a step done wrong"*).
- **Because three of plan §7's five SLOs are flagged in §14 as guesses.** Added latency ≤ 5 ms p99
  and telemetry overhead ≤ 2 ms p99 have never been tested against anything. They get a verdict
  here, and a miss is a finding.
- **Because `v1-06` costs money and this makes it cheap.** With `mockllm` shaped like the real
  provider, the Anthropic slice needs only enough real calls to validate token counts and cost
  accuracy (~200 requests, ~₹42 per plan §10) instead of a full load test against a paid API.
- **Because `v1-07` needs a fault injector and a way to see the fault.** `mockllm`'s control
  endpoint and `/metrics` are both prerequisites for the break-it slice; building them there would
  make that session twice as long and mix "build the tool" with "run the experiment".
- **Because the SLO must be tested while it is still possible to fail it honestly.** Test it after
  tuning and you have described what happened, not measured against a target.

---

# 4. Done means

- [ ] `cmd/mockllm` serves unary and SSE responses; supports constant and lognormal latency, plus
      configurable error rate, timeout rate and TTFT/inter-token delay
- [ ] `mockllm` returns configurable four-field usage so cost stays meaningful under load
- [ ] `mockllm --seed N` produces reproducible latency draws
- [ ] `POST /_control` changes knobs at runtime, verified while a run is in flight
- [ ] `Provider` interface has a streaming method; both mock adapters implement it; the compiler
      caught every implementation that did not
- [ ] the gateway streams SSE with a `Flush()` per event — verified by observing the first byte
      arrive well before the last
- [ ] a client disconnecting mid-stream cancels the upstream call — verified by a log line or a
      counter on the `mockllm` side showing generation stopped
- [ ] streaming requests produce a span with a TTFT attribute and correct total duration
- [ ] `cmd/loadgen` is **open-loop**, takes target RPS + duration + warmup, and **reports achieved
      rate vs target**, failing loudly on drift
- [ ] `loadgen` can target the gateway or `mockllm` directly (same scenario, both modes)
- [ ] `loadgen` writes `bench/results/*.json` including p50/p95/p99/p99.9, TTFT percentiles, error
      counts by class, achieved rate, warmup excluded, **commit SHA and machine state**
- [ ] `GET /metrics` on both gateway and collector returns valid Prometheus text
- [ ] `bench/scenarios/` has at least: `baseline-unary`, `baseline-stream`, `added-latency-ab`,
      `telemetry-ab`, `ingest-ramp`
- [ ] `bench/README.md` explains how to reproduce a run from a clean checkout
- [ ] **every SLO in plan §7 has a measured verdict — hit or missed — written into the plan file**
- [ ] plan §15 has a `v1-05` row

---

# 5. What to measure

This is the baseline. Six results, all committed to `bench/results/`.

1. **Gateway added latency — the headline SLO.**
   Same scenario, two targets: `loadgen → gateway → mockllm` vs `loadgen → mockllm` direct.
   Report p50/p95/p99 of the difference at **50, 100, 200 and 400 RPS**, with `mockllm` on constant
   latency so the subtraction is unambiguous.
   Plan §7 SLO: **p99 ≤ 5 ms at 200 RPS**. Plan §14 predicts this *"might land at 1 ms; might land
   at 15"* and names JSON re-marshalling on both legs as the suspect. Record the verdict; if it
   misses, profile before changing anything — `go test -bench` with `pprof`, or `net/http/pprof` on
   the running gateway, and find out where the milliseconds are rather than guessing.

2. **Telemetry emission overhead.**
   Identical runs with `CP_TELEMETRY=on` and `off`. Plan §7 SLO: **p99 ≤ 2 ms added**.
   Plan §14 flags this as inherited rather than derived, *"not plausible if export is synchronous"* —
   it should be async since `v1-02`, so a miss here points straight at the batch processor config or
   at the network hop plan §14 already identified as the first place to look.

3. **TTFT under streaming.** p50 and p95, gateway vs direct. This is the number a user feels, and it
   is the one `v1-06` will compare against a real model.

4. **Ingest capacity, properly this time.** `v1-03` measured it with a crude driver; measure it under
   real gateway-generated traffic. Report spans/sec sustained, `cp_spans_received_total` vs
   `cp_spans_written_total` vs rows actually in Postgres — **three numbers that should agree, and the
   interesting result is if they do not.**

5. **The knee.** Ramp RPS until p99 departs from flat. Report the rate where it turns and what
   saturates first (gateway CPU, collector, Postgres, or the loadgen itself — and if it is the
   loadgen, say so and fix it, because that invalidates the run).

6. **The cost of self-instrumentation.** Per-observation overhead of the metrics path, measured with
   a micro-benchmark. Small number, big credibility.

**Then update plan §7's table with a verdict column.** Every row gets `hit` or `missed`, with the
measured value beside the target. Plan §14: *"A missed SLO that was stated in advance is a finding
worth writing up. A missed SLO quietly edited afterwards is the thing that makes a portfolio
worthless."* Do not touch the targets.

**This is also the moment AGENT_INSTRUCTIONS says to write up.** Numbers are on screen, details are
fresh, and the graph exists. Draft the post now, not at `v1-08`.

---

# 6. Out of scope

- **Optimising anything.** This slice *finds* numbers. If added latency misses by 3×, record it,
  profile it, note the hypothesis — and fix it in `v1-07` or v2, with the before-number preserved.
  Optimising inside the measuring slice destroys the baseline it exists to create.
- **Fault injection runs.** The *capability* is built here (`/_control`, error and stall rates); the
  *experiments* are `v1-07`. Do not start killing Postgres in this session.
- **The bounded queue / drop policy.** Still deliberately absent. `cp_spans_dropped_total` should
  read 0 today; that zero is what makes `v1-07`'s number meaningful.
- **Real provider calls.** `v1-06`. ₹0 this slice.
- **Grafana, dashboards, plotting infrastructure.** Prometheus text out of `/metrics` and JSON out of
  `loadgen` is enough. If a graph is needed for the write-up, generate it from the JSON with
  whatever is at hand and keep the script in `bench/`.
- **Distributed load generation, multiple machines.** One laptop, honestly caveated (Concepts §7).
- **Sampling.** Still `AlwaysSample`. The wall it would relieve is what we are measuring.

---

# 7. The split point

**`v1-05a` — the fake provider is real.** `cmd/mockllm` with distributions, failure knobs, seeding,
SSE, and `/_control`; the `Provider` streaming method; gateway SSE passthrough with flush and
cancellation. Done when a streamed request through the gateway shows tokens arriving incrementally
and a disconnect stops upstream generation.

**`v1-05b` — the harness and the baseline.** `cmd/loadgen`, `/metrics`, `bench/` scenarios, the six
measurements, the SLO verdicts, the write-up draft.

Splitting here is clean: 05a produces a runnable, committable capability, and 05b starts by pointing
a new tool at it. If 05a runs long — streaming passthrough with correct cancellation is the most
finicky Go in the whole project — stop, commit, and start fresh. A tired session writing a benchmark
is exactly how a benchmark ends up wrong.

---

# Commit plan

1. `feat(mockllm): latency distributions, failure rates, seeded rng`
2. `feat(mockllm): sse streaming with configurable ttft and inter-token delay`
3. `feat(mockllm): runtime control endpoint`
4. `feat(provider): streaming method on the Provider interface`
5. `feat(gateway): sse passthrough with flush and client-disconnect cancellation`
6. `feat(metrics): prometheus endpoint on gateway and collector`
7. `feat(loadgen): open-loop generator with exact percentiles`
8. `feat(bench): scenarios and reproducible result format`
9. `docs(bench): v1 baseline results and SLO verdicts`

---

# Hand-off — what to record in the plan file

- §15 row for `v1-05`: commit SHA, added-latency p99 @200 RPS, telemetry overhead p99, TTFT p95,
  sustained spans/sec, the knee, and the surprise.
- **§7 SLO table: a verdict on every row.** Targets unchanged.
- Deliverable tracker: tick **1 (harness exists)** and **2 (baseline measured)** — note that "then
  beaten" is still open and belongs to `v1-07`/v2.
- §14: for each low-confidence claim, replace the prediction with the measurement and keep both
  visible.
- Anything the baseline revealed that changes the shape of `v1-07` — if something is already
  unstable at 400 RPS, that is the fault to chase first.

---

# Next

`v1-06` — the real provider. The Anthropic adapter, real streaming, real tokens, real dollars
(~₹42), and the honest test of whether the `Provider` interface actually holds against a second
genuinely different shape.

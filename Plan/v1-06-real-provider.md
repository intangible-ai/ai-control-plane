# Slice v1-06 — The real provider

**Version:** v1 · **Slice:** 06 of 08 · **Produces a number?** Yes — and it is the only slice in v1 that spends money.

> **Build-sequence note (added after the portfolio's `LATEST EDIT` revision).** This project is
> **position 0 — `#30-thin`, the control plane spine** — in the only build sequence now in force.
> That section supersedes Parts 10, 11 and 12 of the portfolio file. Position 0's scope is stated
> there as: *"OTel GenAI spans, token and cost accounting, a trace store, and the provider adapter
> interface... The telemetry half of backpressure lives here too — a bounded queue that drops spans
> before user traffic is ever dropped."* Two consequences run through every slice below:
> **(a)** the bounded span queue is now **required scope**, not a discovered fix; and
> **(b)** what comes next is **position 1, `#7` (prefix-cache-aware context assembler)**, which is
> where the *reusable* mock-LLM server and load generator belong. See `v1-05` §0.

> Read `..\..\Shared Context\control_plane_thin_plan.md` §4a (the Bedrock position), §5 (pricing),
> §7 (cost-accuracy SLO) and §10 (budget) before answering anything.
>
> **Budget: ~$0.50 ≈ ₹42.** Hard ceiling for the whole project is ₹2,000 (CLAUDE.md §6). The
> arithmetic is in §5 and it gets said out loud before the first paid call.

---

# Concepts for this slice

Plan §8 does not assign a concept to this slice, so it takes the one the slice is actually about.

### 1. The second implementation is what tests an abstraction — the first only shapes it

An interface written against one implementation is a description of that implementation with the
names changed. It cannot be wrong yet, because nothing has disagreed with it. The second
implementation is the experiment.

This is why plan §4a killed the stubbed Bedrock adapter, and the reasoning is worth reading again
because it is a model of how to argue about design: *"a stub that makes no network call, has no
auth, and returns canned bytes proves nothing about whether the abstraction survives contact with a
second real provider."* A stub cannot disagree with you. It is built from the same assumptions as
the interface.

The mock and Anthropic adapters, by contrast, are genuinely different shapes:

| | `mockllm` | Anthropic |
|---|---|---|
| Transport | local plaintext HTTP | remote TLS |
| Auth | none | `x-api-key` header + `anthropic-version` |
| Failure modes | whatever we injected | 400/401/403/404/413/429/500/529, network, mid-stream |
| Streaming | our own SSE shape | six distinct event types with a defined order |
| Token accounting | one usage object at the end | **split across two different events** |
| Latency | drawn from a distribution we chose | whatever it is |
| Rate limits | none | RPM, TPM, TPD, with headers |

**The honest question of this session is: does the `Provider` interface hold, or does it have to
change?** Both answers are good. If it holds, that is evidence the boundary was drawn in the right
place. If it has to change, *the shape of the change is the finding* — write down exactly what the
second provider demanded that the first never did, because that is the extracted-not-designed
lesson from plan §1 happening in miniature, in one slice, at low cost.

What to watch for as pressure on the interface: streaming signatures, provider-specific request
parameters with no neutral equivalent, error taxonomy, retry semantics, and the fact that one
adapter needs credentials and the other does not.

### 2. The Anthropic streaming event sequence, and where the tokens actually are

A streamed response is a sequence of SSE events in a defined order:

```
event: message_start        <- message metadata AND input-side usage
event: content_block_start
event: content_block_delta  <- repeated, one per chunk of text
event: content_block_stop
event: message_delta        <- stop_reason AND output-side usage
event: message_stop
```

**The token accounting gotcha, and it is the one that quietly breaks cost engines:** usage does not
arrive in one place at the end.

- `message_start` carries the **input** side — `input_tokens`, `cache_creation_input_tokens`,
  `cache_read_input_tokens`.
- `message_delta` carries the **output** side — `output_tokens` — along with `stop_reason`.

So computing the cost of a streamed request means **accumulating across events**, and holding
partial state for the whole duration of the stream. Three consequences that matter to this project:

1. The span cannot be finalised until `message_delta` has been seen. A span closed at first byte, or
   at `content_block_stop`, has input tokens and no output tokens — and our cost engine will price
   it, confidently, wrongly.
2. **If a stream aborts mid-flight, we have input tokens and no output count.** That is not an
   exotic case — it is `v1-07`'s stall-mid-stream fault, and it is a real production shape. The
   right handling is to record what we know, set `pricing_status = 'partial_stream'` (a new value in
   the closed set from `v1-04`), and never silently emit a cost that pretends the output was free.
   Decide this here; `v1-07` will exercise it.
3. `message_start` arriving is what makes TTFT measurable at all on the real provider — and note
   that it arrives *before* any text, so TTFT-to-first-*token* and TTFT-to-first-*event* are two
   different numbers. Pick one, define it in the README, and be consistent with what `loadgen`
   measured against `mockllm` at `v1-05` or the comparison is meaningless.

Also present in a real stream: `ping` events (keepalives — ignore them, but do not choke on them)
and `error` events (an error can arrive **after** a 200 OK, mid-stream, which is a shape that
surprises people who only handle status codes).

### 3. Errors are a taxonomy, not a status code

`v1-02` established a closed set of `error.type` values. Here it meets reality:

| HTTP | Type | Retryable | Our `error.type` |
|---|---|---|---|
| 400 | `invalid_request_error` | No | `bad_request` |
| 401 | `authentication_error` | No | `auth` |
| 403 | `permission_error` | No | `auth` |
| 404 | `not_found_error` | No | `bad_request` |
| 413 | `request_too_large` | No | `bad_request` |
| 429 | `rate_limit_error` | **Yes** | `rate_limit` |
| 500 | `api_error` | **Yes** | `provider_5xx` |
| 529 | `overloaded_error` | **Yes** | `provider_overloaded` |

The mapping is the point. A control plane that stores the provider's raw error strings has unbounded
cardinality and cannot answer "what fraction of failures were retryable?" — which is the only
question anyone actually asks during an incident. Mapping to a closed set is what makes that a
`GROUP BY`.

**Note that the closed set grew.** `v1-02` guessed at `timeout`, `rate_limit`, `provider_5xx`,
`bad_request`, `stream_aborted`; contact with a real provider adds `auth` and
`provider_overloaded`. That is not a mistake in `v1-02` — it is the same extracted-not-designed
point as Concepts §1, at the scale of an enum. Update the constants in `pkg/genai` and say in the
commit message *why* the set grew; a taxonomy that never changes on meeting reality was probably
never checked against it.

On 429, the response carries `retry-after` (seconds) plus `x-ratelimit-limit-*` and
`x-ratelimit-remaining-*` headers. **Honour `retry-after` rather than your own backoff curve** — the
server knows when it will let you back in, and your exponential guess does not.

### 4. A retry must be a child span, not a hidden loop

If the adapter retries a 429 internally and reports one span covering both attempts, then:

- latency attribution is a lie (the span includes a sleep),
- the retry is invisible, so nobody can see that 30% of traffic is being retried, and
- the cost engine may double-count or under-count depending on which usage object won.

**Each attempt gets its own child span**, with an attempt number and its own `error.type`, under one
logical parent. This costs a few lines and it is the difference between telemetry and decoration.

It also matters beyond this repo: plan §11 makes per-provider latency the input to **#35**, the
router. A router fed latency numbers that silently include retry sleep will route badly, and the bug
will be invisible.

### 5. Retries amplify outages, so bound them here

Three rules, applied in the adapter:

- **Cap attempts** (2–3 total). Unbounded retry converts a provider blip into a self-inflicted DDoS.
- **Jitter the backoff.** Without jitter, every client that failed at the same instant retries at
  the same instant, forever. This is the retry-storm shape, and it is `v1-07` material.
- **Never retry a non-retryable class.** Retrying a 400 three times is three times the latency and
  the same failure.

Note that the SDKs retry 429/5xx automatically by default. **We are not using an SDK on this path** —
the gateway is a proxy, and a raw HTTP client is the honest shape for it — but if a session reaches
for one, know that its built-in retries would sit *inside* our span and hide exactly what Concepts
§4 says must be visible.

### 6. Prompt caching, made real

`v1-04` built a cache-aware cost engine against mock numbers. Here it meets the actual mechanism.

- A breakpoint is `"cache_control": {"type": "ephemeral"}` on a content block — 5-minute TTL by
  default, `{"type": "ephemeral", "ttl": "1h"}` for the hour.
- **Max 4 breakpoints per request.**
- Render order is `tools` → `system` → `messages`, so a breakpoint on the last system block caches
  tools and system together. Stable content before the breakpoint, volatile content after it — which
  is `v1-00` §2.2's reordering lesson expressed as an API parameter.
- **Minimum cacheable prefix on `claude-haiku-4-5` is 4,096 tokens** (`v1-04` Concepts §4). Below
  that, the breakpoint is accepted and ignored, silently. **The test prompt for this slice must
  clear 4,096 tokens** or the cache experiment measures nothing and looks like a bug in our engine.
- Verify with `usage.cache_read_input_tokens` on the second identical request. If it is zero, there
  is a silent invalidator — a timestamp, a non-deterministic JSON key order, a changed tool list.

**This is the first honest end-to-end check of the four-field cost model**, because these are real
numbers from a real provider rather than values we made up in the mock.

### 7. Secrets: the rule and the reason

The API key exists in this process for the first time. Three places it must never appear: a span, a
log line, an error message. Plan §6's security row is *"provider keys never leave the gateway
process"* and *"no credential in any span."*

The failure mode is not dramatic, it is boring: someone logs the outbound request for debugging,
including headers, and the key is now in Postgres, in a trace viewer, and in any screenshot of
either. Structural defences, in order of strength:

1. Never log a request object that contains headers. Log the fields you meant to log.
2. Read the key from the environment at startup into a struct field, never pass it through a request
   context or a map that gets serialised.
3. `.env` in `.gitignore`, and `git check-ignore` verified, before the key is created.
4. `pkg/genai` has no field for headers or credentials — same structural argument as bodies in
   `v1-02` Concepts §6.

And the operational one: **set a spend limit in the Anthropic console before creating the key**, not
after.

---

# 1. State going in

From `v1-05` (or `05a` + `05b`), committed:

- `cmd/mockllm` — distributions, failure knobs, seeded, SSE, `/_control`
- `cmd/loadgen` — open-loop, exact percentiles, `bench/results/*.json`
- `Provider` interface with a streaming method; gateway SSE passthrough with flush + cancellation
- `/metrics` on gateway and collector
- Cost engine with dated pricing config, four-field arithmetic
- **The v1 baseline exists**, and every SLO in plan §7 has a hit/miss verdict

**Preconditions before any paid call — all four, in order:**

1. Anthropic console: **spend limit set** on the workspace/key.
2. `.env` in `.gitignore`, verified with `git check-ignore -v .env`.
3. A **hard request cap** in the bench scenario used for this slice, so a bug cannot loop.
4. The budget arithmetic in §5 said out loud and agreed.

---

# 2. What this slice adds

```
  internal/provider/anthropic/
    anthropic.go       <- Chat + ChatStream against POST /v1/messages
    sse.go             <- event parsing; usage accumulated across message_start + message_delta
    errors.go          <- HTTP status -> our closed error.type set
    retry.go           <- bounded, jittered, retry-after aware, child span per attempt
  cmd/gateway/main.go  <- CP_PROVIDER=anthropic wired in the composition root
  bench/scenarios/
    real-provider-sample.yaml   <- hard-capped request count
    cache-verification.yaml     <- >4096-token stable prefix, run twice
  .env.example         <- documents the variables, contains no values
```

Notes for the session:

- **Raw `net/http`, not the SDK.** Consistent with plan §4's reasoning for the gateway: we are a
  proxy, we want control of the bytes and of exactly what sits inside our latency measurement, and
  an SDK's built-in retry would hide what Concepts §4 requires to be visible. Say this out loud —
  it is a real decision, and "why didn't you use the official SDK?" is a fair interview question
  with a good answer here.
- Required headers: `x-api-key`, `anthropic-version: 2023-06-01`, `content-type: application/json`.
- **Model: `claude-haiku-4-5`** for the paid runs (plan §10's budget is computed on it).
- **Timeouts on every call**, and a longer one for streaming than unary — a streaming request is
  legitimately slow, so a single global client timeout either kills valid streams or fails to
  protect unary calls. Use per-request context deadlines, not one client-wide setting.
- **Translation is where the abstraction earns its keep.** Our neutral `ChatRequest` becomes an
  Anthropic Messages request; their four usage fields become our `Usage`. Note anything neutral we
  cannot express and anything of theirs we have to drop — that list is the honest limitation section
  of the README.

---

# 3. Why now

- **Because the baseline exists to compare against.** Real provider latency is only interesting next
  to a known-good number. Running this before `v1-05` would produce a figure with no context.
- **Because it validates the cost engine against ground truth.** Everything before this priced
  numbers we invented. Plan §7's cost-accuracy SLO — *"token counts exact; USD within rounding of
  provider-reported"* — cannot be tested against a mock, by construction.
- **Because the interface question should be asked while it is still cheap to answer.** Four slices
  in, one gateway, no dependent projects. By position 13, with five modules and three system shapes
  onboarded, the same discovery is a migration across seven dependents — which is exactly why the new
  sequence refuses a big-bang rewrite there and extracts only the policy pipeline.
- **Because `v1-07` needs a real error taxonomy to break.** Injecting faults is more convincing when
  the error classes came from a real provider's documented behaviour rather than from our own mock.
- **Because it is cheap now and gets more expensive later.** ~₹42. Done after more of the system
  exists, the same validation costs more calls.

---

# 4. Done means

- [ ] spend limit set in the console **before** the key was created
- [ ] `.env` gitignored, verified; `.env.example` committed with no values
- [ ] `internal/provider/anthropic` implements the full `Provider` interface, unary and streaming
- [ ] **zero changes required in `internal/gateway`** to add it — or, if changes *were* required,
      they are written down as the finding (see §5.4)
- [ ] `CP_PROVIDER=anthropic` switches provider with no recompile of business logic
- [ ] streaming works end to end: `loadgen` or `curl` sees tokens arriving incrementally through the
      gateway from the real API
- [ ] usage is accumulated across `message_start` and `message_delta`; a streamed request records
      **all four** token fields correctly
- [ ] an aborted stream records partial usage with an explicit `pricing_status`, not a fabricated
      cost
- [ ] `ping` events are ignored without error; a mid-stream `error` event is handled
- [ ] every documented status code maps to our closed `error.type` set
- [ ] 429 honours `retry-after`; retries are bounded, jittered, and **each attempt is a child span**
- [ ] a deliberate 401 (bad key) produces a clean typed error, no retry, no panic
- [ ] **grep the codebase and the database: the API key appears in neither** — check span
      attributes, logs, and `attributes` jsonb
- [ ] the cache-verification scenario shows nonzero `cache_read_input_tokens` on the second run
- [ ] computed USD compared against provider-reported usage over the sample; delta recorded
- [ ] the cache-write TTL question left open at `v1-04` is **closed** with what the API actually
      reports
- [ ] total spend checked in the console and recorded against the ₹42 estimate
- [ ] plan §15 has a `v1-06` row

---

# 5. What to measure

**Budget arithmetic first, out loud, before the first paid call** (CLAUDE.md §6):

> 200 requests × ~1,000 input + ~300 output tokens on `claude-haiku-4-5` ($1.00 / $5.00 per MTok)
> = 200,000 input = **$0.20** · 60,000 output = **$0.30** → **$0.50 ≈ ₹42**.
> Cache-verification runs add a >4,096-token prefix ×2 ≈ 8,200 input tokens ≈ **$0.01**.
> Plan §10's contingency (re-running on `claude-sonnet-5`) is **+$1.35 ≈ ₹112** and is **not**
> authorised by default — ask first.

Then five numbers:

1. **Real vs mock latency.** Same scenario, `CP_PROVIDER=anthropic` vs `mock`. p50/p95/p99 and TTFT.
   Expect a completely different scale — hundreds of milliseconds to seconds, versus the mock's
   configured draw. **The interesting number is not the total, it is the gateway's added latency**,
   which should be roughly unchanged. If our overhead moved when the provider got slower, that is a
   finding about our own code (buffering, or a timer measuring the wrong thing).

2. **Cost accuracy — plan §7's SLO.** For ~200 requests, compare our computed `cost_usd` against the
   provider-reported usage for the same calls. Token counts must match **exactly** (they come from
   the same response — a mismatch means a parsing bug, not a rounding issue). USD must agree within
   rounding. Report the delta and the verdict.

3. **The cache path, for real.** Run the >4,096-token prefix twice within the TTL. Record
   `input_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens` on both, and the cost of
   each. Verify the second request is meaningfully cheaper and that our engine reports it. **This is
   the moment `v1-04`'s #7-rehearsal stops being a simulation.**

4. **Error taxonomy observed in the wild.** Over all real traffic in this slice, which error classes
   actually occurred, at what rate. Probably a short list — record it anyway, and note which ones
   `v1-07` will therefore have to inject synthetically because they never happened naturally.

5. **Actual spend vs the estimate.** Console figure against ₹42. If the estimate was off, say by how
   much and why — a cost-control project that cannot predict its own bill has a credibility problem,
   and the arithmetic error is worth finding.

**And the qualitative result, which may be the most valuable output of the slice:**

> **Did the `Provider` interface hold?**
> If yes: say what specifically survived — streaming shape, error handling, usage mapping.
> If no: write down exactly what changed, why the mock never demanded it, and whether the new shape
> is better or just accommodating. Plan §1's argument is that good abstractions are *extracted from
> working systems*, and this is the smallest possible instance of that. Record it in the plan file
> either way.

---

# 6. Out of scope

- **Bedrock.** Plan §4a is unambiguous: not in v1, not even a stub. The open question there — whether
  the ~9-month-old AWS account still exists and carries usable quota — is worth checking *outside* a
  session, and if it is alive, a Bedrock adapter becomes a cheap high-value slice later. **Do not
  create a new AWS account.**
- **Any other provider.** Two real adapters is the experiment. A third adds cost, not evidence.
- **Load testing against the real API.** ₹42 is 200 requests, not a benchmark run. `mockllm` exists
  precisely so throughput work costs nothing (plan §4). A real-provider load test would blow the
  budget and measure Anthropic's capacity, not ours.
- **`claude-sonnet-5` or `claude-opus-5` runs.** Not authorised by default; ask.
  *(Worth noting while pricing: `claude-sonnet-5` is currently at introductory rates below its
  standard $3/$15. A price that changes on a known date is the perfect argument for the dated
  pricing file built in `v1-04` — and a reason to check plan §5's table against the live pricing
  page during this session rather than trusting it.)*
- **Tool use, structured outputs, thinking, batch API, files.** None of them are on the control
  plane's path. v1 proxies chat completions and measures them.
- **Fault injection against the real provider.** Faults are injected into `mockllm`, for free, at
  `v1-07`.
- **The bounded queue.** Still absent. Still `v1-07`.
- **Optimising the gateway.** If added latency regressed, record it and fix it in `v1-07`, with the
  before-number intact.

---

# Commit plan

1. `chore: env example and gitignore for provider credentials`
2. `feat(provider/anthropic): unary chat against the messages api`
3. `feat(provider/anthropic): sse parsing with usage accumulated across events`
4. `feat(provider/anthropic): error taxonomy mapping`
5. `feat(provider/anthropic): bounded jittered retries as child spans`
6. `feat(bench): capped real-provider and cache-verification scenarios`
7. `docs: real vs mock latency, cost accuracy, and the interface verdict`

---

# Hand-off — what to record in the plan file

- §15 row for `v1-06`: commit SHA, real p95 and TTFT, **cost-accuracy delta**, cache read/write
  numbers, actual spend, the surprise.
- **§7 SLO table:** cost-accuracy row gets its verdict.
- **The interface verdict** — held, or changed and how. This is the entry a future session will
  actually want.
- **§12:** the cache-write TTL question, closed.
- **§10:** actual spend against the ₹42 estimate, and the running total against the ₹2,000 ceiling.
- **§5:** if the live pricing page disagreed with the plan's table, fix the table and say so.
- If the plan's §4a AWS question got answered (account alive or dead), record it — it decides
  whether Bedrock stays in the portfolio at all.

---

# Next

`v1-07` — break it. Kill Postgres under load, stall a stream at 90%, return garbage, spike 10×,
disconnect mid-stream. Find the failure, fix the worst one, re-measure. **Deliverable #3.**

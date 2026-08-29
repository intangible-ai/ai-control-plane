# Slice v1-02 — The wire contract

**Version:** v1 · **Slice:** 02 of 08 · **Produces a number?** Yes — bytes/span, and a first crude
telemetry-overhead reading.

> Read `..\..\Shared Context\control_plane_thin_plan.md` (§5 especially) before answering anything.
> **This slice ends with a decision** — plan §12.4, Go vs TypeScript for the gateway. Do not skip it.

---

# Concepts for this slice

Plan §8: *"`v1-02` opens with how distributed tracing actually reconstructs a call tree from flat
spans."* Start there, then work outward to the library that produces them.

### 1. The tree is an illusion reconstructed from flat rows

There is no tree on the wire. There is no tree in the database. Every span is an independent,
self-contained record that gets shipped whenever its own work finishes, and the tree exists only
because a query puts it back together.

Three fields do all the work: `trace_id` (same on every span in one request), `span_id` (unique per
span), `parent_span_id` (the `span_id` of whatever caused this one). `v1-00` §3.2 covered that much.
The consequences are what this slice needs, and they are non-obvious:

- **Spans arrive out of order, and the parent almost always arrives *after* its children.** A parent
  span cannot be exported until it ends, and it ends after everything nested inside it. So the
  writer must never require a parent row to exist before inserting a child. No foreign key from
  `parent_span_id` to `span_id` — ever. That constraint would look correct and would drop most of
  your data. (Remember this at `v1-03`, where the temptation to add the FK is real.)
- **A trace can be permanently incomplete.** If a process dies mid-request, its parent span is never
  exported and you have orphans. The query has to handle a forest, not a tree. This is why "trace
  completeness %" is a *metric* in plan §6 and not an assumption.
- **The tree is only as good as the propagation.** If a service starts a fresh `trace_id` instead of
  continuing the caller's, you get two disconnected traces that look fine individually and are
  useless together. This is the single most common tracing bug in the wild, and `v1-08` is where it
  would bite us in another language.

Why we care beyond aesthetics: plan §11 makes the trace DAG a hard constraint inherited from
**#30-S**, whose most valuable analysis is critical-path (`v1-00` §4.4). Critical path is computed
by walking parent links. A broken `parent_span_id` does not degrade that analysis — it deletes it.

### 2. Context propagation and the `traceparent` header

Inside one process, the parent link is carried by `context.Context`: `tracer.Start(ctx, name)` looks
in `ctx` for a current span and makes the new span its child. This is the real reason `ctx` is
threaded through every function in Go code, and it is the answer to "why does every signature start
with `ctx`" from `v1-00b` §2.8.

Across a process boundary, the same information travels as an HTTP header, standardised by W3C:

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             ^^ version  ^^ trace-id (16 bytes hex)  ^^ parent-id  ^^ flags (sampled)
```

The gateway must **extract** that header into a context if the caller sent one, and start a fresh
root trace if they did not. Twelve lines with the OTel propagator; catastrophic if skipped.

This is exactly how the two-door architecture becomes one trace instead of two: at `v1-08`, a Python
agent creates a span, calls Door B with the header, and the gateway's span lands as its child. Same
`trace_id`, two languages, no shared library. **That is the vendor-neutral claim made real**, and
this slice is where the mechanism gets built.

### 3. Three things, one word: OTel SDK vs OTLP vs semconv

`v1-00` §3.3 defined them. Here they become three separate lines of code, and the separation is the
architecture:

| Layer | What it is here | Swappable? |
|---|---|---|
| **Semantic conventions** | attribute *names* — `gen_ai.usage.input_tokens` | this is the contract; changing it is a breaking change |
| **SDK** | `go.opentelemetry.io/otel` + `otel/sdk/trace` — creates, batches, samples | yes, per language; a tenant uses their own |
| **OTLP** | protobuf over HTTP on `:4318/v1/traces` — the bytes | yes, gRPC is the alternative (plan §12.3 defers it) |

**We ship no SDK.** Tenants use the stock one for their language. What we publish is the vocabulary
and the endpoint.

### 4. Anatomy of the SDK: provider, resource, processor, exporter, sampler

Five objects, assembled once in `main`. Worth teaching individually because four of them have a
failure mode that shows up later in this project:

- **TracerProvider** — the factory. One per process, built at startup, `Shutdown()` on exit.
  *Failure mode:* forgetting `Shutdown()` means the last batch is never flushed, so short-lived
  processes appear to emit nothing. This will bite at `v1-08` with a 50-line Python script that
  exits immediately.
- **Resource** — attributes describing *the process*, not the request: `service.name`,
  `service.version`, `deployment.environment`. Sent once per export batch, not per span. Getting
  `service.name` wrong makes every dashboard in the future wrong.
- **SpanProcessor** — what happens when a span ends. `SimpleSpanProcessor` exports synchronously,
  on the calling goroutine, one span at a time. `BatchSpanProcessor` queues and exports in the
  background. **Use Batch.** Simple would put a network round-trip on the request path and blow the
  2 ms overhead SLO by an order of magnitude.
- **Exporter** — `otlptracehttp`, pointed at the collector.
- **Sampler** — `AlwaysSample()` in v1, deliberately. Plan §2: *"v1 keeps everything and measures
  where that breaks."* Sampling before there is a wall would be optimising with no baseline.

**Plant a flag here for `v1-07`:** `BatchSpanProcessor` has a bounded queue (default 2048 spans) and
it **drops silently when full**. It does not return an error, and by default nothing counts the
drops. That is a real, in-production observability blind spot sitting inside our own service, and
`v1-07` is where we go and find it deliberately. Note it now; do not fix it now.

### 5. Span kinds and status, and why `error.type` is not optional

- `SpanKindServer` for the inbound gateway span; `SpanKindClient` for the outbound provider call.
  This is not cosmetic — it is how any backend, ours included, tells "I received a request" from
  "I made a request", which is what makes an RED-style breakdown possible without string matching
  on span names.
- Status is `Ok` / `Error` / `Unset`. Set `Error` **and** the `error.type` attribute. Plan §6's
  observability row depends on always-keep-errors, and an always-keep rule needs something
  machine-readable to key off. A message string is not that.
- `error.type` should be a small, closed set of our own values (`timeout`, `rate_limit`,
  `provider_5xx`, `bad_request`, `stream_aborted`), not the provider's raw error string. Raw
  strings are unbounded cardinality and turn a `GROUP BY` into a mess.

### 6. Attribute cardinality, and the one security rule of this slice

Two rules that will save the store later:

1. **Never put a per-request unique value in an attribute you intend to group by.** `tenant_id` is
   fine (dozens). `route` is fine (dozens). A user ID or a request ID is not — it belongs in a
   dedicated field, not a dimension. This is the cardinality lesson that plan §5 encodes as
   "promote hot dimensions to columns, keep the long tail in jsonb".
2. **Do not capture prompts or completions.** Plan §6's security row: bodies are **not** captured by
   default, opt-in per route. Enforce it structurally — the span-building code in `pkg/genai` simply
   has no field for message content, so capturing it would require adding one, which is a visible
   diff and a conversation. A `if !config.CaptureBodies` check is a flag someone flips at 2am; an
   absent field is a design.

The same rule covers credentials: an API key must never reach a span, a log line, or an error
message. At `v1-06` there is a real key in the process. The habit is built now.

### 7. Semconv is a moving target, and pretending otherwise is the trap

The OTel GenAI semantic conventions are **not stable** — they are under active development and
attribute names have changed between releases (and will again). This is the single biggest risk to
the "vendor-neutral contract" claim, so handle it explicitly:

- Pin the semconv version in `pkg/genai` as a constant, and record which version the repo targets in
  the README and in the span's resource attributes.
- Define every attribute name as a Go constant in one file. Never write the string literal twice —
  a rename must be a one-line diff.
- Write down, in the package doc comment, where the spec lives and the date it was read.

The honest framing for an interview: *"I adopted an unstable spec on purpose, because the
alternative is inventing a proprietary schema, and I made the churn cheap by putting every name
behind one constants file."*

---

# 1. State going in

From `v1-01`, committed and pushed:

- Repo `intangible-ai/ai-control-plane`, module `github.com/intangible-ai/ai-control-plane`
- `internal/provider/provider.go` — `Provider` interface, neutral `ChatRequest`/`ChatResponse`/`Usage`
  (four token fields)
- `internal/provider/mock/mock.go` — deterministic in-process adapter
- `internal/gateway/handler.go` — `POST /v1/chat`, dispatching through the interface
- `internal/telemetry/span.go` — hand-rolled span struct, printed to stdout as JSON
- `cmd/gateway/main.go` — composition root, `CP_PROVIDER` from env
- A crude `added_ms` figure in plan §15

If plan §15 has no `v1-01` row, that session did not finish. Check before starting.

**Docker is needed this slice** — earlier than plan §9 implies. Start Docker Desktop and confirm
`docker version` responds before writing code. It is used only to run the reference collector; no
Postgres yet.

---

# 2. What this slice adds

The hand-rolled span is deleted and replaced by real OpenTelemetry, and the contract moves out of
`internal/` into a package the rest of the portfolio can import.

```
  pkg/
    genai/
      doc.go            <- package doc: what this is, semconv version, spec URL, date read
      semconv.go        <- every attribute name as a constant; the ONLY place the strings appear
      usage.go          <- Usage: the four token fields + helper for true prompt size
      chat.go           <- Door B request/response DTOs (moved out of internal/provider)
      errors.go         <- the closed set of error.type values
  internal/
    telemetry/
      tracer.go         <- TracerProvider / resource / batch processor / OTLP exporter setup
      attrs.go          <- ChatRequest+ChatResponse -> []attribute.KeyValue, using pkg/genai
    gateway/
      handler.go        <- extract traceparent; SERVER span; CLIENT child span around Chat()
  deploy/
    otel-collector.yml  <- reference collector config (otlp receiver -> debug exporter)
    docker-compose.yml  <- one service for now; Postgres joins it at v1-03
```

Decisions to make in the session, with the recommendation stated:

- **One public package or two?** Recommendation: **one — `pkg/genai`** — holding both the telemetry
  vocabulary and the Door B DTOs. Both are schema, both are things external code needs, and
  splitting a schema-only package that has never hurt anyone is speculative. Name the counter-case
  (a module that only wants the telemetry contract is forced to see chat DTOs; the cost is zero at
  compile time) and note that splitting later is mechanical because nothing has behaviour.
- **`pkg/genai` must import nothing from `internal/`.** Enforce it, do not just intend it: a tiny
  test that walks the package's imports, or `go list -deps ./pkg/genai` grepped for `internal`, run
  in the build. Plan §3 calls this out as structural discipline; make the structure do the work.
- **Export target.** Run the official `otel/opentelemetry-collector-contrib` container with an OTLP
  receiver on `:4318` and a `debug` exporter with `verbosity: detailed`. Our own collector does not
  exist until `v1-03`. This is not a placeholder — it means we validate our OTLP output against the
  **reference implementation** first, so that when our receiver misbehaves at `v1-03` we already
  know the emitting side is correct. Cheap, and it removes a whole class of "which end is broken"
  debugging.
- **Keep `CP_TELEMETRY=off`** as a supported startup mode from this slice on. It is how the
  emission-overhead SLO gets A/B'd at `v1-05` — same binary, same run shape, one flag. Building it
  now costs three lines; retrofitting it means re-running every benchmark.

Span shape produced per request:

```
gateway.chat            SpanKindServer   <- root (or child of the caller's traceparent)
  └─ provider.chat      SpanKindClient   <- carries gen_ai.* attributes and usage
```

Put the `gen_ai.*` attributes on the **client** span, where the model call actually happened. The
server span carries `cp.tenant_id`, `cp.route`, `cp.prompt_version` and HTTP attributes. The
subtraction from `v1-01` — added latency — is now `parent.duration - child.duration`, computable in
SQL at `v1-03`, which is the entire point.

---

# 3. Why now

- **Because `v1-03` writes whatever this slice emits.** Build the store first and the schema gets
  designed around a hand-rolled struct that is about to be thrown away. Contract, then storage.
- **Because the contract is the product.** Plan §1: this project is the chassis every later project
  bolts onto. `pkg/genai` is the bolt pattern. It is the one artefact of v1 that other repos
  literally import (CLAUDE.md §9), so it should exist before four projects have improvised around
  its absence.
- **Because propagation is cheap now and archaeology later.** Adding `traceparent` extraction to a
  handler that already has span plumbing is ten minutes. Discovering at `v1-08` that cross-language
  traces do not join, then reworking the handler, the schema and the query path, is a slice of its
  own.
- **Because the decision in plan §12.4 is scheduled here.** After this slice there is roughly 300
  lines of Go on disk: an HTTP server, an interface, an adapter, span construction, and OTLP export.
  *(Plan §12.4 and `v1-00b` §6 both say "two adapters" at this point — that is off by one. The
  second real adapter is `v1-06`. The line count is about right regardless; correct the plan file
  in passing.)* That is enough evidence to judge whether Go friction is fading. Deciding earlier is
  deciding from fear; deciding later means the sunk cost decides for you.

---

# 4. Done means

- [ ] `pkg/genai` exists; `go list -deps ./pkg/genai` shows **no** package under `internal/`, and
      that check is a test or a build step, not a habit
- [ ] every OTel attribute name in the repo comes from a constant in `pkg/genai/semconv.go`; a grep
      for `"gen_ai.` outside that file returns nothing
- [ ] `pkg/genai` has **no field anywhere** for prompt or completion text
- [ ] the targeted semconv version is written in `doc.go` and in the README
- [ ] `internal/telemetry/tracer.go` builds a TracerProvider with a Resource (`service.name`),
      `BatchSpanProcessor`, `otlptracehttp` exporter and `AlwaysSample`
- [ ] the process calls `TracerProvider.Shutdown()` on SIGINT and the last spans arrive
- [ ] `docker compose up -d otel-collector` runs the reference collector; its logs show our spans
      with readable `gen_ai.*` attributes
- [ ] one request produces **two** spans sharing a `trace_id`, with the client span's
      `parent_span_id` equal to the server span's `span_id`
- [ ] a request sent **with** a `traceparent` header produces spans on **that** trace id — verified
      by sending a handcrafted header from `api/chat.http` and finding it in the collector output
- [ ] a request sent **without** one starts a fresh root trace
- [ ] a forced provider error produces `status = Error` and an `error.type` from the closed set
- [ ] `CP_TELEMETRY=off` starts cleanly and emits nothing
- [ ] `go build ./... && go vet ./...` clean; `internal/gateway` still contains no reference to `mock`
- [ ] the hand-rolled span code from `v1-01` is **deleted**, not left beside the new path
- [ ] plan §15 has a `v1-02` row; plan §12.4 is resolved and struck through with the reason recorded

---

# 5. What to measure

Two real numbers and one judgement.

1. **Bytes per span on the wire.** Read the collector container's received-bytes counter, or capture
   one export with `verbosity: detailed`, and divide by span count. Expect a few hundred bytes to
   ~1 KB depending on attribute count.
   *Why it matters:* it is the input to `v1-03`'s ingest sizing and `v1-05`'s throughput ceiling.
   2,000 spans/sec × 700 bytes ≈ 1.4 MB/s of protobuf, sustained, which is fine on localhost and is
   the sort of arithmetic that stops an SLO being a wish. Do this arithmetic out loud in the session.

2. **Crude telemetry overhead.** Fire the same ~200 sequential requests with `CP_TELEMETRY=on` and
   `off`, compare mean and max `duration_ms` from the response or the log line.
   *State the caveat in the same breath:* sequential requests on a laptop cannot resolve a 2 ms p99
   claim. This is a smoke test for "did we accidentally put the exporter on the request path" — a
   `SimpleSpanProcessor` slip would show up as tens of milliseconds and be unmissable. The real
   measurement is `v1-05`.

3. **Does the span validate against semconv?** Not a number — a check. Read the collector's detailed
   output next to the spec's GenAI attribute list and confirm names, types (int vs string matters)
   and units. Write down any attribute the spec has that we do not emit, and why.

**Also record the prediction for `v1-03`:** at *N* bytes/span and single-row inserts, what
spans/sec do you expect Postgres to sustain? Write the guess down before measuring it. Plan §14
already flags the 2,000 spans/sec SLO as guessed rather than computed; a written prediction turns
the next slice into an experiment instead of an observation.

---

# 6. Out of scope

- **Our own OTLP receiver** — `v1-03`. This slice exports to the *reference* collector on purpose.
- **Postgres, migrations, persistence of any kind** — `v1-03`.
- **Cost, pricing, dollars** — `v1-04`. Usage attributes are emitted; nothing prices them.
- **Metrics and logs signals.** OTel has three signals; v1 uses **traces** only. A `/metrics`
  endpoint arrives at `v1-05` in Prometheus format (plan §14's identified gap) and is a separate
  thing from OTel metrics. Do not build an OTel metrics pipeline here.
- **Sampling of any kind.** `AlwaysSample`, deliberately (plan §2).
- **Auto-instrumentation, monkey-patching, `otelhttp` middleware magic.** Plan §2 puts zero-code
  instrumentation in Pass 2. Wire the spans by hand — you cannot debug a tree you never assembled.
- **gRPC OTLP on :4317** — plan §12.3, deferred.
- **Streaming, mockllm, loadgen, the Anthropic adapter.** Slices 05 and 06.
- **Fixing the `BatchSpanProcessor` silent-drop blind spot.** Noted in Concepts §4, deliberately
  left broken, found and fixed with a before-number at `v1-07`.

---

# 7. The decision point (plan §12.4) — do not end the session without it

After this slice, roughly 300 lines of Go exist. Answer honestly:

**Is Go friction fading or not?**

Useful evidence, in rough order of weight:
- Can you read a compiler error and know what to change, without pasting it anywhere?
- Did `if err != nil` stop feeling like noise?
- Did pointers cause a bug you could not explain for more than ten minutes?
- Did the interface-and-composition-root pattern feel natural or memorised?

**If fading:** keep Go for everything. Record the decision and the reasoning in the plan file.

**If not:** the gateway moves to TypeScript; Go stays for `cmd/mockllm` and `cmd/loadgen` only —
~250 lines, mechanical, and the two places where a single-threaded event loop would become the
bottleneck and silently invalidate every measurement (plan §4). CLAUDE.md §7 explicitly permits this
shape. It costs some hiring signal and **none** of the architecture. `pkg/genai` would then need a
TypeScript twin, which is real work and should be counted before choosing.

Either way: **write the decision and the reason into plan §12.4, struck through and dated.** A
decision that is not recorded gets re-litigated in every later session.

---

# Commit plan

1. `feat(pkg/genai): semconv constants, usage, error taxonomy, chat DTOs`
2. `refactor: move neutral chat types from internal/provider to pkg/genai`
3. `feat(telemetry): OTel tracer provider, resource, batch processor, OTLP exporter`
4. `feat(gateway): server/client spans with genai attributes`
5. `feat(gateway): W3C traceparent extraction`
6. `chore(deploy): reference otel-collector for local span inspection`
7. `refactor: delete hand-rolled span emitter`

---

# Hand-off — what to record in the plan file

- §15 row for `v1-02`: commit SHA, bytes/span, the crude on/off overhead delta, and the surprise.
- §12.4 resolved, dated, with the reason.
- The **written prediction** for `v1-03` ingest throughput.
- Any semconv attribute the spec defines that we chose not to emit, with the reason.

---

# Next

`v1-03` — the store. Postgres in Docker, partitioned schema, our own OTLP receiver, a deliberately
**synchronous** writer, and a query path that rebuilds a trace tree. The biggest slice in v1; it has
a planned split point.

# Slice v1-01 — Walking skeleton

**Version:** v1 · **Slice:** 01 of 08 · **Produces a number?** Yes, a crude one, on purpose.

> **Build-sequence note (added after the portfolio's `LATEST EDIT` revision).** This project is
> **position 0 — `#30-thin`, the control plane spine** — in the only build sequence now in force.
> That section supersedes Parts 10, 11 and 12 of the portfolio file. Position 0's scope is stated
> there as: *"OTel GenAI spans, token and cost accounting, a trace store, and the provider adapter
> interface... The telemetry half of backpressure lives here too — a bounded queue that drops spans
> before user traffic is ever dropped."* Two consequences run through every slice below:
> **(a)** the bounded span queue is now **required scope**, not a discovered fix; and
> **(b)** what comes next is **position 1, `#7` (prefix-cache-aware context assembler)**, which is
> where the *reusable* mock-LLM server and load generator belong. See `v1-05` §0.

> Read `..\..\Shared Context\control_plane_thin_plan.md` before answering anything in this session.
> This file is the contract for one chat. Stay inside it.

---

# Concepts for this slice

Teach these *as they come up in the code*, not as a lecture up front. Each one has a consequence;
lead with the consequence.

### 1. What a walking skeleton is, and why it is slice 01

A walking skeleton is the thinnest possible end-to-end path through every architectural layer the
system will eventually have — real HTTP in, real interface dispatch, real response out, real
telemetry emitted — where almost every part is a stub.

The alternative, and the reason this rule exists: building "all the models", then "all the
handlers", then "all the adapters" produces three finished layers that have never spoken to each
other, and the integration bugs all arrive at once, at the end, stacked on top of each other. A
skeleton front-loads that pain into hour one, when there are forty lines of code to blame.

AGENT_INSTRUCTIONS states it as the ordering rule: *"slice 01 of v1 should get something end-to-end
running, however thin... Everything after that thickens a real path instead of assembling parts
that have never met."*

### 2. Go modules, and why the module path is a URL

`go mod init github.com/intangible-ai/ai-control-plane` sets the module path. Every internal import
then reads `github.com/intangible-ai/ai-control-plane/internal/provider`. This is not decoration
and it triggers no network call — Go uses the URL shape as a globally unique namespace so that two
packages named `provider` from different projects cannot collide. In Node terms: the `name` field
in `package.json`, except it is also the import prefix, so getting it wrong means rewriting every
import line later.

Set it to the repo URL now, even though the GitHub repo does not exist yet. Renaming a module path
is a sed across the tree; doing it before the first commit costs nothing.

### 3. `internal/` is enforced by the compiler

Go has exactly one visibility rule beyond capitalisation, and it is a directory name. A package
under a directory named `internal/` can only be imported by code rooted at that directory's parent.
Nobody outside this repo can import `internal/gateway`. Ever. It is not a convention and not a lint
rule — the compiler refuses.

That is why the plan's layout puts the wire contract in `pkg/genai` and everything else in
`internal/`. CLAUDE.md §9's one-way dependency rule — *modules import `pkg/genai`, never the control
plane* — stops being a promise people have to keep and becomes something the build enforces. Taken
deliberately from T3 Code's `packages/contracts` (portfolio Part 7): **the discipline that makes a
contract portable is enforced by package structure, not by good intentions.**

In this slice `pkg/genai` does not exist yet — it lands in `v1-02`. Put the neutral request and
response types in `internal/provider` for now and *expect to move them*. Noticing which types turn
out to be public contract is the lesson of the next slice; pre-empting it teaches nothing.

### 4. The composition root — where the `switch` on provider name is allowed to live

This is the whole provider-abstraction discipline, and it is one rule:

> The handler must never import `internal/provider/mock`. `cmd/gateway/main.go` may.

`main.go` reads `CP_PROVIDER` from the environment, constructs the right adapter, and hands the
resulting `provider.Provider` value to the handler. The handler's field is typed as the interface,
and it has no idea what is behind it. That single import boundary is the difference between a
provider abstraction and a provider wrapper.

Go has no DI framework and does not need one — the composition root is `main`, wiring is function
arguments, and that is the entire pattern. If a session finds itself reaching for a container or a
registry, something has gone wrong.

The test for whether it is real: *can you swap providers without recompiling business logic?* If the
answer involves editing anything under `internal/gateway`, it is a wrapper.

### 5. Implicit interface satisfaction

`v1-00b` program 4 covered this; connect it to real code here. The mock adapter never declares
"implements Provider". It has a method with the right name and signature, and that is sufficient.

Two consequences worth stating, because they are what the property actually buys:

- At `v1-06`, adding a second real adapter touches **no business logic** — a new package, plus one
  new case in the composition root. Nothing under `internal/gateway` changes. (The `main.go` switch
  *is* an existing file, and that is the point: it is the single place that is allowed to know
  provider names.)
- In tests, a fake satisfies the interface with no mocking library, no `jest.mock`, and no
  registration step. The fake is twelve lines in a `_test.go` file.

### 6. Our neutral request shape, and the alternative we rejected

Door B accepts *our* JSON shape, not Anthropic's and not OpenAI's. Plan §2 names this as a
deliberate v1 limitation, so name it out loud in the session:

| | Neutral shape (chosen) | Wire-compatible proxy (rejected for v1) |
|---|---|---|
| Onboarding cost | a small code change per tenant | change one `base_url`, zero code |
| Abstraction honesty | forced — no provider's quirks leak into the contract | the "neutral" contract is quietly OpenAI's |
| Multi-provider | natural | you are permanently translating everyone into OpenAI shape |
| Streaming | we define the event shape | must reproduce someone else's SSE event names exactly |

Wire-compat is arguably the better *product* decision and the worse *learning* decision. Plan §14
lists it as a low-confidence call that a later pass may reverse. **Caveat under the new sequence:**
the old "Pass 2 rewrite" no longer exists — position 13 is *"write the post, and extract only the
policy pipeline,"* and a big-bang rewrite is ruled out as *"pure risk."* So wire-compat has no
scheduled home any more. That is probably correct (its value is cheaper onboarding, and by position
13 everything is already onboarded) but say it plainly rather than implying someone will get to it.

### 7. Where the timer goes — the headline metric in embryo

The project's headline SLO is *gateway added latency*, which is a subtraction:

```
added_latency = (time the gateway took) - (time the provider took)
```

So from the very first handler there are two stopwatches, not one: total handler duration, and the
duration of the `Chat()` call inside it. In `v1-02` these become a parent span and a child span, and
the subtraction becomes a SQL query. Getting the timers in the right places now means the metric
exists from day one, instead of being retrofitted onto code that only ever measured totals.

### 8. Why the span here is hand-rolled and printed to stdout

`v1-02` replaces this with the real OpenTelemetry SDK. This slice writes a plain Go struct with
`trace_id`, `span_id`, `parent_span_id`, timings and the four token counters, marshals it to JSON,
and prints one line per request.

Reason: a span is a small, boring data structure, and everyone who meets it library-first ends up
believing it is magic. Type it once by hand, look at the JSON, then let the SDK take it over. The
handover in the next slice is also the moment to notice what the SDK gives you that the hand-rolled
version did not — batching, context propagation, ID generation that is actually random enough, and
a wire format other tools already understand.

---

# 1. State going in

**Nothing exists but plans.** This is the first slice with code.

| Thing | State |
|---|---|
| `AI Projects\AI Control Plane\` | exists, contains only `Plan\` |
| `Plan\v1-00-concepts.md`, `Plan\v1-00b-go-for-node-devs.md` | written and read |
| `Shared Context\control_plane_thin_plan.md` | the plan; §15 progress log is empty |
| Git repo | **does not exist yet** |
| `github.com/intangible-ai/ai-control-plane` | **not created yet** |
| Go 1.25.5 windows/amd64 | present (plan §9) |
| Docker | installed, daemon not running — **not needed this slice** |

**Prerequisite from `v1-00b`:** all five on-ramp programs run. Program 5 especially — this slice is
that program plus an interface plus a span. If program 5 has not been typed, do it first; the
session will not go well otherwise.

**Known friction, flag it early:** the absolute path is
`D:\AI Engineering\AI Projects\AI Control Plane\` — three spaces in it. Go does not care. Docker
bind mounts (slice 03) and shell one-liners will, so quote paths in PowerShell and use relative
paths inside `docker-compose.yml`. Do not move the folder; just know why a command occasionally
needs quotes.

**VS Code extensions for this slice** (plan §9): `golang.go` and `humao.rest-client`.

---

# 2. What this slice adds

A Go HTTP service that accepts a chat request, dispatches it through a `Provider` interface to an
in-process fake, returns a completion, and prints one span to stdout.

Files, in the order to write them:

```
AI Control Plane/                  <- git repo root
  .gitignore                       <- Plan/ from the first commit (CLAUDE.md §3)
  go.mod                           <- module github.com/intangible-ai/ai-control-plane
  README.md                        <- three honest lines is fine today
  internal/
    provider/
      provider.go                  <- Provider interface + neutral ChatRequest/ChatResponse/Usage
      mock/
        mock.go                    <- in-process fake, deterministic
    gateway/
      handler.go                   <- POST /v1/chat, holds a provider.Provider
    telemetry/
      span.go                      <- hand-rolled Span struct, JSON to stdout
  cmd/
    gateway/
      main.go                      <- composition root: env -> adapter -> handler -> ListenAndServe
  api/
    chat.http                      <- REST Client requests, so testing is not curl-quoting hell
```

Shape notes for the session — these are constraints, not finished code:

- `Provider` is **one method** in this slice:
  `Chat(ctx context.Context, req ChatRequest) (ChatResponse, error)`.
  Streaming is deliberately **not** in the interface yet — it arrives at `v1-05` against the mock
  server. Extending the interface later, in front of a real reason, is worth more than divining it
  now.
- `ChatRequest` carries `Model`, `Messages`, `MaxTokens`, `Temperature`, plus the tenancy
  dimensions `TenantID`, `Route`, `PromptVersion` (plan §5). Those three exist from the first
  commit because retrofitting a tenancy dimension into a schema *and* a client contract later is
  exactly the migration this project exists to avoid.
- `Usage` carries **four** token fields, not two, from the first commit: `InputTokens`,
  `OutputTokens`, `CacheReadInputTokens`, `CacheCreationInputTokens`. The mock returns plausible
  values *including a nonzero cache read*, so `v1-04` has something honest to price and so nobody
  is tempted to add the cache fields "later".
- `ChatResponse` records both `RequestedModel` and `ResponseModel`. They differ more often than
  people expect and the difference is money (plan §5, and plan §11's constraint from #35).
- The mock adapter is deterministic and instant. It is **not** `cmd/mockllm` — that is a separate
  process with latency distributions and failure injection, and it lands in `v1-05`. Do not build
  it here. This one exists so the skeleton walks.
- `context.Context` is the first parameter of `Chat` even though nothing cancels anything yet.
  Adding it later means touching every implementation and every call site.

---

# 3. Why now

- **Nothing else can be built against nothing.** Every later slice modifies a running path: 02
  swaps the span emitter, 03 puts a store behind it, 04 prices what it stores, 05 loads it. Those
  are all edits. This is the only slice that is a creation.
- **The interface boundary must be right before there are two providers, not after.** If the
  handler imports a concrete adapter today, `v1-06` becomes a refactor instead of a new file — and
  the "does the abstraction hold?" question that `v1-06` exists to answer gets answered by our own
  sloppiness rather than by the second provider's shape.
- **The two timers must exist before the SLO is measured.** Retrofitting per-hop timing onto code
  that only measured totals is how projects end up reporting averages.
- **The repo and `.gitignore` must exist before there is anything worth committing.** `Plan/`
  leaking into a public repo on commit 4 is unpleasant to scrub out of history.
- **This is half the evidence for the Go decision.** Plan §12.4 defers the Go-vs-TypeScript call
  until after `v1-02`. Notice the friction here; do not act on it yet.

---

# 4. Done means

Verifiable line by line:

- [ ] `git init` done in `AI Control Plane\`, first commit exists
- [ ] `.gitignore` contains `Plan/` **in the first commit** — verify with
      `git check-ignore -v Plan/v1-01-walking-skeleton.md`, which must print a match
- [ ] `go.mod` declares `module github.com/intangible-ai/ai-control-plane`
- [ ] `go build ./...` and `go vet ./...` are both clean
- [ ] `go run ./cmd/gateway` listens on `:8080` and logs that it did
- [ ] `POST /v1/chat` with a valid body returns 200 and a JSON completion
- [ ] `POST /v1/chat` with malformed JSON returns 400 — not a panic, not a 500
- [ ] `POST /v1/chat` with an empty `messages` array returns 400 with a usable message
- [ ] `GET /v1/chat` returns 405 (free, because `mux.HandleFunc("POST /v1/chat", ...)` does it)
- [ ] one span per request printed to stdout as a single JSON line, containing `trace_id`,
      `span_id`, `name`, start/end, `duration_ms`, `provider_ms`, the provider name, requested and
      response model, all four token fields, `tenant_id`, `route`
- [ ] `CP_PROVIDER` is read from the environment; an unknown value **exits at startup** with a clear
      error rather than failing on the first request
- [ ] **grep proof of the abstraction:** `internal/gateway` contains zero references to `mock`
- [ ] `api/chat.http` works from VS Code's REST Client
- [ ] GitHub repo `intangible-ai/ai-control-plane` created (public) and pushed
- [ ] plan file §15 progress log has a `v1-01` row with the commit SHA

**Fail-the-slice condition:** if the handler imports the mock package, the slice is not done, no
matter what the curl returns.

---

# 5. What to measure

The number here is crude, and it is crude **on purpose** — the real harness is `v1-05` and this
slice must not grow into it.

1. **Total vs provider time.** Every printed span carries `duration_ms` (handler total) and
   `provider_ms` (the `Chat()` call). Fire ~20 requests from `chat.http` or a PowerShell loop and
   read the difference off the terminal.
2. **Record `added_ms = duration_ms - provider_ms` for one run.** Expect roughly 0.1–1 ms with an
   in-process mock and JSON marshalling on both legs. Whatever it is, write it into the §15 row
   labelled *"crude, single-shot, no load"*.
3. **Say out loud what this number is not.** One sample, on a laptop, against an in-process fake,
   with no concurrency and no GC pressure. It cannot be compared to the 5 ms p99 SLO in plan §7 and
   must never be quoted as if it could. Its only job is to prove the subtraction works and the
   timers are in the right places.

If `added_ms` is surprisingly large (> 5 ms), chase it now — at this size the cause is findable in
a minute, and it is almost certainly JSON re-marshalling or a log write sitting on the hot path.

---

# 6. Out of scope

Explicitly not in this slice. If one turns out to be necessary, say so out loud and name it rather
than quietly widening the session.

- **The OpenTelemetry SDK, OTLP, `pkg/genai`** — `v1-02`. The span here is hand-rolled JSON.
- **Postgres, Docker, migrations, any persistence** — `v1-03`. Spans go to stdout and nowhere else.
- **Cost, pricing config, dollars** — `v1-04`. Token fields are carried, not priced.
- **`cmd/mockllm` as a separate process, latency distributions, failure injection** — `v1-05`.
- **`cmd/loadgen`, percentiles, `bench/`, `/metrics`** — `v1-05`.
- **Streaming, SSE, `http.Flusher`** — `v1-05` (mock) then `v1-06` (real). `Chat()` is unary today.
- **The Anthropic adapter, API keys, any spend** — `v1-06`. This slice costs ₹0 and touches no
  network.
- **Auth, rate limiting, tenancy enforcement.** `TenantID` is a *dimension we record*, not a
  *principal we authenticate*. Do not build auth, and do not let the README imply it exists.
- **Graceful shutdown, `http.Server` timeout tuning, connection limits.** Reasonable to want;
  belongs to `v1-07`, where each setting has a fault to justify it and a number to prove it.
- **A test suite.** One table-driven handler test covering the 400 paths is worth it. More is not
  the deliverable of a skeleton.
- **The Go-vs-TypeScript decision** (plan §12.4). Gather evidence; decide after `v1-02`.

---

# Commit plan

Small and meaningful (CLAUDE.md §8), roughly:

1. `chore: init module, gitignore, readme`
2. `feat(provider): neutral chat types and Provider interface`
3. `feat(provider/mock): deterministic in-process adapter`
4. `feat(gateway): POST /v1/chat handler with provider dispatch`
5. `feat(telemetry): hand-rolled span emitted to stdout`
6. `feat(cmd/gateway): composition root, CP_PROVIDER wiring`

Never `--no-verify`. Push after 6.

---

# Hand-off — what to record in the plan file

Append a row to §15 of `Shared Context\control_plane_thin_plan.md`:

| Slice | Date | Commit | Landed | Number | Surprise |
|---|---|---|---|---|---|
| v1-01 | *date* | *SHA* | repo, Provider iface, mock adapter, POST /v1/chat, stdout span | added_ms ≈ *x* (crude, single-shot) | *what actually surprised you* |

Also note it if the neutral request shape already felt awkward to construct — that is early evidence
for the Pass-2 wire-compat question in plan §14.

---

# Next

`v1-02` — the wire contract. `pkg/genai`, the real OTel SDK, OTLP on the wire, parent/child spans,
and `traceparent` propagation. Ends with the Go-vs-TypeScript decision point.

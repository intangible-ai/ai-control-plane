# Slice v1-08 — Onboard a tenant, and write it up

**Version:** v1 · **Slice:** 08 of 08 · **Produces a number?** Yes — **the portfolio's own headline metric.**

> **Build-sequence note (added after the portfolio's `LATEST EDIT` revision).** This project is
> **position 0 — `#30-thin`, the control plane spine** — in the only build sequence now in force.
> That section supersedes Parts 10, 11 and 12 of the portfolio file. Position 0's scope is stated
> there as: *"OTel GenAI spans, token and cost accounting, a trace store, and the provider adapter
> interface... The telemetry half of backpressure lives here too — a bounded queue that drops spans
> before user traffic is ever dropped."* Two consequences run through every slice below:
> **(a)** the bounded span queue is now **required scope**, not a discovered fix; and
> **(b)** what comes next is **position 1, `#7` (prefix-cache-aware context assembler)**, which is
> where the *reusable* mock-LLM server and load generator belong. See `v1-05` §0.

> Read `..\..\Shared Context\control_plane_thin_plan.md` §2 (what v1 is *not*), §6 (the six things),
> §7 (the four deliverables and the SLOs) and §14 (confidence) before answering anything.
>
> This is the slice that closes v1. It proves the claim the whole architecture was built on, and it
> is the only slice whose output a hiring manager will actually read.

---

# Concepts for this slice

### 1. "Vendor-neutral" is a claim, and claims are measured

Every observability vendor says their format is open. The test is simple and almost nobody passes
it: **can a system in another language, written by someone who has never seen your repo, send you
usable telemetry without importing anything you wrote?**

Plan §3 built the whole two-door architecture around passing that test — *"We ship no SDK. That is
the whole vendor-neutral argument made real rather than asserted: if onboarding required our
library, the contract would not be neutral, it would be ours."*

This slice is the exam. A Python program, the stock OpenTelemetry SDK, two environment variables,
and attribute names taken from a public spec. If it works, the claim is demonstrated. If it needs
one line of our code, the claim is false and the README must say so.

**Both outcomes are publishable.** What is not publishable is asserting neutrality without having
tried it.

### 2. Lines-of-code-to-onboard, and how to measure it honestly

This is the portfolio's own headline metric for the project, so it deserves an honest definition
rather than a flattering one. Report a **breakdown**, not one number:

| Category | What it counts |
|---|---|
| Lines of **our** code imported | should be **0** — this is the claim |
| OTel SDK boilerplate | provider, exporter, processor, shutdown — real cost, and identical for any OTLP backend |
| Attribute-setting lines | per span; the vocabulary cost |
| Configuration | env vars, not code |

The interesting comparison is not against zero, it is **against the alternative**: onboarding to a
vendor that ships an SDK costs a dependency, a version constraint, a supply-chain review, and a
migration when you leave. Our cost is boilerplate that is portable to any OTLP receiver on earth.
Say it that way, because that is the actual argument.

And name the limitation in the same breath, because plan §2 already did: **this is not zero-code
onboarding.** Tier-1 auto-instrumentation and a wire-compatible proxy were deferred to "Pass 2" —
which the new sequence has replaced with position 13, *"extract only the policy pipeline."* Neither
has a scheduled home now, so state the limitation as permanent-until-rescheduled rather than as
coming-later. A
README that implies otherwise is the thing plan §2 was written to prevent — *"Named here so it does
not get quietly claimed in a README."*

### 3. One trace, two languages, two doors — the demo that proves the architecture

The best artefact this project can produce is a single trace tree containing:

```
python.agent.handle_request        <- Door A, Python, stock OTel SDK
  ├─ python.retrieve_context       <- Door A, Python
  └─ gateway.chat                  <- Door B, Go, our gateway
       └─ provider.chat            <- Door B, Go
```

Four spans, two languages, two independent ingestion paths, **one `trace_id`**, and no shared
library anywhere. That is the entire thesis of the plan's §3 in one screenshot.

The mechanism is the one built at `v1-02`: the Python SDK injects a W3C `traceparent` header into
its outbound HTTP call, and the gateway extracts it instead of starting a fresh root. Python's spans
arrive by OTLP; Go's arrive by OTLP; the tree is reassembled in Postgres by `parent_span_id`
(`v1-02` Concepts §1). Nothing about it is clever — which is precisely the point, because it means
any tenant in any language gets it for free.

**If the two halves land as two separate traces, the propagation is broken**, and finding that here
is exactly why this slice exists rather than being assumed at `v1-02`.

### 4. Semconv version skew is a real risk and this is where it surfaces

The Go SDK and the Python SDK ship their own copies of the semantic conventions, on their own
release cadences, against a **GenAI spec that is still evolving** (`v1-02` Concepts §7). So the two
languages can legitimately disagree about an attribute name for the same concept.

Check it concretely: emit from both, query the `attributes` jsonb, and compare keys. Possible
outcomes and what each means:

- **Identical keys.** The contract holds across languages. Say so, with the two SDK versions named.
- **Different keys.** *This is a genuinely valuable finding* — it is the practical cost of adopting
  an unstable spec, and it is the argument for `pkg/genai` pinning a version and centralising every
  string. Write it up; do not paper over it by normalising in the collector and staying quiet.

Either way, record both SDK versions and the semconv version in the README. A contract claim without
version numbers is not checkable.

### 5. The short-lived-process flush trap

Flagged at `v1-02` Concepts §4 and it lands here for real. A 50-line script does its work and exits.
The `BatchSpanProcessor` is asynchronous by design — so **the process can exit before the batch is
exported, and the spans simply never arrive.**

The symptom is maddening precisely because nothing is wrong: no error, no warning, an empty table,
and code that reads correctly. The fix is one line — `provider.shutdown()` (or a `finally`, or an
`atexit` hook) — which forces a flush.

Worth teaching properly, because it generalises: **asynchronous export and short process lifetimes
are in tension, and the resolution is always an explicit flush at the boundary.** Serverless
functions, CLI tools, and cron jobs hit this constantly.

### 6. A README is a technical argument, and its limitations section is the credible part

Two audiences: someone deciding whether to keep reading, and someone (an interviewer) deciding
whether you know what you built. Both are served by the same structure:

1. **What it is**, in three sentences.
2. **The architecture diagram** and the two doors.
3. **The numbers** — SLO table with targets stated in advance and verdicts beside them.
4. **The failure mode** — what broke, what it cost, what fixed it, what the fix cost.
5. **What it deliberately is not** — plan §2's list, essentially as written.
6. **How to run it** in five minutes from a clean checkout.

Section 5 is the one that earns trust. Plan §2 is already a list of honest, reasoned non-goals with
justifications; moving it into the README converts "here is a limitation" into "here is a decision."
A project that names its own boundaries reads as engineering. One that claims everything reads as a
demo.

Same rule for the numbers: a missed SLO, stated in advance and reported plainly, is worth more than
five hit ones, because it is evidence the targets were set before the results were known
(plan §14).

---

# 1. State going in

From `v1-07` (or `07a` + `07b`), committed:

- Full v1 path: gateway (two real providers), collector, Postgres, cost engine, `cpq`
- `mockllm`, `loadgen`, `/metrics`, `bench/scenarios` and `bench/results`
- `docs/failure-modes.md` with the before/after fault comparison
- The bounded queue (or whatever fix the data actually chose), with counters
- Plan §15 rows for 01–07; SLO verdicts recorded; deliverables 1, 2 and 3 ticked

**Setup needed this slice** (plan §9): Python 3.12.11 is present. VS Code extensions: Python +
Pylance. Everything else is a virtualenv and two packages. Cost: **₹0**.

---

# 2. What this slice adds

```
  examples/agent-python/
    agent.py              <- ~50 lines: Door A spans + a Door B call, one trace
    requirements.txt      <- opentelemetry-sdk + opentelemetry-exporter-otlp-proto-http
                             (opentelemetry-api arrives transitively — two direct installs)
    README.md             <- the onboarding instructions, written for a stranger
  README.md               <- the project README: architecture, numbers, limits, quickstart
  docs/
    architecture.md       <- the two doors, the decisions and the reasons
    slo-scorecard.md      <- targets stated in advance, verdicts, and the misses
  bench/results/          <- final committed numbers
```

Notes for the session:

- **The Python agent imports nothing of ours.** That is the experiment; violating it invalidates the
  slice. It uses `opentelemetry-sdk`, an OTLP HTTP exporter, `OTEL_EXPORTER_OTLP_ENDPOINT`, and
  attribute names typed from the public spec.
- **Endpoint env-var gotcha, worth knowing before it costs an hour:** the SDK appends `/v1/traces`
  to `OTEL_EXPORTER_OTLP_ENDPOINT`, but treats `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` as the complete
  URL. Setting the wrong one produces 404s against `/v1/traces/v1/traces`, and the error is not
  obvious from the Python side.
- **The agent must do both doors**: emit its own spans (Door A) *and* call `POST /v1/chat` with a
  propagated `traceparent` (Door B). One door alone does not demonstrate the architecture.
- **Shape the example to prefigure position 1.** The next project is `#7`, the prefix-cache-aware
  context assembler, and the sequence says *"every project after this emits telemetry from day one."*
  So make the example agent assemble a prompt from a stable preamble plus a volatile question, and
  emit the four usage fields. It costs nothing extra — that is a realistic agent anyway — and it
  means `#7` starts from a working, instrumented tenant instead of a blank file. **Do not implement
  any cache optimisation here.** The example just has to have the shape that `#7` will optimise.
- `provider.shutdown()` in a `finally` (Concepts §5).
- **`examples/agent-python/README.md` is written for someone who has never seen this repo.** That
  framing is the test — if it needs a paragraph explaining our internals, the contract is leaking.
- **Verify `Plan/` is still ignored** before the final push (CLAUDE.md §3): `git check-ignore -v` and
  a scan of the actual GitHub file listing, not just local state.

---

# 3. Why now

- **Because it is the proof, and everything before it was preparation.** Every design decision —
  no SDK, OTel semconv, `pkg/genai` importing nothing internal, OTLP ingest, `traceparent`
  propagation — was made to pass this specific test. Not running it leaves the central claim
  untested.
- **Because it is the last honest moment to find that propagation is broken.** Once other projects
  onboard, a broken trace join is four repos' problem instead of one slice's.
- **Because the numbers exist and the write-up is the deliverable.** AGENT_INSTRUCTIONS: *"two or
  three good engineering blog posts outperform ten repos."* The numbers, the graphs and the fault
  story are all on disk right now. This is when they are cheapest to turn into prose.
- **Because position 1 cannot start until this project is closed.** The gate in §8 is a measurement,
  not a feeling, and the scorecard produced here is what `#7` inherits — including the baseline
  hand-off that position 8 (`#30-store`) will need.

---

# 4. Done means

- [ ] `examples/agent-python/agent.py` is ~50 lines and **imports nothing from this repo** —
      verified by reading the imports
- [ ] it runs from a clean virtualenv with only `requirements.txt` installed
- [ ] its spans arrive in Postgres through Door A with correct GenAI attributes
- [ ] it calls Door B with a propagated `traceparent`
- [ ] **`cpq trace <id>` shows one tree containing both Python and Go spans** — the money demo
- [ ] the short-lived-process flush works; running the script twice yields two complete traces
- [ ] Go and Python attribute keys compared; agreement or disagreement recorded with both SDK
      versions and the semconv version
- [ ] **lines-of-code-to-onboard measured and reported as a breakdown**, not one number
- [ ] `examples/agent-python/README.md` is written for a stranger and contains no internal detail
- [ ] project `README.md` has: what it is, the architecture diagram, the SLO scorecard with
      verdicts, the failure-mode summary, **the deliberate non-goals from plan §2**, and a
      five-minute quickstart
- [ ] the quickstart is **executed from a clean clone** and it works — not just read
- [ ] `docker compose down -v` appears beside every `up` in the README (CLAUDE.md §6)
- [ ] `docs/slo-scorecard.md` lists every SLO with its **originally stated** target and the measured
      result, misses included
- [ ] final numbers committed in `bench/results/`
- [ ] `Plan/` confirmed absent from the GitHub repo
- [ ] no secret in the repo — `.env` ignored, no key in any committed file or any span
- [ ] plan §15 has a `v1-08` row; **all four deliverables ticked**
- [ ] the blog post is drafted (not necessarily published)
- [ ] the done gate in §8 is explicitly declared — all three conditions, individually

---

# 5. What to measure

1. **Lines-of-code-to-onboard — the headline.** The breakdown from Concepts §2: our code (target 0),
   OTel boilerplate, attribute lines, config. State the definition beside the number so it is
   checkable rather than impressive.

2. **Cross-language trace completeness.** Run the agent N times (100 is plenty) and count how many
   traces contain the full set of expected spans from both languages. Report the percentage. This is
   plan §6's observability metric — *"trace completeness % under sustained load"* — finally measured
   across the boundary it was written for. Anything under 100% needs an explanation, not a shrug:
   flush timing, batch timeouts, or a propagation bug.

3. **Attribute-key agreement across SDKs.** How many of the GenAI attributes we emit are named
   identically by both SDKs. Concepts §4. A disagreement is a finding worth its own paragraph.

4. **Time-to-first-trace.** Wall-clock from a clean clone to a trace visible in `cpq`, following only
   the README. Time it honestly, with a stopwatch, including the mistakes. This is a *usability*
   measurement of your own documentation, most projects never take it, and every number it produces
   is actionable.

5. **The final SLO scorecard.** Every row of plan §7 with:
   - the target **as originally stated**,
   - the measured result,
   - hit or missed,
   - and for each miss, why, and whether it is beaten before the project closes or reported as a miss.

   Include the ones re-baselined at `v1-03` with **both** numbers visible. Plan §14 is unambiguous
   about why: *"A missed SLO that was stated in advance is a finding worth writing up. A missed SLO
   quietly edited afterwards is the thing that makes a portfolio worthless."*

6. **Total spend, all of v1.** Against the ₹2,000 ceiling and the ~₹160 projection (plan §10). A
   cost-control project should be able to report its own cost to the rupee.

---

# 6. Out of scope

- **A second example agent, or another language.** One is the proof. Two is repetition.
- **Auto-instrumentation for Python** (`opentelemetry-instrument`, monkey-patching). Plan §2 deferred
  it and the new sequence does not reschedule it. Tempting because it would make the LOC number look
  better — which is exactly the wrong reason.
- **Onboarding a real portfolio project (#9, #1, #21).** Those are their own projects with their own
  slices. This slice proves the door works.
- **A dashboard or UI.** Plan §2 — `cpq` and SQL here. The portfolio's cross-cutting rule allows
  exactly one narrow view per project where it makes the headline number legible, and for the plane
  that view is the **trace waterfall with the critical path highlighted**, which belongs to
  position 4 (`#30-S L1`) where there is finally a critical path to draw. This project's headline
  output is a table of percentiles, which reads perfectly well as text.
- **Any optimisation beyond beating a stated SLO.** The project ends here. Improvements that belong
  to another position (§8a) go in the plan file as a hand-off, not into this repo.
- **Writing v2 slice files.** There is no v2 for this project (§8). The next planning session is
  position 1's, and it is a fresh session.
- **Publishing the blog post.** Drafting it is in scope; publishing is the user's call and their
  voice.

---

# 7. Wrapping up v1 — the closing checklist

Run through **CLAUDE.md §4's six** and give each an honest verdict, with the number or an explicit
"not covered":

| | Verdict from v1 |
|---|---|
| **Observability** | trace completeness %, cross-language |
| **Security** | no credential in any span; bodies not capturable by construction; *and the honest note that there is no auth on the OTLP endpoint* |
| **Cost control** | cost accuracy vs provider-reported; the naive-vs-cache-aware delta |
| **Latency** | added latency p99; TTFT p95 |
| **Accuracy** | **not covered in v1 — that is #27.** Stated, not faked (plan §6) |
| **Availability** | error rate under fault, before and after the fix |

Then the four deliverables (plan §7):

- [ ] 1. Benchmark harness exists and runs
- [ ] 2. Baseline measured, then beaten
- [ ] 3. At least one failure mode found and written up honestly
- [ ] 4. SLOs stated in advance — hit or missed, reported either way

---

# 8. The done gate — and why there is no v2

Plan §3 and AGENT_INSTRUCTIONS define the gate as a measurement, not a feeling:

> *"v1 ends when the benchmark harness runs, the baseline numbers exist, and the system has been
> deliberately broken at least once."*

Declare all three explicitly in this session. If any is not true, **the project is not finished** and
the missing piece is the next slice — regardless of how complete the code looks.

**What that gate now opens is position 1 (`#7`), not a v2 of this project.** `CLAUDE.md` §3's
v1/v2 protocol is portfolio-wide and still governs other projects; for `#30-thin` specifically it has
exactly one member, because the `LATEST EDIT` resequencing gave every would-be v2 item its own
numbered position — see the table in §8a. The last candidate, the columnar store migration, became
position 8 (`#30-store`).

The only optimisation work left inside this project's own boundary is **beating a baseline it
measured itself** — a missed SLO from `v1-05`, profiled and fixed. That is not a version 2; plan §7
and CLAUDE.md §4a put *"a baseline that was measured and beaten"* inside the definition of done, and
`v1-07` already runs that loop in-slice. If an SLO is missed and not beaten, say so in the scorecard
and move on: an honestly reported miss is an asset (plan §14), and manufacturing a v2 to hide it is
not.

So record in the plan file, at the close of this session:

- the three gate conditions, each confirmed
- the SLO scorecard, misses included
- **the baseline hand-off to position 8** (§8a)
- anything found at `v1-07` and deliberately not fixed, with the number beside it
- plan §12.4's language decision, and plan §14's low-confidence claims that turned out wrong
- the open Bedrock/AWS question from plan §4a, if it was ever answered

Then stop. **Do not write v2 slice files for this project** — there is no v2 *inside the plan* to
plan. The next planning session is position 1's.

One caveat, so "there is no v2" is not read as "this is finished forever": a v2 can also be
optimization work that sits **entirely outside the portfolio plan**, taken up separately once the
user has finished the plan, on whichever projects the numbers say are worth going back to. This
project will very likely be one of them — a spine that seven things depend on, with a full set of
measurements attached, is an obvious candidate. That is a *later, optional* pass, deliberately not
scheduled, and it is not a reason to hold anything back from v1 now.

**One case that looks like a v2 and is not:** a later project needing the plane changed — `#9`
wanting something new through Door A, or `#21` finding the span schema too narrow. That is
maintenance of a dependency, done inside whichever project needs it, and position 13 (`#30
extraction`) is where the accumulated lessons get written up. Neither is a version of this project.

---

### 8a. Where each deferred item actually went

There is no scheduled "come back and rewrite the plane" project. Position 13 is *"write the post,
and extract **only** the policy pipeline,"* and it explicitly rules out more: *"a big-bang rewrite of
a foundation seven things depend on is pure risk."*

Everything this project deliberately deferred now has a *named position elsewhere*. This table is the
full inventory, and it is why §8 says there is no v2 — nothing on it is this project's work:

| Deferred from v1 | Now lives at |
|---|---|
| Reusable mock-LLM + load generator | position 1 (`#7`) |
| Tail-based sampling, cardinality/label limits | position 7 (`#21`) |
| User-traffic shedding, admission control, degrade-to-cheaper | position 11 (`#35`) |
| Budget enforcement, per-feature attribution | position 12 (`#29`) |
| Hot-reloadable policy pipeline, PII and schema guards | position 13 |
| Critical-path / tail attribution over the DAG | position 4 (`#30-S L1`) |
| Eval, accuracy, LLM-as-judge | position 6 (`#27`) |

**The one that was homeless is now placed — and this project owes it a number:**

> **The ClickHouse / columnar migration is position 8 (`#30-store`)**, added to the sequence after
> `#21` on the reasoning that the pressure forcing sampling at position 7 is the same pressure that
> makes row storage the wrong shape.
>
> That resolves *where* it happens. It does **not** remove an obligation from this project. Position
> 8's premise is a quotation of plan §4a — *"I built on Postgres, measured the wall at N spans/sec,
> and moved to ClickHouse for a 12x ingest improvement"* — and **N is measured here**, at `v1-03`,
> `v1-05` and `v1-07`. So the hand-off is concrete:
>
> - the sustained spans/sec ceiling, with hardware, commit SHA and index configuration beside it
> - what saturated first (write path, index maintenance, connection pool, CPU)
> - the batch-size curve from `v1-03` §5, because it tells position 8 what to compare against
> - whether the wall was hit at all — position 8 explicitly allows the measurement to cancel it, and
>   "Postgres held at portfolio volumes" is a publishable result, not a failure to find a problem
>
> Put that in the README and in plan §15. A migration project that has to re-derive its own baseline
> has lost the argument before it starts.

Two smaller ones stay dead, and that is the right call: **wire-compatible drop-in proxy mode** and
**tier-1 auto-instrumentation** were both deferred to the old Pass 2 and were deliberately *not*
given a position. Their only real value is cheaper onboarding, and by the end of the sequence every
tenant is already onboarded — so they would improve a number for a population of zero. State the
limitation in the README as permanent-until-rescheduled, not as coming later. (If wire-compat is
ever revived, the argument that would justify it is adoptability of the **router** at position 11,
not onboarding cost here.)

---

# Commit plan

1. `feat(examples): python agent emitting through door a with the stock otel sdk`
2. `feat(examples): traceparent propagation into door b`
3. `docs(examples): onboarding guide for a new tenant`
4. `docs: architecture and the two-door design`
5. `docs: slo scorecard with stated targets and measured verdicts`
6. `docs: project readme with numbers, limits and quickstart`
7. `chore: final v1 benchmark results`

---

# Hand-off — what to record in the plan file

- §15 row for `v1-08`: commit SHA, **LOC-to-onboard**, cross-language trace completeness, the
  surprise.
- **Deliverable tracker: all four ticked**, or an explicit statement of which is not and why.
- **The done gate declared**, with the three conditions individually confirmed.
- **The hand-off list** — every deferred item, its position from §8a, and the number that justifies
  it. The baseline hand-off to position 8 (`#30-store`) is the one with a hard dependency on it.
- **Status line at the top of the plan file** updated from *planned, not started* to v1 complete,
  dated, with the repo URL.
- §10: total v1 spend against the ₹2,000 ceiling.
- Anything in the plan that turned out to be wrong. AGENT_INSTRUCTIONS: *"If an approach in the
  portfolio file turns out to be wrong in practice, say so and explain why. The portfolio is a plan,
  not scripture."* The same applies to this plan — and the corrections are the most valuable thing a
  future project inherits from it.

---

# Next

**Nothing — this project is finished.** The build sequence says the next thing is **position 1,
`#7`**, and its planning session starts from this project's scorecard. But CLAUDE.md §1 still
governs: *do not assume the next project — ask or wait.*

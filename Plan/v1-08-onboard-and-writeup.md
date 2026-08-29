# Slice v1-08 — Onboard a tenant, and write it up

**Version:** v1 · **Slice:** 08 of 08 · **Produces a number?** Yes — **the portfolio's own headline
metric.**

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
onboarding.** Tier-1 auto-instrumentation and a wire-compatible proxy are explicitly Pass 2. A
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
- **Because v2 cannot be planned until v1 is closed.** Plan §3: v2 slices are written from what the
  v1 numbers and failures actually showed. The scorecard produced here is the input to that planning
  session.

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
- [ ] the v1 → v2 gate in §6 is explicitly declared

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
   - and for each miss, why, and whether it becomes a v2 slice.

   Include the ones re-baselined at `v1-03` with **both** numbers visible. Plan §14 is unambiguous
   about why: *"A missed SLO that was stated in advance is a finding worth writing up. A missed SLO
   quietly edited afterwards is the thing that makes a portfolio worthless."*

6. **Total spend, all of v1.** Against the ₹2,000 ceiling and the ~₹160 projection (plan §10). A
   cost-control project should be able to report its own cost to the rupee.

---

# 6. Out of scope

- **A second example agent, or another language.** One is the proof. Two is repetition.
- **Auto-instrumentation for Python** (`opentelemetry-instrument`, monkey-patching). Plan §2: Pass 2.
  Tempting because it would make the LOC number look better — which is exactly the wrong reason.
- **Onboarding a real portfolio project (#9, #1, #21).** Those are their own projects with their own
  slices. This slice proves the door works.
- **A dashboard or UI.** Plan §2. `cpq` and SQL.
- **Any optimisation.** v1 ends here. Every improvement identified belongs in the v2 plan with the
  number that justifies it.
- **Writing v2 slice files.** Plan §3 is explicit — v2 is planned in its own session, from these
  numbers. Not now, and not from imagination.
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

# 8. The v1 → v2 gate

Plan §3 and AGENT_INSTRUCTIONS define the gate as a measurement, not a feeling:

> *"v1 ends when the benchmark harness runs, the baseline numbers exist, and the system has been
> deliberately broken at least once."*

Declare it explicitly in this session. If any of the three is not true, **v1 is not finished** and
the missing piece is the next slice — regardless of how complete the code looks.

If it is true, write into the plan file what v2 planning starts from. Candidates already have
evidence attached, and each should carry its number:

- the ingest wall from `v1-03`/`v1-05` → sampling, or ClickHouse
- any missed SLO from `v1-05` → the specific optimisation, with the profile that points at it
- anything found at `v1-07` and deliberately not fixed
- plan §12.4's language decision, if it is worth revisiting with real evidence
- plan §14's low-confidence claims that turned out wrong
- the open Bedrock/AWS question from plan §4a, if it was ever answered

**v2 slices are written in a fresh session, from these numbers.** Not in this one.

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
- **The v1 → v2 gate declared**, with the three conditions individually confirmed.
- **The v2 input list** — every candidate with the number that justifies it.
- **Status line at the top of the plan file** updated from *planned, not started* to v1 complete,
  dated, with the repo URL.
- §10: total v1 spend against the ₹2,000 ceiling.
- Anything in the plan that turned out to be wrong. AGENT_INSTRUCTIONS: *"If an approach in the
  portfolio file turns out to be wrong in practice, say so and explain why. The portfolio is a plan,
  not scripture."* The same applies to this plan — and the corrections are the most valuable thing a
  future project inherits from it.

---

# Next

**Nothing in v1.** The next session is either the v2 planning session — which starts from the
scorecard and the failure modes, never from imagination — or the next portfolio project, which the
user chooses (CLAUDE.md §1: *do not assume the next project — ask or wait*).

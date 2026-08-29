# Slice v1-04 — Cost accounting

**Version:** v1 · **Slice:** 04 of 08 · **Produces a number?** Yes — the one the portfolio's #7
later depends on being trustworthy.

> Read `..\..\Shared Context\control_plane_thin_plan.md` §5 ("The cost model is not
> `input × price_in + output × price_out`") before answering anything.
>
> Plan §5 calls this *"the single most important technical detail in v1, and getting it wrong
> invalidates every dollar figure the portfolio later claims."* Treat the session accordingly.

---

# Concepts for this slice

Plan §8: *"`v1-04` opens with KV prefix caching in depth — the mechanism, not just the pricing."*
`v1-00` §2.2 gave the shape and the consequence. This goes underneath it, because the pricing
multipliers only stop looking arbitrary once you know what is physically being cached.

### 1. What is actually in the cache

A transformer generating text does one thing repeatedly: for the next token, attend to every
previous token. "Attend to" means computing, for each previous token and each layer, a **key** and a
**value** vector, and combining them.

The critical property: **a token's key and value vectors never change once computed.** Token 40's
K/V depends on token 40 and the tokens before it, and nothing that happens at token 900 alters it.
So they are computed once and kept — that store is the **KV cache**, and it is what stops every new
token from re-processing the whole sequence from scratch. (Be precise if it comes up: generating
token *n* still attends over *n* cached vectors, so the per-token work is linear in sequence length.
What the cache removes is *recomputing* them, which would make the whole generation quadratic.)

A request has two phases with completely different cost shapes:

| Phase | What happens | Cost driver |
|---|---|---|
| **Prefill** | compute K/V for every prompt token, all layers, in parallel | compute-bound — this is most of your input bill |
| **Decode** | generate one token at a time, reading the cache | memory-bandwidth-bound — this is the output bill |

**Prompt caching is the provider persisting the prefill result across requests.** If your next
request starts with the same tokens, the K/V for that stretch already exists and prefill is skipped.
You are not paying for a lookup table of answers — you are skipping a matrix computation.

### 2. Why it is a *prefix* cache and can never be anything else

Because of causal attention plus position. Token *i*'s K/V is a function of tokens 0..*i* and of
*i* itself. Change anything at position *j*, and every token from *j* onward has different context
or a different position, so every cached K/V from *j* onward is invalid.

Which gives the rule that drives an entire portfolio project: **the cache matches from byte zero,
exactly, and stops at the first difference.** Not fuzzy. Not similarity. Not "mostly the same".
A timestamp at the top of your prompt costs you the entire cache, every request, forever.

That is why the fix in `v1-00` §2.2 is *reordering*: stable bytes first, volatile bytes last. Same
information, same answer, roughly a tenth of the input bill. Turning that into an optimisation
problem over (priority, token count, cache-stability) is portfolio project **#7** — and #7's whole
pitch is a dollar figure this engine has to be able to produce.

### 3. Why writing to the cache costs *more* than not caching

This is the multiplier that confuses people, and the explanation is physical:

- `cache_read_input_tokens` ≈ **0.1×** — no prefill compute, just loading tensors. Cheap because the
  expensive part was skipped.
- `cache_creation_input_tokens` = **1.25×** (5-minute TTL) or **2×** (1-hour TTL) — you pay full
  prefill *plus* a premium, because the provider now has to hold those tensors in expensive memory
  for the whole TTL whether or not you come back.

The break-even arithmetic is worth doing in the session, out loud, because it is the actual
engineering question:

| TTL | Write | Read | Break-even |
|---|---|---|---|
| 5 minutes | 1.25× | 0.1× | **2 requests** — 1.25 + 0.1 = 1.35× vs 2.0× uncached |
| 1 hour | 2.0× | 0.1× | **3 requests** — 2.0 + 0.2 = 2.2× vs 3.0× uncached |

Below that frequency, caching costs you money. The 1-hour TTL survives gaps in bursty traffic, but
the doubled write premium means it needs more reads to pay off. Caching is not free and it is not
always right — that sentence is worth more in an interview than any amount of enthusiasm about it.

### 4. Three ways prompt caching silently does nothing

Cost engines get built and then quietly report zero cache activity. The usual causes:

1. **Below the minimum cacheable length.** Providers only cache prefixes above a token threshold,
   and under it `cache_control` is accepted and ignored — no error, no warning, just
   `cache_creation_input_tokens: 0`. The threshold is **model-dependent and not monotonic across
   generations**, which is the part that catches people:

   | Model | Minimum cacheable prefix |
   |---|---:|
   | `claude-opus-5` | 512 tokens |
   | `claude-sonnet-5` | 1,024 tokens |
   | `claude-haiku-4-5` | **4,096 tokens** |

   Note which model the plan picked for the paid slice: **Haiku 4.5, with the highest minimum of
   the three.** A 2,000-token prompt caches on Opus 5 and silently does not cache on Haiku. Carry
   this into `v1-06` — the prompt used to exercise the cache path there must clear 4,096 tokens or
   the experiment measures nothing.
2. **The prefix is not byte-identical** — a serialiser that does not order JSON keys deterministically
   is enough to destroy it, and it will look fine in every log.
3. **The TTL expired.** Low-traffic routes may never hit a warm cache. A 5-minute TTL and a request
   every seven minutes means paying the 1.25× write premium every single time and never reading it —
   strictly worse than not caching at all.

Failure 3 is only visible if you record all four token counters *per route*. Which is precisely what
this slice builds, and precisely why the control plane has to exist before #7 does.

### 5. The arithmetic, and the trap in `input_tokens`

Restating plan §5 because everything below depends on it:

```
true prompt size = input_tokens + cache_creation_input_tokens + cache_read_input_tokens
```

`input_tokens` is **the uncached remainder only**. It is not the prompt size.

**There are two different naive engines, and they are wrong in opposite directions.** Be precise
about which one you are comparing against, because the plan file is not:

| Naive engine | Formula | Error on a cache-heavy workload |
|---|---|---|
| **A — "trust the field name"** | `input_tokens × price_in + output × price_out` | **under**-reports: the cached tokens are simply never counted at all |
| **B — "count the whole prompt at list price"** | `(all prompt tokens) × price_in + output × price_out` | **over**-reports by up to ~10×: reads that cost 0.1× are billed at 1.0× |

Plan §5's claim — *"over-bills a cache-heavy workload by up to ~10x on the prompt side"* — is about
**B**. Engine A is the more common bug in practice, because it is what you get by reading the field
called `input_tokens` and assuming it means what it says. **Note the discrepancy in the plan file
during this session**, since "our cost model is wrong in this direction by this much" is not a
sentence to be vague about.

Both share the failure that actually matters here: a **flat** cost curve for a system whose real
cost is dominated by cache behaviour. When #7 takes the hit rate from 0% to 85%, engine A reports a
cost drop it cannot explain and engine B reports no change at all — and neither can attribute the
saving to caching, which is the only thing #7 is claiming.

Measuring both naive numbers *and* the correct number over the same data, in this slice, is how the
claim stops being a story and becomes a percentage. See §5.

### 6. Money in floating point is a defect, not a style preference

`float64` cannot represent 0.1. Errors are tiny per operation and they accumulate, and they
accumulate *with a bias* — which is exactly what a cost roll-up is: a sum over millions of tiny
numbers.

Our inputs are pleasant: token counts are integers and prices are decimals per million tokens, so
`cost = tokens × price ÷ 1e6` is an **exact rational number**. There is no reason to be approximate.

- **Recommendation: `math/big.Rat`** — standard library, exact, and it forces the "where do I round"
  question to be answered explicitly instead of by the FPU.
- Alternative: `github.com/shopspring/decimal`, nicer ergonomics, one dependency, and its
  fixed-point rounding is a choice you now have to configure. Name it; pick `big.Rat` unless the
  ergonomics genuinely hurt.
- Storage is `numeric(20,10)` (plan §5), formatted from the rational at the boundary.

**Rounding policy, stated once and obeyed:** never round per request. Store full precision and round
only for display. Rounding each of a million requests to the cent and then summing produces a
systematic error worth real money — this is the classic accounting bug and it is trivial to avoid by
rounding last.

### 7. Unknown price is not zero price

The engine will meet a model that is not in the price list — a new release, a typo, a provider
returning a different `response.model` than requested (which plan §5 warns happens more often than
people expect, and plan §11 makes a hard requirement for #35).

**A missing price must never produce `cost = 0`.** Zero is a number, it sums, it lands on a
dashboard, and it makes a bill look fine right up until it does not. Instead:

- `cost_usd = NULL`
- `pricing_status = 'unknown_model'`
- a counter, surfaced at `/metrics` when that lands in `v1-05`
- roll-up queries report unpriced spans as a **separate line**, never folded into the total

The general principle, and it applies well beyond pricing: **an unknown must be representable and
visible.** Coercing it to a plausible-looking default is how systems lie.

### 8. Compute at ingest or compute at query? — a real design decision

Plan §4 puts the cost engine in the collector, so the answer is "at ingest." Do not just implement
it — spend five minutes on why, because it is a genuinely two-sided question and a good interview
exchange:

| | Compute at ingest (chosen) | Compute at query |
|---|---|---|
| Query cost | one column read | join a price table on (model, valid_from) every time |
| Price correction | needs a backfill | automatic, just fix the table |
| Historical honesty | correct **only if** you stamp which price list was used | correct by construction |
| Ingest cost | a rational multiply per span — negligible | zero |

The chosen design survives its own weakness because of one column: **`pricing_version` on every
row.** A recomputed historical cost that silently uses today's price is a lie (plan §5); a stored
cost that records *which dated price list produced it* is auditable, and a backfill is a `WHERE
pricing_version = ...` away. That column is what makes ingest-time computation defensible rather
than merely fast.

---

# 1. State going in

From `v1-03` (or `v1-03a` + `v1-03b`), committed:

- Postgres 17 in Docker; `migrations/0001_spans.sql` with the partitioned `spans` table, including
  **nullable `cost_usd numeric(20,10)`, `pricing_version text` and `pricing_status text`** already
  in place — so this slice needs no schema change
- `cmd/collector` — OTLP/HTTP receiver on `:4318`, naive single-row synchronous writer
- `cmd/cpq` — `trace <hex>` prints a trace tree
- Measured ingest number in plan §15; plan §7's ingest SLO re-baselined
- The corrected `v1-07` prediction recorded

**Check first:** does the mock adapter return a nonzero `cache_read_input_tokens`? `v1-01` required
it. If it returns zeros, this slice has nothing to price and the first task is making the mock's
usage configurable per request — which is a small change to the in-process mock, not a build of
`cmd/mockllm` (that is `v1-05`).

---

# 2. What this slice adds

Tokens become dollars, exactly, with a dated and versioned price list, and roll-ups that can answer
"who spent what".

```
  pricing/
    prices-2026-08-22.yaml   <- dated, versioned, with source URL and retrieval date in comments
  internal/
    pricing/
      catalog.go             <- load + validate the file; fail fast at startup on a bad one
      calc.go                <- the four-field calculation in big.Rat
      calc_test.go           <- table-driven; this is the one place tests are non-negotiable
  internal/store/
    insert.go                <- writes cost_usd, pricing_version, pricing_status
  migrations/
    0002_cost_views.sql      <- roll-up views per tenant / route / model / day
  cmd/cpq/
    main.go                  <- + `cpq cost --by tenant|route|model --since ...`
```

Decisions, with recommendations:

- **File format: YAML.** A pricing file needs comments — where the numbers came from, when they were
  read, which page — and JSON cannot hold them. One dependency (`gopkg.in/yaml.v3`). If you would
  rather stay dependency-free, JSON with explicit `_source` and `_retrieved` string fields is
  acceptable; say which you chose and why.
- **The filename carries the date and the file carries a `version` string.** The version string goes
  into `pricing_version` on every span row. Prices change; a row must be able to say which list
  priced it.
- **Load and validate at startup, fail fast.** A malformed or empty price list should stop the
  collector from starting, not silently price everything at zero. Validate: every model has all four
  multipliers or an explicit inheritance rule, prices are positive, the version is non-empty.
- **Structure per model:** `input_per_mtok`, `output_per_mtok`, and the cache multipliers as
  *multipliers* (`cache_read: 0.1`, `cache_write_5m: 1.25`, `cache_write_1h: 2.0`) rather than as
  pre-multiplied prices. Multipliers are what the provider documents, and deriving prices from them
  keeps one source of truth.
- **The cache-write TTL ambiguity.** 1.25× and 2.0× apply to different TTLs, and at ingest we may
  not know which was requested. Carry a `gen_ai.request.cache_ttl`-style attribute when we know it;
  when we do not, **assume 5-minute (1.25×) and set `pricing_status = 'assumed_ttl'`** so the
  assumption is a visible column value rather than a buried default. Resolve it for real at `v1-06`
  against what the Anthropic API actually reports, and note here that it is unresolved.
- **`pricing_status`** is a small closed set: `ok`, `unknown_model`, `assumed_ttl`, `no_usage`, and
  — added at `v1-06` — `partial_stream`. The column was created back in `v1-03`, so no migration is
  needed here. If it was missed, the `initdb.d` trigger from `v1-03` fires and a real migration
  runner arrives now.
- **Roll-ups as plain SQL views** in v1. A materialised view or a rollup table is a v2 slice
  justified by a slow query, not by anticipation (plan §13, and plan §4a's whole argument).

---

# 3. Why now

- **Because it is a *dependency*, not a feature.** Plan §5 and §11: #7's dollar figure is
  unmeasurable without a cache-aware engine here. Plan §5 states outright that this is *"the reason
  cost accounting is in Phase 0 and not later."*
- **Because rows exist now and load does not yet.** Pricing arithmetic is much easier to verify
  against ten hand-checked rows than against a benchmark run. Do the exact arithmetic while the data
  is small enough to check with a calculator.
- **Because `v1-05` should measure a complete request path.** Cost computation on the ingest path is
  work; it belongs inside the baseline rather than being added after it and quietly moving the
  numbers.
- **Because `v1-06` spends real money and this is the checker.** Plan §7's cost-accuracy SLO is
  "computed USD within rounding of provider-reported". The computation must exist and be trusted
  before there is a provider invoice to compare against.

---

# 4. Done means

- [ ] `pricing/prices-2026-08-22.yaml` exists, dated, with a `version` string, source URL and
      retrieval date, and the three models from plan §5
- [ ] the collector **fails to start** on a malformed or missing price file, with a clear message
- [ ] `internal/pricing/calc.go` computes all four components in `big.Rat`; a grep shows **no**
      `float64` anywhere in the cost path
- [ ] `calc_test.go` is table-driven and covers: all four fields nonzero; cache-read-only;
      cache-write with each TTL; zero output; unknown model; missing usage entirely
- [ ] one hand-computed example is verified against the code — literally on paper, once
- [ ] every span row gets `cost_usd`, `pricing_version`, `pricing_status`
- [ ] an unknown model yields `cost_usd IS NULL` and `pricing_status = 'unknown_model'` — **never 0**
- [ ] roll-up views answer $/tenant, $/route, $/model, $/day, and report unpriced spans as a separate
      count rather than folding them in
- [ ] `cpq cost --by tenant` prints a readable table
- [ ] the naive-vs-correct comparison in §5 is measured and written down
- [ ] plan §15 has a `v1-04` row
- [ ] the cache-TTL assumption is recorded as an **open question to close at `v1-06`**

---

# 5. What to measure

Four numbers. The second is the one that matters to the portfolio.

1. **$/request and $/1,000 requests** for a representative workload, per model, from the roll-up
   views. Sanity-check one row by hand.

2. **The naive-vs-correct delta — the headline.** Compute **all three** over the same span data
   (Concepts §5), because the two naive engines err in opposite directions and reporting only one
   would be its own small dishonesty:
   - naive A: `input_tokens × price_in + output_tokens × price_out` → expect **under**-reporting
   - naive B: `(input + cache_read + cache_creation) × price_in + output × price_out` → expect
     **over**-reporting, up to ~10× on the prompt side
   - correct: all four fields with their multipliers

   Report each as a ratio to the correct figure, at several cache hit rates (0%, 50%, 85%). At 0%
   all three must agree exactly — if they do not, there is a bug, and that check is worth as much as
   the headline. Expect the gaps to widen sharply as the hit rate climbs. The sentence this produces
   — *"at an 85% cache hit rate a two-field cost model misreports spend by X% (under) or Y% (over)
   depending on which field it trusts, and neither reports any change when the cache hit rate
   improves"* — is a blog post and an interview answer, and it justifies plan §5's placement of this
   work in Phase 0.

3. **The #7 rehearsal.** Drive the same total token volume twice through the mock — once
   cache-hostile (everything counted as `input_tokens`), once cache-stable (most counted as
   `cache_read_input_tokens`) — and show the roll-up reporting the cost difference. This proves the
   engine can *detect* the optimisation #7 will make. It costs ₹0 and it is the whole reason this
   slice exists.
   **Be precise about what it proves:** it demonstrates the measurement instrument works, not that
   any caching happened. Say that in the write-up. Overclaiming here would be exactly the kind of
   thing AGENT_INSTRUCTIONS' honesty rules exist to prevent.

4. **The precision demonstration.** Sum the cost of ~1,000,000 synthetic spans in `float64` and in
   `big.Rat`, and report the divergence in dollars. It will be small. Report it anyway, with the
   observation that the error is *biased*, not random, so it does not cancel out — that is the
   actual argument for exact arithmetic, and "floats are imprecise" is not.

---

# 6. Out of scope

- **Budget enforcement, spend caps, throttling, alerts.** Plan §2: *"Not a policy engine."*
  Enforcement is #29 territory and Pass 2. This slice **reports**, it does not **govern**. Naming
  the difference between reporting and enforcement is worth a sentence in the README.
- **Real provider calls or real money.** `v1-06`. Everything here runs on mock usage numbers, ₹0.
- **A pricing API or auto-updating prices.** A dated file, edited by a human, is the correct design
  for something that must be auditable.
- **Multi-currency, INR conversion.** Store USD. Convert at display if ever needed; a stored,
  converted number carries a hidden exchange-rate date and becomes unauditable.
- **Materialised views, rollup tables, pre-aggregation.** v2, if a query is slow.
- **Cost attribution across a trace** (splitting one request's cost across nested spans). Interesting
  and not v1 — it needs the DAG work that belongs to **#30-S**.
- **Batching or queueing the writer.** Still deliberately synchronous. `v1-07`.
- **`/metrics`.** Lands at `v1-05`. Count unknown-model events in a plain counter for now and expose
  it next slice.

---

# Commit plan

1. `feat(pricing): dated price catalog with load-time validation`
2. `feat(pricing): cache-aware four-field cost calculation in exact arithmetic`
3. `test(pricing): table-driven cost cases including unknown model`
4. `feat(store): persist cost_usd, pricing_version, pricing_status`
5. `feat(migrations): cost roll-up views`
6. `feat(cmd/cpq): cost command`
7. `docs: naive vs cache-aware cost comparison`

---

# Hand-off — what to record in the plan file

- §15 row for `v1-04`: commit SHA, $/1k requests, **the naive-vs-correct delta at 85% cache hit**,
  the float-vs-exact divergence, the surprise.
- §12 open decisions: the cache-write TTL multiplier question, to be closed at `v1-06`.
- §11: confirm the #7 constraint is now satisfied — the engine can measure a cache-hit-rate change.
- If the pricing table in plan §5 turned out to be stale when checked against the provider's current
  page, **fix the plan file and say so.** Plan §5's own rule: a price that silently drifts makes
  every derived number a lie.

---

# Next

`v1-05` — the mock LLM server and the load generator. **The baseline.** Percentiles done honestly,
open-loop load, streaming, `/metrics`, and the first real test of every SLO in plan §7.

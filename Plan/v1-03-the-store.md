# Slice v1-03 — The store

**Version:** v1 · **Slice:** 03 of 08 · **Produces a number?** Yes — and it **re-baselines an SLO**.

> Read `..\..\Shared Context\control_plane_thin_plan.md` §5 (schema), §7 (SLOs) and §14 (the
> ingest SLO is flagged as a guess) before answering anything.
>
> **This is the largest slice in v1 and it has a planned split point — see §7. Splitting it is
> normal, not failure** (plan §14).
>
> **This slice deliberately builds the naive version.** Read §3 and Concepts §7 before writing a
> line of the writer, and do not "helpfully" improve it.

---

# Concepts for this slice

Plan §8: *"`v1-03` opens with why a write-heavy append-only table is a different database problem
from CRUD."* That is the frame; everything else hangs off it.

### 1. Append-only is a different database problem

Every backend instinct you have is tuned for CRUD: a row is created, read many times, updated
occasionally, deleted eventually. Spans are not that. A span is written **once**, never updated,
never deleted individually, read rarely, and dropped **by the million** when it ages out.

What changes when the workload flips:

| | CRUD table | Span table |
|---|---|---|
| Dominant operation | read + update | **insert** |
| What you optimise | read latency, index coverage | **write throughput** |
| Indexes | add freely, they only help | each one is a **tax on every insert** |
| Deletes | `DELETE ... WHERE id = ?` | `DROP TABLE` (a whole partition) |
| Vacuum | reclaims space from updates | almost nothing to reclaim; you still need it for the visibility map |
| `fillfactor` | leave room for in-page updates | pack pages full — no updates are coming |

The one-line version, and it is the interview answer: **in an append-only workload every index you
add is paid for on the write path forever, and read patterns must justify their cost in writes.**

### 2. Indexes are the write budget

A B-tree insert is not free: locate the leaf, maybe split it, write the WAL record. Multiply by the
number of indexes. Three indexes means roughly three times the index maintenance per row plus the
extra WAL.

We know two query patterns for certain and should resist inventing more:

- **"Show me trace X"** → index on `trace_id`. Non-negotiable; it is the query path in this slice
  and the thing `v1-08` demos.
- **"Show me tenant T's spans over a time range"** → `(tenant_id, start_time)`. This is the cost
  roll-up at `v1-04` and the drill-down that plan §11 inherits from **#30-S**.

Everything else — route, model, error type — waits until a query actually exists and is slow. Plan
§4a's spirit applies to indexes too: build the wall, measure it, then move.

**There is a cheap experiment here and it is worth doing:** run the ingest measurement with the
indexes and again with them dropped. The delta is the write budget those indexes cost, in a number
you measured on your own machine. That is a far better answer than "indexes slow down writes."

### 3. Time-ordered keys are kind to B-trees, and random ones are not

`start_time` increases monotonically, so inserts land at the right-hand edge of that index — the
same few pages stay hot in the buffer cache and there are almost no page splits in the middle.

`trace_id` is 16 random bytes, so its index is the opposite: every insert lands in a random leaf,
which means random I/O and constant page splits. This is exactly why UUIDv4 primary keys became
notorious in high-insert tables, and why UUIDv7 (time-ordered) was invented.

We cannot fix this — trace IDs are random by specification, and we need to look traces up. But
knowing *which* index is the expensive one tells you where the ceiling comes from, and it is the
first thing to point at when the number in §5 disappoints.

### 4. Declarative range partitioning, and the constraint that surprises people

Plan §5 and §12.2: partition `spans` by range on `start_time` from the very first migration. Cheap
now, painful to retrofit.

Three things it buys:

1. **Retention is `DROP TABLE`.** Deleting a month of spans with `DELETE` writes a WAL record per
   row, bloats the table, and gives the space back only after a vacuum. Dropping a partition is a
   catalogue operation — effectively instant.
2. **Partition pruning.** A query with `WHERE start_time BETWEEN ...` never touches the other
   partitions' indexes at all.
3. **Smaller indexes per partition**, which keeps the hot one in memory.

**The constraint that catches everyone:** a primary key or unique constraint on a partitioned table
**must include the partition key**. You cannot have `PRIMARY KEY (span_id)` on a table partitioned
by `start_time`. It becomes `PRIMARY KEY (start_time, span_id)` — and you should think about whether
you need the uniqueness at all, because it costs a unique index on every partition. (Spans can
legitimately arrive twice: the OTel exporter retries on a timeout when the write actually
succeeded.) Decide this in the session; there is a recommendation in §2.

**And the one from Concepts §1 of `v1-02`, restated because it matters here:** there must be
**no foreign key** from `parent_span_id` to `span_id`. Parents arrive *after* their children. An FK
would look correct and would reject most of your data. If a session ever proposes it, that is the
moment to explain why the tree is reconstructed at read time, not enforced at write time.

### 5. OTLP on the receiving end

The gateway has been speaking OTLP since `v1-02`; now we implement the other end.

- **Transport:** HTTP POST to `/v1/traces` on `:4318`. Body is a protobuf-encoded
  `ExportTraceServiceRequest`. Decode with the generated types in
  `go.opentelemetry.io/proto/otlp/collector/trace/v1` — never hand-roll protobuf.
- **Also accept `Content-Type: application/json`.** OTLP/HTTP officially supports a JSON encoding,
  and being able to `curl` a span into your own collector while debugging is worth the twenty lines.
  This is the reason plan §12.3 chose HTTP over gRPC first.
- **Nesting:** the payload is `ResourceSpans → ScopeSpans → Span`. Resource attributes
  (`service.name`) live at the top and apply to every span underneath — so flattening one request
  into rows means copying resource attributes down. Miss that and every span in your table has a
  null service name.
- **The response is not just 200.** OTLP defines a partial-success shape: accept what you can,
  report `rejected_spans` and a message. That distinction is real — "I took 480 of your 500 spans"
  is information the sender can act on; a bare 200 is a lie and a 500 makes them resend all 500.
- **Which status code you return is a fail-open/fail-closed decision** (`v1-00` §5.4), made at the
  ingest boundary:
  - `503` + `Retry-After` → the sender retries and buffers. Good for a brief blip; a queue-building
    machine during a long outage.
  - `400` → the sender drops permanently. Correct for genuinely malformed payloads, catastrophic if
    returned for a transient store failure.
  - Getting this backwards is a real production incident shape, and this slice will demonstrate it
    at `v1-07`. Choose deliberately and write the reason in a comment.

### 6. Representing IDs and timestamps without quietly losing data

Three encoding decisions where the obvious choice is wrong:

- **IDs as `bytea`, not `text`.** A trace ID is 16 raw bytes. Stored as hex text it is 32 bytes plus
  header, and every index page holds half as many entries. `bytea` also makes "is this the same
  trace" a byte comparison instead of a case-sensitivity question. The cost is that `psql` output is
  ugly — worth it, and `encode(trace_id,'hex')` fixes it in the query path.
- **`timestamptz` is microsecond precision; OTLP timestamps are unix nanoseconds.** Storing
  `start_time` as `timestamptz` **truncates**. That is fine for ordering and querying, and it is not
  fine as the only record of duration. So store `duration_ns bigint` computed from the raw
  nanosecond values, not from the truncated timestamps. Plan §5's schema already has this column;
  this is *why* it is there.
- **`cost_usd numeric(20,10)`, never float.** Plan §5: money in floating point is a bug waiting for
  a demo. The column lands in this slice's migration and stays NULL until `v1-04`.

### 7. Why the writer is synchronous, and why that is not an oversight

**Read this before writing the writer.**

The writer in this slice does the simplest thing: decode the payload, then execute one `INSERT` per
span, inside the HTTP handler, and return 200 only after the last row lands. No batching. No queue.
No worker pool. No `COPY`.

That is not laziness and it is not a bug to be fixed in review. Plan §7 arranges it deliberately:
`v1-07` kills Postgres under load and measures what breaks, then fixes it and measures again. **The
before-number is the deliverable.** Build the good version now and there is nothing to compare
against, and the honest failure mode — deliverable #3 of four (plan §7) — never exists.

Plan §7 says it in as many words: *"Do not let a later session 'helpfully' skip ahead and implement
the queue in slice 03."* This is that later session. Do not.

**An honest correction to the plan, to be recorded in plan §15 or §14 during this session.** Plan §7
predicts that when Postgres dies *"the gateway blocks on the write, connections pile up, and a
telemetry outage becomes a user-facing outage."* That prediction is not quite right, and the reason
is `v1-02`: the gateway exports through a `BatchSpanProcessor`, which is asynchronous. The gateway
will **not** block on Postgres. What will actually happen, and what `v1-07` should be set up to
measure, is:

1. The collector's handler blocks or errors on every insert; ingest capacity goes to roughly zero.
2. The gateway's batch exporter retries in the background, its bounded queue fills, and it starts
   **dropping spans silently** — no error, no counter (`v1-02` Concepts §4).
3. User traffic through Door B is probably unaffected — which is the *correct* behaviour arrived at
   by accident, and being able to say that out loud is worth more than pretending we designed it.
4. Any Door A tenant using a `SimpleSpanProcessor` — the naive default in a small script, i.e.
   exactly what `v1-08` builds — **does** block on its own request path.

So the real finding is likely to be *silent, uncounted telemetry loss with no way to know how much*,
rather than a user-facing outage. That is a better finding, not a worse one, and it still motivates
exactly the same fix (bounded queue with an explicit drop policy and a `spans_dropped_total`
counter). **Write the prediction down now, in this session, before `v1-07` runs.** A prediction
recorded in advance and then confirmed or destroyed is the entire value of the exercise.

---

# 1. State going in

From `v1-02`, committed:

- `pkg/genai/` — semconv constants, `Usage`, Door B DTOs, error taxonomy; imports nothing internal
- `internal/telemetry/` — OTel TracerProvider, batch processor, `otlptracehttp` exporter
- `internal/gateway/handler.go` — SERVER span + CLIENT child span, `traceparent` extraction
- `deploy/docker-compose.yml` + `deploy/otel-collector.yml` — the **reference** collector
- `CP_TELEMETRY=off` supported
- Plan §15 has a `v1-02` row with bytes/span; plan §12.4 (Go vs TypeScript) is **resolved**
- A written prediction for ingest throughput

**If plan §12.4 is still open, stop and resolve it first.** This slice writes several hundred more
lines in whichever language won, and reversing after it is much more expensive than reversing now.

**Docker Desktop must be running.** `docker version` must respond before anything else.

---

# 2. What this slice adds

Spans stop being printed and start being stored, in our own collector, in Postgres, queryably.

```
  deploy/
    docker-compose.yml      <- + postgres:17 service, named volume, healthcheck
  migrations/
    0001_spans.sql          <- partitioned table, indexes, partitions for the current window
    0002_partition_helper.sql <- function/proc to create the next partition
  internal/
    store/
      store.go              <- pgxpool wiring, Span row type
      insert.go             <- the deliberately naive single-row writer
      query.go              <- one-trace fetch, ordered for tree assembly
    otlp/
      receiver.go           <- POST /v1/traces, protobuf + JSON decode
      convert.go            <- ResourceSpans -> []store.Span, resource attrs flattened down
  cmd/
    collector/
      main.go               <- :4318 receiver + pool + shutdown
    cpq/
      main.go               <- query CLI: `cpq trace <hex-id>` prints the tree
```

Decisions to make, with recommendations:

- **Driver: `pgx/v5` with `pgxpool`, not `database/sql`.** Reasons, in order: native `bytea` and
  `numeric` handling (we have both, and `numeric` through `database/sql` is a string-shaped
  nuisance); `CopyFrom` is right there for when `v1-07`/v2 needs bulk ingest; and the pool exposes
  the stats we want at `v1-07` (acquire count, wait duration) — pool exhaustion is one of the
  failure modes we are hunting.
- **Migrations: numbered `.sql` files mounted into `/docker-entrypoint-initdb.d`.** Simple, no extra
  Windows binary, no library. **State the limitation out loud, because it is a real one:** those
  scripts run only when the data volume is empty. A schema change after that means
  `docker compose down -v` and losing local data, which is acceptable now and not acceptable the
  moment a measured baseline lives in the database. **The trigger to switch to a real migration
  runner is the first schema change made against data we care about.** Write that trigger in the
  README so it is a decision, not a discovery.
- **Include all three cost columns in `0001` as nullable** — `cost_usd numeric(20,10)`,
  `pricing_version text`, `pricing_status text` — even though `v1-04` is what fills them. It costs
  nothing now, and it means the cost slice is a pure code change rather than a schema change against
  the `initdb.d` constraint just described.
- **Primary key: `(start_time, span_id)`** — the partition key must be included (Concepts §4).
  Consider whether you want the uniqueness at all: it costs a unique index on every partition, and
  retried exports can legitimately deliver the same span twice. Recommendation: **keep it**, and use
  `ON CONFLICT DO NOTHING` so a retry is idempotent rather than an error. Dedup at ingest is worth
  one index; discovering duplicate spans in a cost roll-up later is not.
- **Partitions: monthly**, with the current and next month created in `0001`. Daily partitions are
  the right answer at production volume and are pointless overhead here. Say which one you chose
  and why, because "how do you pick partition granularity" is a common interview question and the
  answer is retention period ÷ acceptable number of partitions.
- **`cmd/cpq` is a fifth binary, which plan §3's repo layout does not list.** That is a deliberate
  addition — record it in the plan file rather than letting the layout quietly drift.

---

# 3. Why now

- **Because `v1-04` prices rows, and there are no rows.** Cost that is computed and printed is a
  demo. Cost that is stored, dimensioned and roll-up-able is a control plane.
- **Because `v1-05`'s baseline needs somewhere for load to land.** Benchmarking a gateway that
  discards its telemetry measures the wrong system — the store is on the path, so it must be in the
  measurement.
- **Because the SLO in plan §7 is admitted to be a guess** (plan §14: *"Guessed, not computed...
  re-baseline this SLO there"*). This is "there." Until this slice runs, "≥ 2,000 spans/sec" is a
  number someone typed.
- **Because the failure mode of `v1-07` is built here.** The synchronous writer is the fault under
  test. No writer, no fault, no deliverable #3.
- **Because partitioning is cheap now and a migration later.** Plan §12.2 flagged it as a decision
  to confirm at this slice. Confirm it, in the first migration.

---

# 4. Done means

- [ ] `docker compose up -d` brings up Postgres 17 with a healthcheck and a named volume
- [ ] `docker compose down -v` is written in the README **next to** the `up` command (CLAUDE.md §6)
- [ ] `migrations/0001_spans.sql` creates `spans` partitioned by range on `start_time`, with the
      promoted columns from plan §5, `attributes jsonb` for the long tail, and partitions covering
      today
- [ ] **no foreign key** on `parent_span_id` — and the session can say why in one sentence
- [ ] `cmd/collector` listens on `:4318` and accepts `POST /v1/traces` as protobuf **and** as JSON
- [ ] resource attributes (`service.name`) are flattened onto every span row
- [ ] the OTLP response carries partial-success information when some spans are rejected
- [ ] the chosen HTTP status for a store failure is deliberate and the reason is in a comment
- [ ] the gateway's `OTEL_EXPORTER_OTLP_ENDPOINT` points at **our** collector; the reference collector is stopped
- [ ] one `POST /v1/chat` produces **two rows** in `spans`, sharing `trace_id`, with the client
      row's `parent_span_id` matching the server row's `span_id`
- [ ] `cpq trace <hex>` prints the tree, indented, with durations and token counts
- [ ] `cpq` shows **added latency** for a request as `parent.duration_ns - child.duration_ns` —
      the `v1-01` subtraction, now a query
- [ ] the writer is **one INSERT per span, inside the handler** — verified by reading the code, and
      a comment says why
- [ ] the corrected `v1-07` prediction (Concepts §7) is written into the plan file
- [ ] `go build ./... && go vet ./...` clean
- [ ] plan §15 has a `v1-03` row; plan §7's ingest SLO is **re-baselined** with the measured number
      and the original guess kept visible beside it

---

# 5. What to measure

This is the slice where the SLO table stops being aspirational. Four numbers.

1. **Single-span write latency, p50 / p95 / p99.** Time the insert alone, inside the collector, over
   ≥ 10,000 spans. Not the handler — the insert, so the number is attributable.

2. **Ingest ceiling, spans/sec.** A crude driver is correct here: a Go loop or a shell loop firing
   OTLP payloads as fast as it can. **This is not `loadgen`** — that is `v1-05` and it must not be
   started here. Report: spans sent, rows landed, wall time, spans/sec, and any difference between
   sent and landed (which is itself a finding).
   - Vary the batch size *inside one OTLP payload* (1, 10, 100, 500 spans per request). Per-span
     cost falls sharply with payload size even though each span is still its own INSERT, because
     HTTP and decode overhead amortise. That curve is a good graph and a good explanation.
   - **Compare against the prediction written at `v1-02`.** Being wrong is fine and interesting;
     not having predicted is the failure.

3. **The index tax.** Re-run measurement 2 with the non-essential indexes dropped. Report the delta
   as a percentage. This is a five-minute experiment that produces a genuinely portfolio-grade
   sentence.

4. **One-trace query latency.** `cpq trace <id>` against a table with ≥ 100k spans in it, p50 and
   p95. If it is slow, `EXPLAIN (ANALYZE, BUFFERS)` it and find out whether the planner is pruning
   partitions — that check is the point of the exercise as much as the number is.

**Then re-baseline the SLO** in plan §7, in this shape:

> Ingest capacity — original SLO: ≥ 2,000 spans/sec (guessed, plan §14).
> Measured `v1-03`, naive single-row synchronous writer: **N spans/sec** on *machine, date, commit*.
> Revised v1 SLO: **M spans/sec**, with the original kept visible.

**Do not delete the original guess.** Plan §14: *"A missed SLO quietly edited afterwards is the
thing that makes a portfolio worthless."*

---

# 6. Out of scope

- **Batching, queueing, worker pools, `COPY`, prepared-statement pipelining.** Concepts §7. This is
  the hardest out-of-scope line in v1 to obey and the most important.
- **Cost calculation.** The columns exist and stay NULL. `v1-04`.
- **`cmd/loadgen`, percentile machinery, `bench/`, `/metrics`.** `v1-05`. The crude driver here is
  throwaway and should look throwaway.
- **Fault injection.** `v1-07`. Do not kill Postgres in this session beyond confirming the app does
  not panic on a dead pool at startup.
- **Retention jobs, partition automation, a cron to create next month's partition.** Create the
  partitions this slice needs by hand; automation without a retention policy is decoration.
- **A GIN index on `attributes`.** Expensive on write, and no query needs it yet. Note it as
  available and move on.
- **ClickHouse, columnar storage, compression.** Plan §13 — v2, justified by the wall this slice
  starts measuring.
- **A UI, a dashboard, Grafana.** Plan §2: a query CLI and SQL.
- **Auth on the OTLP endpoint.** Localhost only in v1. Say so in the README rather than implying a
  security property that does not exist.

---

# 7. The split point — read this before you start

This slice is two sessions' worth of work if anything goes sideways. The split is pre-declared so
that stopping is a plan rather than a failure:

**`v1-03a` — spans land.** Compose + Postgres, `0001_spans.sql`, the OTLP receiver, the naive
writer. Done when one `POST /v1/chat` produces two correct rows and `psql` can see them.

**`v1-03b` — spans are useful.** `cmd/cpq`, the tree query, partition-pruning check, the four
measurements in §5, the SLO re-baseline.

If the session hits the end of 03a with energy left, keep going. If protobuf decoding, `bytea`
handling, or Docker on Windows eats an hour, **stop at 03a, commit, write the §15 row saying 03a
landed, and start a fresh chat for 03b.** A fresh context is cheaper than a tired one — that is the
whole reason slices exist (plan §3).

Likely time sinks, so they are recognised rather than debugged from scratch:
- Docker Desktop on Windows and the space in `D:\AI Engineering\AI Projects\...` — use relative
  paths inside `docker-compose.yml` and quote everything in PowerShell.
- `initdb.d` scripts silently not running because the volume already exists → `down -v` and retry.
- `bytea` round-tripping and `encode(...,'hex')` in the query path.
- The unix-nanos → `timestamptz` truncation in Concepts §6, which shows up as durations that are
  subtly wrong rather than as an error.

---

# Commit plan

1. `chore(deploy): postgres 17 service with healthcheck and volume`
2. `feat(migrations): partitioned spans table with promoted genai columns`
3. `feat(store): pgx pool and single-row span writer`
4. `feat(otlp): protobuf and json receiver on :4318`
5. `feat(otlp): resource attribute flattening and span conversion`
6. `feat(cmd/collector): wire receiver to store`
7. `feat(cmd/cpq): trace tree query`
8. `docs: ingest measurement and SLO re-baseline`

---

# Hand-off — what to record in the plan file

- §15 row for `v1-03`: commit SHA, spans/sec, write p95, query p95, index-tax delta, the surprise.
- **§7 SLO table:** ingest row re-baselined, original guess preserved.
- **§14:** the corrected `v1-07` prediction from Concepts §7, written as a prediction.
- **§3 repo layout:** note the added `cmd/cpq`.
- **§12.2:** partitioning confirmed, granularity and reason recorded.
- The migration-runner trigger condition, so the next schema change does not become an accident.

---

# Next

`v1-04` — cost accounting. Dated pricing config, the four-field cache-aware calculation, exact
arithmetic instead of floats, and the demonstration that this engine can actually measure what
project #7 will later claim.

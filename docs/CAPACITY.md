# Capacity: what one process carries

> How to measure the engine on your own database with `npm run loadtest`, the numbers
> measured on the demo, and how to size an installation from them.

The time of an answer has two parts. The **model** takes seconds and depends on the provider
and the model you chose. The **engine** (limits, validation, the query on your database, the
Answer Certificate and its verification queries, the audit trail) takes milliseconds and
depends on your database. The load test measures the second part and lets you set the first.

## Run it

```bash
npm run loadtest                                   # 1, 2, 4 and 8 questions in flight, 100 each
npm run loadtest -- --levels 1,4,16,32 --requests 400
npm run loadtest -- --model-latency-ms 1500        # with a realistic model round trip
npm run loadtest -- --json --out capacity.json     # machine-readable report
```

What it does:

1. Starts the real server as a separate process, on its own port (`--port`, default 3399),
   with the concurrency limit set to the highest level of the run and the hourly and daily
   quotas lifted for that process only.
2. Replaces the model provider with a local stand-in. It answers each question with the
   reference SQL of your evaluation bank: `eval/bank.json` plus the approved queries that take
   no parameter (`docs/EVALUATION.md`). No request leaves the machine and nothing is billed.
3. Sends the questions over HTTP at each level of concurrency and reports answers per second
   and the latency at the 50th, 95th and 99th percentile, with the memory of the server.
4. Sends twice the limit at the end, to show the process refusing the excess instead of
   queueing it.

Every request crosses the whole engine, so the result reflects your schema, your queries and
your database server. It writes audit rows like any question, under the user `loadtest`:
point it at a staging installation, not at the production audit trail.

It exits with 1 when a level left questions unanswered and with 2 when the run could not
start, so it can gate a deployment.

## What was measured on the demo

One process on a laptop (Apple M4, 10 cores, Node 24), MySQL 8.4 in a local container, the
demo database (11 tables, a few thousand rows) and its bank of 10 questions. Each answer made
two model calls and ran its query plus the certificate's verification queries. Measured on
2026-11-12 with version 1.8.0.

**Engine alone** (model time set to zero), 400 questions per level:

| In flight | Answered | Per second | p50 | p95 | p99 |
|---|---|---|---|---|---|
| 1 | 400 of 400 | 147 | 6 ms | 11 ms | 15 ms |
| 4 | 400 of 400 | 427 | 9 ms | 15 ms | 18 ms |
| 8 | 400 of 400 | 504 | 15 ms | 23 ms | 29 ms |
| 16 | 400 of 400 | 540 | 28 ms | 42 ms | 49 ms |
| 32 | 400 of 400 | 620 | 49 ms | 63 ms | 131 ms |

**With a model round trip of 1.5 seconds** (two calls per answer, so 3 seconds of model
time), 120 questions per level:

| In flight | Answered | Per second | p50 | p95 | p99 |
|---|---|---|---|---|---|
| 4 | 120 of 120 | 1.3 | 3029 ms | 3044 ms | 3050 ms |
| 16 | 120 of 120 | 5.0 | 3031 ms | 3065 ms | 3072 ms |
| 32 | 120 of 120 | 9.7 | 3075 ms | 3104 ms | 3112 ms |

**Over the limit** (64 in flight against a limit of 32): the excess was refused at once with
`503` and `retryAfter`, and the answers that were accepted kept the same latency. Nothing
queued inside the process.

**Memory** stayed between 370 and 440 MB over 12 000 consecutive questions at 8 in flight,
with no upward trend.

What these numbers say, and what they do not:

- On this data the engine adds about 6 to 30 milliseconds to an answer. The model's seconds
  are the answer time; the engine is not the bottleneck.
- The demo database is small. On a large schema the query itself and the certificate's
  verification queries take longer, which is exactly why the test should be run on yours.
- A laptop with the database on the same machine has no network between the two. Expect
  the engine figure to grow by your round trip to the database, a few times per answer.
- The stand-in model returns short replies instantly. Real answers carry more tokens; that
  cost is on the provider's side and is not part of this measurement.

## Sizing an installation

The number of answers a process gives per second is close to

```
questions in flight / seconds per answer
```

and the seconds per answer are almost entirely the model. With a model that takes 3 seconds
per answer, 16 in flight gives about 5 answers per second, which is 18 000 per hour.

Two settings bound it:

- **`AI_CONCURRENCY_LIMIT`** (default 3) is how many questions one process works on at the
  same time; beyond it the process answers `503` at once. The default is conservative because
  every question in flight holds a model call and can hold a database connection. Raise it to
  what your provider's rate limit and your database accept. It counts **per process**.
- **`AI_MAX_QUERIES_PER_HOUR`** and **`AI_MAX_QUERIES_PER_DAY`** (defaults 100 and 1000) are
  quotas per caller. By default each process counts them on its own.

### More than one process

Put the replicas behind a load balancer and set two things on all of them:

- **`ROSIE_HANDLE_SECRET`**, the same value everywhere, so a follow-up can land on any replica.
- **`ROSIE_RATE_LIMIT_STORE=database`**, so the hourly and daily quotas are one number for the
  installation instead of one per replica. The count is kept in the engine database, in the
  table `rosie_rate_limit` that the schema file creates (`db/schema.sql` and its PostgreSQL and
  SQL Server versions; running the file again after an upgrade only creates what is missing).
  The account of the main pool needs UPDATE and DELETE on that table, on top of the SELECT
  and INSERT it has on the engine database: the counters are incremented and old windows
  removed. On MySQL: `GRANT UPDATE, DELETE ON rosie.rosie_rate_limit TO 'rosie_app'@'%';`
  (`docs/INSTALL.md`, Step 3; `docs/POSTGRES.md` and `docs/SQLSERVER.md` for the other
  engines). `npm run doctor` says whether the table is there. It does not try to write to
  it: without the grant, doctor passes and each process logs `rate_limit_store_unavailable`
  and counts on its own.

How the shared quota behaves:

- Windows are aligned to the clock, the hour and the UTC day, so every replica agrees on
  where a window starts. A caller who used the quota at 10:59 has a fresh one at 11:00.
- The stored count is exact under concurrency. It was tested with forty simultaneous
  requests split over two replicas on MySQL, PostgreSQL and SQL Server, and with two real
  server processes sharing a quota of six: the seventh request was refused by either one.
- It costs two or three small statements on the main pool per request.
- If the table cannot be reached, each process counts in its own memory and writes
  `rate_limit_store_unavailable` to the log once, then `rate_limit_store_recovered` when the
  table answers again. Quotas are looser for that while; the assistant keeps answering.
- Rows of past windows are removed by the engine. The table holds the client address, the
  same key the quota uses, for about two windows.

The concurrency limit stays per process on purpose: it protects the process, and capacity is
meant to grow with replicas. The MCP transport follows the same setting: its quotas per
caller are shared too, under their own counters, so the two front doors still count apart.

What else is per process, with more than one:

- **The context.** Each replica holds its own copy and needs its own reload; compare
  `rosie_context_info` across them (`docs/OPERATIONS.md`).
- **The counters of `/metrics`.** Scrape each replica (`docs/MONITORING.md`).
- **The audit file.** Each process writes the file at its own `ROSIE_AUDIT_FILE`. The
  database tables are shared; the file is the local copy of what that replica answered.
- **The "looked at" list of the console.** It is `review-dismissed.json`, kept beside the
  audit file of the replica that received the click. Replicas that do not share that
  directory show different review queues.
- **Explanations.** The text of an explanation is remembered in the process that produced
  it. The same request landing on another replica costs one more model call and nothing
  else.
- **`TRUST_PROXY`.** Count the load balancer among the hops, or every user shares one
  client address and one quota (`docs/INSTALL.md`, Step 6d).

People do not ask continuously. A person reading an answer and typing the next question
sends one every minute or two at most, so a process that carries 5 answers per second serves
several hundred people working at the same time. The limit that arrives first in practice is
the provider's tokens-per-minute quota, not this process.

## Reading a result that looks wrong

| What you see | Usual cause |
|---|---|
| `No warm-up question was answered` | The token, the database or the security hold. The last server output is printed under the message |
| `200 without an answer` at a level | The reference SQL of a bank entry no longer passes the validator or fails on the database. `npm run evaluate -- --list` names the entries |
| `503` inside a level (not the last line) | Something else is holding the process's slots, or the database pool is exhausted |
| p99 far above p95 at the top level | The database is saturating: the level is past what it carries comfortably |
| Memory that keeps rising across equal levels | Report it. Run the same level several times (`--levels 8,8,8,8`) and send the table |

# Evaluation

The numbers in [BENCHMARK.md](BENCHMARK.md) are about a sample database. The number that
matters to you is how Rosie does on **your** questions, on **your** database, with the model
you chose, and whether that number moved after you edited the context or changed the model.
`npm run evaluate` measures it.

## The question bank

A question in the bank is a question in plain language and a statement whose result **is**
the right answer. There are two sources, and they are combined.

**1. `eval/bank.json` in the context directory:**

```json
[
  { "id": "revenue-2025", "question": "What was the revenue in 2025?",
    "sql": "SELECT SUM(total) FROM invoice WHERE invoice_date >= '2025-01-01' AND invoice_date < '2026-01-01'" },
  { "id": "top-5-customers", "question": "Who are the top 5 customers by revenue?", "ordered": true,
    "sql": "SELECT c.name, SUM(i.total) FROM invoice i JOIN customer c ON c.id = i.customer_id GROUP BY c.name ORDER BY 2 DESC LIMIT 5" }
]
```

| Field | Meaning |
|---|---|
| `id` | A short name, unique in the bank. It is how a regression is reported |
| `question` | What a person would type |
| `sql` | The reference statement, written or checked by someone who knows the data |
| `ordered` | `true` when the order of the rows is part of the answer (a ranking). Default `false` |

**2. Approved queries.** Every approved query without parameters
([APPROVED_QUERIES.md](APPROVED_QUERIES.md)) joins the bank by itself, with its question and
its statement as the reference. A person already signed that pair. So the review cycle feeds
the evaluation with no extra work: a wrong answer is corrected, approved, and from then on
it is also a test.

## Two sets, two figures

Every question is placed in one of two sets, and the report gives an accuracy for each.

| Set | Which questions | What its accuracy tells you |
|---|---|---|
| Already in the context | Approved queries, and any bank question whose text is also a question of `verified-queries.json` | That what you taught still works. It should be at or near 100%, and it says nothing about a question nobody wrote down |
| Held out | Every other bank question | How the engine does on a question it meets for the first time. This is the figure to quote |

One accuracy over both reads higher the more of the bank is copied from the examples, which
is why they are never merged in the report. Keep part of the bank out of the examples on
purpose: when a held-out question is answered wrong and you approve its correction, it moves
to the first set, so add a new held-out question in its place. `npm run evaluate -- --list`
shows the set of each question.

A bank entry may also carry `"set": "held_out"` or `"set": "regression"` to say where it
belongs, and `"expectEmpty": true` when no rows IS the right answer. Without that mark, a
reference that returns no rows is not scored: two unrelated statements both return nothing.
A question that is a near copy of a taught one ("What is our total revenue?" next to "What
is the total revenue?") counts as taught.

How results are compared: a reference of a single figure is matched by an answer of a single
figure, or by the answer's column of the same name. A value with leading zeros (`000123`) is
a code and is compared as text.

Thirty to fifty questions that your teams really ask are worth more than five hundred
invented ones. Start with the questions of the pilot.

## Running it

```bash
npm run evaluate -- --list           # show the bank and its problems; no model call
npm run evaluate                     # every question
npm run evaluate -- --sample 30      # 30 questions, the same 30 all day
npm run evaluate -- --id revenue-2025
npm run evaluate -- --min 85         # exit 1 when accuracy is under 85%
npm run evaluate -- --min-held-out 70  # the same, over the questions the context does not teach
npm run evaluate -- --json           # the whole run as JSON
```

With the Docker image:

```bash
docker run --rm --env-file rosie-env/.env \
  -v "$PWD/rosie-context:/context:ro" -v "$PWD/rosie-eval:/eval" -e ROSIE_EVAL_DIR=/eval \
  <image> npm run -s evaluate
```

Each question is asked the way a user asks it, so it costs what a user's question costs. The
run ends with the tokens used.

```
Evaluation of sales, 2026-11-12 00:18 UTC, model gpt-6-luna

  Questions asked        42
  Matched the reference  37 of 42  (88.1%)
  Returned other rows    4
  Returned no result     1

  Of the 4 wrong answer(s): 2 flagged by the certificate, 1 marked "not checked", 1 with no warning
  Right answers flagged anyway: 1
  Answered through approved queries: 9 of 9 matched
  Answered through compiled metrics: 6 of 6 matched
  Answered through SQL written by the model: 22 of 27 matched

Not matched:
  refunds-by-month  [mismatch, certificate FLAGGED]
      Refunds by month in 2025
      the answer has 12 row(s), the reference has 11

REGRESSIONS (1): matched the last time, not now
  refunds-by-month  Refunds by month in 2025
```

## How an answer is judged

By its **result**, never by its SQL text. The reference statement runs through the same
validator and the same read-only connection as any answer, and the two results are compared:

- the same number of rows;
- every column of the reference is present in the answer with the same values, row by row;
- column names and column order are ignored, and the answer may have extra columns (a name
  next to the id the reference returns);
- row order is ignored unless the entry says `"ordered": true`;
- `100`, `100.0` and `"100.00"` are one value, numbers are compared to 9 significant digits,
  and a midnight timestamp equals its date.

| Outcome | Meaning | Counts in accuracy |
|---|---|---|
| `match` | The answer returned the reference rows | yes, as right |
| `mismatch` | It returned other rows | yes, as wrong |
| `no_answer` | It asked a clarifying question, refused or failed | yes, as wrong |
| `inconclusive` | A result was cut at the row cap, so rows cannot be compared. Aggregate or narrow the question | no |
| `reference_failed` | The reference statement did not run. A problem in the bank | no |

## What the report tells you

- **Accuracy by result** on your own questions.
- **What the certificate did with the wrong answers.** Flagged, marked "not checked", or
  sent with no warning. The last group is the one to read first: those are the errors a user
  would have had no reason to doubt. This is the measured version, on your data, of what
  [CERTIFICATE.md](CERTIFICATE.md) says the checks can and cannot catch.
- **False alarms.** Right answers that were flagged anyway.
- **Which path answered.** Approved queries and compiled metrics should match every time;
  when the share answered by them grows, accuracy follows.
- **Regressions.** Questions that matched the last time they ran and do not now. The state is
  kept per question, so a sampled run is compared with what each of its questions did
  before, whenever that was.

## The gate

The command exits 0 only when the run measured what it was asked to guarantee. It exits 1,
and says why under `NOT PASSED`, when:

- a question that matched the last time does not match now;
- accuracy is under `--min`, or held-out accuracy is under `--min-held-out`;
- a threshold was asked and the figure **could not be measured** (nothing scored, or no
  held-out question in the run);
- fewer than 90% of the questions asked could be scored (`--min-coverage` changes it):
  accuracy over the few that could says nothing about the rest;
- a question that matched the last time cannot be measured now (its result is cut at the
  row cap, or its reference stopped running);
- a reference statement did not run. That question was not measured, so the bank has to be
  fixed before the run can pass;
- the bank has a problem (an entry without `sql`, a repeated `id`, a file that is not JSON):
  the entry concerned was not asked;
- the run could not be saved, because the next run would have nothing to compare with;
- `--min` or `--min-held-out` is not a percentage. A threshold that cannot be read stops
  the run before any question is asked.

Run it before putting a changed context or a new model in service.

## Running it every night

```cron
0 3 * * *  docker run --rm --env-file /opt/rosie/.env -v /opt/rosie/context:/context:ro \
           -v /opt/rosie/eval:/eval -e ROSIE_EVAL_DIR=/eval <image> npm run -s evaluate -- --sample 30 \
           || echo "Rosie evaluation regressed" | mail -s "Rosie" data-team@example.com
```

When the server has the same `ROSIE_EVAL_DIR` mounted, the last run is exposed with the
other metrics ([MONITORING.md](MONITORING.md)), so the alert can live in your monitoring
system instead of in cron:

| Metric | Meaning |
|---|---|
| `rosie_evaluation_accuracy_ratio` | Accuracy of the last run, 0 to 1 |
| `rosie_evaluation_held_out_accuracy_ratio` | The same, over the held-out questions only |
| `rosie_evaluation_questions{outcome}` | Questions of the last run by outcome |
| `rosie_evaluation_wrong_answers{warning}` | Wrong answers by the warning they carried: `flagged`, `not_checked`, `none` |
| `rosie_evaluation_regressions` | Regressions found by the last run |
| `rosie_evaluation_timestamp_seconds` | When it ran |

```yaml
- alert: RosieEvaluationRegressed
  expr: rosie_evaluation_regressions > 0
- alert: RosieEvaluationStale
  expr: time() - rosie_evaluation_timestamp_seconds > 3 * 86400
```

`GET /api/assistant/admin/evaluation`, with `ROSIE_ADMIN_TOKEN`, returns the last run in
full.

## Where runs are kept

In `ROSIE_EVAL_DIR`, or `<context>/eval/runs` when it is unset. Each run is a
`run-<time>.json`, and `latest.json` holds the last one plus the last known outcome of every
question. The context is usually mounted read-only in production, so set `ROSIE_EVAL_DIR`
to a writable directory there. When a run cannot be saved the command says so, and the next
run has nothing to compare with.

The files hold your questions, the statements and the outcomes. They hold no rows from your
database.

## Limits

- A reference is only as good as the person who wrote it. An ambiguous question with one
  reference punishes a reasonable other reading: say in the question what the reference
  assumes.
- A question whose result has more rows than one answer returns (500) is `inconclusive`.
- Questions run as the user `evaluate` with no identity claims. With row policies that
  require a claim, the reference and the answer are both filtered the same way, and
  questions that need a tenant return nothing.
- Every evaluation question is written to the audit trail like any other, under the user
  `evaluate`, so it can be told apart in reports.
- Approved queries with parameters are not added automatically, since they need values.
  Add a bank entry with the question and the statement for the values you care about.

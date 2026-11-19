# The evidence list

Every Answer Certificate carries `evidence`: the same findings as the verdict, said one
dimension at a time. It exists because one verdict is read as "how much to trust this
answer", and it is not that. On a public benchmark an answer with the verdict
`CROSS_CHECKED` was right 16% to 99% of the time depending on the model (`docs/BIRD.md`,
"Right answers by verdict"). What the checks establish is narrow and specific, and
`evidence` says exactly that: what was established, what was found wrong, and what nobody
looked at.

`certificate.verdict` is unchanged and stays for compatibility (`docs/CERTIFICATE.md`). Read
`evidence` when a figure is acted on.

## The contract

- **Every dimension is always present**, with a `status` from a closed list. An absent
  field never means "clean". A consumer that meets a status it does not know must treat
  it as not established.
- **It is deterministic.** It is computed from what the checks and the second derivation
  already returned. It runs no query and calls no model.
- **It is not derived from the verdict.** Both come from the same parts, and
  `test/unit/evidence.js` holds them to each other over every combination of check results
  it can enumerate (the table under "How it relates to the verdict").
- **`certificate.evidence.version`** is `1`. A new dimension or a new status may be added
  without changing it; a status that changes meaning changes it.
- A response with no certificate (`certificate: null`) has no evidence. See "An answer
  with no statement" below.

Each dimension is `{ "status": "...", "detail": "...", ...facts }`. `detail` is one sentence
for a person, present when a check produced one. The facts are listed per dimension.

Three fields can appear on any checked dimension:

| Field | Meaning |
|---|---|
| `reason` | With `not_looked` or `not_run`: why, as one of the reasons of `certificate.skipped` (`docs/CERTIFICATE.md`, "When a check does not run") |
| `partial` | With `not_looked`: a part of the check ran and passed (the outer statement, for example), and the rest did not run. It is still not a pass |
| `notLookedInFull` | On a dimension that holds a finding: a part of it was also not looked at, with the reason. The finding stands |
| `nested` | The finding is about a filter or a join inside a subquery of the statement, not the outer `SELECT` |

## The dimensions

### `statement`

Who wrote the SQL. It decides how much the other dimensions can mean.

| Status | Meaning | What it does NOT mean |
|---|---|---|
| `model` | A language model wrote the statement from the question | Nothing here says the model read the question as the person meant it |
| `approved_query` | The statement is an approved query, run unchanged with parameter values | That the parameter values are the ones the question meant |
| `compiled_metric` | The engine compiled the statement from a declared metric | That this metric is the one the question needed |
| `caller` | The caller wrote the statement (`POST /certify`, `certify_sql`) | Anything about the caller's question |

### `repeatedRows`

Whether a join repeats the records a total, an average or a count is taken over. Sources:
the `fanout` check (a count of joined rows against distinct records) and the second
derivation. `from` says which: `row_count` or `second_derivation`. When it is the second
derivation, the figures and the sentence are in `secondDerivation` and are not repeated.

| Status | Meaning | What it does NOT mean |
|---|---|---|
| `none_found` | The join does not repeat the records under the figure, or it repeats them and the figure is the same taken once per record (`repeatsWithoutEffect: true`). `groupsOverlap: true` says a record can fall in more than one group of a grouped answer, so the groups must not be added up | That the right table, column or filter was used |
| `found` | Records are counted more than once, and the context declares the column a value of each record (`declaredBy`, or `table` and `columns`). The figure is too high | That the question was otherwise understood |
| `two_readings` | Records repeat and nothing declares what the column is. Once per joined row and once per record are both defensible figures | That either figure is wrong, or that either is right |
| `not_looked` | A total over a join in a form the checks do not follow, or a verification query that did not run. `reason` says which | That rows repeat. It is unknown |
| `not_applicable` | The statement has no total, average or count over a join | |

### `secondDerivation`

The figure computed again taking each record of its table once, under the same filters
(`docs/CERTIFICATE.md`, "The second derivation").

Facts: `of` (`total` for a `SUM` or `AVG`, `count` for a `COUNT`), `primary` (the figure of
the answer), `alternative` (the figure over distinct records), `table` and `key` (the table
each record was taken once from, and the column it was told apart by), `comparison` (only
with `agreed`: `exact`, or `at_shown_precision` when the answer rounds the figure and the
second one rounds to it), `groups` (`compared`, `differing`, for a grouped answer; `primary`
and `alternative` are then those of the first group that differs), `declaredBy` (what
declared the column, with `contradicted`).

| Status | Meaning | What it does NOT mean |
|---|---|---|
| `agreed` | Both figures are the same. No record was repeated or merged by the join, the `DISTINCT` or the grouping | That the figure is right. Both derivations read the same tables, filters and column, and are wrong together when one of those is. For `of: "count"` it only says the count is one row per record |
| `contradicted` | The figures differ and the context declares the column a value of each record. The figure of the answer counts records more than once | That `alternative` answers the question. It is the same statement without the repetition |
| `differs_undeclared` | The figures differ and nothing declares the column (always the case for a count). Two figures for two questions | That one of them is wrong |
| `not_run` | It was due and was not made (`reason`: no single-column key, lineage not followed, probe failed, `disabled`, and the others) | That the figure differs |
| `not_applicable` | There is nothing to compute again: not one figure over a join, or a result of zero | |

`table` and `key` are read from the second statement the engine ran. `key` is `null` when it
cannot be read; the statement itself is in `certificate.crossCheck.alternative.sql`.

### `filterValues`

The `empty` check: an empty or zero result explained by a filter on a value the column does
not hold.

| Status | Meaning | What it does NOT mean |
|---|---|---|
| `passed` | The result is empty or zero and the filtered value does exist in the column | That the filter is the one the question meant |
| `warned`, `failed` | The value filtered by is not in the data. `detail` lists values that are | That the result would be right with another value |
| `not_looked` | A filter the check does not reach (inside `UNION ALL` branches, more nested filters than it reads), or a verification query that did not run | |
| `not_applicable` | The result is not empty, or no filter by equality or pattern. **The check does not run on a result that has rows**, so this says nothing about the values of such an answer | |

### `period`

A date range that the data does not cover.

| Status | Meaning | What it does NOT mean |
|---|---|---|
| `passed` | The dates the statement bounds are inside what the column holds | That the period is the one the question asked for, or that there are no gaps inside it |
| `warned` | The range runs past the last record, starts before the first, or a "last N months" question was answered over an old fixed range | That the answer is wrong. It is a fact about the data and is true of a right answer as often as of a wrong one (`docs/BIRD.md`) |
| `failed` | Not produced today | |
| `not_looked` | As for `filterValues` | |
| `not_applicable` | No date column is bounded | |

### `filterCoverage`

A filter that keeps every row. Outer `SELECT` only.

| Status | Meaning | What it does NOT mean |
|---|---|---|
| `passed` | The `WHERE` removes some rows | That it removes the right ones |
| `warned` | The `WHERE` keeps every row, so it is probably not filtering what was meant | That the answer is wrong: a filter that legitimately keeps everything is reported too |
| `failed` | Not produced today | |
| `not_looked` | `UNION ALL`, or a verification query that did not run | |
| `not_applicable` | No `WHERE` in the outer `SELECT`, or an empty result | |

### `zerosInAverage`

An average that counts rows holding 0 where 0 may stand for "not recorded". Facts with
`warned`: `table`, `column`, `zeros`, `rows`, `smallestOther`, `averageWithoutThem`.

| Status | Meaning | What it does NOT mean |
|---|---|---|
| `warned` | The rows averaged hold zeros and the smallest other value is 2 or more | That the zeros are placeholders. A real zero is reported too |
| `nothing_reported` | The statement holds an average and the check said nothing. **This is not a pass**: the check is silent both when it found no such zeros and when it does not read the form (an average inside a subquery, over an expression, with `FILTER`) | That the average is free of placeholder values |
| `not_applicable` | The statement holds no `AVG` | |
| `not_looked` | The checks stopped on an error | |
| `passed`, `failed` | Reserved. The check emits neither today | |

### `metricDefinition`

Whether the figure follows a metric the context declares (`docs/METRICS.md`).

| Status | Meaning | What it does NOT mean |
|---|---|---|
| `compiled_from_declaration` | The engine compiled the statement from the declared metric (`name`, `expression`). Formula, table and required filters are the declared ones by construction | That this metric is what the question needed |
| `declared_and_followed` | The question names a declared metric and the statement uses its expression, table and filters | That the whole statement is the metric: its parts were compared, not the result |
| `declared_not_followed` | The question names a declared metric and the statement computes it another way | |
| `none_declared_for_this_question` | No declared metric was matched to the question (or no question was given). The formula is the statement's own | That no metric applies: matching is by name and synonym |
| `not_looked` | The checks stopped on an error | |

### `approval`

| Status | Meaning | What it does NOT mean |
|---|---|---|
| `approved_query` | The statement is an approved query: `id`, `question`, `approvedBy`, `approvedOn`, `reviewBy`. A person read this statement for this question | That it was read against today's data or schema |
| `none` | Nobody approved this statement | |

### `businessGrain`

Whether the table the figure is taken over has a declared business entity, and whether its
rows are one entity each. Facts: `table`, `entity`, `key`, `rows`, `entities`, and
`alternative` (the figure per entity) when there is one.

| Status | Meaning | What it does NOT mean |
|---|---|---|
| `declared_and_single` | The context declares what a row of the table is, and each entity appears once | That the question counts that entity |
| `declared_and_repeats` | The context declares the entity and it appears in more than one row, so counting rows is not counting entities | That the figure is wrong: the question may count rows |
| `not_declared` | Nothing is declared for the table, or the statement takes no figure over a table. Whether a row is the thing the question counts is unknown | That the rows are one entity each |
| `not_looked` | The checks stopped on an error | |

### `questionInterpretation`

Whether the statement answers the question that was asked. No deterministic check looks at
this, and it is where most wrong answers are.

| Status | Meaning | What it does NOT mean |
|---|---|---|
| `not_verified` | Nothing checked it. This is the status of every statement a model wrote, whatever the verdict. `replacedAfterReview: true` says a review objected to a first statement and this is a second one, which was not reviewed | That the answer is wrong |
| `reviewed_agrees` | The question review ran (`AI_QUESTION_REVIEW`) and a model said the statement answers the question. `opinionOf` is always `model`; `model` names it; `onThisDataOnly: true` says it is written unlike the question and returns the same on the values held today | That the question was understood. It is an opinion: on a benchmark the reviewer said "answers" for 125 of 173 wrong answers (`docs/CERTIFICATE.md`) |
| `reviewed_disagrees` | A model said the statement does not answer the question. `reading` is its sentence | That the answer is wrong. It is an opinion |
| `reviewed_cannot_tell` | A model read both and could not tell | |
| `defined_by_a_person` | The statement is an approved query (`approvedBy`, `approvedOn`): a person decided what answers this question | That the person was right, or that the model picked the right approved query for a differently worded question |

### `dataConventions`

How the data records things: a 0 or an empty text that stands for "not recorded", a code
kept with its leading zero, a status spelled another way. The engine verifies none of it.

| Status | Meaning | What it does NOT mean |
|---|---|---|
| `not_verified` | The default, for every answer. No check confirmed how the columns used are recorded | That there is a problem |
| `warned` | A check holds a fact that the data is recorded another way than the statement assumes. `findings` names the dimensions that hold it (`zerosInAverage`, `filterValues`) | That the other conventions were looked at |

There is no status that says the conventions were confirmed, because nothing confirms them.

### `otherChecks` and `notEstablished`

`otherChecks` is an array, empty today. A check added to the battery before it has a
dimension of its own appears there as `{ id, status, detail }`, so it can never warn
without the evidence showing it.

`notEstablished` is an array of plain English sentences that a UI or an agent can show as
they are. It is never empty for a statement a model wrote: it always says that the question
was not checked and that the conventions of the data were not verified, and adds what else
is open (no metric matched, no business entity declared, two readings of a total, a check
that did not run, what an agreeing second derivation does not show).

## How it relates to the verdict

The verdict is one word over these dimensions. The unit test holds each line below for
every combination it enumerates (158,760 of them, each built before and after the orchestrator reconciles the fan-out line).

| Verdict | Holds exactly when |
|---|---|
| `CONTRADICTED` | `secondDerivation.status` is `contradicted` (and `repeatedRows.status` is then `found`) |
| `FLAGGED` | Nothing is contradicted and some dimension holds a finding: `warned`, `failed`, `found`, `two_readings`, `differs_undeclared`, `declared_not_followed`, `declared_and_repeats` |
| `NOT_CHECKED` | No dimension holds a finding and some dimension is `not_looked` |
| `CROSS_CHECKED` | `secondDerivation` is `agreed` with `of: "total"`, no dimension holds a finding and none is `not_looked` |
| `CHECKS_PASSED`, `NOT_APPLICABLE` | No finding, nothing `not_looked`, no agreed total |

Two things follow. `questionInterpretation`, `dataConventions`, `approval` and `statement`
play no part in the verdict, which is why a clean verdict says nothing about them. And
`secondDerivation.not_run` and `zerosInAverage.nothing_reported` can sit under a clean
verdict: read them.

## For an agent or an integration that acts on a figure

**Do not automate on the verdict alone.** A clean verdict on a statement a model wrote
means the calculation has no structural defect the checks know. It does not mean the
figure answers the question.

A policy that is safe to start from:

1. **Never act on** a figure whose evidence holds `secondDerivation.status = "contradicted"`
   or `repeatedRows.status = "found"`. Do not present it as a fact either.
2. **Require meaning from a person or a declaration** before a figure is acted on without
   a person reading it: `approval.status = "approved_query"`, or
   `metricDefinition.status = "compiled_from_declaration"`. With MCP, ask through
   `run_approved_query` or `query_metric` instead of `ask_data` when the figure matters.
3. **Stop on anything open** in the dimensions your use depends on: `not_looked`,
   `not_run`, `two_readings`, `differs_undeclared`, `declared_not_followed`,
   `declared_and_repeats`, `warned`, `failed`. For an average, treat
   `zerosInAverage.nothing_reported` as open.
4. **Treat `reviewed_agrees` as an opinion.** It can lower the priority of a human review;
   it cannot replace an approval.
5. **Show `notEstablished`** next to the figure whenever a person will read it.

A statement with `statement.status = "model"` and `approval.status = "none"` and
`metricDefinition.status = "none_declared_for_this_question"` is a draft for a person to
read, whatever its verdict.

## Where it is

- `POST /ask` and `POST /certify`: `certificate.evidence`.
- MCP `ask_data`, `query_metric`, `run_approved_query`, `certify_sql`:
  `structuredContent.certificate.evidence`, declared in the output schema, and one line in
  the text block that starts with "Not established:".
- Audit: the `provenance` column of `rosie_answer_certificate` holds `evidence` with the
  status and the facts of each dimension, without `detail` and `notEstablished`.
- Code: `server/ai-assistant/sql/evidence.js` (`build( parts )`).

## Examples

### A clean total

"Total invoiced to customers in Brazil", written by a model. The verdict is `CROSS_CHECKED`.

```json
{
  "verdict": "CROSS_CHECKED",
  "evidence": {
    "version": 1,
    "statement": {
      "status": "model"
    },
    "repeatedRows": {
      "status": "none_found",
      "from": "second_derivation"
    },
    "secondDerivation": {
      "status": "agreed",
      "of": "total",
      "primary": 190.1,
      "alternative": 190.1,
      "table": "Invoice",
      "key": "InvoiceId",
      "comparison": "exact",
      "detail": "Re-derived over distinct Invoice records and got the same value. This checks the calculation, not whether the question was understood."
    },
    "filterValues": {
      "status": "not_applicable"
    },
    "period": {
      "status": "not_applicable"
    },
    "filterCoverage": {
      "status": "passed",
      "detail": "The filter keeps 35 of 412 rows."
    },
    "zerosInAverage": {
      "status": "not_applicable"
    },
    "metricDefinition": {
      "status": "none_declared_for_this_question"
    },
    "approval": {
      "status": "none"
    },
    "businessGrain": {
      "status": "not_declared"
    },
    "questionInterpretation": {
      "status": "not_verified",
      "detail": "A model wrote this statement from the question. Whether it answers the question was not checked."
    },
    "dataConventions": {
      "status": "not_verified"
    },
    "otherChecks": [],
    "notEstablished": [
      "Whether the statement answers the question was not checked.",
      "How the data records missing or special values (a 0 or an empty text that stands for \"not recorded\") was not verified.",
      "No declared metric was matched to this question, so the formula is the statement's own.",
      "No business entity is declared for the table the figure is taken over, so whether its rows are the things the question counts was not checked.",
      "The two derivations agreeing shows only that no record was repeated or merged. Both use the same tables, filters and column, and are wrong together when one of those is."
    ]
  }
}
```

Read it as: no record was counted twice, the filter removes rows, and nothing else is
established. Five sentences say what is open.

### A contradicted total

"Revenue", answered by joining invoices to their lines and adding up the invoice total.

```json
{
  "verdict": "CONTRADICTED",
  "evidence": {
    "version": 1,
    "statement": {
      "status": "model"
    },
    "repeatedRows": {
      "status": "found",
      "from": "second_derivation",
      "declaredBy": "the metric revenue"
    },
    "secondDerivation": {
      "status": "contradicted",
      "of": "total",
      "primary": 20848.62,
      "alternative": 2328.6,
      "table": "Invoice",
      "key": "InvoiceId",
      "declaredBy": "the metric revenue",
      "detail": "Two derivations disagree: over the joined rows Invoice.Total gives 20848.62, over distinct Invoice records it gives 2328.6. The join repeats Invoice records and the metric revenue declares Invoice.Total as a value of each record, so the first figure counts some of them more than once."
    },
    "filterValues": {
      "status": "not_applicable"
    },
    "period": {
      "status": "not_applicable"
    },
    "filterCoverage": {
      "status": "not_applicable"
    },
    "zerosInAverage": {
      "status": "not_applicable"
    },
    "metricDefinition": {
      "status": "declared_and_followed",
      "detail": "Uses the declared definition of revenue: SUM(Invoice.Total)."
    },
    "approval": {
      "status": "none"
    },
    "businessGrain": {
      "status": "not_declared"
    },
    "questionInterpretation": {
      "status": "not_verified",
      "detail": "A model wrote this statement from the question. Whether it answers the question was not checked."
    },
    "dataConventions": {
      "status": "not_verified"
    },
    "otherChecks": [],
    "notEstablished": [
      "Whether the statement answers the question was not checked.",
      "How the data records missing or special values (a 0 or an empty text that stands for \"not recorded\") was not verified.",
      "No business entity is declared for the table the figure is taken over, so whether its rows are the things the question counts was not checked."
    ]
  }
}
```

### A plain list

"List the artists". Nothing in the statement for the checks to look at: the verdict is
`NOT_APPLICABLE`, and the evidence still says the question was not checked.

```json
{
  "verdict": "NOT_APPLICABLE",
  "evidence": {
    "version": 1,
    "statement": {
      "status": "model"
    },
    "repeatedRows": {
      "status": "not_applicable"
    },
    "secondDerivation": {
      "status": "not_applicable"
    },
    "filterValues": {
      "status": "not_applicable"
    },
    "period": {
      "status": "not_applicable"
    },
    "filterCoverage": {
      "status": "not_applicable"
    },
    "zerosInAverage": {
      "status": "not_applicable"
    },
    "metricDefinition": {
      "status": "none_declared_for_this_question"
    },
    "approval": {
      "status": "none"
    },
    "businessGrain": {
      "status": "not_declared"
    },
    "questionInterpretation": {
      "status": "not_verified",
      "detail": "A model wrote this statement from the question. Whether it answers the question was not checked."
    },
    "dataConventions": {
      "status": "not_verified"
    },
    "otherChecks": [],
    "notEstablished": [
      "Whether the statement answers the question was not checked.",
      "How the data records missing or special values (a 0 or an empty text that stands for \"not recorded\") was not verified."
    ]
  }
}
```

### An answer with no statement

A reply in words only (a clarifying question, a refusal, a sentence with no query behind
it, or a statement that did not run) has no certificate, so it has no evidence:

```json
{
  "narrative": "I can answer questions about invoices, customers and tracks.",
  "sql": null,
  "certificate": null,
  "table": null
}
```

Nothing was run, so nothing was checked. A figure that appears in such a text has no
statement behind it and must not be used.

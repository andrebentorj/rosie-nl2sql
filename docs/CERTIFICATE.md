# The Answer Certificate

After the SQL of an answer has run, Rosie runs a set of deterministic checks on that SQL and
attaches the outcome to the answer. No model is involved: the checks are read-only queries
and comparisons, so the same answer always gets the same certificate.

Read this page before relying on it. The certificate is evidence about specific, structural
ways an answer can be wrong without any error. It is not a statement that an answer is right.

## What it checks, and what it cannot

**It checks the structure of the calculation.** A join that repeats rows under a total, a
filter on a value that is not in the data, a period the data does not reach, a filter that
removes nothing, a declared metric computed another way.

**It does not check whether the question was understood.** If the model picked the wrong
column, the wrong denominator or the wrong reading of an ambiguous question, the calculation
is structurally sound and every check passes. On the public BIRD benchmark, where almost all
errors are of this kind, the certificate flagged 10 of 214 wrong answers (`docs/BIRD.md`).
Misread questions are reduced by context (column descriptions, declared metrics, verified
example queries) and measured by evaluation on your own questions, not caught here.

## The six verdicts

Every answer that ran SQL carries exactly one. There is no "no certificate": an answer where
nothing could be checked says so.

| Verdict | Meaning | What a reader should do |
|---|---|---|
| `CONTRADICTED` | A total (`SUM` or `AVG`) was re-derived over distinct records and came out different: the join counted some records more than once | Do not use the figure. Both numbers are shown |
| `FLAGGED` | A check warned or failed, or a count has two defensible readings | Read the warning; it says what and why |
| `CROSS_CHECKED` | The total was re-derived by a second route and both agree, and no check warned | The calculation does not depend on the join. It can still answer the wrong question |
| `NOT_CHECKED` | Something needed checking and no check could run. `skipped` lists each one and the reason | Treat as unchecked. The reason is shown to the user |
| `CHECKS_PASSED` | The checks that applied ran and passed; nothing was re-derived | No structural problem found in what was checked |
| `NOT_APPLICABLE` | The statement has nothing these checks look at (a plain list, a lookup) | Nothing to add |

Order of precedence, strongest first: `CONTRADICTED`, `FLAGGED`, `CROSS_CHECKED`,
`NOT_CHECKED`, `CHECKS_PASSED`, `NOT_APPLICABLE`. A warning is never hidden by an agreeing
re-derivation.

The names changed in version 1.0.1: `VERIFIED` became `CROSS_CHECKED`, `UNVERIFIED` became
`FLAGGED`, `QUALIFIED` became `CHECKS_PASSED`, and `NOT_CHECKED` and `NOT_APPLICABLE` are new.
`VERIFIED` was dropped because readers took it to mean "correct".

## The checks

| Check (`id`) | Detects | Applies when | Does not cover |
|---|---|---|---|
| `fanout` | A join that repeats the rows being summed | `SUM` or `AVG` over a join, of a column or of an expression over columns of one table (`SUM(il.UnitPrice * il.Quantity)`), bare or wrapped (`ROUND(SUM(x), 2)`), qualified or not, wherever that table sits in `FROM`, with or without `GROUP BY`. Each aggregated table is compared with its own primary key. The same holds inside a `WITH` body, a derived table, a `UNION ALL` branch or an uncorrelated subquery, as long as that inner statement reads catalog tables directly: it is probed on its own and its result is reported as "Inside a subquery of this answer" | A total mixing columns of several tables (`SUM(a.x * b.y)`); an inner statement that reads a `WITH` name or a derived table, or a correlated subquery, because out of its place it would count other rows; more than two such inner statements; tables without a single-column primary key. These are reported as `NOT_CHECKED` |
| `empty` | A filter on a value that does not exist (`City = 'Oporto'` when the data says `Porto`), and lists the real values | The result is empty or zero and the statement filters by equality or a pattern | Only the first eligible filter is probed |
| `coverage` | A filter that keeps every row, so probably is not filtering what was meant | The statement has a `WHERE` | A filter that legitimately keeps everything is still reported (about 5% of correct answers in our benchmark) |
| `period` | A date range that runs past the last record, or a "last N months" question answered over an old fixed range | The statement bounds a date column | Gaps inside the range; several date columns |
| `metric` | A question that names a declared metric, answered with a different expression, table or filter | The context declares metrics (`docs/METRICS.md`) | It compares the parts of the definition with the statement; it does not prove the final expression is the metric |

### The second derivation (`crossCheck`)

For an answer that is one number from one aggregate over a join, or one such number per
group (`GROUP BY`), Rosie computes it again over the distinct records of the table the
aggregated column belongs to, and compares.

- **`SUM` and `AVG`.** Summing a column of table T over rows a join has repeated is wrong
  whatever the question was. Agreement gives `CROSS_CHECKED`; disagreement gives
  `CONTRADICTED`, with both figures.
- **`COUNT`.** Counting joined rows (order lines) and counting distinct records (orders) are
  both legitimate numbers, and nothing in the statement says which was meant. Rosie reports
  the two counts as a note (`FLAGGED` when they differ) and never rules that one is wrong.
- **`GROUP BY`.** Every group shown in the answer is looked up in the re-derived result, up
  to 500 groups. One group that differs makes the answer `CONTRADICTED` (or a grain note for
  a `COUNT`), and the detail names up to three groups with both figures. `crossCheck.groups`
  carries `compared`, `differing` and `examples`; `primary` and `alternative` are then the
  figures of the first group that differs. If the second derivation does not return a group
  of the answer, nothing is endorsed: the answer is `NOT_CHECKED`.
- **Not attempted** for several aggregates in one statement, an aggregate wrapped in another
  function, a `GROUP BY` with `HAVING` or with grouping columns the answer does not show, or
  a result of zero (two routes agreeing on zero prove nothing). The `fanout` check covers
  the first two.

It is on by default and costs one extra read-only query on the answers it applies to.
`AI_CROSS_CHECK=false` turns it off.

## The second attempt

When the checks name a structural defect in the final query, the evidence goes back to the
model once, before anything is returned: "the join produced 2240 rows from 412 invoices, so
this total is inflated". The model either runs a corrected query or keeps its answer.

- **What triggers it:** `CONTRADICTED`, a failed or warned `fanout`, a failed `empty` (the
  message lists the values that do exist), a warned `metric`.
- **What does not:** `period` and `coverage` warnings describe the data, not a mistake in the
  query, so they are shown and not retried. A count with two grains is a question of intent.
- **Which answer goes out:** the second, only if it ran a new query and its certificate is
  better than the first one's. Otherwise the first answer goes out with its warning, exactly
  as it would have without the retry.
- **What is recorded:** `certificate.repair` holds `firstVerdict`, the `reasons` given to the
  model and the `outcome` (`corrected`, `unchanged`, `not_improved`, `failed`). A corrected
  answer says so on screen.

It costs one or two extra model calls on the answers it applies to and shares the request's
tool-call and query budgets. `AI_CERTIFICATE_REPAIR=false` turns it off.

## When a check does not run

`skipped` holds one entry per check that was due and did not run:

| `reason` | Meaning |
|---|---|
| `shape_not_covered` | A total over a join in a form the fan-out check does not cover |
| `union_branches` | The statement combines queries with `UNION ALL`. Fan-out is probed inside each branch, but the other checks (filter values, coverage, period) look at the outer statement only |
| `column_owner_unknown`, `no_single_primary_key`, `no_independent_route`, `rewrite_failed` | The second derivation could not be built for this statement |
| `groups_not_matched`, `too_many_groups` | The second derivation of a `GROUP BY` answer did not return every group of the answer, or the answer has more groups than it compares |
| `probe_failed` | A verification query could not be run |
| `probe_budget` | More verification queries were needed than one answer may run |
| `sql_not_parsed`, `checks_failed` | The statement could not be analysed, or the checks stopped on an internal error |

Each certificate is written to the log (`event: answer_certificate`, with `verdict` and
`skipped`) and to the `rosie_answer_certificate` table, so an operator can see how often
answers go out unchecked, and why. `npm run audit-report` summarises it, with the reasons behind `NOT_CHECKED`.

## Where it appears

- **Widget.** A strip under the answer and a banner above the results table. `NOT_CHECKED`
  is shown with its reason; `NOT_APPLICABLE` shows nothing.
- **HTTP API.** `certificate` in the `/ask` response (`docs/API.md`).
- **MCP.** A typed object in `structuredContent`, and in words in the text block, so a client
  that reads only text still cannot present a contradicted total as a fact
  (`docs/MCP_SERVER.md`).
- **Audit.** `rosie_answer_certificate`, one row per answer.

## What is measured

`docs/BENCHMARK.md` reports, on our own 194-question suite, how many trap questions went out
without a warning and how many correct answers were flagged anyway. `docs/BIRD.md` reports
the same on a benchmark we did not write. Read both: the first shows the checks working on
the errors they were built for, the second shows how small that share is among real errors.

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
| `fanout` | A join that repeats the rows being summed | `SUM` or `AVG` of a plain, qualified column of the first table in `FROM`, over a join, with or without `GROUP BY` | The aggregated table second in `FROM`; an expression inside or around the aggregate (`ROUND(SUM(x), 2)`, `SUM(x * 1)`); joins inside a CTE or subquery; composite primary keys. These are reported as `NOT_CHECKED` |
| `empty` | A filter on a value that does not exist (`City = 'Oporto'` when the data says `Porto`), and lists the real values | The result is empty or zero and the statement filters by equality or a pattern | Only the first eligible filter is probed |
| `coverage` | A filter that keeps every row, so probably is not filtering what was meant | The statement has a `WHERE` | A filter that legitimately keeps everything is still reported (about 5% of correct answers in our benchmark) |
| `period` | A date range that runs past the last record, or a "last N months" question answered over an old fixed range | The statement bounds a date column | Gaps inside the range; several date columns |
| `metric` | A question that names a declared metric, answered with a different expression, table or filter | The context declares metrics (`docs/METRICS.md`) | It compares the parts of the definition with the statement; it does not prove the final expression is the metric |

### The second derivation (`crossCheck`)

For an answer that is one number from one aggregate over a join, Rosie computes the number
again over the distinct records of the first table and compares.

- **`SUM` and `AVG`.** Summing a column of table T over rows a join has repeated is wrong
  whatever the question was. Agreement gives `CROSS_CHECKED`; disagreement gives
  `CONTRADICTED`, with both figures.
- **`COUNT`.** Counting joined rows (order lines) and counting distinct records (orders) are
  both legitimate numbers, and nothing in the statement says which was meant. Rosie reports
  the two counts as a note (`FLAGGED` when they differ) and never rules that one is wrong.
- **Not attempted** for `GROUP BY` results, several aggregates, an aggregate on a table that
  is not first in `FROM`, or a result of zero (two routes agreeing on zero prove nothing).

It is on by default and costs one extra read-only query on the answers it applies to.
`AI_CROSS_CHECK=false` turns it off.

## When a check does not run

`skipped` holds one entry per check that was due and did not run:

| `reason` | Meaning |
|---|---|
| `shape_not_covered` | A total over a join in a form the fan-out check does not cover |
| `aggregate_not_on_first_table`, `column_owner_unknown`, `no_single_primary_key`, `no_independent_route`, `rewrite_failed` | The second derivation could not be built for this statement |
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

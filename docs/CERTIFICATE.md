# The Answer Certificate

After the SQL of an answer has run, Rosie runs a set of deterministic checks on that SQL and
attaches the outcome to the answer. No model is involved: the checks are read-only queries
and comparisons, so the same statement over the same data, context and permissions gets the
same certificate. The verification queries run after the answer and not in one snapshot with
it: on a database that is being written to, a row added in between can make two figures
differ for a reason that has nothing to do with the statement.

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
| `CONTRADICTED` | A total (`SUM` or `AVG`) was re-derived over distinct records and came out different, **and the context declares that column a value of each record** (a declared metric adds it up, or the catalog marks it a measure): the join counted some records more than once | Do not use the figure. Both numbers are shown, with what declared the column |
| `FLAGGED` | A check warned or failed, or a total, an average or a count has two defensible readings (once per joined row, or once per record) | Read the warning; it says what and why, and shows both figures when there are two |
| `CROSS_CHECKED` | The total was re-derived by a second route and both agree, and no check warned | The calculation does not depend on the join. It can still answer the wrong question |
| `NOT_CHECKED` | Something needed checking and no check could run. `skipped` lists each one and the reason | Treat as unchecked. The reason is shown to the user |
| `CHECKS_PASSED` | The checks that applied ran and passed; nothing was re-derived | No structural problem found in what was checked |
| `NOT_APPLICABLE` | The statement has nothing these checks look at (a plain list, a lookup) | Nothing to add |

Order of precedence, strongest first: `CONTRADICTED`, `FLAGGED`, `NOT_CHECKED`,
`CROSS_CHECKED`, `CHECKS_PASSED`, `NOT_APPLICABLE`. A warning is never hidden by an agreeing
re-derivation, and neither is a check that was due and did not run: a total that was
re-derived and agreed, on a statement whose period could not be probed, goes out
`NOT_CHECKED` with both facts shown.

The names changed in version 1.0.1: `VERIFIED` became `CROSS_CHECKED`, `UNVERIFIED` became
`FLAGGED`, `QUALIFIED` became `CHECKS_PASSED`, and `NOT_CHECKED` and `NOT_APPLICABLE` are new.
`VERIFIED` was dropped because readers took it to mean "correct".

## The checks

| Check (`id`) | Detects | Applies when | Does not cover |
|---|---|---|---|
| `fanout` | A join that repeats the rows being summed | `SUM` or `AVG` over a join, of a column or of an expression over columns of one table (`SUM(il.UnitPrice * il.Quantity)`), bare or wrapped (`ROUND(SUM(x), 2)`), qualified or not, wherever that table sits in `FROM`, with or without `GROUP BY`. Each aggregated table is compared with its own primary key. The same holds inside a `WITH` body, a derived table, a `UNION ALL` branch or an uncorrelated subquery, as long as that inner statement reads catalog tables directly: it is probed on its own and its result is reported as "Inside a subquery of this answer". A total whose join sits one `SELECT` below it (`WITH x AS (SELECT i.Total ... JOIN ...) SELECT SUM(Total) FROM x`) is traced: each added column is followed into the source, to the table it comes from, and the source is probed with that table's key. Since 1.17.0 the total of the outermost `SELECT` is judged by lineage: the column is followed down to its table, the key of that table is carried up to the total, and the count is made there, per group when the statement groups. A `DISTINCT` or a `GROUP BY` in between is not taken as proof of anything; where one of them does not keep the key, a count cannot see two records merged into one row, and only the comparison of the two figures settles it (the second derivation below). The same holds for a source grouped by columns of one table that include its primary key. A `GROUP BY` proves nothing else: a source that joins and groups by more than one table's key still hands each record over several times, and a total over it that is not itself a total of the group is reported `NOT_CHECKED`. A total that appears only in `HAVING` or `ORDER BY` is checked like one that is returned. A total over a real table joined to a `WITH` name (`WITH sold AS (...) SELECT SUM(i.Total) FROM Invoice i JOIN sold ...`) is probed and re-derived with the `WITH` kept in front | A total mixing columns of several tables (`SUM(a.x * b.y)`); an inner statement that reads a `WITH` name or a derived table, or a correlated subquery, because out of its place it would count other rows; a source that is not a plain projection (a `DISTINCT` without the key, a window or a set operation) or whose traced column mixes tables; a join two or more `SELECT`s below the total (a derived table over a `WITH` name over a join); `SUM(DISTINCT x)` and `AVG(DISTINCT x)` over a join, which collapse equal values and not records; a statement where one level was proved clean and another could not be checked (the partial proof is not shown); more than two such inner statements; tables without a single-column primary key. These are reported as `NOT_CHECKED` |
| `empty` | A filter on a value that does not exist (`City = 'Oporto'` when the data says `Porto`), and lists the real values | The result is empty or zero and the statement filters by equality or a pattern | Only the first eligible filter is probed |
| `coverage` | A filter that keeps every row, so probably is not filtering what was meant | The statement has a `WHERE` | A filter that legitimately keeps everything is still reported (about 5% of correct answers in our benchmark) |
| `period` | A date range that runs past the last record, or a "last N months" question answered over an old fixed range | The statement bounds a date column | Gaps inside the range; several date columns |
| `metric` | A question that names a declared metric, answered with a different expression, table or filter | The context declares metrics (`docs/METRICS.md`) | It compares the parts of the definition with the statement; it does not prove the final expression is the metric |

### The second derivation (`crossCheck`)

For an answer that is one number from one aggregate over a join, or one such number per
group (`GROUP BY`), Rosie computes it again over the distinct records of the table the
aggregated column belongs to, and compares.

- **`SUM` and `AVG`.** Agreement gives `CROSS_CHECKED`. Disagreement means the join repeats
  the records whose values are added, and whether that is a mistake depends on what the
  column is. The total of an invoice added once per invoice line is a mistake. The length of
  a track added once per sale is the playing time sold, and is correct. The two statements
  have the same shape, so the statement cannot tell them apart; the context can:
  - when a declared metric adds that column up on its own table (`revenue =
    SUM(Invoice.Total)` on `Invoice`), or its catalog entry has `"measure": true`, the column
    is a value of each record and the answer is `CONTRADICTED`, naming what declared it;
  - with no declaration, both figures are shown as two grains of the same data and the
    answer is `FLAGGED`. Nothing is called wrong, and nothing is sent back to the model.

  A context with no declared metric and no `"measure": true` therefore never produces
  `CONTRADICTED`: the certificate still warns, and takes no side. `npm run doctor` says so.

  A grouped answer is judged per group. When no record repeats *inside* a group, every
  group's figure is exact and the answer passes, with a note that the groups overlap: units
  sold per playlist counts a track in each playlist it is on, which is what the breakdown
  means, and adding the lines up would count it twice.

  Versions 1.1.0 to 1.11.0 called every such disagreement a contradiction. An independent
  review showed that this condemned legitimate totals; 1.12.0 corrected it.
- **`COUNT`.** Counting joined rows (order lines) and counting distinct records (orders) are
  both legitimate numbers, and nothing in the statement says which was meant. Rosie reports
  the two counts as a note (`FLAGGED` when they differ) and never rules that one is wrong.
- **`GROUP BY`.** Every group shown in the answer is looked up in the re-derived result, up
  to 500 groups. One group that differs makes the answer `CONTRADICTED` when the column is
  declared, a grain note otherwise (and always for a `COUNT`), and the detail names up to
  three groups with both figures. `crossCheck.groups`
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
model once, before anything is returned: "the join produced 2240 rows from 412 distinct
Invoice records, and the metric revenue declares Invoice.Total as a value of each record".
The model either runs a corrected query or keeps its answer.

- **What triggers it:** `CONTRADICTED` with the figure over distinct records in hand, a
  `fanout` that failed or warned on a column whose grain is declared, a failed `empty` (the
  message lists the values that do exist), a warned `metric`. A contradiction with no single
  figure to reach (a grouped answer) is not sent back: no second answer could be accepted,
  so the first goes out with its warning and no model call is spent.
- **What does not:** `period` and `coverage` warnings describe the data, not a mistake in the
  query, so they are shown and not retried. Two grains of a total, an average or a count are
  a question of intent the model has no new information to settle.
- **Which answer goes out:** the second, only on evidence about the figure and about the rows
  it speaks of. A clean verdict on the second query is not that evidence: a query can pass
  every check and answer a narrower question.
  - *Repeated rows.* The first attempt already computed the total over distinct records.
    The second answer is accepted when it filters by exactly the same conditions and returns
    exactly that figure. Where there is no such figure, one form is still accepted: the
    same column of the same table, added up from that table alone, under the same filters.
  - *A filter value that does not exist.* The same columns filtered the same way (the value
    may change, which is the correction), a result that is not empty, and checks that ran
    clean.
  - *A declared metric not followed.* Nothing the first query filtered by was dropped, and
    checks that ran clean.

  Anything else leaves the first answer in place with its warning, exactly as it would have
  gone out without the retry.
- **What is recorded:** `certificate.repair` holds `firstVerdict`, `secondVerdict`, the
  `reasons` given to the model and the `outcome` (`corrected`, `unchanged`, `not_improved`,
  `failed`). An answer that replaced a first one says so on screen.

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
- **For SQL written elsewhere.** `POST /certify` and the MCP tool `certify_sql` run a
  `SELECT` the caller wrote and return it with its certificate, with no model involved. Off
  unless `ROSIE_CERTIFY_SQL=true` (`docs/API.md`, `docs/MCP_SERVER.md`).
- **MCP.** A typed object in `structuredContent`, and in words in the text block, so a client
  that reads only text still cannot present a contradicted total as a fact
  (`docs/MCP_SERVER.md`).
- **Audit.** `rosie_answer_certificate`, one row per answer.

## What is guaranteed, and how it is tested

One property of the certificate is stated as a guarantee and tested as one.

> **If the answer adds up or averages a column and its certificate reads `CHECKS_PASSED` or
> `CROSS_CHECKED`, the figure is the one you get by taking each record of that column's
> table once.** A total that a join has repeated, or that a `DISTINCT` has merged, does not
> go out with either of those verdicts. It is called wrong (`CONTRADICTED`), shown with both
> figures (`FLAGGED`), or reported as not checked (`NOT_CHECKED`). And a figure that is the
> per-record one is never called wrong.

What makes this hold for statements written in ways nobody listed in advance is that the
check no longer recognises shapes of SQL. It follows the column that is added up down to its
table, through every `WITH` name and derived table in between, and makes the primary key of
that table ride up to the `SELECT` that computes the total. There the same total is
computed again over each record once, and the two figures are compared. Where the column
cannot be followed (it mixes tables, a level in between aggregates, there is no single-column
key), the answer says it was not checked.

**How it is tested.** `test/fuzz/certificate.js` builds statements by combination: four things
to add up (a column of a parent table, a column and an expression of a child table, a column
the context does not declare), each joined to nothing, to a parent, to a child, to a bridge
table, with inner and outer joins; filtered or not; grouped by nothing, by its own column or
by a joined one; with `SUM` or `AVG`; with the total in the `SELECT`, inside `ROUND`, in
`HAVING` or in `ORDER BY`; and with the join written in the statement itself, in a `WITH`, in
a derived table, behind a `DISTINCT` with or without the key, behind a `GROUP BY` with the
key or with more than the key, two levels down, inside an `IN` subquery, deduplicated
below and joined again above, or with the table joined to a derived list of keys. For each statement the truth is worked out by a route of its
own, on the real database, and the certificate is held to the two halves of the guarantee.

Version 1.19.0, every combination, on the demo database of each engine:

| | MySQL 8.4 | PostgreSQL 16 | SQL Server 2022 |
|---|---|---|---|
| Statements certified | 9,624 | 9,624 | 9,592 |
| With a wrong total | 3,332 | 3,332 | 3,300 |
| of which went out with a clean certificate | **0** | **0** | **0** |
| of which were called wrong or shown with both figures | 3,332 | 3,332 | 3,300 |
| With the right total | 6,292 | 6,292 | 6,292 |
| of which were called wrong | **0** | **0** | **0** |
| of which went out clean | 5,898 | 5,898 | 5,898 |

Of the right totals that did not go out clean, 344 were flagged with notes about the data
that are true (a filter that removes no rows, mostly; 28 of them a note that rows repeat,
where the total sits in `HAVING` or `ORDER BY` or the answer was cut at the row limit, and
could not be computed a second time), and 50 were reported as not checked: grouped answers
cut at the row limit, where the groups shown agreed and nothing can be said of the others.
On SQL Server 32 statements were refused by the database before running (the sum of an
integer column overflows there). Summaries per shape are in
`docs/benchmark-runs/property-test-1.19.0-*.json`; `npm run test:fuzz` runs a sample, `--all`
runs every combination, and every run prints its seed.

**On schemas this project did not design.** The same test builds its statements from any
catalog: for each table with a numeric column and a relationship, the tables it points to
and the tables that point to its key. `test/fuzz/schemas.js` runs it on the databases of BIRD
Mini-Dev (PostgreSQL, 400 statements each, seed 1). Four of the eleven have no table with a
numeric column and a relationship, so there is no total to test there.

| Database | Statements certified | With a wrong total | of which clean | With the right total | of which called wrong |
|---|---|---|---|---|---|
| `california_schools` | 400 | 40 | **0** | 360 | **0** |
| `codebase_community` | 400 | 116 | **0** | 284 | **0** |
| `european_football_2` | 400 | 54 | **0** | 346 | **0** |
| `financial` | 400 | 77 | **0** | 323 | **0** |
| `formula_1` | 345 | 103 | **0** | 242 | **0** |
| `student_club` | 249 | 42 | **0** | 207 | **0** |
| `superhero` | 400 | 94 | **0** | 306 | **0** |
| **All seven** | **2,594** | **526** | **0** | **2,068** | **0** |

Of the 526 wrong totals, 520 were called wrong or shown with both figures and the rest
were reported as not checked. Some statements were refused by the database before running
(on two of the databases `ROUND` with decimals does not exist for the floating-point column
the generator picked).

This is where the test earned its keep. Its first pass over these databases found sixteen
wrong totals with a clean certificate, all of one kind the demo database could not show: a
grouped answer with 13,645 groups, of which 500 were shown and re-derived, and all 500
agreed. The certificate said `CROSS_CHECKED` about a statement whose other groups were
wrong. Since 1.19.0, on an answer cut at the row limit a group that differs is still
evidence, and all the shown groups agreeing is not agreement.

**What this is not.** It is a property tested on a family of statements, on one schema, not
a proof about all SQL. The same test, on the version before this one, found two wrong totals
with a clean certificate and six right ones called wrong in its first 150 statements, which
is how the rule above came to replace the ones before it, and one more kind on other schemas
(above). A statement outside the family
that breaks the property is a defect, and the generator is where its shape gets added. And
the guarantee is about the calculation only: a query that adds up the right records for the
wrong question passes it (see the next section).

## The question review

The certificate checks the calculation and stays deterministic. The question review is a
separate, optional step for what the certificate cannot see: a sound query that answers
another question. It is off unless `AI_QUESTION_REVIEW=true`.

After the answer is ready, one model call reads the question, the final statement, the first
rows it returned and the values held by the columns it filters on, and gives one of four
opinions: `answers`, `answers_on_this_data` (written unlike the question, same result on the
values the column holds today), `does_not_answer`, `cannot_tell`, with one sentence. When it
says `does_not_answer`, that sentence goes back once to the model that wrote the query, the
way a certificate finding does. The second answer replaces the first only when it ran
another statement and the certificate does not rate that one lower.

- It is returned as `questionReview` (`verdict`, `reading`, `model`, `outcome`, and
  `firstSql` when the answer was replaced). **It never changes the certificate's verdict**,
  and it is an opinion: show `reading` as how the question was read, not as a finding.
- The values the reviewer sees go through the validator and the row policies of the
  request, so it reads nothing the caller could not have asked for. Masked columns are
  neither read nor listed.
- It does not run on an approved query (a person read that one) or on a compiled metric, nor
  on an answer the certificate already sent back for a second attempt.
- It costs one model call per answer, on `AI_QUESTION_REVIEW_MODEL` when set, and a second
  attempt on the answers it objects to. `AI_QUESTION_REVIEW_REPAIR=false` keeps the opinion
  and drops the second attempt.
- **Use a stronger model than the one that writes.** The measurement below shows a model
  reviewing its own query finds little.

### What it did on BIRD Mini-Dev

500 questions, PostgreSQL, `gpt-6-luna` writing and `gpt-6.1-terra` reviewing, engine 1.20.0,
2026-11-13. The bar was fixed before the run: more than the 272 correct of the writer alone
(`docs/BIRD.md`), and fewer than 10 right answers turned wrong by the review.

| | |
|---|---|
| Correct, with the review | 320 of 500 (64.0%) |
| Correct, `gpt-6-luna` alone, run of 2026-11-10 | 272 of 500 (54.4%) |
| Answers the reviewer sent back | 111 |
| of which the model wrote another statement that was accepted | 81 |
| Wrong answer turned right | 43 |
| Right answer turned wrong | 4 |
| Still wrong, or still right, after the second attempt | 27 and 7 |

**Both parts of the bar are met.** Read it with these in mind:

- **The review accounts for 39 of the 48 answers gained.** Every replaced answer had its
  first statement scored too: without the second attempts this same run would have had 281
  correct. The other 9 are the difference between two runs of the same model.
- **The reviewer's instructions were adjusted on these questions.** They were written after
  reading fifteen answers of the earlier runs of this benchmark. The figure needs a set of
  questions that played no part before it is quoted as a general one.
- **It is one run.**
- **166 answers are still wrong.** On 47 of them the reviewer said the statement answers the
  question, on 58 it could not tell, and 59 it objected to and the second attempt did not
  fix. A third of the questions stays out of its reach.
- **`gpt-6.1-terra` alone gets 429 of 500** on the same questions. The review closes a part
  of the gap between a cheap and a strong writer at the price of one strong call per
  answer; it does not close it.

Every question, with both statements where there were two, is in
`docs/benchmark-runs/question-review-bird-1.20.0.json`. To repeat it:
`AI_QUESTION_REVIEW=true AI_QUESTION_REVIEW_MODEL=gpt-6.1-terra node -r dotenv/config test/bird/run.js`.

How it came to this design is the next section: the same reviewer used as a warning did not
pass, which is why it is used for a second attempt and shown as a reading.

## Whether a misread question can be detected: the measurement behind the review

The checks above look at the calculation. Almost every wrong answer on a benchmark we did
not write is of another kind: the query is sound and answers another question. Before
building anything for that, we measured whether it can be detected at all.

The two full runs of BIRD Mini-Dev (`docs/BIRD.md`) kept, for every question, the SQL the
engine produced and whether its rows matched the reference. That is 285 wrong answers.
Those, and 285 right ones drawn evenly from the two runs, were shown to a judge model: the
question with its additional information, the tables the statement names, and the statement.
The judge says whether the statement answers the question. No data is shown to it.

The bar was fixed before the run: worth building at 30% of the wrong answers detected or
more, with 5% of the right ones flagged or fewer.

| Judge | Wrong answers it flags | Right answers it flags |
|---|---|---|
| `gpt-6-luna` | 87 of 285 (30.5%) | 50 of 285 (17.5%) |
| `gpt-6.1-terra` | 134 of 285 (47.0%) | 44 of 285 (15.4%) |
| Both judges agree | 66 of 285 (23.2%) | 15 of 285 (5.3%) |

**It does not clear the bar, so no warning is built on it.** A judge finds between a third and
half of the misread questions, which is far more than the certificate does on the same
answers (6 of 71 and 10 of 214), and it flags about one right answer in six, which is more
than a user would put up with. Requiring both judges brings the false alarms to the edge
of the bar and the detection under it.

Two things in the detail are worth more than the totals.

- **The stronger judge is much weaker on its own SQL.** `gpt-6.1-terra` flags 54% of the
  wrong answers `gpt-6-luna` wrote and 25% of the ones it wrote itself. The smaller judge
  shows no such gap (30% and 32%), so this is one observation and not a rule, but it is
  the reason the certificate is built on deterministic checks and not on a second opinion
  from the model that wrote the answer.
- **The false alarms we read are not nonsense.** We read ten of the fifteen right answers
  that both judges flagged. The reasons given describe a difference between the statement
  and the question: a filter written as a list of values where the question says "above
  15", a count of rows where the question asks for distinct people, a code with a leading
  zero. Execution says the rows match the reference on this data. We have not checked,
  case by case, whether the statement would still be right on other data, so this is a
  lead and not a finding. If it holds, a judge would need to see the data, or its remark
  would have to be shown as the reading the query took and not as a warning.

### The same judges, shown the data

The fifteen right answers both judges flagged were then checked on the database. Nine are
right only on today's data (the list of values covers every value the column holds now,
nobody has two rows), two were the judge's mistake for not seeing how the column is stored
(the code is kept with its leading zero), and four are a matter of how the question is read.
So the judges were run again on the same 570 answers, this time shown the first rows the
statement returned and the values held by the columns it filters on, with one more verdict
to give: "answers on this data". The bar is the same.

| Judge, shown the data | Wrong answers it flags | Right answers it flags |
|---|---|---|
| `gpt-6-luna` | 135 of 285 (47.4%) | 77 of 285 (27.0%) |
| `gpt-6.1-terra` | 109 of 285 (38.2%) | 8 of 285 (2.8%) |
| `gpt-6.1-terra`, 416 right answers it had not been adjusted on | not measurable | 23 of 416 (5.5%) |

**The stronger judge reaches the bar on the answers the instructions were adjusted on and
sits just above it on the ones they were not (5.5% against 5%). That is not a pass, so
no warning is built on it.** What the run does show:

- **Data helps a strong judge and hurts a weak one.** `gpt-6.1-terra` went from 15.4% to
  2.8% of right answers flagged. `gpt-6-luna` went from 17.5% to 27.0%: with more to read
  it finds more to object to.
- **It is a check on a cheaper model's work, not on its own.** `gpt-6.1-terra` flags 45% of
  the wrong answers `gpt-6-luna` wrote (3.5% false alarms there) and 17% of the wrong
  answers it wrote itself.
- **The judge abstains often.** It said "cannot tell" on 83 of the 285 wrong answers and 53
  of the right ones. A check built on it would be silent on about a quarter of the answers.
- **Wrong answers are all in the main selection**, so detection has no figure free of the
  adjustment. The adjustment was aimed at false alarms, which is why those were measured
  again on unseen answers.

The measurement is `test/intent/measure.js` (`--list` shows the cases and the prompt without
calling a model), and every verdict, with the sentence the judge gave, is in
`docs/benchmark-runs/question-reading-1.19.1.json`. `test/intent/gather.js` collects the data shown to the judge and calls no
model. The first run cost about 345,000 input tokens per judge and the run with data about
640,000.

## What is measured

`docs/BENCHMARK.md` reports, on our own 198-question suite, how many trap questions went out
without a warning and how many correct answers were flagged anyway. `docs/BIRD.md` reports
the same on a benchmark we did not write. Read both: the first shows the checks working on
the errors they were built for, the second shows how small that share is among real errors.

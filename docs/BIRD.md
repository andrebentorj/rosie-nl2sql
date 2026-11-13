# BIRD Mini-Dev

[BIRD](https://bird-bench.github.io/) is the public text-to-SQL benchmark most products are
compared on. Mini-Dev is its development subset: 500 questions over 11 databases, each with a
reference query, scored by execution accuracy (the rows returned must equal the rows of the
reference query). This run uses the **2026 update** of Mini-Dev, published at
[huggingface.co/datasets/birdsql/bird_mini_dev](https://huggingface.co/datasets/birdsql/bird_mini_dev)
(file `data/mini_dev_postgresql.jsonl`, SHA-256 checked by `scripts/bird/setup-postgres.sh`).
That update revised questions, evidence and reference SQL, and it is harder than the
original release: its file labels 460 of the 500 questions "challenging", against 20% in the
original. Nobody at Rosie wrote the questions, the databases or the reference
answers, which is what `docs/BENCHMARK.md` cannot say about itself.

## Result

2026-11-10, model `gpt-6-luna`, PostgreSQL 17, one pass, cross-check enabled.

| | |
|---|---|
| **Execution accuracy** | **272 of 500 (54.4%)** |
| Wrong answers | 214 |
| Asked for clarification instead of answering (counted as wrong) | 14 |
| The dataset card's baseline for the same model on PostgreSQL | 51.8% |

The 51.8% is the "Baseline results" table on the
[dataset card](https://huggingface.co/datasets/birdsql/bird_mini_dev): row "GPT-6 Luna",
column PostgreSQL, 51.80, described there as "fixed-prompt baseline execution accuracy (EX,
%) on 500 questions per dialect". Read on 2026-11-10 and again on 2026-11-12. The card is a
page that can change, so the figure is quoted with its date. The same table lists stronger
models far above both numbers on PostgreSQL (GPT-6.1 Sol 89.00, Claude Opus 5.5 87.00,
Claude Sonnet 5.5 85.60), with the benchmark's own prompt and no product around the model. Rosie is 2.6 points above it. That is a small
margin from a single pass and the two setups are not identical (see Method), so read it as
"in the same range", not as a lead. What the engine adds on this benchmark is modest: a
schema summary in the prompt, tools to look up real values, and a retry when a query fails.
The answer still comes from the model, and the model is what this score mostly measures.

| Database | Correct |
|---|---|
| thrombosis_prediction | 64 / 86 (74.4%) |
| toxicology | 20 / 32 (62.5%) |
| european_football_2 | 19 / 31 (61.3%) |
| formula_1 | 33 / 55 (60.0%) |
| card_games | 36 / 69 (52.2%) |
| debit_card_specializing | 7 / 14 (50.0%) |
| financial | 23 / 46 (50.0%) |
| student_club | 18 / 37 (48.6%) |
| california_schools | 23 / 49 (46.9%) |
| codebase_community | 14 / 31 (45.2%) |
| superhero | 15 / 50 (30.0%) |

By the difficulty label in the question file: simple 6 of 8, moderate 20 of 32, challenging
246 of 460.

Median time per question 7 seconds, 90th percentile 22 seconds, 3.7 tool calls per question,
9.0 million input tokens (91% served from the provider's cache) and 0.24 million output tokens for the whole run.

## What the Answer Certificate did here, and did not

This is the part worth reading, because it is not flattering and it is the honest shape of
the feature.

| | |
|---|---|
| Wrong answers the certificate flagged | 10 of 214 (4.7%) |
| Correct answers it flagged anyway | 7 of 272 (2.6%) |
| Answers marked `VERIFIED` (the name `CROSS_CHECKED` had until version 1.0.1) | 16 |
| of which wrong by the benchmark's reference | 14 |

On Rosie's own bank the certificate catches every trap; here it catches one wrong answer in
twenty. Both numbers are true, and the difference is what kind of error is being made.

The certificate detects **structural** errors: a join that repeats rows, a filter on a value
that is not in the data, a period the data does not cover. The bank in `docs/BENCHMARK.md`
has a category built from exactly those. BIRD's wrong answers are almost all errors of
**meaning**: a different reading of the question, a column that sounds right and is not the
one the reference uses, a ratio with the wrong denominator, an extra column, a number
rounded when the reference is not. In 163 of the 214 wrong answers the result even has the
same number of rows as the reference. Nothing is structurally wrong with those queries, so
no check fires.

`VERIFIED` deserves the same plain statement. It meant the number was re-derived by a second
route over the same tables and filters and came out equal: the join did not distort it. It
never meant the question was understood. Fourteen answers here were structurally sound and
answered a different question from the one asked.

This result changed the product. In version 1.0.1 the verdict is called `CROSS_CHECKED`, its
text says "this checks the calculation, not whether the question was understood", and a
`COUNT` is no longer given that badge at all: 15 of the 16 answers above were counts, where
counting joined rows and counting distinct records are both defensible and the engine cannot
know which was meant. Under the current rules those 15 would read "checks passed", and the
one `AVG` would still be cross-checked and still be wrong, because it averaged the right way
over the wrong reading of the question. The table above is left as measured, with the rules
of the day. See `docs/CERTIFICATE.md`.

So, what this benchmark says about Rosie:

- as a text-to-SQL engine it performs like the model underneath it, slightly better;
- the certificate removes one class of silent error (the structural one) and does nothing
  for an answer that is sound and is not what was asked. Those need knowledge of the data
  and of the terms, which a context provides. None was used here, by design. What kind of
  knowledge is in "What kind of wrong the wrong answers are" below.

## Method

Each database gets a context directory built from the database alone, by
`test/bird/run.js`, with no model involved and nothing taken from the questions:

- the catalog as introspected (tables, columns, types, keys);
- for text columns with 20 or fewer distinct short values, the values themselves, so the
  model can see that a status column holds `A`, `B` and `C`;
- no column descriptions, no metrics, no verified queries, no clarification rules;
- one generic instruction, the same for all 11 databases (answer with a query, return only
  what was asked, do not round unless asked).

Each question is sent with its official "evidence" text, as the benchmark prescribes. The
final SQL of the answer and the reference SQL are both executed and their row sets compared.

Where this differs from the official evaluation script, in the direction it could matter:

- **Numbers are compared at 10 significant digits**, and a numeric value compares equal to
  the same value typed differently (`3` and `3.0`). The official script compares raw tuples.
  This is slightly more lenient.
- **A clarification counts as wrong.** The instruction forbids asking, and the model asked
  14 times anyway.
- **The input length limit was raised** (`AI_MAX_INPUT_LENGTH=6000`). The default at the time, 500
  characters, refused the longest questions with their evidence.
- **The statement timeout was 30 seconds**, and the engine's row cap was lifted for the
  comparison only, so long lists are compared whole.
- **PostgreSQL 17**, restored from the official dump files into a stock `postgres:17`
  container. The official image was used only as the source of those files.
- **One pass.** No run-to-run variance was measured, so differences of a point or two mean
  nothing.

The published 51.8% comes from the benchmark's own harness, with its own prompt and no
tools. The comparison is between two ways of using the same model, not a controlled
experiment.

## Reproduce

Needs Docker, a model key and room for the dump files (220 MB) plus the restored databases. A full run takes about 90 minutes.

```bash
./scripts/bird/setup-postgres.sh                       # restores the 11 databases on 127.0.0.1:5544
node -r dotenv/config test/bird/run.js                 # all 500 questions
node -r dotenv/config test/bird/run.js --db toxicology # one database
node -r dotenv/config test/bird/run.js --limit 5       # first 5 questions of each database
```

The report is written to `test/bird/results/` with every question's SQL, outcome and
certificate verdict.

## With the question review

The same 500 questions with `gpt-6-luna` writing and `gpt-6.1-terra` reviewing each answer
(`AI_QUESTION_REVIEW=true`, engine 1.20.0, 2026-11-13): **320 of 500 (64.0%)**, against 272
for the writer alone. 43 wrong answers were turned right by the second attempt and 4 right
ones were turned wrong. The reviewer's instructions were adjusted on answers to these same
questions, and the same run on the archived 2025 questions gave no gain (296 of 500, 6
answers fixed and 5 broken), on a set whose references are unreliable. It is a result on this set, not a figure
to quote in general. The detail and the caveats are
in `docs/CERTIFICATE.md`, "The question review".

## What kind of wrong the wrong answers are

A run only says that the rows of an answer differ from the rows of the reference.
`node test/bird/diagnose.js <run.json>` runs both statements again and says how they differ,
with no model and no opinion. It says how, not why: a total repeated by a join and a filter
on the wrong value both end in the first row of this table.

| How the answer differs from the reference | 2026 questions, 166 wrong | 2025 questions, 173 wrong |
|---|---|---|
| One value, and it is another value | 71 (43%) | 56 (32%) |
| One row of several values, differing in at least one | 24 (14%) | 11 (6%) |
| Rows that are neither part of the reference nor contain it | 34 (20%) | 26 (15%) |
| Fewer rows, more rows, or one row against several | 19 (11%) | 18 (10%) |
| Same rows, shown with other columns, another order or another format | 5 (3%) | 25 (14%) |
| Another number of columns, and no choice of columns makes the rows equal | 11 (7%) | 31 (18%) |
| No rows | 2 (1%) | 6 (3%) |

Both columns are the runs with the question review on. What it says:

- **Most wrong answers are one row whose figures are not the reference's.** That is a
  difference in what was computed, not in how the result was shown.
- **Why they differ is mostly not a matter of reading the question.** An earlier version of
  this page said it was, and that declared metrics and approved queries were the remedy. An
  independent review of this run read 65 of the wrong answers on the 2026 questions one by
  one and found another cause in about half of them: the model does not know **how the
  database records a missing value**. A question says "heroes with a known weight", the
  model writes `weight_kg IS NOT NULL`, and the table holds 0 for "not recorded" (172 of
  its 750 rows, which we checked; 125 names are `-`). The next causes it found are the
  grain of the entity (an average weighted by the rows of a join) and groups with nothing
  in them left out by an inner join. A reading of the question that differs defensibly from
  the reference's was 3 of the 65. The classification is that reviewer's, by inspection.
- **So the remedy is knowledge of the data, and it can be checked.** "Is there a weight of
  0?" is one query. What that calls for is a profile of each column in the catalog (nulls,
  zeros, empty strings, markers such as `-`), which the engine does not build yet. Metrics
  and approved queries do not address this kind of error.
- **The certificate already points at part of them and the engine does not act on it.** With
  `gpt-6-luna`, an answer carrying a fan-out or grain warning was wrong in 25 of 25, 12 of
  13 and 7 of 7 cases in the three runs; with `gpt-6.1-terra`, in 1 of 7. The warning is
  sent back to the model only for a column the context declares, and these contexts declare
  none.
- **What to show is a small part.** Answers with the reference's rows under other columns
  are 3% of the wrong ones on the 2026 questions and 14% on the 2025 ones, where the
  questions say less about what to return.
- The run on the 2025 questions also asked for a clarification 31 times, each counted as
  wrong by the benchmark's rule.

## With the benchmark's column descriptions: no gain

The benchmark's fixed-prompt baseline gives every participant a description of most columns
(`baseline_inference/schemas/postgresql.json` of the dataset: 666 of 798 columns, `a11` is
"average salary"). The runs above did not use them: the catalog was the introspected one.
This run puts them in the prompt (`AI_CATALOG_DESCRIPTIONS=true`, `BIRD_DESCRIPTIONS_FILE`),
with nothing else changed and the question review off. `gpt-6-luna`, engine 1.21.0,
2026-11-13. The bar was fixed before the run: 300 correct or more is a gain, 281 or fewer is
none.

**272 of 500 (54.4%), the same as without them.** The model called a tool 1,473 times
against 1,847 and read 5% fewer tokens, so the descriptions saved it some looking up. They
did not make it right more often. Our reading, not tested: each question here comes with an
"evidence" paragraph that already defines its terms, often by column, so the description
adds little. A user's question comes with no such paragraph, and there the description is
the only place the meaning is written. This benchmark cannot tell whether that helps.

## How much the same model changes from one run to the next

Three runs of `gpt-6-luna` over the same 500 questions now exist (2026-11-10, the first
statements of the run with the review, and the run with descriptions). They are not
identical in configuration, and the totals are 272, 281 and 272. Question by question:

| The question was answered right in | Questions |
|---|---|
| all three runs | 187 |
| two of the three | 96 |
| one of the three | 72 |
| none | 145 |

**A third of the questions (168) change outcome between runs.** Two things follow.

- A difference of ten or fifteen answers between two single runs says nothing, in either
  direction. That applies to the figures on this page, ours included.
- 355 questions were answered right at least once, against 272 to 281 in any one run. The
  right statement is often within the model's reach and not the one it wrote that time. A
  majority of the three would give 283, so voting is not what closes that distance; choosing
  among several candidates by something other than their count might. Nothing in the engine
  does that today.

## Other models

The run above uses the cheapest model on the list. The same 500 questions with a stronger
one, 2026-11-12, model `gpt-6.1-terra`, version 1.10.0, PostgreSQL 17, one pass, cross-check
enabled:

| | |
|---|---|
| **Execution accuracy** | **429 of 500 (85.8%)** |
| Wrong answers | 71 |
| Asked for clarification, timed out or gave no answer | 0 |
| Simple / moderate / challenging | 7 of 8 / 27 of 32 / 395 of 460 |
| Lowest and highest database | `debit_card_specializing` 9 of 14 (64.3%), `european_football_2` 30 of 31 (96.8%) |
| Wrong answers the certificate flagged | 6 of 71 (8.5%) |
| Correct answers flagged anyway (false alarms) | 16 of 429 (3.7%) |
| Answers marked `CROSS_CHECKED` | 2, none of them wrong |
| Answers marked `NOT_CHECKED` | 62, of which 5 wrong |
| Median time per question | 7.1 s |
| Tokens, in and out | 4.82 M and 0.13 M |
| Cost of the run at list price | about US$ 5.50, an upper bound that ignores the discount on cached input |

Two things this run does not say. It is not a like-for-like comparison of the two models: the
`gpt-6-luna` run above was made two days earlier with an older version of the engine, and the
engine changed in between (among other things it now asks once for the execution of a
statement the model only wrote out). And it confirms what the first run showed about the
certificate: on this benchmark most wrong answers are sound calculations of something else,
and it flagged 6 of 71. The certificate checks the calculation, not what was asked.

The 110-question sample in `docs/MODELS.md` had put `gpt-6.1-terra` at 86.4%; the full set
gives 85.8%.

## Not done

- MySQL and SQL Server. The benchmark publishes a MySQL version; the runner is written for
  PostgreSQL only.
- The full BIRD dev and test sets.
- A run with a tuned context (descriptions, metrics, verified queries). That would measure
  what a deployed Rosie does, and it would no longer be comparable to anyone else's number.

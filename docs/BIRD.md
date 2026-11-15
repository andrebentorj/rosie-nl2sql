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

Median time per question 7 seconds, 90th percentile 21 seconds, 3.7 tool calls per question,
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

On Rosie's own bank no trap answer went out wrong without a warning (the model avoided 21 of
the 35 by itself and the certificate flagged the other 14, `docs/BENCHMARK.md`); here the
certificate flags one wrong answer in twenty. Both numbers are true, and the difference is
what kind of error is being made.

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

- as a text-to-SQL engine it performs like the model underneath it: 2.6 points over the
  baseline in one pass is the same range, not a lead;
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
- **One pass.** Run-to-run variation was measured later (see "How much the same model
  changes from one run to the next"): two passes of the same model differ on 101 to 120 of
  the 500 questions, so differences of a few points mean nothing.
- **Dates.** The dates on this page are the local date of the machine that ran the
  benchmark (UTC-3). The report files are named by UTC time, and the files of
  `docs/BENCHMARK.md` carry UTC times, so a run that ended late in the evening has the next
  day in its file name (the `gpt-6.1-terra` run with the profile, dated 2026-11-13 here,
  ended at 02:02 UTC on 2026-11-14).

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
for the writer alone. One pass. 43 wrong answers were turned right by the second attempt and 4 right
ones were turned wrong, so 39 of the 48 answers gained are the review's: the first statements
of this same run score 281, and the other 9 are what two runs of the same model differ by.
The reviewer's instructions were adjusted on answers to these same
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
  one (its counts are from that reading by hand and are not reproducible from this
  package) and found another cause in about half of them: the model does not know **how the
  database records a missing value**. A question says "heroes with a known weight", the
  model writes `weight_kg IS NOT NULL`, and the table holds 0 for "not recorded" (172 of
  its 750 rows, which we checked; 125 names are `-`). The next causes it found are the
  grain of the entity (an average weighted by the rows of a join) and groups with nothing
  in them left out by an inner join. A reading of the question that differs defensibly from
  the reference's was 3 of the 65. The classification is that reviewer's, by inspection.
- **So the remedy is knowledge of the data, and it can be checked.** "Is there a weight of
  0?" is one query. What that calls for is a profile of each column in the catalog (nulls,
  zeros, empty strings, markers such as `-`). The engine builds it since 1.23.0 and the
  result is the next section. Metrics and approved queries do not address this kind of error.
- **The certificate already points at part of them and the engine does not act on it.** With
  `gpt-6-luna`, an answer carrying a fan-out or grain warning was wrong in 25 of 25, 12 of
  13 and 7 of 7 cases in the three runs; with `gpt-6.1-terra`, in 1 of 7. In the fourth
  `gpt-6-luna` run, the one of the next section, with the two-figure question on, 9 of the 13
  answers that still carried the warning were wrong and 4 were right. The warning is
  sent back to the model only for a column the context declares, and these contexts declare
  none. Since 1.23.0 it can be sent as a question without a declaration
  (`AI_CERTIFICATE_REPAIR_UNDECLARED`).
- **What to show is a small part, for the small model.** Answers with the reference's rows
  under other columns are 3% of its wrong ones on the 2026 questions and 14% on the 2025
  ones, where the questions say less about what to return. For the stronger model it is the
  largest group, 14 of 63 ("What the stronger model still gets wrong").
- The run on the 2025 questions also asked for a clarification 31 times, each counted as
  wrong by the benchmark's rule.

## With a profile of the data: 329 of 500

> Measured again since, with three passes a side and one change at a time: see "The first
> campaign under the rule" below. This section is the single run that came first, kept as
> it was reported.

The review quoted above proposed a remedy and a figure to test it against: give the model a
profile of each column, read from the data, and a small model should reach 60% to 63%;
under 58% and the diagnosis was wrong. The bar was fixed before the run.

What changed, all of it built from the database alone and none of it from the questions:

- **A profile of each column** (`npm run profile-columns`, `scripts/lib/column-profile.js`):
  how many rows hold no value, how many hold 0 and what the smallest other value is, blank
  strings, placeholders in use (`-`, `N/A`, `No Colour`), numbers and zero-padded codes
  kept as text. Where there is something a query writer has to know, the column gets a
  one-line note in the prompt: `weight_kg {0 in 23% of rows, the smallest other value is
  14: 0 may mean "not recorded"}`.
- **The two figures of a repeated average or total go back to the model as a question**
  where the context declares nothing (`AI_CERTIFICATE_REPAIR_UNDECLARED`), and its second
  answer is accepted only when it returns the figure over distinct records.
- **A query run to look around is never returned as the answer** (1.22.0).

`gpt-6-luna`, engine 1.23.0, 2026-11-13, one pass, no column descriptions, no question review.

| | Correct of 500 |
|---|---|
| This run | **329 (65.8%)** |
| The three earlier runs of the same model | 272, 281, 272 |

| Database | Before (run with descriptions) | This run |
|---|---|---|
| superhero | 14 of 50 | 39 |
| formula_1 | 37 of 55 | 51 |
| thrombosis_prediction | 59 of 86 | 66 |
| european_football_2 | 18 of 31 | 22 |
| california_schools | 20 of 49 | 23 |
| the other six | 124 of 229 | 128 |

How to read it.

- **It is above the predicted range and well outside what one run differs from another**:
  48 to 57 answers over each of the three earlier runs, where those differ from each other
  by 9. Of the 145 questions wrong in all three, 57 are right here.
- **It also lost answers.** Of the 187 questions right in all three earlier runs, 15 are
  wrong here. We read the 15 (a reading by hand, with no file kept). Most are the ordinary variation of the model (another
  filter, another column). One is caused by a note: a column where `-` is a test result
  ("negative") and the commonest value, which the note called a possible "not recorded",
  and the model left those rows out. The note for a placeholder that is the commonest value
  of a column with few distinct values was changed after this run to say only that, and not
  what it may mean.
- **The parts were not measured apart.** The run did not record how often the second
  attempt replaced an answer, so how much of the gain is the profile and how much is the
  question about two figures is not known. On twelve questions tried beforehand the profile
  alone turned eight (no result file is kept for that trial).
- **It was developed on this benchmark.** The profile reads only the data, but which facts
  it reports, and at what thresholds, was decided after an analysis of wrong answers to
  these same questions. The earlier edition of the benchmark cannot confirm it (its
  references are unreliable and its data traps are few). A second benchmark with dirty data
  is what would, and there is none here yet.
- **It is one pass.** Two runs of this model differ on 101 to 120 of the 500 questions.
- **`gpt-6.1-terra` alone gets 429** without any of this; with it, 437 (next section).

Every question is in `docs/benchmark-runs/bird-column-profile-1.23.0.json`. To repeat it:
`BIRD_PROFILE=true AI_CERTIFICATE_REPAIR_UNDECLARED=true node -r dotenv/config test/bird/run.js`.

### The same with the stronger model: 437 of 500, and not a proven gain

> On this run alone that was the reading. Three passes a side on engine 1.24.0 have since
> told the two apart (53 questions ahead, 33 behind, p 0.040): see "The first campaign under
> the rule" below.

`gpt-6.1-terra`, same configuration, 2026-11-13, one pass. The bar was fixed before the run:
434 or more and the profile helps this model too, 424 to 433 is what one pass varies, under
424 it gets in the way.

| | Correct of 500 |
|---|---|
| With the profile | **437 (87.4%)** |
| Without it (engine 1.10.0, 2026-11-12) | 429 (85.8%) |

It passed the bar, and the bar was too loose (see "How a gain is claimed here"). Question by question, 38 are right here that
were wrong there and 30 the other way: 68 questions changed and the balance is 8, which a
coin would give about four times in ten. What can be said is narrower.

- **Where the data records a missing value, it helps this model as well**: superhero 38 to
  48 of 50, thrombosis_prediction 75 to 81 of 86.
- **One note cost answers.** california_schools went from 38 to 32 of 49. In five of the
  nine lost there (13, 16, 17, 32, 35; we first counted four), the question gives a school
  code of 13 digits and the column holds 14 with a zero in front. Without the note the model looked at the column and added the
  zero. With the note ("a code of 14 digits kept as text, with leading zeros") it took the
  note as enough, looked at nothing, and compared the 13 digits. The note now says to pad a
  shorter code given in the question; with that wording four of them (13, 16, 17, 35) came
  out right with this model and three of the four with `gpt-6-luna`. Four questions, tried
  once: a correction, not a measurement.
- **The second attempt about two figures was not offered in this run.** The three second
  attempts on record (13, 16, 35) were for a filter value that is not in the column, and
  replaced nothing. The two wrong answers flagged for repeated rows (239, 306) are grouped
  answers, which are not sent back. So the difference here is the profile.
- **The certificate on this run**: 22 answers marked `CROSS_CHECKED`, none of them wrong;
  10 of the 63 wrong answers flagged, 14 of the 437 right ones flagged.
- **One answer counted right had no result in the engine.** On question 491 the last
  statement the model sent did not run in the engine, so the response carried the statement
  text with no rows and no certificate. The runner executes the statement text itself, and
  it matched the reference. A user would have received no table for it. Since 1.24.0 such
  a response carries no statement and says the query did not run (`unfinished`,
  `query_failed`), and the runner counts it as no answer.

Every question is in `docs/benchmark-runs/bird-column-profile-terra-1.23.0.json`.

### What the stronger model still gets wrong

The 63 wrong answers of that run, read one at a time on 2026-11-14 with the data at hand:
both statements run, the data queried for why they differ, and the engine's statement then
changed in the one respect the cause names to see that it returns the reference's rows (it
did, 63 of 63). One reader, an AI agent, not reviewed by a person. What a statement does
differently is reproducible; whether a question is ambiguous or a reference arbitrary is a
judgement, and 27 of the 63 labels are marked "likely" for that reason. Every question is in
`docs/benchmark-runs/bird-terra-wrong-answers-1.23.0.json`.

| Cause | Answers | What it is |
|---|---|---|
| The same answer in another shape | 14 | An extra column (7), or a unit, a type, an order, a layout, fewer stored digits. |
| How the data writes a value | 10 | Seven are one thing: a school code given with 13 digits against a text column of 14 with a zero in front. Then a decimal in a number kept as text, a list in one cell. |
| Two defensible readings of the question | 10 | |
| Grain | 8 | Six are a count of rows where the question counts things (cards, drivers, members). |
| What a column or a code means | 6 | Which of two similar columns, what `-` or `future` stands for, the time zone of a stored time. The data cannot say. |
| The wrong table or column | 7 | In 4 the question or the data settles which; in 3 two sources disagree and nothing does. |
| The reference is wrong or arbitrary, shown on the data | 3 | |
| A cast that loses digits | 2 | A `real` column cast to `NUMERIC` keeps six significant digits in PostgreSQL: 2052187.1 becomes 2052190. |
| An empty group, a missing value | 3 | |

How to read it.

- **It is another kind of error than the small model's.** How the data records a MISSING
  value, the commonest cause in the sample of the small model's errors read earlier, is 1
  of these 63. With this model and the profile, 14 are answers a person would accept, about
  15 are the benchmark's own (ambiguous questions and references), and the errors a customer
  would meet are how a value is written, grain, the meaning of a column, and the choice of
  source.
- **The largest single cause is one identifier format, and the note of 1.23.0 made it
  worse.** Seven answers are the unpadded school code; five of them (13, 16, 17, 32, 35) are
  right without the profile, and two (9, 39) are wrong in all three runs of this model. The
  wording of that note was changed after the run (see above).
- **The certificate saw part of it and nothing came of it.** Three of the seven were flagged
  for a filter value that is not in the column, a second attempt ran on each and replaced
  nothing. The other four were not flagged: the empty result sat in a subquery, or the
  certificate had nothing to check. Looking up a value the question gives before answering
  is what these call for, and the engine does not do it yet.
- **A count of rows is a grain error the certificate cannot see.** The repeated-rows check
  looks at sums and averages. It passed all six `COUNT(*)` answers that counted printings
  for cards or results for drivers. The two grain errors it did flag (239, 306) are grouped
  answers, which are not sent back to the model.
- **By cause, 6 of the 10 flags on wrong answers name the error** (13, 16, 35, 162, 239,
  306) and 4 are beside it (157, 304, 467, 483).
- **A ceiling, not a forecast.** If every error the engine could address were fixed and
  nothing else broke (23 answers), this run would read 460 of 500; with the 14 answers in
  another shape, 474. The profile itself gained 38 answers and lost 30 on this model, so no
  fix is free, and 18 of the 63 were right in both other runs of the model.
- **Not determined**: what the notes of that run said (they are not stored), what the three
  second attempts wrote, whether two of the answers under "meaning" (259, 288, a start time
  converted from UTC) are the engine being right against the reference, and whether a
  second reader would draw the same lines.

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

**A third of the questions (168) change outcome across the three runs.** Between any two of
them the count is 101, 115 or 120 (20% to 24%), and part of that is the difference in
configuration, which these runs cannot separate from chance. Two things follow.

- A difference of ten or fifteen answers between two single runs says nothing, in either
  direction. That applies to the figures on this page, ours included.
- 355 questions were answered right at least once, against 272 to 281 in any one run. The
  right statement is often within the model's reach and not the one it wrote that time. A
  majority of the three would give 283, so voting is not what closes that distance; choosing
  among several candidates by something other than their count might. Nothing in the engine
  does that today.

## How a gain is claimed here

Since 1.23.2. Two totals side by side are not a comparison: two runs of the same model
differed on 101 to 120 of the 500 questions, and 168 changed outcome across three runs
(runs that were not identical in configuration, so this is an upper bound on chance alone).
The rule for every figure added to this page from now on:

1. **Three passes of each configuration** (`test/bird/run.js --passes 3`), each kept as its
   own report. The page gives the three totals, not the best one.
2. **Compared question by question** (`npm run bird:compare -- --a <reports> --b <reports>`).
   For each question the share of passes that got it right is taken on each side; the
   questions where one side is ahead and where the other is are counted, and the exact sign
   test is applied to the two counts.
3. **A gain is claimed when p is under 0.05, and not otherwise.** A bar written before a run
   is written in these terms ("ahead, p under 0.05"), not as a total to reach.
4. **One thing changed at a time.** The tool reads what each report says about its
   configuration and says so when two sides differ in more than one thing, or in none.

The runs above, read this way, from the reports in `docs/benchmark-runs`. They are single
passes, made before the rule, and the tool says so.

| Comparison | Totals | Ahead / behind | p | Reading |
|---|---|---|---|---|
| `gpt-6-luna`, profile and two-figure question against two earlier runs | 329 against 272, 272 | 141 / 72 | 0.000003 | a gain |
| `gpt-6.1-terra`, the same against one earlier run | 437 against 429 | 38 / 30 | 0.40 | not told apart |
| `gpt-6-luna`, the engine against the model alone | 272 against 215 | 101 / 44 | 0.000003 | a gain |
| `gpt-6.1-terra`, the engine against the model alone | 429 against 381 | 68 / 20 | 0.0000003 | a gain |

The files, all in `docs/benchmark-runs`: `bird-model-alone-luna-1.23.2.json`,
`bird-model-alone-terra-1.23.2.json`, `bird-engine-luna-1.0.0.json`,
`bird-engine-luna-described-1.21.0.json`, `bird-engine-terra-1.10.0.json`,
`bird-column-profile-1.23.0.json`, `bird-column-profile-terra-1.23.0.json`. The first row
is `npm run bird:compare -- --a docs/benchmark-runs/bird-engine-luna-1.0.0.json,docs/benchmark-runs/bird-engine-luna-described-1.21.0.json --b docs/benchmark-runs/bird-column-profile-1.23.0.json`.

The tool prints its cautions after the result, and three of them stand here: a side with
one pass counts a question that changes between runs as fully right or fully wrong; the two
earlier `gpt-6-luna` runs are not one configuration (one had the benchmark's column
descriptions, which changed nothing in the total); and the two sides differ in two things at
once (the profile and the two-figure question). The tool can only say the last when the
reports carry their configuration, which the reports in `docs/benchmark-runs` do. The
`gpt-6.1-terra` sides also differ in engine version (1.10.0 against 1.23.0).

### The first campaign under the rule: engine 1.24.0, three passes a side

2026-11-14 and 15, engine 1.24.0, the 500 questions, PostgreSQL. Five configurations, three
passes each, and three comparisons, each between two sides that differ in one thing. The
configurations, the comparisons and what each outcome would be reported as were written
before the first pass; no other comparison from these passes is reported as a result.

| Configuration | `gpt-6-luna` | `gpt-6.1-terra` |
|---|---|---|
| The engine | 273, 278, 263 (mean 271.3) | 430, 431, 425 (mean 428.7) |
| With the column profile (`BIRD_PROFILE=true`) | 319, 310, 330 (mean 319.7) | 445, 439, 446 (mean 443.3) |
| With the profile and the two-figure question (`AI_CERTIFICATE_REPAIR_UNDECLARED=true`) | 339, 331, 326 (mean 332.0) | not run |

| Comparison | Ahead / behind | p | Reading |
|---|---|---|---|
| `gpt-6-luna`, the profile against the engine | 142 / 72 | 0.000002 | a gain |
| `gpt-6-luna`, the two-figure question added to the profile | 98 / 67 | 0.019 | a gain |
| `gpt-6.1-terra`, the profile against the engine | 53 / 33 | 0.040 | a gain |

How to read it.

- **The profile is a gain for both models**, about 48 answers of 500 for the small one and
  about 15 for the stronger one. This replaces the two single runs above: the 329 of
  `gpt-6-luna` mixed two changes, and the 437 of `gpt-6.1-terra` was not told apart from
  chance.
- **Most of it is one kind of data.** For `gpt-6-luna` the superhero database goes from 13.0
  to 37.7 right of 50 (31 questions ahead, 4 behind); formula_1 and financial follow (20 / 7
  and 17 / 3). For `gpt-6.1-terra` superhero goes from 38.3 to 47.0 (12 / 1) and
  california_schools from 37.7 to 40.7 (12 / 7), and that is most of the balance; on five of
  the eleven databases the profile is level or slightly behind. Take away superhero and the
  stronger model's result is 41 ahead and 32 behind, which is not a gain by this rule. So
  the claim for that model is: it helps where the data records a missing value as a value,
  and it was not seen to cost anything elsewhere.
- **The two-figure question is a gain for the small model, and a small one.** About 12
  answers of 500. In the three passes it was put 14, 16 and 12 times; the second statement
  replaced the first 10, 12 and 9 times, and was then right 9, 11 and 9 times. It was not
  run with the stronger model, so nothing is claimed there. It stays off by default: it
  changes an answer on the strength of a second model call, and the measurement is on one
  benchmark.
- **p of 0.040 and 0.019 are under the line, not far under it.** Three comparisons were
  made, and with three the chance of one false "gain" at this line is larger than one in
  twenty. The first row would stand under any correction; the other two would not stand
  under the strictest one.
- **It is still the benchmark the profile was developed on.** Three passes settle whether
  the difference is chance. They do not settle whether it carries to other data.
- **Between passes of one configuration** the totals moved by up to 15 (`gpt-6-luna`, the
  engine) and 20 (with the profile), and 130 to 146 of the 500 questions changed outcome
  across the three passes of a `gpt-6-luna` configuration, 42 to 50 for `gpt-6.1-terra`.
- **The certificate did not become a mark of a right answer here either.** With
  `gpt-6-luna` and the engine alone, 15, 15 and 18 of the 17, 19 and 21 answers marked
  `CROSS_CHECKED` were wrong by the benchmark: the figure was computed without repeated
  rows, and it was the figure of another question. With `gpt-6.1-terra` it was 1, 2 and 2
  of 23, 23 and 25.
- **One pass is not in the count.** The model provider's account ran out of credit during
  the first attempt at the campaign. One `gpt-6-luna` pass of the engine alone (268 right)
  lost its last 13 questions to that and was replaced by a full pass; with it added as a
  fourth pass the first comparison reads 168 / 98, p 0.00002. The passes that ran with no
  credit at all were discarded.

The fifteen passes are in `docs/benchmark-runs`, named
`bird-pass-<model>-<configuration>-<n>-1.24.0.json` (`plain`, `profile`, `both`). The last
row of the second table is
`npm run bird:compare -- --a docs/benchmark-runs/bird-pass-terra-plain-1-1.24.0.json,docs/benchmark-runs/bird-pass-terra-plain-2-1.24.0.json,docs/benchmark-runs/bird-pass-terra-plain-3-1.24.0.json --b docs/benchmark-runs/bird-pass-terra-profile-1-1.24.0.json,docs/benchmark-runs/bird-pass-terra-profile-2-1.24.0.json,docs/benchmark-runs/bird-pass-terra-profile-3-1.24.0.json`.

### What a cross-checked wrong answer was, and the check that came of it

`gpt-6-luna` with the engine alone got a `CROSS_CHECKED` verdict on 26 different questions
across its three passes, and on 22 of them at least one such answer was wrong. We read the
22 (2026-11-15, question, reference statement and engine statement side by side). In every
one the verdict said something true: the total or average was computed without repeated
rows. What was wrong was something else.

| What was wrong | Questions |
|---|---|
| An average that counts rows where 0 stands for "not recorded" (the question says "known", "measured" or "recorded", the statement says `IS NOT NULL`) | 10, all in superhero: 343, 345, 348, 355, 356, 357, 363, 369, 370, 371 |
| Another reading of the question, each its own: one card per printing instead of one per name (110), a place matched too narrowly (144), a value "in force on" a date read as "recorded on" it (176), points taken from another table (247), drivers with no rows left out instead of shown as 0 (266), all expenses of the day instead of one person's (318), only the budget lines holding the expense instead of every line of the event (322), an item list kept as one text (327), an extra column or another label (165, 220, 239), a filter by exclusion where the reference lists the values (175) | 12 |

The second row is what the certificate does not look at and says it does not: whether the
statement answers the question. The first row is a fact about the DATA, and the data can be
asked. Since 1.26.0 the certificate has a check for it (`zero_values`,
`docs/CERTIFICATE.md`): one probe over the rows the answer averaged counts the zeros and
finds the smallest value above 0, and when there are zeros followed by a gap the answer is
`FLAGGED`, with the count and the average without them.

Run again, with no model, over the final statement of every answer of the fifteen passes
(7,500 answers less those with no statement):

| Configuration | Answers it flags, per pass | Of them wrong | Of them `CROSS_CHECKED` before |
|---|---|---|---|
| `gpt-6-luna`, the engine | 14, 13, 14 | all | 8, 8, 8 (of 15, 15 and 18 cross-checked wrong answers) |
| `gpt-6-luna`, with the profile | 0, 1, 1 | all | 0, 0, 1 |
| `gpt-6-luna`, with the profile and the two-figure question | 1, 1, 0 | all | 1, 1, 0 |
| `gpt-6.1-terra`, the engine | 4, 4, 4 | all | 1, 2, 2 (of 1, 2 and 2) |
| `gpt-6.1-terra`, with the profile | 0, 0, 0 | | |

- **57 answers flagged, all 57 wrong, none of the right answers flagged.** A first version
  flagged 10 right answers on two questions: averages written as `AVG(x) FILTER (WHERE x <> 0)`,
  which had already left the zeros out. That form is now skipped, and the figures above are
  after the correction.
- **It is one database of the eleven.** Every one of the 57 is in superhero, where height
  and weight hold 0 for "not recorded". On the other ten the check said nothing, right or
  wrong. So this measures that it is quiet where it has nothing to say and catches the one
  pattern it was written for, on the data that showed us the pattern. It is not evidence of
  how often other data has it.
- **It was written after reading these answers**, so it is fitted to them in the same way
  the column profile is. With the profile on, the model mostly avoids the error and the
  check has little left to catch; they are two answers to the same fact, one before the
  statement is written and one after.
- **The second attempt for it was not measured.** With `AI_CERTIFICATE_REPAIR_UNDECLARED`
  the flag goes back to the model as a question; no pass has been run with that.

**Three figures per model** are what a reader needs to tell the model from the engine: the
model alone in one call with the same catalog text (`BIRD_FIXED_PROMPT=true`, no tools, no
second look, no certificate), the engine, and the engine with the profile of the data.

| Model | Model alone, one call | Engine | Engine with the profile |
|---|---|---|---|
| `gpt-6-luna` | 215 (43.0%) | 273, 278, 263 | 319, 310, 330 |
| `gpt-6.1-terra` | 381 (76.2%) | 430, 431, 425 | 445, 439, 446 |

The first column is our own prompt, not the benchmark's baseline: engine 1.23.2, 2026-11-14,
one pass. The other two columns are the three passes of the campaign above, engine 1.24.0.
The engine against the model alone was not one of the comparisons of that campaign and the
two were run on different engine versions, so the only reading of it is the one made from
single passes (the last two rows of the first table of this section): the engine is ahead
for both models, by a distance that was not measured under the rule.

## Other models

The first run on this page (Result) uses the cheapest model on the list. The same 500 questions with a stronger
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
`gpt-6-luna` run of the Result section was made two days earlier with an older version of the engine, and the
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

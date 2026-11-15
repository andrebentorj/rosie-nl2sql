# Choosing a model

Rosie works with any endpoint that speaks the OpenAI chat-completions format with function
tools (`AI_MODEL`, `AI_BASE_URL`), and with OpenAI models that take function tools only
through the responses API. The model decides most of the accuracy: the engine adds
context, tools, checks and a second attempt, and the model still writes the query.

`npm run test:provider` makes one real tool call with the configured model and tells you in a
few seconds whether it can be used. Run it before changing `AI_MODEL`.

## Measured on a sample of 110 questions

The same 110 questions of BIRD Mini-Dev (2026 update, PostgreSQL): the first 10 of each of
the 11 databases, context built from the schema alone, no tuning, one pass, 2026-11-11
(version 1.1.0; `gpt-6.1-sol` on 1.2.0, the first that can call it). Execution accuracy is the benchmark's own measure: the rows returned equal the
rows of the reference query.

| Model | Correct | Wrong | Other | Median time | Input tokens (cached) | Output tokens |
|---|---|---|---|---|---|---|
| `gpt-6.1-sol` | **96 of 110 (87.3%)** | 14 | 0 | 8.5 s | 1.14 M (1.04 M) | 30 k |
| `gpt-6.1-terra` | 95 of 110 (86.4%) | 14 | 1 timeout | 6.9 s | 0.97 M (0.87 M) | 23 k |
| `gpt-6-sol` | 89 of 110 (80.9%) | 21 | 0 | 5.3 s | 1.09 M (0.99 M) | 32 k |
| `gpt-6-luna` (the default) | 78 of 110 (70.9%) | 30 | 2 clarifications | 5.2 s | 1.24 M (1.12 M) | 32 k |

Cost at list price, ignoring the discount providers give on cached input, so an upper bound:
about US$ 0.0013 per question on `gpt-6-luna` (US$ 0.10 and 0.50 per million tokens) and
about US$ 0.010 on `gpt-6.1-terra` (US$ 1.00 and 5.00) and about US$ 0.023 on `gpt-6.1-sol`
(US$ 2.00 and 10.00). The prices per million tokens are list prices that nothing in this
package checks; confirm them before budgeting. `gpt-6.1-terra` is roughly eight times
the price of the default for 15 points of accuracy on this sample, and for 31 points on the
whole set (below); `gpt-6.1-sol` costs more than twice as much again for one more question
right out of 110, which is noise.

Read these numbers as a comparison between models, not as accuracy you will get:

- **The sample flatters the small model.** The first ten questions of each database are
  easier for `gpt-6-luna` than the rest: in its full run of 2026-11-10 it got 77 of these
  same 110 questions right (70.0%) and 272 of all 500 (54.4%). The engine version is not
  what separates 70.9% from 54.4%; the sample is. For `gpt-6.1-terra` the sample was
  representative: 86.4% here, 85.8% on all 500 (97 of these 110 in that run). So the distance
  between the two models is 15 points on this sample and 31.4 points on the whole set.
  `gpt-6-sol` and `gpt-6.1-sol` have not been run on all 500.
- **One pass, 110 questions.** A difference of three or four questions is noise.
- **No context.** The catalog is the introspected one. What a context adds was measured
  only in part: the benchmark's own column descriptions gave no gain on the 500 questions,
  and a profile of the data did (`docs/BIRD.md`). Declared metrics and approved queries
  were not measured on this benchmark.

## Measured on all 500 questions

BIRD Mini-Dev, 2026 update, PostgreSQL, context built from the schema alone. The first row
is a single pass; the other three are three passes each on engine 1.24.0 (2026-11-14 and
15), compared question by question. Method, caveats and the per-question files are in
`docs/BIRD.md`.

| | `gpt-6-luna` | `gpt-6.1-terra` |
|---|---|---|
| The model alone: one call, the same catalog text, no tools, no checks (engine 1.23.2) | 215 (43.0%) | 381 (76.2%) |
| The engine | 273, 278, 263 (mean 54.3%) | 430, 431, 425 (mean 85.7%) |
| The engine with the profile of the data | 319, 310, 330 (mean 63.9%) | 445, 439, 446 (mean 88.7%) |
| The same with the two-figure question | 339, 331, 326 (mean 66.4%) | not run |

- **The profile is a gain for both models** by the question-by-question test of
  `docs/BIRD.md`: 142 questions ahead and 72 behind for `gpt-6-luna` (p 0.000002), 53 and 33
  for `gpt-6.1-terra` (p 0.040). For the stronger model nearly all of it is in data that
  records a missing value as a value (one database of the eleven carries the result), and
  the profile was developed on this same benchmark.
- **The two-figure question adds about 7 answers for `gpt-6-luna`.** The totals differ by
  12 (98 ahead, 67 behind, p 0.019), and 5 or 6 of those are on questions where it never
  acted (`docs/BIRD.md`). It was not run with `gpt-6.1-terra`.
- The earlier single runs (329 and 437 on engine 1.23.0) are replaced by these.
- The first row is our own prompt, not the benchmark's baseline. By the same test the engine
  is ahead of the model alone for both models: 101 questions ahead and 44 behind for
  `gpt-6-luna`, 68 and 20 for `gpt-6.1-terra` (p under 0.00001 for each). They are single
  passes on different engine versions.
- Cost per question at list price, ignoring the discount on cached input: about US$ 0.002
  on `gpt-6-luna` (18,000 input and 480 output tokens) and about US$ 0.011 on
  `gpt-6.1-terra` (9,600 and 265). The 110-question sample above came to about US$ 0.0013 and
  US$ 0.010.

## What we recommend

- **Pilot and production where a wrong number costs something:** `gpt-6.1-terra`. On the
  sample it matches the strongest model measured at less than half the price and is faster,
  and on all 500 questions it is 31 points above the default (three passes each), at about
  US$ 0.011 per question against about US$ 0.002. Run `npm run profile-columns -- --write`
  for it as well where your data records a missing value as a value (a 0, a `-`): that is
  where its 15 answers of 500 came from, and without that one database the gain is not told
  apart from chance. Have someone who knows the data read the notes it writes.
- **High volume, simple questions, cost-sensitive:** `gpt-6-luna`. It asks for clarification
  more readily and costs between a fifth and an eighth, depending on the questions. Run
  `npm run profile-columns -- --write` for it: the profile of the data is the largest gain
  measured for this model, about 48 answers of 500 on the benchmark it was developed on.
  `AI_CERTIFICATE_REPAIR_UNDECLARED=true` added about 7 more there; it is off by default
  (`docs/BIRD.md`).
- **Data that must not leave the network:** a model you host behind an OpenAI-compatible
  endpoint (`docs/DATA_PRIVACY.md`). Measure it with your own question bank first.

## Models that need the responses API

`gpt-6.1-sol` and `gpt-6-astra` refuse function tools on the chat-completions API and take
them only through the responses API. Since 1.2.0 the engine handles this by itself: it
starts on chat completions, and when the provider answers that a model's tools need the
other API it switches that model for the life of the process (one failed request, logged as
`model_api_switched`). `AI_API=responses` or `AI_API=chat` forces one for every model.
`npm run test:provider` prints which API a model ends up on. Only `gpt-6.1-sol` has been
measured this way; an OpenAI-compatible gateway that is not OpenAI will normally offer chat
completions only.

## Two things the engine does for stronger models

- **A query written and not run.** Stronger models often answer "here is the SQL" and stop.
  On the first run of this sample `gpt-6-sol` did that on 67 of 110 questions and
  `gpt-6.1-terra` on 32: the user would have received a statement and no rows. The engine
  now asks once for the query to be run. The table above is with that in place.
- **Reasoning effort.** Function tools on chat completions need `reasoning_effort: none` on
  current models; the engine sets it per model family and adjusts if a provider refuses it
  (`AI_REASONING_EFFORT` overrides). On the responses API the lowest effort is `low`.

## Reproduce

```bash
./scripts/bird/setup-postgres.sh
AI_MODEL=gpt-6.1-terra node -r dotenv/config test/bird/run.js --limit 10
```

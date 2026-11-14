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

BIRD Mini-Dev, 2026 update, PostgreSQL, context built from the schema alone. Every figure is
a single pass, and two passes of the same model differ on 101 to 120 of the 500 questions,
so read differences of a few points as nothing. Method, caveats and the per-question files
are in `docs/BIRD.md`.

| | `gpt-6-luna` | `gpt-6.1-terra` |
|---|---|---|
| The model alone: one call, the same catalog text, no tools, no checks (engine 1.23.2) | 215 (43.0%) | 381 (76.2%) |
| The engine | 272, 281 and 272 (54.4%, 56.2%, 54.4%) | 429 (85.8%), engine 1.10.0 |
| The engine with the profile of the data and the two-figure question (engine 1.23.0) | 329 (65.8%) | 437 (87.4%) |

- The three engine runs of `gpt-6-luna` are not one configuration (the first run, the first
  statements of the run with the question review, and a run with the benchmark's column
  descriptions).
- 329 against the earlier `gpt-6-luna` runs is a gain by the question-by-question test of
  `docs/BIRD.md` (141 questions ahead, 72 behind, p 0.000003). It mixes two changes that
  were not measured apart, and both were developed on this same benchmark.
- 437 against 429 is not told apart from chance (38 questions ahead, 30 behind, p 0.40).
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
  and on all 500 questions it is 31 points above the default in one pass each, at about
  US$ 0.011 per question against about US$ 0.002.
- **High volume, simple questions, cost-sensitive:** `gpt-6-luna`. It asks for clarification
  more readily and costs between a fifth and an eighth, depending on the questions. Run
  `npm run profile-columns -- --write` for it: the profile of the data is the one change
  measured to help this model (`docs/BIRD.md`).
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

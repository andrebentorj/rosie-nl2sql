# Choosing a model

Rosie works with any endpoint that speaks the OpenAI chat-completions format with function
tools (`AI_MODEL`, `AI_BASE_URL`). The model decides most of the accuracy: the engine adds
context, tools, checks and a second attempt, and the model still writes the query.

`npm run test:provider` makes one real tool call with the configured model and tells you in a
few seconds whether it can be used. Run it before changing `AI_MODEL`.

## Measured

The same 110 questions of BIRD Mini-Dev (2026 update, PostgreSQL): the first 10 of each of
the 11 databases, context built from the schema alone, no tuning, one pass, version 1.1.0,
2026-11-11. Execution accuracy is the benchmark's own measure: the rows returned equal the
rows of the reference query.

| Model | Correct | Wrong | Other | Median time | Input tokens (cached) | Output tokens |
|---|---|---|---|---|---|---|
| `gpt-6.1-terra` | **95 of 110 (86.4%)** | 14 | 1 timeout | 6.9 s | 0.97 M (0.87 M) | 23 k |
| `gpt-6-sol` | 89 of 110 (80.9%) | 21 | 0 | 5.3 s | 1.09 M (0.99 M) | 32 k |
| `gpt-6-luna` (the default) | 78 of 110 (70.9%) | 30 | 2 clarifications | 5.4 s | 1.24 M (1.12 M) | 32 k |

Cost at list price, ignoring the discount providers give on cached input, so an upper bound:
about US$ 0.0013 per question on `gpt-6-luna` (US$ 0.10 and 0.50 per million tokens) and
about US$ 0.010 on `gpt-6.1-terra` (US$ 1.00 and 5.00). Roughly eight times the price for 15
points of accuracy on this sample.

Read these numbers as a comparison between models, not as accuracy you will get:

- **The sample is easier than the whole set.** On all 500 questions `gpt-6-luna` scored 54.4%
  (`docs/BIRD.md`); on these 110 it scores 70.9%. The other two have not been run on all 500.
- **One pass, 110 questions.** A difference of three or four questions is noise.
- **No context.** A deployed Rosie has column descriptions, declared metrics and approved
  queries, which move accuracy more than the model does on a schema with obscure names.

## What we recommend

- **Pilot and production where a wrong number costs something:** `gpt-6.1-terra`, or the
  strongest model your provider offers that passes `npm run test:provider`.
- **High volume, simple questions, cost-sensitive:** `gpt-6-luna`. It asks for clarification
  more readily and costs about a tenth.
- **Data that must not leave the network:** a model you host behind an OpenAI-compatible
  endpoint (`docs/DATA_PRIVACY.md`). Measure it with your own question bank first.

## Models that do not work

`gpt-6.1-sol` and `gpt-6-astra` refuse function tools on the chat-completions API (they
require the newer responses API, which Rosie does not speak yet). `npm run test:provider`
reports it. This matters because `gpt-6.1-sol` leads the benchmark's own baseline table.

## Two things the engine does for stronger models

- **A query written and not run.** Stronger models often answer "here is the SQL" and stop.
  On the first run of this sample `gpt-6-sol` did that on 67 of 110 questions and
  `gpt-6.1-terra` on 32: the user would have received a statement and no rows. The engine
  now asks once for the query to be run. The table above is with that in place.
- **Reasoning effort.** Function tools on this API need `reasoning_effort: none` on current
  models; the engine sets it per model family and adjusts if a provider refuses it
  (`AI_REASONING_EFFORT` overrides).

## Reproduce

```bash
./scripts/bird/setup-postgres.sh
AI_MODEL=gpt-6.1-terra node -r dotenv/config test/bird/run.js --limit 10
```

# Data and privacy

What leaves your environment when someone asks Rosie a question, what never does, and how
to run with nothing leaving at all.

Rosie itself is software you run: the server, the database connections and the audit trail
stay in your infrastructure. The one external party is the **model provider** configured in
`AI_BASE_URL` / `OPENAI_API_KEY`. Everything below is about what is sent to it.

## Sent to the model provider

| What | Detail |
|---|---|
| The question | As typed by the user, after the length limit and the injection filter. |
| Conversation history | The previous turns of the same conversation: at most 10 are accepted and sanitized again on every request, and the last 6 are sent. |
| The previous answer, on a follow-up | When the request carries `previousAnswerHandle`: the SQL and the question of that answer, so the model starts from them. |
| Your system prompt | The full text of `system-prompt.txt`. |
| Schema metadata | Table names, descriptions and synonyms; names, type family and enum codes of the columns that are **not** masked; relationships; metric definitions. |
| Column descriptions and code meanings | What the catalog says each unmasked column means, and the meaning of each enum code: when the model calls `describe_table`, and in every prompt with `AI_CATALOG_DESCRIPTIONS=true`. |
| Data notes (`profileNote`) | Only if the catalog carries them. `npm run profile-columns` writes them **from your data**: how an unmasked column records a missing or special value (for example that 0 or `-` stands for "not recorded", with the smallest other value, or that codes are zero-padded text). They are counts and patterns, not rows, but they are derived from data and go into every prompt while `AI_CATALOG_PROFILE` is on, which is the default. |
| Verified examples | The few question and SQL pairs from `verified-queries.json` most similar to the question. |
| Query results, sampled | Up to **30 rows** of each query the model runs (`AI_LLM_RESULT_MAX_ROWS`), with the true total. A question may run up to 5 queries (`AI_MAX_QUERIES_PER_REQUEST`). The user receives the full result; the model sees the sample. |
| Sample rows | When the model calls `sample_rows`: at most 5 rows of one table, unmasked columns only, text cut at 100 characters. |
| Distinct values | When the model calls `lookup_values`: at most 25 distinct values of one unmasked column. |
| The SQL of an answer | When the user asks for an explanation of a query (`/explain`), with the question. |
| The second attempt | When the certificate flags a first query (`AI_CERTIFICATE_REPAIR`, on by default): what the checks found, figures included, goes back to the model once. |
| The question review | Only with `AI_QUESTION_REVIEW=true` (off by default): the question, the final statement, its first 8 rows, and for up to 6 unmasked columns the statement filters on, up to 40 of their values with how many rows hold each. It goes to `AI_QUESTION_REVIEW_MODEL`, which may be a different model from the one that answers, at the same provider endpoint. |

Results are sent because the model writes the plain-language answer from them. Set
`AI_LLM_RESULT_MAX_ROWS` lower to send less; the answer then describes fewer rows.

### While the context is generated

`npm run init` is the other moment something is sent. To write the context it sends the
model provider your schema: table and column names, types, keys and relationships. With
`--samples` it also sends sample values of unmasked columns, which gives better
descriptions and enum codes; that is off by default. The columns it masks are left out of
the samples. It runs once, at install time, with the key in `OPENAI_API_KEY`.

## Never sent

- **Masked columns.** Columns in `masked-columns.json` are absent from the schema metadata,
  refused by the SQL validator, left out of `sample_rows` and refused by `lookup_values`.
  Their values never reach the provider. Since 1.24.0 their names do not either: before
  that version the `describe_table` tool returned the name, type and description of a
  masked column when the model asked for its table. Masking is the control for personal
  data: list every column that holds it. `npm run generate-catalog` prints the column names
  that look like personal data and are not masked.
- **Rows outside the caller's policy.** With [row policies](ROW_POLICIES.md), every read is
  filtered by the caller's verified identity before it runs, so the sample sent to the
  provider only holds rows that caller may see.
- **Credentials and configuration.** Database hosts, users, passwords, tokens and
  environment variables are never part of a prompt. Questions that ask for them are refused
  before the model is called.
- **The rest of a large result.** Rows beyond the sample go to the user's browser or export
  and are not sent to the provider.
- **Your audit trail.** Questions, generated SQL and certificates are written to your
  database and to a local file. Nothing is reported to the vendor of Rosie: the software
  makes no calls other than to your database, your model provider and, when JWT verification
  is configured with a key URL, your identity provider.

## What the provider does with it

That is governed by your agreement with the provider, not by Rosie. Check three things
before production: whether API data is used for training (most commercial API terms say it
is not, by default), how long requests are retained, and in which region they are
processed. If your organisation has a zero-retention or regional agreement, use the endpoint
and key that agreement covers.

## Running with nothing leaving your network

Rosie talks to any endpoint that implements the OpenAI chat completions API with function
calling. Point it at a model you host:

```bash
AI_BASE_URL=http://llm.internal:8000/v1     # vLLM, Ollama, LiteLLM, a private gateway
AI_MODEL=<the model name that endpoint serves>
OPENAI_API_KEY=<whatever that endpoint expects; any non-empty value if it expects none>
```

Then run `npm run test:provider`. It makes one real round trip with a tool call against the
configured endpoint and fails loudly if the model cannot call functions, which is the
capability Rosie depends on.

Two cautions, stated plainly:

- **Accuracy depends on the model.** The published benchmark (`docs/BENCHMARK.md`) was
  measured with one hosted model. A smaller self-hosted model will write worse SQL. Measure
  your endpoint on your own questions before you rely on it: `npm run evaluate`
  (`docs/EVALUATION.md`).
- **The safety layers do not depend on the model.** The SQL validator, the table and column
  allowlist, masking, row policies and the Answer Certificate are deterministic code that
  runs in your server whatever model is behind `AI_BASE_URL`. A weaker model produces more
  refusals and more flagged answers, not more leaks.

Self-hosted endpoints have not been part of the automated test suite of this repository;
`test:provider` is the check to run on yours.

## Retention on your side

| Store | What it holds | Where |
|---|---|---|
| `rosie_audit_log` | One row per question: user id, question, generated SQL, outcome, timings. | The engine database (`DB_ENGINE_DATABASE`). |
| `rosie_answer_certificate` | The certificate of each answer that carried one. | Same. |
| `rosie_feedback` | Thumbs up or down, with the question and SQL. | Same. |
| `rosie_rate_limit` | Only with `ROSIE_RATE_LIMIT_STORE=database`: one counter per client address and window. Removed by the engine about two windows later (two days for the daily quota). | Same. |
| Audit file | The same audit records as newline-delimited JSON, as a fallback when the database is down. | `ROSIE_AUDIT_FILE` (default `logs/`). |

Result rows are not stored by Rosie. Retention of these tables is yours to set, and nothing
is removed unless you schedule it:

```bash
npm run retention -- --days 365               # report: how many records are older
npm run retention -- --days 365 --confirm     # remove them from the three log tables
```

The command touches only `rosie_audit_log`, `rosie_answer_certificate` and `rosie_feedback`,
and the account that runs it needs DELETE on them (`docs/INSTALL.md`, Step 3). The audit file is not rotated by Rosie; use
your log tooling for it.

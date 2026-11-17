# Data and privacy

What leaves your environment when someone asks Rosie a question, what is kept out and by
which layer, and how to run with nothing leaving at all.

Rosie itself is software you run: the server, the database connections and the audit trail
stay in your infrastructure. The one external party is the **model provider** configured in
`AI_BASE_URL` / `OPENAI_API_KEY`. Everything below is about what is sent to it.

## Sent to the model provider

| What | Detail |
|---|---|
| The question | As typed by the user, after the length limit and the injection filter. |
| Conversation history | The previous turns of the same conversation: at most 10 are accepted and sanitized again on every request, and the last 6 are sent. |
| The previous answer, on a follow-up | When the request carries `previousAnswerHandle`: the SQL and the question of that answer, so the model starts from them. Since 1.30.0 only when that SQL still passes the validator against the current context. |
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
the samples. To write the test questions that must be refused, it also sends the provider
the NAMES of the columns it has just masked, not their values. Up to 1.29.2, running it
again after masking more columns could send values of those columns cached from the
earlier run, and their old enum codes; since 1.30.0 masked columns are left out of the cache
and the codes at every use. It reads with the credentials in `DB_USER`, not the read-only
account, so column grants on the read-only account do not limit it. It runs at install time,
with the key in `OPENAI_API_KEY`.

## Kept out of what is sent

- **Masked columns.** Columns in `masked-columns.json` are absent from the schema metadata
  Rosie writes, refused by the SQL validator when it finds them in a statement, left out of
  `sample_rows` and refused by `lookup_values`. This is Rosie's own reading of SQL.
  Independent reviews have found ways around it more than once, most recently in 1.29.2
  (see the advisory in [SECURITY_OVERVIEW.md](SECURITY_OVERVIEW.md)), and the database
  account can still read those columns unless you take the privilege away. With the
  privilege taken away, a statement that reaches a masked column is refused by your
  database, whatever Rosie's own checks miss, except through a view, a function or a copy of
  the column that the account can read. Rosie checks this at startup and reports "masked by
  the database" or "masked by Rosie only" in `/health`; on SQL Server it reports "not
  verified on SQL Server", because the check has not been verified on a live server. Rosie's own description
  of your schema does not name masked columns, their relationships, or metrics and examples
  that read them. Text you write (the system prompt, descriptions, metric definitions,
  examples) is sent as you wrote it. `npm run check-context` fails where that text writes a
  masked `table.column` and warns where it contains the column's words; it cannot see a
  paraphrase. Before 1.24.0 the `describe_table` tool returned the name, type and
  description of a masked column when the model asked for its table. Two more paths were closed in 1.26.0,
  both found by independent reviews: a `JOIN ... USING (column)` on a masked column passed
  the validator in MySQL and PostgreSQL, which let a crafted statement compare a masked
  column and read its values back; and when a table listed `maskedColumns` of its own in the
  catalog, the columns listed for it in `masked-columns.json` were ignored. Ten more were
  closed in 1.29.2, found by an independent review that was asked to break this promise: an
  alias with the masked column's own name (`SELECT email AS email`), a `WITH` name equal to
  the table's name, on PostgreSQL a table alias read as a whole row, on SQL Server a
  double-quoted column name, and six forms of one alias used for two sources at different
  depths of a statement. They share a cause: the validator decided what a name means by
  the parser's rule, and the database decides by scope. Since 1.29.2 the validator resolves
  names by scope itself. Two days later another review found more ways around it on
  PostgreSQL (a double-quoted column inside some functions and operators, the operand of
  `IS DISTINCT FROM`) and on every engine (aliases that differ only by letter case); those
  were closed in 1.30.0. The 1.29.2 fixes were verified on MySQL 8.4 and PostgreSQL 16. The
  1.29.2 release said they were also checked on SQL Server by the test battery; because of
  a defect in the test, the battery ran those cases on the MySQL dialect only. They were run
  on the PostgreSQL and SQL Server dialects later and hold. SQL Server has not been tested
  on a live server. If you ran an earlier version with masked columns, see the advisory and
  "Corrections" in [SECURITY_OVERVIEW.md](SECURITY_OVERVIEW.md). Since 1.26.0 a
  relationship that ends on a masked column and a masked primary key are not named to the
  model either, and an explanation of an earlier answer is refused when its statement is no
  longer allowed; since 1.30.0 the same holds for the previous answer of a follow-up.
  Masking is the control for personal data: list every column that holds it, and take the
  `SELECT` privilege on those columns away from the read-only account
  ([INSTALL.md](INSTALL.md), Step 2), so that the mask does not rest on Rosie alone. Column
  privileges do not reach through a view, which reads with its owner's rights: mask the
  view's column too, or leave the view out of the catalog. `npm run generate-catalog` prints the column names
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
  refusals and more flagged answers. What the validator misses does not depend on the
  model; the database's column privileges cover that.

Self-hosted endpoints have not been part of the automated test suite of this repository;
`test:provider` is the check to run on yours.

## Retention on your side

| Store | What it holds | Where |
|---|---|---|
| `rosie_audit_log` | One row per question: user id, question, generated SQL, outcome, timings. | The engine database (`DB_ENGINE_DATABASE`). |
| `rosie_answer_certificate` | The certificate of each answer that carried one. | Same. |
| `rosie_feedback` | Thumbs up or down, with the question and SQL. | Same. |
| `rosie_rate_limit` | Only with `ROSIE_RATE_LIMIT_STORE=database`: one counter per client address and window. Removed by the engine about two windows later (two days for the daily quota). | Same. |
| Audit file | The same audit records as newline-delimited JSON, as a fallback when the database is down. Since 1.30.0 it also lists the statements the model ran for a question with `execute_query`, refused ones included, up to 20 (`sql_trace`), with their literal values. | `ROSIE_AUDIT_FILE` (default `logs/`). |
| Process log (stdout) | Events of each request. SQL text and database error text only at `LOG_LEVEL=debug` (up to 1.29.2 database error text was logged at error level, and it can quote a stored value). | Wherever you ship stdout. |

Result rows are not stored by Rosie. The certificate of an answer is, and it holds figures and a few values: the totals of the second derivation, and where a filter value was not found, up to eight real values of that (unmasked) column. Retention of these tables is yours to set, and nothing
is removed unless you schedule it:

```bash
npm run retention -- --days 365               # report: how many records are older
npm run retention -- --days 365 --confirm     # remove them from the three log tables
```

The command touches only `rosie_audit_log`, `rosie_answer_certificate` and `rosie_feedback`,
and the account that runs it needs DELETE on them (`docs/INSTALL.md`, Step 3). The audit file is not rotated by Rosie; use
your log tooling for it.

A mask applies from the reload on. Records already written keep the question, statement and
certificate they had; Rosie does not rewrite its trail. To remove them, use
`npm run retention` or your own procedure.

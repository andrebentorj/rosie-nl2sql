<p align="center">
  <img src="docs/assets/rosie.svg" alt="Rosie" width="160" height="160">
</p>

<h1 align="center">Rosie</h1>

<p align="center"><b>Ask your database in plain language. See what was checked, and what was not.</b></p>

Rosie is a self-hosted text-to-SQL engine for MySQL, PostgreSQL and SQL Server (MariaDB uses
the MySQL dialect and is not part of the test matrix). You point it at
a database, it learns the schema, and people (through an embeddable chat widget) or AI agents
(through MCP) ask questions in plain language. Rosie writes the SQL, checks it, runs it
read-only and answers.

What sets it apart is what happens after the SQL runs. The answer goes through a set of
deterministic checks for the ways a query can be wrong without any error, and the outcome is
shown next to the number: the **Answer Certificate**. When a check could not run, Rosie says
that too. The checks look at the structure of the calculation, not at whether the question
was understood; [docs/CERTIFICATE.md](docs/CERTIFICATE.md) says exactly what is covered.

Two real answers from the demo database. On the left, the number was re-derived by a second
route and matched. On the right, the question asks for a period that runs six months past
the last record, and Rosie says so instead of handing over a total that looks complete.

| Checked and confirmed | Flagged |
|---|---|
| ![An answer to "How many tracks were sold in 2024?" with the note "Cross-checked: Re-derived independently over distinct InvoiceLine records and got the same value"](docs/assets/widget-verified.png) | ![An answer with the warning that the requested period is not fully covered by the data](docs/assets/widget-flagged.png) |

The pictures are from an earlier version. The note under a cross-checked answer now ends
with "This checks the calculation, not whether the question was understood."

A query that runs and returns a plausible number is the failure that hurts, because nobody
looks twice. An example written to show it, on the demo data: a join that repeats each
invoice once per line turns a revenue of 2,328.60 into 20,848.62 without any error; given
that statement, the certificate reports both figures and marks the answer `CONTRADICTED`.

## Highlights

- **It learns your database by itself.** One command reads the schema and writes the whole
  context: the catalog, the columns to mask, the relationships the database does not
  declare, descriptions, the assistant's instructions, and example queries it keeps only if
  they run on your data. Then it reads how your data records a missing value, which no
  schema says. [How Rosie learns your database](#how-rosie-learns-your-database).
- **Every answer says what was checked.** After the SQL runs, deterministic checks look for
  the ways a query is wrong without an error, and the outcome is shown next to the number,
  with the evidence. A check that could not run is reported as not run.
  [docs/CERTIFICATE.md](docs/CERTIFICATE.md).
- **Read-only, and it proves it.** SQL is validated on its syntax tree against an allowlist
  before it runs, masked columns are never read, row policies are applied by the engine from
  the caller's verified identity, and Rosie stops answering if its database user can do more
  than SELECT. [docs/SECURITY_OVERVIEW.md](docs/SECURITY_OVERVIEW.md).
- **Short to install, short to embed.** `npm run init` to set up, `npm run doctor` to find
  what is wrong and how to fix it, one script tag for the chat widget, one command to add
  it to Claude or Cursor over MCP. [Install](#install), [Connect](#connect).
- **It gets better where people use it.** A person approves the query behind a recurring
  question and it is run unchanged from then on; a metric is declared once and the engine
  writes its query; a weekly page lists what needs a person; a context change is checked in
  the pull request, with no database and no model.
  [docs/APPROVED_QUERIES.md](docs/APPROVED_QUERIES.md), [docs/METRICS.md](docs/METRICS.md),
  [docs/OPERATIONS.md](docs/OPERATIONS.md).
- **Measured in public, with what went wrong.** Three database engines, a public benchmark
  we did not write, every question's result in the repository, and a rule for when a gain
  may be claimed. [Measured, on public data](#measured-on-public-data).
- **Yours to run.** Your server, your database, your model key, any OpenAI-compatible
  endpoint including a model you host. [docs/DATA_PRIVACY.md](docs/DATA_PRIVACY.md).

## How Rosie learns your database

Rosie ships with no schema and no domain knowledge. Everything it knows about a database
lives in a **context directory** of plain files that you can read, edit and review in a
pull request. Nobody has to write that directory by hand: Rosie builds the first version
from the database, and it grows from use.

**1. `npm run init` builds the context** ([docs/INSTALL.md](docs/INSTALL.md)).

| Step | What it reads | What it writes |
|---|---|---|
| Structure | Tables, views, columns, types, primary and foreign keys and approximate row counts, from the database's own system catalogs. No rows. | `schema-catalog.json`, which is also the allowlist: what is not in it cannot be queried. |
| Relationships | Column names, where the database declares no foreign key. | Proposed joins, for you to confirm. |
| Personal data | Column names only. | `masked-columns.json`: columns that are never read, returned or sent to the model. You confirm the list. |
| Meaning | The catalog, and sample values only if you allow it (`--samples`). | Descriptions, synonyms, what each code means, metrics, the assistant's instructions and the questions it should ask back when a request is ambiguous. |
| Examples | Your database. | `verified-queries.json`: each example has to pass the validator and run on your data, or it is dropped. |
| A first question bank | Your database. | Questions with expected results taken from real execution, to measure accuracy before anyone uses it. |
| Access | | With `--create-ro-user`, a user that can only SELECT. With `--create-engine-tables`, the tables for the audit trail. A `.env` with a generated token. |

It ends with a report that marks every file a model wrote as provisional, for a person to
review. The structure is never written by a model.

**2. `npm run profile-columns -- --write` reads how the data records things.** A model
reads "heroes with a known weight" and writes `weight IS NOT NULL`; the table holds `0`
for "not recorded". Nothing in a schema says so, and the data does. For every unmasked
column this counts the empty values, the zeros and the smallest value after them, blank
strings, placeholders such as `-` or `N/A`, numbers kept as text and zero-padded codes, and
where there is something a query writer has to know it puts a one-line note beside the
column in the prompt. Without `--write` it only shows the notes. A note you edit by hand is
kept. The notes hold a few real values of unmasked columns (a placeholder, a smallest
value), and those reach the model. On the public benchmark this took the small model from
272 to 329 right answers of 500, in one run, on the benchmark it was developed on
([docs/BIRD.md](docs/BIRD.md)).

**3. It keeps learning from use.**

- A wrong or recurring answer becomes an **approved query** in the administration console:
  a person reviews the statement once, and it is run unchanged whenever the question comes
  back ([docs/APPROVED_QUERIES.md](docs/APPROVED_QUERIES.md)).
- A figure the business defines becomes a **declared metric**, written once or imported
  from dbt or Cube; the engine then writes that query itself ([docs/METRICS.md](docs/METRICS.md)).
- `npm run evaluate` measures accuracy on your own questions and `npm run weekly-report`
  lists what needs a person ([docs/EVALUATION.md](docs/EVALUATION.md),
  [docs/OPERATIONS.md](docs/OPERATIONS.md)).
- When the schema changes, the server notices, says what changed and suspends the approved
  queries that touch it ([docs/OPERATIONS.md](docs/OPERATIONS.md#schema-drift)).
- `npm run check-context` validates every file of the context, with no database and no
  model, so a change is reviewed like code.

What leaves your network while the context is built: table and column names, and sample
values only with `--samples` ([docs/DATA_PRIVACY.md](docs/DATA_PRIVACY.md)).

## How it differs

| Approach | What it does about a number that is wrong without an error |
|---|---|
| A model and a prompt | Nothing after the SQL runs. The answer looks the same right or wrong. |
| A semantic layer (dbt Semantic Layer, Cube and similar) | Prevents it by construction for what has been modeled: a metric defined once cannot be inflated by a join. Someone has to model the data first, and a question outside the model is out of reach. |
| Rosie | Starts from the schema, with no modeling. Checks each answer after it runs and shows what it found. For a metric you do declare, it writes the query itself, as a semantic layer does, and it imports definitions from dbt and Cube. |

Rosie is not the only tool that deals with inflated totals: semantic layers prevent them
where the metric is modeled. Its difference is that it checks after the fact, on questions
nobody modeled, and shows the evidence. The limits are stated as plainly as the checks: the
certificate looks at the structure of the calculation, not at whether the question was
understood, and a passed check does not mean the answer is right
([docs/CERTIFICATE.md](docs/CERTIFICATE.md)).

## What it does

| | |
|---|---|
| **Answer Certificate** | After the SQL runs: detects joins that inflate totals, filters on values that do not exist, periods the data only partly covers, filters that remove nothing, and queries that ignore a declared metric definition. For totals over joins it re-derives the number by a second route: `CROSS_CHECKED` when both agree, `CONTRADICTED` when they differ and the context says the column counts once per record, and both figures side by side when the statement alone cannot say which was meant. It does not re-derive every shape (several totals in one statement, a total inside a function, a zero result: [docs/CERTIFICATE.md](docs/CERTIFICATE.md)), and then the verdict is `NOT_CHECKED` with the reason. With the checks on, which is the default, an answer that ran SQL carries a verdict. |
| **Read-only by construction** | SQL is parsed and validated before it runs: SELECT only, tables and columns from an allowlist, no system functions. The database user is checked at startup and Rosie refuses to answer if it can do more than SELECT. |
| **Personal data stays put** | Masked columns are never read, never returned and never sent to the model provider. |
| **Per-tenant isolation** | Row policies filter every query by the caller's verified identity (JWT). The filter is applied by the engine, not by the model, and cannot be talked around. |
| **Approved queries** | A query a person reviewed for a question is run unchanged when the question comes back, with only the values swapped, and the answer says who approved it. The checks cover the calculation; this covers the meaning. |
| **Metrics with one definition** | Declare "revenue" once, or import it from dbt or Cube. For a declared metric the engine writes the query itself, formula and joins, so it cannot be inflated by a join. A query the model writes on its own is compared with the definition when it adds the column up straight from tables; through a subquery that comparison stays silent and the fan-out checks speak. |
| **Two ways in** | A chat widget (one script tag, white-label) and an MCP server for Claude, Cursor and other AI clients. The widget shows each step while the answer is prepared and draws the result as bars or lines, chosen from its shape. |
| **Runs where your data is** | Your server, your database, your model key. Works with any OpenAI-compatible endpoint, including a model you host. |

## Measured, on public data

Our own regression suite: 198 questions over the public Chinook sample database, the same
bank on every engine, `gpt-6-luna`, one pass on version 1.17.0, with every question's result in
[docs/benchmark-runs](docs/benchmark-runs). "Passed" means the answer met the suite's
weighted criteria (structure, result size, and the exact value where the question has one);
it is not exact-match accuracy, and we wrote the questions. Method, limits and how to
reproduce: [docs/BENCHMARK.md](docs/BENCHMARK.md).

| | MySQL 8.4 | PostgreSQL 16 | SQL Server 2022 |
|---|---|---|---|
| Questions passed | 196 of 198 (99.0%) | 195 of 198 (98.5%) | 195 of 198 (98.5%) |
| Must-refuse questions refused (writes, masked data, injection) | 20 of 20 | 20 of 20 | 20 of 20 |
| Trap questions, built from the error classes the certificate covers | 35 | 35 | 35 |
| of which: wrong answer went out with no warning | 0 | 0 | 0 |
| of which: flagged for repeated rows, the check the certificate is known for | 2 | 1 | 2 |
| Right answers flagged anyway (false alarms; trap questions not counted) | 6 of 94 | 5 of 92 | 7 of 92 |
| Answers that failed the suite and carried a clean certificate (misread questions) | 1 | 1 | 1 |
| Cost of the whole run, at list price | US$ 0.14 | US$ 0.15 | US$ 0.15 |

**The guarantee, and its test.** If an answer adds up or averages a column and its certificate
reads `CHECKS_PASSED` or `CROSS_CHECKED`, the figure is the one you get by taking each record
of that table once. A property test generates statements by combination (joins, `WITH`,
derived tables, `DISTINCT`, `GROUP BY`, `HAVING`, `ORDER BY`), works out the truth by a route
of its own and holds the certificate to that: 9,624 statements on each of MySQL and
PostgreSQL and 9,592 on SQL Server, and 2,594 more built from the catalogs of seven public
databases we did not design: no wrong total with a clean certificate and no right total
called wrong ([docs/CERTIFICATE.md](docs/CERTIFICATE.md#what-is-guaranteed-and-how-it-is-tested)).
It is a property tested on a family of statements, not a proof about all SQL; the checks
read the database after the answer and not in the same snapshot; and it is about the
calculation, not about whether the question was understood.

**A certificate for SQL you did not get from Rosie.** With `ROSIE_CERTIFY_SQL=true`,
`POST /certify` and the MCP tool `certify_sql` run a `SELECT` written by another assistant, a
BI tool or a person, and return its result with the same certificate, with no model
involved ([docs/API.md](docs/API.md)). Off by default.

**On a benchmark we did not write.** [BIRD Mini-Dev](docs/BIRD.md): 500 public questions over
11 databases, PostgreSQL, context built from the schema alone. Every figure below is one
pass, and a fifth to a quarter of the questions change outcome between two passes of the
same model.

| Right answers of 500 | `gpt-6-luna` | `gpt-6.1-terra` |
|---|---|---|
| The model alone: one call, same catalog text, no tools, no checks | 215 (43.0%) | 381 (76.2%) |
| Rosie | 272 (54.4%) | 429 (85.8%) |
| Rosie with the profile of the data | 329 (65.8%) | 437 (87.4%) |

- **The engine adds to the model, for both.** Against the model alone, question by
  question: 101 ahead and 44 behind for `gpt-6-luna`, 68 ahead and 20 behind for
  `gpt-6.1-terra`. Single passes on different engine versions, so read it as the size of
  what the tools, the second look and the checks add, not as an exact figure.
- **54.4% is in the same range as the model's published baseline, not a lead**: the dataset
  card lists 51.8% for the same model with a fixed prompt, and the setups are not identical.
- **65.8% is the profile together with one more change** (a second attempt when a total
  repeats rows), not measured apart, on the benchmark the profile was developed on. Question
  by question it is a gain beyond chance (141 questions ahead, 72 behind).
- **87.4% against 85.8% is not a gain that can be claimed**: 38 questions ahead and 30
  behind is what chance gives four times in ten.
- **Accuracy follows the model first.** `gpt-6.1-terra` costs about one US cent per
  question and is what we recommend for a pilot ([docs/MODELS.md](docs/MODELS.md)).
- **The certificate caught few of these errors**: 10 of 214 wrong answers in the first run.
  Almost all the others compute something sound that is not what was asked, which it does
  not catch. An independent reading of a sample of them found about half come from how the
  data records a missing value, which is what the profile addresses.

[docs/BIRD.md](docs/BIRD.md) lists what went wrong, including a hole in the injection filter
that this benchmark found and that is now fixed, and the rule for when a gain is claimed.
Your numbers will differ, and the same commands measure them.

## Get it

Rosie is commercial software. It is delivered as a private Docker image that you run on
your own infrastructure; this repository holds the documentation, not the source.

The way in is a **six-week pilot**: one database, a question bank built from what your
teams actually ask, and a report of how many Rosie answered correctly and what the
certificate caught. Write to [andrebentorj@gmail.com](mailto:andrebentorj@gmail.com?subject=Rosie%20pilot)
or see the [site](https://andrebentorj.github.io/rosie-nl2sql/).

With access to the image, setup is two commands. The first reads your schema and writes
everything Rosie needs (catalog, masked columns, instructions, validated example queries,
configuration); the second starts the server.

```bash
mkdir -p rosie-context rosie-env
docker run --rm -v "$PWD/rosie-context:/context" -v "$PWD/rosie-env:/out" \
  -e OPENAI_API_KEY -e DB_URL='postgres://user:password@host:5432/dbname' \
  <image> node scripts/rosie-init.js --context /context --env-dir /out --name "My Project"

docker run -d -p 3001:3001 --env-file rosie-env/.env -v "$PWD/rosie-context:/context:ro" <image>
```

Step by step: [docs/INSTALL.md](docs/INSTALL.md).

## Connect

**A web page.** One script tag and one element. The bundle is served by the Rosie backend.

```html
<script src="/api/assistant/widget.js"></script>
<rosie-widget api-url="/api/assistant"></rosie-widget>
```

**An AI client, over MCP.**

```bash
npm run mcp:config                                # prints the entry for Claude Desktop, Claude Code, Cursor
npm run mcp:config -- --install claude-desktop    # or writes it for you, with a backup
```

Walkthrough: [docs/MCP_QUICKSTART.md](docs/MCP_QUICKSTART.md). Reference: [docs/MCP_SERVER.md](docs/MCP_SERVER.md).
HTTP API: [docs/API.md](docs/API.md).

## How it works

```
Question
   │
   ├─ refused before the model: write requests, credential requests, prompt injection
   │
   ├─ model writes SQL  ◄── catalog, metric definitions, verified examples
   │
   ├─ validator ── SELECT only · allowlisted tables and columns · masked columns blocked
   ├─ row policy ─ every table reference filtered by the caller's verified identity
   ├─ read-only pool ── statement timeout · SELECT-only user
   │
   └─ Answer Certificate ── fan-out · empty filter · period coverage · no-op filter · metric
                            definition · independent re-derivation
```

All knowledge about a database lives outside the code, in a **context directory**: the
catalog (which is also the allowlist), masked columns, the prompt, metrics, verified
queries and row policies. One engine serves any number of projects, each with its own
directory. The contract is in [contexts/example/README.md](docs/CONTEXT_DIRECTORY.md).

## Documentation

| I want to | Read |
|---|---|
| Install it | [docs/INSTALL.md](docs/INSTALL.md) |
| See how accurate it is, and reproduce the numbers | [docs/BENCHMARK.md](docs/BENCHMARK.md) |
| Choose a model, and see how much accuracy it buys | [docs/MODELS.md](docs/MODELS.md) |
| Know exactly what the Answer Certificate checks | [docs/CERTIFICATE.md](docs/CERTIFICATE.md) |
| See the result on the public BIRD benchmark | [docs/BIRD.md](docs/BIRD.md) |
| Run it on PostgreSQL | [docs/POSTGRES.md](docs/POSTGRES.md) |
| Run it on SQL Server | [docs/SQLSERVER.md](docs/SQLSERVER.md) |
| Connect Claude or another AI client | [docs/MCP_QUICKSTART.md](docs/MCP_QUICKSTART.md), [docs/MCP_SERVER.md](docs/MCP_SERVER.md) |
| Authenticate users with their own tokens | [docs/IDENTITY.md](docs/IDENTITY.md) |
| Isolate tenants or teams | [docs/ROW_POLICIES.md](docs/ROW_POLICIES.md) |
| Declare metrics, or import them from dbt or Cube | [docs/METRICS.md](docs/METRICS.md) |
| Have a person approve the query behind a recurring question | [docs/APPROVED_QUERIES.md](docs/APPROVED_QUERIES.md) |
| Measure accuracy on my own questions, and catch regressions | [docs/EVALUATION.md](docs/EVALUATION.md) |
| Monitor it with Prometheus and set alerts | [docs/MONITORING.md](docs/MONITORING.md) |
| Know how much one process carries, measured on my database | [docs/CAPACITY.md](docs/CAPACITY.md) |
| Get one page a week on what needs a person | [docs/OPERATIONS.md](docs/OPERATIONS.md#the-weekly-report) |
| Reload the context without a restart, and catch schema changes | [docs/OPERATIONS.md](docs/OPERATIONS.md) |
| Answer a security review or a vendor questionnaire | [docs/SECURITY_OVERVIEW.md](docs/SECURITY_OVERVIEW.md) |
| Know what leaves my network, and use my own model | [docs/DATA_PRIVACY.md](docs/DATA_PRIVACY.md) |
| Call the HTTP API | [docs/API.md](docs/API.md) |
| Write or edit a context directory by hand | [contexts/example/README.md](docs/CONTEXT_DIRECTORY.md) |
| Report a vulnerability | [SECURITY.md](SECURITY.md) |

## Security at a glance

- Authentication by static token, or by the users' own JWTs verified against your identity
  provider's keys (asymmetric algorithms only).
- A dedicated read-only database user, audited at startup and every 15 minutes. If it gains
  a privilege beyond SELECT, Rosie stops answering.
- SQL validated on its syntax tree before it runs: SELECT only, allowlisted tables and
  columns, no system catalogs, no dangerous functions. The one set operation accepted is
  `UNION ALL` inside a derived table, with every branch validated on its own. On PostgreSQL
  and SQL Server, functions are allowlisted.
- Masked columns enforced by the validator, in every tool, on every engine.
- Row policies applied to every read, including exports and the certificate's own probes.
- A prompt-injection filter in front of the model, applied to the question and to the
  conversation history.
- An audit trail of every question, the SQL it produced and its certificate.

The security test battery has more than 700 cases and runs on every commit, together with
live suites against real MySQL, PostgreSQL and SQL Server servers.

## Operating it

**Configuration.** [.env.example](docs/env.example) is the complete reference: every variable,
what it does and its default. A test fails the build if the code reads a variable that file
does not document. The ones worth knowing first:

| Variable | Default | Why you would change it |
|---|---|---|
| `ROSIE_CONTEXT_DIR` | (required) | The project's context directory |
| `DB_DIALECT` | `mysql` | `postgres` for PostgreSQL, `mssql` for SQL Server |
| `DB_DATABASE` / `DB_RO_USER` | (required) | The database to query, and the SELECT-only user that queries it |
| `ROSIE_API_TOKEN` | (generated by `npm run init`) | The token callers present |
| `AI_ASSISTANT_ENABLED` | `false` (`npm run init` writes `true`) | Emergency stop: anything but `true` makes the question, export, explanation, feedback and certify endpoints return 404 |
| `AI_CROSS_CHECK` | `true` | `false` turns off the second derivation (`CROSS_CHECKED` / `CONTRADICTED`) |
| `ROSIE_AUDIT_POLICY` | `fail-open` | `fail-closed` refuses to answer while no audit sink can record |

**Things to know before production.**

- With the static token, every caller is the same principal. Per-user identity, and with it
  row policies and per-user answer handles, comes from JWT ([docs/IDENTITY.md](docs/IDENTITY.md)).
- By default rate limits live in the process, so with two or more replicas each one counts
  on its own. `ROSIE_RATE_LIMIT_STORE=database` shares the hourly and daily quotas
  ([docs/CAPACITY.md](docs/CAPACITY.md)); the concurrency counter and the circuit breaker
  stay per process. Answer handles work across replicas when `ROSIE_HANDLE_SECRET` is shared.
- Behind a proxy, set `TRUST_PROXY`, or every user shares one rate-limit bucket.
- Keep the token on the server side: a browser should reach Rosie through your backend or a
  same-origin proxy that adds the header.

## License

Proprietary, under a commercial licence. See [LICENSE](LICENSE). This repository is
documentation: it grants no right to use the software.

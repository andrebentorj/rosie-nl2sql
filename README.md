<p align="center">
  <img src="docs/assets/rosie.svg" alt="Rosie" width="160" height="160">
</p>

<h1 align="center">Rosie</h1>

<p align="center"><b>Ask your database in plain language. See what was checked, and what was not.</b></p>

Rosie is a self-hosted text-to-SQL engine for MySQL, MariaDB, PostgreSQL and SQL Server. You point it at
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
| ![An answer with the note "Cross-checked: re-derived independently and got the same value"](docs/assets/widget-verified.png) | ![An answer with the warning that the requested period is not fully covered by the data](docs/assets/widget-flagged.png) |

A query that runs and returns a plausible number is the failure that hurts, because nobody
looks twice. On the demo data, a join that repeats each invoice once per
line turns a revenue of 2,328.60 into 20,848.62 without any error; the certificate reports
both figures and marks the answer `CONTRADICTED`.

## What it does

| | |
|---|---|
| **Answer Certificate** | After the SQL runs: detects joins that inflate totals, filters on values that do not exist, periods the data only partly covers, filters that remove nothing, and queries that ignore a declared metric definition. For totals over joins it re-derives the number by a second route and reports `CROSS_CHECKED` or `CONTRADICTED`. Every answer carries an explicit verdict, including `NOT_CHECKED` with the reason. |
| **Read-only by construction** | SQL is parsed and validated before it runs: SELECT only, tables and columns from an allowlist, no system functions. The database user is checked at startup and Rosie refuses to answer if it can do more than SELECT. |
| **Personal data stays put** | Masked columns are never read, never returned and never sent to the model provider. |
| **Per-tenant isolation** | Row policies filter every query by the caller's verified identity (JWT). The filter is applied by the engine, not by the model, and cannot be talked around. |
| **Metrics with one definition** | Declare "revenue" once, or import it from dbt or Cube. Answers that compute it differently are flagged. |
| **Two ways in** | A chat widget (one script tag, white-label) and an MCP server for Claude, Cursor and other AI clients. |
| **Runs where your data is** | Your server, your database, your model key. Works with any OpenAI-compatible endpoint, including a model you host. |

## Measured, on public data

Our own regression suite: 194 questions over the public Chinook sample database, the same
bank on every engine, `gpt-6-luna`, one pass. "Passed" means the answer met the suite's
weighted criteria (structure, result size, and the exact value where the question has one);
it is not exact-match accuracy, and we wrote the questions. Method, limits and how to
reproduce: [docs/BENCHMARK.md](docs/BENCHMARK.md).

| | MySQL 8.4 | PostgreSQL 16 | SQL Server 2022 |
|---|---|---|---|
| Questions passed | 191 of 194 (98.5%) | 187 of 194 (96.4%) | 192 of 194 (99.0%) |
| Must-refuse questions refused (writes, masked data, injection) | 20 of 20 | 19 of 20 | 20 of 20 |
| Trap questions, built from the error classes the certificate covers | 34 | 34 | 34 |
| of which: wrong answer went out with no warning | 0 | 0 | 0 |
| Correct answers flagged anyway (false alarms) | 4 of 82 | 4 of 79 | 5 of 79 |
| Cost of the whole run | US$ 0.12 | US$ 0.14 | US$ 0.13 |

**On a benchmark we did not write.** On [BIRD Mini-Dev](docs/BIRD.md) (500 public questions,
11 databases, context built from the schema alone) Rosie scores 54.4% execution accuracy on
PostgreSQL, against the 51.8% baseline the dataset card lists for the same model. There the
certificate flagged 10 of 214 wrong answers: almost all of them were misread questions, which
it does not catch.

The benchmark page also lists what went wrong, including a hole in the injection filter that
this benchmark found and that is now fixed. The numbers come from one schema and one model;
yours will differ, and the same command measures them.

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
| Know exactly what the Answer Certificate checks | [docs/CERTIFICATE.md](docs/CERTIFICATE.md) |
| See the result on the public BIRD benchmark | [docs/BIRD.md](docs/BIRD.md) |
| Run it on PostgreSQL | [docs/POSTGRES.md](docs/POSTGRES.md) |
| Run it on SQL Server | [docs/SQLSERVER.md](docs/SQLSERVER.md) |
| Connect Claude or another AI client | [docs/MCP_QUICKSTART.md](docs/MCP_QUICKSTART.md), [docs/MCP_SERVER.md](docs/MCP_SERVER.md) |
| Authenticate users with their own tokens | [docs/IDENTITY.md](docs/IDENTITY.md) |
| Isolate tenants or teams | [docs/ROW_POLICIES.md](docs/ROW_POLICIES.md) |
| Declare metrics, or import them from dbt or Cube | [docs/METRICS.md](docs/METRICS.md) |
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
  columns, no system catalogs, no set operations, no dangerous functions. On PostgreSQL
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
| `AI_ASSISTANT_ENABLED` | `true` | Emergency stop: `false` makes every endpoint return 404 |
| `AI_CROSS_CHECK` | `true` | `false` turns off the second derivation (`CROSS_CHECKED` / `CONTRADICTED`) |
| `ROSIE_AUDIT_POLICY` | `fail-open` | `fail-closed` refuses to answer while no audit sink can record |

**Things to know before production.**

- With the static token, every caller is the same principal. Per-user identity, and with it
  row policies and per-user answer handles, comes from JWT ([docs/IDENTITY.md](docs/IDENTITY.md)).
- Rate limits, the concurrency counter and the circuit breaker live in the process. With two
  or more replicas each one counts on its own. Answer handles work across replicas when
  `ROSIE_HANDLE_SECRET` is shared.
- Behind a proxy, set `TRUST_PROXY`, or every user shares one rate-limit bucket.
- Keep the token on the server side: a browser should reach Rosie through your backend or a
  same-origin proxy that adds the header.

## License

Proprietary, under a commercial licence. See [LICENSE](LICENSE). This repository is
documentation: it grants no right to use the software.

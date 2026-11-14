# Operations

What changes after the first install: the context is edited, the database schema moves, and
somebody has to know when either one breaks an answer. This page covers the mechanisms for
that: checking a context change, reloading it, the weekly report, the administration
console and schema drift. It ends with how to rotate a secret and a reference of the
settings. Counters and alerts are in [MONITORING.md](MONITORING.md). Upgrading to a new
version is in [INSTALL.md](INSTALL.md), "Upgrading".

## Check a context change before it is merged

The context is where an installation's mistakes are made: a column renamed in the catalog and
not in an approved query, a mask whose name has a typo and so protects nothing, a metric over
a table that was dropped. The server stops at startup for some of these and quietly drops
others (an example that no longer validates is only logged). `npm run check-context` finds
them in the pull request, with no database and no model, and names the file and the entry.

```bash
npm run check-context -- --context ./rosie-context                              # is it valid
npm run check-context -- --context ./rosie-context --base ../main/rosie-context # and what the change does
```

What it checks:

- every file is present and parses, and the context loads the way the server loads it
  (catalog, metrics, row policies);
- **every mask names a column that exists**; a mask that names nothing is a failure, because
  the column it meant to hide is being read;
- every relationship and primary key names a column of the catalog;
- every approved query would run as approved (parameters declared, review date ahead), and
  every example and every reference of the evaluation bank passes the SQL validator against
  this catalog, with the reason when it does not;
- clarification rules compile and have a question and options.

With `--base` it compares against another copy of the directory (the main branch) and
reports tables and columns added or removed, metrics and approved queries changed, and the
size of the prompt. **Anything that makes more data readable than before fails**: a mask
removed from a column that still exists, a row policy removed from a table that still
exists. `--allow-less-protection` turns that into a warning, for the pull request whose
purpose is that decision. A new column with no mask is a warning, with its name.

`--markdown report.md` writes the same report as a table for a pull request comment,
`--json` prints it for a script, `--strict` makes warnings fail. The exit code is 0 when
nothing failed, 1 when something did, 2 when the command could not start. SQL is validated
for the dialect in `DB_DIALECT`.

A pipeline for a repository that holds the context, as a GitHub Actions job:

```yaml
name: rosie-context
on:
  pull_request:
    paths: [ 'rosie-context/**' ]
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { path: head }
      - uses: actions/checkout@v4
        with: { path: base, ref: '${{ github.base_ref }}' }
      - uses: actions/checkout@v4
        with: { repository: your-org/rosie, path: rosie, token: '${{ secrets.ROSIE_REPO_TOKEN }}' }
      - uses: actions/setup-node@v4
        with: { node-version: 22 }
      - run: npm ci --omit=dev
        working-directory: rosie
      - run: >
          node scripts/check-context.js
          --context ../head/rosie-context
          --base ../base/rosie-context
          --markdown ../report.md
        working-directory: rosie
        env: { DB_DIALECT: postgres }
      - if: always()
        run: cat report.md >> "$GITHUB_STEP_SUMMARY"
```

This job needs no secret of the database and no model key. Where the pipeline can reach a
database, run `npm run doctor -- --offline` after it: doctor checks the whole installation,
the catalog against the live schema included, and `--offline` skips its one call to the
model. With a model key, run `npm run evaluate -- --min <n>` (answers against the evaluation
bank, `eval/bank.json`, `docs/EVALUATION.md`). All three exit with 0 when nothing failed and
with a non-zero code otherwise, so each can gate a merge. Only `check-context` uses 2 for
"could not start"; `doctor` and `evaluate` exit with 1 in that case too.

## Reload the context without a restart

The context directory (catalog, masked columns, instructions, metrics, approved queries, row
policies) is read into memory. After you edit a file there, tell the running server to read
it again. There are two ways, and they do the same thing.

**A signal**, which needs no configuration:

```bash
docker kill --signal=HUP <container>          # or: kill -HUP <pid>
```

**An HTTP call**, which also tells you what happened:

```bash
ROSIE_ADMIN_TOKEN=<a long random value>       # openssl rand -hex 32

curl -X POST -H "Authorization: Bearer $ROSIE_ADMIN_TOKEN" \
  http://localhost:3001/api/assistant/admin/reload
```

```json
{
  "ok": true,
  "projectId": "sales",
  "changed": true,
  "version": "5c1e0b7a92d4",
  "previousVersion": "a81f3e6602bb",
  "loadedAt": "2026-11-11T18:42:07.113Z",
  "tables": 42,
  "metrics": 6,
  "examples": 31,
  "approvedQueries": 12
}
```

The endpoint answers 404 until `ROSIE_ADMIN_TOKEN` is set. The token is its own secret:
`ROSIE_API_TOKEN`, which the host application holds, does not open it, because an
application that can ask questions has no business replacing the catalog.

### What a reload guarantees

- **All or nothing.** The new context is built and validated completely before it replaces
  the old one. A catalog that is not valid JSON, a table with no columns, a metric that
  names a missing column or a row policy on an unknown table makes the reload fail. The
  endpoint then answers 422 with the reason, the signal logs `context_reload_failed`, and
  the server keeps answering with the context it had. There is no half-loaded state.
- **A question in flight is not disturbed.** It finishes with the context it started with,
  so one answer is never validated against two catalogs. The next question gets the new one.
- **The project cannot be renamed.** Answer handles and MCP scopes are bound to the project
  id. A reload that would change it is refused; do that with a restart.
- **`.env` is not reread.** Database credentials, the model and limits are process
  configuration. Changing them needs a restart.

### With more than one replica

Each process holds its own copy, so each one needs the signal or the call. To confirm they
agree, compare `version`: it is a digest of the context files, and it is also exposed as the
`rosie_context_info` metric. A replica reporting a different value was not reloaded, or does
not mount the same directory.

Both ways reach the HTTP server (`npm start`) only. The MCP entry points read the context
once, when they start, and have neither the signal handler nor the endpoint:

- The MCP server over stdio is one process per client. Restart the client to pick up an
  edit.
- The MCP server over HTTP (`npm run mcp:http`) must be restarted. Do not send it `HUP`:
  with no handler installed, that signal ends the process.

### The usual sequence

```bash
npm run review                                   # answers a person should look at
npm run review -- --approve <messageId> --id revenue-by-country --by "Dana Reyes, Finance"
docker kill --signal=HUP rosie                   # the approved query is in service
```

## The weekly report

One page for the person who answers for the assistant, built from what the engine already
records:

```bash
npm run weekly-report                            # last 7 days, on the terminal
npm run weekly-report -- --days 30
npm run weekly-report -- --out report.md         # Markdown, to send or to post
npm run weekly-report -- --json                  # the same content as data
npm run weekly-report -- --fail-on-attention     # exit code 1 when something needs a person
```

It opens with **what needs a person this week**, in this order: answers the second derivation
contradicted, objects the catalog names and the database no longer has, evaluation questions
that stopped matching, approved queries past their review date, answers rated down, answers
that went out with a warning, an error share that is rising, approved queries due for review
within 30 days, and an evaluation that was never run or is more than 14 days old. A quiet
week says so.

Then the numbers, each compared with the period before: usage and outcomes, the verdicts of
the Answer Certificate with the reasons answers went unchecked, the ratings, the ten most
recent entries of the review queue, the approved queries by review date, the last evaluation,
and the catalog against the database.

It is read-only and calls no model. It reads the engine tables, the context directory and,
with the read-only user, the column list of the database (`--no-schema` skips that). From a
scheduler, inside the container:

```bash
docker exec rosie node scripts/weekly-report.js --out /app/logs/weekly.md
```

`npm run audit-report` and `npm run review` remain for the detail behind each line.

## The administration console

`https://<your host>/api/assistant/admin/console` is a page for the person who answers for
the assistant. It asks for a credential when it opens, and it is read-only unless
`ROSIE_ADMIN_WRITE=true` is also set.

There are two ways in, and an installation uses one or both:

| Credential | Setting | What a change is recorded under |
|---|---|---|
| The shared admin token | `ROSIE_ADMIN_TOKEN` | The name the person typed. The token says the request is allowed, not who sent it |
| The administrator's own sign-in token | `ROSIE_ADMIN_JWT_CLAIM` and `ROSIE_ADMIN_JWT_VALUE`, with the identity provider of [IDENTITY.md](IDENTITY.md) | The name and user id in the verified token. What is typed is ignored |

With sign-in, the token is verified exactly as a token for questions is (signature, issuer,
audience, expiry) and must also carry the role: the claim you name holding the value you
name, as a string, a space-separated string or a list. A valid token without the role is
refused with 403, so a token issued for asking questions opens nothing here. There is no
default claim or value. Leave `ROSIE_ADMIN_TOKEN` unset to accept sign-in only; removing an
administrator is then done at the identity provider, with no secret to rotate.

The page takes the token in its one field. It does not run a sign-in flow of its own: the
person pastes a token, or your portal opens the console for them. It is kept for the browser
tab and gone with it.

| View | What it shows |
|---|---|
| Status | Engine version, model, database engine, whether the read-only user is SELECT-only, whether the audit trail is recording, and the "needs a person" list and numbers of the weekly report for the last 7 days |
| To review | Answers that went out with a warning or were rated down, with the question, the reasons and the SQL |
| Catalog | Every table and column the assistant may read, with the masked columns and the declared measures marked. This is the allowlist as the engine holds it, after the last reload |
| Metrics and queries | Declared metrics and how each is answered, approved queries with who approved them and the review date, and the suspended ones with the reason |
| Evaluation | The last `npm run evaluate`: held-out accuracy, the taught set, wrong answers by the warning they carried, regressions, and the questions not matched |

### Changes from the console

With `ROSIE_ADMIN_WRITE=true`, three things can be done from the page, and nothing else:

| Action | Where | What it does |
|---|---|---|
| Approve as a query | To review | The answer's question and SQL become an approved query, with the id you choose, your name, today's date and an optional review date. The server takes the SQL from its own record of that answer, validates it against the catalog, and writes it to `verified-queries.json` |
| Looked at, nothing to approve | To review | The answer leaves the queue. Recorded with your name in `review-dismissed.json`, beside the audit file |
| Edit a description | Catalog | Rewrites the `description` of one table or column in `schema-catalog.json` (500 characters at most) |

What it cannot change, on purpose: which tables and columns the assistant may read, the
masked columns, the declared measures and metrics, the row policies and the project's prompt
file. Those decide what leaves the database, and they stay in files reviewed where the
context is kept.

A table description is not neutral text: it is shown to the model with the catalog, so it
changes how questions are answered, which is what it is for. It cannot open a table, a
masked column or another tenant's rows. A description that addresses the assistant ("system
note", "for every question", "silently") is refused. Give the write switch to people you
would let edit the context files.

Before a change is applied its attempt is written to the audit file, and the change is
refused if that write fails. Approving sends the SQL the page showed, and the server
refuses if the statement on record is another one. Requests are authenticated first and
limited per administrator afterwards; refused requests are counted per address and never
lock out a valid credential.

Each change to the context is applied the same way: the new file is written beside the old
one and renamed over it, the context is rebuilt exactly as a reload does, and if it does not
build the old file is put back and the context in service is untouched. A change that is
accepted is in service at once. Every attempt, accepted or refused, goes to the audit file
as a `rosie_admin_change` record and to the log, with the name given, the address and the
time.

Things to know before turning it on:

- **The context directory must be writable by the server.** The installation steps mount it
  read-only (`:ro`), and with that mount the first and third actions are refused with a
  message saying so. Remove `:ro` only if you want changes from the console.
- **The files change where the server runs.** If the context is versioned somewhere else,
  bring the change back there, or the next deploy overwrites it. The file keeps its
  indentation, but a catalog written with objects on one line is rewritten one field per
  line the first time it is saved.
- **More than one replica.** The change is applied by the replica that received it. The
  others pick it up on their next reload (`docker kill --signal=HUP`), and only if they
  share the same context directory.
- **With the shared token, the name is what the person typed.** For a record tied to an
  identity, configure administrator sign-in (above): the record then carries the verified
  name and user id, and says `identityVerified: true`.

Put it behind your VPN or your reverse proxy's access rules, like any administration page.
The page is served with a content security policy that lets it talk to its own origin only,
is never cached and cannot be framed. Text typed by users (questions, comments) is shown as
text.

## Schema drift

The catalog is a file. The database keeps changing after it was written. When a column is
renamed, the catalog still lists the old name, the model keeps writing it, and every
question that needs it fails with a database error that nobody connects to the cause. For an
approved query it is worse: a person signed it for a schema that no longer exists.

Rosie compares the catalog with the database at startup, every hour, and after each context
reload. The comparison runs as the read-only user, so it answers the question that matters:
can Rosie still read what the catalog names? A table that user lost access to counts as
missing.

The comparison is scheduled by the HTTP server (`npm start`). The MCP entry points
(`npm run mcp`, `npm run mcp:http`) do not run it: in those processes nothing is suspended,
and an approved query that reads a missing column fails with a database error when it is
run. Where Rosie is used only through MCP, run `npm run doctor` on a schedule; it makes the
same comparison and names what is missing.

When something is missing:

1. **One warning in the log**, naming the tables and columns. It is logged when the
   situation changes, never per request.
2. **Counters in `/metrics`**: `rosie_schema_drift_objects` by kind, and
   `rosie_approved_queries_suspended`.
3. **The approved queries and examples that read a missing object are suspended.** The
   model is not shown them and cannot run them by id; it writes the query itself, and the
   answer goes out without the "Approved query" mark, because that mark would no longer be
   true. A query on tables that did not change is not affected.

Nothing else is blocked, and the catalog is never edited for you. Tables and columns that
exist in the database and are not in the catalog are not drift: the catalog is the
allowlist, and leaving something out is how it stays out of reach.

### Fixing it

```bash
npm run doctor                # names every missing table and column
npm run generate-catalog      # rewrites the catalog from the database, keeping what you
                              # wrote by hand (the list is in INSTALL.md, Step 4c)
npm run check-context -- --base <the previous copy of the context>
```

`npm run generate-catalog` keeps descriptions, synonyms, enum values, metrics and masked
columns, and since 1.24.0 also `measure`, `profile`, `profileNote`, `promptHidden`, a
primary key written by hand and the relationships the database does not declare. It does
**not** keep your choice of tables: it writes every table and view of the schema, including
the ones you left out of the catalog on purpose, and the catalog is the allowlist. Remove
them again before the reload. `check-context --base` lists every table and column the new
file adds, so nothing comes back unnoticed. To fix one renamed column, editing its name in
`schema-catalog.json` by hand is the smaller change.

Then review the suspended approved queries in `verified-queries.json` (the statement, and
`approvedBy` and `approvedOn` if a person looked at it again), and reload the context. The
suspension lifts by itself at the next check once the statement no longer reads anything
missing.

### Settings

| Variable | Default | Meaning |
|---|---|---|
| `ROSIE_SCHEMA_DRIFT_INTERVAL_MS` | `3600000` | How often the comparison runs. Minimum one minute. `0` turns it off |
| `ROSIE_ADMIN_TOKEN` | unset | Opens `POST /api/assistant/admin/reload`, `GET /api/assistant/admin/overview` and the console page. Unset, those routes answer 404 and the reload signal still works |
| `ROSIE_ADMIN_JWT_CLAIM`, `ROSIE_ADMIN_JWT_VALUE` | unset | The claim that carries the administrator role and the value it must hold. Set both or neither; they need `ROSIE_JWT_ISSUER` |
| `ROSIE_ADMIN_WRITE` | `false` | With the admin token, allows the three changes of the console (approve, dismiss, edit a description). Off, the console is read-only and those routes answer 403 |

### Limits

- The comparison is by name. A column whose type changed, or whose meaning changed while
  the name stayed, is not detected.
- Which statements are affected is decided by the names they mention. A statement is
  suspended when it names a missing table, or a missing column together with that column's
  table. That errs on the side of suspending.
- If the database cannot be reached, or returns no columns at all, the check reports "not
  checked" and suspends nothing. `rosie_schema_drift_checked` reads 0 in that case.

## Rotating a secret

Every secret is an environment variable, and `.env` is read once, when a process starts. A
context reload does not reread it. So every rotation has the same shape: change the value
where the process gets it, restart every process that uses it, change it on the other side.
Only one value is valid at a time; there is no period in which the old and the new one are
both accepted.

**`ROSIE_API_TOKEN`** (the host application's credential):

1. Generate a value: `openssl rand -hex 32`.
2. Set it for every Rosie replica and restart them.
3. Set it where the caller keeps it, normally the `Authorization` header your reverse proxy
   adds (`docs/INSTALL.md`, Step 6d), and reload the proxy.

Between steps 2 and 3 the proxy still sends the old value and every request answers 401.
Do the two together, in a quiet moment. Answer handles are not affected: they are signed
with `ROSIE_HANDLE_SECRET`.

**`ROSIE_HANDLE_SECRET`** (signs answer handles):

1. Generate a value of 16 characters or more. A shorter one is ignored with a warning and
   the process signs with a random key of its own.
2. Set the same value for every replica, and for the MCP processes that should accept the
   same handles, and restart them.

Every handle issued before the restart stops verifying. Handles live 15 minutes, so the
cost is small: an export or an explanation of an older answer gets "That answer is no longer
available" (404) and the user asks again; a follow-up question is answered without the
previous answer as its starting point; a rating is still recorded, without the question and
SQL snapshot. While replicas restart one by one, a handle from a replica on the old value
is refused by one on the new value.

**`ROSIE_METRICS_TOKEN`, `ROSIE_ADMIN_TOKEN`, `ROSIE_MCP_HTTP_TOKEN`:** the same two steps.
Set the new value, restart, then update the scraper, the people who use the console (the
page keeps the token for the browser tab only and asks again), or the MCP client.

**Database passwords and the model key:** change them at the database or the provider, set
the new value, restart. Nothing else holds them.

With JWT (`docs/IDENTITY.md`) there is no shared user credential to rotate: keys rotate at
the identity provider, and Rosie fetches the new key set from `ROSIE_JWT_JWKS_URL` when a
token arrives signed with a key it does not have. With `ROSIE_JWT_JWKS` (the key set inline)
a key rotation is a new value and a restart.

## Settings reference

Every setting is described in `.env.example`. This table is the short form of the ones no
other page explains. All are read at startup; changing one needs a restart.

| Variable | Default | Meaning |
|---|---|---|
| `TRUST_PROXY` | unset | Number of reverse-proxy hops in front of Rosie (usually `1`), or a list of proxy addresses. Unset, the client address Rosie sees is the proxy's and every user shares one rate-limit bucket. `true` trusts any `X-Forwarded-For` and is logged as a warning |
| `NODE_ENV` | unset | `production` makes authentication fail closed: with no `ROSIE_API_TOKEN` and no JWT configured, every request is refused with 503. The Docker image sets it |
| `ROSIE_CORS_ORIGIN` | `http://localhost:4200` | Origins allowed to call the API from a browser, comma-separated. `*` is refused |
| `LOG_LEVEL` | `info` | `error`, `warn`, `info` or `debug`. Full SQL text is logged only at `debug`; filter values can hold personal data, so keep production at `info` |
| `LOG_FORMAT` | `pretty` | `json` writes one JSON object per line, for a log aggregator |
| `ROSIE_EVAL_DIR` | `<context>/eval/runs` | Where `npm run evaluate` writes its runs and where the server reads the last one. Set it when the context is mounted read-only |
| `ROSIE_PERMCHECK_INTERVAL_MS` | `900000` | How often the privileges of the read-only user are checked again. A value under one minute is replaced by the default |
| `ROSIE_MCP_HTTP_ENABLED` | `false` | Must be `true` for `npm run mcp:http` to start ([MCP_SERVER.md](MCP_SERVER.md)) |
| `ROSIE_MCP_HTTP_HOST`, `ROSIE_MCP_HTTP_PORT` | `127.0.0.1`, `3002` | Where the MCP HTTP transport listens |
| `AI_MAX_TOKENS` | `2048` | Tokens the model may write per reply |
| `AI_TEMPERATURE` | unset | Sampling temperature. Unset, the engine chooses by model family; set it only to override |
| `AI_REASONING_EFFORT` | unset | Reasoning effort for models that have one. Unset, the engine chooses by model family. Run `npm run test:provider` after changing it |
| `AI_API` | unset | `chat` or `responses`: forces the wire API for every model. Unset, the engine starts on chat completions and moves a model to the responses API by itself when the provider says its function tools need it |
| `AI_MAX_TOOL_CALLS` | `20` | Turns the model may take inside one question before it must answer |
| `AI_MAX_QUERIES_PER_REQUEST` | `5` | Queries that reach the database inside one question. Further attempts are refused and the model must answer with what it has |
| `AI_MAX_TOTAL_TOKENS` | `0` | Model tokens one question may use before an answer is forced. `0` is no limit |
| `AI_LLM_RESULT_MAX_ROWS` | `30` | Rows of each result shown to the model. The user always gets the full result. `0` sends the whole result |
| `AI_QUERY_TIMEOUT_MS` | `10000` | Statement timeout on the read-only pool |
| `AI_CIRCUIT_BREAKER_THRESHOLD`, `AI_CIRCUIT_BREAKER_COOLDOWN_MS` | `5`, `60000` | After that many provider failures in a row, questions are answered with "temporarily unable" for the cooldown, without calling the provider |
| `AI_SCHEMA_MAX_TABLES` | `40` | Catalogs with more tables than this get a prompt with only the tables relevant to the question. `0` turns that off. The allowlist is never affected |
| `AI_SCHEMA_TOPK` | `15` | The most tables that selection may pick for one question |
| `AI_CATALOG_COLUMNS` | `true` | Column names, type family and enum codes of unmasked columns in the prompt |
| `AI_CATALOG_DESCRIPTIONS` | `false` | With the above, each column's description too, cut at 100 characters |
| `AI_CATALOG_PROFILE` | `true` | With the above, each column's `profileNote` too (`npm run profile-columns`). A catalog without notes is unaffected |
| `AI_ANSWER_CHECKS` | `true` | The Answer Certificate. `false` turns every check off |
| `AI_CROSS_CHECK` | `true` | The second derivation ([CERTIFICATE.md](CERTIFICATE.md)) |
| `AI_CERTIFICATE_REPAIR` | `true` | One second attempt by the model when the checks name a structural defect |
| `AI_CERTIFICATE_REPAIR_UNDECLARED` | `false` | The same second attempt where the context declares nothing about the column |
| `AI_QUESTION_REVIEW` | `false` | A model reads the question against the final statement and returns an opinion (`questionReview`). One more model call per answer |
| `AI_QUESTION_REVIEW_MODEL` | `AI_MODEL` | The model that reviews. A stronger one than `AI_MODEL` is the point |
| `AI_QUESTION_REVIEW_REPAIR` | `true` | With the review on, one second attempt when the review says the statement does not answer the question |
| `AI_UNION_ALL` | `true` | Accepts `UNION ALL` between the branches of a derived table or `WITH` body. Every other set operation is refused whatever this says |

## Other commands

Commands that no other page describes.

| Command | What it does |
|---|---|
| `npm run generate-vault` | Writes an Obsidian vault from the context directory: one note per table, linked by the catalog's relationships, one per declared metric, and diagrams of the request flow. Output goes to `<context>/vault`, or `--out <dir>`; `--context <dir>` picks another context. It overwrites its own folders (`Schema/`, `Metrics/`, `Flows/` and the hub note) and leaves anything else in the vault alone. No database and no model. |
| `npm run test:live` | Runs the four suites that need real servers, one after the other: `test:mcp`, `test:mysql`, `test:postgres`, `test:sqlserver`. |
| `npm run audit:deps` | `npm audit --audit-level=high`: fails on a known high or critical vulnerability in a dependency. The CI runs it on every push. |
| `node scripts/print-prompt-hash.js` | Prints the length and the SHA-256 of the assembled system prompt for the current context. Run it before and after a change that should not alter the prompt, and compare the two lines. |

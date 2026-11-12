# Operations

What changes after the first install: the context is edited, the database schema moves, and
somebody has to know when either one breaks an answer. This page covers the two mechanisms
for that. Counters and alerts are in [MONITORING.md](MONITORING.md).

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

The MCP server over stdio is one process per client. It reads the context when the client
starts it, so restart the client to pick up an edit.

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
npm run generate-catalog      # rewrites the catalog from the database, keeping your
                              # descriptions, synonyms, metrics and masked columns
```

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

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
| `ROSIE_ADMIN_TOKEN` | unset | Opens `POST /api/assistant/admin/reload`. Unset, the route answers 404 and the signal still works |

### Limits

- The comparison is by name. A column whose type changed, or whose meaning changed while
  the name stayed, is not detected.
- Which statements are affected is decided by the names they mention. A statement is
  suspended when it names a missing table, or a missing column together with that column's
  table. That errs on the side of suspending.
- If the database cannot be reached, or returns no columns at all, the check reports "not
  checked" and suspends nothing. `rosie_schema_drift_checked` reads 0 in that case.

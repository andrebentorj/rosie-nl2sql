# Rosie HTTP API reference

All routes are mounted under `/api/assistant`. Bodies are JSON (`Content-Type: application/json`,
50 KB cap, compressed bodies refused). Authentication is a bearer token
(`Authorization: Bearer <ROSIE_API_TOKEN>`) on every route except `GET /project-info` and
`GET /health`. In production the token is injected by the host app's proxy, never by the
browser (see `docs/INSTALL.md`). Rate limits are per client IP and per route group:
`AI_MAX_QUERIES_PER_HOUR` / `AI_MAX_QUERIES_PER_DAY`, plus `AI_CONCURRENCY_LIMIT` in-flight
requests per process. The MCP surface is documented separately in `docs/MCP_SERVER.md`; both
front doors run the same engine.

Middleware order on `/ask`: feature flag (`AI_ASSISTANT_ENABLED`) → read-only privilege gate
(503 while the probe fails) → audit-policy gate → auth → hourly limit → daily limit →
concurrency.

## POST /ask

Ask a question in natural language.

```json
{
  "question": "how many orders are paid?",
  "conversationId": "optional-client-id-up-to-64-chars",
  "user": { "id": "jdoe", "name": "Jane Doe" },
  "history": [
    { "role": "user", "content": "previous question" },
    { "role": "assistant", "content": "previous answer" }
  ]
}
```

- `question` is required, 3 to `AI_MAX_INPUT_LENGTH` (default 2000) characters. Raw SQL,
  write intent, requests for credentials and prompt-injection patterns are refused before any
  model call.
- `user` is **audit and personalization only**, never authorization.
- `history` is optional: at most 10 entries are kept, each truncated and passed through the
  injection filter; the model sees the last 6.

Response `200`:

```json
{
  "conversationId": "…",
  "messageId": "uuid",
  "answer": "There are 412 paid orders.",
  "clarification": null,
  "sql": "SELECT COUNT(*) AS n FROM `orders` WHERE `status` = 'paid' LIMIT 501",
  "certificate": {
    "verdict": "CHECKS_PASSED",
    "assertions": [ { "id": "coverage", "verdict": "pass", "detail": "Filter kept 412 of 630 rows (65.4%)." } ],
    "crossCheck": null,
    "skipped": [],
    "provenance": { "projectId": "sales", "model": "gpt-6-luna", "engineVersion": "1.0.0", "catalogVersion": "…", "generatedAt": "…" }
  },
  "answerHandle": "base64url.payload.signature",
  "columns": [ "n" ],
  "rows": [ { "n": 412 } ],
  "truncated": false,
  "totalCount": 1,
  "durationMs": 5800,
  "tokenUsage": "2310+85 tok"
}
```

- `clarification`, when present, is `{ "question": "...", "options": ["...", "..."] }` and
  `sql`/`rows` are null: render the options as chips and ask again with the refined question.
- `certificate.verdict` is one of `CONTRADICTED`, `FLAGGED`, `CROSS_CHECKED`, `NOT_CHECKED`,
  `CHECKS_PASSED`, `NOT_APPLICABLE`. Every answer that ran SQL has a certificate; it is
  `null` only on a clarification or a refusal. `skipped` lists the checks that were due and
  could not run, each with a `reason` and a sentence in `detail`. The certificate checks the
  structure of the calculation, not whether the question was understood: never show
  `CROSS_CHECKED` or `CHECKS_PASSED` as "verified" or "correct". Full reference:
  `docs/CERTIFICATE.md`.
- `approval` (not shown above) is `null` unless the answer ran an approved query; then it
  carries `id`, `question`, `approvedBy`, `approvedOn`, `reviewBy` and the `params` used
  (`docs/APPROVED_QUERIES.md`).
- `metric` (not shown above) is `null` unless the statement was compiled by the engine from a
  declared metric; then it carries the metric's `name`, `expression`, `table`, the
  `dimensions` and `filters` used and the `joins` added (`docs/METRICS.md`).
- `certificate.crossCheck`, when present, carries the second derivation: `verdict`, a sentence
  in `detail`, and the two figures in `primary` and `alternative`. For a `GROUP BY` answer it
  also has `groups` (`compared`, `differing`, up to three `examples`).
- `certificate.repair`, when present, records that the checks flagged a first query and the
  model was asked once to correct it (`docs/CERTIFICATE.md`).
- `answerHandle` is the signed ticket for the follow-up routes below; it expires after 15
  minutes and is bound to the caller identity and the project.
- A refused question (write intent, sensitive request, injection, too long) still returns
  `200` with the refusal text in `answer` and no `sql`.

Errors: `400` invalid body or question, `401` bad token, `429` rate limited (`Retry-After`),
`503` disabled / security hold / busy / audit policy, `500` engine error (audited as
`status=error`).

### Streaming the answer

Add `"stream": true` to the body and the response is a server-sent event stream
(`Content-Type: text/event-stream`) instead of one JSON document:

```
event: progress
data: {"step":1,"message":"Reading the question and planning the query"}

event: progress
data: {"step":2,"message":"Ran query 1 (12 rows)"}

event: progress
data: {"step":4,"message":"Verifying the answer (Answer Certificate)"}

event: result
data: { ...the same body a plain /ask returns... }
```

- `progress` events arrive while the engine works: each model turn, each query that ran, the
  certificate, a second attempt when the checks ask for one. They carry no data from the
  database.
- Exactly one `result` event ends a stream that succeeded; an `error` event (`{ "error": "…" }`)
  ends one that did not. Lines starting with `:` are keep-alives and can be ignored.
- **The answer text is not streamed word by word, on purpose.** An answer is released only
  after its checks have run, so nothing is shown that the certificate then contradicts. What
  streams is the work, not the conclusion.
- Refusals that happen before the engine starts (invalid body, rate limit, bad token, security
  hold) are still plain JSON with their HTTP status, so check the status before reading events.
- Behind a proxy, turn response buffering off for this route. The response already carries
  `X-Accel-Buffering: no` and `Cache-Control: no-cache, no-transform`.

The widget asks for the stream by itself and falls back to the plain response when the
browser or the backend does not offer one.

## POST /export

Re-run a previous answer's validated SQL past the display cap (up to 50 000 rows).

```json
{ "answerHandle": "…" }
```

Response `200`: `{ "columns": [{ "key": "n", "label": "n" }], "rows": [...], "rowCount": 1, "exportCapped": false }`.
`404` with a generic message for any unverifiable handle (expired, tampered, another owner or
project): the reason is never disclosed. `400` if the SQL no longer passes the validator.
Audited as `status=export` (or `export_rejected` / `export_error`).

## POST /explain

Plain-language explanation of a previous answer's SQL. One model call, memoized per handle.

```json
{ "answerHandle": "…" }
```

Response `200`: `{ "explanation": "…", "cached": false }`. Same `404` semantics as `/export`.
Audited as `status=explain` / `explain_error`.

## POST /feedback

Thumbs up or down on an answer. Always accepted (`202`), even when the handle has expired.

```json
{ "messageId": "uuid", "rating": "up", "comment": "optional, 500 chars", "answerHandle": "…" }
```

Response: `{ "ok": true }`. Stored in `rosie_feedback` with the question and SQL snapshot
taken from the handle when it still verifies.

## GET /project-info (public)

Metadata the widget needs before the first question, plus the security gate state:

```json
{
  "name": "Sales", "fullName": "…", "description": "…", "welcomeMessage": "…",
  "suggestions": [ "…" ], "aiDisclaimer": "…", "governance": { "owner": "…" }, "theme": null,
  "maxInputLength": 2000,
  "securityHold": false, "securityCode": null, "securityMessage": null, "supportContact": null
}
```

`maxInputLength` is the longest question the server accepts (`AI_MAX_INPUT_LENGTH`); the
widget stops the typing there. `securityHold: true` means the read-only user has more than SELECT privileges and every
question is refused until an administrator fixes the grants.

## GET /health (public)

```json
{
  "status": "ok",
  "audit": { "dbHealthy": true, "schemaBlockedTables": null, "pendingRetries": 0,
             "dbWriteFailures": 0, "droppedRecords": 0, "lastDbFailureAt": null, "fileSinkOk": true }
}
```

`status` is `degraded` while either audit sink is failing; page on `dbHealthy=false` or
`fileSinkOk=false` (a governance control that fails silently is not a control). Raw database
error text is never included here; it lives in the server logs.

## GET /metrics (its own token)

Counters in the Prometheus text format: questions by outcome, certificate verdicts, checks
that could not run, answers from approved queries and compiled metrics, latency and tokens.
Off until `ROSIE_METRICS_TOKEN` is set, and then open only to a request that presents that
value as a bearer token; `ROSIE_API_TOKEN` does not open it. See
[MONITORING.md](MONITORING.md) for the list of metrics and suggested alerts.

## POST /admin/reload (its own token)

Rebuilds the context from its directory and puts it in service without a restart. Off until
`ROSIE_ADMIN_TOKEN` is set; `ROSIE_API_TOKEN` does not open it. Answers 200 with a summary
(`changed`, `version`, counts of tables, metrics and approved queries), or 422 with the
reason when the new context does not build, in which case the context in service is
unchanged. See [OPERATIONS.md](OPERATIONS.md).

## GET /admin/overview and GET /admin/console (the admin token)

`GET /admin/overview` returns, as one read-only JSON document, what an administrator needs to
see about the assistant in service: engine version, model and database engine, the state of
the audit trail and of the read-only user, the catalog (tables, columns, which are masked,
which are declared measures), declared metrics, approved and suspended queries, the last
evaluation with its two sets, and the numbers of the period with the review queue
(`?days=7`, 1 to 90). It carries no secret, no connection setting and no prompt text, and a
masked column appears by name only.

`GET /admin/console` is a page that shows that document. The page holds no data: it asks
for the admin token, keeps it for the browser tab, and calls `/admin/overview`. Both routes
answer 404 until an administrator credential is configured, and `ROSIE_API_TOKEN` opens
neither. See [OPERATIONS.md](OPERATIONS.md).

Every `/admin/*` route accepts, as bearer, the shared `ROSIE_ADMIN_TOKEN` or a JWT carrying
the administrator role (`ROSIE_ADMIN_JWT_CLAIM`, `ROSIE_ADMIN_JWT_VALUE`). A valid JWT without
the role answers 403; an invalid one, 401.

## POST /admin/approve, /admin/describe, /admin/dismiss (the admin token and the write switch)

The three changes the console can make. They need the admin token **and**
`ROSIE_ADMIN_WRITE=true`; without the switch they answer 403.

| Route | Body | Effect |
|---|---|---|
| `/admin/approve` | `{ "messageId", "id", "by", "reviewBy"?, "question"? }` | The recorded SQL of that answer becomes the approved query `id`. The body carries no SQL: the server reads it from its own record and validates it |
| `/admin/describe` | `{ "table", "column"?, "description", "by" }` | Rewrites one description in the catalog. An empty description removes it |
| `/admin/dismiss` | `{ "messageId", "by", "note"? }` | The answer leaves the review queue |

Answers: 200 `{ "ok": true, ... }`; 403 when writing is off; 404 for an unknown answer, table
or column; 409 when the id exists or the context directory cannot be written; 422 when the
input is refused or the context would not build with the change, in which case nothing
changed. See [OPERATIONS.md](OPERATIONS.md).

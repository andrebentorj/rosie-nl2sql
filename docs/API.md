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

- `question` is required, 3 to `AI_MAX_INPUT_LENGTH` (default 500) characters. Raw SQL,
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
- `certificate.repair`, when present, records that the checks flagged a first query and the
  model was asked once to correct it (`docs/CERTIFICATE.md`).
- `answerHandle` is the signed ticket for the follow-up routes below; it expires after 15
  minutes and is bound to the caller identity and the project.
- A refused question (write intent, sensitive request, injection, too long) still returns
  `200` with the refusal text in `answer` and no `sql`.

Errors: `400` invalid body or question, `401` bad token, `429` rate limited (`Retry-After`),
`503` disabled / security hold / busy / audit policy, `500` engine error (audited as
`status=error`).

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
  "securityHold": false, "securityCode": null, "securityMessage": null, "supportContact": null
}
```

`securityHold: true` means the read-only user has more than SELECT privileges and every
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

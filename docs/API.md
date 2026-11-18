# Rosie HTTP API reference

All routes are mounted under `/api/assistant`. Bodies are JSON (`Content-Type: application/json`,
50 KB cap, compressed bodies refused). Authentication is a bearer token
(`Authorization: Bearer <ROSIE_API_TOKEN>`, or a user's JWT when `docs/IDENTITY.md` is
configured) on the routes that answer or record something: `/ask`, `/export`, `/explain`,
`/feedback` and `/certify`. `GET /project-info`, `GET /health` and `GET /widget.js` are
public. `/metrics` and `/admin/*` take their own credentials, described with each route. In
production the shared token is injected by the host app's proxy, never by the browser (see
`docs/INSTALL.md`). Since 1.35.0, with Rosie's own chat page on (`ROSIE_CHAT_PAGE=true`,
[CHAT_PAGE.md](CHAT_PAGE.md)), a request with no `Authorization` header is authenticated by the
page's session cookie instead: the person who signed in is the verified identity (as with a
JWT), a missing, expired or signed-out session answers `401`
`{ "error": "Sign in again to continue.", "code": "E_SIGNIN_REQUIRED" }`, and a request with
the cookie that changes anything must come from the page's exact origin and carry
`X-Rosie-CSRF` with the session's token, or it answers `403`
`{ "error": "This request did not come from the chat page.", "code": "E_CSRF" }` (counted and
audited as a refusal with reason `csrf_refused`). Requests with an `Authorization` header are
unchanged. With the page on, the API never answers without a session, a valid token or a
valid JWT: the "no token configured, development, open" rule does not apply. Rate limits are
per verified person (the chat page's session or a verified JWT, since 1.35.0) or else per
client IP, and per route group (`/ask`, `/export`,
`/explain`, `/feedback` and `/certify` each count on their own):
`AI_MAX_QUERIES_PER_HOUR` / `AI_MAX_QUERIES_PER_DAY`, plus `AI_CONCURRENCY_LIMIT` in-flight
requests per process. The MCP surface is documented separately in `docs/MCP_SERVER.md`; both
front doors run the same engine.

Errors common to every route: `413` for a body over 50 KB and `400` for a body that is not
valid JSON, both as `{ "error": "..." }`. A request whose `Origin` header is not in
`ROSIE_CORS_ORIGIN` is refused with `403` and
`{ "error": "...", "code": "E_CORS_ORIGIN", "origin": "<the origin>", "setting": "ROSIE_CORS_ORIGIN" }`
(`500` until 1.33.0).

Middleware order on `/ask`: feature flag (`AI_ASSISTANT_ENABLED`) → read-only privilege gate
(503 while the probe fails) → masked-column grant gate (503 while
the gate is on, by default in strict mode (`ROSIE_REQUIRE_COLUMN_GRANTS` sets it explicitly in
either mode), and the context is neither "masked by the database" nor
"no masked columns", decided when the request arrives; see
`/health` below) → audit-policy gate → auth → hourly limit → daily limit → concurrency.
`/export`, `/explain` and `/certify` carry the same two gates.

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
  ],
  "previousAnswerHandle": "optional: the answerHandle of the answer this question refines",
  "widgetVersion": "optional: the version of the calling widget",
  "stream": false
}
```

- `question` is required, 3 to `AI_MAX_INPUT_LENGTH` (default 2000) characters. A question
  typed as a SQL statement, write intent, requests for credentials or for a masked column,
  and prompt-injection patterns are refused before any model call. Since 1.33.0 a question
  counts as SQL only when it reads as a statement: a `SELECT ... FROM` with something only
  SQL writes, `SELECT` and `FROM` in capitals, a `select <names> from <a catalog table>` in
  any case, a `select` with `@@`, a function call, a number, a semicolon, a comment,
  `UNION SELECT` or `INFORMATION_SCHEMA`, or the head of a write or DDL statement. "Select
  the top 5 customers", "Select customers from Brazil", "Create a list of ..." and "Update
  me on sales" are questions. A question that puts a catalog table name right after "from"
  ("Select tracks from Album Thriller" when `Album` is a table), or carries a function call
  or a semicolon after "select" ("Select sum(total) by country"), is still taken for SQL
  and refused, as it was in 1.32.1; rephrase it ("Show the tracks of the album Thriller").
  This is a courtesy refusal before the model, not a security control: the validator
  checks whatever statement runs. Write and credential requests are also recognized in
  Portuguese; a change verb about the answer itself ("mude o gráfico para barras", "crie um
  novo gráfico", "remova os dados de 2020 da comparação") is not a write request.
- `user` is **audit and personalization only**, never authorization. It is ignored when the
  request carries a verified JWT. With the static token it names the person every record of
  this answer is kept under: the answer's handle carries it (signed), so the export,
  explanation and rating of the answer are recorded under the same person (since 1.33.0;
  before, those three were recorded as `authenticated`).
- `history` is optional: at most 10 entries are kept, each truncated and passed through the
  injection filter; the model sees the last 6.
- `widgetVersion` is optional (since 1.33.0): the widget sends its version here as well as in
  the `X-Rosie-Widget-Version` header, which it sends only to its own origin. Either one (the
  header wins), when it reads as a version (letters, digits, `.`, `_`, `-`, 32 characters at
  most), is added to the audit row's user agent as ` rosie-widget/<version>`, bound to the
  request's log lines as `widgetVersion`, and logged once per version per process
  (`widget_version_seen`, with `via` `header` or `body`). Only a release-shaped version
  (`2026.11.17`, `2026.11.17.2`) from a request that passes authentication is logged, and at
  most 20 per process; then one `widget_version_seen_full` warning. Anything else is ignored.
- `previousAnswerHandle` is optional. When it verifies (same caller, same project, not
  expired), the engine starts from that answer's SQL and question, for follow-ups such as
  "now only for Q3", when that SQL still passes the validator against the current catalog
  (since 1.30.0); otherwise the handle is ignored. A handle that does not verify is ignored,
  never an error.
- `stream` is optional, see "Streaming the answer" below.

Response `200`:

```json
{
  "conversationId": "…",
  "messageId": "uuid",
  "answer": "There are 412 paid orders.",
  "blocked": false,
  "reason": null,
  "clarification": null,
  "sql": "SELECT COUNT(*) AS n FROM `orders` WHERE `status` = 'paid' LIMIT 501",
  "certificate": {
    "verdict": "CHECKS_PASSED",
    "assertions": [ { "id": "coverage", "verdict": "pass", "detail": "Filter kept 412 of 630 rows (65.4%)." } ],
    "crossCheck": null,
    "skipped": [],
    "provenance": { "projectId": "sales", "model": "gpt-6-luna", "engineVersion": "1.0.0", "catalogVersion": "…", "generatedAt": "…" }
  },
  "answerHandle": "<base64url payload>.<base64url signature>",
  "columns": [ "n" ],
  "rows": [ { "n": 412 } ],
  "truncated": false,
  "totalCount": 1,
  "durationMs": 5800,
  "tokenUsage": "2310+85 tok"
}
```

- `clarification`, when present, is `{ "question": "...", "options": ["...", "..."] }`. Then
  `sql`, `certificate` and `answerHandle` are `null`, `rows` and `columns` are empty and
  `answer` repeats the question: render the options as chips and ask again with the refined
  question.
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
- `narrativeReplaced` (not shown above, returned since 1.27.0) is `null` unless the answer is
  a single number and the sentence the model wrote states numbers none of which is that
  number (as shown, rounded as the sentence rounds it, as a percentage or in thousands or
  millions). Then it is `{ "reason": "figure_mismatch" }` and `answer` says that the result
  is in the table and the sentence is not shown. The certificate is about the statement and
  its rows; nothing else ties the sentence to them.
- `unfinished` (not shown above, returned since 1.24.0) is `null` unless the request ended
  with no query run as the answer. Then it is `{ "reason" }`, `sql` is `null` and `rows` and
  `columns` are empty. The reason is the limit that stopped the model (`query_budget`,
  `tool_calls`, `token_budget` or `loop_limit`), and `answer` says that there is no answer;
  or `query_failed` when the statement meant as the answer was refused by the validator or
  failed in the database, and `answer` keeps what the model said about it. A query the model
  ran only to look at the data is never returned as the answer, and neither is a statement
  that did not run (before 1.24.0 its text was returned in `sql` with no rows).
- `questionReview` (not shown above, returned since 1.24.0) is `null` unless
  `AI_QUESTION_REVIEW=true` and the review ran; then it carries `verdict` (`answers`,
  `answers_on_this_data`, `does_not_answer`, `cannot_tell`), `reading` (one sentence),
  `model`, `outcome` (`corrected`, `not_improved`, `unchanged`, `failed` or `null`) and
  `firstSql` when the answer was replaced. It is an opinion of a model and never changes
  `certificate` (`docs/CERTIFICATE.md`, "The question review").
- `metric` (not shown above) is `null` unless the statement was compiled by the engine from a
  declared metric; then it carries the metric's `name`, `expression`, `table`, the
  `dimensions` and `filters` used and the `joins` added (`docs/METRICS.md`).
- `certificate.crossCheck`, when present, carries the second derivation: `verdict`, a sentence
  in `detail`, and the two figures in `primary` and `alternative`. For a `GROUP BY` answer it
  also has `groups` (`compared`, `differing`, up to three `examples`).
- `certificate.evidence` (since 1.29.0) says the same findings one dimension at a time
  (`repeatedRows`, `secondDerivation`, `filterValues`, `period`, `filterCoverage`,
  `zerosInAverage`, `metricDefinition`, `approval`, `businessGrain`,
  `questionInterpretation`, `dataConventions`, and `statement` for who wrote the SQL). Every
  dimension is always present with a `status`, and `notEstablished` lists in plain sentences
  what no check established. Do not automate on `verdict` alone: `questionInterpretation` is
  `not_verified` for every statement a model wrote, whatever the verdict
  ([EVIDENCE.md](EVIDENCE.md)). The certificate returned by `POST /certify` carries it too,
  with `statement.status` `caller`.
- `certificate.repair`, when present, records that the checks flagged a first query and the
  model was asked once to correct it (`docs/CERTIFICATE.md`).
- `answerHandle` is the signed ticket for the follow-up routes below; it expires after 15
  minutes and is bound to the caller identity and the project. The caller identity is the
  verified user under JWT. With the shared `ROSIE_API_TOKEN` every caller is the same
  identity, so the handle is then bound to the installation, not to a person.
- `blocked` and `reason` (since 1.33.0) tell a refusal from an answer, with the same values
  as the MCP tool. A question refused by the engine's own guards still returns `200`, with
  `blocked: true`, the sentence to show in `answer`, no `sql`, no `answerHandle` and no
  certificate. `reason` is `write_intent`, `sensitive_request` (credentials or a masked
  column), `raw_sql`, `prompt_injection`, `question_too_short`, `question_too_long` or
  `provider_unavailable` (the model provider is failing; try again in a minute). Do not show
  thumbs or a "nothing was checked" line under a refusal. For an answer, `blocked` is `false`
  and `reason` is `null`.
- A question refused at the door returns `400`: missing or empty, longer than
  `AI_MAX_INPUT_LENGTH` (`code: "E_INPUT_TOO_LONG"`, `blocked: true`,
  `reason: "question_too_long"`), or caught by the injection filter (`{ "error": "Your
  question was blocked by the safety filter.", "code": "E_SAFETY_FILTER", "blocked": true,
  "reason": "prompt_injection" }`). Since 1.33.0 both leave an audit row (`status=blocked`,
  `rejection_reason` `input_too_long` or `prompt_injection_detected`).

Errors: `400` invalid body or question, `401` bad token (`E_SIGNIN_REQUIRED` for the chat
page's session), `403` `E_CSRF` (chat page session from another origin or without its token),
`404` assistant disabled
(`AI_ASSISTANT_ENABLED=false`, or no model key is set), `429` rate limited
(`Retry-After`, `code: "E_RATE_LIMITED"`), `503` security hold, busy (`code: "E_BUSY"`,
`Retry-After: 5`, `retryAfter: 5`) or audit policy, or no credential configured with
`NODE_ENV=production`, `500` engine error (audited as `status=error`).

### Error bodies and the reference

Since 1.33.0 every error response of the API (status 400 and above with an `error`) carries
`reference`: 8 hexadecimal characters the person can quote. That includes the errors answered
before the routes run: a malformed or oversized body (`400` `Malformed request body`, `413`)
and a refused browser origin (`403` `E_CORS_ORIGIN`). It is the start of the request's
correlation id, which the audit row (`correlation_id`) and every log line of the request
carry. To find the request, filter `rosie_audit_log` on `correlation_id` starting with the
reference, or search the log for `"reference":"<reference>"` (the `error_response` line; a
body or origin error has no audit row). A streamed `error` event carries it too.

```json
{ "error": "Something went wrong processing your question. Please try again.", "reference": "3f2a9c10" }
```

### Requests turned away before they are read

Since 1.33.0 a `401`, a `429`, a busy `503`, a security or column-grant hold and an
audit-policy refusal are each counted (`rosie_refused_total{route,reason}`,
[MONITORING.md](MONITORING.md)) and leave an audit row with `status=refused`,
`rejection_reason` `unauthenticated`, `rate_limited`, `busy`, `security_hold`,
`column_grants_hold`, `audit_unavailable` or (1.35.0) `csrf_refused`, the person (the host's `user.id` under the
static token, `unauthenticated` for a bad token), the address and the correlation id, and no
question text. Per route, reason, address (grouped as the rate limiter groups it) and person
(when the request names one), the first refusal of a minute is written as its own row; when
that minute ends, one more row says in `error_message` how many more there were ("3 more
refusals like this one (HTTP 429 on ask) between ... and ... were not recorded one by one"),
whether or not another refusal arrives. A client that keeps one identity and ignores
`Retry-After` therefore cannot flood the trail. The person comes from the request, so a
holder of the API token who sends a different `user.id` with every request gets one row per
request for its refusals (as many as it sends); with a verified JWT the person cannot be
changed that way. The counter counts every one. An audit-policy refusal is counted but
its row is not written: nothing can record it, which is why it was refused.

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

Response `200`: `{ "columns": [{ "key": "n", "label": "n" }], "rows": [...], "rowCount": 1, "exportCapped": false, "verdict": "CONTRADICTED" }`. `verdict` (since 1.27.0) is the certificate verdict of the answer the handle was minted for, `null` when it had no certificate. A client that writes these rows to a file must carry it: the widget starts the CSV with a `Rosie verdict` row, the certificate's sentences and one empty row before the header when the verdict is `CONTRADICTED`, `FLAGGED` or `NOT_CHECKED`, and leaves the file starting with the header otherwise.
`404` with a generic message for any unverifiable handle (expired, tampered, another owner or
project): the reason is never disclosed. `400` if the SQL no longer passes the validator.
`403` when a row policy refuses the caller (`docs/ROW_POLICIES.md`). `500` when the query
fails. Audited as `status=export` (or `export_rejected` / `export_denied` /
`export_error`); since 1.33.0 under the person who asked (the handle carries them) and with
the statement that ran (the export cap in place of the display cap) in `sql_generated`.

## POST /explain

Plain-language explanation of a previous answer's SQL. One model call, memoized per handle.

```json
{ "answerHandle": "…" }
```

Response `200`: `{ "explanation": "…", "cached": false }`. Same `404` semantics as `/export`.
`410` when the answer's statement is no longer allowed (a column masked or a table removed
since).
Audited as `status=explain` / `explain_error`.

## POST /feedback

Thumbs up or down on an answer. A well-formed vote is always accepted (`202`), even when
the handle has expired or is missing. `400` when `messageId` is missing or `rating` is not
`"up"` or `"down"`.

```json
{ "messageId": "uuid", "rating": "up", "comment": "optional, 500 chars", "answerHandle": "…" }
```

Response: `{ "ok": true }`. Stored in `rosie_feedback` with the question and SQL snapshot
taken from the handle when it still verifies and is the handle of that `messageId`. Since
1.33.0 the rating is recorded under the person who asked, as the handle names them; with no
valid handle, under the `user.id` of the body, else the account.

## POST /certify (off by default)

Runs a `SELECT` that the caller wrote and returns its result with an Answer Certificate. No
model is involved: this is the certificate as a service for SQL that came from somewhere
else, another assistant, a BI tool, a person.

```bash
curl -s https://<host>/api/assistant/certify -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{ "sql": "SELECT SUM(i.Total) FROM Invoice i JOIN InvoiceLine il ON il.InvoiceId = i.InvoiceId",
        "question": "What is the total revenue?" }'
```

| Field | Meaning |
|---|---|
| `sql` | One `SELECT`, in the dialect of the database, 8,000 characters at most |
| `question` | Optional. What the statement is meant to answer. It is what lets the certificate compare the statement with a declared metric the question names |

Response: `{ messageId, sql, certificate, columns, rows, rowCount, totalCount, truncated }`,
where `sql` is the statement as it ran (with the row cap) and `certificate` is the same
object `/ask` returns. For the example above, on the demo data, the result is 20848.62 and
the certificate reads `CONTRADICTED`, with 2328.60 as the figure over distinct invoices.

The statement goes through what every statement of the assistant goes through: read-only,
only the tables and columns of the catalog, refusal of any statement in which it finds a
masked column, the row policies of the
caller, the row cap, the read-only connection. A statement that fails any of that answers
422 with `{ "error": "<the reason>", "suggestion": "<a hint, or null>" }` and nothing runs.
`400` when `sql` is missing. Same credential, gates and quotas as `/ask`, counted on their
own, and every request is audited like an answer.

It answers 404 until `ROSIE_CERTIFY_SQL=true`. That is deliberate: everywhere else a caller
asks in words and never hands the engine SQL. Turning it on does not widen what the
validator allows. It changes who writes the statement, so a caller holding the API
credential can read anything that credential's row policies allow, in any combination,
instead of what a question happens to produce, and can try statements against the validator
directly, with no model in between. Every validator gap fixed in 1.29.2 and 1.30.0 was
reachable this way ([SECURITY_OVERVIEW.md](SECURITY_OVERVIEW.md), "Corrections"). Turn it
on where that is acceptable, and only with column grants in place.

## GET /project-info (public)

Metadata the widget needs before the first question, plus the security gate state:

```json
{
  "name": "Sales", "fullName": "…", "description": "…", "welcomeMessage": "…",
  "suggestions": [ "…" ], "aiDisclaimer": "…", "governance": { "owner": "…" }, "theme": null,
  "brand": null, "widgetVersion": "2026.11.18.2",
  "maxInputLength": 2000,
  "securityHold": false, "securityCode": null, "securityMessage": null, "supportContact": null
}
```

`maxInputLength` is the longest question the server accepts (`AI_MAX_INPUT_LENGTH`); the
widget stops the typing there. `brand` is the validated white-label block of `project.json`
(`assistantName`, `tagline`, `hideMascot`) or `null`. `widgetVersion` is the version of the
bundle this backend serves at `/widget.js`; a copied widget compares itself with it.
Since 1.33.0 `securityMessage` and `supportContact` are written for the person asking:
"Rosie is paused while a security setting is fixed. Your question was not run." and "Please
tell <who>, and quote the code.", where `<who>` comes from `project.json` `"supportContact"`
(a string, or `{ "name": "Maria in IT", "contact": "it@example.com" }`; plain text, 160
characters at most), or a generic sentence when it is not set. Which account, privilege or
setting is behind the hold is in the server log, `/health` and the console, never here.
`supportContact` is `null` outside a hold. A held request's `503` carries the same
`supportContact`.
`securityHold: true` means every question is refused until an administrator fixes the
grants: either the read-only user has more than SELECT privileges, or (since 1.31.0, with
the column-grant gate on: by default in strict mode, or with `ROSIE_REQUIRE_COLUMN_GRANTS=true`
in either mode) the masked-column check of the default context did not
pass, in which case `securityCode` is one of the `E_COLUMN_GRANTS_` codes below and
`securityMessage` the hold message. The permission hold wins when both apply.

## GET /widget.js (public)

The widget bundle this backend was deployed with, as `application/javascript`. It carries
`ETag`, `Cache-Control: public, max-age=300, must-revalidate` and `X-Rosie-Widget-Version`,
and answers `304` to a matching `If-None-Match`. `404` when the deployment does not ship the
bundle.

What the widget sends and reads (since 1.33.0, widget 2026.11.17):

- Requests to its own origin carry `X-Rosie-Widget-Version: <version>`; across origins it
  leaves the header out (a server before 1.33.0 refuses it in the preflight). `/ask` also
  carries `widgetVersion` in its body, so the version is recorded from any origin. Rosie's
  CORS layer allows the header since 1.33.0; a reverse proxy that answers preflights itself
  must list it in `Access-Control-Allow-Headers`.
- Element attribute `layout` (1.35.0, widget 2026.11.18.2): `page` for a full-page chat with
  no launcher, opened by itself, filling the element, which the page sizes
  ([WIDGET.md](WIDGET.md)). On a same-origin page with `<meta name="rosie-csrf-token">`
  every request carries `X-Rosie-CSRF` with credentials `same-origin` and no
  `Authorization` (an `api-token` attribute is ignored there); Rosie's chat page is that
  page. A `403` with `code: "E_CSRF"` is shown with its own message and the code.
- Element attributes: `locale` (BCP 47; number separators and the CSV rule, default the
  browser's language, else `en`) and `timeout-seconds` (how long it waits for `/ask`,
  default `130`). On a `401` the element dispatches `rosie:auth-expired` (bubbles, composed,
  `detail.path`).
- It reads `reference` from error bodies (also `ref` or `correlationId`), `code: "E_BUSY"`
  or a numeric `retryAfter` for a busy `503` (asked again twice), `blocked` on `/ask`, and
  `securityMessage`, `supportContact` and `securityCode` from `/project-info` (or `error`,
  `supportContact` and `code` of a `503` body) for a hold.
- CSV exports start with a UTF-8 byte order mark; in a locale whose decimal mark is a comma
  they use `;` and decimal commas ([WIDGET.md](WIDGET.md#the-csv-file-carries-the-verdict)).

## GET /health (public)

`status` says whether Rosie answers (since 1.33.0):

| `status` | HTTP | Meaning |
|---|---|---|
| `ok` | 200 | Answering, and both audit sinks are recording |
| `degraded` | 200 | Answering, but an audit sink is failing (`audit` says which; the default `fail-open` policy keeps answering while one records), or the model provider refused the key recently (`refusing` has `model_key_refused`) |
| `unavailable` | 503 | Every question is refused. `refusing` lists why, as `{ "code", "reason" }` |

The codes in `refusing`: `assistant_disabled` (`AI_ASSISTANT_ENABLED=false`),
`model_key_missing` (`OPENAI_API_KEY` unset or not usable), `model_key_refused` (the provider
answered 401 or 403 to model calls: reported after a second refusal, or once the first is 30
seconds old with no success since, so one stray 403 changes nothing; it makes `status`
`degraded`, not `unavailable`, and clears at the next call that succeeds or 10 minutes after
the last refusal), the security-hold code of the
read-only account's privilege check (for example `E_DB_RO_NOT_SEGREGATED`), the column-grant
hold code (`E_COLUMN_GRANTS_*`), and the audit gate's code under `ROSIE_AUDIT_POLICY=fail-closed`.
`refusing` is `[]` otherwise. Until 1.33.0 `status` was about the audit trail only, so it
said `ok` (and the Docker `HEALTHCHECK` said healthy) while every question got 503. A load
balancer or the `HEALTHCHECK` reads the HTTP code. A refused key is not a 503: /health makes
no model call, so it only knows what earlier calls saw, and a 503 there would keep a
balancer from sending the question that shows the key works again. This is a readiness
check: the cause of a 503 is the configuration (a setting, the database grants), and
restarting with the same configuration does not change it. Fix the cause: a grant change is
picked up by the next check (every 15 minutes) or by a restart, a setting by a restart. A
liveness probe that restarts the process on failure should use `GET /health/live`, which
answers `200 { "status": "alive" }` while the process serves HTTP.

```json
{
  "status": "ok",
  "refusing": [],
  "audit": { "dbHealthy": true, "schemaBlockedTables": null, "pendingRetries": 0,
             "dbWriteFailures": 0, "droppedRecords": 0, "rejectedRows": 0,
             "columnsMissing": null, "lastDbFailureAt": null,
             "fileSinkOk": true, "fileDurability": "write", "fileWriteFailures": 0,
             "fileReopened": 0, "lastFileError": null },
  "columnGrantsRequired": true,
  "maskingMode": "strict",
  "columnGrants": [
    { "projectId": "my-project", "maskingMode": "strict",
      "mode": "database", "label": "masked by the database",
      "total": 4, "denied": 4, "readable": 0, "unverified": 0,
      "reasons": [], "limits": [ "mysql_join_using_natural", "mysql_column_statistics" ],
      "checkedAt": "2026-11-17T09:00:00.000Z", "hold": false,
      "checkAgeMs": 412000, "stale": false, "holdReason": null }
  ]
}
```

`columnGrants` has one entry per context: whether the read-only account can read the
context's masked columns, as checked at startup, every 15 minutes and after a context
reload. `maskingMode` is `express` or `strict` (`ROSIE_MASKING_MODE`, since 1.32.0); it is
also at the top level, next to `columnGrantsRequired`, so it shows before any context has
been checked. Since 1.32.0 each entry also carries `checkAgeMs` (milliseconds since the start
of the check behind the result, `null` before any check), `stale` (`true` once that is past
`ROSIE_COLUMN_GRANTS_MAX_AGE_MS`, in both modes) and `holdReason` (with the gate on, why a
passing result holds: `expired`, `account_changed`, `pool_not_rotated` or `no_lease`; `null`
otherwise); such a hold answers with `E_COLUMN_GRANTS_NOT_CHECKED`. `readable` and `unverified` are counts. `mode` is `database`
("masked by the database": every masked column denied to the account and to every role it can
switch to, every unmasked catalog column readable, every catalog view reviewed), `rosie`
("masked by Rosie only": anything else), `mssql` ("not verified on SQL Server": nothing
readable was found, but something could not be established there), `none` (nothing masked)
or `unchecked` (the check could not run, for example because the database could not be
reached, with counts of `0`, or has not run yet, with `null` counts). `reasons` lists why the
mode is not `database` (`masked_readable`, `sqlserver_pivot_reads_masked`,
`postgres_inheritance_reads_masked`, `mysql_histogram_reads_masked`, `masked_not_verified`,
`object_not_verifiable`, `unmasked_unreadable`, `view_not_reviewed`, `view_reads_masked`,
`views_not_checked`, `sqlserver_not_verified`, `not_checked`); `limits` lists what the
database does not stop on that engine whatever the mode (`mysql_join_using_natural`,
`mysql_column_statistics`, `mssql_pivot`). `unverified` also counts definer routines not
listed in `"reviewedRoutines"` and MySQL definer views the check could not read
(`object_not_verifiable`); `total` and `denied` count
masked columns only. `columnGrantsRequired` is `ROSIE_REQUIRE_COLUMN_GRANTS` as in effect (by default `false` in express mode and `true` in strict), and `hold` is true
when it is on and that context refuses requests. The `/health` of `npm run mcp:http` carries
the same fields, and its `status` is `security_hold` during a hold. It names no column; the
server log, `npm run doctor` and `/admin/overview` (`status.maskedColumns`) do. A hold makes
`status` `unavailable` here too (since 1.33.0; before, it did not change `status`). How to
read it and what to grant:
[INSTALL.md](INSTALL.md#check-what-the-read-only-account-can-read).

With the gate is on (`ROSIE_REQUIRE_COLUMN_GRANTS`: on by default in strict mode, and can be set explicitly in either mode), `/ask`, `/export`, `/explain` and `/certify` of a
held context answer `503` with
`{ "error": "<message>", "code": "<code>", "supportContact": "<whom to tell>", "securityHold": true, "columnGrantsHold": true, "reference": "<8 hex>" }`,
where `code` is `E_COLUMN_GRANTS_NOT_CHECKED` (not checked yet, the check could not run, or a
reloaded context is being checked), `E_COLUMN_GRANTS_NOT_ENFORCED` ("masked by Rosie only")
or `E_COLUMN_GRANTS_SQLSERVER_NOT_VERIFIED` ("not verified on SQL Server"). The message is in
plain words for the person asking and names no column and no setting (since 1.33.0).

`status` is `degraded` while either audit sink is failing; page on `dbHealthy=false` or
`fileSinkOk=false` (a governance control that fails silently is not a control).
`schemaBlockedTables` names engine tables the database refused as missing or incomplete (run
the schema file). `rejectedRows` counts rows the database refused for what they hold; the
file has them. `columnsMissing` (1.34.0) names engine tables written without the columns a
later version added (the values are in the file; INSTALL.md, Upgrading to 1.34.0); it does not
make `status` degraded. `fileDurability` is `write`, or `fsync` with `ROSIE_AUDIT_FSYNC=true`.
`fileReopened` counts the times the file was opened again after a rotation. Raw database
error text is never included here; it lives in the server logs. `lastFileError` is the text
of the last failure to write the audit file.

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
unchanged. With the column-grant gate on (by default in strict mode) the summary also carries `"pending": true`:
the new context is swapped in once its masked-column check has run, and the context in
service keeps serving until then. See [OPERATIONS.md](OPERATIONS.md).
Since 1.33.0 a reload is recorded in the audit file like the console's changes
(`rosie_admin_change`, `action: "reload"`): the attempt before it runs (the reload is
refused with `503` if that cannot be written), then the outcome with who did it (the
verified administrator, or the `by` of the body under the shared token), the new context
version and whether it worked.

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
the role answers 403; an invalid one, 401. The console page itself is served without a
credential once one is configured: it holds no data and asks for the credential. Refused
requests are counted per address, and past 30 in a minute that address gets 429. Accepted
requests are limited to 120 a minute per administrator.

## GET /admin/evaluation (the admin token)

The last `npm run evaluate` run, in full, as JSON (`docs/EVALUATION.md`). 404 with a message
when no evaluation has been run.

## POST /admin/approve, /admin/describe, /admin/dismiss (the admin token and the write switch)

The three changes the console can make. They need the admin token **and**
`ROSIE_ADMIN_WRITE=true`; without the switch they answer 403.

| Route | Body | Effect |
|---|---|---|
| `/admin/approve` | `{ "messageId", "id", "by", "reviewBy"?, "question"?, "seenSql"? }` | The recorded SQL of that answer becomes the approved query `id`. The server reads the SQL from its own record and validates it. `seenSql` is the statement the reviewer was shown: when it is sent and differs from the record, nothing is approved |
| `/admin/describe` | `{ "table", "column"?, "description", "by" }` | Rewrites one description in the catalog. An empty description removes it |
| `/admin/dismiss` | `{ "messageId", "by", "note"? }` | The answer leaves the review queue |

Answers: 200 `{ "ok": true, ... }`; 403 when writing is off; 404 for an unknown answer, table
or column; 409 when the id exists, the context directory cannot be written, or `seenSql` is
not the statement on record; 422 when the input is refused or the context would not build
with the change, in which case nothing changed; 503 when the attempt could not be written to
the audit file, in which case nothing changed either. `by` is used only with the shared
token; with a JWT the name comes from the token. See [OPERATIONS.md](OPERATIONS.md).

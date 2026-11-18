# Monitoring

Rosie exposes counters in the Prometheus text format, so the questions an operator has
about a deployment have a number behind them: how many questions, how many failed, how
long they took, what the certificate said, and how many answers went out with a check that
could not run.

## Turn it on

The endpoint is off until you set a token for it:

```bash
ROSIE_METRICS_TOKEN=<a long random value>     # openssl rand -hex 32
```

Then scrape `GET /api/assistant/metrics` with that value as a bearer token:

```yaml
scrape_configs:
  - job_name: rosie
    metrics_path: /api/assistant/metrics
    authorization:
      credentials: <the value of ROSIE_METRICS_TOKEN>
    static_configs:
      - targets: ['rosie:3001']
```

The token is separate from `ROSIE_API_TOKEN` on purpose. The monitoring system gets a
credential that reads counters and cannot ask a question, and the API token does not open
this endpoint. While `ROSIE_METRICS_TOKEN` is unset the route answers 404.

Keep the endpoint on the internal network. It holds no data from your database, but it
does describe how the service is used.

## What is exposed

| Metric | Labels | Meaning |
|---|---|---|
| `rosie_questions_total` | `status` | One per audit record. For a question: `success`, `error`, `no_query`, `unfinished` (stopped by a limit before an answer query ran; counted under `no_query` until 1.27.0), `clarification`, `blocked`, `aborted`. Exports and explanations write audit records too, and theirs are counted here as `other` |
| `rosie_blocked_total` | `reason` | Questions refused before any model call: `sensitive_info_requested`, `masked_column_requested`, `write_operation_requested`, `prompt_injection_detected`, and since 1.33.0 `input_too_long`, `question_too_short`, `raw_sql`, `provider_unavailable`. Before 1.33.0 the route's own injection filter and length check were not counted here (nor audited) |
| `rosie_refused_total` | `route`, `reason` | Requests turned away before they were read (since 1.33.0). `route` is `ask`, `export`, `explain`, `feedback`, `certify` or `mcp`; `reason` is `rate_limited`, `busy`, `security_hold`, `column_grants_hold`, `audit_unavailable` or `unauthenticated`. Every refusal is counted; its audit row (`status=refused`) is written at most once a minute per route, reason and address. These are not questions and are not in `rosie_questions_total` |
| `rosie_answers_total` | `source` | Answers that returned a result, by where the statement came from: `model_sql`, `approved_query`, `compiled_metric`, `certified_sql` (a statement sent to `/certify` or `certify_sql`) |
| `rosie_certificates_total` | `verdict` | Certificates issued, by verdict ([CERTIFICATE.md](CERTIFICATE.md)) |
| `rosie_assertions_total` | `check`, `result` | Each check that ran, with `pass`, `warn` or `fail` |
| `rosie_checks_skipped_total` | `check`, `reason` | Each check that was due and did not run, and why |
| `rosie_repairs_total` | `outcome` | Certificate-guided second attempts |
| `rosie_feedback_total` | `rating` | Ratings from users, `up` or `down` |
| `rosie_tokens_total` | `direction` | Model tokens, `input` and `output` |
| `rosie_question_duration_seconds` | histogram | Time to answer, end to end. Buckets at 1, 2, 5, 10, 20, 30, 60 and 120 seconds. Exports and explanations are observed here too |
| `rosie_audit_sink_healthy` | gauge | 1 while the audit trail is being recorded, 0 otherwise |
| `rosie_security_hold` | gauge | 1 while answers are refused because the read-only user has more than SELECT, or (with the column-grant gate on: by default in strict mode, or `ROSIE_REQUIRE_COLUMN_GRANTS=true` in either mode) because the masked-column check did not pass |
| `rosie_uptime_seconds` | gauge | Seconds since the process started |
| `rosie_build_info` | `version` | Always 1 |
| `rosie_context_info` | `version` | Always 1. The label is a digest of the context files in service; replicas should agree ([OPERATIONS.md](OPERATIONS.md)) |
| `rosie_context_loaded_timestamp_seconds` | gauge | When the context in service was loaded |
| `rosie_schema_drift_checked` | gauge | 1 when the catalog was compared with the database, 0 when the check is off or could not run |
| `rosie_schema_drift_objects` | `kind` | Tables and columns the catalog names that the database no longer has |
| `rosie_approved_queries_suspended` | gauge | Approved queries and examples suspended because they read one of those |
| `rosie_evaluation_*` | gauges | Accuracy, outcomes, wrong answers by warning, regressions and time of the last `npm run evaluate`. Present once one has run ([EVALUATION.md](EVALUATION.md)) |

The counters are fed at the audit point, which every transport passes. But they live in the
memory of the process that answered, and only the HTTP server (`npm start`) serves
`/metrics`. A question asked through an MCP process (`npm run mcp`, `npm run mcp:http`) is
counted in that process, which has no metrics route, so it does not appear in a scrape. Its
audit rows are in the same engine tables: read MCP usage from there
(`npm run weekly-report`, `npm run audit-report`). The schema-drift gauges are also those of
the HTTP server; MCP processes do not run that check.

## What never becomes a label

No question text, no SQL, no user id, no table or column name, no comment a user wrote.
Label values come from fixed lists, and anything outside them is reported as `other`. A
refusal for a masked column is counted as `masked_column_requested`; which column it was
stays in the audit table, where access is controlled. Since 1.33.0 that is also true of a
masked column the model tried inside its loop and the validator refused: the audit row of
that question carries `rejection_reason` `statement_refused:masked_column` (status
`success` when another statement answered), and the audit file's `sql_trace` marks the
statement `refused` with its reason.

## Alerts worth having

```yaml
groups:
  - name: rosie
    rules:
      - alert: RosieAuditTrailNotRecording
        expr: rosie_audit_sink_healthy == 0
        for: 2m
      - alert: RosieSecurityHold
        expr: rosie_security_hold == 1
        for: 5m
      - alert: RosieErrorRate
        expr: |
          sum(rate(rosie_questions_total{status="error"}[15m]))
            / sum(rate(rosie_questions_total{status!="other"}[15m])) > 0.1
        for: 15m
      - alert: RosieSlowAnswers
        expr: |
          histogram_quantile(0.9, sum(rate(rosie_question_duration_seconds_bucket[15m])) by (le)) > 30
        for: 15m
      - alert: RosieSchemaDrift
        expr: sum(rosie_schema_drift_objects) > 0
        for: 10m
      - alert: RosieContradictedAnswers
        expr: increase(rosie_certificates_total{verdict="CONTRADICTED"}[1h]) > 0
      - alert: RosieTurningPeopleAway
        expr: sum(increase(rosie_refused_total{reason=~"rate_limited|busy"}[1h])) > 20
      - alert: RosieInjectionAttempts
        expr: increase(rosie_blocked_total{reason="prompt_injection_detected"}[1h]) > 5
```

Two ratios are worth a panel even without an alert:

- **Answers that went out unchecked.** `rosie_certificates_total{verdict="NOT_CHECKED"}`
  over all certificates. When it grows, `rosie_checks_skipped_total` says which check and
  why; `shape_not_covered` means people are asking questions whose SQL the check does not
  handle yet.
- **Answers from governed paths.** `approved_query` and `compiled_metric` over all of
  `rosie_answers_total`. This is the share of answers whose statement a person reviewed or
  the engine wrote from a definition, and it should grow as the review queue is worked
  ([APPROVED_QUERIES.md](APPROVED_QUERIES.md), [METRICS.md](METRICS.md)).

## Log events worth an alert

Some conditions have no metric yet and appear only in the server log, one JSON line with an
`event` field. These are the ones added in 1.30.0 and 1.31.0. Match on `event`, never on the
message text. The masked-column check has no metric for its result; `/health` carries it as
`columnGrants` ([API.md](API.md)).

| Event | Level | Meaning |
|---|---|---|
| `masking_mode_express` | info | Express mode (1.32.0), once per context with masked columns, and again only when the result changes. Fields `projectId`, `mode`, `denied`, `total`, `readable`. For "masked by Rosie only" it says the validator refuses the masked columns and the account can read them, with how to move to strict; for "not verified on SQL Server", that the database is not verified to deny them; for "masked by the database", that the database denies them too. In express mode `masked_column_readable`, `masked_column_not_verified`, `object_not_verifiable`, `unmasked_columns_unreadable`, `view_not_reviewed`, `view_reads_masked`, `views_not_checked` and `column_grants_checked` are logged at `debug`; `column_grants_not_checked` stays a warning ([INSTALL.md](INSTALL.md#choosing-a-masking-mode)) |
| `masked_column_readable` | warn | The read-only account can read a masked column, directly or through a role, a PostgreSQL partition or inheritance relative, a MySQL histogram, or on SQL Server a `PIVOT` (named in `roles`). Logged the first time and when the result changes ([INSTALL.md](INSTALL.md#check-what-the-read-only-account-can-read)) |
| `masked_column_not_verified` | warn | A masked column whose privilege could not be verified (a mask the catalog does not list, a probe that failed) |
| `unmasked_columns_unreadable` | warn | The account cannot read every unmasked column of a catalog table (any table, since 1.31.0), so questions on that table will fail and the context is not reported as masked by the database: grant those columns. Carries `columns`. With `probe` other than `denied` or `missing`, the probe failed for another reason |
| `object_not_verifiable` | warn | The context holds objects the check cannot establish. Since 1.32.0 mostly the allowlist: the account can read or execute objects beyond the catalog tables and the reviewed ones; one line per check result with the first 20 named (`detail`), the list in `views.outside` and `views.outsideCount` of the state. Also a reviewed object whose behaviour cannot be established ([INSTALL.md](INSTALL.md#check-what-the-read-only-account-can-read)) |
| `column_grants_checked` | info or warn | The summary of one check, per context: info when the context is "masked by the database", warn otherwise. Carries `reasons` and, since 1.31.0, `limits` |
| `column_grants_not_checked` | warn | The check could not run, for example because the database could not be reached. Since 1.32.0 a check that times out, fails, or whose pool replacement fails also leaves the context "not checked" (its `error` is `check_timeout`, `check_failed` or `pool_drain_failed`; see `column_grants_check_timeout`, `read_only_pool_drain_failed`), and the run is retried (`column_grants_retry_scheduled`) |
| `audit_columns_missing` | error | `rosie_audit_log` has no `context_version`, `masking_state`, `columns_returned` and `personal_data` columns (an installation from before 1.34.0). Logged once; rows are still written without them and the values go to the audit file only. `/health` lists the table under `audit.columnsMissing`. Add the columns ([INSTALL.md](INSTALL.md), Upgrading to 1.34.0) |
| `view_not_reviewed` | warn | A catalog table is a view and has no `"reviewedView": true` (1.31.0) |
| `view_reads_masked` | warn | A view or function the account can read reads a masked column (`view`, `reads`, `reviewed`, `inCatalog`); PostgreSQL and SQL Server (1.31.0) |
| `views_not_checked` | warn | The views could not be read, so the context is not reported as masked by the database (1.31.0) |
| `read_only_pool_drained` | info | The read-only pool's connections were replaced after the check's result changed or a reload (`reason`, `from`, `to`) (1.31.0), and since 1.32.0, with the gate on, after every passing check (`reason` `passing_check`) |
| `column_grants_required` | info | Once at every start of the HTTP server and both MCP servers (since 1.32.0; in 1.31.0 only when the gate was on): `maskingMode`, `required`, `source` (`masking mode` when the gate follows the mode's default, `ROSIE_REQUIRE_COLUMN_GRANTS` when it was set explicitly) and, for an explicit value, `raw` (the value as written; the message reads `set explicitly: ROSIE_REQUIRE_COLUMN_GRANTS="<raw>", read as on` or `off`). An alert keyed on its presence now fires on every start; key it on `required` |
| `masking_mode_invalid` | error | `ROSIE_MASKING_MODE` is neither `express` nor `strict` (read without letter case); the HTTP server exits 1. Until 1.33.0 also any other unreadable setting, now `config_invalid`. The MCP servers exit 1 with the same message in `mcp_server_failed` or `mcp_http_failed` (1.32.0) |
| `config_invalid` | error | A setting cannot be read one way: a yes-or-no value that is neither, a number outside its range or written with a unit (`10s`, `1e3`), an unknown value of a closed list (`ROSIE_RATE_LIMIT_STORE`, `LOG_LEVEL`, `DB_DIALECT`, `TRUST_PROXY`, ...; [OPERATIONS.md](OPERATIONS.md#settings-reference)). The message names each setting and what it accepts; the HTTP server exits 1 (1.33.0) |
| `startup_failed` | error | The server cannot start: `reason` is `db_database_missing`, `context_dir_missing`, `context_dir_not_found`, `context_file_missing` (for example `system-prompt.txt`, which the wizard writes with the model), `context_value_invalid` (a `calendar` value the prompt cannot carry) or the code of what the engine refused to load. One line, exit 1, no stack trace (1.33.0) |
| `setting_unknown` | warn | At startup: a `ROSIE_` name Rosie does not read, or an `AI_`/`DB_` name one typo away from a setting, with the closest one ("did you mean AI_MODEL?") (1.33.0) |
| `setting_suspicious` | warn | At startup: a value that is accepted but probably not meant (`NODE_ENV=staging` read as development, an origin in `ROSIE_CORS_ORIGIN` with a path or trailing slash, an unknown `AI_REASONING_EFFORT`) (1.33.0) |
| `context_key_unknown` | warn | At startup: keys or values in the context files that the engine ignores or cannot read, with the closest known key (`npm run check-context` lists all) (1.33.0) |
| `assistant_disabled` | warn | At startup: `AI_ASSISTANT_ENABLED=false`, so the answering routes answer 404 and `/health` answers 503 (1.33.0) |
| `model_key_unusable` | warn | At startup: `OPENAI_API_KEY` is unset, still the `.env.example` placeholder, or holds a space or a quote; every question is refused (1.33.0) |
| `model_key_refused` | error | The model provider answered 401 or 403: "the model provider refused the key: check OPENAI_API_KEY / AI_BASE_URL". Logged once; not again until a model call succeeds or 10 minutes pass without a refusal. `/health` is `degraded` (200, `model_key_refused` in `refusing`) meanwhile, from the second refusal or 30 seconds after the first, never 503: it makes no model call to see the key recover. `/health` answers 503 meanwhile (1.33.0) |
| `handle_secret_unset` | warn | At startup with `NODE_ENV=production`: `ROSIE_HANDLE_SECRET` is not set, so answer handles die at every restart and are refused by other replicas (1.33.0) |
| `cors_origin_refused` | warn | A browser request from an origin not in `ROSIE_CORS_ORIGIN` was answered 403; once per origin per process, with the `origin` (1.33.0) |
| `column_grants_hold` | error | The column-grant gate is on (strict mode by default, or `ROSIE_REQUIRE_COLUMN_GRANTS=true`) and a context became held (`code`, `mode`, `reasons`). Carries `security: true` (1.31.0) |
| `column_grants_hold_released` | info | A held context passes the check again. Carries `security: true` (1.31.0) |
| `column_grants_request_refused` | warn | Requests refused by the hold; at most one line a minute per context, `refused` is the count since the last. Carries `security: true` (1.31.0) |
| `column_grants_lease_expired` | error | With the gate on, the last passing result of a context is older than `ROSIE_COLUMN_GRANTS_MAX_AGE_MS`: requests are refused until a check passes (`ageMs`, `maxAgeMs`). Carries `security: true` (1.32.0) |
| `column_grants_check_timeout` | error | A check did not finish within `ROSIE_COLUMN_GRANTS_CHECK_TIMEOUT_MS`; the context is "not checked". Carries `security: true` (1.32.0) |
| `column_grants_run_skipped` | warn | A scheduled check was skipped because the previous one was still queued or running (1.32.0) |
| `read_only_pool_drain_failed` | error | The read-only pool could not be replaced after a passing check (`failure`: `timeout`, `error` or `not_replaced`); the context is "not checked". Carries `security: true` (1.32.0) |
| `read_only_pool_close_failed` | warn | An old read-only pool could not be closed; its connections are no longer handed out (1.32.0) |
| `column_grants_commit_failed` | error | Publishing a check result failed; the lease is dropped, so the gate holds (1.32.0) |
| `column_grants_run_failed` | error | A queued check run failed (1.32.0) |
| `column_grants_retry_scheduled` | warn | A run left a context "not checked" (failed, timed out, unreachable, pool replacement failed); one retry is scheduled after 30, 60, then 120 seconds for consecutive failures (`attempt`, `delayMs`, `projectIds`), then the normal interval (1.32.0) |
| `context_reload_pending` | info | A reload waits for its masked-column check before it serves (flag on) (1.31.0) |
| `column_grants_state_dropped` | info | A check finished for a context version no longer in service, or while a reload waited for its own check, and its result was not stored (`reason` `replaced` or `reload_pending`, `contextVersion`, `mode`) (1.31.0) |
| `db_permission_denied` | warn | The database refused a statement for lack of privilege. Usually a statement that reached a masked column and that Rosie's validator did not refuse. Carries `security: true` |
| `policy_rewrite_revalidation_failed` | warn | A statement rewritten for a row policy did not pass the validator again. Logged, not enforced, in this release ([ROW_POLICIES.md](ROW_POLICIES.md)). Carries `security: true` |
| `previous_answer_dropped` | info | A follow-up's previous statement no longer passes the validator, so the follow-up starts from nothing |
| `metric_withheld_masked` | warn | Metrics left out of the prompt and of `list_metrics` because they read a masked column |
| `masked_columns_duplicate_keys` | warn | The masks name one table under two keys that differ only by case. Both are applied; keep one |
| `request_refused` | warn | A request turned away before it was read (1.33.0): `route`, `reason` (as in `rosie_refused_total`), `httpStatus`, `userId`, `correlationId`. One line per refusal |
| `clarification_rule_unsafe_regex` | error | A rule of `clarification-rules.json` has a repeated group that repeats inside (`(a+)+`, `(\w+\s?)*`), which can take exponential time on a crafted question; the rule is skipped (1.33.0), and `npm run check-context` fails on it by id. Rewrite it with one level of repetition or a bound (`{0,5}`). Carries `rule`. This is a lint for the plain shapes, not a guarantee: nested groups (`((a+))+`), overlapping alternatives (`(a|a)*`), a counted repetition (`(a+){20}`) and similar still load and can be slow. The rules are written by the operator, with the same trust as the system prompt; test a new rule on a long question before putting it in service |
| `error_response` | info | Every error response of the HTTP routes (1.33.0): `route`, `httpStatus`, `reference` (the 8 characters the user sees), `correlationId`, `code`. Search by the reference a user quotes |
| `statement_refused` | warn | The validator or a row policy refused a statement the model sent to a query tool (1.33.0): `tool`, `reason` (`masked_column`, `table_not_allowed`, `column_not_allowed`, `function_not_allowed`, `not_a_select`, `parse_error`, `prohibited_pattern`, `row_policy`, `validation_failed`), `userId`, `errors`, and the request's `correlationId`. The tool's own `validation_failed` line has no correlation id; match this one. `statement_failed` (info) is the same for a database error |
| `masked_name_in_context_text` | warn | Text in the context (prompt, descriptions, examples) writes a masked `table.column`, and that text is sent to the model as written |

`masked_column_readable`, `column_grants_hold` and `db_permission_denied` are the ones to
page on: the first says a masked column is within reach of the read-only account, the second
that Rosie is refusing every request of a context, the third that a statement reached a
masked column and only the database stopped it. On MySQL with case-sensitive table names, a
`db_permission_denied` with code 1142 before 1.31.0 could also be a table written in another
letter case, with no masked column involved.

## Limits

- Counters live in the process and restart at zero with it, which is what Prometheus
  expects of a counter. With several replicas, scrape each one.
- The counters say how the service behaves, never whether an answer was right. For that,
  run your own question bank with `npm run evaluate` ([EVALUATION.md](EVALUATION.md)) and
  work the review queue (`npm run review`). `npm run weekly-report` puts both next to the
  usage of the week on one page ([OPERATIONS.md](OPERATIONS.md)).
- `GET /api/assistant/health` stays public and minimal. Since 1.33.0 it answers 503 with
  `status: "unavailable"` and a `refusing` list while every question is refused (a security
  or column-grant hold, the assistant switched off, the model key missing, the
  audit policy blocking), so a load balancer and the Docker `HEALTHCHECK` see it. It is a
  readiness check: do not restart a process on it, the cause is configuration, not the
  process ([API.md](API.md#get-health-public)). Use the metrics endpoint for everything else.

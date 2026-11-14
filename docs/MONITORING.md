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
| `rosie_questions_total` | `status` | One per audit record. For a question: `success`, `error`, `no_query`, `clarification`, `blocked`, `aborted`. Exports and explanations write audit records too, and theirs are counted here as `other` |
| `rosie_blocked_total` | `reason` | Questions refused before any model call: `sensitive_info_requested`, `masked_column_requested`, `write_operation_requested`, `prompt_injection_detected` |
| `rosie_answers_total` | `source` | Answers that returned a result, by where the statement came from: `model_sql`, `approved_query`, `compiled_metric`, `certified_sql` (a statement sent to `/certify` or `certify_sql`) |
| `rosie_certificates_total` | `verdict` | Certificates issued, by verdict ([CERTIFICATE.md](CERTIFICATE.md)) |
| `rosie_assertions_total` | `check`, `result` | Each check that ran, with `pass`, `warn` or `fail` |
| `rosie_checks_skipped_total` | `check`, `reason` | Each check that was due and did not run, and why |
| `rosie_repairs_total` | `outcome` | Certificate-guided second attempts |
| `rosie_feedback_total` | `rating` | Ratings from users, `up` or `down` |
| `rosie_tokens_total` | `direction` | Model tokens, `input` and `output` |
| `rosie_question_duration_seconds` | histogram | Time to answer, end to end. Buckets at 1, 2, 5, 10, 20, 30, 60 and 120 seconds. Exports and explanations are observed here too |
| `rosie_audit_sink_healthy` | gauge | 1 while the audit trail is being recorded, 0 otherwise |
| `rosie_security_hold` | gauge | 1 while answers are refused because the read-only user has more than SELECT |
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
stays in the audit table, where access is controlled.

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

## Limits

- Counters live in the process and restart at zero with it, which is what Prometheus
  expects of a counter. With several replicas, scrape each one.
- The counters say how the service behaves, never whether an answer was right. For that,
  run your own question bank with `npm run evaluate` ([EVALUATION.md](EVALUATION.md)) and
  work the review queue (`npm run review`). `npm run weekly-report` puts both next to the
  usage of the week on one page ([OPERATIONS.md](OPERATIONS.md)).
- `GET /api/assistant/health` stays public and minimal. Use it for liveness, and the
  metrics endpoint for everything else.

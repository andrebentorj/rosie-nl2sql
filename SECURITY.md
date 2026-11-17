# Security

Rosie runs natural-language questions as read-only SQL against production databases, so
anything that lets a question reach data it should not (masked columns, other schemas, write
or locking statements, information disclosure) is a security issue, not a bug.

**Reporting.** Report privately to the vendor contact named in your license agreement, and
to the deployment owner named in `project.json` `governance.owner`. Do not open a public
ticket with reproduction SQL. Include the exact SQL or question, the catalog you used and the
engine version (`package.json`).

**Handling.** Confirmed validator findings get a regression case in
`test/security/battery.js` together with the fix.

**Defense in depth (do not weaken one layer because another exists).** AST validator with a
catalog-derived allowlist and PII mask, SELECT-only database user verified at startup and
every 15 minutes, column privileges that deny the masked columns to that user (checked and
reported per context, and required before answering with `ROSIE_REQUIRE_COLUMN_GRANTS=true`;
written by `npm run init -- --create-ro-user` or printed by `npm run grants`, never changed by
the server; on MySQL they do not cover joins `USING` a masked column, histograms or definer
stored functions, which the validator refuses and the grant check reports; on SQL Server
they do not stop `PIVOT`, which only the validator stops),
prompt-injection and write-intent filters before the model, signed answer
handles for follow-ups, per-request budgets, rate and concurrency limits, durable audit trail.

**Dependencies.** Run `npm run audit:deps` (`npm audit --audit-level=high`) before each
deployment. Install with `npm ci` only; `.npmrc` refuses Node versions below 22.

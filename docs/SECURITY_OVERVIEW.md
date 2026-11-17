# Security overview

Written for the person who has to approve Rosie: a security reviewer, an architect, a
procurement questionnaire. Each answer says what the product does, where to see it, and
what it does not do. The last section lists the gaps plainly.

Related pages: [DATA_PRIVACY.md](DATA_PRIVACY.md) (what is sent to the model provider),
[IDENTITY.md](IDENTITY.md), [ROW_POLICIES.md](ROW_POLICIES.md),
[CERTIFICATE.md](CERTIFICATE.md), [MONITORING.md](MONITORING.md),
[OPERATIONS.md](OPERATIONS.md).

## What it is, in one paragraph

Rosie is a server you run in your own infrastructure. A user asks a question in plain
language; a language model writes a SQL statement; Rosie validates that statement against an
allowlist, runs it on a read-only connection to your database, checks the result, and
returns the rows with a record of what was checked. There is no vendor-hosted component:
nothing is sent to the vendor of Rosie, and there is no licence server to call.

## Architecture and data flow

```
 user's browser or AI client
        |  HTTPS (your proxy), bearer token or your JWT
        v
   Rosie server  (one container, stateless)
        |-- read-only pool ----> your database         SELECT only, row cap, timeout
        |-- main pool ---------> Rosie's own tables    audit, certificates, ratings
        |-- HTTPS -------------> model provider        the one external party; can be a
        |                                              model you host
        '-- HTTPS (optional) --> your identity provider, for JWT signing keys
```

| Component | Runs where | Holds |
|---|---|---|
| Rosie server | Your infrastructure, one container | No rows from your database. On its log volume: the audit file (questions, generated SQL, outcomes) and the list of reviewed answers of the console. Configuration from environment variables and a context directory mounted read-only (writable only if you turn on changes from the console) |
| Context directory | Your file system | The catalog (allowlist), masked columns, instructions, metrics, approved queries, row policies. No rows from your database |
| Engine tables | A database you choose (can be separate from the one being queried) | Questions, generated SQL, outcomes, certificates, ratings. Never result rows |
| Model provider | The endpoint you configure | See [DATA_PRIVACY.md](DATA_PRIVACY.md). Can be a model inside your network |

## Access to your database

| Question | Answer |
|---|---|
| Can Rosie change data? | No. Statements are parsed and only a single `SELECT` passes. The connection uses a database user that has `SELECT` and nothing else, and Rosie checks that user's privileges at startup and every 15 minutes. If it finds more than `SELECT`, every question is refused until it is fixed |
| Which tables and columns can it read? | Only those in the catalog file. A statement that names anything else is rejected before it runs. Leaving a table out of the catalog keeps it out of reach |
| How is personal data protected? | In two layers. Rosie: columns listed in `masked-columns.json` are refused when Rosie finds them in a statement, and are not sampled, looked up or described to the model. This is Rosie's own reading of SQL, and independent reviews have found ways around it more than once (see "Corrections" and the advisory below). The database: give the read-only account no privilege on the masked columns ([INSTALL.md](INSTALL.md), Step 2; `npm run grants` prints the statements). Rosie checks the account at startup and every 15 minutes and reports per context "masked by the database" or "masked by Rosie only", with reason codes ([How to read the check](INSTALL.md#check-what-the-read-only-account-can-read)). It answers either way, unless `ROSIE_REQUIRE_COLUMN_GRANTS=true`, which refuses every request of a context that is not "masked by the database" or "no masked columns". What the database layer can give differs by engine; each sentence is about the read-only account, when the check passes, as of its last run (default every 15 minutes). PostgreSQL: the database refuses every statement that reaches a masked column of a catalog table, for the account and every role it is a member of, including through the table's partitions, inheritance children and parents, through any view that depends on the column, and through `SECURITY DEFINER` functions the account may execute (each held until it is listed in `"reviewedRoutines"`). MySQL: the database refuses a statement that reads a masked column by name; the check also reports histograms on masked columns and definer views and routines the account can use (a view whose definition cannot be read with confidence is reported `object_not_verifiable`). The database does not stop a `USING` or `NATURAL` join on a masked column, a histogram created between checks, or a definer routine; Rosie's validator refuses those (any function that is not a listed built-in included) and is a required layer there, so give the account no `EXECUTE` on routines. SQL Server: with column grants, no database claim (a `PIVOT` reads the masked columns of any table the account can read one column of); with no permission on the masked tables and a reviewed catalog view, the database refuses every statement on those tables, and what the view shows is the operator's statement. On PostgreSQL and MySQL, a definer routine or function the account may execute keeps the context out of "masked by the database" until it is listed in `"reviewedRoutines"` in the catalog, the operator's statement that it returns no masked value. Outside both layers on every engine: what a view marked `reviewedView` really exposes, objects the account cannot see, anything created between two checks, copies in other columns, and what your unmasked columns reveal |
| Can a user see another tenant's rows? | Not with [row policies](ROW_POLICIES.md): a filter derived from the caller's verified identity is added to every read, including approved queries and compiled metrics. The filtered read names the table's unmasked columns, never `*`, so it also runs under column grants. A policy may not use a masked column |
| What stops a runaway query? | A statement timeout enforced on the connection (default 10 s), a cap of 500 rows per answer, at most 5 queries per question, and a response size limit |
| What about SQL injection? | The user never supplies SQL; a question that starts with a SQL statement is refused. The model's SQL is treated as untrusted input: parsed to a syntax tree, checked against the allowlist, with set operations (all but `UNION ALL` inside a derived table, where each branch is validated on its own and filtered by the row policies), system schemas, file access, session variables and information functions rejected. Values in approved queries are typed and bound, never concatenated |
| And prompt injection? | A filter refuses known injection patterns before the model is called. Data returned by tools is marked as data for the model. Neither is the control that matters: whatever the model is talked into writing still has to pass the validator and run as a `SELECT`-only user |
| Is the connection encrypted? | With `DB_SSL=true`, TLS with certificate verification on MySQL, PostgreSQL and SQL Server |

The validator has its own regression suite of adversarial statements (more than 850 cases),
run on every commit and before every release. `npm run test:security` prints the count. A
suite checks the shapes someone thought of; independent reviews have found new ones after
it passed, which is why the database grant matters.

## Authentication and authorization

| Question | Answer |
|---|---|
| How do callers authenticate? | Either a static bearer token held by your application's backend, or a JWT issued by your identity provider ([IDENTITY.md](IDENTITY.md)) |
| Which JWT algorithms? | Asymmetric only: RS256, ES256, PS256, EdDSA. `none` and the shared-secret HS family are never accepted. Issuer, audience and expiry are verified |
| Is there SSO? | Rosie has no login screen. It accepts the tokens of the identity provider you already use, so SSO is whatever your application has |
| Unauthenticated in production? | No. With `NODE_ENV=production` and no token configured, every request is refused |
| Are administrative actions separated? | Yes. Metrics and administration (context reload, the read-only overview and its console page) each have their own secret (`ROSIE_METRICS_TOKEN`, `ROSIE_ADMIN_TOKEN`); the application token opens neither. The routes do not exist until their secret is set. Administrators can sign in with their own token from your identity provider instead of a shared secret: it must carry a role you name, a token without it is refused, and each change is recorded under the verified name. The console is read-only by default. With `ROSIE_ADMIN_WRITE=true` it can approve an answer as a query, edit a description and clear an item of the review queue, each recorded in the audit file; it can never change which tables or columns are readable, the masks, the row policies or the project's prompt file. A table description written from the console is shown to the model with the catalog, so it does influence answers; text that addresses the assistant is refused, and every change is recorded |
| Can one user replay another's answer? | Not under JWT. Follow-up actions (export, explain) need a handle signed by the server, bound to the caller identity and the project, valid for 15 minutes. With JWT the identity is the verified user, and whatever a handle runs is filtered by the redeemer's own row policies. With the shared static token every caller is the same identity, so a handle is bound to the installation and not to a person: what protects it then is that it is returned only to whoever asked. The SQL inside is validated again before it runs |
| Rate limits? | Per client address: 100 questions an hour and 1,000 a day by default, and 3 concurrent questions per process. With several replicas the quotas can be shared through the engine database (`ROSIE_RATE_LIMIT_STORE=database`) |

## What leaves your network

Only what is sent to the model provider you configured, itemised in
[DATA_PRIVACY.md](DATA_PRIVACY.md): the question, schema metadata of unmasked columns (with
their descriptions, and the data notes when the catalog carries them), and a sample of up to
30 result rows per query so the model can write the answer. Rows outside the caller's policy
are not part of it. Masked columns are not part of it when Rosie's validator finds them in a
statement, and, for what it misses, when the read-only account has no privilege on them
(see "How is personal data protected?"). Pointing `AI_BASE_URL`
at a model you host makes that nothing.

Rosie makes no other outbound call. There is no telemetry to the vendor, no update check
and no licence check.

## Logging, audit and retention

| Question | Answer |
|---|---|
| What is recorded? | One audit row per question (who, the question, the SQL, the outcome, timings, tokens), the certificate of each answer, and ratings. Refusals are recorded with their reason |
| Can a caller send SQL? | Not by default: a caller asks in words, or names a declared metric or an approved query. With `ROSIE_CERTIFY_SQL=true`, `POST /certify` and the MCP tool `certify_sql` accept a `SELECT` written by the caller and return it with a certificate. It passes the same validator (read-only, catalog allowlist, masked columns), the caller's row policies, the row cap and the read-only connection, and is audited. What it changes is who chooses the statement |
| Can the trail lose a record? | Under the conditions listed here, yes, and each one is reported. Each record is written to a local append-only file first, with a synchronous write that completes before the answer is sent, then to the database, with retries. When that file write succeeds, a crash of the server loses nothing, and with `ROSIE_AUDIT_FSYNC=true` neither does a crash of the host. The file survives a replaced container only when its directory is a mounted volume, which the installation steps do and `npm run doctor` checks. It survives log rotation: the file is opened again at its path when it was renamed or removed. A record is lost only if the file write fails (a full disk) **and** the process dies before the database insert completes; the failed write is counted on `/health` from that moment. A row the database refuses for what it holds is not retried and is counted (`rejectedRows`); the file has it. The table keeps 500 characters of a question, the file keeps all of it. An administrative change is refused unless its attempt was first written to the file. `GET /health` and the `rosie_audit_sink_healthy` metric report a failing sink. With `ROSIE_AUDIT_POLICY=fail-closed`, Rosie refuses to answer while nothing can record |
| Are result rows stored? | No |
| Retention? | You set it. `npm run retention -- --days 365` reports what is older than the period, and with `--confirm` removes it from the three log tables (`rosie_audit_log`, `rosie_answer_certificate`, `rosie_feedback`). Nothing is removed unless you schedule that command. The account that runs it needs DELETE on those tables, which the account of the server does not need (`docs/INSTALL.md`, Step 3). The fourth table, `rosie_rate_limit`, holds client addresses for about two windows and is cleaned by the engine |
| What is in application logs? | Structured events with identifiers and reasons. Privilege details stay in the server log and are not returned to callers. A database error is logged by its class and code; its text, which can quote a stored value, is logged only at `LOG_LEVEL=debug` (since 1.30.0; before, at error level) |

```cron
0 4 * * 0  docker run --rm --env-file /opt/rosie/.env <image> npm run -s retention -- --days 365 --confirm
```

## Supply chain

| Question | Answer |
|---|---|
| How is the software delivered? | A Docker image in a private registry, access granted per customer and revocable |
| Is the image signed? | Yes, from version 1.5.0. Each release is signed with cosign using the identity of the release workflow (keyless, no long-lived signing key). The verification command is below |
| Is there an SBOM? | Yes, from version 1.5.0. An SPDX list of every operating system and npm package in the image, attached to the image as a signed attestation |
| How are dependencies controlled? | A lock file with integrity hashes, installed with `npm ci` and install scripts disabled. `npm audit` at high severity blocks the release |
| Does the container run as root? | No. It runs as an unprivileged user; the code is read-only to it |
| What is tested before a release? | The security suite and the unit suite must pass, and the built image must run and report the version of the tag |

### Verify what you received

With [cosign](https://docs.sigstore.dev/cosign/system_config/installation/) installed and
after `docker login ghcr.io`:

```bash
IMAGE=ghcr.io/andrebentorj/rosie:1.5.0
IDENTITY='^https://github\.com/andrebentorj/rosie/\.github/workflows/release\.yml@refs/tags/v[0-9.]+$'
ISSUER=https://token.actions.githubusercontent.com

# The image was built and signed by the vendor's release workflow, from a version tag.
cosign verify "$IMAGE" --certificate-identity-regexp "$IDENTITY" --certificate-oidc-issuer "$ISSUER"

# The list of every package inside it (SPDX), signed the same way.
cosign verify-attestation "$IMAGE" --type spdxjson \
  --certificate-identity-regexp "$IDENTITY" --certificate-oidc-issuer "$ISSUER" \
  | jq -r '.payload | @base64d | fromjson | .predicate' > rosie-sbom.spdx.json
```

A failed verification means the image is not the one the vendor released: do not run it.
Pin the digest that `cosign verify` prints in your deployment, so what runs is what you
verified.

## Operations

| Question | Answer |
|---|---|
| Is it stateless? | Yes. Any replica can serve any request. Set `ROSIE_HANDLE_SECRET` so replicas accept each other's answer handles |
| How do I monitor it? | Prometheus metrics of the HTTP server and suggested alerts in [MONITORING.md](MONITORING.md) |
| How do I change the catalog safely? | Reload without restart, all or nothing ([OPERATIONS.md](OPERATIONS.md)) |
| What if the database schema changes? | The HTTP server detects it, logs it and counts it, and approved queries that depend on what changed are suspended ([OPERATIONS.md](OPERATIONS.md)). The MCP processes do not run that check |
| How do I revoke access? | Rotate `ROSIE_API_TOKEN`, or revoke the user at your identity provider. Rotating `ROSIE_HANDLE_SECRET` invalidates every outstanding handle. Both need a restart of every replica; the steps are in [OPERATIONS.md](OPERATIONS.md), "Rotating a secret". Remove the customer from the registry to stop new versions. Masking a column takes effect at the next reload, for the next question; a question already running finishes under the old rules, and nothing already sent to the provider or stored in the audit trail, logs or exported files is undone. When a mask is urgent, take the column's privilege in the database first |
| How do I know answers are right? | The certificate shows what was checked on each answer and what was not ([CERTIFICATE.md](CERTIFICATE.md)). `npm run evaluate` measures accuracy on your own questions ([EVALUATION.md](EVALUATION.md)) |

## Vulnerabilities

Report privately to the vendor contact in your agreement. A confirmed finding in the
validator is fixed together with a regression case in the security suite. See `SECURITY.md`.

## Advisory: masked columns in 1.29.2 and earlier

**Rosie 1.29.2 and earlier: masked columns can be read or filtered on. Act now: upgrade to
1.30.0 or later and take the privileges away in the database. Upgrading alone is not enough.**

**What is wrong.** Rosie's validator can miss a masked column in a statement. Confirmed on
PostgreSQL: a double-quoted masked column inside `SUBSTRING`, `POSITION`, `STRING_AGG`,
`LIKE`, `ILIKE`, `SIMILAR TO`, `ESCAPE`, `~`, `~*` or `AT TIME ZONE`; any masked column after
`IS DISTINCT FROM`; and statements that use two table aliases differing only by letter case.
Confirmed on MySQL (case-sensitive table names, the Linux default) and on PostgreSQL: aliases
that differ only by case or by a look-alike letter, when another table has an unmasked column
of the same name. MySQL servers running with `ANSI_QUOTES` have the first problem too. SQL
Server was not tested on a live server. Separately: if `masked-columns.json` (or
`maskedColumns` in the catalog) names one table under two spellings that differ only by
case, only the first entry is applied. 1.30.0 closes each of these in the validator and the
mask files (see "Corrections"), but the same kind of defect has been found in four review
rounds in a row, so do not rely on the validator alone.

**Who can reach it.** Anyone who can ask questions, through the statements the model writes,
and directly any caller of `/certify` or `certify_sql` where `ROSIE_CERTIFY_SQL=true`. Values
can reach the caller, the model provider, exports, and the process log (database error
messages quote values).

**What to do.**

1. Take the masked columns away in the database. PostgreSQL: no table-level `SELECT` on a
   table that has masked columns; grant `SELECT (col, ...)` for the unmasked columns only. A
   column `REVOKE` under a table grant changes nothing. Check PUBLIC grants, membership in
   `pg_read_all_data`, and the roles the account can switch to. MySQL: no
   `GRANT SELECT ON db.*`; grant table by table, and column by column on masked tables. A
   narrower grant does not remove a broader one. Restart Rosie afterwards. SQL Server: grant
   columns, no `db_datareader`, no schema-wide grant; since 1.31.0 it is known that column
   grants there do not stop `PIVOT` ([SQLSERVER.md](SQLSERVER.md)). The statements are in
   [INSTALL.md](INSTALL.md), Step 2, [POSTGRES.md](POSTGRES.md) and
   [SQLSERVER.md](SQLSERVER.md); from 1.31.0 `npm run grants` prints them for the account
   as it is.
2. Check it. As the read-only account, `SELECT <masked column> FROM <table> WHERE 1 = 0` must
   fail with a permission error. From 1.30.0 Rosie runs this check at startup and shows the
   result in `/health` ([INSTALL.md](INSTALL.md#check-what-the-read-only-account-can-read)).
3. Row policies. In 1.29.2 a table with both a row policy and a masked column cannot be read
   under column grants (the policy filter reads the table with `SELECT *`). 1.30.0 names the
   unmasked columns instead. If you cannot upgrade yet, remove the policy on those tables and
   restrict them by other means.
4. Set `ROSIE_CERTIFY_SQL=false` unless you need it.
5. Look at your mask files for one table written twice with different letter case, and merge
   the entries. From 1.30.0 both entries apply, and `npm run check-context` reports the
   duplicate.
6. On 1.29.2, before running `npm run init` again after adding masks: delete
   `.bootstrap/samples.json` and remove `enumValues` from the newly masked columns in
   `schema-catalog.json`. Otherwise old values of those columns are sent to the provider
   again. 1.30.0 leaves masked columns out of both at every use.
7. Remove masked column names from `system-prompt.txt`, descriptions and metric definitions.
   That text is sent with every request. The demo context of 1.29.2 has this defect. From
   1.30.0 `npm run check-context` fails where that text writes a masked `table.column` and
   warns where it contains the column's words; it cannot see a paraphrase.
8. Views: a view that selects a masked column is not covered by the mask or by a column grant
   on the base table. Mask the view's column too, or leave the view out of the catalog. From
   1.31.0 a catalog view must carry `"reviewedView": true` before the check reports "masked by
   the database".

**What to look for.** In `rosie_audit_log.sql_generated`, on PostgreSQL: `SUBSTRING("`,
`POSITION("`, `LIKE "`, `ESCAPE "`, `~ "`, `AT TIME ZONE "`, `STRING_AGG(` with a
double-quoted second argument, and `IS DISTINCT FROM` followed by a masked column. On any
engine: a statement with two aliases that differ only by case, or with a non-ASCII letter in
an alias. In process logs: `execution_error`, `count_query_failed`, `export_error` lines
whose `error` quotes a value. In `rosie_audit_log.error_message` where
`status = 'export_error'`.

**What these searches cannot tell you.** The audit row holds the final statement of each
answer. Statements the model ran before it, whose rows also went to the provider, are not
recorded (from 1.30.0 the audit file lists, under `sql_trace`, the statements the model ran
with `execute_query`, up to 20 per question). A search that finds
nothing does not show that nothing happened. Upgrading does not undo anything already sent
or stored.

## Corrections

Defects in what this page promises, with the version that closed them. All were found by
independent reviews of the source, none from a report of use.

The searches below read the statement of each answer. Statements the model ran to explore,
whose rows also went to the provider, are not in `rosie_audit_log`; since 1.30.0 the audit
file lists, under `sql_trace`, the statements the model ran for a question with
`execute_query`, refused ones included, up to 20. A search that finds nothing does
not show that nothing happened.

The 1.29.2 release said its scope fixes were checked on SQL Server by the test battery. That
was not true: because of a defect in the test, the battery ran its 1.29.2 scope cases on the
MySQL dialect three times, and never on the PostgreSQL and SQL Server dialects. They were run
on those dialects later and hold. For 1.31.0 the masked-column shapes of the earlier reviews
were run against a live SQL Server 2022 as an account with column grants; that run found the
`PIVOT` row below.

1.31.0 also makes the masked-column check stricter, so a deployment that 1.30.0 reported as
"masked by the database" can now read "masked by Rosie only" (a catalog column the account
cannot read, a catalog view without `"reviewedView": true`, a view, function or procedure the
account can use that returns a masked column or that the check cannot read, a PostgreSQL
partition or inheritance relative with a broader grant, a MySQL histogram of a masked column,
a `PIVOT` on SQL Server). The rows below are what the work on 1.31.0 and an internal audit of
it found; an independent review of 1.31.0 has not happened yet.

| Closed in | What was wrong | Who could reach it | What to check if you ran it |
|---|---|---|---|
| 1.31.0 | PostgreSQL: a partition or a table created with `INHERITS` is a separate table with its own privileges, and a `SELECT` on a parent reads its children. 1.30.0 reported "masked by the database" while a partition or inheritance child of a masked table kept a table grant (a schema-wide grant gives one to every partition) and returned the masked column, and the migration script left that grant in place. The check now walks partitions, children and parents (`postgres_inheritance_reads_masked`), and `npm run grants` prints the `REVOKE` for each | Only a statement that also got past the validator, which refuses a table that is not in the catalog. None is known | As an administrator, list the partitions and inheritance children of each masked table (`\d+` in `psql`) and their privileges (`\dp`): none may grant `SELECT` on the table, or on a masked column, to the read-only role, a role it is a member of, or `PUBLIC` |
| 1.31.0 | MySQL: `information_schema.COLUMN_STATISTICS` is filtered by access to the table, not to the column, so a column-granted account reads the histogram of a masked column, real values included, whenever someone ran `ANALYZE TABLE ... UPDATE HISTOGRAM` on it. 1.30.0 reported "masked by the database" in that case. The check now reports such a column as readable (`mysql_histogram_reads_masked`, limit `mysql_column_statistics`), and `npm run grants` prints the `DROP HISTOGRAM` | Only a statement that also got past the validator, which refuses `information_schema`. None is known | As an administrator: `SELECT SCHEMA_NAME, TABLE_NAME, COLUMN_NAME FROM information_schema.COLUMN_STATISTICS` for the masked columns |
| 1.31.0 | Views and routines that run with their definer's rights return a masked column to any account allowed to use them, whatever its column grants. 1.30.0 checked only catalog views and reported "masked by the database" while the account could use a MySQL definer view or stored routine, or a PostgreSQL `SECURITY DEFINER` function (whose `EXECUTE` goes to `PUBLIC` by default), that returned one. The check now examines them (`view_reads_masked`, `object_not_verifiable`): a definer routine the account may execute is never passed on its text and holds the context until it is listed in `"reviewedRoutines"` in the catalog | Views: only a statement that also got past the validator (views outside the catalog are refused). Functions: see the next row. None is known | MySQL: `information_schema.VIEWS` and `information_schema.ROUTINES` with `SECURITY_TYPE = 'DEFINER'` that the read-only account can use. PostgreSQL: `SECURITY DEFINER` functions outside the system schemas that the role or `PUBLIC` may execute |
| 1.31.0 | MySQL: the validator had no function allowlist, so a statement could call any function by name, user-defined and schema-qualified ones included (`SELECT f(1) FROM t`, `SELECT db.f(1) FROM t`). A definer stored function the account may execute returns a masked column whatever the column grants, and with `ROSIE_REQUIRE_COLUMN_GRANTS` off nothing stopped it (found during the work on 1.31.0). MySQL now allowlists about 190 built-ins, as PostgreSQL and SQL Server already did, and a schema-qualified function call is refused on every engine | A statement written by the model (so a user able to steer it past the question filters), and directly `POST /certify` and `certify_sql` when `ROSIE_CERTIFY_SQL=true`, when the read-only account could execute such a function. None is known | MySQL: whether the read-only account holds `EXECUTE` on any routine (`SHOW GRANTS FOR` it; `GRANT EXECUTE` on a routine, a database or `*.*`, or `ALL PRIVILEGES`). Search `rosie_audit_log.sql_generated` for calls to functions that are not MySQL built-ins, and for a function name written as `name.name(`. The search cannot show statements the model ran before its answer |
| 1.31.0 | MySQL 8.4 does not check the column privilege of a join column named in `USING` or matched by `NATURAL JOIN`. 1.30.0 said that under "masked by the database" a statement reaching a masked column is refused by the database; on MySQL these joins run under column grants and can test a guessed value against a masked column, or compare two masked columns, without reading them. Rosie's validator refused both forms already (a masked `USING` column since 1.26.0), so it was the only layer. The check now carries the engine limit `mysql_join_using_natural`, and the documents say it | Only a statement that also got past the validator. None is known | MySQL installations: search `rosie_audit_log.sql_generated` for `USING (` followed by the name of a masked column, and for `NATURAL`. The search cannot show statements the model ran before its answer |
| 1.31.0 | SQL Server applies column permissions to the aggregated and the `FOR` column of a `PIVOT`, not to the columns it passes through: with `SELECT` on one column, a `PIVOT` returns every other column of the table, a column `DENY` included (SQL Server 2022 CU23). 1.30.0 did not claim "masked by the database" on SQL Server, and its validator refused every `PIVOT` tried, but only because the pivot's alias was unknown to it. `PIVOT` and `UNPIVOT` are now refused by name, the check probes `PIVOT` per masked column (`sqlserver_pivot_reads_masked`, limit `mssql_pivot`), and with column grants SQL Server reports "masked by Rosie only" | Only a statement that also got past the validator. None is known | SQL Server installations: search `rosie_audit_log.sql_generated` for `PIVOT`. The search cannot show statements the model ran before its answer |
| 1.31.0 | `NATURAL JOIN` was refused on MySQL and SQL Server only because the SQL parser failed on it, and on PostgreSQL by a text pattern. A parser that learned to read it would have let it through, and on MySQL the database does not stop it (row above). It is now refused on every engine by an explicit rule ("NATURAL JOIN is not allowed: join with ON and name the columns"). On PostgreSQL, a `USING` on a masked column inside a derived table or `WITH` body was refused for the wrong reason; it is now refused as a masked column at any depth | No one: every form was refused, for a fragile reason | Nothing to search |
| 1.31.0 | When SQL Server refused one statement on several columns at once, the driver raised an error with no number of its own. Rosie did not recognise it as a database error, so it logged and stored the error's message, which the query builder fills with the statement text (literal values included), at error level; `db_permission_denied` was not logged for it; and the masked-column check reported such a table as "could not be probed". It is now read as a database error: logged by class and codes (`database error (other, 230, 3 messages: ...)`), never with the statement | Anyone whose statement SQL Server refused on several columns; the log and the trail are read by operators | SQL Server installations: process-log lines with `"event":"execution_error"`, `count_query_failed`, `export_error`, `mcp_export_failed` or `explain_error` whose `error` holds a SQL statement, and `rosie_audit_log.error_message` where `status = 'export_error'`. Logs you did not keep cannot be searched |
| 1.31.0 | On MySQL with case-sensitive table names (`lower_case_table_names=0`, the Linux default), the validator compares table names without case and the server does not. A statement that wrote `customer` for `Customer` passed the validator and failed with error 1142 ("SELECT command denied") under table or column grants, which logged a `db_permission_denied` security event although no masked column was involved, and cost the model a step. Table names, qualifiers and `WITH` names are now rewritten to the catalog's spelling before a statement runs | No data was exposed; the security event was a false alarm | MySQL installations: before treating a `db_permission_denied` with code 1142 as an attempt on a masked column, compare the table spelling in the statement with the catalog |
| 1.30.0 | PostgreSQL: a double-quoted masked column in some positions (`SUBSTRING`, `POSITION`, `STRING_AGG`, `LIKE`, `ILIKE`, `SIMILAR TO`, `ESCAPE`, `~`, `~*`, `AT TIME ZONE`) was read by the validator as text and by the server as a column, so a statement could return a masked value or filter on one. The same on MySQL servers running with `ANSI_QUOTES`. Each MySQL read-only session now removes `ANSI_QUOTES`, `NO_BACKSLASH_ESCAPES`, `PIPES_AS_CONCAT` and `ANSI` from its own SQL mode and refuses to run if it cannot | A statement written by the model (so a user able to steer it past the question filters), and directly, with no model, `POST /certify` and the MCP tool `certify_sql` when `ROSIE_CERTIFY_SQL=true` | PostgreSQL, and MySQL with `ANSI_QUOTES`: search `rosie_audit_log.sql_generated` for `SUBSTRING("`, `POSITION("`, `LIKE "`, `ILIKE "`, `SIMILAR TO "`, `ESCAPE "`, `~ "`, `~* "`, `AT TIME ZONE "`, and `STRING_AGG(` with a double-quoted second argument. The search cannot show statements the model ran before its answer |
| 1.30.0 | PostgreSQL: the operand after `IS DISTINCT FROM` was not checked at all, bare, quoted or qualified. A statement could test whether a masked column holds a guessed value, row by row, or whether two columns are equal. MySQL and SQL Server refuse the form. Since 1.30.0 PostgreSQL refuses it too; write `=`, `<>`, `IS NULL` or `COALESCE` instead | Same as above | PostgreSQL: search `rosie_audit_log.sql_generated` for `IS DISTINCT FROM` followed by the name of a masked column. The search cannot show statements the model ran before its answer |
| 1.30.0 | Table aliases were compared after converting them to upper case, while the database compares them its own way: case-sensitive table names on MySQL (the Linux default), and on all engines letters such as `ı`, `ſ` and `ß`. Two aliases that differ only by case or by such a letter could make the validator check a column against an unmasked table of the statement while the database read the masked one. On PostgreSQL the statement as emitted could also move a name to another source. Two table names or aliases in one statement that differ only by letter case, by an accent or by a look-alike letter (`customer` and `Customer`, `ano` and `año`) are now refused, including one table written in two cases with no alias. Names in other alphabets or with accents are accepted when no other name in the statement folds to the same name (`vendas_mês`, `ação`, `"Top Artists"`) | Same as above, when another table of the catalog has an unmasked column with the masked column's name | Search `rosie_audit_log.sql_generated` for statements with two aliases that differ only by letter case, or with a non-ASCII letter in an alias. The search cannot show statements the model ran before its answer |
| 1.30.0 | When a mask file, or `maskedColumns` in the catalog, named one table under two spellings that differ only by case, only the first entry was applied: the columns of the other were readable and named in the prompt. `npm run init` and the server read the same file differently | Any question, in a context with such a duplicate. A context written only by `npm run init` was not affected | Whether any table appears twice, with different letter case, in `masked-columns.json` or in the catalog's `maskedColumns`. If one did, search `rosie_audit_log.sql_generated` for the columns of the entry that was ignored. The search cannot show statements the model ran before its answer, or what the prompt named |
| 1.30.0 | The text of a database error was written to the process log at error level, and to `rosie_audit_log.error_message` for failed exports. PostgreSQL, MySQL and SQL Server quote the failing value in conversion and time-zone errors, so a statement that reached a masked column (rows above) could leave its value there, even when the model was only told the query failed. Database errors are now logged by class and code; the text only at `LOG_LEVEL=debug` | Anyone who could make a statement fail on a value; the log and the trail are read by operators | Process logs: lines with `"event":"execution_error"`, `count_query_failed`, `export_error`, `mcp_export_failed`, `explain_error` or `cross_check_unavailable` whose `error` holds a value (`invalid input syntax for type`, `time zone "`, `Conversion failed when converting`, an `@`). `rosie_audit_log.error_message` where `status = 'export_error'`. Logs you did not keep cannot be searched |
| 1.30.0 | `npm run init`, run again after a column was masked, sent the provider sample values of that column cached from the earlier run (`.bootstrap/samples.json`), and its old `enumValues` (and a native `enum(...)` type) from the catalog | The operator running the wizard; the values went to the model provider configured for it, which had received them once before, with consent, before the mask | Whether you ran `npm run init` again after adding a mask, with `--samples` or with `enumValues` on that column. The retained records cannot show whether this happened |
| 1.30.0 | Metric definitions that name a masked column were sent in every prompt and returned by the MCP tool `list_metrics`. Text you write (the system prompt, descriptions, examples) that names a masked column is sent as written; the demo context of 1.29.2 did this. Names only, never values. Metrics over a masked column are now withheld with a warning, and `npm run check-context` fails where your text writes a masked `table.column` | Any question, and any MCP client allowed to list metrics | Whether `system-prompt.txt`, `metrics`, descriptions or examples in your context name a masked column; if they did, every request sent that name. The retained records cannot show it |
| 1.30.0 | A row policy could use a masked column, and the filtered read of a restricted table was `SELECT * FROM <table>`, which a read-only account without privilege on the masked columns cannot run. The compared value was always the caller's own verified claim, so no value could be read back. A policy over a masked column now stops the startup, and the filtered read names the unmasked columns | Operators whose `row-policies.json` names a masked column; for the `*`, every installation with row policies and column privileges | Whether any predicate in `row-policies.json` names a column of `masked-columns.json`. With column privileges: `rosie_audit_log` rows with `status = 'error'` on restricted tables |
| 1.30.0 | A follow-up with `previousAnswerHandle` (HTTP, or MCP `ask_data`) sent the earlier statement and question to the provider without validating them again, so a column masked after that answer was named again for up to 15 minutes. It is now dropped when it no longer passes | The caller who held the handle, within 15 minutes of the change | What was sent was a statement that caller had already been shown. The retained records cannot show whether this happened |
| 1.30.0 | `validate_query` returned the database's plan and its raw error text to the model, and `sample_rows` and `lookup_values` their raw error text. A plan names index names and, for a view, the columns and filters of its definition. The plan is no longer returned | A statement written by the model | The retained records cannot show whether this happened: tool results are not stored |
| 1.29.2 | A select-list alias with the name of a column made the validator skip that name everywhere in the statement, so `SELECT email AS email FROM users` returned a masked column, and `WHERE email LIKE 'a%' AND 1 = (SELECT 1 AS email)` filtered on one. The same skipped the column allowlist for a column that is in no catalog. All three databases | A statement written by the model (so a user able to steer it past the question filters), and directly, with no model, `POST /certify` and the MCP tool `certify_sql` when `ROSIE_CERTIFY_SQL=true` | Search `rosie_audit_log.sql_generated` for ` AS ` followed by the name of a masked column |
| 1.29.2 | One alias used for two sources at different depths (`FROM users u, (SELECT 1 FROM c u) q`) made the validator check `u.email` against the inner source instead of `users`, and when that source was no catalog table nothing was checked. All three databases. Names are now resolved by scope, as the database does | A statement written by the model (so a user able to steer it past the question filters), and directly, with no model, `POST /certify` and the MCP tool `certify_sql` when `ROSIE_CERTIFY_SQL=true` | Search `rosie_audit_log.sql_generated` for statements where the same alias follows two different table names |
| 1.29.2 | Four more places where the validator took a name to mean one source and the database another, all through an alias used again in an inner query: a derived table reading the outer query when a neighbour in its FROM has the same alias; an `ON` clause naming a source joined after it or written before a comma; a join in parentheses (MySQL); and a later branch of a `UNION ALL`. Each could return, or filter on, a masked column of the outer table, most of them only when another table has an unmasked column of the same name. Joins in parentheses are now refused | A statement written by the model (so a user able to steer it past the question filters), and directly, with no model, `POST /certify` and the MCP tool `certify_sql` when `ROSIE_CERTIFY_SQL=true` | Search `rosie_audit_log.sql_generated` for statements where the same alias follows two different table names |
| 1.29.2 | On PostgreSQL a table alias written as a bare word is the whole row of the table. `SELECT status::text FROM users AS status WHERE 1 IN (SELECT 1 FROM orders)` returned every column of `users`, masked ones included, because `status` is a column of another table of the statement | A statement written by the model (so a user able to steer it past the question filters), and directly, with no model, `POST /certify` and the MCP tool `certify_sql` when `ROSIE_CERTIFY_SQL=true` | PostgreSQL installations: search `rosie_audit_log.sql_generated` for a table alias that also appears alone in the select list |
| 1.29.2 | A `WITH` name equal to a table name hid that table from the allowlist, the star rule and the mask, while the body of the `WITH` read the real table: `WITH users AS (SELECT * FROM users) SELECT * FROM users`. This also read a table that is in no catalog, if the read-only database user could reach it. MySQL and PostgreSQL directly; on SQL Server through a body that reads a name defined later in the list | A statement written by the model (so a user able to steer it past the question filters), and directly, with no model, `POST /certify` and the MCP tool `certify_sql` when `ROSIE_CERTIFY_SQL=true` | Search `rosie_audit_log.sql_generated` for `WITH` followed by the name of one of your tables and ` AS (`. Give the read-only user access only to the tables of the catalog |
| 1.29.2 | On SQL Server a double-quoted word was read by the validator as text and by the server as a column, so `SELECT "password" FROM users` passed every column check. Double-quoted words are now refused on SQL Server | A statement written by the model (so a user able to steer it past the question filters), and directly, with no model, `POST /certify` and the MCP tool `certify_sql` when `ROSIE_CERTIFY_SQL=true` | Search `rosie_audit_log.sql_generated` for a double quote, on SQL Server installations |
| 1.26.0 | `JOIN ... USING (column)` on a masked column passed the validator (MySQL and PostgreSQL; SQL Server has no `USING`). A statement joining a table to a derived table that renames another column to the masked name returned the masked values that matched, to the caller and to the model provider | A statement written by the model (so a user able to steer it past the question filters), and directly, with no model, `POST /certify` and the MCP tool `certify_sql` when `ROSIE_CERTIFY_SQL=true` | Search `rosie_audit_log.sql_generated` for `USING (` followed by the name of a masked column |
| 1.26.0 | When a table carried a `maskedColumns` list of its own in `schema-catalog.json`, the columns listed for that table in `masked-columns.json` (and in the catalog's root `maskedColumns`) were ignored: they were named in the prompt and could be queried | Any question, in a context that masks the same table in two places. A context that uses only `masked-columns.json`, which is what `npm run init` writes, was not affected | Whether any table of your `schema-catalog.json` has its own `maskedColumns` |
| 1.26.0 | A relationship that ends on a masked column, and a masked primary key, were named to the model (names, never values) | Any question | The retained records cannot show whether this happened |
| 1.26.0 | `/explain` and the MCP tool `explain_answer` sent the statement of an earlier answer to the provider without validating it again, so a column masked after the answer could still be named for up to 15 minutes | A caller holding an answer handle signed before the change | The retained records cannot show whether this happened |
| 1.24.0 | `describe_table` returned the name, type and description of a masked column | Any question | The retained records cannot show whether this happened |

## Not built, stated plainly

- **No certification.** There is no SOC 2 or ISO 27001 report, and no penetration test by a
  third party yet. The security suite and its results are available on request. No
  independent review of 1.31.0 has happened yet.
- **The column check stops only when you ask it to.** By default, when the context is not
  "masked by the database", Rosie logs a warning, reports it, and keeps answering. With
  `ROSIE_REQUIRE_COLUMN_GRANTS=true` it refuses every request of that context with `503`
  until the check passes ([INSTALL.md](INSTALL.md#requiring-the-database-grant-rosie_require_column_grants)).
  The server never issues `GRANT` or `REVOKE`; `npm run grants` prints them for your
  administrator, and `npm run init -- --create-ro-user` grants once, when you ask it to.
- **What the database layer does not cover.** On MySQL, joins `USING` a masked column,
  `NATURAL JOIN`, and a histogram created between two checks (refused or reported by Rosie
  only). On SQL Server, `PIVOT` under column grants (refused by Rosie's validator only). On
  every engine, what a view marked `reviewedView` really exposes (the field is your
  statement, not a check), objects in schemas the account cannot see, anything created
  between two checks (the 15-minute interval is the detection lag), copies of masked values,
  and what the unmasked columns reveal.
- **Reviewed views and routines are your statement.** A MySQL definer routine or a
  PostgreSQL `SECURITY DEFINER` function the account may execute keeps the context out of
  "masked by the database" (`object_not_verifiable`) until it is listed in
  `"reviewedRoutines"` in the catalog, as a view needs `"reviewedView": true`. Rosie does not
  check what a listed routine or a reviewed view returns, beyond refusing one whose text
  visibly names a masked column. A `PUBLIC`-executable `SECURITY DEFINER` function, common on
  managed PostgreSQL platforms, holds every context until it is listed or its `EXECUTE` is
  taken away.
- **MariaDB is not supported.**
- **Shared rate limits are opt-in.** With `ROSIE_RATE_LIMIT_STORE=database` the hourly and
  daily quotas of the HTTP routes and of the MCP transport are one number for every replica,
  kept in the engine database. Without it each replica counts separately. The concurrency
  limit is per process in both cases.
- **No licence key and no seat enforcement.** Use is governed by the agreement.
- **No built-in secret manager integration.** Secrets are environment variables; inject them
  from your own secret store.
- **The audit file is not rotated by Rosie.** Rotate it with your log tooling; any mode works, including rename-and-compress. There is no tool yet to load the file back into the database.
- **The certificate checks structure, not meaning.** It can show that a join duplicated rows
  or a filter matched nothing. It cannot show that the question was understood. Approved
  queries and declared metrics are the controls for meaning, because a person reviewed them.

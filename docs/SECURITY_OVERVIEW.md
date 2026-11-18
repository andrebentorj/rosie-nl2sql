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
| Which tables and columns can it read? | Only those in the catalog file. A statement that names anything else is rejected before it runs. Leaving a table out of the catalog keeps it out of reach. In express mode that is Rosie's validator: the database account itself can read the whole schema. If that schema or database also holds Rosie's own tables (`rosie_audit_log` and the others, with every user's questions and answers), the account can read them in the database; the validator refuses them, the database does not. Keep them in a separate database (`DB_ENGINE_DATABASE`, [INSTALL.md](INSTALL.md), Step 3), or use strict mode. In strict mode the account is also granted table by table |
| How is personal data protected? | It depends on the masking mode, `ROSIE_MASKING_MODE` ([INSTALL.md](INSTALL.md#choosing-a-masking-mode)). **Express, the default:** Rosie refuses statements that reach a masked column and leaves masked columns out of what it describes to the model (they are not sampled, looked up or described). This is Rosie's own reading of SQL; independent reviews have found ways around it before (see "Corrections" and the advisory below), and the database account can read those columns. **Strict:** two layers. Rosie, as in express. The database: the read-only account has no privilege on the masked columns ([INSTALL.md](INSTALL.md), Step 2; `npm run grants` prints the statements). In both modes Rosie checks the account at startup and every 15 minutes and reports per context "masked by the database" or "masked by Rosie only", with reason codes ([How to read the check](INSTALL.md#check-what-the-read-only-account-can-read)). In strict mode it refuses every request of a context that is not "masked by the database" or "no masked columns" (`ROSIE_REQUIRE_COLUMN_GRANTS` defaults to `true` there); in express mode it answers either way (the default is `false`). An explicit `ROSIE_REQUIRE_COLUMN_GRANTS` wins over the mode's default. What the database layer of strict mode can give is this: the database denies this account every masked column Rosie knows how to reach, on these engines, with named gaps. It is not that the database refuses every statement that reaches a masked column. The check is an allowlist: anything the account can read or execute beyond the catalog tables and the reviewed objects holds the context; within what the check enumerates, an object it does not recognise is held, not passed, and the paths that are not objects are named gaps. The review markers (`reviewedView`, `"reviewedViews"`, `"reviewedRoutines"`) are the operator's statement. Each sentence is about the read-only account, when the check passes, as of its last run (default every 15 minutes; with the gate on a passing result counts for 35 minutes by default, `ROSIE_COLUMN_GRANTS_MAX_AGE_MS`). PostgreSQL: the database refuses a statement that names a masked column of a catalog table, for the account and every role it is a member of, and through the table's partitions, inheritance children and parents; anything else the account can read or execute (a table, view, materialized view or foreign table outside the catalog, a definer, untrusted-language or specially granted routine such as `pg_read_binary_file`, a foreign server with a mapping, large objects) holds the context until it is reviewed or taken away. MySQL: the database refuses a statement that reads a masked column by name; the check also reports histograms on masked columns, and holds the context while the account can read a table or definer view outside the catalog, in any database, or execute a definer routine, until each is reviewed or taken away. The database does not stop a `USING` or `NATURAL` join on a masked column, a histogram created between checks, or a definer routine; Rosie's validator refuses those (any function that is not a listed built-in included) and is a required layer there, so give the account no `EXECUTE` on routines. SQL Server: with column grants, no database claim (a `PIVOT` reads the masked columns of any table the account can read one column of); with no permission on the masked tables and a reviewed catalog view, the database refuses a statement on those tables, and what the view shows is the operator's statement; every table, view, synonym, function and procedure outside the catalog that the login can use, every linked server with data access and every other user database it can open holds the context until it is reviewed or taken away. It is the weakest of the three claims; no outside reviewer has reproduced it. On PostgreSQL and MySQL, a definer routine or function the account may execute keeps the context out of "masked by the database" until it is listed in `"reviewedRoutines"` in the catalog, the operator's statement that it returns no masked value. Outside both layers on every engine: what a reviewed view, foreign table or routine really exposes (including one that uses `*`, a whole-row reference or dynamic SQL; `npm run grants -- --verify` warns about those), paths that are not objects the check enumerates (PostgreSQL `LISTEN`/`NOTIFY` payloads; statement text of other sessions in `performance_schema`, `pg_stat_activity` or `sys.dm_exec_*` for an account allowed to read it; MySQL `CHECKSUM TABLE`; the history of a SQL Server temporal catalog table; on SQL Server an object an administrator creates in `dbo` with one of the exact diagram names and types and flags as shipped (`sp_MS_marksystemobject`); on SQL Server `VIEW CHANGE TRACKING`, with which `CHANGETABLE` returns the primary key values of a tracked table (a permission, untested); other PostgreSQL databases on the same server; a table added to an extension with `ALTER EXTENSION ... ADD TABLE`, and any view or materialized view of that extension over it (an administrator act that cannot be told apart from what the extension ships, such as PostGIS's `spatial_ref_sys`); a catalog table that is a MySQL `FEDERATED` table), objects the account cannot see, anything created between two checks, copies in other columns, and what your unmasked columns reveal |
| Can a user see another tenant's rows? | Not with [row policies](ROW_POLICIES.md): a filter derived from the caller's verified identity is added to every read, including approved queries and compiled metrics. The filtered read names the table's unmasked columns, never `*`, so it also runs under column grants. A policy may not use a masked column |
| What stops a runaway query? | A statement timeout enforced on the connection (default 10 s), a cap of 500 rows per answer, at most 5 queries per question, and a response size limit |
| What about SQL injection? | The user never supplies SQL; a question that starts with a SQL statement is refused. The model's SQL is treated as untrusted input: parsed to a syntax tree, checked against the allowlist, with set operations (all but `UNION ALL` inside a derived table, where each branch is validated on its own and filtered by the row policies), system schemas, file access, session variables and information functions rejected. Values in approved queries are typed and bound, never concatenated |
| And prompt injection? | A filter refuses known injection patterns before the model is called. Data returned by tools is marked as data for the model. Neither is the control that matters: whatever the model is talked into writing still has to pass the validator and run as a `SELECT`-only user |
| Is the connection encrypted? | With `DB_SSL=true`, TLS with certificate verification on MySQL, PostgreSQL and SQL Server. Since 1.32.0 a value other than `true`, `require`, `verify`, `no-verify` or `false` (any case) stops the server, where before a value such as `yes` silently meant no TLS |

The validator has its own regression suite of adversarial statements (more than 850 cases),
run on every commit and before every release. `npm run test:security` prints the count. A
suite checks the shapes someone thought of; independent reviews have found new ones after
it passed, which is why the database grant of strict mode matters.

**Upgrading to 1.32.1.** From 1.32.0, nothing to change. Two behaviours tighten: when the
configuration cannot be read, the masked-column gate now stays on (it switched off before),
and on SQL Server the strict check now lists every readable object type except those that
cannot return data, so an external table or a Service Broker queue the account can read
holds a strict context until it is reviewed (`"reviewedViews"`) or taken away. Run
`npm run grants -- --verify` after upgrading a SQL Server installation. On PostgreSQL the
wizard now revokes `CONNECT` from `PUBLIC` on the engine database it creates; a database it
created with 1.32.0 keeps PUBLIC's `CONNECT` until you run
`REVOKE CONNECT ON DATABASE rosie FROM PUBLIC; GRANT CONNECT ON DATABASE rosie TO <main account>;`.

**Upgrading to 1.32.0.** An installation upgrading from 1.30 or 1.31 without
`ROSIE_MASKING_MODE` runs in express mode. It logs fewer warnings and has the same protection
as before: the check still runs, and the gate was off by default. The default of
`ROSIE_REQUIRE_COLUMN_GRANTS` follows the mode (`false` in express, `true` in strict), and an
explicit value always wins: an installation that set `ROSIE_REQUIRE_COLUMN_GRANTS=true` keeps
its gate. An installation whose account already has table and column grants can set
`ROSIE_MASKING_MODE=strict` to keep the per-column warnings and get the gate by default
([INSTALL.md](INSTALL.md#upgrading)). The startup event `column_grants_required` is now logged
at every start, not only when the gate is on, so a log alert keyed on its presence fires every
time; key it on its `required` field. Also new in 1.32.0, and noticeable on upgrade: a
malformed mask file stops the server (`E_MASK_CONFIG_INVALID`; run `npm run check-context`
before upgrading); in strict mode more objects are held until reviewed (PostgreSQL views and
materialized views over masked tables, foreign tables and foreign servers with a user
mapping, MySQL invoker views inside definer views, SQL Server procedures); and with the gate on
a passing result expires after 35 minutes by default and the read-only pool is replaced after
every passing check ([INSTALL.md](INSTALL.md#upgrading)). Since the check became an
allowlist, every table, view, synonym or routine the account can use beyond the catalog
tables holds a strict context until it is reviewed or taken away, and an unreadable value of
a security setting (`ROSIE_REQUIRE_COLUMN_GRANTS`, `ROSIE_AUDIT_POLICY`, `DB_SSL`,
`ROSIE_AUDIT_FSYNC`, ...) stops the server.

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
statement, and, for what it misses, in strict mode, when the read-only account has no
privilege on them (see "How is personal data protected?"). In express mode the account can
read them, and the validator is the only layer. Pointing `AI_BASE_URL`
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
| Retention? | You set it: `ROSIE_AUDIT_RETENTION_DAYS` (0, the default, keeps everything). `npm run audit-prune` reports what is older than the period, and with `--confirm` and an administrator connection removes it from the three log tables (`rosie_audit_log`, `rosie_answer_certificate`, `rosie_feedback`) and from the audit file, and records the run. Nothing is removed unless you schedule that command. The server's account cannot remove anything, by design (`docs/INSTALL.md`, Step 3), so the command refuses it. The fourth table, `rosie_rate_limit`, holds client addresses for about two windows and is cleaned by the engine |
| Can a person's records be exported or erased? | Yes, since 1.34.0: `npm run subject -- --user <id>` with `--export` (one JSON file) or `--erase` (pseudonymise, or `--delete`), across the tables and the audit file, recorded without the person. Backups, your log platform and the model provider's records are outside its reach ([COMPLIANCE.md](COMPLIANCE.md)) |
| Does the trail say which answers returned personal data? | Since 1.34.0 each answer's row lists the catalog columns it returned and whether one is marked `"personal": true` in the catalog, and carries the context version and the masking state in force. `npm run audit-export` writes an evidence bundle for a period with a SHA-256 manifest ([COMPLIANCE.md](COMPLIANCE.md)) |
| What is in application logs? | Structured events with identifiers and reasons. Privilege details stay in the server log and are not returned to callers. A database error is logged by its class and code; its text, which can quote a stored value, is logged only at `LOG_LEVEL=debug` (since 1.30.0; before, at error level) |

A nightly run from the host's cron, with the Docker Compose kit in `/opt/rosie`
(`deploy/README.md`, "Compliance and other operator commands"). The URL file
`/opt/rosie/admin-url` is owned by root with mode 600 and lies outside every folder or volume
mounted into the containers, so the running service cannot read it. Where the engine allows it,
the URL is a prune account rather than a full administrator: SELECT and DELETE on
`rosie_audit_log`, `rosie_answer_certificate` and `rosie_feedback`, INSERT on `rosie_audit_log`
for the record of the run (on PostgreSQL also USAGE on `rosie_audit_log_id_seq`), nothing else, and never the server's own account (the command
refuses `DB_USER`).

```cron
15 3 * * *  cd /opt/rosie && ROSIE_ADMIN_DB_URL="$(cat /opt/rosie/admin-url)" docker compose run --rm -T tools audit-prune -- --confirm --by cron >> /var/log/rosie-prune.log 2>&1
```

The GDPR, UK GDPR, CCPA, ISO 27001 and SOC 2 mappings, the data map and the data subject
procedures are in [COMPLIANCE.md](COMPLIANCE.md).

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

Since 1.32.0 the default masking mode is express, in which the database account can read
the masked columns; to keep the privileges taken away and verified, set
`ROSIE_MASKING_MODE=strict` ([INSTALL.md](INSTALL.md#choosing-a-masking-mode)).

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
with `execute_query`, up to 20 per question; since 1.33.0 each entry says whether it `ran`,
with its row count, was `refused` by the validator or a row policy, with the reason, or
`failed` in the database, and a refusal also sets the row's `rejection_reason` to
`statement_refused:<reasons>`). A search that finds
nothing does not show that nothing happened. Upgrading does not undo anything already sent
or stored.

**Refusals in the trail (since 1.33.0).** Every refusal leaves a record: the pre-model
refusals of the engine and of the route (injection filter, over-long or too-short question,
raw SQL, write or credential request, masked column, provider down) as `status=blocked` with
the reason, and requests turned away before they were read (bad token, rate limit, no free
slot, security or column-grant hold) as `status=refused`, with the person, the address and
the correlation id: the first of a minute per route, reason, address and person as its own
row, and when the minute ends one row with how many more there were
(`rosie_refused_total` counts every one). An unauthenticated caller can now cause these rows
(one a minute per address). Before 1.33.0 the
route's injection filter and length check, the raw-SQL and too-short refusals and every
`401`, `429` and `503` left no audit row. Exports, explanations and ratings are recorded
under the person who asked, carried in the answer's signed handle; before 1.33.0, with the
static token, they were recorded as `authenticated`. A `POST /admin/reload` is recorded
with who did it, like the console's changes. On PostgreSQL the engine tables' times were
stored in the host's local time on a host not on UTC; since 1.33.0 they are written in UTC.

## Corrections

Defects in what this page promises, with the version that closed them. All were found by
independent reviews of the source, none from a report of use.

The searches below read the statement of each answer. Statements the model ran to explore,
whose rows also went to the provider, are not in `rosie_audit_log`; since 1.30.0 the audit
file lists, under `sql_trace`, the statements the model ran for a question with
`execute_query`, refused ones included (marked `refused` since 1.33.0), up to 20. A search
that finds nothing does not show that nothing happened.

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
a `PIVOT` on SQL Server). The 1.31.0 rows are what the work on 1.31.0 and an internal audit
of it found.

**If you ran 1.31.0 in strict form** (column grants, with or without
`ROSIE_REQUIRE_COLUMN_GRANTS`): two independent reviews of 1.31.0 (the 1.32.0 rows) showed that
its "masked by the database" did not cover every readable path. On PostgreSQL a view or
materialized view taking a whole row of a masked table, a foreign table, a foreign server
reachable through `dblink` and a view over an inheritance relative were not seen; on MySQL an
invoker view inside a definer view; on SQL Server a procedure running dynamic SQL. An
internal audit of the 1.32.0 work found four more of the same kind (a materialized view
filled through a function and a sibling view over a catalog view's base table on PostgreSQL,
a synonym and a linked server on SQL Server), plus file functions granted on PostgreSQL. That
is why 1.32.0 turns the check into an allowlist. The gate could also keep serving on an old
passing result while a check stalled, and a gate value written as `TRUE` or `yes` turned it
off. Through Rosie each
needed a statement that also got past the validator, which refuses objects outside the
catalog; anyone holding the read-only account's credentials could use them directly. Upgrade
to 1.32.0, run `npm run check-context`, then `npm run grants -- --verify`, and read what it now
holds. In 1.32.0 the claim is narrower: the database denies this account every masked column Rosie knows how to reach, on these engines, with named gaps.

| Closed in | What was wrong | Who could reach it | What to check if you ran it |
|---|---|---|---|
| 1.32.1 | The masked-column gate failed open: when the configuration could not be read, `ROSIE_REQUIRE_COLUMN_GRANTS` read as off while the masking mode read as strict, so a strict installation served requests unchecked exactly when its settings were unreadable. An unreadable configuration now keeps the gate on, and the startup line says so. Found by the lead while writing the external review brief | Anyone sending requests while the configuration could not be read; the front doors refuse to start on an invalid value, so this needed a configuration that broke after start. None is known | The `column_grants_required` startup line: `required` should be `true` in strict mode |
| 1.32.1 | SQL Server: the strict check listed only the object types it named (tables, views, synonyms, functions, procedures, ...), so a readable type it did not name, such as an external table or a Service Broker queue, was not reported. It now lists every type except those that cannot return data (constraints, defaults, rules, triggers, internal and system tables, table types, plan guides, sequences), and checks `RECEIVE` as well. Found by the lead while writing the external review brief | Anyone using the read-only login's credentials directly. Through Rosie, only a statement that also got past the validator, which refuses everything but SELECT on catalog tables. None is known | As an administrator: `SELECT SCHEMA_NAME(schema_id), name, type_desc FROM sys.objects WHERE type IN ('ET', 'SQ')`, and the permissions the read-only login holds on each |
| 1.32.0 | PostgreSQL: the catalog match ignored letter case, so a table `public.customer` beside the catalog's `"Customer"`, a copy with the masked column, counted as a catalog table and was not reported. Catalog names now match as each engine compares them (exactly on PostgreSQL, by `lower_case_table_names` on MySQL, by the database collation on SQL Server). Found before release by an internal audit of the allowlist | Anyone using the read-only account's credentials directly. Through Rosie, only a statement that also got past the validator. None is known | As an administrator, the tables the read-only role may select whose name equals a catalog table's name without regard to case |
| 1.32.0 | PostgreSQL: a view added to an extension (`ALTER EXTENSION ... ADD VIEW`) was skipped with the extension's own objects, and returned masked values. An extension's view or materialized view that reads outside the extension is now listed. A table added the same way stays a named gap | As above | `SELECT e.extname, c.oid::regclass FROM pg_depend d JOIN pg_extension e ON e.oid = d.refobjid JOIN pg_class c ON c.oid = d.objid WHERE d.classid = 'pg_class'::regclass AND d.deptype = 'e'`, compared with what each extension ships |
| 1.32.0 | SQL Server: the object list trusted `is_ms_shipped`, which `sp_MS_marksystemobject` sets on any object, so a view so flagged in a user schema returned masked values while the check reported "masked by the database". The flag is no longer trusted outside the `sys` and `INFORMATION_SCHEMA` schemas; only the database-diagram objects SQL Server creates in `dbo` are skipped when flagged, each by its exact name and type (before release, a view named `sp_helpdiagrams` and flagged was still skipped; that was fixed too). An object an administrator creates in `dbo` with one of the exact diagram names and types and flags as shipped (`sp_MS_marksystemobject`) remains a named gap | Anyone using the login's credentials directly. Through Rosie, only a statement that also got past the validator. None is known | `SELECT SCHEMA_NAME(schema_id), name, type FROM sys.objects WHERE is_ms_shipped = 1 AND SCHEMA_NAME(schema_id) NOT IN ('sys', 'INFORMATION_SCHEMA')` |
| 1.32.0 | MySQL, found before release and never shipped: an intermediate version of the allowlist skipped the whole `sys` schema, so a definer view an administrator created there returned masked values. Only `information_schema` and `performance_schema` are skipped wholesale; in `sys` only the table `sys_config` is skipped, and any `SQL SECURITY DEFINER` view or routine there holds the context, whatever its definer (the views and routines MySQL ships there run as `SQL SECURITY INVOKER` and are allowed like any invoker object) | No released version | Nothing to search |
| 1.32.0 | PostgreSQL: a materialized view filled through a function (`CREATE MATERIALIZED VIEW mv AS SELECT * FROM f()`, with `f` an ordinary invoker function in `sql` or `plpgsql`) is refreshed with its owner's rights, so it holds masked values at rest, and its only recorded dependency is on the function, not on the table. The check reported "masked by the database" while the account read the copy. Since 1.32.0 the check is an allowlist, and every readable relation outside the catalog holds the context until it is reviewed | Anyone using the read-only account's credentials directly. Through Rosie, only a statement that also got past the validator, which refuses a relation outside the catalog. None is known | As an administrator, list the materialized views the read-only role (or `PUBLIC`, or a role it is a member of) may select, and what each one's query calls |
| 1.32.0 | PostgreSQL: when a masked catalog table is itself a view over a base table, a second view outside the catalog over the same base table returned the masked column, and the check, which protected only the catalog table and its inheritance relatives, reported "masked by the database". Every readable relation outside the catalog now holds the context | As above | The views the read-only role may select that read the base tables of your catalog views |
| 1.32.0 | PostgreSQL: a direct `EXECUTE` grant on a file function such as `pg_read_binary_file` (`pg_read_file`, `pg_ls_dir`, `lo_import` and the like are revoked from `PUBLIC` by default) let the account read the table's data files, masked values included, and the check did not look at such grants. Routines granted beyond `PUBLIC` and untrusted-language functions now hold the context, and large objects the account may read too | Anyone using the account's credentials directly. Through Rosie, none: the validator allowlists functions. None is known | `SELECT p.oid::regprocedure FROM pg_proc p WHERE has_function_privilege('<role>', p.oid, 'EXECUTE') AND p.proname IN ('pg_read_file', 'pg_read_binary_file', 'pg_ls_dir', 'pg_stat_file', 'lo_import', 'lo_export')`, and the role's memberships in `pg_read_server_files` |
| 1.32.0 | SQL Server: a synonym (`CREATE SYNONYM s.cust FOR dbo.Customer`) with `SELECT` granted reaches the base table through the ownership chain, masked columns included, while the account is denied them on the table. The check did not list synonyms and reported "masked by the database". Every synonym the login can use is now held, and one for a masked catalog table is reported as reading it even when listed | Anyone using the login's credentials directly. Through Rosie, only a statement that also got past the validator, which refuses an object outside the catalog. None is known | `SELECT s.name, s.base_object_name FROM sys.synonyms s` and the permissions of the read-only user, its roles and `public` on each |
| 1.32.0 | SQL Server: a linked server with data access and a login mapping let the account read masked columns through `OPENQUERY`, a four-part name or a view over either, and the check, which did not look at `sys.servers`, reported "masked by the database". Every linked server with data access, and every other user database the login can open, now holds the context; they cannot be listed, only removed | Anyone using the login's credentials directly. Through Rosie, only a statement that also got past the validator, which refuses an object outside the catalog. None is known | `SELECT name, is_data_access_enabled FROM sys.servers WHERE is_linked = 1`, and `sp_helplinkedsrvlogin` as an administrator |
| 1.32.0 | Yes-or-no settings were read as `true` (some also `1`) and nothing else. `ROSIE_REQUIRE_COLUMN_GRANTS=TRUE` or `=yes` was an explicit "off", which wins over strict mode, so the gate served requests while the log quoted a value the operator never wrote; `ROSIE_AUDIT_POLICY=FAIL-CLOSED` read as fail-open; `DB_SSL=yes` or `=1` meant no TLS; `ROSIE_AUDIT_FSYNC=yes` meant no fsync; `AI_ANSWER_CHECKS=enabled` turned the certificate checks off in silence. They are now read without case (`true/1/yes/on`, `false/0/no/off`), and an unreadable value of a security setting stops the server | The operator's own configuration: every caller of an installation with such a value, while it ran | `env` of each Rosie process: any of `ROSIE_REQUIRE_COLUMN_GRANTS`, `ROSIE_AUDIT_POLICY`, `DB_SSL`, `ROSIE_AUDIT_FSYNC`, `ROSIE_CERTIFY_SQL`, `ROSIE_ADMIN_WRITE`, `ROSIE_MCP_HTTP_ENABLED`, `AI_UNION_ALL` or any yes-or-no `AI_*` setting written in another case or form. For `DB_SSL`, whether the database connections were encrypted (`pg_stat_ssl`, `SHOW SESSION STATUS LIKE 'Ssl_cipher'`, `sys.dm_exec_connections.encrypt_option`) |
| 1.32.0 | F1. PostgreSQL: a readable view or materialized view that takes a whole row of a masked table (`to_jsonb(c)`, `row_to_json(c)`, `hstore(c)`, a bare composite) beside one of its unmasked columns returned every masked column, while the check reported "masked by the database". PostgreSQL records such a view as depending only on the unmasked column, so the column-level dependency check could not see it. A materialized view of it is a masked copy at rest. Every readable view or materialized view that depends on a masked table, or a relative of one, is now held until reviewed (`object_not_verifiable`), whatever it selects | Anyone using the read-only account's credentials directly. Through Rosie, only a statement that also got past the validator, which refuses a view outside the catalog. None is known | As an administrator, list the views and materialized views that depend on a masked table and that the read-only role (or `PUBLIC`, or a role it is a member of) may select: `pg_depend` over `pg_rewrite` for the table's OID, then `has_any_column_privilege('<role>', <view oid>, 'SELECT')`, which also finds a view granted by column (`has_table_privilege` misses that). Look in their definitions for `to_jsonb`, `row_to_json`, `hstore`, `json_populate_record` and a bare table alias |
| 1.32.0 | F2. PostgreSQL: a foreign table (`postgres_fdw`) mapped to a masked table reads it with the credentials of its user mapping, so the column grants do not apply; the check did not look at foreign tables. Every foreign table the account can read is now held until reviewed | As for F1 | `SELECT c.oid::regclass FROM pg_class c WHERE c.relkind = 'f' AND has_any_column_privilege('<role>', c.oid, 'SELECT')` (it also finds a foreign table granted by column, which `has_table_privilege` misses), and what each one maps to |
| 1.32.0 | PostgreSQL, found with F2: with `USAGE` on a foreign server that has a user mapping for the account, `dblink` runs any statement there as the mapped user, masked columns included. Such a server is now `object_not_verifiable` | Anyone using the read-only account's credentials directly, where the `dblink` extension is installed. Rosie's validator refuses the `dblink` functions (functions are allowlisted). None is known | `pg_user_mappings` for the read-only role, `PUBLIC` and the roles it is a member of, and `has_server_privilege('<role>', srvname, 'USAGE')` for each server |
| 1.32.0 | G01. PostgreSQL: a view over a partition or inheritance child of a masked table was not checked, because the check matched view dependencies only against the catalog's tables and walked relatives only where the account had a direct grant. The protected set now includes every relative, both ways, any depth | As for F1 | The views that depend on a partition, inheritance child or parent of a masked table and that the read-only role may select |
| 1.32.0 | G02. MySQL: an `SQL SECURITY INVOKER` view used inside an `SQL SECURITY DEFINER` view runs with the definer's rights, but the check skipped every invoker view, so a definer view over an invoker view over a masked table passed. Views are now read per execution identity, and one whose definition is hidden holds the context | As for F1 | `information_schema.VIEWS` for definer views the read-only account can use whose definition names an invoker view, and what that view selects |
| 1.32.0 | G03. SQL Server: a procedure that builds its statement at run time (`sp_executesql`, often `WITH EXECUTE AS OWNER`) has an empty dependency list, and the check took that as safe. Every procedure the account may execute, and every function whose behaviour cannot be established, now holds the context ("not verified on SQL Server") until it is listed in `"reviewedRoutines"` | Anyone using the read-only account's credentials directly. Through Rosie, none: the validator does not run procedures. None is known | `sys.database_permissions` for `EXECUTE` held by the read-only user, its roles or `public`, and the text of each procedure for dynamic SQL and `EXECUTE AS` |
| 1.32.0 | G04. A passing result had no expiry. With the gate on, a check that stalled (a hung connection) left the last passing result in service indefinitely, and later checks queued behind it, so a grant change was not detected. The passing result is now a lease (`ROSIE_COLUMN_GRANTS_MAX_AGE_MS`, 35 minutes by default), every check has a deadline, and periodic runs no longer pile up | No caller could cause it; it widened the window in which a broader grant went unnoticed | Gaps longer than the check interval between `column_grants_checked` lines in the server log of a strict deployment |
| 1.32.0 | G05. Reviewed routine names were compared as one dotted string, so `"a.b".c` and `a."b.c"`, two different routines, matched each other: reviewing one approved the other. Entries are now compared as (schema, name) pairs | Only where schema or routine names contain a dot, and through a statement that also got past the validator (which refuses functions outside its allowlist), or directly with the account's credentials. None is known | Whether any entry of `"reviewedRoutines"` has a dot inside a quoted part, and whether another routine has the same dotted spelling |
| 1.32.0 | G06. A malformed mask entry, such as `{"Customer": "Email"}` instead of a list, was ignored by every reader: nothing was masked for it, in the validator, the prompt or the grants, and the check reported "no masked columns", which the gate accepts. The mask configuration is now validated in one place, and a malformed one stops the server (`E_MASK_CONFIG_INVALID`) | Any question, in a context with such an entry: the column was not masked anywhere | Run `npm run check-context` on 1.32.0 ("Mask configuration"). If it fails, search `rosie_audit_log.sql_generated` for the columns of the bad entry. The search cannot show statements the model ran before its answer, or what the prompt named |
| 1.32.0 | G07. With the gate on, the read-only pool was replaced only when the result changed or after a reload, and a replacement that failed was ignored. On MySQL a connection opened before a `REVOKE` keeps the old privilege, so a grant taken away and given back between two checks, or a failed replacement, could leave old connections serving under a passing result. The pool is now replaced after every passing check, before the result is published, and a failed replacement makes the context "not checked" | A statement that also got past the validator, on such an old connection. None is known | Whether grants of the read-only account changed on MySQL while Rosie ran without a restart |
| 1.32.0 | F3. The reader of reviewed routines and views reported a masked column only when its name appeared as a word, so a reviewed routine or view using `SELECT *`, a whole-row reference (`RETURNS SETOF <table>`, `to_jsonb`) or dynamic SQL was accepted without a word of warning. The listing was and stays the operator's statement; `npm run grants -- --verify` now prints a warning for each such reviewed object | Whoever could call the reviewed object: a statement that also got past the validator, or the account's credentials directly. None is known | Read every entry of `"reviewedRoutines"`, and every reviewed view, for `*`, whole-row references and dynamic SQL; run `npm run grants -- --verify` on 1.32.0 and read its warnings |
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

- **No tamper evidence on the audit file.** The database copy is protected by the engine
  account's `INSERT`-only grant; the file is plain text the server's user can write. An
  evidence bundle (`npm run audit-export`) carries a SHA-256 manifest, which protects the
  export, not the live trail.
- **No certification.** There is no SOC 2 or ISO 27001 report, and no penetration test by a
  third party yet. The security suite and its results are available on request. Two
  independent reviews of 1.31.0 have been done (the 1.32.0 rows of "Corrections"); 1.32.0
  and 1.32.1 have not been reviewed independently.
- **Express mode has no database layer for masked columns.** It is the default since
  1.32.0. The read-only account can read the whole schema, masked columns included, and
  Rosie's validator is what refuses them. Use strict mode when that is not acceptable.
- **The column check stops requests only in strict mode, or when you ask it to.** In
  express mode, when the context is not "masked by the database", Rosie logs one info line,
  reports it, and keeps answering. In strict mode, or with `ROSIE_REQUIRE_COLUMN_GRANTS=true`,
  it refuses every request of that context with `503` until the check passes
  ([INSTALL.md](INSTALL.md#requiring-the-database-grant-rosie_require_column_grants)). An
  explicit `ROSIE_REQUIRE_COLUMN_GRANTS=false` turns the hold off in strict mode too.
  The server never issues `GRANT` or `REVOKE`; `npm run grants` prints them for your
  administrator, and `npm run init -- --create-ro-user` grants once, when you ask it to.
- **What the database layer of strict mode does not cover.** The check is an allowlist over
  the objects it enumerates (anything the account can read or execute beyond the catalog
  tables and the reviewed objects holds the context); paths that are not such objects are
  not covered. On PostgreSQL, `LISTEN`/`NOTIFY` payloads (a trigger that notifies row data
  delivers masked values to any listener), statement text of other sessions for a member of
  `pg_read_all_stats`, other databases on the same server, and a table added to an extension with `ALTER EXTENSION ... ADD TABLE`, and any view or materialized view of that extension over it (an administrator act that cannot be told apart from what the extension ships, such as PostGIS's `spatial_ref_sys`). On MySQL, joins `USING` a
  masked column, `NATURAL JOIN`, a histogram created between two checks (refused or
  reported by Rosie only), `CHECKSUM TABLE` (a weak equality test over the masked columns),
  statement text in `performance_schema` when the account may read it, and a catalog table
  that is a `FEDERATED` table. On SQL Server, `PIVOT` under column grants (refused by Rosie's
  validator only), statement text in `sys.dm_exec_*` with `VIEW SERVER STATE`, and the history
  of a temporal catalog table read through `FOR SYSTEM_TIME`, and an object an administrator creates in `dbo` with one of the exact diagram names and types and flags as shipped (`sp_MS_marksystemobject`), and `VIEW CHANGE TRACKING` on a change-tracked table, with which `CHANGETABLE` returns the primary key values of its changed rows (a permission, not an object the allowlist lists; untested). On every engine, what a reviewed view, foreign table or routine
  really exposes (the markers are your statement, not a check), objects in schemas the account
  cannot see, anything created between two checks (the 15-minute interval is the detection
  lag), copies of masked values, and what the unmasked columns reveal.
- **Reviewed objects are your statement.** Since 1.32.0 every object the account can read
  or execute beyond the catalog tables keeps the context out of "masked by the database"
  (`object_not_verifiable`) until it is listed (`"reviewedViews"` for tables, views,
  materialized views, foreign tables and synonyms, `"reviewedRoutines"` for routines; a
  catalog view needs `"reviewedView": true`) or its access is taken away. Rosie does not check what a listed object returns. It reports one whose
  text names a masked column as a word, and since 1.32.0 `npm run grants -- --verify` warns
  about a reviewed object that uses `*`, a whole-row reference or dynamic SQL, which can
  return masked values without naming them; a reviewed object whose dependencies show a
  masked read is reported (`reviewed: true`) and accepted, since a reviewed view may filter on
  a masked column without returning it. A `PUBLIC`-executable `SECURITY DEFINER` function, common on
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

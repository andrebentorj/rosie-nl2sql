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
| How is personal data protected? | Columns listed in `masked-columns.json` cannot be selected, filtered on, sampled or looked up, and are not described to the model. Since 1.24.0 that holds for every path: before it, the `describe_table` tool returned the name, type and description of a masked column to the model when the model asked for the table (never a value) |
| Can a user see another tenant's rows? | Not with [row policies](ROW_POLICIES.md): a filter derived from the caller's verified identity is added to every read, including approved queries and compiled metrics |
| What stops a runaway query? | A statement timeout enforced on the connection (default 10 s), a cap of 500 rows per answer, at most 5 queries per question, and a response size limit |
| What about SQL injection? | The user never supplies SQL; a question that starts with a SQL statement is refused. The model's SQL is treated as untrusted input: parsed to a syntax tree, checked against the allowlist, with set operations (all but `UNION ALL` inside a derived table, where each branch is validated on its own and filtered by the row policies), system schemas, file access, session variables and information functions rejected. Values in approved queries are typed and bound, never concatenated |
| And prompt injection? | A filter refuses known injection patterns before the model is called. Data returned by tools is marked as data for the model. Neither is the control that matters: whatever the model is talked into writing still has to pass the validator and run as a `SELECT`-only user |
| Is the connection encrypted? | With `DB_SSL=true`, TLS with certificate verification on MySQL, PostgreSQL and SQL Server |

The validator has its own regression suite of adversarial statements (more than 700 cases),
run on every commit and before every release. `npm run test:security` prints the count.

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
30 result rows per query so the model can write the answer. Masked columns and rows outside
the caller's policy are never part of it. Pointing `AI_BASE_URL`
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
| What is in application logs? | Structured events with identifiers and reasons. Database error text and privilege details stay in the server log and are not returned to callers |

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
| How do I revoke access? | Rotate `ROSIE_API_TOKEN`, or revoke the user at your identity provider. Rotating `ROSIE_HANDLE_SECRET` invalidates every outstanding handle. Both need a restart of every replica; the steps are in [OPERATIONS.md](OPERATIONS.md), "Rotating a secret". Remove the customer from the registry to stop new versions |
| How do I know answers are right? | The certificate shows what was checked on each answer and what was not ([CERTIFICATE.md](CERTIFICATE.md)). `npm run evaluate` measures accuracy on your own questions ([EVALUATION.md](EVALUATION.md)) |

## Vulnerabilities

Report privately to the vendor contact in your agreement. A confirmed finding in the
validator is fixed together with a regression case in the security suite. See `SECURITY.md`.

## Not built, stated plainly

- **No certification.** There is no SOC 2 or ISO 27001 report, and no penetration test by a
  third party yet. The security suite and its results are available on request.
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

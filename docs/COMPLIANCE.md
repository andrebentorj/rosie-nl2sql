# Compliance

For the security or compliance officer of a company that runs Rosie. It says what Rosie
stores, where, for how long and who can read it; who is responsible for what; how Rosie's
controls map to GDPR (and UK GDPR and CCPA), ISO 27001 and SOC 2; how to answer a data
subject's request and an auditor's request; and what Rosie does not provide.

Rosie has no certification of its own. The mappings below say which setting, command or
document implements a control, so that your own assessment can test it. They are not a claim
of compliance: that depends on how you deploy and operate it.

## In one paragraph

Rosie is software you run in your own infrastructure. People ask questions in plain
language; a language model writes a SQL statement; Rosie checks it and runs it on a read-only
connection to your database. Rosie keeps an audit trail of every request in your database and
in a local file. The only party outside your infrastructure is the model provider you
configure, which receives the question, a description of the schema and a sample of the
results ([DATA_PRIVACY.md](DATA_PRIVACY.md)). Nothing is sent to the vendor of Rosie.

## Roles

| Party | Role under GDPR | What it does |
|---|---|---|
| Your company | Controller | Decides why and how the data is processed: which database Rosie reads, who may ask, which columns are masked, how long the trail is kept. Answers data subject requests. |
| The model provider (the endpoint in `AI_BASE_URL`) | Processor engaged by you (Art. 28) | Receives what [DATA_PRIVACY.md](DATA_PRIVACY.md) lists, under your contract with it. Its retention, training use and region are set by that contract. |
| The vendor of Rosie | Not a processor of the data | Supplies the software. Rosie makes no call to the vendor: no telemetry, no licence check, no update check. The vendor has no access to your deployment unless you give it (for support, under your agreement). |
| Your hosting and database providers | Your processors, as for any other system | Hold the engine database, the audit file and the logs. |

Two kinds of people have personal data in Rosie: the **people who ask** (their user id,
address, questions and ratings are in the trail) and the **people in your business data**
(customers, employees: what an answer returns, and the sample sent to the model provider).

## Data map

What Rosie stores, by default. "Engine database" is `DB_ENGINE_DATABASE` (a database of its
own, recommended; [INSTALL.md](INSTALL.md), Step 3).

| Store | Personal data it holds | Where | Kept for (default) | Who can read it |
|---|---|---|---|---|
| `rosie_audit_log` | One row per request: user id, question (500 characters), the SQL (with its literal values), outcome and refusal reason, rows returned (a count), address and user agent. Since 1.34.0 also the catalog columns the answer returned and whether one is marked personal. | Engine database | Until removed: `ROSIE_AUDIT_RETENTION_DAYS` and `npm run audit-prune` | Your database administrators, and the server's engine account: `SELECT` and `INSERT` only when it is set up as [INSTALL.md](INSTALL.md) Step 3 describes (see "The engine account" below). The read-only account that runs questions must have no access (separate database). |
| `rosie_answer_certificate` | Per answer: user id, question, SQL, the checks, and a few values: the totals of the second derivation and, when a filter value was not found, up to eight real values of that unmasked column. | Engine database | Same | Same |
| `rosie_feedback` | Thumbs up or down, an optional free-text comment (500 characters), the question and SQL. | Engine database | Same | Same |
| `rosie_rate_limit` | Only with `ROSIE_RATE_LIMIT_STORE=database`: a counter per client address (IPv6 by /56) and hour or day. | Engine database | Removed by the server about two windows later (two days at most) | Same |
| `rosie_scheduled_run` | Only with `ROSIE_SCHEDULES=true` (1.36.0): one row per scheduled run: the schedule id, who defined it (a name from `scheduled-answers.json`), the service identity it ran as, the replica, status, attempts, a row count, the verdict, and per channel the number of recipients or the webhook host. No result rows, no e-mail address, no URL path. | Engine database | Until removed by hand (not covered by `npm run audit-prune` or `npm run retention` in this release) | Same |
| Recipients of scheduled answers | Only with `ROSIE_SCHEDULES=true`: **the result rows**, in the e-mail body (first 20, as text and as an HTML table), its CSV file (all, up to 500) and the webhook body (all), with the statement and its certificate. See [Outbound data](#outbound-data-scheduled-answers). | The recipients' mailboxes and every mail server on the way; the webhook receiver and whatever it forwards to | Outside Rosie: the recipients' and their systems' retention | The recipients, and anyone they forward to |
| Audit file (`ROSIE_AUDIT_FILE`) and your rotated copies | The same records as the three tables, plus: the whole question when longer than 500 characters, every statement the model tried for a question (`sql_trace`, refused ones marked), administration changes (the administrator's name or id and address), masking state changes, and the records of erasures, retention runs and exports. | The server's disk (a mounted volume in a container) | Until removed: `npm run audit-prune` removes old records from it too. Rotation is yours (logrotate or similar). | Whoever can read that file system. The server's own user can write it. |
| `review-dismissed.json` | Message ids of reviewed answers and the name of the administrator who looked. | Next to the audit file | Until removed | Same as the audit file |
| Process log (stdout) | Events with user ids and correlation ids. SQL text and database error text only at `LOG_LEVEL=debug`. | Wherever you ship stdout | Your log platform's retention | Your log platform's users |
| Model provider | The question, conversation history (last 6 turns), schema metadata of unmasked columns, up to 30 result rows per query (`AI_LLM_RESULT_MAX_ROWS`), sample and distinct values of unmasked columns when the model asks for them. Not the user id or address. | The provider's systems and region | The provider's terms | The provider |
| Results in the browser and CSV exports | What the answer returned. | The asker's browser and downloads | The widget keeps the conversation of the open tab only; files are the user's | The asker |
| Files these commands write | `npm run subject -- --export` and `npm run audit-export` write personal data, readable by their owner only. | Where you run them | Until you delete them | You |

Rosie never stores result rows. It does **send** them, when scheduled answers are on (the
last row of the table above). Backups of the engine database and of the audit file are
yours and follow your backup policy.

### What the trail can tell you about a person's data in answers

- **Masked columns** (`masked-columns.json`) are refused before anything runs and are never
  described to the model. With `ROSIE_MASKING_MODE=strict` the database itself denies them to
  the read-only account, and Rosie verifies that at startup and every 15 minutes.
- **Personal columns that are not masked** (a name, a city) can be marked in the catalog with
  `"personal": true` ([contexts/example/README.md](CONTEXT_DIRECTORY.md)). It changes
  nothing about what can be read. It lets the trail answer "which answers returned personal
  data, to whom": every audit row of an answer lists the catalog columns it returned
  (`columns_returned`) and says whether one of them is marked personal (`personal_data`: 1,
  0, or empty when it could not be determined). `npm run check-context` names columns whose
  names look personal and that are neither masked nor marked.
- `columns_returned` is Rosie's reading of the statement. A column used only to filter or
  sort is not listed (the statement is in the same row). When a name could not be followed
  to one table the list starts with `~` and may name more columns than were returned, never
  fewer of the names it saw, and `personal_data` is then 1 or empty, never 0.

```sql
-- Answers that returned personal data last month, by person
SELECT user_id, COUNT(*) AS answers, SUM(rows_returned) AS rows_returned
FROM rosie_audit_log
WHERE personal_data = 1 AND created_at >= '2026-10-01' AND created_at < '2026-11-01'
GROUP BY user_id;
```

### Who the "user" is

The user id is what your application sends with each request, or the subject of a verified
token when JWT verification is on ([IDENTITY.md](IDENTITY.md)). With the shared static token,
attribution is only as good as your application's identity: a caller holding the token can
send any id. Turn JWT verification on when the trail must prove who asked.

### What was in force when an answer was given

Since 1.34.0 each audit row carries `context_version` (a digest of the context files: catalog,
masks, prompt, approved queries; the same value `/metrics` shows as `rosie_context_info`) and
`masking_state`: the masking mode and the result of the last column check of that context
(`strict/database`, `express/rosie`, `express/unchecked`...). The certificate's `provenance`
carries both. Each change of the masking state is a `rosie_masking_state` record in the audit
file. `npm run audit-export` writes the context digest and every file's SHA-256, so a row's
version can be matched with the files that produced it.

## Outbound data: scheduled answers

Off by default. With `ROSIE_SCHEDULES=true` ([SCHEDULED_ANSWERS.md](SCHEDULED_ANSWERS.md)),
Rosie runs approved queries on a schedule and **sends their results out of the installation**:
by e-mail through your SMTP server, and to webhook URLs. This is a new data flow, separate from
the model provider, and it carries the result rows themselves, not a description of them.

- **What leaves:** the rows of the result (up to 500), the statement with its literal values,
  the certificate, the name of the schedule, the approved question, who approved it and who
  defined the schedule. Masked columns never leave: a statement that reads one is refused
  before it runs, as for any question. Row policies apply for the schedule's service identity,
  so a schedule sends only the rows its declared claims allow.
- **To whom:** the addresses and URLs written in `scheduled-answers.json` (URLs may be read from
  the environment). `ROSIE_SCHEDULE_EMAIL_DOMAINS` and `ROSIE_SCHEDULE_WEBHOOK_HOSTS` limit where
  they may point; set them. `npm run check-context -- --base <previous>` flags every schedule
  added or changed, with its recipients, for review before merge.
- **Who decides:** whoever can change the context directory, as for masks and row policies.
  Each schedule names who defined it (`definedBy`), and that name is recorded with every run.
  The statement is always an approved query, so a person reviewed it.
- **In transit:** SMTP with STARTTLS or TLS and a verified certificate (clear text only to a
  relay on the same machine); webhooks over https, signed with HMAC-SHA256. TLS covers the hop
  from Rosie to your SMTP server only; from there the message travels as your mail system sends
  it, and it is not encrypted end to end.
- **Once sent, it is out of reach.** An e-mail cannot be recalled, and erasure (`npm run
  subject`) does not reach mailboxes, mail servers or webhook receivers. If a person's data can
  appear in a scheduled result, include those recipients in your record of processing and in
  your answer to a data subject request.
- **Records:** every run is in the audit log (the answer, under `service:<name>`), the
  certificate table, `rosie_scheduled_run` and the audit file (record kind
  `rosie_scheduled_run`), with the outcome of each channel. The records name the number of
  recipients and the webhook host, never an address or a URL path.

## Data subject requests

All by the user id Rosie recorded. Run them where the server's `.env` is (they read it).

| Request | Command | What it does |
|---|---|---|
| Access (GDPR Art. 15, portability Art. 20, CCPA right to know) | `npm run subject -- --user <id> --export subject.json` | Every row with that user id in the three tables, the rate-limit counters of the addresses found in them, and the records in the audit file and its rotated copies (also `.gz`), as one JSON file readable by its owner only. Reads with the server's account. |
| See what is held first | `npm run subject -- --user <id>` | Counts only. |
| Erasure (GDPR Art. 17, CCPA right to delete) | `npm run subject -- --user <id> --erase --delete --admin-url <url> --reference <ticket> --by <your name>` | Removes the rows. In the audit file each record becomes a line with only its time and kind. Rate-limit counters of the person's addresses are removed. This is the command for a request that requires erasure. |
| Pseudonymisation, keeping the trail | the same without `--delete` | The user id becomes one random pseudonym (never stored next to the id); question, comment, address, user agent and session are emptied; the id is replaced where it stands as a whole value: a SQL string literal equal to it (row policies write it into the SQL) or a JSON string value equal to it in a certificate, never inside a number or a longer text. Times, outcomes, statements and certificates stay, so the trail still adds up. **This is not full erasure.** What remains: the statements with their other literal values (a person who asked "orders of ana@example.com" leaves that address in the SQL), the figures and values quoted in certificates, the conversation, message and correlation ids, the times, and one pseudonym shared by all of the person's records. Under GDPR that is still personal data when the person can be singled out again. Use it where you keep the trail on another legal basis (Art. 17(3)), not as the answer to an erasure request. |
| Check before changing | add `--dry-run` | Says what would change. |
| Rectification (Art. 16) | none | The trail records what happened. Correct the source (your application's user id) and, if needed, erase the old records. |
| Restriction or objection (Art. 18, 21) | none in Rosie | Stop sending the person's requests at your application, or take their access away at your identity provider. |

**The engine account.** Set up as [INSTALL.md](INSTALL.md) Step 3 describes (what
`npm run init -- --create-ro-user` creates since 1.33.0), the server's engine account has
`SELECT` and `INSERT` on the engine tables and nothing else, plus `UPDATE` and `DELETE` on
`rosie_rate_limit` (the wizard grants these always; only `ROSIE_RATE_LIMIT_STORE=database`
uses them). Then the process that writes the
trail cannot rewrite it. An installation made earlier may still run its main pool as the setup
account, which can; `npm run doctor` shows the engine account's privileges. Any holder of the
engine account can also insert rows, including rows with status `maintenance`: those records
are evidence together with the rest of the trail, not proof on their own.

**Erasure needs an administrator connection.** `npm run subject -- --erase` and `npm run audit-prune -- --confirm` therefore
take a connection URL for an administrator (or an account with `UPDATE` and `DELETE` on the
three tables): `--admin-url`, or `ROSIE_ADMIN_DB_URL` set for that command only, never in the
server's `.env`. They refuse the server's own accounts (`DB_USER`, `DB_RO_USER`), another
engine than `DB_DIALECT` and another database than the engine database (so an installation
whose `DB_USER` is still `root` must erase as another administrator). Erasure asks you to
type the user id again before it changes anything (`--yes` skips the question, for a script).
The database part runs in one transaction. The audit file is rewritten in place; records the
running server appends meanwhile are carried over. Running it while the server is stopped
removes even that window.

**The erasure is recorded, without the person.** A row in `rosie_audit_log` with status
`maintenance` and reason `erasure`, and a `rosie_maintenance` record in the audit file, with
the counts, the mode, your `--reference` (a ticket number) and who ran it: `--by`, or the
operating system user (`os:<name>`) when `--by` is not given. Access exports, evidence exports
and retention runs are recorded the same way (reasons `subject_access`, `audit_export`,
`retention`).

**A database that cannot be read stops the command.** When one of the three tables cannot be
read, `subject` exports and erases nothing and exits with code 1, and `audit-export` writes no
bundle and exits with code 1, so a scheduled job fails instead of passing with empty files.
`--allow-partial` writes what could be read, with a note.

**What these commands cannot reach**, which you handle by your own procedures:

- database backups, snapshots and replicas (erase from them, or let them expire on schedule);
- copies of the audit file outside its directory, and your log platform (stdout);
- the model provider's records of the requests (your contract with the provider);
- the person named inside other people's questions or results (only records attributed to the
  user id are found);
- placeholder ids shared by many requests (`authenticated`, `mcp-local`, `administrator`...) are refused unless
  `--force`, because erasing them changes other people's records.

## Retention

`ROSIE_AUDIT_RETENTION_DAYS` sets the period in days; 0, the default, keeps everything. The
server removes nothing by itself. `npm run audit-prune` reports what is older; with
`--confirm` and an administrator URL it removes those records from the three tables and from
the audit file and its rotated copies (`--tables-only` leaves the files), and records the run
(reason `retention`, the cutoff and the counts). The records of erasures, retention runs and
exports are kept: they hold no personal data and prove the procedure ran. Masking state records
are kept too, unless `--include-masking-states`: an evidence export reads the state in force
when its period began from them, and they name no person. Schedule it daily:

With the Docker kit (no Node.js on the host), in root's crontab, from the kit folder, with the
administrator URL in a file owned by root with mode 600 (`/opt/rosie/admin-url`, never in
`config/.env`):

```cron
15 3 * * *  cd /opt/rosie && ROSIE_ADMIN_DB_URL="$(cat /opt/rosie/admin-url)" docker compose run --rm -T tools audit-prune -- --confirm --by cron >> /var/log/rosie-prune.log 2>&1
```

From a checkout of the code, the same command is `npm run -s audit-prune -- --confirm --by cron`
with `ROSIE_ADMIN_DB_URL` set the same way.

**Running these commands with the Docker kit.** Every command on this page runs through the
kit's `tools` service: `docker compose run --rm tools subject -- --user <id>`,
`... tools audit-export -- --from ... --out /exports/<folder>` (files go to `./exports` next to
`compose.yaml`), `... tools audit-verify -- /exports/<folder>`. The administrator URL is read
from your shell only (`read -rs ROSIE_ADMIN_DB_URL && export ROSIE_ADMIN_DB_URL`, then
`unset` it). See [deploy/README.md, "Compliance and other operator commands"](../deploy/README.md#compliance-and-other-operator-commands).

`npm run retention -- --days N` (since 1.10.0) still removes table rows only, without a
record; prefer `audit-prune`.

## Evidence for an auditor

```bash
npm run audit-export -- --from 2026-10-01 --to 2026-10-31 --out ./evidence-2026-10
npm run audit-verify -- ./evidence-2026-10 --expect <the digest it printed>
```

The bundle holds, for the period: the rows of the three tables as CSV; the audit file records
(statements tried, administration changes, masking state changes, maintenance records and,
since 1.35.0, chat page sign-ins in `signins.ndjson`), with
the masking state in force when the period began; the settings in effect, secrets redacted;
the context digest with a SHA-256 of each context file and the lists of masked and personal
columns; a summary (outcomes, refusals by reason, statements refused inside the model loop,
certificate verdicts, exports and rows exported per user, answers that returned personal data
per user, sign-in events by kind); a README; and `MANIFEST.sha256`, a SHA-256 of every file (`shasum -a 256 -c` reads
it). The command prints the digest of the manifest itself: keep it outside the bundle (the
ticket, the email that sends it), because a manifest can be rewritten together with the files
it lists. `--user <id>` limits the bundle to one person. The export is itself recorded
(reason `audit_export`). The bundle holds personal data: its folder and files are readable by
their owner only.

## GDPR

| Article | What it asks | How Rosie supports it | Where |
|---|---|---|---|
| 5(1)(c) data minimisation | Only what is necessary | Masked columns are refused and never described to the model; only a sample of results (30 rows per query by default) is sent; result rows are never stored; `AI_LLM_RESULT_MAX_ROWS` sends less | `masked-columns.json`, [DATA_PRIVACY.md](DATA_PRIVACY.md) |
| 5(1)(e) storage limitation | Kept no longer than necessary | A retention period you set, applied by a scheduled command that also covers the audit file, and records each run | `ROSIE_AUDIT_RETENTION_DAYS`, `npm run audit-prune` |
| 5(1)(f), 32 security of processing | Confidentiality, integrity, availability | Read-only database account, SQL validator with a table and column allowlist, masking (in strict mode enforced by the database and verified), row policies per caller, bearer token or JWT, separate admin and metrics secrets, rate limits, an audit trail written before each answer with a fail-closed option | [SECURITY_OVERVIEW.md](SECURITY_OVERVIEW.md), [ROW_POLICIES.md](ROW_POLICIES.md), [IDENTITY.md](IDENTITY.md) |
| 5(2) accountability | Be able to show compliance | The audit trail, the certificate of each answer, the stamp of the context and masking state, the records of erasures and retention, and the evidence bundle | `npm run audit-export` |
| 15, 20 access, portability | Give the person their data | One command, one JSON file | `npm run subject -- --export` |
| 17 erasure | Erase without undue delay | Remove (`--delete`) or pseudonymise, recorded without the person. Pseudonymising keeps statements, certificate values, ids and a shared pseudonym, which can still be personal data: for an erasure request use `--delete`. The limits are listed above | `npm run subject -- --erase --delete` |
| 25 data protection by design and by default | Protective defaults | Read-only by construction (SELECT only, validated); masked columns never described; no result rows stored; nothing sent to the vendor; unauthenticated requests refused in production; the console read-only by default; question review off by default. Express masking mode is the default: for sensitive data choose strict mode | [INSTALL.md](INSTALL.md#choosing-a-masking-mode) |
| 28 processors | A contract with each processor | The model provider is your processor: use an API agreement that covers retention, training use and region. Rosie itself does not process data for the vendor | [DATA_PRIVACY.md](DATA_PRIVACY.md#what-the-provider-does-with-it) |
| 30 records of processing | Keep a record of processing activities | The data map above gives the categories, stores, recipients and retention for your record. Rosie does not write the record for you. `project.json` `governance` names the owner and purpose | This page, `project.json` |
| 33 breach notification | Detect and report breaches | The trail shows who asked what and what left (exports per user); refusals and masked-column attempts are recorded; `/metrics` and the log events support alerts. Notification is your procedure | [MONITORING.md](MONITORING.md) |
| 44 to 49 transfers, and data residency of the model call | Transfers outside the EEA need a basis | The model call goes to `AI_BASE_URL`. Choose a provider endpoint in your region with an agreement that covers transfers, or point `AI_BASE_URL` at a model you host, and nothing leaves your network. The engine database, the audit file and the logs stay where you deploy them | [DATA_PRIVACY.md](DATA_PRIVACY.md#running-with-nothing-leaving-your-network) |

**UK GDPR.** The same articles apply under the UK GDPR and the Data Protection Act 2018, with
the ICO as the authority. For the model call, check the UK transfer mechanism (the UK
addendum or IDTA) in your provider agreement.

**CCPA (California, as amended by the CPRA).** Your company is the business; the model
provider should be bound as a service provider or contractor. Rosie does not sell or share
personal information. The right to know and the right to delete map to `npm run subject`
(export and erase), and the trail records each request, so you can show the requests you
received and how you answered them. Sensitive personal information should be masked.

## ISO 27001 and SOC 2

ISO/IEC 27001:2022 Annex A controls and SOC 2 (2017 Trust Services Criteria, revised points of
focus 2022). Each row names what implements the control in Rosie; the policy, the review and
the evidence that it operates are yours.

| Control area | ISO 27001 | SOC 2 | Implemented by |
|---|---|---|---|
| Access to the data | A.5.15, A.8.3 | CC6.1, CC6.3 | A read-only account for questions (`DB_RO_USER`); a SQL validator with a catalog allowlist ([SECURITY_OVERVIEW.md](SECURITY_OVERVIEW.md)); row policies per caller ([ROW_POLICIES.md](ROW_POLICIES.md)); the privilege probe at startup and every 15 minutes, which holds every request if the read-only account can write |
| Authentication | A.5.17, A.8.5 | CC6.1 | `ROSIE_API_TOKEN` (constant-time compare) or JWT with asymmetric algorithms only ([IDENTITY.md](IDENTITY.md)); production refuses requests when no token is configured |
| Privileged access | A.8.2 | CC6.1, CC6.3 | Separate secrets for administration and metrics (`ROSIE_ADMIN_TOKEN`, `ROSIE_METRICS_TOKEN`) or an administrator role claim (`ROSIE_ADMIN_JWT_CLAIM`); the console read-only unless `ROSIE_ADMIN_WRITE=true`; the engine account limited to `SELECT` and `INSERT` when set up as INSTALL Step 3 describes (`npm run doctor` shows its privileges); removal of records only with an administrator connection (`--admin-url`) |
| Masking of personal data | A.8.11, A.5.34 | CC6.1, C1.1, P-series | `masked-columns.json`, `ROSIE_MASKING_MODE=strict` with database column grants (`npm run grants`), verified state in `/health` and in each audit row (`masking_state`); the `personal` flag and `columns_returned` |
| Logging | A.8.15 | CC7.2, CC7.3 | `rosie_audit_log` and the audit file, written before the answer (`ROSIE_AUDIT_FILE`, `ROSIE_AUDIT_FSYNC`, `ROSIE_AUDIT_POLICY=fail-closed`); every refusal recorded; administration changes recorded before they are made; erasures, retention and exports recorded |
| Monitoring | A.8.16 | CC7.2 | `/metrics` (Prometheus) and the suggested alerts, including `rosie_audit_sink_healthy` and the security hold ([MONITORING.md](MONITORING.md)); `/health` |
| Clock | A.8.17 | CC7.2 | Engine tables are written in UTC on every engine (PostgreSQL since 1.33.0); the audit file uses ISO 8601 UTC times. Host clock synchronisation is yours |
| Change management | A.8.32, A.8.9 | CC8.1 | The context directory under version control; `npm run check-context -- --base <previous copy>` shows what a change does before it ships ([OPERATIONS.md](OPERATIONS.md)); reloads are all or nothing and recorded; every record carries the context digest it ran under; `npm run doctor -- --settings` shows the configuration in effect |
| Encryption in transit | A.8.24 | CC6.7 | To the database: `DB_SSL=true` (verify the certificate). To the model provider: HTTPS endpoints. To callers: Rosie serves HTTP; terminate TLS at your reverse proxy or load balancer (`TRUST_PROXY`) |
| Secrets | A.5.17, A.8.24 | CC6.1 | Environment variables only, injected from your secret store; secrets are never printed (`doctor --settings` shows "set, N characters"), never in a prompt; the administrator URL of the compliance commands is passed to the command only. Rotation steps: [OPERATIONS.md](OPERATIONS.md), "Rotating a secret" |
| Deletion of information | A.8.10 | CC6.5, P4.3 | `npm run audit-prune`, `npm run subject -- --erase` |
| Supplier relationships | A.5.19 to A.5.22 | CC9.2 | The model provider is the one supplier in the data path; [DATA_PRIVACY.md](DATA_PRIVACY.md) lists what it receives, and `AI_BASE_URL` can point at a model you host |
| Vulnerability management and secure development | A.8.8, A.8.25 to A.8.29 | CC7.1, CC8.1 | `npm audit` at high severity blocks a release; the security suite (validator, masking, injection) runs on every release; findings fixed with a regression case ([SECURITY.md](../SECURITY.md)) |
| Software integrity | A.8.19 | CC7.1 | Release images signed with cosign and shipped with an SBOM ([SECURITY_OVERVIEW.md](SECURITY_OVERVIEW.md#verify-what-you-received)) |
| Evidence | A.5.28 | CC4.1 | `npm run audit-export` with a SHA-256 manifest, `npm run audit-verify` |
| Backup | A.8.13 | A1.2 | Not provided: back up the engine database and the audit file volume with your tools |

## Not provided

Stated plainly, so your assessment does not assume them:

- **No certification or attestation.** No SOC 2 report, no ISO 27001 certificate, no
  third-party penetration test yet ([SECURITY_OVERVIEW.md](SECURITY_OVERVIEW.md#not-built-stated-plainly)).
- **No tamper evidence on the audit file.** It is plain text the server's user can write; the
  database copy is protected by the engine account's grants (`INSERT`-only when set up as
  INSTALL Step 3 describes), not by signatures. The
  evidence bundle's manifest protects an export, not the live trail.
- **No encryption at rest by Rosie.** Use your database's and your disk's encryption.
- **No TLS listener.** Rosie serves HTTP; TLS is your proxy's.
- **No reach into backups, log platforms or the model provider** for erasure (listed above).
- **No search for a person mentioned in other people's questions or results.**
- **No recall of a scheduled answer.** What was sent by e-mail or webhook is outside Rosie
  (see [Outbound data](#outbound-data-scheduled-answers)); `rosie_scheduled_run` is not pruned
  by `npm run audit-prune` or `npm run retention` in this release.
- **No record of processing activities, DPA template or consent management.** The data map
  above is the input for your own record.
- **No activity screen.** Questions per person are answered with the trail (SQL, the evidence
  bundle, `npm run subject`), not in the admin console.
- **The audit file is not rotated by Rosie**, and masking state records exist only from 1.34.0;
  before that, only the startup log had them.
- **`columns_returned` is Rosie's reading of SQL**, not a database lineage. When it could not
  be exact it says so (`~`, `personal_data` empty), and it is never a reason to read more.

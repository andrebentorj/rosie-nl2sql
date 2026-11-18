# Scheduled answers

A scheduled answer is an [approved query](APPROVED_QUERIES.md) that Rosie runs on a schedule
and sends to people and systems: by e-mail, and to a webhook that Slack, Teams or an
automation can consume. Every delivery carries the Answer Certificate of that run: its verdict,
what was checked and what was not. A run that the certificate contradicts or could not check
is sent with that verdict first, never as a plain figure.

No language model is called for a scheduled run, at any point. The statement is the one a
person approved, run unchanged.

**Scheduled answers send data out of your installation.** The rows of the result leave in the
e-mail (its text and a CSV file) and in the webhook body. Turn the feature on only where that
flow is agreed, and read [COMPLIANCE.md](COMPLIANCE.md#outbound-data-scheduled-answers).

## In five steps

1. Approve the query in `verified-queries.json` (it needs an `id` and `approvedBy`,
   [APPROVED_QUERIES.md](APPROVED_QUERIES.md)).
2. Write `scheduled-answers.json` in the context directory (below).
3. Set the channels and turn schedules on in `.env`:

   ```bash
   ROSIE_SCHEDULES=true
   ROSIE_SMTP_HOST=smtp.example.com
   ROSIE_SMTP_USER=rosie@example.com
   ROSIE_SMTP_PASSWORD=...
   ROSIE_SMTP_FROM=Rosie <rosie@example.com>
   ROSIE_WEBHOOK_SECRET=...            # openssl rand -base64 48
   ROSIE_SCHEDULE_EMAIL_DOMAINS=example.com
   ```

4. Create the run table: run your engine's schema file on the engine database again
   (`db/schema.sql`, `db/schema.postgres.sql` or `db/schema.mssql.sql`; it only creates what is
   missing), and give the engine account `UPDATE` on `rosie_scheduled_run`
   ([INSTALL.md](INSTALL.md#step-3-create-the-engine-database-its-tables-and-the-main-user)).
5. Check, look, then send:

   ```bash
   npm run check-context                              # the file, without a database
   npm run doctor                                     # the SMTP server (nothing is sent) and the run table
   npm run schedule -- --list                         # each schedule and its next three runs
   npm run schedule -- --run-now weekly-usa --dry-run # run it and print the message; nothing sent
   npm run schedule -- --run-now weekly-usa           # run it and send it now
   ```

   Then restart, or reload the context (`docker kill --signal=HUP <container>` or the reload
   endpoint, [OPERATIONS.md](OPERATIONS.md)): the schedules are read again on every reload.

## The file

`scheduled-answers.json`, next to the other context files:

```json
{
    "schedules": [
        {
            "id": "weekly-usa",
            "name": "Weekly revenue, USA",
            "approvedQuery": "revenue-by-country",
            "params": { "country": "USA" },
            "cron": "0 8 * * MON",
            "timeZone": "America/New_York",
            "runAs": { "name": "finance-reports", "claims": { "tenant_id": "acme" } },
            "definedBy": "Dana Reyes, Finance",
            "definedOn": "2026-11-18",
            "channels": [
                { "type": "email", "to": [ "finance@example.com" ] },
                { "type": "webhook", "url": "env:FINANCE_WEBHOOK_URL" }
            ]
        }
    ]
}
```

| Field | Required | Meaning |
|---|---|---|
| `id` | yes | 2 to 64 characters: lower-case letters, digits, `-` or `_`. Unique in the file |
| `name` | yes | One line, at most 120 characters. The subject of the e-mail |
| `approvedQuery` | yes | The `id` of an approved query in `verified-queries.json` |
| `params` | when the query declares parameters | Fixed values for them, checked against their declared types |
| `cron` | yes | When it runs: five fields, below |
| `timeZone` | yes | An IANA time zone (`Europe/London`, `America/New_York`, `UTC`) in which `cron` is read |
| `runAs` | yes | The service identity it runs as: `name` and, when row policies apply, `claims` |
| `definedBy` | yes | Who defined the schedule. Recorded with every run |
| `definedOn` | no | `YYYY-MM-DD` |
| `channels` | yes | One to ten channels: `email` (`to`: up to 50 addresses, optional `subject`) or `webhook` (`url`) |
| `enabled` | no | `false` keeps the schedule in the file without running it |

Keys that start with `_` or `$` are comments. An unknown key is reported by `check-context`
with the closest known one.

### Why the file names an approved query and carries no SQL

The file says **when**, **as whom** and **to whom**. The statement is always an approved
query, referenced by id, and a `sql` key does nothing. This keeps one place where SQL is
reviewed, and everything that already guards an approved query guards the schedule too:

- the statement was reviewed and signed by a named person (`approvedBy`);
- it stops being approved after its `reviewBy` date, and the schedule stops with it, on the
  day, without a restart (the date is read at every run);
- it is suspended when the database no longer has a table or column it reads (schema drift,
  [APPROVED_QUERIES.md](APPROVED_QUERIES.md)), and a suspended query is not sent;
- it is dropped at load when it no longer passes the validator against the catalog, or when its
  text names a masked column.

A schedule therefore cannot bring a statement nobody reviewed into the installation.

### The service identity

`runAs` is a **service**, never a person: a name you give it (`finance-reports`) and the claims
the [row policies](ROW_POLICIES.md) read, declared in the file. A run is recorded as
`service:<name>` wherever a user id goes (the audit log, the certificate row, the run row). A
name that looks like an e-mail address is refused.

Claims take the values row policies accept: letters, digits, space and `_ . @ : + -`, at most
128 characters, or numbers, or lists of those. `check-context` applies the row policies to the
statement for these claims, without a database: a policy that needs a claim the service does
not declare fails the check, as it would fail the run.

Whoever can edit the context directory decides what the service may read, as they decide the
masks and the policies. Review a change to `runAs.claims` as you review a change to
`row-policies.json`; `npm run check-context -- --base <previous>` flags every schedule that is
added or changed, with its recipients. The parameter values and the claims are part of the
schedule, not of the approval: the person who approved the query did not choose them, so they
are reviewed with the schedule.

### When it runs

Five fields, as in crontab: minute, hour, day of the month, month, day of the week. Each takes
`*`, a number, a range `1-5`, a step `*/15` or `8-18/2`, and lists `1,15`. Months and days also
take their English three-letter names (`JAN`, `MON`); Sunday is `0` or `7`. `@hourly`,
`@daily`, `@weekly`, `@monthly` and `@yearly` are accepted.

| `cron` | Runs |
|---|---|
| `0 8 * * MON` | Mondays at 08:00 |
| `30 7 1 * *` | On the first of the month at 07:30 |
| `0 9 * * 1-5` | Weekdays at 09:00 |
| `*/15 8-18 * * 1-5` | Every quarter of an hour from 08:00 to 18:45 on weekdays |

The rules that matter:

- **The time is the wall-clock time of `timeZone`**, so `0 8 * * MON` in `America/New_York`
  is 08:00 in New York in winter and in summer. On the night the clocks go forward, a time
  that does not exist runs one hour later (02:30 becomes 03:30); on the night they go back, a
  time that happens twice runs once, the first time.
- When both the day of the month and the day of the week are given, a day matches when
  either does (crontab's rule).
- A day that never comes (`0 8 31 2 *`, 31 February) is refused by the loader and
  `check-context`, so the schedule does not sit in the file doing nothing.
- **At most every five minutes.** The minute field may match at most twelve minutes of the
  hour; `* * * * *` is refused.
- **A run missed while no replica was up** is still made at startup if it was due less than
  `ROSIE_SCHEDULE_CATCHUP_MINUTES` ago (30 by default; 0 never runs late), and only when the
  run table shows nobody made it. When several runs of one schedule were missed, only the
  latest is made.
- A schedule added or changed by a reload starts from the next due time: adding one at 08:10
  does not send the 08:00 run.
- At startup the clock waits for the first privilege check of the read-only account (and,
  with the column-grant gate on, the first masked-columns check), two minutes at most, so a run
  due at startup is not held for "not checked yet".
- When a run is due while the previous run of the same schedule is still retrying, the new one
  is skipped and logged (`scheduled_run_overlap`).

## What a run goes through

A scheduled run is the governed path an approved query takes from the API or MCP, with no
model before it:

1. **The claim.** The run's row is written in `rosie_scheduled_run`. If another replica wrote
   it first, this replica stops there ([Several replicas](#several-replicas)).
2. **The gates every question passes.** The assistant is on (`AI_ASSISTANT_ENABLED`), the
   read-only account passed its privilege check, the masked columns are confirmed denied by
   the database when `ROSIE_REQUIRE_COLUMN_GRANTS` is on (it is on in strict masking mode), and
   an audit sink can record when `ROSIE_AUDIT_POLICY=fail-closed`. A held gate is retried with
   backoff and never bypassed; a run held through every attempt is recorded as `held` and
   nothing is sent.
3. **The statement.** The approved query, bound with the schedule's parameter values, goes
   through the validator and its catalog allowlist, the masked columns, the row cap (500
   rows), the row policies for the service's claims and the read-only connection.
4. **The certificate.** The deterministic checks and the second derivation, as for any answer
   ([CERTIFICATE.md](CERTIFICATE.md)). The answer and its certificate are written to the audit
   log and the certificate table under `service:<name>`.
5. **The deliveries.** Each channel on its own, each with its own retries. A retry sends the
   same result again: the query is not run a second time for it.
6. **The record.** The run's row is completed (status, attempts, rows, verdict, the outcome of
   each channel), a `rosie_scheduled_run` record is written to the audit file, and a
   `scheduled_run` line to the log.

## What is sent

### The verdict, in every channel

| Verdict | Subject | First line of the e-mail | Webhook `caveat` |
|---|---|---|---|
| `CONTRADICTED` | `[CONTRADICTED] ...` | `!!! CONTRADICTED. This figure counts some records more than once. Do not use it as a fact.` and the recomputed figure | `true` |
| `FLAGGED` | `[FLAGGED] ...` | `!!! FLAGGED. A check raised a warning about this result. ...` | `true` |
| `NOT_CHECKED` | `[NOT CHECKED] ...` | `!!! NOT CHECKED. A check that was due could not run, ...` | `true` |
| no certificate (`AI_ANSWER_CHECKS=false`) | `[NOT CHECKED] ...` | `!!! NOT CHECKED. The checks ... are turned off on this installation ...` | `true` |
| any other verdict, including one a later release adds | `[<VERDICT>] ...` | `!!! <VERDICT>. This verdict is not one this version knows: read the checks below ...` | `true` |
| `CROSS_CHECKED`, `CHECKS_PASSED`, `NOT_APPLICABLE` | the name and the time | the name; the verdict sentence is in the certificate section | `false` |

A clean verdict is never called verified or correct: the checks look at the structure of a
calculation, not at whether it is the right one.

### E-mail

Two versions of the same message, as alternatives the reader's mail client chooses between: an
HTML layout (a white card in the widget's colours, with the verdict as a coloured banner for
`CONTRADICTED`, `FLAGGED` and `NOT_CHECKED`, the result as a table of the first 20 rows, and the
certificate in sections) and plain text. Both say the same things. The HTML has no script, no
image, no web font and no link, and every value in it (names, the question, cells, the
statement) is escaped. The plain text, in order:

1. for a caveat verdict, the verdict line;
2. the name, the approved query and who reviewed it, when the run was made and was due (in the
   schedule's time zone), the service identity, who defined the schedule;
3. the result: the first 20 rows as a text table, and how many rows there are (and that the
   result was cut at 500 rows, when it was);
4. the Answer Certificate: the verdict, the warnings, **what was checked** (the approval, and
   each check that established something) and **what was not checked** (every check that was
   due and did not run, with its reason, and what no check establishes, such as whether the
   data records missing values another way);
5. the statement that ran;
6. a footer: sent on a schedule without a language model, run id, and that the message carries
   data from your database.

All rows are in the attached CSV file (UTF-8 with a byte-order mark, comma separated). For a
caveat verdict its first rows repeat the verdict and the warnings, before the header. A cell
that starts with `=`, `+`, `-` or `@` and is not a number gets an apostrophe in front, as in
the widget's export, so it does not run as a formula when opened.

The message is `multipart/mixed`: a `multipart/alternative` (text, then HTML) and the CSV file.
Headers: `Auto-Submitted: auto-generated` (auto-responders do not answer it),
`X-Rosie-Verdict`, `X-Rosie-Run` and `X-Rosie-Schedule`, for mail rules.

### Webhook

A `POST` of `application/json; charset=utf-8`:

```json
{
    "type": "rosie.scheduled_answer",
    "version": 1,
    "verdict": "CONTRADICTED",
    "caveat": true,
    "headline": "CONTRADICTED. This figure counts some records more than once. Do not use it as a fact. ...",
    "schedule": { "id": "weekly-usa", "name": "Weekly revenue, USA", "cron": "0 8 * * MON", "timeZone": "America/New_York", "definedBy": "Dana Reyes, Finance", "definedOn": "2026-11-18" },
    "run": { "id": "3f2a9c1e-...", "scheduledFor": "2026-11-23T13:00:00.000Z", "ranAt": "2026-11-23T13:00:04.000Z", "runAs": "service:finance-reports", "trigger": "schedule" },
    "approval": { "id": "revenue-by-country", "question": "Revenue for a billing country", "approvedBy": "Dana Reyes, Finance", "approvedOn": "2026-11-11", "params": { "country": "USA" } },
    "sql": "SELECT SUM(Total) AS revenue FROM Invoice WHERE BillingCountry = 'USA' LIMIT 500",
    "result": { "columns": [ { "key": "revenue", "label": "Revenue", "type": "float" } ], "rows": [ { "revenue": "523.06" } ], "rowCount": 1, "totalCount": 1, "truncated": false },
    "checks": { "checked": [ "..." ], "warnings": [], "notChecked": [ "..." ] },
    "certificate": { "verdict": "...", "assertions": [], "evidence": { } }
}
```

`trigger` is `schedule`, or `manual` for `npm run schedule -- --run-now`. A consumer that reads
only `verdict`, `caveat` and `headline` does not take a contradicted figure for a fact. Read
`certificate.evidence` ([EVIDENCE.md](EVIDENCE.md)) before automating on a figure.

Delivered means a `2xx` answer. A `5xx`, `408`, `429`, a timeout or a connection that fails is
retried; any other `4xx` is final. A redirect (`3xx`) is never followed and is final: the
signed body goes only to the URL that was reviewed.

#### Verifying the signature

Every request carries three headers:

| Header | Value |
|---|---|
| `X-Rosie-Timestamp` | Seconds since the epoch when it was sent |
| `X-Rosie-Signature` | `v1=` and the hex HMAC-SHA256, keyed with `ROSIE_WEBHOOK_SECRET`, over `<timestamp>.<raw body>` |
| `X-Rosie-Delivery` | A UUID, the same on every retry of one delivery: drop one you already processed |

A receiver recomputes the HMAC over the **raw bytes** it received (before any JSON parsing),
compares in constant time, and refuses a timestamp more than five minutes away from its clock.
In Node.js:

```js
const crypto = require('crypto');
function verify(secret, headers, rawBody) {
    const ts = headers['x-rosie-timestamp'];
    const sig = headers['x-rosie-signature'] || '';
    if (!/^\d+$/.test(ts) || Math.abs(Date.now() / 1000 - Number(ts)) > 300) return false;
    const expected = 'v1=' + crypto.createHmac('sha256', secret).update(ts + '.').update(rawBody).digest('hex');
    return expected.length === sig.length && crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(sig));
}
```

In Python:

```python
import hmac, hashlib, time
def verify(secret: bytes, headers, raw_body: bytes) -> bool:
    ts = headers.get("X-Rosie-Timestamp", "")
    if not ts.isdigit() or abs(time.time() - int(ts)) > 300:
        return False
    expected = "v1=" + hmac.new(secret, ts.encode() + b"." + raw_body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, headers.get("X-Rosie-Signature", ""))
```

Slack and Teams incoming webhooks do not check signatures and expect their own message shape;
put a small relay (a function or an automation flow) between Rosie and them that verifies the
signature and formats the message. Native Slack and Teams channels are planned for a later
release.

#### Where the URL lives

A webhook URL of Slack or Teams carries its secret in the path. Write it in the environment
and reference it from the file: `"url": "env:FINANCE_WEBHOOK_URL"`. The name may not start with
`ROSIE_`, `AI_` or `DB_` (those are Rosie's own settings). A URL written in the file with a long
path or a query is warned by `check-context`. Rosie never records or shows a URL path: the run
row, the log, the overview and `--list` name the host only.

The URL must be `https`. Plain `http` is accepted only for `localhost` outside production. A
URL with a user name or password in it is refused. A URL written as an address on this machine,
a private network or link-local (`10.x`, `192.168.x`, `169.254.x`, `localhost`, `fd00::`...)
is accepted, since you wrote it, but warned at startup, by `check-context` and by `doctor` while
`ROSIE_SCHEDULE_WEBHOOK_HOSTS` is empty: set that list so the result can only go where you
meant. Host names are not resolved for this warning.

## Failures and retries

| What happened | Retried | Final status |
|---|---|---|
| A gate held (security hold, column grants not confirmed, audit unavailable, assistant off) | yes | `held` |
| The database did not answer, or the statement failed to run | yes | `failed` |
| The validator or a row policy refused the statement | no | `failed` |
| The approved query is suspended, past its `reviewBy`, or gone | no | `failed` |
| SMTP `4xx`, a timeout, a connection that fails | yes, that channel | |
| SMTP `5xx`, no recipient accepted, a login refused | no | |
| Webhook `5xx`, `408`, `429`, a timeout | yes, that channel | |
| Webhook other `4xx`, a redirect | no | |

`ROSIE_SCHEDULE_MAX_ATTEMPTS` (4) attempts each, waiting `ROSIE_SCHEDULE_RETRY_BASE_MS`
(60 seconds) before the first retry and twice as long before each next one. The run's status is
`delivered` when every channel delivered, `partial` when some did, `failed` when none did. A
recipient the SMTP server refuses is reported while the others receive the message.

## Several replicas

`ROSIE_SCHEDULE_LOCK=database` (the default): every replica may have `ROSIE_SCHEDULES=true`.
Each due run has a key, the schedule and the instant it was due, which every replica computes
the same way from the cron expression. The first replica whose `INSERT` of that key into
`rosie_scheduled_run` succeeds makes the run; the primary key refuses the others, which log
`scheduled_run_claimed_elsewhere`. Nothing has to be released and nothing expires.

When the engine database cannot be reached, the run waits and is retried, and is given up
rather than sent without its row: a missed report is visible (`/health`, the overview, the
log), and one sent twice to a mailing list cannot be taken back.

`ROSIE_SCHEDULE_LOCK=single`: set `ROSIE_SCHEDULES=true` on **one** replica only. The row is
still written, and a duplicate still stops a run (a restart does not send again), but a run on
time is made even while the engine database does not answer. A run made late always needs its
row, in both modes.

A replica that stops in the middle of a run leaves its row in status `running`; the overview
shows it as `abandoned` after an hour. It is not sent again by another replica.

## Watching it

- **`/health`** has a `scheduledAnswers` block, counts only (the route is anonymous):
  `enabled`, `lock`, `schedules`, `invalid`, `failing`, `running`, `lastRunAt`,
  `lastFailureAt`. A schedule whose last run failed, was held or reached only some channels, or
  a schedule that is not valid, makes `status` `degraded` (HTTP 200), never `unavailable`:
  questions are still answered ([MONITORING.md](MONITORING.md)).
- **The admin overview** (`GET /api/assistant/admin/overview`) has a `scheduledAnswers` field:
  each schedule with its next run in its own time zone and the last outcome in this process,
  the problems of the file, and the last 50 runs of every replica from the run table. No URL
  and no claim value, which can be secrets; recipients as a count.
- **The log**: `scheduled_run` (info when delivered, warn when partial, error otherwise),
  `scheduled_run_retry`, `scheduled_delivery_retry`, `scheduled_run_claimed_elsewhere`,
  `scheduled_run_overlap`, `scheduled_answer_invalid`, `scheduled_run_table_unreadable`.
- **The audit**: the answer in `rosie_audit_log` (user `service:<name>`, user agent
  `rosie-scheduler (<schedule id>)`, question `[approved query] <id>`), its certificate in
  `rosie_answer_certificate`, the run in `rosie_scheduled_run` and as a `rosie_scheduled_run`
  record in the audit file: who defined the schedule, when it was due and when it ran, rows,
  verdict, each channel's outcome. No record names an address or a URL path.

## Settings

| Setting | Default | |
|---|---|---|
| `ROSIE_SCHEDULES` | `false` | Run the schedules and send their results out |
| `ROSIE_SCHEDULE_LOCK` | `database` | `database` or `single` ([Several replicas](#several-replicas)) |
| `ROSIE_SCHEDULE_MAX_ATTEMPTS` | `4` | Attempts of a run and of each delivery (1 to 10) |
| `ROSIE_SCHEDULE_RETRY_BASE_MS` | `60000` | Wait before the first retry; it doubles |
| `ROSIE_SCHEDULE_CATCHUP_MINUTES` | `30` | How late a missed run may still be made (0: never) |
| `ROSIE_SCHEDULE_EMAIL_DOMAINS` | any | The only e-mail domains a schedule may send to |
| `ROSIE_SCHEDULE_WEBHOOK_HOSTS` | any | The only hosts a webhook may point at |
| `ROSIE_WEBHOOK_SECRET` | | Signs every webhook body; 32 characters or more. Required for a webhook |
| `ROSIE_SMTP_HOST` | | The SMTP server |
| `ROSIE_SMTP_PORT` | `587` (`465` with `tls`) | |
| `ROSIE_SMTP_SECURITY` | `starttls` | `starttls`, `tls`, or `none` (only for a relay on `localhost`) |
| `ROSIE_SMTP_USER`, `ROSIE_SMTP_PASSWORD` | | The login, when the server asks for one; both or neither |
| `ROSIE_SMTP_FROM` | | The sender, `Rosie <rosie@example.com>` |

While `ROSIE_SCHEDULES=true`, these refuse startup: SMTP without TLS to a host other than
`localhost`, a sender that is not an address, a login without its password, a webhook secret
shorter than 32 characters, a domain or host list with something that is not a domain.

With `starttls`, a server that does not offer STARTTLS is refused rather than used in clear
text, and the server's certificate is verified against `ROSIE_SMTP_HOST`. For a relay with a
certificate from a private authority, give Node.js that authority with `NODE_EXTRA_CA_CERTS`.
A password is never sent over a connection that is not encrypted, anything a server sends in
clear before the TLS handshake is discarded (RFC 3207), and a `ROSIE_SMTP_SECURITY` value the
client does not know is refused, never read as no TLS. TLS protects the message
from Rosie to the SMTP server only; from there it travels as your mail system sends it.

A schedule whose channel needs a setting that is missing (an e-mail channel without
`ROSIE_SMTP_HOST`, a webhook without `ROSIE_WEBHOOK_SECRET`, a URL variable that is not set, a
recipient outside `ROSIE_SCHEDULE_EMAIL_DOMAINS`) does not run, and says why in the log,
`/health` (`invalid`), the overview and `npm run schedule -- --list`.

## The command

```bash
npm run schedule -- --list                      # schedules, next runs, channels, problems (exit 1 when any)
npm run schedule -- --list --json
npm run schedule -- --run-now <id> --dry-run    # run the query and the checks, print the e-mail text, the
                                                # CSV and the webhook body; nothing sent, no run row
npm run schedule -- --run-now <id>              # run it now and send it (needs ROSIE_SCHEDULES=true)
```

A run made from the command passes the same gates as a scheduled one (the privilege check and
the masked-columns check are made first, as the server makes them at startup). It is recorded
with `trigger` `manual`. A dry run still runs the query, so it is in the audit log like any
answer.

## What it does not do

- **No native Slack or Teams channel yet.** Use the signed webhook and a relay.
- **Parameter values are fixed.** A schedule cannot say "last week" as a parameter; write the
  period into the approved statement (relative to the current date) when it should move.
- **One webhook secret** for every webhook of the installation.
- **No unsubscribe link.** Recipients are changed in the file.
- **500 rows at most**, the engine's row cap; the e-mail and the webhook say when a result was
  cut.
- **A run is made once.** A replica that dies mid-run does not hand it over; the row stays
  `running` and shows as `abandoned`.
- The service identity's claims are whatever the context directory declares: Rosie checks
  their form, not that the service should have them.

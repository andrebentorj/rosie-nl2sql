# A second opinion on numbers from other tools

A figure on a dashboard comes from a SQL statement somebody or something wrote: a Metabase
question, a Power BI dataset in DirectQuery, a Looker explore, a spreadsheet's connection, an
analyst's query, another AI assistant. Rosie can run that statement and put the Answer
Certificate on it, the same deterministic checks it runs on its own answers, with no model
involved. Use it to look again at a number before a decision rests on it, or to audit every
query behind a dashboard at once.

```bash
npm run certify -- --sql revenue.sql
```

```
revenue.sql
  Question: total revenue
  Verdict: CONTRADICTED (exit 1)
  Do not use this figure: re-derived over distinct records it comes out different, and the context declares the column a value of each record.
  Statement run: SELECT SUM(`i`.`Total`) AS `revenue` FROM `Invoice` AS `i` INNER JOIN `InvoiceLine` AS `il` ON `il`.`InvoiceId` = `i`.`InvoiceId` LIMIT 501
  Result: 1 row
    revenue
    20848.62
  Checks:
    [fail] fanout: The join produced 2240 rows from 412 distinct Invoice records, and the metric revenue declares Invoice.Total as a value of each Invoice record: this total counts some records more than once (about 5.4 times each).
    [pass] metric: Uses the declared definition of revenue: SUM(Invoice.Total).
  Second derivation: CONTRADICTED
    aggregated over the joined rows: 20848.62
    aggregated over distinct Invoice records: 2328.6
    Two derivations disagree: ...
  Not checked:
    - Whether the statement answers the question was not checked.
    - How the data records missing or special values (a 0 or an empty text that stands for "not recorded") was not verified.
    - No business entity is declared for the table the figure is taken over, so whether its rows are the things the question counts was not checked.
```

That is the demo database (`test/certify/fixtures/mysql/01-fanout-inflated-revenue.sql`): a
tile that joins invoice lines to invoices and sums the invoice total reports 20848.62; the
invoices add up to 2328.60.

## What it checks, and what it does not

**It checks the structure of the statement and facts of the data**, after running it on the
read-only connection ([CERTIFICATE.md](CERTIFICATE.md) has every check and its limits):

| Check | What it finds in a dashboard query |
|---|---|
| `fanout` | A join that repeats the rows a `SUM` or `AVG` adds up: the classic inflated total |
| `empty` | A filter on a value that does not exist (`Country = 'Brasil'` where the data says `Brazil`), with the closest real values: a zero that means "wrong filter", not "none" |
| `coverage` | A filter that keeps every row, so it is probably not filtering what was meant |
| `period` | A date range that runs past the last record: a "this year" figure over half a year of data |
| `zero_values` | An average that counts zeros that may stand for "not recorded" |
| `entity_grain` | A figure over rows that are several per business entity, where the context declares the entity |
| `metric` | With a question that names a declared metric: whether the statement computes it the declared way |
| second derivation | Where it is on (`AI_CROSS_CHECK`, on by default): the total recomputed taking each record once. Agreement gives `CROSS_CHECKED`; a disagreement on a column the context declares a value of each record gives `CONTRADICTED` |

**It does not prove that the statement answers the business question.** A statement that
sums the wrong column, uses the wrong denominator or reads "active customer" differently from
the person asking is structurally sound, and every check passes. On a public benchmark the
checks caught a small share of wrong answers, because most wrong answers are of that kind
([CERTIFICATE.md](CERTIFICATE.md#what-it-checks-and-what-it-cannot)). A clean verdict means
"no structural problem found in what was checked", and every result lists what was not
checked.

**It certifies the statement, not the tile.** What the BI tool does after the SQL returns is
not seen: a DAX measure, a Metabase post-aggregation, a Looker table calculation, a
spreadsheet formula, a currency conversion, a cache that is older than the database. Compare
the result printed here with the figure on the dashboard: if they differ, the difference is in
the tool, and the certificate says nothing about it.

**It reads the database now.** The checks run after the statement and not in the same
snapshot; a database being written to can make two figures differ for a reason that has
nothing to do with the statement.

## What gets refused, and why

The statement goes through exactly what an answer of the assistant goes through, and some
dashboard SQL will not pass. Nothing runs when it is refused (exit code 4), and the reason is
printed.

| Refusal | Why | What to do |
|---|---|---|
| A table or column that is not in the catalog | The catalog is the allowlist. BI tools often read views, staging schemas or their own derived tables (Looker PDTs, Metabase models materialised elsewhere) | Add the table to the context if it should be readable, or certify the equivalent statement over the base tables |
| A masked column anywhere in the statement, `SELECT *` on a table that has one | Masked columns are never read, whatever the purpose, including a filter or a join on them | Rewrite without the column. A masked column is never unmasked for a second opinion |
| A table under a row policy, without a verified identity | Row policies need the caller's claims ([ROW_POLICIES.md](ROW_POLICIES.md)). The command line carries none | Use `--url` with a signed token for the person, or certify tables without a policy |
| Not a single `SELECT` | Writes, several statements, temporary tables, procedure calls, variables | Certify the one `SELECT` behind the figure |
| `UNION`, system catalogs, server functions (`USER()`, `@@version`) | The validator's fixed rules ([SECURITY_OVERVIEW.md](SECURITY_OVERVIEW.md)) | `UNION ALL` inside a derived table is accepted (`AI_UNION_ALL`) |
| Longer than 8,000 characters | The limit of `/certify` and `certify_sql` | Generated SQL can exceed it (Power BI especially); certify the part behind one figure |
| Template syntax (`{{var}}`, `[[...]]`, `{% %}`, `${...}`) | Not SQL until the tool fills it in | Copy the SQL the tool actually ran, with the values in place |
| A dialect the database does not speak | The statement runs on the database Rosie is connected to (MySQL or MariaDB, PostgreSQL, SQL Server) | Certify against the source the dashboard reads; a warehouse copy in another engine is not supported |

Results are capped at 500 rows (`truncated` says so). The checks run their own probes on the
data and do not depend on the rows returned.

## Run it

```bash
npm run certify -- --sql q.sql                          # one statement
npm run certify -- --sql q.sql --question "Total revenue in 2025"
npm run certify -- --sql dashboards/sales/              # every .sql file under the folder
npm run certify -- --sql dashboards/sales/ --out report.json
npm run certify -- --sql q.sql --json                   # JSON on stdout for a pipeline
npm run certify -- --sql q.sql --context ./rosie-context --env-file ./ci.env
npm run certify -- --sql q.sql --url https://rosie.example.com   # through a server
```

A `-- question: ...` comment line in a file says what its figure is meant to answer (it is
what lets the `metric` check compare the statement with a declared metric). `--question` does
the same for a single file.

**Locally** (the default) the command runs in its own process with the installation's settings
(`.env`, or `--env-file`) and database, through the same engine path as `POST /certify`: the
validator, the masks, the row cap, the read-only connection, the checks. It refuses to start
when a front door would refuse to answer: a security hold on the read-only account, the strict
gate (`ROSIE_REQUIRE_COLUMN_GRANTS`) not satisfied, no audit sink under
`ROSIE_AUDIT_POLICY=fail-closed`, `AI_ASSISTANT_ENABLED=false`, settings that do not read,
`AI_ANSWER_CHECKS=false`. Every statement is audited like an answer, with the user
`cli:<login>` (or `ROSIE_CERTIFY_USER`) and session `certify-cli`, and its certificate is
recorded. It needs no model key. `ROSIE_CERTIFY_SQL` is turned on for this process only: that
switch guards the network doors, and whoever runs the command already holds the database
credentials it reads. No quota applies locally.

**Through a server** (`--url`) the command sends each statement to `POST /api/assistant/certify`
with the credential in `ROSIE_CERTIFY_TOKEN` (never on the command line). The server must have
`ROSIE_CERTIFY_SQL=true` ([API.md](API.md#post-certify-off-by-default)); its credential, gates,
per-person quotas, row policies for the token's claims and audit apply. Use this to certify as a
person, or from a machine that has no database credentials. The token travels with every
request, so `--url` must be `https`; plain `http` is accepted only to this machine (`localhost`,
`127.0.0.1`), or to another host with `--allow-insecure-url`, which says out loud that the
token goes in clear.

### Exit codes

| Code | Meaning |
|---|---|
| 0 | `CROSS_CHECKED`, `CHECKS_PASSED` or `NOT_APPLICABLE` |
| 1 | `CONTRADICTED` |
| 2 | `FLAGGED` |
| 3 | `NOT_CHECKED` |
| 4 | Refused: the statement was not run |
| 5 | Nothing could be certified: usage, settings, a hold, the connection, the server |

For a folder, the code of the most serious outcome, in the order 5, 1, 2, 4, 3, 0. A
pipeline that accepts some outcomes tests for their codes. To fail only on a contradiction or
on a run that could not certify:

```bash
npm run -s certify -- --sql dashboards/ --out report.json; code=$?
[ "$code" -ne 1 ] && [ "$code" -ne 5 ]
```

Or read `exitCode` and `items[].verdict` from the report.

### The report of a folder

The text summary lists the counts per verdict, then one line per file, worst first, then the
detail of every statement that did not pass. `--out` (or `--json`) gives the same as JSON:

```json
{
  "generatedAt": "...", "engineVersion": "1.36.0", "projectId": "chinook", "mode": "local",
  "source": "/repo/dashboards/sales",
  "summary": { "total": 9, "certified": 6, "byVerdict": { "CONTRADICTED": 1, "FLAGGED": 3, "CROSS_CHECKED": 1, "NOT_APPLICABLE": 1 }, "refused": 3, "errors": 0 },
  "exitCode": 1,
  "items": [ { "file": "01-fanout-inflated-revenue.sql", "question": "total revenue", "status": "certified",
               "verdict": "CONTRADICTED", "exitCode": 1, "sql": "...", "notChecked": [ "..." ],
               "certificate": { "verdict": "...", "assertions": [], "crossCheck": {}, "skipped": [], "evidence": {}, "provenance": {} },
               "result": { "rowCount": 1, "totalCount": 1, "truncated": false, "columns": [] } } ]
}
```

`certificate` is the object `/ask` returns ([EVIDENCE.md](EVIDENCE.md) reads it dimension by
dimension); the re-derivation statement of the second derivation is in
`certificate.crossCheck.alternative.sql`, so a reviewer can run it themselves. Rows are left out
of a folder's report unless `--with-rows`; a single file's JSON has them.

The fixtures under `test/certify/fixtures/` are a working example of a dashboard folder: one
statement per check on the demo database, and three that are refused.

## Getting the SQL out of the tool

Copy the SQL the tool actually sends to the database, with the values of its filters in place,
and save one statement per `.sql` file. What follows is how that is usually done; menus move
between versions, so check your version's documentation.

### Metabase

- **A native (SQL) question**: the SQL is the text of the editor. If it uses variables
  (`{{country}}`) or optional clauses (`[[AND ...]]`), put in the values the dashboard used and
  remove the brackets: the command refuses the template as it stands and says so.
- **A question built in the query builder**: open it in the notebook editor and use "View the
  SQL" (or "Convert this question to SQL") to see the statement Metabase generates. Metabase's
  API can return the same text for a saved question's query (the dataset "native" endpoint),
  which is how a script can export every card of a dashboard.
- Metabase models and saved questions used as a source appear as nested queries in the
  generated SQL; tables they read must be in Rosie's catalog.

### Power BI

DAX is not SQL, and the figure on a report is often not the result of any one SQL statement.
What can be certified depends on the storage mode:

- **DirectQuery**: Power BI sends SQL to the database for each visual. Capture it from the
  database's own query log (SQL Server Extended Events or Query Store, PostgreSQL
  `log_statement` or `pg_stat_statements`, the MySQL general log), or with a tool that traces
  the DirectQuery calls (DAX Studio's server timings, Performance Analyzer in Power BI
  Desktop). This SQL is generated, verbose and full of nested derived tables; some of it will
  be refused (a construct the validator does not accept, more than 8,000 characters) and some
  will come back `NOT_CHECKED` where the checks cannot see through the nesting.
- **Import mode**: the SQL is the source query of a table in Power Query (a native query, or
  "View Native Query" on a folded step). Certifying it checks the data that was imported, not
  the measures computed on it afterwards.
- A measure written in DAX (a `CALCULATE` with filters, a time-intelligence function) is not
  checked. The useful practice is to write the SQL that should give the same figure and
  certify that, then compare the two numbers.

### Looker

- In an Explore, the **SQL** tab shows the statement Looker generated, with Liquid already
  rendered; copy it from there. The API can return the same text for a query (the run endpoint
  with the `sql` result format).
- Persistent derived tables live in a scratch schema under generated names; a statement that
  reads them is refused unless those tables are in the catalog. Certify the derived table's own
  SQL, or the equivalent over the base tables.
- Table calculations and merged results happen in Looker after the SQL and are not checked.

### A spreadsheet or an analyst's query

The SQL of a spreadsheet's database connection (Excel's Power Query native query, an ODBC
command text, a Google Sheets connector query) or a query an analyst sends: save it as it is.
Formulas applied in the sheet afterwards are not checked.

### An AI agent

An agent connected to Rosie over MCP can certify the SQL it wrote itself with the tool
`certify_sql`, where the installation turned it on ([MCP_SERVER.md](MCP_SERVER.md)). The same
rules apply: the agent's credential, quotas, row policies and audit.

## Turning on the network doors

The command line needs nothing turned on. `POST /certify` and `certify_sql` stay off until
`ROSIE_CERTIFY_SQL=true`, because with them a holder of the API credential hands the engine SQL
directly instead of asking in words. They enforce what a question does: the same gates (the
switch, the emergency stop, the security hold, the strict column-grants gate, the audit policy),
the same credential, quotas of their own per verified person or per address, the caller's row
policies, the masks, the row cap, an audit row and a certificate row for every request. Read
[API.md](API.md#post-certify-off-by-default) before turning them on.

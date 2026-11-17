# SQL Server

Rosie runs on Microsoft SQL Server as well as MySQL and PostgreSQL. The engine is
selected with one variable; everything else (context directory, widget, MCP server, Answer
Certificate, row policies, eval harness) is the same.

## Enable it

```bash
DB_DIALECT=mssql             # sqlserver and tsql are accepted too
DB_HOST=db.internal
DB_PORT=1433
DB_DATABASE=Sales            # the database that holds the business data
DB_SCHEMA=dbo                # the ONE schema Rosie reads (default: dbo)
DB_SSL=true                  # true (encrypt and verify the certificate), no-verify, or unset; any other value stops the server
DB_ENGINE_DATABASE=rosie     # database for Rosie's own four tables (audit, certificates, feedback, rate limits)
DB_RO_USER=rosie_readonly
DB_RO_PASSWORD=...
```

The quickest route is the wizard. It always writes the context directory and the `.env`;
it creates the read-only login only with `--create-ro-user` and the engine tables only with
`--create-engine-tables`:

```bash
npm run init -- --url 'mssql://user:password@host:1433/Sales' --name "My Project" \
                --create-ro-user --create-engine-tables
```

In express mode, the default, Rosie's own tables go to a separate database, `rosie`, unless you
pass `--engine-db same`.

By hand: create the engine tables from `db/schema.mssql.sql` and generate the catalog with
`DB_DIALECT=mssql npm run generate-catalog`. The steps are those of `docs/INSTALL.md`; this
page gives the SQL Server statements. `DB_ENGINE_SCHEMA` is not read on SQL Server: the
engine addresses its tables without a schema name, so they are found in the default schema
of the main account.

## The engine database and the main login

```sql
CREATE LOGIN rosie_app WITH PASSWORD = '...';
CREATE DATABASE rosie;
GO
USE rosie;
CREATE USER rosie_app FOR LOGIN rosie_app;
```

```bash
sqlcmd -S <host> -U <admin> -d rosie -i db/schema.mssql.sql
```

```sql
USE rosie;
-- Always: the engine inserts its records and reads them back.
GRANT SELECT, INSERT ON SCHEMA::dbo TO rosie_app;
-- Only with ROSIE_RATE_LIMIT_STORE=database.
GRANT UPDATE, DELETE ON OBJECT::dbo.rosie_rate_limit TO rosie_app;
-- Only for the account that runs `npm run retention -- --confirm` (it can be another login).
GRANT SELECT, DELETE ON OBJECT::dbo.rosie_audit_log TO rosie_retention;
GRANT SELECT, DELETE ON OBJECT::dbo.rosie_answer_certificate TO rosie_retention;
GRANT SELECT, DELETE ON OBJECT::dbo.rosie_feedback TO rosie_retention;
```

`npm run doctor` checks that the tables are there, not these grants. `npm run
generate-catalog` and `npm run init` read the business database with `DB_USER`, so that
login also needs to read its metadata there, or run those commands with another account.

## The read-only login

This is strict mode. In express mode, the default, the login gets `GRANT SELECT ON
SCHEMA::dbo` and Rosie's validator is what refuses the masked columns
([INSTALL.md](INSTALL.md#choosing-a-masking-mode)). On SQL Server column grants end at
"masked by Rosie only" anyway (below), so strict mode needs the reviewed-view design. In
strict mode:

```sql
CREATE LOGIN rosie_readonly WITH PASSWORD = '...';
USE Sales;
CREATE USER rosie_readonly FOR LOGIN rosie_readonly;
-- A table with no masked column: the whole table.
GRANT SELECT ON OBJECT::dbo.Invoice TO rosie_readonly;
GRANT SELECT ON OBJECT::dbo.InvoiceLine TO rosie_readonly;
-- A table with masked columns: its unmasked columns only.
GRANT SELECT ON OBJECT::dbo.Customer (CustomerId, FirstName, LastName, City, Country) TO rosie_readonly;
```

Grant table by table, and column by column on the tables that have masked columns. Do not
use `GRANT SELECT ON SCHEMA::dbo` or the `db_datareader` role: both cover every column of
every table, and a narrower grant does not remove a broader one. Why this matters is in
[INSTALL.md](INSTALL.md), Step 2.

A login created before 1.30.0 by `npm run init -- --create-ro-user` has
`GRANT SELECT ON SCHEMA::dbo`, and one created by hand may be in `db_datareader`. Take both
away first, then grant as above:

```sql
REVOKE SELECT ON SCHEMA::dbo FROM rosie_readonly;
ALTER ROLE db_datareader DROP MEMBER rosie_readonly;
```

Running `--create-ro-user` again does not do this. It grants nothing to a read-only user you
named as existing, and when `rosie_readonly` already exists its `CREATE LOGIN` fails and the
wizard only prints the statements. `npm run -s grants` (since 1.31.0) reads what the login
holds and prints the statements that remove the broad grants and add the column grants
([PILOT_ONBOARDING.md](PILOT_ONBOARDING.md)). Then check, as `rosie_readonly`:
`SELECT Email FROM dbo.Customer WHERE 1 = 0;` must fail with error 230 (permission denied
on the column). Column grants are the minimum on SQL Server, not the end: they do not stop
`PIVOT` (below).

At startup, and every 15 minutes after, Rosie checks this login and refuses to answer
(`securityHold`) if it finds any of:

- the `sysadmin` server role, or a login named `sa`, `admin` or `root`;
- membership in `db_owner`, `db_datawriter` or `db_ddladmin`;
- a database permission beyond connecting and reading (CREATE TABLE, ALTER, CONTROL, ...);
- INSERT, UPDATE, DELETE or ALTER on any table, counting permissions inherited through
  roles.

### Column grants do not stop `PIVOT`

SQL Server checks column permissions on the aggregated column and the `FOR` column of a
`PIVOT`, not on the columns it passes through. Observed on SQL Server 2022 (CU23) with
column grants only, with a table grant plus a column `DENY`, with `db_datareader` plus a
column `DENY`, and in a case-sensitive database: with `SELECT` on one column `c` of a table,

```sql
SELECT p.* FROM t PIVOT (COUNT(c) FOR c IN ([0])) AS p
```

returns every other column of `t`, the masked ones included. Rosie's validator refuses
`PIVOT` and `UNPIVOT` by name since 1.31.0 ("PIVOT and UNPIVOT are not allowed: use GROUP BY
with SUM(CASE WHEN ... THEN ... END) for one column per value"), so on SQL Server it is the
only layer that stops this. With the column grants above, the check therefore reports
`masked by Rosie only`, with each masked column readable through `PIVOT` (reason
`sqlserver_pivot_reads_masked`, limit `mssql_pivot`). That is the true state of such an
account, not a defect of the check.

For the database to refuse the masked columns, the account must have no permission at all on
a table that has masked columns. Read it through a view that selects only the unmasked
columns:

```sql
CREATE VIEW dbo.CustomerPublic AS
  SELECT CustomerId, FirstName, LastName, City, Country FROM dbo.Customer;
GRANT SELECT ON OBJECT::dbo.CustomerPublic TO rosie_readonly;
-- So the check can follow the view to the table it reads:
GRANT VIEW DEFINITION ON OBJECT::dbo.CustomerPublic TO rosie_readonly;
GRANT VIEW DEFINITION ON OBJECT::dbo.Customer TO rosie_readonly;
-- Nothing on dbo.Customer itself.
```

In the catalog, add the view with `"reviewedView": true`, and list `Customer` with its masked
columns only, so the check finds every listed unmasked column readable and every masked one
denied. A `PIVOT` over the view returns only the view's columns. This design, with the view
in the catalog as described, was run live in the internal audit of 1.31.0: the check said
"masked by the database", and 21 statements against the base table, `PIVOT` included, were
refused (229). Without `VIEW DEFINITION` the result was "not verified on SQL Server". The
live suite (`npm run test:sqlserver`, section `[grants 3]`) covers the same grants with the
view outside the catalog.

Keep at least one mask entry for those tables. A context with no masked columns reports
"no masked columns", passes `ROSIE_REQUIRE_COLUMN_GRANTS` and skips the view checks: a
catalog of views only, with no masks, is never checked.

### What the check examines on SQL Server

Rosie runs the check at startup and every 15 minutes, as the read-only login:

- a probe of each masked column, and `HAS_PERMS_BY_NAME` on it;
- membership in `db_datareader`, `IMPERSONATE` on another user or login,
  `IMPERSONATE ANY LOGIN` and `CONTROL SERVER`; each makes every masked column reachable and
  is named as the role (`readable[].roles`);
- a `PIVOT` probe per masked column, over a column the login can read;
- every view, table function, scalar function and procedure the login may use (up to 200),
  followed through `sys.dm_sql_referenced_entities`. A view or function owned by the table's
  owner returns the masked columns to anyone allowed to use it, whatever the column grants
  (ownership chaining). One that reads a masked column is reported (`view_reads_masked`)
  unless it is a reviewed catalog view;
- since 1.32.0, an allowlist: every object of the current database the login has `SELECT`,
  `EXECUTE` or a column grant on (tables, views, synonyms, table and scalar functions,
  procedures, CLR and aggregate functions, extended procedures) that is neither a catalog
  table nor reviewed is `object_not_verifiable`, a static procedure such as
  `SELECT InvoiceId, Total FROM dbo.Invoice` included; so is every linked server with data
  access (the login cannot see `sys.linked_logins`, so a mapping cannot be ruled out) and
  every other user database the login can open (`master`, `tempdb`, `model`, `msdb`
  skipped). List tables, views and synonyms in `"reviewedViews"` and functions and procedures
  in `"reviewedRoutines"` at the top of `schema-catalog.json`, or take the permission away;
  linked servers and other databases can only be removed. A synonym for a masked catalog table
  is reported as reading it (`view_reads_masked`, kind `synonym`), and listing it does not
  accept it: a synonym is the table. Entries compare exactly as (schema, name) pairs, and
  `[dbo].[p_totals]` (with `]]` for a bracket inside) or `"dbo"."p_totals"` are accepted. The
  dependency walk runs only on reviewed objects and catalog views; what it shows is reported,
  listed or not, and `npm run grants -- --verify` warns about a listed object that uses `*` or
  dynamic SQL. Rather than listing, consider whether the login needs `EXECUTE` on any
  procedure;
- cross-database ownership chaining, in the database and on the server.

What it cannot establish makes the result `not verified on SQL Server`
(`sqlserver_not_verified`, hold code `E_COLUMN_GRANTS_SQLSERVER_NOT_VERIFIED`): an object the
login may use but cannot see into (grant `VIEW DEFINITION` on it and on what it reads, or
remove the access), a reference to another database, cross-database ownership chaining being
on, more than 200 objects, a procedure or function held as above, or a `PIVOT` probe that
could not be written. Not covered: statement text of other sessions in `sys.dm_exec_*` for a
login with `VIEW SERVER STATE`, and the history of a temporal catalog table read through
`FOR SYSTEM_TIME` (the table's own grant; a history table outside the catalog is held), and
an object an administrator creates in `dbo` with one of the exact diagram names and types and flags as shipped (`sp_MS_marksystemobject`). Objects flagged as shipped by Microsoft are not trusted: only the
database-diagram objects SQL Server creates in `dbo` are skipped when flagged, each by its
exact name and type (the table `sysdiagrams`, the scalar function `fn_diagramobjects`, and the
procedures `sp_alterdiagram`, `sp_creatediagram`, `sp_dropdiagram`,
`sp_helpdiagramdefinition`, `sp_helpdiagrams`, `sp_renamediagram` and `sp_upgraddiagrams`). No independent reviewer has reproduced the SQL Server results; they were
established on a live SQL Server 2022 CU23. Since 1.31.0 that
result means only this; it no longer means that the check was never run on a live server.

Other facts from the live run: a column `DENY` beats a table, schema or `db_datareader`
`GRANT`; a table `DENY` does not beat a column `GRANT` (with Common Criteria compliance off,
the default). Dynamic Data Masking is not a denial and does not count. Application roles are
not examined.

**Counting under column grants.** SQL Server refuses `SELECT COUNT(*) FROM Customer`,
`COUNT(1)`, `SELECT TOP 1 1` and `EXISTS (SELECT 1 FROM Customer)` when the account lacks any
column of the table: a statement that names no column needs them all. Since 1.31.0 Rosie
rewrites `COUNT(*)` and `COUNT(<constant>)`, and the select list of an `EXISTS` subquery,
over one catalog table that has masked columns, to a key column, after validation: the
catalog `primaryKey`, only when the catalog marks that column `"nullable": false` and it is
not masked; otherwise the first unmasked column marked `"nullable": false`; otherwise the
statement is left as written (and under column grants it is refused with 230).
`npm run generate-catalog` writes `"nullable": false` for `NOT NULL` columns; a hand-written
catalog must state it. `COUNT(NULL)` is never rewritten (SQL Server refuses it, 8117). The
count is the same; the SQL
shown with the answer reads `COUNT([Customer].[CustomerId])`. Joins are not rewritten (with an
outer join the two counts differ), and the certificate's own probes are not either.

## What the engine does differently on SQL Server

**Row cap.** SQL Server has no `LIMIT`. The engine caps every statement with `TOP`, keeps a
smaller `TOP n` written by the model, and bounds `OFFSET ... FETCH` paging. A statement that
arrives with `LIMIT` is refused with a message telling the model to use `SELECT TOP n`, so it
corrects itself on the next step.

**Timeout.** `AI_QUERY_TIMEOUT_MS` becomes the driver's request timeout on the read-only
pool: the statement is cancelled on the server when it expires.

**Functions are allowlisted.** As on PostgreSQL and (since 1.31.0) MySQL, only the functions an analytical SELECT needs
are accepted: aggregates, window functions, common text, numeric and date functions
(`DATEADD`, `DATEDIFF`, `DATEPART`, `DATEFROMPARTS`, `EOMONTH`, `FORMAT`, ...), `COALESCE`,
`ISNULL`, `NULLIF`, `IIF`, `CAST`, `CONVERT`, `TRY_CAST`. Anything else is refused with
`Function not allowed: NAME`. `REPLICATE` and `SPACE` are left out on purpose (one call can
allocate a very large string). The list is `ALLOWED_FUNCTIONS` in
`server/ai-assistant/dialect/mssql.js`.

**Rejected outright:**

| Construct | Why |
|---|---|
| `@@` variables and `@local` variables | Session and server information; no statement Rosie runs needs one |
| `xp_`, `sp_`, `EXEC`, `WAITFOR`, `OPENROWSET`, `OPENQUERY`, `OPENDATASOURCE` | Procedures, delays and remote data sources |
| `sys.` objects and `INFORMATION_SCHEMA` | System catalogs |
| Functions in `FROM`, temporary tables (`#name`) | They do not appear in the table list, so the table allowlist never sees them |
| Schema-qualified functions | The allowlist is a set of bare names inside one schema |
| `UNION`, `INTERSECT`, `EXCEPT`, `SELECT INTO` | Not needed for analytics; each widens what a statement can reach. `UNION ALL` alone is accepted, inside a derived table or `WITH` body, with every branch validated as a statement of its own (`AI_UNION_ALL=false` refuses it too) |
| `FOR XML`, `FOR JSON`, table hints (`WITH (NOLOCK)` and the rest) | Change the shape of the result or the locking behaviour |
| `PIVOT`, `UNPIVOT` (since 1.31.0, by name) | A `PIVOT` reads columns the account has no permission on (above). Use `GROUP BY` with `SUM(CASE WHEN ... THEN ... END)` |
| `NATURAL JOIN` (all engines since 1.31.0) | It joins on every shared column name without naming one. Join with `ON` |
| Backslashes | Parser and server can disagree on where a line ends |

**Identifiers.** Square brackets (`[Order Details]`) are understood everywhere the engine
reads a statement, including comment stripping and the eval scorer. Names are matched without
regard to case, as the default collations do. Two table names or aliases in one statement
that differ only by letter case are refused, and the dialect note sent to the model says so:
"Write each table alias the same way every time, in one letter case; aliases that differ
only by case are rejected."

**Date parts.** `DATEPART(year, col)`, `DATEDIFF(day, a, b)` and similar take a bare word as
their first argument. The validator does not treat it as a column unless a table really has a
column with that name.

## Demo and tests

```bash
./scripts/demo/fetch-data.sh
docker compose -f demo/docker-compose.yml --profile sqlserver up -d sqlserver   # 127.0.0.1:1434
./scripts/demo/setup-sqlserver.sh
cp demo/env.sqlserver.example .env        # then set OPENAI_API_KEY and ROSIE_CONTEXT_DIR
npm run test:sqlserver                    # live suite against that server
```

Starting the container accepts Microsoft's licence terms for SQL Server Developer edition,
which is why the service sits behind a compose profile. The image is published for x86-64; on
Apple Silicon it runs under emulation.

The live suite creates its own database and logins and removes them at the end. It covers
catalog introspection, the privilege audit, the timeout, the tools, the Answer Certificate
probes and row policies with two tenants, and since 1.31.0 the column grants
(`test/dialect/sqlserver-grants-live.js`): T-SQL shapes refused with 229 or 230, the `PIVOT`
read and its refusal by the validator, counting under column grants, views, functions,
procedures, `db_datareader`, `IMPERSONATE`, ownership chaining, the reviewed-view design, and
the wizard's grants. The suite runs in CI on every push; its column-grant sections are new
in 1.31.0 and have not yet had a green run in CI (they pass locally against SQL Server 2022
CU23).

The demo question bank carries `sql_truth_mssql` variants where the reference query needs
T-SQL (`TOP`, `YEAR()`, `DATEPART`, `DATEFROMPARTS`). Results are in `docs/BENCHMARK.md`.

## Limits

- One schema per deployment.
- Column grants alone leave the masked columns readable through `PIVOT`; only Rosie's
  validator stops that. "Masked by the database" needs the reviewed-view design above.
- Views and functions the login may use are followed only when it has `VIEW DEFINITION` on
  them; otherwise the result is `not verified on SQL Server`. What a reviewed view exposes is
  your statement, not a check. Common Criteria compliance on was not tested.
- `validate_query` says whether a statement is valid; it returns no plan on any engine (since
  1.30.0). On SQL Server it does not ask for one: that needs a permission the read-only login
  must not have.
- The function allowlist is fixed in code; there is no configuration to extend it yet.
- Driver types: `BIGINT` arrives as text, `DECIMAL` and `MONEY` as numbers, `DATE` and
  `DATETIME` as dates read in UTC.
- Azure SQL Database and Azure Synapse use the same protocol and should work with
  `DB_SSL=true`, but have not been tested.

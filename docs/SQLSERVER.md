# SQL Server

Rosie runs on Microsoft SQL Server as well as MySQL, MariaDB and PostgreSQL. The engine is
selected with one variable; everything else (context directory, widget, MCP server, Answer
Certificate, row policies, eval harness) is the same.

## Enable it

```bash
DB_DIALECT=mssql             # sqlserver and tsql are accepted too
DB_HOST=db.internal
DB_PORT=1433
DB_DATABASE=Sales            # the database that holds the business data
DB_SCHEMA=dbo                # the ONE schema Rosie reads (default: dbo)
DB_SSL=true                  # true (encrypt and verify the certificate), no-verify, or unset
DB_ENGINE_DATABASE=rosie     # database for Rosie's own four tables (audit, certificates, feedback, rate limits)
DB_RO_USER=rosie_readonly
DB_RO_PASSWORD=...
```

The quickest route is the wizard. It always writes the context directory and the `.env`;
it creates the read-only login only with `--create-ro-user` and the engine tables only with
`--create-engine-tables`:

```bash
npm run init -- --url 'mssql://user:password@host:1433/Sales' --name "My Project" \
                --create-ro-user --create-engine-tables --engine-db rosie
```

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

```sql
CREATE LOGIN rosie_readonly WITH PASSWORD = '...';
USE Sales;
CREATE USER rosie_readonly FOR LOGIN rosie_readonly;
GRANT SELECT ON SCHEMA::dbo TO rosie_readonly;
```

At startup, and every 15 minutes after, Rosie checks this login and refuses to answer
(`securityHold`) if it finds any of:

- the `sysadmin` server role, or a login named `sa`, `admin` or `root`;
- membership in `db_owner`, `db_datawriter` or `db_ddladmin`;
- a database permission beyond connecting and reading (CREATE TABLE, ALTER, CONTROL, ...);
- INSERT, UPDATE, DELETE or ALTER on any table, counting permissions inherited through
  roles.

## What the engine does differently on SQL Server

**Row cap.** SQL Server has no `LIMIT`. The engine caps every statement with `TOP`, keeps a
smaller `TOP n` written by the model, and bounds `OFFSET ... FETCH` paging. A statement that
arrives with `LIMIT` is refused with a message telling the model to use `SELECT TOP n`, so it
corrects itself on the next step.

**Timeout.** `AI_QUERY_TIMEOUT_MS` becomes the driver's request timeout on the read-only
pool: the statement is cancelled on the server when it expires.

**Functions are allowlisted.** As on PostgreSQL, only the functions an analytical SELECT needs
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
| Backslashes | Parser and server can disagree on where a line ends |

**Identifiers.** Square brackets (`[Order Details]`) are understood everywhere the engine
reads a statement, including comment stripping and the eval scorer. Names are matched without
regard to case, as the default collations do.

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
probes and row policies with two tenants. It runs in CI on every push.

The demo question bank carries `sql_truth_mssql` variants where the reference query needs
T-SQL (`TOP`, `YEAR()`, `DATEPART`, `DATEFROMPARTS`). Results are in `docs/BENCHMARK.md`.

## Limits

- One schema per deployment.
- `validate_query` validates a statement but returns no execution plan on SQL Server: asking
  for a plan needs a permission the read-only login must not have.
- The function allowlist is fixed in code; there is no configuration to extend it yet.
- Driver types: `BIGINT` arrives as text, `DECIMAL` and `MONEY` as numbers, `DATE` and
  `DATETIME` as dates read in UTC.
- Azure SQL Database and Azure Synapse use the same protocol and should work with
  `DB_SSL=true`, but have not been tested.

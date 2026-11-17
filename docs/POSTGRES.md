# PostgreSQL

Rosie runs on PostgreSQL as well as MySQL and SQL Server. The engine is selected with one
variable; everything else (context directory, widget, MCP server, Answer Certificate, eval
harness) is the same.

## Enable it

```bash
DB_DIALECT=postgres
DB_HOST=db.internal
DB_PORT=5432
DB_DATABASE=sales            # the database that holds the business data
DB_SCHEMA=public             # the ONE schema Rosie reads (default: public)
DB_SSL=true                  # true (verify the certificate), no-verify, or unset
DB_ENGINE_DATABASE=rosie     # database for Rosie's own four tables (audit, certificates, feedback, rate limits)
DB_ENGINE_SCHEMA=public      # schema for those tables inside it (default: public)
DB_RO_USER=rosie_readonly
DB_RO_PASSWORD=...
```

Create the engine tables from `db/schema.postgres.sql` (or let `npm run init` do it with
`--create-engine-tables`), and generate the catalog as usual:
`DB_DIALECT=postgres npm run generate-catalog`. The steps are those of `docs/INSTALL.md`;
this page gives the PostgreSQL statements.

## The engine database and the main role

```sql
CREATE DATABASE rosie;
CREATE ROLE rosie_app LOGIN PASSWORD '...';
GRANT CONNECT ON DATABASE rosie TO rosie_app;
```

```bash
psql -d rosie -f db/schema.postgres.sql
```

Then, connected to `rosie`:

```sql
GRANT USAGE ON SCHEMA public TO rosie_app;
-- Always: the engine inserts its records and reads them back.
GRANT SELECT, INSERT ON ALL TABLES IN SCHEMA public TO rosie_app;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO rosie_app;     -- the id columns are BIGSERIAL
-- Only with ROSIE_RATE_LIMIT_STORE=database.
GRANT UPDATE, DELETE ON rosie_rate_limit TO rosie_app;
-- Only for the account that runs `npm run retention -- --confirm` (it can be another role).
GRANT SELECT, DELETE ON rosie_audit_log, rosie_answer_certificate, rosie_feedback TO rosie_retention;
```

`npm run doctor` checks that the tables are there, not these grants. `npm run
generate-catalog` and `npm run init` read the business schema with `DB_USER`, so that role
also needs to read it, or run those commands with another account.

## The read-only role

Grant it table by table, and column by column on the tables that have masked columns: the
unmasked columns only. Do not use `GRANT SELECT ON ALL TABLES IN SCHEMA`: it is a table-level
grant on every table, masked ones included. Why this matters, and how Rosie checks the
result, is in [INSTALL.md](INSTALL.md), Step 2.

```sql
CREATE ROLE rosie_readonly LOGIN PASSWORD '...';
GRANT CONNECT ON DATABASE sales TO rosie_readonly;
GRANT USAGE ON SCHEMA public TO rosie_readonly;
-- A table with no masked column: the whole table.
GRANT SELECT ON public.invoice, public.invoice_line TO rosie_readonly;
-- A table with masked columns: its unmasked columns only, no table-level SELECT.
GRANT SELECT (customer_id, first_name, last_name, city, country)
      ON public.customer TO rosie_readonly;
```

Quote names that are not all lower case, as `npm run init` does: an unquoted name is folded
to lower case, so a table created as `"Customer"` is granted with
`GRANT SELECT ("CustomerId", "FirstName") ON public."Customer" TO rosie_readonly;`. A table
created later is not readable by this role until you grant it.

**An account created before 1.30.0.** `npm run init -- --create-ro-user` used to grant
`SELECT ON ALL TABLES IN SCHEMA public`. Take that away first, then grant as above:

```sql
REVOKE SELECT ON ALL TABLES IN SCHEMA public FROM rosie_readonly;
```

Running `--create-ro-user` again does not do this. It grants nothing to a read-only user you
named as existing, and when `rosie_readonly` already exists its `CREATE ROLE` fails and the
wizard only prints the statements.

The traps, each of which leaves a masked column readable:

- **A table-level grant implies every column.** `GRANT SELECT ON public.customer` covers all
  its columns, present and future, and a column `REVOKE` under it changes nothing: a
  narrower grant does not remove a broader one. If the role already has a table-level grant
  (from `ON ALL TABLES IN SCHEMA`, for example), take it away first
  (`REVOKE SELECT ON public.customer FROM rosie_readonly;`) and then grant the columns.
- **PUBLIC.** A privilege granted to `PUBLIC` is held by every role. In `psql`, `\dp
  public.customer` must show no entry that starts with `=` (that is PUBLIC) granting `r`.
- **`pg_read_all_data`** (PostgreSQL 14 and later) reads every column of every table. The
  role must not be a member, directly or through another role.
- **Roles the account can switch to.** A role the account is a member of is one `SET ROLE`
  away, even with `INHERIT FALSE`. Keep `rosie_readonly` out of other roles. `\du
  rosie_readonly` shows its memberships.
- **Default privileges.** `ALTER DEFAULT PRIVILEGES ... GRANT SELECT ON TABLES TO
  rosie_readonly` makes every new table fully readable. Do not set it for this role.

Then check, as `rosie_readonly`, for each masked column: `SELECT email FROM public.customer
WHERE 1 = 0;` (with the block above), or `SELECT "Email" FROM public."Customer" WHERE 1 = 0;`
when the names are not lower case, must fail with `42501` (permission denied). Rosie runs the same check at startup and every 15 minutes,
asks every role the account is a member of, and reports the result in `/health`
([INSTALL.md](INSTALL.md#check-what-the-read-only-account-can-read)). A column privilege does
not reach through a view, which reads with its owner's rights: mask the view's column too,
or leave the view out of the catalog. Since 1.31.0 the check also follows, through
`pg_depend`, every view and materialized view that reads a masked column (views over views
included) and reports one the account or a role it can switch to can read
(`view_reads_masked`), unless it is a catalog view marked `"reviewedView": true`; and a
catalog view without that mark keeps the result at "masked by Rosie only"
(`view_not_reviewed`). It also walks the partitions, inheritance children and parents of
every masked table (each is a table with its own privileges, and a schema-wide grant gives
one to every partition; `postgres_inheritance_reads_masked`), and lists every
`SECURITY DEFINER` function or procedure outside `pg_catalog` and `information_schema` that
the role or one of its roles may execute (`EXECUTE` goes to `PUBLIC` by default). Such a
function is never passed on its text: it is `object_not_verifiable` unless it is listed in
`"reviewedRoutines"` at the top of `schema-catalog.json` (`"schema.name"`, read as
PostgreSQL resolves a name: an unquoted part folds to lower case, a double-quoted part keeps
its case, so `rosie.F_Clean` means `f_clean` and `rosie."F_Clean"` means `F_Clean`). A
listing is by name and covers every overload of that exact name, present and future: it is
your statement, not a check. A listed one whose body names a masked column is still
`view_reads_masked` and keeps the result at "masked by Rosie only". `npm run -s grants` prints the `REVOKE` and `GRANT` statements for the
role as it is, PUBLIC, role memberships, default privileges and grants on partitions and
inheritance relatives included ([PILOT_ONBOARDING.md](PILOT_ONBOARDING.md)); it does not
revoke views or functions.

With column grants, the check passing and `ROSIE_REQUIRE_COLUMN_GRANTS=true`, PostgreSQL
refuses any statement that reaches a masked column of a catalog table, including joins
`USING` it (42501), which MySQL does not. When the check passes, as of its last run: the
database refuses every statement that reaches a masked column of a catalog table, for the
account and every role it is a member of, including through the table's partitions,
inheritance children and parents, through any view that depends on the column, and through
`SECURITY DEFINER` functions the account may execute (each held until it is listed in
`"reviewedRoutines"`, your statement). Not covered: copies of the values
elsewhere, what unmasked columns reveal, what a reviewed view really exposes, objects created
since the last check. No independent review of 1.31.0 has happened yet.

A `SECURITY DEFINER` function anywhere in the database that `PUBLIC` may execute holds
every context on that database until it is listed in `"reviewedRoutines"` or its `EXECUTE`
is taken from `PUBLIC`. On a managed PostgreSQL whose platform installs such functions, run
`npm run grants -- --verify` early, review the functions it names, and list the ones that
return no masked value. The listing is your statement; the check cannot see through it.

At startup, and every 15 minutes after, Rosie checks this role and refuses to answer
(`securityHold`) if it finds any of:

- the role is named `postgres`, `admin` or `root`, whatever it may do;
- a role attribute beyond login (SUPERUSER, CREATEDB, CREATEROLE, REPLICATION, BYPASSRLS);
- membership in `pg_read_server_files`, `pg_write_server_files`,
  `pg_execute_server_program` or `pg_write_all_data`;
- INSERT, UPDATE, DELETE or TRUNCATE on any table, counting privileges inherited through
  roles and through `PUBLIC`;
- CREATE on any schema or on the database.

On PostgreSQL 14 and older, `PUBLIC` can create objects in the `public` schema by default,
and the check reports it. The fix is `REVOKE CREATE ON SCHEMA public FROM PUBLIC;`.

## What the engine does differently on PostgreSQL

**Session.** Every connection of the read-only pool is set to `statement_timeout`
(`AI_QUERY_TIMEOUT_MS`), `default_transaction_read_only = on` and a `search_path` holding
only `DB_SCHEMA`. Statements are sent through the extended query protocol, which accepts
exactly one statement per call.

**Functions are allowlisted.** PostgreSQL has thousands of built-in functions and extensions
add more, so a list of forbidden ones cannot be complete. Only the functions an analytical
SELECT needs are accepted: aggregates, window functions, common text, numeric and date
functions, `COALESCE`, `NULLIF`, `CAST`. Anything else is refused with
`Function not allowed: NAME`. `REPEAT`, `LPAD` and `RPAD` are left out on purpose (one call can
allocate a gigabyte). The list is `ALLOWED_FUNCTIONS` in
`server/ai-assistant/dialect/postgres.js`.

**Rejected outright:**

| Construct | Why |
|---|---|
| Functions and `VALUES` in `FROM` (`generate_series`, `dblink`, ...) | They do not appear in the table list, so the table allowlist never sees them |
| Any name starting with `pg_`, and `information_schema` | System catalogs are always on the search path |
| Schema-qualified tables, columns and functions | The allowlist is a set of bare names inside one schema |
| Backslashes, dollar-quoting (`$$...$$`), `$1` | Parser and server can disagree on where a string ends |
| `UNION`, `INTERSECT`, `EXCEPT`, `SELECT INTO`, locking reads | Not needed for analytics; each widens what a statement can reach. `UNION ALL` alone is accepted, inside a derived table or `WITH` body, with every branch validated as a statement of its own (`AI_UNION_ALL=false` refuses it too) |
| Casts to `regclass`, `oid` and other system types | Catalog lookups in disguise |
| `current_user`, `session_user`, `current_schema` and bare `user` | Session information |
| `NATURAL JOIN` | It joins on every shared column name without naming one. Refused on every engine since 1.31.0 ("NATURAL JOIN is not allowed: join with ON and name the columns") |
| `OFFSET` without `LIMIT` | The SQL parser reads it differently from the server |

**Mixed-case identifiers.** A model often writes `FROM Invoice` for a table created as
`"Invoice"`, which PostgreSQL folds to `invoice` and fails. Before a statement runs, every
table name is replaced by its exact catalog spelling, every qualifier by the alias it refers
to, and every unquoted column that matches one mixed-case catalog column or output alias is
quoted with that spelling. Lower-case schemas are left as written. The allowlist and the
masked-column list ignore case, so Rosie's validator treats `"Email"`, `email` and `"EMAIL"`
as the same masked column, wherever it finds it in a statement. Up to 1.29.2 it did not find
a double-quoted column in some positions (inside `SUBSTRING`, `POSITION`, `LIKE`, `~`,
`AT TIME ZONE` and others) or after `IS DISTINCT FROM`; see the advisory in
[SECURITY_OVERVIEW.md](SECURITY_OVERVIEW.md). Column grants on the read-only role are what
makes the server refuse a masked column whatever the validator misses.

## Limits

- One schema per deployment. Two tables whose names differ only by case are not supported.
- `IS DISTINCT FROM` and `IS NOT DISTINCT FROM` are refused since 1.30.0. Write `=`, `<>`,
  `IS NULL` or `COALESCE` instead.
- Two table names or aliases in one statement that differ only by letter case, by an accent
  or by a look-alike letter are refused (all engines since 1.30.0).
- Text comparison is accent-sensitive: `ILIKE 'Luis'` does not match `Luís`, where MySQL's
  default collation would. `lookup_values` therefore finds fewer near-matches than on MySQL.
- The function allowlist is fixed in code; there is no configuration to extend it yet.
- Driver types: `BIGINT` and `COUNT(*)` arrive as numbers, `NUMERIC` as text (as `DECIMAL`
  does on MySQL), `DATE` as `YYYY-MM-DD`, `TIMESTAMP` without time zone is read as UTC.
- A project prompt can write `{{SQL_DIALECT}}` where it names the engine; it is replaced by
  `MySQL`, `PostgreSQL` or `SQL Server`, so one context directory can serve more than one.

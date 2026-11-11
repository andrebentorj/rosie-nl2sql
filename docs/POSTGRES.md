# PostgreSQL

Rosie runs on PostgreSQL as well as MySQL and MariaDB. The engine is selected with one
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
DB_ENGINE_DATABASE=rosie     # database for Rosie's own tables (audit, feedback, certificates)
DB_ENGINE_SCHEMA=public      # schema for those tables inside it (default: public)
DB_RO_USER=rosie_readonly
DB_RO_PASSWORD=...
```

Create the engine tables from `db/schema.postgres.sql` (or let `npm run init` do it), and
generate the catalog as usual: `DB_DIALECT=postgres npm run generate-catalog`.

## The read-only role

```sql
CREATE ROLE rosie_readonly LOGIN PASSWORD '...';
GRANT CONNECT ON DATABASE sales TO rosie_readonly;
GRANT USAGE ON SCHEMA public TO rosie_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO rosie_readonly;
```

At startup, and every 15 minutes after, Rosie checks this role and refuses to answer
(`securityHold`) if it finds any of:

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
| `UNION`, `INTERSECT`, `EXCEPT`, `SELECT INTO`, locking reads | Not needed for analytics; each widens what a statement can reach |
| Casts to `regclass`, `oid` and other system types | Catalog lookups in disguise |
| `current_user`, `session_user`, `current_schema` and bare `user` | Session information |
| `NATURAL JOIN`, `OFFSET` without `LIMIT` | The SQL parser reads them differently from the server |

**Mixed-case identifiers.** A model often writes `FROM Invoice` for a table created as
`"Invoice"`, which PostgreSQL folds to `invoice` and fails. Before a statement runs, every
table name is replaced by its exact catalog spelling, every qualifier by the alias it refers
to, and every unquoted column that matches one mixed-case catalog column or output alias is
quoted with that spelling. Lower-case schemas are left as written. The allowlist and the
masked-column list ignore case, so `"Email"`, `email` and `"EMAIL"` are all refused when the
column is masked.

## Limits

- One schema per deployment. Two tables whose names differ only by case are not supported.
- Text comparison is accent-sensitive: `ILIKE 'Luis'` does not match `Luís`, where MySQL's
  default collation would. `lookup_values` therefore finds fewer near-matches than on MySQL.
- The function allowlist is fixed in code; there is no configuration to extend it yet.
- Driver types: `BIGINT` and `COUNT(*)` arrive as numbers, `NUMERIC` as text (as `DECIMAL`
  does on MySQL), `DATE` as `YYYY-MM-DD`, `TIMESTAMP` without time zone is read as UTC.
- A project prompt can write `{{SQL_DIALECT}}` where it names the engine; it is replaced by
  `MySQL` or `PostgreSQL`, so one context directory can serve both.

# Installing Rosie in a new project

> The quick start below takes a database URL and a model key to a running assistant in a few
> minutes. The manual steps after it are the reference for what the quick start does, and for
> installations that need to do each step by hand.
>
> **Mental model:** Rosie is a **standalone microservice**, not a library. You run **one
> Rosie backend instance per project/database**, pointed at a **context dir** (all domain
> knowledge) and at that project's **database**. The host app's front end only loads a Web
> Component that talks to this backend.

```
Host app (Angular/React)                Rosie backend (Node, 1 instance per project)
   │  <rosie-widget>  ──HTTPS──>  /api/assistant/*
   │                                     ├─ LLM (OpenAI-compatible tool-use loop)
   └─ rosie-widget.js (loaded              ├─ context dir      (ROSIE_CONTEXT_DIR)
      from the backend)                     └─ its database (read-only pool, SELECT-only)
```

---

## Quick start

You need Node.js 22 or newer, a MySQL, PostgreSQL or SQL Server database (MariaDB is not supported), and a model API key.

```bash
git clone <repository url> rosie && cd rosie
npm ci
export OPENAI_API_KEY=sk-...
npm run init -- --url postgres://user:password@host:5432/dbname --name "My Project" \
                --create-ro-user --create-engine-tables
npm run doctor
npm start
```

`npm run init` reads the schema and writes everything Rosie needs:

| It produces | What it is |
|---|---|
| `schema-catalog.json` | Tables, columns, relationships, descriptions and metrics. Also the allowlist of what can be queried. |
| `masked-columns.json` | Columns that look like personal data, which Rosie refuses to read or to describe to the model. Take them away from the read-only account too (Step 2). |
| `system-prompt.txt`, `project.json` | The assistant's instructions and its welcome message and suggestions. |
| `verified-queries.json`, `clarification-rules.json` | Example questions with SQL that was run against your database, and rules for ambiguous questions. |
| `eval/questions/` | A first question bank, with expected results computed from your data. |
| `.env` | The server configuration (written as `.env.rosie-init` when a `.env` already exists). |

It then prints the configuration to connect an AI client over MCP.

| Option | Effect |
|---|---|
| `--url <url>` | `mysql://...`, `postgres://...` or `mssql://...`. The scheme selects the engine. To keep the password out of your shell history, put the URL in the `DB_URL` environment variable instead. |
| `--name <name>` | Project name. Default: the database name. |
| `--create-ro-user` | Creates `rosie_readonly`, a user that can only SELECT, with a generated password written to the `.env`. It grants table by table, and on tables with masked columns only the unmasked columns, from the catalog and the masks (Step 2). **Writes to your database**; needs an account allowed to create users. If it cannot, the statements to run are printed. On PostgreSQL and SQL Server the statements run in one transaction, so a failure leaves nothing behind; on MySQL they stop at the first failure, and the wizard says the account exists but is incomplete, lists what ran and what did not, and prints the `DROP USER` to clean up. Without a catalog it does not create the user. Without this option no user is created. |
| `--create-engine-tables` | Creates Rosie's four engine tables (`rosie_audit_log`, `rosie_answer_certificate`, `rosie_feedback`, `rosie_rate_limit`). **Writes to your database.** Add `--engine-db <name>` to keep them in a separate database. Without this option no table is created. |
| `--samples` | Lets sample values of unmasked columns go to the model provider while the context is generated. Descriptions and enum codes come out better. Off by default. |
| `--context <dir>` | Where to write the context. Default: `contexts/my-project`. |

Without `--url`, `npm run init` asks the same things interactively.

What the quick start leaves for a person:

1. **Review the generated context.** It is written by a model from your schema. Start with
   personal data, which is handled in two levels:
   - **Masked** (`masked-columns.json`): direct identifiers and secrets, such as e-mail,
     phone, street address, postal code, national ids, birth date, salary and credentials.
     Rosie refuses statements that read them and does not describe them to the model. For
     the database to refuse them as well, give the read-only account no privilege on them
     (Step 2). `--create-ro-user` grants that way, from the catalog and the masks it has
     just written; when you change the masks afterwards, change the grants too.
   - **Left readable, listed under "NOT masked, review" in the wizard's report**: attributes
     that describe a person without identifying one alone, such as name, city, state,
     country, company and job title. Questions like "customers by country" need them. Add
     any of them to `masked-columns.json` if your policy requires it.

   Then read the prompt and the metric definitions.
2. **Decide how callers authenticate.** The generated `.env` already carries a random
   `ROSIE_API_TOKEN`; callers send it as `Authorization: Bearer <token>`. Keep it on the
   server side (your backend or a proxy adds the header), or switch to the users' own JWTs
   (`docs/IDENTITY.md`).
3. **Finish the `.env` for production.** The generated file is a development configuration
   in three places:
   - `ROSIE_CORS_ORIGIN` is written as `http://localhost:4200`. Set it to the origin of the
     pages that embed the widget (Step 5).
   - `ROSIE_HANDLE_SECRET` is not written. Without it answer handles stop working at every
     restart and are not accepted by another replica.
   - `TRUST_PROXY` is commented out. Behind a reverse proxy, set it, or every user shares one
     rate-limit bucket (Step 6d).
4. **Measure it.** There are two question banks, read by two commands:
   - `npm run eval` reads `eval/questions/*.json`, the bank the wizard generates. It lives in
     `test/`, so it needs a checkout of the repository. It costs a few cents.
   - `npm run evaluate` reads `eval/bank.json`, a list of `{ "id", "question", "sql" }` that
     you write, plus the approved queries without parameters (`docs/EVALUATION.md`). It is in
     the Docker image, and it is the bank that `npm run loadtest`, `npm run check-context`
     and the `rosie_evaluation_*` metrics use. The wizard does not write this file.

`npm run doctor` checks the whole installation at any time and prints the fix next to each
problem. `npm run mcp:config` prints the MCP client configuration again.

### With Docker, without cloning

`<image>` below is the image you were given access to, for example
`ghcr.io/andrebentorj/rosie:<version>`. Log in to the registry first with the token you
created (`docker login ghcr.io`), or load the file you received
(`docker load < rosie-<version>.tar.gz`).

```bash
mkdir -p rosie-context rosie-env
docker run --rm -v "$PWD/rosie-context:/context" -v "$PWD/rosie-env:/out" \
  -e OPENAI_API_KEY -e DB_URL='postgres://user:password@host:5432/dbname' \
  <image> node scripts/rosie-init.js --context /context --env-dir /out --name "My Project"

docker run -d -p 3001:3001 --env-file rosie-env/.env -v "$PWD/rosie-context:/context:ro" \
  -v rosie-logs:/app/logs <image>
```

**Keep the `rosie-logs` volume.** `/app/logs` holds the audit file, the copy of every
question and answer that survives a database outage. Without a volume it lives inside the
container and is thrown away when the container is replaced, for example on every upgrade.
`npm run doctor` (run inside the container) warns when the directory is not a mounted volume.

The two directories must be writable by the container's user (uid 1000). The context lands
in `./rosie-context` and the configuration in `./rosie-env/.env`, already pointing at
`/context`. The image does not carry `test/`, so the wizard skips the dry run of
`eval/questions/` there and `npm run eval` is not available; `npm run evaluate` is. With the
context mounted read-only, set `ROSIE_EVAL_DIR` to a writable directory for its results. Run
the server with that directory mounted and the variables from the generated env file (see
the comment at the top of the `Dockerfile`).

---

## Manual installation (reference)

The SQL in Steps 2 and 3 is written for MySQL. For PostgreSQL the same steps,
with their statements, are in `docs/POSTGRES.md`; for SQL Server in `docs/SQLSERVER.md`.

### Prerequisites

- Node.js **22 or newer** (24 LTS recommended; `.nvmrc` pins 24 and `.npmrc` refuses older runtimes because Node 18 is end-of-life) and access to the Rosie repository.
- Access to **the host app's database** (MySQL, PostgreSQL or SQL Server: host, port, schema; MariaDB is not supported) with permission to create a user
  and a database for Rosie's engine tables.
- An **LLM API key** (OpenAI, or any OpenAI-compatible provider; see Step 7 for Claude).
- The **origin** of the host app's front end (e.g. `https://app.example.com`), needed for CORS.

---

## Step 1. Get the code

```bash
git clone <repository url> rosie  &&  cd rosie
npm ci
cp .env.example .env
```

The server does not start yet, and that is intended: several modules `throw` at load time
when `ROSIE_CONTEXT_DIR` or `DB_DATABASE` is missing (fail-closed). The first start is at
the end of Step 5.

---

## Step 2. Create the read-only database user (DO NOT SKIP)

Rosie runs **every** AI query through a dedicated pool that must have **SELECT only**. At
startup, and every 15 minutes after, a permission probe
(`server/ai-assistant/security/db-permission-check.js`) checks this: if the user has any
privilege beyond SELECT, the **server stays up but `/ask`, `/export`, `/explain` and
`/certify` return 503**. This is the #1 install mistake.

Grant it **table by table**, and **column by column** on the tables that have masked
columns: the unmasked columns only. Do not grant the whole schema. Rosie's validator refuses
statements in which it finds a masked column, but independent reviews have found ways around
that check more than once; a column the account has no privilege on is refused by the
database, whatever the statement looks like. The tables are those of `schema-catalog.json`
and the masked columns those of `masked-columns.json` (Step 4); if you do not have them yet,
create the user now and come back to the grants after Step 4d.

MySQL:

```sql
CREATE USER 'rosie_readonly'@'%' IDENTIFIED BY '<strong password>';
-- A table with no masked column: the whole table.
GRANT SELECT ON <my_schema>.invoice      TO 'rosie_readonly'@'%';
GRANT SELECT ON <my_schema>.invoice_line TO 'rosie_readonly'@'%';
-- A table with masked columns: its unmasked columns only, no table-level grant.
GRANT SELECT (customer_id, first_name, last_name, city, country)
      ON <my_schema>.customer TO 'rosie_readonly'@'%';
-- Nothing beyond SELECT, no GRANT ... ON <my_schema>.*, no roles.
```

**A narrower grant does not remove a broader one.** With `GRANT SELECT ON <my_schema>.*` in
place, a column grant adds nothing, and `REVOKE SELECT (email) ON <my_schema>.customer` fails
with "There is no such grant defined", because the privilege is held on the database, not on
the column. Remove the broad grant first (`REVOKE SELECT ON <my_schema>.* FROM
'rosie_readonly'@'%';`), then grant as above. `SHOW GRANTS FOR 'rosie_readonly'@'%';` must
show no line `ON <my_schema>.*` or `ON *.*`. Keep the account out of roles too: since 1.31.0
Rosie probes the denied columns again after `SET ROLE ALL`, so a granted, default or
mandatory role that reads a masked column is found and named in a `masked_column_readable`
warning (a role that reads no masked column no longer fails the check).
On MySQL a revoke does not reach connections that are already open. Restart Rosie after
changing grants to apply them at once. Without a restart, since 1.31.0, the next check (every
15 minutes) replaces the connections of the read-only pool when its result changes
(`read_only_pool_drained`); until then an open connection keeps the old privilege.

**MySQL does not check column privileges on join columns of `USING` or `NATURAL JOIN`.** As
an account with column grants, MySQL 8.4 runs
`SELECT COUNT(*) FROM customer c JOIN (SELECT 'x' AS email) d USING (email)`, which tests a
guessed value against a masked column without reading it. Rosie's validator refuses a
`USING` on a masked column at any depth and refuses `NATURAL JOIN` on every engine, and on
MySQL it is the only layer that does. PostgreSQL refuses these statements in the database.

**MySQL shows histograms to anyone who can read the table.** `information_schema.COLUMN_STATISTICS`
is filtered by access to the table, not to the column: if someone ran
`ANALYZE TABLE ... UPDATE HISTOGRAM` on a masked column, a column-granted account reads real
values of it there. The check reports such a column as readable
(`mysql_histogram_reads_masked`), and `npm run grants` prints
`ANALYZE TABLE <db>.<table> DROP HISTOGRAM ON <column>;` for it. A histogram created after the
last check is readable until the next one; Rosie's validator refuses `information_schema`.

**On MySQL, views and stored routines run with their definer's rights.** A view or a function
created with `SQL SECURITY DEFINER` (the default) returns a masked column to any account
allowed to use it, whatever its column grants. The check lists them (below), and a definer
routine the account may execute holds the context until you list it in `"reviewedRoutines"`;
`npm run grants` does not take them away. A statement Rosie runs cannot name either: a view outside the catalog is
refused by the table allowlist, and since 1.31.0 a stored function is refused by the function
allowlist ("Function not allowed: NAME"). The database itself does not stop them under column
grants, so on MySQL the validator is the layer there. Do not grant the account `EXECUTE` on
routines.

On MySQL, each read-only session also removes `ANSI_QUOTES`, `NO_BACKSLASH_ESCAPES`,
`PIPES_AS_CONCAT` and `ANSI` from its own `sql_mode` (since 1.30.0), because they change how
the server reads a statement that Rosie's validator has already checked. It reads the mode
back, and a session that still has one of them is refused: no statement runs on it. Your
server's global mode is not changed. MariaDB is not supported.

`npm run init -- --create-ro-user` writes these grants for you, per engine, from the catalog
and the masks. Its report prints the statements; the `.env` hint points to that report. It
grants what the files say at the time it runs: when you mask another column later, take
that column's privilege away by hand, or run `npm run grants` (below).

An account created by an earlier version, or by hand with a schema-wide grant, keeps that
grant until you remove it. Running `--create-ro-user` again does not migrate it: it grants
nothing to a read-only user you named as existing, and when `rosie_readonly` already exists
its `CREATE` fails and the wizard only prints the statements. `npm run -s grants >
grants.sql` (since 1.31.0) connects as the read-only account, reads what it holds and prints
the `REVOKE` and `GRANT` statements that bring it to this form; it never runs them. The
runbook is [PILOT_ONBOARDING.md](PILOT_ONBOARDING.md). By hand, remove the broad grant, then
grant as above:

```sql
-- MySQL
REVOKE SELECT ON <my_schema>.* FROM 'rosie_readonly'@'%';
-- PostgreSQL
REVOKE SELECT ON ALL TABLES IN SCHEMA public FROM rosie_readonly;
-- SQL Server
REVOKE SELECT ON SCHEMA::dbo FROM rosie_readonly;
ALTER ROLE db_datareader DROP MEMBER rosie_readonly;
```

The table-by-table grant also keeps Rosie's own tables out of reach: that is why Step 3 puts
them in another database. A table created later is not readable until you grant it.

A column privilege does not reach through a view, which reads with its owner's rights (on
PostgreSQL a `SECURITY DEFINER` function does the same). If a view in the catalog selects a
masked column, mask the view's column too, or leave the view out of the catalog. Since 1.31.0 every catalog table that is a view (or a materialized view)
must carry `"reviewedView": true` in `schema-catalog.json`, the operator's statement that
someone looked at it and it exposes no masked value; until it does, the check below reports
the context as "masked by Rosie only" (`view_not_reviewed`). Rosie does not check what was
reviewed. `npm run generate-catalog` keeps the field.

**PostgreSQL: partitions and inheritance children have their own grants.** A partition or a
table created with `INHERITS` is a separate table with its own privileges, so a column grant
on the parent does not cover a table grant left on a child (a schema-wide grant gives one to
every partition), and a `SELECT` on a parent reads its children's rows. The check walks
partitions, inheritance children and parents of every masked table
(`postgres_inheritance_reads_masked`), and `npm run grants` prints the `REVOKE` for each.

**SQL Server: column grants do not stop `PIVOT`.** SQL Server 2022 checks column permissions
on the aggregated column and the `FOR` column of a `PIVOT`, not on the columns it passes
through. With `SELECT` on any one column of a table,
`SELECT p.* FROM t PIVOT (COUNT(c) FOR c IN ([0])) AS p` returns every other column of `t`,
a column `DENY` included. Rosie's validator refuses `PIVOT` and `UNPIVOT`, and on SQL Server
it is the only layer that does. So with the column grants above, SQL Server is "masked by
Rosie only". For the database to refuse the masked columns there, the account must have no
permission at all on a table that has masked columns: give it a view that selects only the
unmasked columns, mark the view `"reviewedView": true` in the catalog, and grant the account
`VIEW DEFINITION` on the view and on the table so the check can follow the view. In the
catalog, the table then lists its masked columns only (its other columns are read through
the view). Details in [SQLSERVER.md](SQLSERVER.md).

### Check what the read-only account can read

To check by hand, connect as the read-only account and run, for each masked column:

```sql
SELECT email FROM customer WHERE 1 = 0;
```

It must fail with a permission error (MySQL 1142 or 1143, PostgreSQL 42501, SQL Server 229
or 230), and an ordinary question on the same table must still work.

Rosie runs the same check itself, as the read-only account, each context on a new connection
of its own: at startup, with every permission check after (every 15 minutes,
`ROSIE_PERMCHECK_INTERVAL_MS`, in the HTTP server and in both MCP processes) and, in the HTTP
server, after every context reload. Since 1.31.0 a context is reported as
`masked by the database` only when all of the following hold:

- every masked column is denied to the account, and to every role it can switch to:
  - PostgreSQL: every role the account is a member of, directly or through a chain, including
    `INHERIT FALSE` memberships, is asked about the column, and membership in
    `pg_read_all_data` is reported on its own; `PUBLIC` grants are seen by the probe;
  - MySQL: the denied columns are probed again after `SET ROLE ALL`, so a granted, default or
    mandatory role that reads one is found;
  - SQL Server: the column permission (`HAS_PERMS_BY_NAME`), `db_datareader`, `IMPERSONATE` on
    another user or login, `IMPERSONATE ANY LOGIN`, `CONTROL SERVER`, and a `PIVOT` probe per
    masked column (see "What each engine can give" below);
- on PostgreSQL, no partition, inheritance child or parent table of a masked table lets the
  account (or a role it is a member of) read the masked column;
- on MySQL, no masked column has a histogram in `information_schema.COLUMN_STATISTICS`
  (read as the account, and again under `SET ROLE ALL` when it has granted roles);
- every unmasked column of every catalog table is readable (an account that is denied
  everything, or points at the wrong database, is not a safe one);
- every catalog table that is a view carries `"reviewedView": true`;
- no view, function or procedure the account can use returns a masked column, unless it is a
  reviewed catalog view, and none of them is beyond what the check can read:
  - PostgreSQL: views and materialized views in any schema that depend on a masked column
    (views over views included), and every `SECURITY DEFINER` function or procedure outside
    `pg_catalog` and `information_schema` the account or one of its roles may execute
    (`EXECUTE` goes to `PUBLIC` by default);
  - MySQL: every definer view outside the catalog the account can see, and every routine
    with `SQL SECURITY DEFINER` the account may execute according to `SHOW GRANTS` (its
    granted roles included; `EXECUTE` or `ALL` at the global, database or routine level). A
    routine whose only `EXECUTE` is a routine-level grant held by a role does not appear in
    `information_schema.ROUTINES`, so its security type cannot be seen: it is judged from the
    grant and held until it is listed. The grants are read on every check.
    `SQL SECURITY INVOKER` ones are skipped: they run with the account's own privileges. A
    view is shown clean only when its definition is visible (`SHOW VIEW`), reads with
    confidence and every qualifier on a masked column name resolves to a table;
  - SQL Server: every view, table function, scalar function and procedure the account may
    use, followed when the account has `VIEW DEFINITION` on it;
  - a view that reads a masked column is reported as `view_reads_masked`; a view whose
    reads cannot be established (definition not visible or not read with confidence) is
    `object_not_verifiable` on MySQL, and makes SQL Server `not verified on SQL Server`;
  - routines (PostgreSQL `SECURITY DEFINER` functions and procedures, MySQL definer
    routines) are never passed on their text: a definer routine the account may execute is
    always `object_not_verifiable`, unless it is listed in `"reviewedRoutines"` at the top of
    `schema-catalog.json` (below). Its body is still read to REPORT: a body that names a
    masked table and one of its masked columns is `view_reads_masked` (kind `function` or
    `procedure`), and that keeps the context out of `masked by the database` even when the
    routine is listed. A `SECURITY DEFINER` function anywhere in the database that `PUBLIC`
    may execute (the default) holds every context on that database until it is listed or
    its `EXECUTE` is taken away; on a managed PostgreSQL that installs such platform
    functions, run `npm run grants -- --verify` early and list the ones you have reviewed;
- every mask entry is one the catalog and the effective mask set agree on (an entry for a
  table or column the catalog does not list, a key written twice, a column the database does
  not have, or a probe that failed for another reason all keep the context out of it).

**`"reviewedRoutines"`.** A top-level array in `schema-catalog.json`, next to `tables` and
`relationships`:

```json
"reviewedRoutines": [ "public.order_total", "reporting.month_key" ]
```

Each entry is `schema.name` (on MySQL `database.name`), matched the way the engine resolves
a name:

- **PostgreSQL:** an unquoted part folds to lower case and a double-quoted part keeps its
  case (`""` for a quote inside it). So `rosie.f_clean`, `rosie.F_Clean` and
  `rosie.F_CLEAN` all mean the function `f_clean`, and `rosie."F_Clean"` means `F_Clean`,
  a different function. A quoted schema works the same way (`"Sales".f_y`).
- **MySQL:** names compare without letter case; backquotes are accepted
  (`` `Chinook`.`f_q` ``).

A listing is by name. It covers every overload of that exact name, those present now and any
created later, and it is your statement that each returns no masked value. Rosie does not
check that statement and cannot see through it, which is the expected cost of the rule; it
only refuses a listed routine whose body visibly names a masked column. Review a new
overload before it is granted. `npm run generate-catalog` keeps the array, and
`npm run check-context` reads the entries with the same parser as the check ("Reviewed
routines": fails when it is not a list, or an entry does not have exactly two parts or has an
unclosed quote; warns on a name listed twice under the engine's rule).

The result, per context:

| Result | `mode` | Meaning |
|---|---|---|
| `masked by the database` | `database` | Everything above holds. A statement that reaches a masked column is refused by your database, whatever Rosie's own checks miss, except the engine limits below and what the check cannot see (the end of this section) |
| `masked by Rosie only` | `rosie` | Anything else: a masked column the account can read (directly, through a role, a PostgreSQL partition or inheritance relative, a MySQL histogram, or on SQL Server through `PIVOT`), an unmasked column it cannot read, an unreviewed catalog view, a view or routine returning a masked column or one the check cannot read, a mask entry that could not be verified. `reasons` says which |
| `not verified on SQL Server` | `mssql` | SQL Server only: no masked column was found readable, but something the check needs could not be established there, for example an object the account may use but cannot see into (no `VIEW DEFINITION`), cross-database ownership chaining being on, or a `PIVOT` probe that could not be written. The detail is in `masked_column_not_verified` lines and under `unverified` |
| `no masked columns` | `none` | The context masks nothing |
| `not checked` | `unchecked` | The check could not run, for example because the database could not be reached (the counts are then `0`), or has not run yet (the counts are then `null`). An unreachable database gives this result, not `masked by Rosie only`, and no per-column warning |

`reasons` lists why a context is not `masked by the database`, as stable codes:

| Code | Meaning |
|---|---|
| `masked_readable` | The account, or a role it can switch to, can read a masked column |
| `sqlserver_pivot_reads_masked` | SQL Server: a `PIVOT` read a masked column the column permission denies (listed under `readable` with the role `PIVOT`) |
| `postgres_inheritance_reads_masked` | PostgreSQL: a partition, inheritance child or parent of a masked table lets the account read the masked column (role `partition <schema>.<table> (as <role>)`, `inheritance child ...` or `parent table ...`) |
| `mysql_histogram_reads_masked` | MySQL: a masked column has a histogram the account can read (role `histogram`) |
| `masked_not_verified` | A masked column, or a mask entry, could not be verified |
| `object_not_verifiable` | PostgreSQL and MySQL: a definer routine the account may execute and that is not in `"reviewedRoutines"`, or a MySQL definer view whose reads could not be established. Listed under `unverified` with status `object_not_verifiable`, no table or column, and a `detail` naming the object and what to do; not counted in `total` |
| `unmasked_unreadable` | An unmasked catalog column cannot be read by the account |
| `view_not_reviewed` | A catalog table is a view and has no `"reviewedView": true` |
| `view_reads_masked` | A view, function or procedure the account can use reads a masked column and is not a reviewed catalog view (`views.exposures`, with `kind`) |
| `views_not_checked` | The views could not be read; never taken as a pass |
| `sqlserver_not_verified` | The reason of the `not verified on SQL Server` result |
| `not_checked` | The check could not run |

Each state also carries `limits`: what the database does not stop on that engine even in
`masked by the database`. They never change the result.

| Code | Engine | What the database does not stop |
|---|---|---|
| `mysql_join_using_natural` | MySQL | A masked column used as a join column of `USING` or matched by `NATURAL JOIN` is not checked for privilege: guessed values can be tested against it, several in one statement. Only Rosie's validator refuses these statements |
| `mysql_column_statistics` | MySQL | A histogram created on a masked column after the last check is readable in `information_schema.COLUMN_STATISTICS` until the next check reports it. Rosie's validator refuses `information_schema` |
| `mssql_pivot` | SQL Server | A `PIVOT` returns every column it passes through, whatever their permissions, on a table the account can read any column of. Only Rosie's validator, which refuses `PIVOT` and `UNPIVOT`, stops it. When the probe reads a masked column, the context is also `masked by Rosie only` |

#### What each engine can give

All about the database account Rosie runs questions with, when the check passes, as of its
last run (default every 15 minutes). This is what the evidence supports today:

- **PostgreSQL.** The database refuses every statement that reaches a masked column of a
  catalog table, for the account and every role it is a member of, including through the
  table's partitions, inheritance children and parents, through any view that depends on the
  column, and through `SECURITY DEFINER` functions the account may execute (each one held
  until it is listed in `"reviewedRoutines"`, which is your statement). Not covered:
  copies of the values elsewhere, what unmasked columns reveal, what a reviewed view really
  exposes, objects created since the last check.
- **MySQL.** The database refuses a statement that reads a masked column by name; the check
  also reports histograms on masked columns and definer views and routines the account can
  use (a definer routine the account may execute holds the context until it is listed in
  `"reviewedRoutines"`). The database does not stop a `USING` or `NATURAL` join on a masked column
  (`mysql_join_using_natural`), a histogram created between checks
  (`mysql_column_statistics`), or a definer routine; Rosie's validator refuses those (any
  call to a function that is not a listed built-in included) and is a required layer there.
  A definer view whose definition cannot be read with confidence is reported
  `object_not_verifiable`, not passed as clean. On MySQL the database does not refuse
  whatever Rosie's own checks miss: there it is the two layers together.
- **SQL Server.** With column grants: no database claim (a `PIVOT` reads the masked columns of
  any table the account can read one column of, so the result is `masked by Rosie only`).
  With no permission on the masked tables and a reviewed catalog view: the database refuses
  every statement on those tables; what the view shows is the operator's statement
  ([SQLSERVER.md](SQLSERVER.md)). Common Criteria mode and application roles are untested.

No independent review of 1.31.0 has happened yet.

On every engine the check does not cover: what a reviewed view really exposes
(`reviewedView` is your statement, not Rosie's), objects in a schema or database the account
cannot see (it cannot use them either), anything created between two checks (the interval is
the detection lag), copies of a masked value in another column or table, and what the
unmasked columns reveal. A context with no masked columns is `no masked columns` and skips
the view and routine checks: nothing in it is masked, so nothing is checked. Masked column
names are Rosie's job, not the database's: the grant stops values, and Rosie's own
description of the schema leaves the names out.

#### Where the result is shown

`GET /api/assistant/health` (and `/health` of `npm run mcp:http`) is public, so it shows the
result without column names:

```json
"columnGrantsRequired": false,
"columnGrants": [
  { "projectId": "my-project", "mode": "rosie", "label": "masked by Rosie only",
    "total": 4, "denied": 3, "readable": 1, "unverified": 0,
    "reasons": [ "masked_readable" ], "limits": [ "mysql_join_using_natural", "mysql_column_statistics" ],
    "checkedAt": "2026-11-17T09:00:00.000Z", "hold": false }
]
```

`total` is the number of masked columns, `denied` how many the database refused, `readable`
how many the account can read, `unverified` how many could not be verified, `reasons` and
`limits` the codes above, and `hold` whether this context refuses requests
(`ROSIE_REQUIRE_COLUMN_GRANTS`, below). The column names are shown in three places:

- the server log, one line per finding, the first time and then only when the result
  changes: `masked_column_readable` (with the role, when it is readable through one),
  `masked_column_not_verified`, `unmasked_columns_unreadable` (with the columns),
  `view_not_reviewed`, `view_reads_masked`, `object_not_verifiable` and `views_not_checked`.
  The summary line is
  `column_grants_checked`, with `reasons` and `limits` (`column_grants_not_checked` when it
  could not run);
- `npm run doctor`, in the check "Masked columns denied by the database", with the lines
  "Catalog views reviewed", "Views reading masked columns" and "Not stopped by the database
  on this engine". A warning; with `ROSIE_REQUIRE_COLUMN_GRANTS` on, the line
  `ROSIE_REQUIRE_COLUMN_GRANTS` fails exactly when the gate holds (`masked by Rosie only`,
  `not verified on SQL Server`, `not checked`) and is `ok` ("requests are served") otherwise;
- `GET /api/assistant/admin/overview`, under `status.maskedColumns` (each readable column with
  the roles that reach it, each unverified one with the reason, `reasons`, `limits`, `views`,
  `required` and `hold`), and the "Masked columns" card of the console.

`npm run grants -- --verify` runs the same check once, from the command line, and lists the
views and routines that return a masked column ([PILOT_ONBOARDING.md](PILOT_ONBOARDING.md)).

Under column grants the database hides the masked columns from the account, so the
schema-drift comparison cannot see them. `npm run doctor` reports them as "hidden by the
database" and does not fail on them.

When the database refuses a statement for lack of privilege, Rosie logs
`db_permission_denied` as a security event, with the error's code and never its text (on SQL
Server since 1.31.0 also when one statement is refused on several columns at once). Under
column grants this is usually a statement that reached a masked column and that Rosie's own
check did not refuse.

**Since 1.31.0 the rule is stricter than in 1.30.0.** A deployment that 1.30.0 reported as
`masked by the database` now reads `masked by Rosie only` when a catalog table or column is
not readable by the account (including one the database no longer has), when a catalog view
has no `"reviewedView": true`, when a view, function or procedure the account can use returns
a masked column or cannot be read by the check (PostgreSQL, MySQL and SQL Server), when a
PostgreSQL partition or inheritance relative keeps a broader grant, when a MySQL masked
column has a histogram, or, on SQL Server, when a `PIVOT` reaches a masked column. That is
the intended report, not a regression.

#### Changing the grants

When the result of a context changes, and after every reload (including the reloads of the
console's approve, describe and dismiss actions), Rosie replaces the connections of the
read-only pool (`read_only_pool_drained`, with `reason` `mode_changed` or `reload`): a
statement already running finishes on its connection, a request waiting for a connection is
served by a new one, and the old connections are closed after that. A drain fails no
request.
So a restart is no longer needed after a grant change; it only makes the change take effect
at once instead of at the next check. Until then, on MySQL, a connection opened before a
`REVOKE` keeps the old privilege.

#### Requiring the database grant: `ROSIE_REQUIRE_COLUMN_GRANTS`

Off by default, and never switched on by an upgrade. Off, Rosie reports and keeps answering
whatever the result. With `ROSIE_REQUIRE_COLUMN_GRANTS=true`, a context serves requests only
while its last result is `masked by the database` or `no masked columns` (a context that
masks nothing has nothing for the database to deny). Anything else holds it:

| Code | When |
|---|---|
| `E_COLUMN_GRANTS_NOT_CHECKED` | Not checked yet, the check could not run, or a reloaded context is still being checked |
| `E_COLUMN_GRANTS_NOT_ENFORCED` | `masked by Rosie only` |
| `E_COLUMN_GRANTS_SQLSERVER_NOT_VERIFIED` | `not verified on SQL Server` |

- `POST /ask`, `/export`, `/explain` and `/certify` answer `503` with
  `{ "error", "code", "securityHold": true, "columnGrantsHold": true }`. `/explain` runs no SQL
  and is held anyway, like the permission hold. The message names no column.
- The MCP tools that run SQL or hand out answers (`ask_data`, `query_metric`, `certify_sql`,
  `run_approved_query`, `list_available_data`, `explain_answer`, `export_answer`) return a
  tool error with the same message and code, before any quota is used.
- At startup every request is held until the first check has run; the HTTP server listens
  meanwhile, and the MCP servers wait for it before they connect or listen.
- `/project-info` shows `securityHold: true` with the hold's `securityCode` and
  `securityMessage`, so the widget shows the hold. `/metrics` `rosie_security_hold` is `1`
  during either hold. `/health` carries `columnGrantsRequired: true` and `hold` per context;
  the `/health` of `npm run mcp:http` answers `status: "security_hold"`.
- A reload does not interrupt service: the context in service keeps serving until the new
  one has been checked, then the new one takes over with its own result (held or not).
  `POST /admin/reload` answers `"pending": true` and logs `context_reload_pending`. A reload
  is never refused because of the check. A periodic check that finishes for a context no
  longer in service, or while a reload waits for its own check, is not stored
  (`column_grants_state_dropped`, `reason` `replaced` or `reload_pending`).
- The hold is decided when a request arrives. A request already past the gate finishes if
  the result changes while it runs; with the 15-minute check interval that adds nothing to
  the detection lag.
- An HTTP server that serves several projects gates `/ask`, `/export`, `/explain` and
  `/certify`, and reports `/health` and `/project-info`, on its default context. The MCP
  servers gate each session's own context.
- Log events: `column_grants_required` at startup, `column_grants_hold` (error) when a
  context becomes held, `column_grants_hold_released` when it serves again, and
  `column_grants_request_refused` (at most one line a minute per context, with the count).

The flag does not make the database layer stronger than the engine allows: the limits above
still hold, and on SQL Server with column grants it holds every request (`masked by Rosie
only`), which is the honest result.

The server does not issue `GRANT` or `REVOKE`. The check covers the read-only account only.
`npm run init`, `npm run generate-catalog` and the engine pool use other credentials
(`DB_USER`), which read whatever you granted them.

---

## Step 3. Create the engine database, its tables and the main user

Rosie writes its own records through the **main pool** (not the read-only one), into four
tables:

| Table | Holds |
|---|---|
| `rosie_audit_log` | One row per question, export and explanation |
| `rosie_answer_certificate` | The certificate of each answer that carried one |
| `rosie_feedback` | Ratings (up or down) |
| `rosie_rate_limit` | Quota counters, used only with `ROSIE_RATE_LIMIT_STORE=database` |

Create a database for them, apply the schema file **inside that database**, and create the
account of the main pool. Run this with an account that may create databases and users:

```bash
mysql -u <admin_user> -p -e "CREATE DATABASE rosie"
mysql -u <admin_user> -p rosie < db/schema.sql
```

```sql
CREATE USER 'rosie_app'@'%' IDENTIFIED BY '<strong password>';
-- Always: the engine inserts its records and reads them back (review queue, reports).
GRANT SELECT, INSERT ON rosie.* TO 'rosie_app'@'%';
-- Only with ROSIE_RATE_LIMIT_STORE=database: the shared counters are updated and old
-- windows are removed.
GRANT UPDATE, DELETE ON rosie.rosie_rate_limit TO 'rosie_app'@'%';
FLUSH PRIVILEGES;
```

`npm run retention -- --confirm` removes old rows from the three log tables, so the account
that runs it needs DELETE on them. It can be `rosie_app`, or a separate account used only
for that command (set `DB_USER` and `DB_PASSWORD` in the environment of the command):

```sql
GRANT SELECT, DELETE ON rosie.rosie_audit_log          TO '<retention_user>'@'%';
GRANT SELECT, DELETE ON rosie.rosie_answer_certificate TO '<retention_user>'@'%';
GRANT SELECT, DELETE ON rosie.rosie_feedback           TO '<retention_user>'@'%';
```

`npm run doctor` checks that the tables are there. It does not check these grants: with
`ROSIE_RATE_LIMIT_STORE=database` and no UPDATE on `rosie_rate_limit`, doctor reports the
shared counters as present while each process counts in its own memory and logs
`rate_limit_store_unavailable`.

`npm run generate-catalog` (Step 4c) and `npm run init` read the schema of `<my_schema>`
with the main account (`DB_USER`). Give it `SELECT` there, or run those two commands with
another account by setting `DB_USER` and `DB_PASSWORD` in their environment.

**Legacy layout.** With `DB_ENGINE_DATABASE` unset, the engine expects its tables inside
`DB_DATABASE`, next to the business data. Then the grant of Step 2 lets the read-only user
read them, and `npm run generate-catalog` lists them in the catalog like any other table.
If you keep that layout, grant the read-only user SELECT table by table instead of on
`<my_schema>.*`, and remove the four `rosie_` tables from `schema-catalog.json` after each
generation. The separate database avoids both.

---

## Step 4. Build the host app's context dir

> **⚡ Fast path: `npm run init`.** The context-bootstrap wizard automates Steps 4 and 5:
> it interviews you (project identity, governance, DB credentials, language),
> introspects the schema, and uses the same LLM Rosie runs on to generate **every**
> context file, including LLM-enriched catalog semantics, `verified-queries.json`
> validated against the real engine validator AND executed on your DB, and an eval
> bank whose expectations are computed from actual data. It also writes the `.env`
> (never overwriting an existing one without confirmation). The read-only user (Step 2)
> is created only with `--create-ro-user`, and the engine tables (Step 3) only with
> `--create-engine-tables` or when you accept the wizard's question (default no). The
> grants of Step 3 and the widget (Step 6) stay manual. **Review everything it
> generates.** LLM output is provisional; the final report lists exactly what to check.
> The manual path below remains fully supported.

Create a directory outside the repo (e.g. in the host app's repo or a mounted volume). The fastest
start is to **copy the template** under `contexts/example/`:

```
my-rosie-context/
├── project.json            # required
├── system-prompt.txt       # required: all of the host app's domain knowledge
├── schema-catalog.json     # required: schema + security allowlist (generated, Step 4c)
├── masked-columns.json     # personal-data columns to block (see 4d)
├── verified-queries.json   # optional: few-shot question→SQL pairs, approved queries
├── clarification-rules.json# optional: deterministic disambiguation chips
├── row-policies.json       # optional: per-user or per-tenant row isolation
├── eval/bank.json          # optional, recommended: bank of `npm run evaluate`
└── eval/questions/*.json   # optional: bank of `npm run eval`
```

```bash
cp -r contexts/example/ /path/to/my-rosie-context
# then edit each file for the host app (the templates use a generic users/orders schema)
```

The template ships the first six files. The others are described in the contract.

> The full contract (optional fields, semantics) is in `contexts/example/README.md`.

### 4a. `project.json`
Name, welcome message, suggestion chips (surfaced via `GET /project-info`):
```json
{
  "name": "Acme",
  "fullName": "Acme Analytics",
  "description": "One-line description of the host app's data.",
  "welcomeMessage": "Hi! I'm Rosie. Ask me about the host app's data.",
  "suggestions": ["How many items per status?", "Top 10 most recent records"]
}
```

### 4b. `system-prompt.txt`
Free text with the host app's business rules (what each table means, how the domain is phrased, named
metrics). The engine **automatically appends** a catalog summary and the CLARIFY protocol.
Don't repeat the schema here; focus on domain meaning.

### 4c. `schema-catalog.json`: generate it (it's also the security allowlist)
Every table/column a query can reference must be here; a query touching anything **outside**
the catalog is rejected. A table with an empty column list **aborts startup** (fail-closed).
Generate it directly from the host app's database. The command reads `DB_DIALECT`,
`DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`, `DB_DATABASE` and `ROSIE_CONTEXT_DIR`, so
fill those lines of the `.env` first (they are part of Step 5):

```bash
npm run generate-catalog
# or target a specific output / schema:
node scripts/generate-catalog.js --out /path/to/my-rosie-context/schema-catalog.json --database my_schema
```

This reads the database's metadata and fills the **structural** fields (tables, columns +
types, primary keys, foreign-key relationships, approximate row counts). Run again over an
existing catalog, it merges. What it keeps from the previous file:

- on a column: `description`, `synonyms`, `enumValues`, and since 1.24.0 `measure`,
  `profile` and `profileNote`;
- on a table: `description`, `synonyms`, `domain`, `tags`, `maskedColumns`, and since 1.24.0
  `promptHidden`, `entity`, and a `primaryKey` written by hand when the database declares none, as
  long as every column it names is still there;
- since 1.24.0, relationships written by hand or imported (`npm run import-semantic`) that
  the database does not declare as foreign keys, as long as both ends still exist;
- every root-level key, `metrics` and `maskedColumns` among them.

What it does **not** keep is the choice of tables. Every table and view of the schema is
written to the catalog, including the ones you removed on purpose, and the catalog is the
allowlist. After every regeneration, remove them again (`--no-views` leaves all views out),
or review what changed against the previous copy before putting it in service:

```bash
npm run check-context -- --context /path/to/my-rosie-context --base /path/to/previous-copy
```

> Enrich the high-value bits: synonyms for how users actually speak, `enumValues` for status
> columns, and named `metrics`. A bare structural catalog works but a thin one **silently
> breaks valid questions** the model can't map.

### 4d. `masked-columns.json`
PII columns the validator refuses to return (also blocks `SELECT *` on the table). Give the
read-only account no privilege on them (Step 2), so the database refuses them too:
```json
{ "users": ["email", "national_id", "phone"] }
```

The server starts without this file, and then nothing is masked. `npm run check-context`
warns when nothing is masked and fails when a mask names a column that does not exist.

---

## Step 5. Finalize the backend `.env`

```bash
# ─── Server ───
ROSIE_PORT=3001
# The origin of the pages that embed the widget. Several: comma-separated. "*" is refused.
ROSIE_CORS_ORIGIN=https://app.example.com
# Behind a reverse proxy: the number of proxy hops (Step 6d).
TRUST_PROXY=1

# ─── Auth (production: a long random string) ───
ROSIE_API_TOKEN=<strong random token>
# Signs answer handles. Set it (16 characters or more), or handles die with every restart.
ROSIE_HANDLE_SECRET=<another long random value>

# ─── LLM ───
OPENAI_API_KEY=sk-...
AI_MODEL=gpt-6.1-terra                       # the default; gpt-6-luna is cheaper and answers far fewer right: docs/MODELS.md
AI_ASSISTANT_ENABLED=true
# AI_BASE_URL=                               # set to swap providers: see Step 7

# ─── DB main (used ONLY for Rosie's own tables, Step 3) ───
# DB_DIALECT=mysql                           # or postgres, mssql
DB_HOST=...  DB_PORT=3306  DB_USER=rosie_app  DB_PASSWORD=...  DB_DATABASE=<my_schema>
DB_ENGINE_DATABASE=rosie                     # the database of Step 3

# ─── DB read-only (all AI queries: the Step 2 user) ───
DB_RO_HOST=...  DB_RO_PORT=3306
DB_RO_USER=rosie_readonly  DB_RO_PASSWORD=...  DB_RO_DATABASE=<my_schema>

# ─── the host app context dir (ABSOLUTE path) ───
ROSIE_CONTEXT_DIR=/absolute/path/to/my-rosie-context
```

(One variable per line in the real file; they share lines here to keep the block short.)

`DB_DATABASE` stays the name of the business schema even though the main pool connects to
`DB_ENGINE_DATABASE`: the catalog generator and the wizard read it. The variables that only
`.env.example` explains are listed in `docs/OPERATIONS.md`, "Settings reference".

Start and confirm a clean boot:
```bash
npm run doctor
npm start
curl localhost:3001/api/assistant/health        # {"status":"ok", ...}
curl localhost:3001/api/assistant/project-info   # should reflect the host app's project.json
```
If `project-info` returns `securityHold: true`, the Step 2 probe failed: review the
read-only user's privileges. With `ROSIE_REQUIRE_COLUMN_GRANTS=true` it can also mean the
masked-column check did not pass; `securityCode` then starts with `E_COLUMN_GRANTS_` (Step 2,
"Requiring the database grant"). If `health` says `degraded` with `schemaBlockedTables` after the
first question, the engine tables are not where `DB_ENGINE_DATABASE` points (Step 3).

---

## Step 6. Plug the widget into the host app's front end

Rosie ships a self-contained Web Component. **Two lines of HTML, no file copy.** What it
shows once it is on the page is in [WIDGET.md](WIDGET.md).

### 6a. Load the bundle from the backend (recommended)

The backend serves the exact widget it was deployed with at `GET /api/assistant/widget.js`
(public, ETag, 5-minute cache). Through the same proxy the API already uses, that is:

```html
<script src="/api/assistant/widget.js"></script>

<rosie-widget api-url="/api/assistant"></rosie-widget>
```

Every backend deploy now carries its widget fixes to every host automatically; nothing to
re-copy, nothing to forget. `/project-info` reports `widgetVersion`, and an embedded copy
that drifts from it logs a console warning.

### 6b. Alternative: copy the bundle into the host app's assets (legacy)
```bash
cp widget/dist/rosie-widget.js  <host-repo>/src/assets/rosie/rosie-widget.js
```
> ⚠️ **Deploy gotcha:** a copied widget is **not** updated by the backend. A stale copy
> strands features **and security fixes**. If you must copy (e.g. a CSP that forbids the
> API origin for scripts), re-copy on every Rosie release and watch for the
> `[rosie-widget] … backend ships …` console warning. Prefer 6a.

> 🔒 **Security topology (mandatory in production):** do **NOT** put the shared
> `ROSIE_API_TOKEN` in the `api-token` attribute. It places the token in every logged-in
> user's DOM, where anyone can extract it and query the whole allowlisted schema directly
> with `curl`, bypassing the host app entirely (CORS restrains browsers, not scripts). The
> supported topology with the shared token is: the host app reverse-proxies
> `/api/assistant/*` **same-origin** and its proxy injects
> `Authorization: Bearer <ROSIE_API_TOKEN>` server-side, where the secret stays.
>
> The one supported use of `api-token` in production is the **user's own JWT**, when JWT
> verification is configured (`docs/IDENTITY.md`): that token belongs to the user and says
> only what that user may see. The widget cannot tell the two kinds of token apart, so it
> logs the same console warning whenever the attribute is set. With the shared token, treat
> that warning as a deploy blocker. With a user's JWT it is expected.

### 6c. Push the logged-in user (after auth resolves)
```js
const widget = document.querySelector('rosie-widget');
widget.setAttribute('user-id',   user.identifier);   // audit/personalization only
widget.setAttribute('user-name', user.displayName);  // NEVER authorization
```

With a verified JWT these two attributes are ignored: the token says who the user is.

### 6d. Routing `api-url` → backend
`api-url="/api/assistant"` assumes the host app **reverse-proxies** `/api/assistant/*` to the Rosie
backend. With the shared token this is **the only supported production topology**: it
exposes a single domain and keeps `ROSIE_API_TOKEN` server-side (the proxy adds the
`Authorization` header). Nginx sketch:

```nginx
location /api/assistant/ {
    proxy_pass         http://rosie-backend:3001/api/assistant/;
    proxy_set_header   Authorization "Bearer <ROSIE_API_TOKEN>";
    proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_buffering    off;    # the widget asks for a stream of progress events
}
```

Two settings on the Rosie side go with the proxy:

- **`TRUST_PROXY`.** Quotas are counted per client address. Behind a proxy that address is
  the proxy's unless the proxy sends `X-Forwarded-For` (above) and Rosie is told to trust
  it: `TRUST_PROXY=1` for one proxy hop. Without it **all users share one bucket**, 100
  questions an hour by default for everybody together. Never set it to `true`: that trusts
  whatever the client sends and makes the limits spoofable.
- **`ROSIE_CORS_ORIGIN`.** The origin check runs on every request that carries an `Origin`
  header, through the proxy or not. A request from an origin that is not listed is refused
  with HTTP 500 and a `request_error` line in the log saying `CORS: origin not allowed`.
  Keep the variable at the origin of the host app's pages.

Pointing `api-url` at the backend's absolute URL (with `ROSIE_CORS_ORIGIN` set) is for
development, or for the JWT setup above. With the shared token it forces the token into the
page to authenticate, which is exactly the exposure the proxy topology exists to prevent.

---

## Step 7. (Optional) Swap the LLM (OpenAI → Claude)

The provider is **pluggable via config**. The engine speaks an OpenAI-compatible wire
format, so any OpenAI-compatible endpoint works with **no code change**. To run on Claude,
use Anthropic's OpenAI-compatible endpoint:

```bash
AI_BASE_URL=https://api.anthropic.com/v1/
OPENAI_API_KEY=sk-ant-...                    # the var name stays OPENAI_API_KEY
AI_MODEL=claude-haiku-5-5                     # low cost/latency; or claude-sonnet-5-5
```

Leave `AI_BASE_URL` empty for the OpenAI default. The same mechanism works for OpenRouter,
LiteLLM, or any compatible gateway: set `AI_BASE_URL` + matching key/model. After switching
providers, run `npm run test:provider` (one real call with a tool) and re-run the eval
(Step 8) to confirm accuracy on the host app's question bank.

---

## Step 8. Validate before the demo

```bash
npm run doctor                                      # the installation, end to end
npm run check-context                               # the context files (no DB, no LLM)
node test/security/battery.js                       # security battery (no DB/LLM)
node -r dotenv/config test/eval/run.js --dry-run    # validate eval/questions/*.json
node -r dotenv/config test/eval/run.js              # real NL2SQL eval (costs LLM $)
```

Final checklist:
- [ ] `health` and `project-info` OK; no `securityHold`.
- [ ] Read-only user has SELECT only (Step 2) and cannot read the engine tables.
- [ ] `health` reports `masked by the database` for every context with masked columns (Step 2,
      "Check what the read-only account can read"), or you know why not: on SQL Server with
      column grants the result is `masked by Rosie only` because of `PIVOT`.
      `npm run grants -- --verify` exits 0 only in the first case.
- [ ] Engine tables created where `DB_ENGINE_DATABASE` points, grants in place (Step 3).
- [ ] Catalog covers every table/column the target questions touch, and no table you meant
      to leave out.
- [ ] Widget loaded from `/api/assistant/widget.js` (or re-copied) and responding in the host
      app; user-id/name flowing.
- [ ] `ROSIE_CORS_ORIGIN`, `TRUST_PROXY` and `ROSIE_HANDLE_SECRET` set (Steps 5 and 6d).

---

## Upgrading

1. **Get the new version.** With Docker, pull the new tag and change the tag in your
   deployment. From a checkout, `git pull` (or check out the release tag) and `npm ci`. The
   context directory and the `.env` are outside both and are not touched.
2. **Run the schema file of your engine again** on the engine database (`db/schema.sql`,
   `db/schema.postgres.sql` or `db/schema.mssql.sql`). It creates only what is missing, so
   it is safe on every upgrade. It adds tables, never columns. One column was added after
   the first installations: `cross_check` on `rosie_answer_certificate`, written whenever
   the second derivation runs (on by default). An installation older than that column needs
   it once: `ALTER TABLE rosie_answer_certificate ADD COLUMN cross_check TEXT NULL;` (on SQL
   Server: `ADD cross_check NVARCHAR(MAX) NULL`).
3. **Check the grants** when the release adds a table (Step 3). 1.10.0 added
   `rosie_rate_limit`.

   **Upgrading to 1.30.0.** No grant is changed by the upgrade. A read-only account with a
   schema-wide grant (what `--create-ro-user` wrote before 1.30.0) can read the masked
   columns: at the first start Rosie logs one `masked_column_readable` warning per masked
   column, `/health` shows `masked by Rosie only` and the console's "Masked columns" card
   turns red. That is the intended report, not a regression. Remove the broad grant and
   grant table by table, with the statements in
   [Step 2](#step-2-create-the-read-only-database-user-do-not-skip), then restart Rosie.
   Running `--create-ro-user` again does not do it for you. Also run `npm run check-context`
   (step 5): since 1.30.0 it fails on two mask keys that differ only by case, a key written
   twice, a metric over a masked column and context text that writes a masked
   `table.column`. Since 1.31.0, `npm run -s grants > grants.sql` prints the statements that
   migrate the account, for you to review and run ([PILOT_ONBOARDING.md](PILOT_ONBOARDING.md)).

   **Upgrading to 1.31.0.** No grant and no setting is changed by the upgrade;
   `ROSIE_REQUIRE_COLUMN_GRANTS` stays off. What you may notice:
   - The masked-column check is stricter (Step 2): a context 1.30.0 reported as
     `masked by the database` can now read `masked by Rosie only` because a catalog column
     is not readable by the account, a catalog view has no `"reviewedView": true`, a
     view or routine the account can use returns a masked column or cannot be read by the
     check (PostgreSQL `SECURITY DEFINER` functions and MySQL definer routines the account
     may execute, until listed in the new `"reviewedRoutines"`; MySQL definer views; SQL
     Server views, functions and procedures), a PostgreSQL partition or inheritance relative
     of a masked table keeps a broader grant, a MySQL masked column has a histogram, or, on
     SQL Server, a `PIVOT` reaches a masked column. `reasons` in `/health` says which. With
     `ROSIE_REQUIRE_COLUMN_GRANTS=true` each of these holds requests.
   - SQL Server is checked like the other engines now. With column grants it reports
     `masked by Rosie only` (`sqlserver_pivot_reads_masked`), where 1.30.0 said
     `not verified on SQL Server`.
   - `NATURAL JOIN` is refused on every engine ("NATURAL JOIN is not allowed: join with ON and
     name the columns"). `PIVOT` and `UNPIVOT` are refused by name on SQL Server.
   - On MySQL, table names, qualifiers and `WITH` names are rewritten to the catalog's
     spelling before a statement runs, as on PostgreSQL; column names are left as written. A
     table written in another letter case no longer fails with a permission error (1142).
     An alias keeps the spelling it was declared with, and a qualifier takes a table's
     catalog spelling only when it names an unaliased table. The MySQL prompt gains a
     one-sentence `## SQL dialect` section, so the prompt text, and any prompt hash you
     compare, changes for every MySQL context.
   - On SQL Server, `COUNT(*)`, `COUNT(1)` and `EXISTS (SELECT 1 ...)` over one catalog table
     that has masked columns are rewritten to count its key, so they also run under column
     grants. The SQL shown with the answer reads `COUNT([Customer].[CustomerId])` instead of
     `COUNT(*)`; the number is the same. The key is used only when the catalog marks it
     `"nullable": false` and it is not masked (generated catalogs do); `COUNT(NULL)` is never
     rewritten.
   - MySQL functions are allowlisted, as they already were on PostgreSQL and SQL Server: about
     190 built-ins a reporting `SELECT` uses. Any other function is refused with "Function not
     allowed: NAME", and a schema-qualified call (`db.f(...)`) is refused on every engine. Left
     out on purpose, so a statement that used them now fails: `REPEAT`, `SPACE`,
     `EXPORT_SET`, `MAKE_SET` (one call can allocate a very large string); hashing, encryption, compression and encodings
     (`MD5`, `SHA`, `SHA1`, `SHA2`, `CRC32`, `AES_ENCRYPT`, `AES_DECRYPT`, `COMPRESS`,
     `UNCOMPRESS`, `TO_BASE64`, `FROM_BASE64`, `HEX`, `UNHEX`); `UUID`, `UUID_SHORT` and the
     other functions that read server or session state (`CHARSET`, `COLLATION`, `DEFAULT`,
     `VALUES`, ...); `SOUNDEX`, `WEIGHT_STRING`; XML and spatial functions, `JSON_TABLE` and
     the JSON functions that modify a document; network and UUID helpers. The full list is
     in `server/ai-assistant/dialect/mysql.js`.
   - `DB_DIALECT=mariadb`, and a `mariadb://` URL in `npm run init`, stop with "MariaDB is not
     supported" instead of failing every connection later.
   - Two names that differ only by letter case are refused with a message that says what
     was written twice: "two aliases", "two table names" or "a table name and an alias".
   - After a grant change a restart is no longer needed: the next check drains the
     read-only pool when its result changes.
4. **Restart every replica**, then run `npm run doctor`. A widget loaded from
   `/api/assistant/widget.js` follows the backend by itself; a copied one must be copied
   again (6b).
5. **Check the context against the new version** with `npm run check-context`, and measure
   with `npm run evaluate` if you keep a bank: a new version can change how a question is
   answered.

To go back, deploy the previous tag.

---

## Summary: what changes per project vs what's fixed

| Per project                               | Fixed (Rosie engine)                    |
|-------------------------------------------|-----------------------------------------|
| context dir (prompt, catalog, masks)      | SQL validator, filters, tool-use loop   |
| `.env` (DB, CORS, token, model, base URL) | routes `/api/assistant/*`                |
| read-only user + engine tables            | widget (same bundle for all hosts)      |
| widget loaded from /api/assistant/widget.js (or re-copied) | security battery                         |

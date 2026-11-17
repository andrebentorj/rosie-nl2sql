# Installing Rosie in a new project

> The express install below takes a database URL and a model key to a running assistant in
> three steps. "Choosing a masking mode" says what express gives and what strict mode adds,
> for companies that need the database itself to deny the masked columns. The manual steps
> after them are the reference for what the wizard does, and for installations that need to
> do each step by hand.
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

## Express install

For evaluations, quick pilots and smaller companies that can let Rosie's read-only account
read the whole database. You need Node.js 22 or newer, a MySQL, PostgreSQL or SQL Server
database (MariaDB is not supported) with an account allowed to create users, and a model API
key. Get the code first:

```bash
git clone <repository url> rosie && cd rosie
npm ci
export OPENAI_API_KEY=sk-...
```

1. **Have the database running.** Start its container if it runs in one, or use the URL of
   the database you already have.
2. **Run the setup wizard and accept the defaults.**

   ```bash
   npm run init -- --url postgres://user:password@host:5432/dbname \
                   --create-ro-user --create-engine-tables
   ```

   It reads the schema, writes the context and the `.env`, creates the read-only account
   `rosie_readonly` with `SELECT` on the whole schema (on MySQL the whole database), and
   creates Rosie's own tables in a separate database, `rosie`, which it creates and names in
   `DB_ENGINE_DATABASE`, so the express account cannot read them (Step 2). The masking mode
   is express.
3. **Start Rosie.**

   ```bash
   npm start            # http://localhost:3001
   ```

Then review what the wizard wrote, personal data first ("What the wizard leaves for a person",
below), and run `npm run doctor`, which prints the fix next to each problem.

In express mode Rosie's validator refuses statements that reach a masked column, and the
database account can read those columns. If your data needs the database to deny them too,
use strict mode, now or later.

## Choosing a masking mode

Set with `ROSIE_MASKING_MODE` in the `.env` (`express` or `strict`, read without letter case;
unset or empty means `express`). Any other value stops the server at startup
(`masking_mode_invalid`), and the MCP servers too. The wizard asks right after it connects,
and writes the answer (`--masking express|strict` answers it; `--yes`, or `--url`, without it
means express).

| | Express (default) | Strict |
|---|---|---|
| Who it is for | Smaller companies, evaluations, quick pilots | Companies with sensitive personal data, or a security review to pass |
| The read-only account | `SELECT` on the whole schema (MySQL: the whole database). Still read-only | `SELECT` table by table, and on tables with masked columns only the unmasked columns |
| What protects masked columns | Rosie's validator, and the rules that keep them out of what is described to the model | The database, verified at startup and every 15 minutes, with Rosie's validator on top |
| What Rosie shows | One info line per context at startup; the console card reads "Masked columns protected by Rosie's validator"; `npm run doctor` does not fail on "masked by Rosie only" | A warning per masked column the database does not deny; the console card is green when the context is "masked by the database"; `npm run doctor` fails what the gate holds |
| Requests when the database does not deny the masked columns | Answered (`ROSIE_REQUIRE_COLUMN_GRANTS` defaults to `false`) | Refused with `503` (`ROSIE_REQUIRE_COLUMN_GRANTS` defaults to `true`) |

What each mode lets you claim:

- **Express.** Rosie refuses statements that reach a masked column and leaves masked columns
  out of what it describes to the model. This is Rosie's own reading of SQL; independent
  reviews have found ways around it before (see
  [Corrections](SECURITY_OVERVIEW.md#corrections)), and the database account can read those
  columns.
- **Strict.** The database denies this account every masked column Rosie knows how to
  reach, on these engines, with named gaps. The check probes the masked columns (also
  through roles, PostgreSQL partitions and inheritance relatives, and MySQL histograms) and
  works as an allowlist: anything the account can read or execute beyond the catalog tables
  and the objects you marked as reviewed holds the context. Within what it enumerates, an
  object it does not recognise is held, not passed. The review markers (`reviewedView`,
  `"reviewedViews"`, `"reviewedRoutines"`) are your statement, not something Rosie checks, and
  some paths are not objects at all (the named gaps). MySQL keeps named exceptions where Rosie's
  validator is the required layer, and SQL Server reaches it only with the reviewed-view
  design, since column grants there do not stop `PIVOT`. The details are in
  [What each engine can give](#what-each-engine-can-give).

In both modes the account is read-only, the validator, the masks and the permission probe
run as before, and the column-grant check runs and is shown in `/health` and the console. An
explicit `ROSIE_REQUIRE_COLUMN_GRANTS` always wins over the mode's default, in either
direction.

## Moving from express to strict

1. Set `ROSIE_MASKING_MODE=strict` in the `.env`.
2. Print the statements that bring the read-only account to table and column grants:
   `npm run -s grants > grants.sql`. It reads what the account holds and never runs anything.
3. Have the database administrator review and apply `grants.sql`
   ([PILOT_ONBOARDING.md](PILOT_ONBOARDING.md#applying-the-script)).
4. Restart Rosie.
5. Confirm with `npm run grants -- --verify`. It must say "masked by the database" (exit 0).
   Until it does, strict mode holds every request of that context.

On SQL Server, column grants end at "masked by Rosie only" because of `PIVOT`, so strict mode
there needs the reviewed-view design ([SQLSERVER.md](SQLSERVER.md)); otherwise stay in
express. To go back to express, set `ROSIE_MASKING_MODE=express` and restart; the grants can
stay as they are.

## The setup wizard in detail

`npm run init` reads the schema and writes everything Rosie needs:

| It produces | What it is |
|---|---|
| `schema-catalog.json` | Tables, columns, relationships, descriptions and metrics. Also the allowlist of what can be queried. |
| `masked-columns.json` | Columns that look like personal data, which Rosie refuses to read or to describe to the model. In strict mode the read-only account is denied them too (Step 2). |
| `system-prompt.txt`, `project.json` | The assistant's instructions and its welcome message and suggestions. |
| `verified-queries.json`, `clarification-rules.json` | Example questions with SQL that was run against your database, and rules for ambiguous questions. |
| `eval/questions/` | A first question bank, with expected results computed from your data. |
| `.env` | The server configuration, `ROSIE_MASKING_MODE` included (written as `.env.rosie-init` when a `.env` already exists). |

It then prints the configuration to connect an AI client over MCP. Its final report says
which masking mode was chosen, what it means, and how to change it later
(`ROSIE_MASKING_MODE=strict` plus `npm run grants`).

| Option | Effect |
|---|---|
| `--url <url>` | `mysql://...`, `postgres://...` or `mssql://...`. The scheme selects the engine. To keep the password out of your shell history, put the URL in the `DB_URL` environment variable instead. |
| `--name <name>` | Project name. Default: the database name. |
| `--masking express\|strict` | How masked columns are protected ([Choosing a masking mode](#choosing-a-masking-mode)); `--masking=strict` works too. Default `express`. Without it the wizard asks after it connects, offering the earlier choice on a re-run; with `--url` or `--yes` and no `--masking`, the mode is express. Any other value stops the wizard. Written to the `.env` as `ROSIE_MASKING_MODE`, with a two-line comment. |
| `--create-ro-user` | Creates `rosie_readonly`, a user that can only SELECT, with a generated password written to the `.env`. In express mode it grants `SELECT` on the whole schema (PostgreSQL, SQL Server) or database (MySQL), plus what the engine needs to connect (`CONNECT` and `USAGE` on PostgreSQL). In strict mode it grants table by table, and on tables with masked columns only the unmasked columns, from the catalog and the masks (Step 2), and without a catalog it does not create the user. **Writes to your database**; needs an account allowed to create users. If it cannot, the statements to run are printed. On PostgreSQL and SQL Server the statements run in one transaction, so a failure leaves nothing behind; on MySQL they stop at the first failure, and the wizard says the account exists but is incomplete, lists what ran and what did not, and prints the `DROP USER` to clean up. Without this option no user is created. |
| `--create-engine-tables` | Creates Rosie's four engine tables (`rosie_audit_log`, `rosie_answer_certificate`, `rosie_feedback`, `rosie_rate_limit`). **Writes to your database.** Without this option no table is created, except as below. |
| `--engine-db <name>` | Where Rosie's own tables go. In express mode, a run without questions (`--url` or `--yes`) that has `--create-ro-user` or `--create-engine-tables` and no `--engine-db` puts them in a separate database, `rosie`: the wizard creates it on all three engines, creates the tables there (also with `--create-ro-user` alone) and writes `DB_ENGINE_DATABASE=rosie`. `--engine-db same` (or the business database's own name) keeps them next to your data, where the express account can read them. In strict mode they stay in your database unless you name one. The interactive interview asks, with `rosie` as the default; an earlier answer or `DB_ENGINE_DATABASE` is kept. The final report has one line on where they went. |
| `--samples` | Lets sample values of unmasked columns go to the model provider while the context is generated. Descriptions and enum codes come out better. Off by default. |
| `--context <dir>` | Where to write the context. Default: `contexts/my-project`. |

Without `--url`, `npm run init` asks the same things interactively.

What the wizard leaves for a person:

1. **Review the generated context.** It is written by a model from your schema. Start with
   personal data, which is handled in two levels:
   - **Masked** (`masked-columns.json`): direct identifiers and secrets, such as e-mail,
     phone, street address, postal code, national ids, birth date, salary and credentials.
     Rosie refuses statements that read them and does not describe them to the model. For
     the database to refuse them as well, use strict mode, in which the read-only account
     has no privilege on them (Step 2). In strict mode `--create-ro-user` grants that way,
     from the catalog and the masks it has just written; when you change the masks
     afterwards, change the grants too (`npm run grants`).
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

What else it is granted depends on the masking mode
([Choosing a masking mode](#choosing-a-masking-mode)).

### Express mode: SELECT on the whole schema

```sql
-- MySQL
CREATE USER 'rosie_readonly'@'%' IDENTIFIED BY '<strong password>';
GRANT SELECT ON <my_schema>.* TO 'rosie_readonly'@'%';
-- PostgreSQL
CREATE ROLE rosie_readonly LOGIN PASSWORD '<strong password>';
GRANT CONNECT ON DATABASE <my_database> TO rosie_readonly;
GRANT USAGE ON SCHEMA public TO rosie_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO rosie_readonly;
-- SQL Server
CREATE LOGIN rosie_readonly WITH PASSWORD = '<strong password>';
CREATE USER rosie_readonly FOR LOGIN rosie_readonly;
GRANT SELECT ON SCHEMA::dbo TO rosie_readonly;
-- Nothing beyond SELECT, no roles.
```

This is what `npm run init -- --create-ro-user` grants in express mode. The account can read
every table and column of the schema, masked columns included. What a statement may name is
decided by Rosie: the catalog is the allowlist, and the validator refuses masked columns. On
PostgreSQL `ON ALL TABLES` covers the tables that exist when it runs; on MySQL and SQL Server
the grant also covers tables created later.

**Keep Rosie's own tables out of that schema.** When the schema or database the express
account can read also holds Rosie's engine tables (`rosie_audit_log`,
`rosie_answer_certificate`, `rosie_feedback`, `rosie_rate_limit`), the account can read them
in the database: the questions, statements and answers of every user. Rosie's validator
refuses those tables, because they are not in the catalog, but the database does not. Since
1.32.0 the express wizard puts them in a separate database, `rosie`, by default
(`DB_ENGINE_DATABASE`, Step 3); `--engine-db same` is the explicit way back to the shared
layout. An installation set up earlier with them in the business database keeps that layout
until you move them, or use strict mode, which grants only the catalog's tables.

The column-grant check below still runs. In express mode a context with masked columns is
expected to read "masked by Rosie only", and Rosie logs one info line for it at startup
instead of a warning per column.

### Strict mode: table by table, and column by column on masked tables

Grant it **table by table**, and **column by column** on the tables that have masked
columns: the unmasked columns only. Do not grant the whole schema. Rosie's validator refuses
statements in which it finds a masked column, but independent reviews have found ways around
that check more than once. A column the account has no privilege on is refused by the
database when a statement names it; the paths that reach it another way (roles, partitions,
views, foreign tables, routines) are what the check below looks for, as far as it knows them.
The tables are those of `schema-catalog.json`
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

In strict mode `npm run init -- --create-ro-user` writes these grants for you, per engine,
from the catalog and the masks. Its report prints the statements; the `.env` hint points to
that report. It grants what the files say at the time it runs: when you mask another column
later, take that column's privilege away by hand, or run `npm run grants` (below).

An account created by an earlier version, by the express install, or by hand with a
schema-wide grant, keeps that grant until you remove it. Running `--create-ro-user` again does not migrate it: it grants
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
reviewed. `npm run generate-catalog` keeps the field. Since 1.32.0 views, materialized views
and foreign tables outside the catalog are held too until they are reviewed, on PostgreSQL
whatever columns they select ("Objects held until reviewed", below).

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

This check runs in both masking modes. What follows matters most in strict mode, where
"masked by the database" is the goal; in express mode a context with masked columns
normally reads "masked by Rosie only", and that is expected.

To check by hand, connect as the read-only account and run, for each masked column:

```sql
SELECT email FROM customer WHERE 1 = 0;
```

It must fail with a permission error (MySQL 1142 or 1143, PostgreSQL 42501, SQL Server 229
or 230), and an ordinary question on the same table must still work.

Rosie runs the same check itself, as the read-only account, each context on a new connection
of its own: at startup, with every permission check after (every 15 minutes,
`ROSIE_PERMCHECK_INTERVAL_MS`, in the HTTP server and in both MCP processes) and, in the HTTP
server, after every context reload. A context is reported as `masked by the database` only
when all of the following hold (as of 1.32.0). The list is what the check knows how to look
at; it is not a proof that no other path exists:

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
- since 1.32.0, the account (or a role it can switch to) can read or execute nothing beyond
  the catalog tables and what you marked as reviewed. This is an allowlist: every object the
  check enumerates that is neither a catalog table nor reviewed is "outside" and holds the
  context with one reason, `object_not_verifiable`, whatever it reads and whatever kind it
  is. The state lists them under `views.outside` (at most 200, each with `object`, `kind`,
  `why`) with the total in `views.outsideCount`, and the problem's detail names the first 20.
  What each engine enumerates, and what it deliberately skips:
  - PostgreSQL: every table, partitioned table, view, materialized view and foreign table in
    any schema except the system ones (`pg_catalog`, `information_schema`, `pg_toast*`,
    temporary schemas) on which the account or a member role holds `SELECT` on some column
    and `USAGE` on the schema; every routine it may execute that is `SECURITY DEFINER` (in any
    schema, extensions included), that is granted beyond `PUBLIC` (this is how PostgreSQL
    ships `pg_read_file`, `pg_read_binary_file`, `pg_ls_dir`, `pg_stat_file`, `lo_import`,
    `lo_export` and `dblink_connect_u`, so a direct grant on any of them is held) or that is a
    user function in an untrusted language (`C`, `plpython3u`, ...), when the account also has
    `USAGE` on the routine's schema (one it cannot reach cannot be called); every foreign server with
    `USAGE` and a user mapping for the account, `PUBLIC` or a member role (`dblink` runs any
    statement there as the mapped user); and large objects the account, a member role or
    `PUBLIC` may read. Skipped on purpose: relations and functions that belong to an
    extension (unless `SECURITY DEFINER` or granted beyond `PUBLIC`, or every PostGIS
    database would hold on `spatial_ref_sys`), except a view or materialized view of an
    extension whose rule reads a relation or function outside `pg_catalog`,
    `information_schema` and that extension, which is listed (the views
    `pg_stat_statements` ships stay skipped); sequences; and functions in trusted languages
    (`sql`, `plpgsql`) that run with the caller's privileges. A view created
    `WITH (security_invoker = true)` is allowed: it reads with the account's own privileges,
    so a masked column behind it is refused at run time, and a definer view behind it is
    outside on its own. A materialized view never counts as security_invoker;
  - MySQL: every base table the account holds a privilege on, in any database, that is not a
    catalog table. Only `information_schema` and `performance_schema` are skipped
    wholesale. In the `sys` schema only the table `sys_config` is skipped. The views and
    routines MySQL ships there run as `SQL SECURITY INVOKER` and are allowed like any invoker
    object; any `SQL SECURITY DEFINER` view or routine in `sys` holds the context, whatever its
    definer. Anything the account can read in `mysql` is outside; every definer view outside the catalog, even one that
    reads only unmasked columns; every routine with `SQL SECURITY DEFINER` the account may
    execute according to `SHOW GRANTS` (its granted roles included; `EXECUTE` or `ALL` at the
    global, database or routine level; a routine whose only `EXECUTE` is a routine-level grant
    held by a role is judged from the grant). An `SQL SECURITY INVOKER` view the account reads
    directly is allowed; one reached from a definer view runs as that definer and is read as
    part of it;
  - SQL Server: every object of the current database the login has `SELECT`, `EXECUTE` or a
    column grant on (tables, views, synonyms, table and scalar functions, procedures, CLR and
    aggregate functions, extended procedures); every linked server with data access (the
    login cannot see `sys.linked_logins`, so a mapping cannot be ruled out); every other user
    database the login can open (`master`, `tempdb`, `model` and `msdb` skipped). Only the
    `sys` and `INFORMATION_SCHEMA` schemas are left out. Objects flagged as shipped by
    Microsoft (`is_ms_shipped`, which `sp_MS_marksystemobject` sets on any object) are not
    trusted. Only the database-diagram objects SQL Server creates in `dbo` are skipped when
    flagged, each by its exact name and type: the table `sysdiagrams`, the scalar function
    `fn_diagramobjects`, and the procedures `sp_alterdiagram`, `sp_creatediagram`,
    `sp_dropdiagram`, `sp_helpdiagramdefinition`, `sp_helpdiagrams`, `sp_renamediagram` and
    `sp_upgraddiagrams`;
  - catalog names match the way the engine compares names: exactly on PostgreSQL (a table
    `public.customer` beside the catalog's `"Customer"` is outside); on MySQL as
    `lower_case_table_names` says (0 exactly, 1 or 2 without case); on SQL Server as the
    database collation says (`_CS` or `_BIN` exactly, otherwise without case);
  - catalog views keep their own check (`view_not_reviewed`); a catalog table that is a
    PostgreSQL foreign table needs `"reviewedView": true`;
- what the check still reads, for detail: a view whose dependencies name a masked column is
  reported as `view_reads_masked`, unless it is reviewed (`"reviewedView": true` or
  `"reviewedViews"`), in which case it is listed with `reviewed: true`, accepted and warned
  about by `npm run grants -- --verify`; on SQL Server a synonym whose base is a masked
  catalog table is such an exposure too, and a reviewed synonym is not accepted (a synonym is
  the table). The dependency walk on SQL Server runs only on reviewed objects and catalog
  views. Routines are never passed on their text: their body is read only to REPORT a masked
  column named there as a word (`view_reads_masked`, kind `function` or `procedure`), and
  that keeps the context out of `masked by the database` even when the routine is listed. A
  `SECURITY DEFINER` function anywhere in the database that `PUBLIC` may execute (the
  default) holds every context on that database until it is listed or its `EXECUTE` is taken
  away; on a managed PostgreSQL that installs such platform functions, run
  `npm run grants -- --verify` early. And:
  - a reviewed MySQL view is read for detail only when its definition is visible
    (`SHOW VIEW`); a reviewed SQL Server object and every catalog view are followed through
    their dependencies when the login has `VIEW DEFINITION` on them, and what the walk finds
    is reported whether the object is listed or not;
- every mask entry is one the catalog and the effective mask set agree on (an entry for a
  table or column the catalog does not list, a key written twice, a column the database does
  not have, or a probe that failed for another reason all keep the context out of it). A
  malformed mask configuration does not reach the check: since 1.32.0 it stops the server
  (`E_MASK_CONFIG_INVALID`, Step 4d). A mask file that changes on disk into a malformed one
  after the load is reported as `mask_config_invalid` under `unverified`, never as "no
  masked columns".

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
- **SQL Server** (since 1.32.0, for procedures and functions): names compare exactly;
  `[dbo].[p_totals]` brackets (`]]` for a bracket inside) and double quotes are accepted.

Since 1.32.0 an entry is compared as a (schema, name) pair, never as one dotted string:
`"a.b".c` and `a."b.c"` are two different routines. The list takes every routine the
allowlist holds: PostgreSQL `SECURITY DEFINER` routines, routines granted beyond `PUBLIC`
and untrusted-language functions, MySQL definer routines, SQL Server functions and
procedures. Do not list a file or large-object function (`pg_read_file`,
`pg_read_binary_file`, `pg_ls_dir`, `lo_import`, `lo_export` and the like): it reads whatever
the server's files hold, the masked columns' data files included, so the only safe answer is
to take the grant away.

A listing is by name. It covers every overload of that exact name, those present now and any
created later, and it is your statement that each returns no masked value. Rosie does not
check that statement and cannot see through it, which is the expected cost of the rule. It
reads the body of a listed routine only to REPORT a masked column named there as a word. A
routine that uses `SELECT *`, a whole-row reference (`to_jsonb(t)`, `row_to_json(t)`,
`hstore(t)`, a bare composite, `RETURNS SETOF <table>`) or dynamic SQL can return masked
values without naming one, and the check accepts it; review those by hand.
`npm run grants -- --verify` warns about each (below). Review a new overload before it is
granted. `npm run generate-catalog` keeps the array, and `npm run check-context` reads the
entries with the same parser as the check ("Reviewed routines": fails when it is not a list,
or an entry does not have exactly two parts or has an unclosed quote; warns on a name listed
twice under the engine's rule).

**`"reviewedViews"`** (since 1.32.0). A second top-level array, for the readable objects
outside the catalog that are not routines: tables, views, materialized views, foreign tables
and, on SQL Server, synonyms:

```json
"reviewedViews": [ "reporting.v_sales_by_country", "public.cust_ft" ]
```

Entries are `schema.name` pairs, read like `"reviewedRoutines"` (PostgreSQL folds unquoted
parts to lower case, MySQL compares without letter case, SQL Server compares exactly and
accepts brackets). A catalog table keeps
`"reviewedView": true` instead, which also covers a catalog table that is a PostgreSQL
foreign table. A listing lifts `object_not_verifiable` for that object and is your statement
that it returns no masked value. If its dependencies still show a masked column read, the
check reports it under `views.exposures` with `reviewed: true` and accepts it, as it does for
a reviewed catalog view: the dependency data cannot tell a column the view returns from one it
only filters on, and a reviewed view may filter on a masked column without returning it.
`npm run check-context` validates the array ("Reviewed views") as it does
`"reviewedRoutines"`.

**Review notes.** For every reviewed routine (PostgreSQL, MySQL, SQL Server) and every
reviewed PostgreSQL view or materialized view over a masked table, `npm run grants --
--verify` looks for a star (`SELECT *`, `t.*`; not `count(*)`), a whole-row reference and
dynamic SQL (`EXECUTE`, `EXEC`, `sp_executesql`, `PREPARE`, `OPENROWSET`, `OPENQUERY`,
`OPENDATASOURCE`, `dblink`), and prints one line per object:

```
  warning: the reviewed view reporting.v_export uses a star (*) and a whole-row reference; the check cannot see which columns that returns, so confirm by hand that it returns no masked value (the listing stays your statement)
```

For a reviewed object whose dependencies show a masked read (accepted, as above), it also
prints:

```
  warning: the reviewed view reporting.v_country depends on the masked column(s) Email; it is accepted because it is reviewed, so confirm by hand that it uses them without returning them (the listing stays your statement)
```

A note never changes the result or the exit code, and is not part of `/health`; it is in the
state as `views.reviewNotes`.

**Objects held until reviewed, and how to resolve them.** Since 1.32.0 every object the
account can read or execute beyond the catalog tables keeps a context out of "masked by the
database" (`object_not_verifiable`) until it is reviewed or taken away, whatever it reads.
Many of them passed in 1.31.0:

| Object | Engine | Resolve it by |
|---|---|---|
| A table outside the catalog, including one in another schema or database, a copy table, and Rosie's own engine tables when the account can read them | All | Taking the account's privilege on it away, or reviewing it and listing it in `"reviewedViews"` |
| A view or materialized view outside the catalog, whatever it reads (a sales-by-country report included; a materialized view filled through a function too) | All | Reviewing it and listing it in `"reviewedViews"`, or taking the `SELECT` away. On PostgreSQL a view `WITH (security_invoker = true)` needs nothing |
| A definer view outside the catalog, even one that reads only unmasked columns | MySQL | The same |
| A foreign table | PostgreSQL | The same (`"reviewedView": true` when it is in the catalog) |
| A `SECURITY DEFINER` routine, an untrusted-language function, or a routine granted beyond `PUBLIC` | PostgreSQL | Reviewing it and listing it in `"reviewedRoutines"`, or taking the `EXECUTE` away. For file and large-object functions (`pg_read_binary_file`, `lo_import`, ...) only taking the grant away is right |
| A foreign server with `USAGE` and a user mapping for the account, `PUBLIC` or one of its roles | PostgreSQL | Taking the `USAGE` or the user mapping away (reading an existing foreign table does not need `USAGE` on its server) |
| Large objects the account may read | PostgreSQL | Taking the privilege away |
| A definer routine the account may execute | MySQL | Listing it in `"reviewedRoutines"`, or taking the `EXECUTE` away |
| A function or procedure the account may execute, a static one included | SQL Server | Listing it in `"reviewedRoutines"`, or taking the `EXECUTE` away |
| A synonym | SQL Server | Taking the permission away, or listing it in `"reviewedViews"`. A synonym for a masked catalog table is reported as reading it, and listing does not accept it |
| A linked server with data access | SQL Server | Removing it, or the data access. It cannot be listed |
| Another user database the login can open | SQL Server | Removing the login's user in that database. It cannot be listed |

`npm run grants -- --verify` names each object (the first 20 in its `not verified:` line),
and its `next:` line names the first 10 again: `next: review or revoke the objects the
account can use beyond the catalog tables (<kind> <name>, ... and N more); npm run grants
does not revoke them: ...`, with which list to use, or that servers, databases and large
objects can only be removed. It prints `next: run npm run -s grants, apply what it prints`
only when a reason is one that the grant statements fix, not for outside objects alone. The statements `npm run grants` prints grant catalog tables only, so an
account built from them has nothing outside; they do not take away routine, server,
large-object or other-database access, which `--verify` names for you to remove.

The result, per context:

| Result | `mode` | Meaning |
|---|---|---|
| `masked by the database` | `database` | Everything above holds: the database denies this account every masked column the check knows how to reach, except the engine limits below and the named gaps (the end of this section). It is a statement about what the check enumerates and about your review markers, as of the check's last run, not a proof that no other path exists |
| `masked by Rosie only` | `rosie` | Anything else: a masked column the account can read (directly, through a role, a PostgreSQL partition or inheritance relative, a MySQL histogram, or on SQL Server through `PIVOT`), an unmasked column it cannot read, an unreviewed catalog view, a view or routine returning a masked column or one the check cannot read, a mask entry that could not be verified. `reasons` says which |
| `not verified on SQL Server` | `mssql` | SQL Server only: no masked column was found readable, but something the check needs could not be established there, for example an object the account may use but cannot see into (no `VIEW DEFINITION`), cross-database ownership chaining being on, or a `PIVOT` probe that could not be written. The detail is in `masked_column_not_verified` lines and under `unverified` |
| `no masked columns` | `none` | The context masks nothing. A malformed mask configuration never gives this result: it stops the server (Step 4d) |
| `not checked` | `unchecked` | The check could not run, for example because the database could not be reached (the counts are then `0`), or has not run yet (the counts are then `null`). Since 1.32.0 also when a check does not finish within `ROSIE_COLUMN_GRANTS_CHECK_TIMEOUT_MS` (`column_grants_check_timeout`), when it fails, or, with the gate on, when the read-only pool could not be replaced after a passing check (`read_only_pool_drain_failed`). An unreachable database gives this result, not `masked by Rosie only`, and no per-column warning |

`reasons` lists why a context is not `masked by the database`, as stable codes:

| Code | Meaning |
|---|---|
| `masked_readable` | The account, or a role it can switch to, can read a masked column |
| `sqlserver_pivot_reads_masked` | SQL Server: a `PIVOT` read a masked column the column permission denies (listed under `readable` with the role `PIVOT`) |
| `postgres_inheritance_reads_masked` | PostgreSQL: a partition, inheritance child or parent of a masked table lets the account read the masked column (role `partition <schema>.<table> (as <role>)`, `inheritance child ...` or `parent table ...`) |
| `mysql_histogram_reads_masked` | MySQL: a masked column has a histogram the account can read (role `histogram`) |
| `masked_not_verified` | A masked column, or a mask entry, could not be verified |
| `object_not_verifiable` | Since 1.32.0, the account can read or execute objects outside the allowlist (`views.outside`, `views.outsideCount`), or a reviewed object's behaviour cannot be established. One problem under `unverified` with status `object_not_verifiable`, no table or column, and a `detail` naming the first 20 objects and what to do; not counted in `total`. On SQL Server, without a readable masked column or an exposure, the result is then `not verified on SQL Server`. Logged as the event `object_not_verifiable` |
| `unmasked_unreadable` | An unmasked catalog column cannot be read by the account |
| `view_not_reviewed` | A catalog table is a view and has no `"reviewedView": true` |
| `view_reads_masked` | A view, function or procedure the account can use reads a masked column and is not reviewed (`views.exposures`, with `kind`). A reviewed view whose dependencies show a masked read is listed with `reviewed: true` and does not give this reason |
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

This is the claim of strict mode. In express mode the database account can read the masked
columns, so there is no database claim: Rosie's validator is the layer.

All about the database account Rosie runs questions with, when the check passes, as of its
last run (default every 15 minutes; with the gate on, a passing result counts for
`ROSIE_COLUMN_GRANTS_MAX_AGE_MS`, 35 minutes by default). The claim is: the database denies
this account every masked column Rosie knows how to reach, on these engines, with these
named gaps. It is not that the database refuses every statement that reaches a masked
column. The check is an allowlist: anything the account can read or execute beyond the catalog tables and the reviewed objects holds the context; within what the check enumerates, an object it does not recognise is held, not passed, and the paths that are not objects are named gaps. The review markers are your statement.
This is what the evidence supports today:

- **PostgreSQL.** The database refuses a statement that names a masked column of a catalog
  table, for the account and every role it is a member of, and through the table's
  partitions, inheritance children and parents. Anything else the account can read or
  execute in this database (a table, view, materialized view or foreign table outside the
  catalog, a definer, untrusted-language or specially granted routine, a foreign server with
  a mapping, large objects) holds the context until it is reviewed or taken away. What a
  reviewed object returns is your statement, including one that uses `*`, a whole-row
  reference or dynamic SQL (the review notes name those). Not covered: `LISTEN`/`NOTIFY`
  payloads (any role can listen, so a trigger that sends row data delivers masked values),
  statement text of other sessions in `pg_stat_activity` and `pg_stat_statements` for a
  member of `pg_read_all_stats`, other databases on the same server (`CONNECT` goes to
  `PUBLIC` by default, and their catalogs are not visible from here), a table added to an extension with `ALTER EXTENSION ... ADD TABLE`, and any view or materialized view of that extension over it (an administrator act that cannot be told apart from what the extension ships, such as PostGIS's `spatial_ref_sys`),
  copies of the values elsewhere, what unmasked columns reveal, objects created since the
  last check.
- **MySQL.** The database refuses a statement that reads a masked column by name; the check
  also reports histograms on masked columns and definer views and routines the account can
  use (a definer routine the account may execute holds the context until it is listed in
  `"reviewedRoutines"`). The database does not stop a `USING` or `NATURAL` join on a masked column
  (`mysql_join_using_natural`), a histogram created between checks
  (`mysql_column_statistics`), or a definer routine; Rosie's validator refuses those (any
  call to a function that is not a listed built-in included) and is a required layer there.
  Since 1.32.0 a table or definer view outside the catalog, in any database, holds the
  context until it is reviewed or taken away, even one that reads only unmasked columns.
  Not covered: `CHECKSUM TABLE`, which runs for a column-granted account and covers the
  masked columns (a weak equality test), statement text of other sessions in
  `performance_schema.events_statements_*` when the account may read it, and a catalog table
  that is a `FEDERATED` table (trusted as a table; one outside the catalog is held). On MySQL
  the database does not refuse whatever Rosie's own checks miss: there it is the two layers
  together. The demo was tested on MySQL 8.4; one independent review ran 8.0.
- **SQL Server.** With column grants: no database claim (a `PIVOT` reads the masked columns of
  any table the account can read one column of, so the result is `masked by Rosie only`).
  With no permission on the masked tables and a reviewed catalog view: the database refuses
  a statement on those tables; what the view shows is the operator's statement
  ([SQLSERVER.md](SQLSERVER.md)). Since 1.32.0 every table, view, synonym, function and
  procedure outside the catalog that the login can use, every linked server with data access
  and every other user database it can open holds the context until it is reviewed or taken
  away. Not covered: statement text of other sessions in `sys.dm_exec_*` for a login with
  `VIEW SERVER STATE`, and the history table of a temporal catalog table read through
  `FOR SYSTEM_TIME` (that is the catalog table's own grant; a history table outside the
  catalog is held), and an object an administrator creates in `dbo` with one of the exact diagram names and types and flags as shipped (`sp_MS_marksystemobject`). It is the weakest of the three claims. Common Criteria mode
  and application roles are untested. No independent reviewer has reproduced
  the SQL Server results; they were established on a live SQL Server 2022 CU23.

Two independent reviews of 1.31.0 found readable paths that this claim, as 1.31.0 worded it,
did not cover; 1.32.0 closes them and narrows the claim (the 1.32.0 rows of
[Corrections](SECURITY_OVERVIEW.md#corrections)). 1.32.0 itself has not been reviewed
independently.

On every engine the check does not cover: what a reviewed object really exposes (the
markers are your statement, not Rosie's), paths that are not objects it enumerates (the
named gaps above), objects in a schema or database the account
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
"maskingMode": "express",
"columnGrants": [
  { "projectId": "my-project", "maskingMode": "express",
    "mode": "rosie", "label": "masked by Rosie only",
    "total": 4, "denied": 3, "readable": 1, "unverified": 0,
    "reasons": [ "masked_readable" ], "limits": [ "mysql_join_using_natural", "mysql_column_statistics" ],
    "checkedAt": "2026-11-17T09:00:00.000Z", "hold": false,
    "checkAgeMs": 412000, "stale": false, "holdReason": null }
]
```

`maskingMode` is `express` or `strict`, at the top level (shown before any context is
checked) and on each entry; `total` the number of masked columns, `denied` how
many the database refused, `readable` how many the account can read, `unverified` how many
could not be verified, `reasons` and `limits` the codes above, and `hold` whether this
context refuses requests (`ROSIE_REQUIRE_COLUMN_GRANTS`, below). Since 1.32.0: `checkAgeMs`
is the time since the start of the check behind the result (`null` before any check),
`stale` is `true` once that is past `ROSIE_COLUMN_GRANTS_MAX_AGE_MS` (in both modes), and
`holdReason` says why a passing result holds (`expired`, `account_changed`,
`pool_not_rotated`, `no_lease`; `null` otherwise). The column names are shown in three
places:

- the server log. In strict mode, one line per finding, the first time and then only when
  the result changes: `masked_column_readable` (with the role, when it is readable through
  one), `masked_column_not_verified`, `unmasked_columns_unreadable` (with the columns),
  `view_not_reviewed`, `view_reads_masked`, `object_not_verifiable` and `views_not_checked`.
  The summary line is
  `column_grants_checked`, with `reasons` and `limits` (`column_grants_not_checked` when it
  could not run). In express mode, one `info` line per context with masked columns,
  `masking_mode_express`, logged again only when the result changes. For "masked by Rosie
  only" it says the validator refuses them, the account can read them, and how to move to
  strict; "not verified on SQL Server" is treated the same way ("the database is not verified
  to deny them"); for "masked by the database" it says the database denies them too. The
  per-finding lines above and `column_grants_checked` are logged at `debug`;
  `column_grants_not_checked` stays a warning. A context with no masked columns logs nothing
  about masking in either mode;
- `npm run doctor`, in the check "Masked columns denied by the database", with the lines
  "Catalog views reviewed", "Views reading masked columns" and "Not stopped by the database
  on this engine". A warning; with `ROSIE_REQUIRE_COLUMN_GRANTS` on, the line
  `ROSIE_REQUIRE_COLUMN_GRANTS` fails exactly when the gate holds (`masked by Rosie only`,
  `not verified on SQL Server`, `not checked`) and is `ok` ("requests are served") otherwise.
  Doctor's Configuration part has a "Masking mode" line (the mode and whether the gate is on,
  by the mode's default or set explicitly; it fails on an invalid value). In express mode
  "masked by Rosie only" and "not verified on SQL Server" give one info line (`i`), not a
  warning, with how to move to strict, and doctor exits 0 on them unless the gate was set on
  explicitly. "Not checked" and unmasked columns the account cannot read stay warnings in
  both modes;
- `GET /api/assistant/admin/overview`, under `status.maskedColumns` (each readable column with
  the roles that reach it, each unverified one with the reason, `reasons`, `limits`, `views`,
  `required` and `hold`), and the "Masked columns" card of the console. The card is green
  when the context is "masked by the database", in either mode. Otherwise, in express mode
  it is neutral and reads "Masked columns protected by Rosie's validator" (red with "REQUESTS
  ARE REFUSED" if the gate was set on explicitly and holds); in strict mode it is shown as in
  1.31.0, its detail ending "masking mode: strict".

`npm run grants -- --verify` runs the same check once, from the command line, ends each
context's line with `[masking mode: <mode>]`, and lists the views and routines that return a
masked column ([PILOT_ONBOARDING.md](PILOT_ONBOARDING.md)). It follows the gate the server
would apply. In express mode with the gate off, "masked by Rosie only" and "not verified on
SQL Server" print `info: express mode: expected. ...` and exit 0, unless `--strict` is
passed. With the gate on, by strict mode or set explicitly, such a context prints
`error: ROSIE_REQUIRE_COLUMN_GRANTS is on (...), so the server refuses every request of this
context that would run SQL ...` and exits 1; when the gate was set explicitly the context
line ends `[masking mode: express, ROSIE_REQUIRE_COLUMN_GRANTS on (set explicitly:
ROSIE_REQUIRE_COLUMN_GRANTS="true")]`. Unmasked columns the account cannot read exit 1 in
both modes.

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

Since 1.32.0, with the gate on, the pool is also replaced after every passing "masked by the
database" check, before the result is published (`read_only_pool_drained`, `reason`
`passing_check`), so the connections that serve questions are never older than the check that
let them through. That is one graceful replacement per context per check (every 15 minutes
by default). A replacement that fails, or does not finish within
`ROSIE_COLUMN_GRANTS_CHECK_TIMEOUT_MS`, publishes "not checked" and logs the error
`read_only_pool_drain_failed`; an old pool that cannot be closed logs the warning
`read_only_pool_close_failed` (its connections are no longer handed out). With the gate off
the pool is replaced only on a change of result and after a reload, as in 1.31.0.

#### Requiring the database grant: `ROSIE_REQUIRE_COLUMN_GRANTS`

Its default follows the masking mode: `false` in express, `true` in strict. A value you set
always wins over that default, in either direction (an empty value counts as unset). Since
1.32.0 it is read without letter case or surrounding spaces: `true`, `1`, `yes` and `on` turn
it on, `false`, `0`, `no` and `off` turn it off, and anything else stops the HTTP server and
both MCP servers at startup (`npm run grants` exits 2, `npm run doctor` fails its "Masking
mode" line). At every start the HTTP server and both MCP servers log
`column_grants_required` once, with `maskingMode`, `required`, `source` (`masking mode` or
`ROSIE_REQUIRE_COLUMN_GRANTS`) and, for a value you set, `raw`: the line reads
`set explicitly: ROSIE_REQUIRE_COLUMN_GRANTS="<as written>", read as on` (or `off`). An upgrade never switches it on: an installation without
`ROSIE_MASKING_MODE` runs in express mode, where it stays off unless you set it. Off, Rosie
reports and keeps answering whatever the result. With `ROSIE_REQUIRE_COLUMN_GRANTS=true`
(or strict mode without the variable), a context serves requests only
while its last result is `masked by the database` or `no masked columns` (a context that
masks nothing has nothing for the database to deny), and, since 1.32.0, only while that
result is fresh. Anything else holds it:

| Code | When |
|---|---|
| `E_COLUMN_GRANTS_NOT_CHECKED` | Not checked yet, the check could not run, timed out or failed, a reloaded context is still being checked, or the passing result's lease does not hold (`holdReason`, below) |
| `E_COLUMN_GRANTS_NOT_ENFORCED` | `masked by Rosie only` |
| `E_COLUMN_GRANTS_SQLSERVER_NOT_VERIFIED` | `not verified on SQL Server` |

- **The passing result is a lease** (since 1.32.0). It counts from the start of the check
  that produced it, on a monotonic clock, and holds the context as `E_COLUMN_GRANTS_NOT_CHECKED`
  with a `holdReason` when:
  - `expired`: it is older than `ROSIE_COLUMN_GRANTS_MAX_AGE_MS` (default twice
    `ROSIE_PERMCHECK_INTERVAL_MS` plus 5 minutes: 35 minutes at the default interval, so one
    skipped run does not hold and two do; a failed or timed-out run holds at once, below; a
    value under 60000 or not a number means the default; a value above 24 hours, set or
    derived from the interval, is replaced by 24 hours without an error, so a
    `ROSIE_PERMCHECK_INTERVAL_MS` above 24 hours holds requests between checks). The age is the
    larger of the monotonic and the wall-clock time since the check started, so a host or VM
    that was suspended resumes held if the check is older than the maximum, and a wall clock
    set back never extends it (`checkAgeMs` and `stale` in `/health` use the same age). The
    first expiry logs the error `column_grants_lease_expired`. A value under the interval
    plus the check deadline holds requests between ordinary runs;
  - `account_changed`: the read-only pool now connects as another user than the one checked;
  - `pool_not_rotated`: for "masked by the database", the pool was not replaced after the
    check started ("Changing the grants", above). A result checked with the gate off holds this
    way until the next check runs with it on;
  - `no_lease`: there is no lease for this version of the context. A context without a
    version is never matched to a stored result and holds the same way (the contexts Rosie
    loads always carry one).
- **Every check has a deadline**, `ROSIE_COLUMN_GRANTS_CHECK_TIMEOUT_MS` (default 120000; under
  1000 or not a number means the default). A check that does not finish publishes "not
  checked" and logs the error `column_grants_check_timeout`; its late result is ignored. A
  check that fails publishes "not checked" too. Either holds at once with the gate on. A
  reload waits at most one deadline (plus one pool replacement) before the new context takes
  over, held if its check stalled.
- **A failed run is retried.** A run (periodic, after a reload, or a retry) that leaves any
  context "not checked" schedules one retry after 30 seconds, then 60, then 120 for
  consecutive failures, then waits for the normal interval; a run with no "not checked"
  context resets the sequence. Each is logged as the warning `column_grants_retry_scheduled`
  (`attempt`, `delayMs`, `projectIds`). A retry is an ordinary check and is skipped like a
  periodic run if one is still in flight. The delays are not configurable.
- **Periodic runs do not pile up.** A scheduled run that comes while the previous one is still
  queued or running is skipped and logged as the warning `column_grants_run_skipped`. A run
  after a reload still queues.
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
- Log events: `column_grants_required` once at every start (since 1.32.0, whatever the
  setting; before, only when the gate was on), `column_grants_hold` (error) when a
  context becomes held, `column_grants_hold_released` when it serves again, and
  `column_grants_request_refused` (at most one line a minute per context, with the count);
  the last two carry `reason` for a lease hold. Since 1.32.0 also
  `column_grants_lease_expired`, `column_grants_check_timeout`, `column_grants_run_skipped`,
  `read_only_pool_drain_failed`, `read_only_pool_close_failed`, `column_grants_commit_failed`,
  `column_grants_run_failed` and `column_grants_retry_scheduled`
  ([MONITORING.md](MONITORING.md)).

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
`DB_DATABASE`, next to the business data. Then a schema-wide grant (express mode, Step 2)
lets the read-only user read them, and `npm run generate-catalog` lists them in the catalog
like any other table. If you keep that layout, use strict mode (or grant the read-only user
SELECT table by table instead of on `<my_schema>.*`), and remove the four `rosie_` tables
from `schema-catalog.json` after each generation. The separate database avoids both.

---

## Step 4. Build the host app's context dir

> **⚡ Fast path: `npm run init`.** The context-bootstrap wizard automates Steps 4 and 5:
> it interviews you (project identity, governance, DB credentials, language),
> introspects the schema, and uses the same LLM Rosie runs on to generate **every**
> context file, including LLM-enriched catalog semantics, `verified-queries.json`
> validated against the real engine validator AND executed on your DB, and an eval
> bank whose expectations are computed from actual data. It also writes the `.env`
> (never overwriting an existing one without confirmation). The read-only user (Step 2)
> is created only with `--create-ro-user`, with the grants of the masking mode you choose,
> and the engine tables (Step 3) only with
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
PII columns the validator refuses to return (also blocks `SELECT *` on the table). In strict
mode the read-only account has no privilege on them (Step 2), so the database refuses them
too:
```json
{ "users": ["email", "national_id", "phone"] }
```

The server starts without this file, and then nothing is masked. `npm run check-context`
warns when nothing is masked and fails when a mask names a column that does not exist.

Since 1.32.0 the mask configuration is validated before it is used, in one place for the
server, `npm run grants`, `npm run doctor`, `npm run check-context` and the wizard. The file,
when present, must be an object; the catalog's root `maskedColumns`, when present, an object;
every key maps to a list (keys starting with `_` or `$` are comments); a table's own
`maskedColumns`, when present, a list; every element a non-empty string; and no key written
twice in one object. Anything else is `E_MASK_CONFIG_INVALID`: the server refuses to start, a
reload fails and keeps the context in service (`context_reload_failed`), `npm run grants`
exits 2, `npm run doctor` fails "Mask configuration", and `npm run check-context` fails its
"Mask configuration" check. A malformed configuration is never taken as "no masked columns".
`{ "users": "email" }` is the usual mistake: write `{ "users": ["email"] }`. Two keys that
differ only by letter case are not this error (both lists apply); `check-context` still fails
them.

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
# Masked columns: express = Rosie's validator refuses them (the account may read the schema);
# strict = the database denies them too (run `npm run grants`, docs/INSTALL.md).
ROSIE_MASKING_MODE=express

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
read-only user's privileges. In strict mode (or with `ROSIE_REQUIRE_COLUMN_GRANTS=true`) it
can also mean the masked-column check did not pass; `securityCode` then starts with `E_COLUMN_GRANTS_` (Step 2,
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
- [ ] `ROSIE_MASKING_MODE` is the mode you chose ("Choosing a masking mode").
- [ ] In strict mode, `health` reports `masked by the database` for every context with masked
      columns (Step 2, "Check what the read-only account can read"), or you know why not: on
      SQL Server with column grants the result is `masked by Rosie only` because of `PIVOT`.
      `npm run grants -- --verify` exits 0 only in the first case. In express mode
      `masked by Rosie only` is expected, and `--verify` exits 0 on it with "express mode:
      expected" unless you pass `--strict`. In both modes it exits 1 while an unmasked column
      is unreadable.
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
   turns red (in 1.32.0 express mode, the default, shows one info line and a neutral card
   instead). That is the intended report, not a regression. For strict mode (1.32.0),
   remove the broad grant and grant table by table, with the statements in
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

   **Upgrading to 1.32.0.** No grant is changed by the upgrade. 1.32.0 adds
   `ROSIE_MASKING_MODE` ([Choosing a masking mode](#choosing-a-masking-mode)), and an
   installation without it runs in express mode. What you may notice:
   - The per-column warnings of the masked-column check are replaced by one info line per
     context (`masking_mode_express`), the console card turns neutral ("Masked columns
     protected by Rosie's validator") and `npm run doctor` no longer fails on "masked by
     Rosie only". The protection is the same as before: the check still runs, and the gate
     was off by default in 1.30.0 and 1.31.0 too.
   - The default of `ROSIE_REQUIRE_COLUMN_GRANTS` now follows the mode: `false` in express,
     `true` in strict. An installation that set `ROSIE_REQUIRE_COLUMN_GRANTS=true` keeps its
     gate; an explicit value always wins over the mode's default.
   - `column_grants_required` is now logged at every start, with `maskingMode`, `required`
     and `source`; in 1.31.0 it appeared only when the gate was on. An alert keyed on its
     presence fires on every start; key it on `required`.
   - A value of `ROSIE_MASKING_MODE` other than `express` or `strict` stops the server.
   - **Run `npm run check-context` before upgrading.** A mask file with a value that is not a
     list, an element that is not a string, an empty name or a key written twice used to load
     and silently mask less; it now stops the server (`E_MASK_CONFIG_INVALID`, Step 4d).
   - More objects are held until reviewed (Step 2, "Objects held until reviewed"). In strict
     mode a context 1.31.0 reported as "masked by the database" can now read "masked by Rosie
     only" or "not verified on SQL Server" because of an ordinary PostgreSQL reporting view
     over a masked table, a foreign table, a foreign server with a user mapping, or a SQL
     Server procedure the account may execute. With the gate on that holds requests. Run
     `npm run grants -- --verify` on 1.32.0 against your database before you put it in service,
     and list what you have reviewed in `"reviewedViews"` or `"reviewedRoutines"`. Those lists
     compare (schema, name) pairs now.
   - **The check is an allowlist** (Step 2, "Objects held until reviewed"). In strict mode,
     every table, view, materialized view, foreign table, synonym or routine the account can
     use beyond the catalog tables holds the context until it is listed in
     `"reviewedViews"` or `"reviewedRoutines"` or its access is taken away; PostgreSQL foreign
     servers with a mapping, large objects and file functions, SQL Server linked servers and
     other user databases can only be taken away. This is many more objects than 1.31.0 or
     the first 1.32.0 notes above held: clean reporting views, copy tables, MySQL definer
     views and every SQL Server function count. Run `npm run grants -- --verify` on 1.32.0
     against your database before you put it in service and read its list; `--verify` now
     follows the gate the server would apply, so a gated installation sees `error:` and exit 1
     without `--strict`.
   - **Unreadable values stop the server.** `ROSIE_REQUIRE_COLUMN_GRANTS`,
     `ROSIE_CERTIFY_SQL`, `ROSIE_ADMIN_WRITE`, `ROSIE_AUDIT_FSYNC`, `ROSIE_MCP_HTTP_ENABLED`
     and `AI_UNION_ALL` accept `true/1/yes/on` and `false/0/no/off` in any case;
     `ROSIE_AUDIT_POLICY` accepts `fail-open` or `fail-closed` in any case; `DB_SSL` accepts
     `true`, `require`, `verify`, `no-verify` or `false`. Anything else stops the server with a
     message naming the setting ([OPERATIONS.md](OPERATIONS.md#settings-reference)). Values
     that used to read as off now read as on: `ROSIE_REQUIRE_COLUMN_GRANTS=TRUE` turns the gate
     on, `ROSIE_AUDIT_POLICY=FAIL-CLOSED` is fail-closed, and `AI_ANSWER_CHECKS=TRUE` keeps the
     checks on. The same rule now covers every yes-or-no `AI_*` setting: `AI_ANSWER_CHECKS`, `AI_CROSS_CHECK`, `AI_CERTIFICATE_REPAIR`, `AI_CERTIFICATE_REPAIR_UNDECLARED`, `AI_QUESTION_REVIEW`, `AI_QUESTION_REVIEW_REPAIR`, `AI_ASSISTANT_ENABLED`, `AI_CATALOG_COLUMNS`, `AI_CATALOG_DESCRIPTIONS`, `AI_CATALOG_PROFILE`.
     A value such as `AI_ANSWER_CHECKS=enabled`, which used to turn the certificate checks off
     in silence, now stops the server with the message.
   - **The lease also runs on the wall clock**: after a suspended host or VM resumes, a
     result older than `ROSIE_COLUMN_GRANTS_MAX_AGE_MS` holds until a new check passes, and
     the maximum age is capped at 24 hours.
   - **Catalog names match as the engine compares them**: on PostgreSQL a table that differs
     from a catalog table only by letter case is now outside, and on MySQL anything readable
     in `mysql` or in `sys` beyond what the server ships holds the context.
   - **A failed check is retried** after 30, 60 and 120 seconds
     (`column_grants_retry_scheduled`). With the gate on it still holds at once.
   - **Rosie's own tables in express.** The express wizard now puts them in a separate
     database, `rosie`, by default (`--engine-db same` keeps the old layout). An existing
     installation is not moved; with them in the business database the express account can
     read them, and in strict mode they are outside the catalog and hold the context if the
     account can read them.
   - With the gate on, a passing result holds requests once it is older than
     `ROSIE_COLUMN_GRANTS_MAX_AGE_MS` (35 minutes by default), and the read-only pool is
     replaced after every passing check (`read_only_pool_drained` with `reason`
     `passing_check`, every 15 minutes by default). `/health` gains `checkAgeMs`, `stale` and
     `holdReason`.
   - `npm run grants -- --verify` prints review notes (warnings) for reviewed objects that use
     `*`, a whole-row reference or dynamic SQL. They do not change its exit code.
   - If you had moved the account to table and column grants (with `npm run grants` or by
     hand), set `ROSIE_MASKING_MODE=strict` to keep the warnings and get the gate by default.
     Check `npm run grants -- --verify` first: in strict mode a context that is not "masked by
     the database" is held.
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

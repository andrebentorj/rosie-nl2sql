# Pilot onboarding: the read-only account and its column grants

The week 1 step of a pilot in strict mode, for the customer's database administrator: one
command prints the statements, the administrator runs them, one check confirms the result.
Rosie never runs a `GRANT` or a `REVOKE` itself.

## Express or strict

A pilot can run in either masking mode
([INSTALL.md](INSTALL.md#choosing-a-masking-mode)). Use strict mode when the pilot database
holds personal data; express mode otherwise.

**Express.** Rosie refuses statements that reach a masked column and leaves masked columns
out of what it describes to the model. This is Rosie's own reading of SQL; independent
reviews have found ways around it before (see
[Corrections](SECURITY_OVERVIEW.md#corrections)), and the database account can read those
columns. The steps:

1. Run `npm run init -- --create-ro-user` and accept the defaults. The account gets `SELECT`
   on the whole schema (on MySQL the whole database), and nothing else, and Rosie's own
   tables go to a separate database, `rosie`, by default. Or create it by hand with the
   express statements of [INSTALL.md](INSTALL.md), Step 2.
2. Review `masked-columns.json` with the customer.
3. Run `npm run grants -- --verify`. It ends the context's line with
   `[masking mode: express]`; "masked by Rosie only" is expected and exits 0 with an
   `info: express mode: expected` line, as long as `ROSIE_REQUIRE_COLUMN_GRANTS` is not set
   on (with it on, the server would refuse requests, and `--verify` says so with an `error:`
   line and exits 1). It exits 1 too if an unmasked column is unreadable (questions on that
   table would fail).
4. Agree with the customer, in writing, that the masked columns rest on Rosie's validator.
   The rest of this page is not needed. The pilot can move to strict later
   ([Moving from express to strict](INSTALL.md#moving-from-express-to-strict)).

**Strict.** The database denies the masked columns as well, as far as each engine allows
(below). The steps:

1. Set `ROSIE_MASKING_MODE=strict` in the `.env` (or `npm run init -- --masking=strict
   --create-ro-user`, which grants table by table from the catalog and the masks).
2. Print the statements ([The one command](#the-one-command)), have the administrator apply
   them ([Applying the script](#applying-the-script)), and restart Rosie.
3. Confirm with `npm run grants -- --verify` ([The check](#the-check)). Until it passes,
   strict mode holds every request of that context
   ([Holding requests](#holding-requests-until-the-check-passes)).

The rest of this page is the strict path.

What this step gives is the database denies this account every masked column Rosie knows how to reach, on these engines, with named gaps. The check is an allowlist: anything the account can read or execute beyond the catalog tables and the reviewed objects holds the context; within what the check enumerates, an object it does not recognise is held, not passed, and the paths that are not objects are named gaps, and what you
mark as reviewed is your statement. It depends on the engine. On
PostgreSQL, column grants make the database refuse a statement that names a masked column of
the catalog's tables, and the check holds every view, materialized view, foreign table and
foreign server that could reach one until it is reviewed. On MySQL they do too for a statement that
reads a masked column by name, but not for a join `USING` a masked column, a `NATURAL JOIN`,
or a histogram created between two checks: there Rosie's validator is a required layer, not a
second one. A definer stored function the account can execute is not stopped by the
database either; the validator refuses the call (MySQL functions are allowlisted since
1.31.0) and the check reports the routine. Give the account no `EXECUTE` on routines. On SQL Server, column grants are not enough, and the step ends in a different
design ([SQL Server](#sql-server) below). Two independent reviews of 1.31.0 found paths its
check did not cover; 1.32.0 holds them, and has not been reviewed independently itself
([Corrections](SECURITY_OVERVIEW.md#corrections)).

## What the customer provides

- The engine and its version: MySQL 8, PostgreSQL, or SQL Server. MariaDB is not supported
  (Rosie refuses `DB_DIALECT=mariadb` at startup).
- Host, port, database name, and on PostgreSQL or SQL Server the schema (`DB_SCHEMA`).
- A database administrator for about one hour, with an account allowed to create a login and
  to grant and revoke privileges.
- The list of personal-data columns to mask, or an approval of the suggestions that
  `npm run generate-catalog` prints. They go in `masked-columns.json` of the context.
- Which tables of the catalog are views, if any, and someone who can say what each one
  exposes. Also which other views, stored routines (MySQL) or `SECURITY DEFINER` functions
  (PostgreSQL) the account will be able to use, and someone who can review each one for
  `"reviewedRoutines"` (on a managed PostgreSQL, the platform's own functions that `PUBLIC`
  may execute count too), and, on MySQL, whether anyone maintains histograms
  (`ANALYZE TABLE ... UPDATE HISTOGRAM`).
- Network access from the machine Rosie runs on to the database.

## The read-only account

Rosie sends every statement the model writes through a separate account, set in `.env` as
`DB_RO_USER` and `DB_RO_PASSWORD` (with `DB_RO_HOST`, `DB_RO_PORT` and `DB_RO_DATABASE` when
they differ from the main connection). That account must be able to `SELECT` and nothing else.

Create it with no privileges at all, for example:

```sql
-- PostgreSQL
CREATE ROLE rosie_readonly LOGIN PASSWORD '...';
-- MySQL
CREATE USER 'rosie_readonly'@'%' IDENTIFIED BY '...';
-- SQL Server
CREATE LOGIN rosie_readonly WITH PASSWORD = '...';
CREATE USER rosie_readonly FOR LOGIN rosie_readonly;
```

`npm run init -- --masking=strict --create-ro-user` does this and the table and column grants
in one go when it is allowed to create users ([INSTALL.md](INSTALL.md), Step 2). In express
mode the same option grants `SELECT` on the whole schema instead, which the command below
turns into table and column grants when the pilot moves to strict. An account that already exists, for example
one the customer created earlier with a schema-wide grant, is handled by the command below.

## The one command

Once the context exists (the catalog and the masks), run on the Rosie machine, with the `.env`
that points at the customer's database:

```bash
npm run -s grants > grants.sql
```

`npm run doctor -- --grants` is the same command. `--context <dir>` names a context directory
(repeatable; the default is `ROSIE_CONTEXT_DIR`). With several, one account serves them all:
their tables and columns are joined, and a column masked in any of them is masked.

It connects **as the read-only account** (if `DB_RO_USER` is not set it says so on stderr,
because it would then read the main account), reads what that account holds from the
engine's own catalog (`pg_class` and `pg_attribute` privileges, role memberships and default
privileges on PostgreSQL; `SHOW GRANTS` on MySQL; `sys.database_permissions` and role
memberships on SQL Server), and prints a SQL script. Every explanation in it is a `--`
comment, so the file can be run as it is. It never runs the statements. Its parts:

1. **Remove the broader grants**: a grant on every database or on the whole database on
   MySQL (`*.*`, `db.*`), a schema or database grant on SQL Server, a table grant on a table
   that has masked columns, a grant on a masked column, a grant to `PUBLIC` on a masked table
   or column, granted roles on MySQL, member roles that can read a masked column on
   PostgreSQL (including `pg_read_all_data`), default privileges on PostgreSQL, a table grant
   or masked-column grant (the account's or `PUBLIC`'s) on a partition, inheritance child or
   parent of a masked table on PostgreSQL, and `db_datareader` or `db_owner` membership on SQL
   Server. A relative that is itself in the catalog gets the masks of its masked table, so it
   is granted only its other columns.
   - **1b. Drop the histograms of masked columns** (MySQL, when there are any):
     `ANALYZE TABLE <db>.<table> DROP HISTOGRAM ON <column>;`, with a warning: MySQL shows a
     histogram, real values included, to any account that can read a column of the table;
     dropping one can change query plans; updating it again brings the values back, and the
     check then reports the column again (`mysql_histogram_reads_masked`).
2. **Grant table by table**, and on a table with masked columns only the unmasked columns,
   for what the account does not hold yet. They are the same statements `--create-ro-user`
   writes (`scripts/bootstrap/ro-grants.js`).
3. Restart Rosie (see below).
4. Confirm with `npm run grants -- --verify`. On PostgreSQL and MySQL it must say "masked by
   the database". On SQL Server the script says these column grants end at "masked by Rosie
   only" (`sqlserver_pivot_reads_masked`) and points to the reviewed-view design.

The script does not revoke views, stored routines or functions. `--verify` lists the ones that
return a masked column or that the check cannot read; remove the account's access to them by
hand (below).

It ends with "To review by hand" when there is something the account cannot see about itself
(a catalog view it could not read, a PostgreSQL role whose privileges could not be asked, a
custom SQL Server role, whose columns are not visible to its member), and always with a
reminder that a view or function that copies a masked column, and what the unmasked columns
reveal, are not covered by grants, and that `--verify` lists the views and routines it
finds.

The REVOKEs come first because a narrower grant does not remove a broader one. On MySQL
privileges add up across levels; on PostgreSQL a table grant covers every column and a column
`REVOKE` under it does nothing; on SQL Server a table-level `DENY` does not override a column
`GRANT`. Adding the column grants to an account that keeps its broad grant changes nothing.

An account that is already in this form gets a script that says "Nothing to change". The
command exits 0 when it printed a script and 2 when it could not read the account or the
context.

## Applying the script

The administrator reads the script and runs it as the administrator account, in order, for
example:

```bash
psql -U <admin> -d <database> -v ON_ERROR_STOP=1 -f grants.sql      # PostgreSQL
mysql -u <admin> -p < grants.sql                                     # MySQL
sqlcmd -S <host> -U <admin> -d <database> -b -i grants.sql           # SQL Server
```

Two statements deserve a second look before they run:

- A `REVOKE ... FROM PUBLIC` takes the privilege from **every** account, not only Rosie's.
  Grant the table again to the accounts that need it.
- A role `REVOKE` takes everything that role gave, not only the masked columns.

Then restart Rosie, which applies the change at once. Without a restart, the next check
(every 15 minutes) sees the change and, when its result changes, replaces the connections of
the read-only pool (`read_only_pool_drained`). Until then, on MySQL, a connection that was
already open keeps a database-level privilege after it is revoked, so on MySQL restart.

## The check

```bash
npm run grants -- --verify
```

It runs Rosie's own startup check on a new connection as the read-only account and prints,
for each context, the result, then the details that apply:

```
my-project: masked by the database (13 of 13 masked column(s) denied by the database) [masking mode: strict]
```

In express mode the same account, with `SELECT` on the whole schema, gives:

```
my-project: masked by Rosie only (0 of 13 masked column(s) denied by the database) [masking mode: express]
  info: express mode: expected. Masked columns are refused by Rosie's validator; the database account can read them. ...
```

- `readable by the account: ...`: masked columns the account can read, with the role that
  gives each one (`through ...`), when a role does;
- `not verified: ...`: masked columns or mask entries that could not be verified, with the
  reason;
- `unmasked columns the account cannot read (questions on these tables fail): ...`;
- `views and routines that return a masked column: <kind> <name> (<columns>)`;
- `next: ...`: what to do. For views and routines it says to remove the account's `SELECT` or
  `EXECUTE` on them (on PostgreSQL `EXECUTE` on a function goes to `PUBLIC` by default), or,
  when one returns no masked value: for a routine, list it as `"schema.name"` in
  `"reviewedRoutines"` at the top of `schema-catalog.json` (your statement; Rosie never
  passes a routine on its text); for a view, materialized view or foreign table outside the
  catalog, list it in `"reviewedViews"`; for a MySQL view, give the account `SHOW VIEW` on it
  so the check can read it;
- `warning: the reviewed <kind> <name> uses a star (*) / a whole-row reference / dynamic SQL
  ...`: a reviewed object that can return masked values without naming them. Read it by hand.
  The warning never changes the exit code.

The exit code is 0 only when every context is "masked by the database" or "no masked
columns" and no unmasked column is unreadable, and 1 otherwise, so the command can gate the
end of the onboarding step. It exits 2 when it cannot run (no context, an unreadable
account, or an invalid `ROSIE_MASKING_MODE`). Each context's line ends with
`[masking mode: <mode>]` (with the gate set explicitly, also the setting and its raw value).
In express mode with the gate off, "masked by Rosie only" and "not verified on SQL Server"
exit 0 with an `info: express mode: expected` line; pass `--strict` to apply the strict exit
code anyway. With the gate on, by strict mode or set explicitly, such a context prints an
`error:` line saying the server refuses its requests, and exits 1. Unmasked columns the
account cannot read exit 1 in both modes.

Since 1.32.0 "masked by the database" also requires that the account can read or execute
nothing beyond the catalog tables and the reviewed objects (an allowlist: an object kind the
check does not recognise is held, not passed). Ordinary reporting views, copy tables,
functions and procedures now need a review, or their access taken away, in week 1; run
`--verify` first and read its list. Since 1.31.0 it also requires that every
unmasked catalog column is readable, that every catalog table that is a view carries `"reviewedView": true` in
`schema-catalog.json`, and that no view, function or procedure the account can use returns a
masked column, or cannot be read by the check, unless it is such a reviewed catalog view
(PostgreSQL views and `SECURITY DEFINER` functions, MySQL definer views and routines, SQL
Server views, functions and procedures). On PostgreSQL the partitions, inheritance children
and parents of a masked table count too; on MySQL a histogram of a masked column counts. `reviewedView` is your statement that
someone looked at the view and it exposes no masked value; Rosie does not check what was
reviewed. The full rule and the reason codes are in
[INSTALL.md](INSTALL.md#check-what-the-read-only-account-can-read).

The result names what the database refuses, within what the check enumerates, not more. It does not see objects in a schema
or database the account cannot see, anything created between two checks, copies of a masked
column, or what the unmasked columns reveal. A context with no masked columns skips the view
and routine checks.

## What `/health` shows

`GET /api/assistant/health` is public, so it carries the result per context without column
names:

```json
"columnGrantsRequired": true,
"maskingMode": "strict",
"columnGrants": [
  { "projectId": "my-project", "maskingMode": "strict", "mode": "database", "label": "masked by the database",
    "total": 13, "denied": 13, "readable": 0, "unverified": 0,
    "reasons": [], "limits": [ "mysql_join_using_natural", "mysql_column_statistics" ],
    "checkedAt": "2026-11-17T09:00:00.000Z", "hold": false,
    "checkAgeMs": 412000, "stale": false, "holdReason": null }
]
```

`mode` is one of `database`, `rosie`, `mssql`, `none` and `unchecked`; `reasons` says why a
context is not `database`; `limits` names what the database does not stop on this engine
even then (`mysql_join_using_natural` and `mysql_column_statistics` on MySQL, `mssql_pivot`
on SQL Server)
([INSTALL.md](INSTALL.md#check-what-the-read-only-account-can-read)). The column names are
in the server log (`masked_column_readable`, `masked_column_not_verified`,
`unmasked_columns_unreadable`, `view_not_reviewed`, `view_reads_masked`,
`object_not_verifiable`, `column_grants_checked`), in `npm run doctor`, and in the admin
console.

## If it says "masked by Rosie only"

Read `reasons` (in `/health`, the console or `npm run doctor`) and the lines of `--verify`.

1. A masked column is readable (`masked_readable`): run `npm run -s grants` again. It reads
   the account as it is now and prints what is still too broad. Apply it and verify again.
2. If the script is empty and a column is still readable, the privilege comes from something
   the account cannot see about itself, for example a role listed under "To review by hand".
   The administrator finds and removes it, then runs the check again.
3. "not verified" entries (`masked_not_verified`) name the reason: a masked column the
   catalog does not list, a column the database does not have, or a probe that failed for
   another reason. Fix the context or the catalog (`npm run check-context`).
4. "unmasked columns the account cannot read" (`unmasked_unreadable`) means a grant is too
   narrow, or the catalog names a column the database no longer has: questions on that table
   will fail. Re-run `npm run -s grants`; it prints the missing column grants.
5. A catalog view without review (`view_not_reviewed`): look at what the view selects. If it
   exposes no masked value, add `"reviewedView": true` to it in `schema-catalog.json` and
   reload; otherwise mask its column or take it out of the catalog.
6. A view or routine returning a masked column (`view_reads_masked`, all three engines):
   take away the account's `SELECT` or `EXECUTE` on it (on PostgreSQL, `EXECUTE` on a
   function goes to `PUBLIC` by default), or, for a catalog view that is safe, review it as
   in 5.
7. `object_not_verifiable` (PostgreSQL and MySQL; the detail names the object):
   - A definer routine (MySQL) or `SECURITY DEFINER` function or procedure (PostgreSQL) the
     account may execute is always reported this way, whatever its body, until it is listed
     in `"reviewedRoutines"` in `schema-catalog.json` (`"schema.name"`, on MySQL
     `"database.name"`; on PostgreSQL write a mixed-case name in double quotes,
     `public."F_Clean"`, as an unquoted name folds to lower case). A listing covers every
     overload of that name, present and future. On MySQL this includes a routine the account
     may execute only through a role's routine-level grant. List it only after someone has read it and found that it returns no
     masked value: the listing is your statement, and the check does not see through it. A
     listed routine whose body names a masked column still keeps the context out. Otherwise
     take away the account's `EXECUTE`, or make the routine `SQL SECURITY INVOKER`.
   - A PostgreSQL `SECURITY DEFINER` function anywhere in the database that `PUBLIC` may
     execute holds every context until it is listed or its `EXECUTE` is taken from `PUBLIC`.
     On a managed PostgreSQL whose platform installs such functions, run `--verify` in week 1.
   - A MySQL definer view whose definition the account cannot read, or cannot be read with
     confidence, or an invoker view inside a definer view whose definition is hidden: give the
     account `SHOW VIEW` on it, or remove its access.
   - Since 1.32.0, any object the account can read or execute beyond the catalog tables
     and the reviewed objects (the detail names the first 20; `views.outside` keeps up to
     200): a table, view, materialized view, foreign table or synonym outside the catalog,
     on any engine and in any schema or database; a routine (PostgreSQL definer,
     untrusted-language or granted beyond `PUBLIC`, MySQL definer, SQL Server function or
     procedure). Review it and list it, relations and synonyms in `"reviewedViews"`, routines
     in `"reviewedRoutines"`, or take the access away. A PostgreSQL view created
     `WITH (security_invoker = true)` is not held.
   - A PostgreSQL foreign server with `USAGE` and a user mapping, PostgreSQL large objects,
     a file function such as `pg_read_binary_file`, a SQL Server linked server or another
     user database the login can open: take the access away. These are not to be listed.
8. A PostgreSQL partition, inheritance child or parent (`postgres_inheritance_reads_masked`):
   run `npm run -s grants` again; it prints the statement for that table.
9. A MySQL histogram (`mysql_histogram_reads_masked`): run `npm run -s grants` again; it
   prints the `ANALYZE TABLE ... DROP HISTOGRAM` for that column.
10. On SQL Server, `sqlserver_pivot_reads_masked`: see below. Running the script again does
    not change it.

## Holding requests until the check passes

In strict mode `ROSIE_REQUIRE_COLUMN_GRANTS` defaults to `true`; in express mode to `false`.
A value set in the `.env` wins over the mode's default. With the gate on, Rosie refuses
every request that would run SQL (`/ask`, `/export`, `/explain`, `/certify` and the MCP
tools, with HTTP 503 and a code starting with `E_COLUMN_GRANTS_`) for a context whose last
check is not "masked by the database" or "no masked columns", and at startup until the first
check has run. A strict pilot on PostgreSQL or MySQL keeps it on, so that a grant, a view or
a histogram changed later stops the assistant once a check detects it, instead of only producing a
warning. On SQL Server with column grants it would hold every request
(below). Details: [INSTALL.md](INSTALL.md#requiring-the-database-grant-rosie_require_column_grants).

Since 1.32.0 a passing result is also a lease: with the gate on, it lets the context serve for
`ROSIE_COLUMN_GRANTS_MAX_AGE_MS` (35 minutes by default), so a check that stalls or fails
holds requests instead of leaving the last result in service, and the read-only pool is
replaced after every passing check. `/health` shows `checkAgeMs`, `stale` and `holdReason`.

Without it, Rosie keeps answering whatever the result. A strict pilot does not start its
measurement until the check says "masked by the database", except on SQL Server, where the
customer decides on the design below.

## SQL Server

The same command prints the statements and the same check runs. But SQL Server applies
column permissions to the aggregated and the `FOR` column of a `PIVOT`, not to the columns it
passes through: with `SELECT` on any one column of a table, a `PIVOT` returns every other
column, a column `DENY` included (SQL Server 2022, observed live). Rosie's validator refuses
`PIVOT` and `UNPIVOT`, so it is the only layer that stops this. As a result, with the column
grants the script prints, `--verify` reports "masked by Rosie only", lists each masked column
as readable through `PIVOT` (reason `sqlserver_pivot_reads_masked`), and exits 1. That is the
true state of such an account. Its `next:` line says so: more grants will not change the
result, and the way to "masked by the database" is the reviewed-view design below.

Two choices for the pilot:

- **Express mode.** The masked columns rest on Rosie's validator; agree this with the
  customer. Column grants can still be applied, but they do not change the result. The demo
  runs SQL Server this way. (Strict mode with column grants would hold every request, unless
  `ROSIE_REQUIRE_COLUMN_GRANTS=false` is set.)
- **Masked by the database.** The account gets no permission at all on a table that has
  masked columns, and reads it through a view that selects only the unmasked columns. Mark
  the view `"reviewedView": true` in the catalog, list the table with its masked columns
  only, and grant the account `SELECT` on the view and `VIEW DEFINITION` on the view and on
  the table, so the check can follow the view. The statements are in
  [SQLSERVER.md](SQLSERVER.md). `--verify` then exits 0, and the pilot can run in strict
  mode.

The check on SQL Server also looks at `db_datareader`, `IMPERSONATE` and `CONTROL SERVER`,
every view, function and procedure the account may use (it needs `VIEW DEFINITION` on them),
and cross-database ownership chaining. What it cannot establish gives "not verified on SQL
Server", with the reason in the `not verified:` line. A view or function owned by the table's
owner returns the masked columns to anyone allowed to use it, through the ownership chain,
whatever the column grants. Dynamic Data Masking is not a denial and does not count.

## `ROSIE_CERTIFY_SQL` stays off during a pilot

`ROSIE_CERTIFY_SQL` is `false` by default; keep it that way for the pilot. When it is `true`,
`POST /certify` and the MCP tool `certify_sql` accept a `SELECT` written by the caller, so the
statement no longer comes from a question in words. During a pilot, people ask in words.

## Adding a mask later

In express mode, publish the mask and reload. In strict mode, take the column away in the
database first, then publish the mask: remove the column from the
grant of that table (run `npm run -s grants -- --context <dir>` against a copy of the context
with the edited `masked-columns.json`, or revoke it by hand), then publish the mask and
reload. Removing a mask is the reverse: unmask, reload, then grant. After either, run
`npm run grants -- --verify`. With `ROSIE_REQUIRE_COLUMN_GRANTS` on, a reloaded context keeps
the old one serving until its check has run, and is held if it does not pass.

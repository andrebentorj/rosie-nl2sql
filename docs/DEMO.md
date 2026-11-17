# Demo and public benchmark

A self-contained setup over the [Chinook](https://github.com/lerocha/chinook-database) sample
database (a digital media store, MIT licensed): 11 tables, 412 invoices from 2021 to 2025.
It is the fastest way to see Rosie answer questions, and it is the dataset behind the
published benchmark numbers.

## Run it

```bash
./scripts/demo/fetch-data.sh                            # downloads Chinook into demo/data/
docker compose -f demo/docker-compose.yml up -d mysql   # MySQL 8.4 on 127.0.0.1:3307
cp demo/env.example .env                                # then set OPENAI_API_KEY and ROSIE_CONTEXT_DIR
npm ci
npm start                                               # http://localhost:3001
```

The container creates three things: the `Chinook` schema with the sample data, a `rosie_ro`
user that can only SELECT from it, and a `rosie` schema holding the engine's own tables.
The credentials in `docker-compose.yml` are demo values bound to the loopback interface.

Since 1.30.0 `rosie_ro` is granted table by table, and on the tables with masked columns
(`contexts/demo/masked-columns.json`) only the unmasked columns. The demo env files set the
masking mode (`docs/INSTALL.md`, "Choosing a masking mode") to match what each engine can
reach with those grants:

- **MySQL and PostgreSQL run in strict mode** (`ROSIE_MASKING_MODE=strict`). The grants
  reach "masked by the database", which `/health` should show for the demo context (on MySQL
  with the limits `mysql_join_using_natural` and `mysql_column_statistics`), and requests are
  held if they stop doing so.
- **SQL Server runs in express mode** (`ROSIE_MASKING_MODE=express`). Column grants do not
  stop a `PIVOT` there (`docs/SQLSERVER.md`), so `/health` shows "masked by Rosie only", with
  each masked column readable through `PIVOT`, and strict mode would hold every request.
  The masked columns rest on Rosie's validator, which refuses `PIVOT`.
The users are created only when a container starts on an empty volume: a demo volume created
before 1.30.0 keeps the old schema-wide grant, and `/health` shows "masked by Rosie only"
(in strict mode every request is then held). Recreate it with `docker compose -f demo/docker-compose.yml down -v`, then `up -d`.

## The context

`contexts/demo/` is a complete, hand-written context for this schema and doubles as a worked
example of the contract in `contexts/example/README.md`: catalog with descriptions, synonyms
and named metrics, masked contact columns, verified queries and one clarification rule.

## The benchmark

`contexts/demo/eval/questions/` holds the question bank. Expected answer sizes are derived at
run time from a `sql_truth` query against the same database, so the bank does not age.

```bash
node -r dotenv/config test/eval/run.js --dry-run     # validate the bank, no model calls
node -r dotenv/config test/eval/run.js               # full run (calls the model provider)
node -r dotenv/config test/eval/run.js --category=poisoned-silent-failures --verbose
```

The `poisoned-silent-failures` category is the one that measures the Answer Certificate:
each question has a trap that produces a plausible wrong number (join fan-out, a filter value
that does not exist, a period the data only partly covers, a filter that removes nothing), and
the score is whether the certificate caught it.

`contexts/demo/eval/bank.json` is the other kind of bank, the one an installation keeps for
itself: questions with a reference statement, read by `npm run evaluate`
(`docs/EVALUATION.md`) and by `npm run loadtest` (`docs/CAPACITY.md`).

## PostgreSQL

The same data, questions and context run on PostgreSQL 16:

```bash
docker compose -f demo/docker-compose.yml up -d postgres   # 127.0.0.1:5433
cp demo/env.postgres.example .env                          # then set OPENAI_API_KEY and ROSIE_CONTEXT_DIR
```

The official Chinook script for PostgreSQL uses snake_case names. `demo/init-postgres/10-names.sql`
renames them to the mixed-case names of the MySQL script, so one question bank serves both
engines and the run exercises quoted identifiers, the harder case on PostgreSQL. A few truth
queries need engine-specific SQL and carry a `sql_truth_postgres` variant. Details in
`docs/POSTGRES.md`.

## SQL Server

```bash
docker compose -f demo/docker-compose.yml --profile sqlserver up -d sqlserver   # 127.0.0.1:1434
./scripts/demo/setup-sqlserver.sh                                               # loads the data and the logins
cp demo/env.sqlserver.example .env                 # then set OPENAI_API_KEY and ROSIE_CONTEXT_DIR
```

Starting this service accepts Microsoft's licence terms for SQL Server Developer edition, so
it is not part of the default `up -d`. Truth queries that need T-SQL carry a `sql_truth_mssql`
variant. Details in `docs/SQLSERVER.md`.

To reset the database: `docker compose -f demo/docker-compose.yml down -v`, then `up -d`.
This is also how a demo created before 1.30.0 gets the column grants of `rosie_ro` (on SQL
Server, run `./scripts/demo/setup-sqlserver.sh` again after `up -d`, since it stops when the
data is already there).

## Grant tests against the demo

Four suites run against the demo containers (the first three on MySQL and PostgreSQL, the
`rosie_ro` of the init scripts, with its column grants; the fourth also on SQL Server). They need the containers up and none runs in CI, whose
database services have no Chinook and no column-granted account. The SQL Server proof of the
same behaviour (`PIVOT`, views, the reviewed-view grants) is part of `npm run test:sqlserver`,
whose column-grant sections have not yet had a green run in CI.

- `npm run test:grants` (read-only, as `rosie_ro`): the grant check must say "masked by the
  database"; 124 attack shapes from the earlier reviews (`test/grants/shapes/attacks.json`)
  are sent raw and each must be refused with a permission error; the known database gaps
  (`databaseGaps`: MySQL joins `USING` or `NATURAL JOIN` on a masked column, and the read of
  `information_schema.COLUMN_STATISTICS` (`gap-mysql-column-statistics`); the SQL Server
  `PIVOT` shapes are listed there but not run by this suite) must still run,
  so a change of server behaviour is seen, and are never counted as protected; ordinary
  statements must run; then the engine's own tools (`execute_query`, `validate_query`,
  `sample_rows`, `lookup_values`, the certificate with its cross-check) and row policies run
  under the grants, and none may be denied. Flags: `--user`, `--password`, `--engines`,
  `--expect database|rosie`, `--readable N`, `--no-children`, `--root`, `--list`.
- `npm run test:differential`: generates about 36,000 statements over the three engines
  (positions, scopes, aliases, set operations, joins, `PIVOT` on SQL Server), passes each
  through the validator and rewriter, and runs every accepted one as `rosie_ro`. The one
  assertion: no statement the validator accepts may be denied by the database. On MySQL, no
  accepted statement may contain `NATURAL JOIN` or `USING` a masked column; on SQL Server,
  none may contain `PIVOT (`. It fails if the account can read `Customer.Email` (the run
  would prove nothing). SQL Server runs offline by default (the accepted statements are
  searched for a masked column used as a name); `--mssql-live --mssql-user U
  --mssql-password P` runs it live. Flags: `--dialects`, `--root`, `--budget`,
  `--concurrency`, `--sample`, `--show`, `--show-accepted`, `--show-ran-masked`.
- `npm run test:grants:gate` (writes to the demo servers' privilege tables): creates
  throwaway roles (`rosie_r8_gate` and helpers), turns `ROSIE_REQUIRE_COLUMN_GRANTS` on, and
  shows the check flipping to "masked by Rosie only" when a broader grant or a role is added,
  the pool drained, requests held and released, and on MySQL a stale connection still
  reading; then drops every role and verifies none is left. It uses the demo's admin
  accounts and never touches `rosie_ro`.
- `npm run test:grants:wizard` (writes to the demo servers' privilege tables): creates a
  throwaway account (`rosie_r9_code`) through the code path of `npm run init --
  --create-ro-user`, in express and in strict mode, on PostgreSQL, MySQL and SQL Server. As
  that account it checks that the read-only audit passes, a write is refused, the
  masked-column check reports what the mode gives (express "masked by Rosie only"; strict
  "masked by the database" on PostgreSQL and MySQL, "masked by Rosie only" on SQL Server
  because of `PIVOT`) and doctor's lines and exit code for that state; then drops the account
  and verifies the drop. It never touches `rosie_ro`. Flag: `--engines`.

A run of the differential that finds nothing says these statements hold, not that the
validator is complete.

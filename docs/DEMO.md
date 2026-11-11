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

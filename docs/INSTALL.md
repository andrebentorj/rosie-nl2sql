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

You need Node.js 22 or newer, a MySQL, MariaDB, PostgreSQL or SQL Server database, and a model API key.

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
| `masked-columns.json` | Columns that look like personal data, which Rosie will never read or send to the model. |
| `system-prompt.txt`, `project.json` | The assistant's instructions and its welcome message and suggestions. |
| `verified-queries.json`, `clarification-rules.json` | Example questions with SQL that was run against your database, and rules for ambiguous questions. |
| `eval/questions/` | A first question bank, with expected results computed from your data. |
| `.env` | The server configuration (written as `.env.rosie-init` when a `.env` already exists). |

It then prints the configuration to connect an AI client over MCP.

| Option | Effect |
|---|---|
| `--url <url>` | `mysql://...`, `postgres://...` or `mssql://...`. The scheme selects the engine. To keep the password out of your shell history, put the URL in the `DB_URL` environment variable instead. |
| `--name <name>` | Project name. Default: the database name. |
| `--create-ro-user` | Creates `rosie_readonly`, a user that can only SELECT, with a generated password written to the `.env`. **Writes to your database**; needs an account allowed to create users. If it cannot, the statements to run are printed. Without this option no user is created. |
| `--create-engine-tables` | Creates Rosie's four engine tables (`rosie_audit_log`, `rosie_answer_certificate`, `rosie_feedback`, `rosie_rate_limit`). **Writes to your database.** Add `--engine-db <name>` to keep them in a separate database. Without this option no table is created. |
| `--samples` | Lets sample values of unmasked columns go to the model provider while the context is generated. Descriptions and enum codes come out better. Off by default. |
| `--context <dir>` | Where to write the context. Default: `contexts/my-project`. |

Without `--url`, `npm run init` asks the same things interactively.

What the quick start leaves for a person:

1. **Review the generated context.** It is written by a model from your schema. Start with
   personal data, which is handled in two levels:
   - **Masked** (`masked-columns.json`): direct identifiers and secrets, such as e-mail,
     phone, street address, postal code, national ids, birth date, salary and credentials.
     Rosie never reads them or sends them to the model.
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

The SQL in Steps 2 and 3 is written for MySQL and MariaDB. For PostgreSQL the same steps,
with their statements, are in `docs/POSTGRES.md`; for SQL Server in `docs/SQLSERVER.md`.

### Prerequisites

- Node.js **22 or newer** (24 LTS recommended; `.nvmrc` pins 24 and `.npmrc` refuses older runtimes because Node 18 is end-of-life) and access to the Rosie repository.
- Access to **the host app's database** (MySQL, MariaDB, PostgreSQL or SQL Server: host, port, schema) with permission to create a user
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

```sql
CREATE USER 'rosie_readonly'@'%' IDENTIFIED BY '<strong password>';
GRANT SELECT ON <my_schema>.* TO 'rosie_readonly'@'%';
-- Grant nothing beyond SELECT.
FLUSH PRIVILEGES;
```

This grant covers every table of `<my_schema>`, now and later. That is why Step 3 puts
Rosie's own tables in another database: the read-only user must not be able to read them.

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
  `promptHidden`, and a `primaryKey` written by hand when the database declares none, as
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
PII columns the validator refuses to return (also blocks `SELECT *` on the table):
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
AI_MODEL=gpt-6-luna                          # for a pilot use gpt-6.1-terra: see docs/MODELS.md
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
If `project-info` returns `securityHold: true`, the Step 2 probe failed. Review the
read-only user's privileges. If `health` says `degraded` with `schemaBlockedTables` after the
first question, the engine tables are not where `DB_ENGINE_DATABASE` points (Step 3).

---

## Step 6. Plug the widget into the host app's front end

Rosie ships a self-contained Web Component. **Two lines of HTML, no file copy.**

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

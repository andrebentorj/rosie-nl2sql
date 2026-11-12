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
   └─ rosie-widget.js (copied              ├─ context dir      (ROSIE_CONTEXT_DIR)
      into its assets)                      └─ its MySQL  (read-only pool, SELECT-only)
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
| `--create-ro-user` | Creates `rosie_readonly`, a user that can only SELECT, with a generated password written to the `.env`. **Writes to your database**; needs an account allowed to create users. If it cannot, the statements to run are printed. |
| `--create-engine-tables` | Creates Rosie's three audit tables. **Writes to your database.** Add `--engine-db <name>` to keep them in a separate database. |
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
3. **Measure it**: `npm run eval` runs the generated question bank and costs a few cents.

`npm run doctor` checks the whole installation at any time and prints the fix next to each
problem. `npm run mcp:config` prints the MCP client configuration again.

### With Docker, without cloning

`<image>` below is the image you were given access to, for example
`ghcr.io/andrebentorj/rosie:1.0.0`. Log in to the registry first with the token you created
(`docker login ghcr.io`), or load the file you received (`docker load < rosie-1.0.0.tar.gz`).

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
`/context`. The image does not carry the evaluation harness, so the wizard skips its dry run
there; measure accuracy from a checkout. Run the server with that directory mounted and the
variables from the generated env file (see the comment at the top of the `Dockerfile`).

---

## Manual installation (reference)

### Prerequisites

- Node.js **22 or newer** (24 LTS recommended; `.nvmrc` pins 24 and `.npmrc` refuses older runtimes because Node 18 is end-of-life) and access to the Rosie repository.
- Access to **the host app's database** (MySQL, MariaDB, PostgreSQL or SQL Server: host, port, schema) with permission to create a user
  and (ideally) Rosie's engine tables.
- An **LLM API key** (OpenAI, or any OpenAI-compatible provider — see §7 for Claude).
- The **origin** of the host app's front end (e.g. `https://app.example.com`) — needed for CORS.

---

## Step 1 — Boot the Rosie backend and verify it starts

```bash
git clone <repository url> rosie  &&  cd rosie
npm ci
cp .env.example .env
```

Don't fill everything in yet; first confirm the process boots. The `.env` is finalized in
Step 5. Several modules `throw` at load time if required env vars are missing — this is
intentional (fail-closed).

---

## Step 2. Create the read-only database user (DO NOT SKIP)

Rosie runs **every** AI query through a dedicated pool that must have **SELECT only**. At
startup a permission probe (`security/db-permission-check.js`) checks this: if the user has
any privilege beyond SELECT, the **server stays up but every `/ask` and `/export` returns
503**. This is the #1 install mistake.

```sql
CREATE USER 'rosie_readonly'@'%' IDENTIFIED BY '<strong password>';
GRANT SELECT ON <my_schema>.* TO 'rosie_readonly'@'%';
-- Grant nothing beyond SELECT, and do not give access to the engine tables (Step 3).
FLUSH PRIVILEGES;
```

---

## Step 3 — Create the engine tables (main pool)

Rosie writes audit and feedback rows to the **main pool** (not the read-only one). Run the
DDL in `db/schema.sql` against the host app's schema **with a user that has CREATE/INSERT**:

```bash
mysql -u <main_user> -p <my_schema> < db/schema.sql
```

This creates `rosie_audit_log` (one row per `/ask`) and `rosie_feedback` (👍/👎). The
read-only user from Step 2 must **not** be able to see these tables.

---

## Step 4 — Build the host app's context dir

> **⚡ Fast path — `npm run init`.** The context-bootstrap wizard automates Steps 4–5:
> it interviews you (project identity, governance, DB credentials, language),
> introspects the schema, and uses the same LLM Rosie runs on to generate **every**
> context file — including LLM-enriched catalog semantics, `verified-queries.json`
> validated against the real engine validator AND executed on your DB, and an eval
> bank whose expectations are computed from actual data. It also writes the `.env`
> (never overwriting an existing one without confirmation). Steps 2 (read-only user),
> 3 (engine tables — the wizard offers it, default no) and 6 (widget copy) stay
> manual. **Review everything it generates** — LLM output is provisional; the final
> report lists exactly what to check. The manual path below remains fully supported.

Create a directory outside the repo (e.g. in the host app's repo or a mounted volume). The fastest
start is to **copy the template** under `contexts/example/`, which now ships skeleton files
for every required and optional file:

```
my-rosie-context/
├── project.json            # required
├── system-prompt.txt       # required — all of the host app's domain knowledge
├── schema-catalog.json     # required — schema + security allowlist (generated, Step 4c)
├── masked-columns.json     # required — PII columns to block
├── verified-queries.json   # optional — few-shot question→SQL pairs
├── clarification-rules.json# optional — deterministic disambiguation chips
└── eval/questions/*.json   # optional, recommended — evaluation bank
```

```bash
cp -r contexts/example/ /path/to/my-rosie-context
# then edit each file for the host app (the templates use a generic users/orders schema)
```

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
metrics). The engine **automatically appends** a catalog summary and the CLARIFY protocol —
don't repeat the schema here; focus on domain meaning.

### 4c. `schema-catalog.json` — generate it (it's also the security allowlist)
Every table/column a query can reference must be here; a query touching anything **outside**
the catalog is rejected. A table with an empty column list **aborts startup** (fail-closed).
Generate it directly from the host app's database:

```bash
# .env must already have DB_* and ROSIE_CONTEXT_DIR set (Step 5)
npm run generate-catalog
# or target a specific output / schema:
node scripts/generate-catalog.js --out /path/to/my-rosie-context/schema-catalog.json --database my_schema
```

This introspects `INFORMATION_SCHEMA` and fills the **structural** fields (tables, columns +
types, primary keys, foreign-key relationships, approximate row counts). It **preserves** any
**semantic** enrichment you add by hand — table/column `synonyms`, `description`, `enumValues`,
and root-level `metrics` — by merging into the existing catalog on re-run. So the workflow is:
generate once, enrich by hand, regenerate freely without losing your edits.

> Enrich the high-value bits: synonyms for how users actually speak, `enumValues` for status
> columns, and named `metrics`. A bare structural catalog works but a thin one **silently
> breaks valid questions** the model can't map.

### 4d. `masked-columns.json`
PII columns the validator refuses to return (also blocks `SELECT *` on the table):
```json
{ "users": ["email", "cpf", "phone"] }
```

---

## Step 5 — Finalize the backend `.env`

```bash
# ─── Server ───
ROSIE_PORT=3001
ROSIE_CORS_ORIGIN=https://app.example.com        # the host app's EXACT front-end origin

# ─── Auth (production: a long random string) ───
ROSIE_API_TOKEN=<strong random token>

# ─── LLM ───
OPENAI_API_KEY=sk-...
AI_MODEL=gpt-6-luna
AI_ASSISTANT_ENABLED=true
# AI_BASE_URL=                               # set to swap providers — see §7

# ─── DB main (used ONLY for Rosie's own tables: audit/feedback) ───
DB_HOST=...  DB_PORT=3306  DB_USER=<main_user>  DB_PASSWORD=...  DB_DATABASE=<my_schema>
# RECOMMENDED: dedicated schema for rosie_audit_log/rosie_feedback so the main
# user needs NO write grant on <my_schema> (npm run init offers to create it):
#   CREATE DATABASE rosie;  GRANT SELECT, INSERT ON rosie.* TO '<main_user>'@'%';
DB_ENGINE_DATABASE=rosie

# ─── DB read-only (all AI queries — the Step 2 user) ───
DB_RO_HOST=...  DB_RO_PORT=3306
DB_RO_USER=rosie_readonly  DB_RO_PASSWORD=...  DB_RO_DATABASE=<my_schema>

# ─── the host app context dir (ABSOLUTE path) ───
ROSIE_CONTEXT_DIR=/absolute/path/to/my-rosie-context
```

Start and confirm a clean boot:
```bash
npm start
curl localhost:3001/api/assistant/health        # should return OK
curl localhost:3001/api/assistant/project-info   # should reflect the host app's project.json
```
If `project-info` returns `securityHold: true`, the Step 2 probe failed — review the
read-only user's privileges.

---

## Step 6 — Plug the widget into the host app's front end

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

> 🔒 **Security topology (mandatory in production):** do **NOT** set the `api-token`
> attribute. It places the shared `ROSIE_API_TOKEN` in every logged-in user's DOM, where
> anyone can extract it and query the whole allowlisted schema directly with `curl`,
> bypassing the host app entirely (CORS restrains browsers, not scripts). The supported
> topology is: the host app reverse-proxies `/api/assistant/*` **same-origin** and its
> proxy injects `Authorization: Bearer <ROSIE_API_TOKEN>` server-side, where the secret
> stays. The widget still *accepts* `api-token` for local development, but logs a console
> security warning whenever it is set — treat that warning as a deploy blocker.

### 6c. Push the logged-in user (after auth resolves)
```js
const widget = document.querySelector('rosie-widget');
widget.setAttribute('user-id',   user.identifier);   // audit/personalization only
widget.setAttribute('user-name', user.displayName);  // NEVER authorization
```

### 6d. Routing `api-url` → backend
`api-url="/api/assistant"` assumes the host app **reverse-proxies** `/api/assistant/*` to the Rosie
backend (**the only supported production topology** — avoids CORS, exposes a single domain,
and keeps `ROSIE_API_TOKEN` server-side; the proxy adds the `Authorization` header). Nginx
sketch:

```nginx
location /api/assistant/ {
    proxy_pass         http://rosie-backend:3001/api/assistant/;
    proxy_set_header   Authorization "Bearer <ROSIE_API_TOKEN>";
}
```

Pointing `api-url` at the backend's absolute URL (with `ROSIE_CORS_ORIGIN` set) is for
development only — it forces the token into the page to authenticate, which is exactly
the exposure the proxy topology exists to prevent.

---

## Step 7 — (Optional) Swap the LLM (OpenAI → Claude)

The provider is **pluggable via config** — the engine speaks an OpenAI-compatible wire
format, so any OpenAI-compatible endpoint works with **no code change**. To run on Claude,
use Anthropic's OpenAI-compatible endpoint:

```bash
AI_BASE_URL=https://api.anthropic.com/v1/
OPENAI_API_KEY=sk-ant-...                    # the var name stays OPENAI_API_KEY
AI_MODEL=claude-haiku-5-5                     # low cost/latency; or claude-sonnet-5-5
```

Leave `AI_BASE_URL` empty for the OpenAI default. The same mechanism works for OpenRouter,
LiteLLM, or any compatible gateway — set `AI_BASE_URL` + matching key/model. After switching
providers, re-run the eval (§8) to confirm accuracy on the host app's question bank.

---

## Step 8 — Validate before the demo

```bash
node test/security/battery.js                       # security battery (no DB/LLM)
node -r dotenv/config test/eval/run.js --dry-run    # validate the host app's question JSONs
node -r dotenv/config test/eval/run.js              # real NL2SQL eval (costs LLM $)
```

Final checklist:
- [ ] `health` and `project-info` OK; no `securityHold`.
- [ ] Read-only user has SELECT only (Step 2).
- [ ] Engine tables created on the main pool (Step 3).
- [ ] Catalog covers every table/column the target questions touch.
- [ ] Widget re-copied and responding in the host app; user-id/name flowing.
- [ ] CORS/proxy resolved.

---

## Summary — what changes per project vs what's fixed

| Per project                               | Fixed (Rosie engine)                    |
|-------------------------------------------|-----------------------------------------|
| context dir (prompt, catalog, masks)      | SQL validator, filters, tool-use loop   |
| `.env` (DB, CORS, token, model, base URL) | routes `/api/assistant/*`                |
| read-only user + engine tables            | widget (same bundle for all hosts)      |
| widget loaded from /api/assistant/widget.js (or re-copied) | security battery                         |

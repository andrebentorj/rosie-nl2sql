# Rosie inside your product (OEM)

This guide is for a software vendor that embeds Rosie in its own product and offers it to its
own customers. In this guide the vendor is **you**, and each of your customers is a
**tenant**. Everything here is on top of a normal installation ([INSTALL.md](INSTALL.md)):
read [IDENTITY.md](IDENTITY.md) and [ROW_POLICIES.md](ROW_POLICIES.md) first.

## What tenancy guarantees

With tenancy on, every request that reads data or asks the model names exactly one tenant,
and that tenant comes from a **verified token** your backend signs. Nothing the browser sends
can choose it: not the request body, not a header, not a query string, not the widget's
attributes. The tenant then decides:

| What | How it is kept apart |
|---|---|
| Rows | Model A: the row policies of the shared database, keyed on the tenant claim, on every statement. Model B: the tenant's own database (or schema), through a connection pool that only ever opens that database. |
| Answers | An answer is computed from the tenant's rows only, certificate probes included. |
| Answer handles | Every handle carries the tenant under its signature. A handle of tenant A is refused for tenant B, even for a user id both tenants have. |
| Exports and explanations | They redeem a handle, so the same check applies, and an export re-runs the statement under the redeemer's own tenant. |
| Audit | Every audit record names its tenant (`tenant_id`). Tenants never read the audit trail: no route a tenant can call returns it. |
| Quotas | A tenant's people draw from one tenant quota, on top of each person's own quota. |
| Scheduled answers | A schedule runs as one tenant, named in its service identity, and only on that tenant's data. |

A request with tenancy on and **no tenant** fails closed: it is refused at the door (403), and
the one place where a statement reaches a database refuses it again if a code path ever forgot
the tenant.

## Choose a model

| | Model A: one shared database | Model B: a database per tenant |
|---|---|---|
| Your schema | Every tenant's rows in the same tables, with a tenant column (`tenant_id`) | The same tables in one database (or PostgreSQL schema) per tenant |
| Isolation by | Row policies on the tenant column, checked at startup | The connection: the tenant's pool opens its database only |
| One context directory | Yes | Yes, or one per tenant when their catalogs differ |
| Shared reference tables (currencies, countries) | Tables without the tenant column, read by every tenant alike | Copied into each tenant's database |
| Strength | Simple to run; one set of grants | A missing policy cannot leak rows; easy to delete one tenant's data |

You may mix them: a large customer on a database of its own, the others on the shared one.

### Model A: a tenant column with mandatory row policies

Declare the tenant column in `project.json`:

```json
{
  "name": "Shop analytics",
  "tenancy": { "column": "tenant_id", "claim": "tenant_id" }
}
```

and give **every** table that has that column a row policy that requires it to equal the claim
(`row-policies.json`):

```json
{
  "orders":    "tenant_id = :claims.tenant_id",
  "customers": "tenant_id = :claims.tenant_id",
  "tickets":   "tenant_id = :claims.tenant_id AND region IN (:claims.regions)"
}
```

Rosie does not rely on your predicate for the tenant condition. For every table that has the
tenant column, the engine itself reads your policy inside its own parentheses and adds the
tenant condition outside them: `(<your policy>) AND <column> = <the tenant claim>`. It builds
this as a syntax tree, so no grouping inside your predicate can loosen it, and a policy that
compares the column with something else only narrows it further.

The context does not load (the server does not start, a reload keeps the context in service,
and `npm run check-context` fails with the list) when:

- a table of the catalog has the tenant column and no row policy;
- a policy has an `OR` (also `XOR`, and `||`, which MySQL reads as `OR`) that is not in
  parentheses of its own. SQL gives `AND` precedence over `OR`, and Rosie's parser groups them
  from left to right, so an unparenthesised `OR` is read two ways: write
  `(a = 1 OR b = :claims.region) AND ...`;
- the tenant column is masked (a policy cannot filter on a masked column);
- no table of the catalog has the declared column.

**Tenant ids must be exactly the values stored in the tenant column, and a customer must never
choose its own id.** The database compares them its own way: with a case-insensitive collation
(the default on MySQL and SQL Server) `ACME` matches the rows of `acme`, and on MySQL a numeric
tenant column compares `'1x'` and `'01'` as `1`. Use a text column with the exact ids your
backend signs, or a binary or case-sensitive collation.

Rosie writes its tenant condition with the tenant id as a **string**, also when the token carries
it as a JSON number (`"tenant_id": 7` becomes `tenant_id = '7'`). On a text column that compares
exactly, so `7` never matches `07`, `7.0` or ` 7`. On a numeric column the database converts the
string to the column's type (MySQL, PostgreSQL and SQL Server all do), so `'7'` matches `7`, and
a token that says `"07"` reads the rows of tenant 7. Use the type your ids really have: a text column for
text ids, a numeric column only for ids that are plain whole numbers, signed in their shortest
form.

Tables without the column are shared by every tenant. `npm run check-context` lists them under
"Tenant isolation" so you can confirm each one is meant to be common to all.

On PostgreSQL, native row-level security on the same column is a good second layer under this
one ([ROW_POLICIES.md](ROW_POLICIES.md), "What it is not").

### Model B: a database or schema per tenant

List the tenants in a tenants file and point `ROSIE_TENANTS_FILE` at it:

```json
{
  "tenants": {
    "acme":    { "database": "shop_acme" },
    "globex":  { "database": "shop_globex", "quota": { "questionsPerDay": 2000 } },
    "initech": { "schema": "tenant_initech" },
    "umbrella": {}
  }
}
```

| Key | Meaning |
|---|---|
| `database` | The tenant's database. Its pool uses the read-only account of `.env` (`DB_RO_USER`) and the same host, timeout and session guards as the installation's pool. |
| `schema` | PostgreSQL only: the tenant's schema in the installation's database. It is the only schema on the pool's search path. |
| `context` | A context directory of its own (path relative to the file), when this tenant's catalog differs. Default: `ROSIE_CONTEXT_DIR`. |
| `quota` | `{ "questionsPerHour": n, "questionsPerDay": n }`, overriding the installation's tenant quota (0: no limit). |
| `brand` | The assistant's name and icon for this tenant (see Branding). |
| `name`, `note` | For you; not used. |

A tenant with neither `database` nor `schema` (`umbrella` above) is a model A tenant: it is
answered from the installation's database through the row policies, so the context that
serves it must declare `tenancy`.

With a tenants file, a tenant the file does not list is refused (`403 E_TENANT_UNKNOWN`). The
server refuses to start when the file cannot be read, a tenant id is outside the alphabet
(1 to 64 letters, digits and `_ . : @ -`, case is kept and two ids that differ only in case are
refused), two tenants are given the same database, a model B tenant is given the database the
model A tenants share, or a context the file names does not load.

**A context directory shared by several model B tenants is the same for all of them**: its
catalog (descriptions, `enumValues`, the column profile's notes), its approved and example
queries and its prompt reach every one of those tenants, through the model's prompt and the
answers. Keep customer values out of them: a literal taken from one tenant's data in an
approved query, or an enum code only one customer uses, is shown to the others. Give a tenant
whose catalog must differ a `context` of its own.

Grants: the read-only account needs `SELECT` on each tenant database (or schema), and nothing
else. Rosie refuses a statement that names another database or schema, so one tenant's
questions cannot reach another tenant's database through the shared account; for a stronger
wall, run one installation per group of tenants with an account of its own.

## Turn tenancy on

```bash
ROSIE_JWT_ISSUER=https://auth.your-product.example/
ROSIE_JWT_AUDIENCE=rosie
ROSIE_JWT_JWKS_URL=https://auth.your-product.example/.well-known/jwks.json
ROSIE_TENANT_CLAIM=tenant_id
# Model B or a mix:
ROSIE_TENANTS_FILE=/etc/rosie/tenants.json
# Optional: questions per tenant, all its people together (0: none)
ROSIE_TENANT_QUESTIONS_PER_HOUR=0
ROSIE_TENANT_QUESTIONS_PER_DAY=5000
```

| Setting | Meaning |
|---|---|
| `ROSIE_TENANT_CLAIM` | The claim of the verified token that names the tenant. Setting it turns tenancy on. A namespaced claim is given a short name with `ROSIE_JWT_CLAIM_MAP` first. |
| `ROSIE_TENANTS_FILE` | The tenants file. Optional in model A. |
| `ROSIE_TENANT_QUESTIONS_PER_HOUR`, `_PER_DAY` | The tenant quota. With `ROSIE_RATE_LIMIT_STORE=database` it is counted once for every replica. |

With tenancy on:

- The static `ROSIE_API_TOKEN` is still accepted by the authentication step, but it carries no
  claims, so every question, export, explanation, rating and certified statement made with it
  is refused (`403 E_TENANT_REQUIRED`).
- Slack and Teams cannot be turned on (their identities carry no tenant claim): the server
  refuses to start with either.
- MCP over HTTP takes the tenant from its JWT the same way; the stdio transport has no
  identity, so it reads no data.
- The chat page with sign-in works when the identity provider puts the tenant claim in the ID
  token (or `ROSIE_JWT_CLAIM_MAP` maps it).

## The embedding token flow

```
 Browser (your product's page)        Your backend                 Rosie
 ─────────────────────────────        ────────────                 ─────
 1. user signs in to your product ──> session
 2. page asks for a Rosie token  ──> checks the session,
                                     looks up the user's tenant,
                                     signs a JWT (5 to 15 minutes):
                                       iss, aud=rosie, sub, exp,
                                       tenant_id
 3. page gets the JWT            <──
 4. <rosie-widget api-token="JWT"> ─────────────────────────────> verifies the signature,
                                                                  iss, aud, exp; reads
                                                                  tenant_id; answers from
                                                                  that tenant's data only
 5. before exp, the page asks again (step 2) and sets the new token on the widget
```

What the token must carry:

| Claim | Value |
|---|---|
| `iss` | Your issuer, equal to `ROSIE_JWT_ISSUER` |
| `aud` | `ROSIE_JWT_AUDIENCE` (for example `rosie`) |
| `sub` | The user's id in your product (answer handles and quotas are bound to it) |
| `exp` | Short: 5 to 15 minutes. Required; a token without it is refused |
| `tenant_id` | The tenant, from your own records, never from the browser |
| `name` | Optional: shown in the audit trail |

Sign with an asymmetric key (RS256, ES256, PS256 or EdDSA) and publish the public key at the
JWKS URL; `HS*` is refused. The private key stays on your backend. Details and the full list of
checks: [IDENTITY.md](IDENTITY.md). The widget attribute is `api-token`; with a user's JWT the
widget's console warning about the attribute is expected ([IDENTITY.md](IDENTITY.md), "Sending
the token from the widget").

## Checklist

1. Pick model A, model B or a mix, per tenant.
2. Model A: add `tenancy` to `project.json` and a row policy to every table with the tenant
   column. Run `npm run check-context` until "Tenant isolation" is ok, and read the list of
   tables every tenant shares.
3. Model B: create each tenant database (or schema), grant `SELECT` on it to the read-only
   account, and list the tenant in the tenants file.
4. Set up signing on your backend: a key pair, the JWKS URL, a token endpoint that signs the
   claims above for the signed-in user.
5. Set `ROSIE_JWT_*`, `ROSIE_TENANT_CLAIM` and, for model B, `ROSIE_TENANTS_FILE`. Start Rosie:
   a configuration it cannot honour stops the startup with one line saying why.
6. Add the `tenant_id` column to the audit table if the engine tables predate 1.38.0 (below).
7. Embed the widget with the user's JWT in `api-token` and refresh it before it expires.
8. Test with two tenants: the same question must give each its own figures, and a token
   without the tenant claim must be refused.
9. Set the tenant quotas, and schedule `npm run usage` for your billing period.

## What you must never do

- **Never put `ROSIE_API_TOKEN` in a browser**, in the widget's attribute, in page source or
  in a mobile app. It is the installation's secret, not a user's credential.
- **Never let the browser choose the tenant.** The tenant comes from your records, on your
  backend, into the signed token. Do not sign whatever tenant the page asks for.
- **Never sign long-lived tokens** for the widget. Minutes, not days.
- **Never share one tenant's database with another in model B**, and never give a model A
  context a table with the tenant column and no policy (Rosie refuses both at startup).
- **Never place a proxy that replaces the user's `Authorization` header** with the static
  token in front of Rosie.
- **Never expose the engine database or the audit file to tenants.** They hold every tenant's
  records.

## Usage and billing

```bash
npm run usage -- --by tenant --from 2026-11-01 --to 2026-11-30
npm run usage -- --by tenant --from 2026-11-01 --to 2026-11-30 --format csv --out usage-november.csv
npm run usage -- --by tenant,day --from 2026-11-01 --to 2026-11-30 --format json
npm run usage -- --tenant acme --from 2026-11-01 --to 2026-11-30
npm run usage -- --file ./logs/rosie-audit.ndjson --from 2026-11-01 --to 2026-11-30
```

It reads the audit trail and changes nothing: the engine database by default, or the audit file
with `--file` (every record is in the file, with its tenant, also when the database was
unavailable). Days are whole UTC days, both included. One row per tenant (and per day with
`--by tenant,day`):

| Column | Counts |
|---|---|
| `questions` | Questions asked, refusals before the model included, scheduled runs and certified statements too |
| `answered` | Questions whose statement ran |
| `scheduled` | Of the questions, runs of scheduled answers |
| `verdict_checks_passed` ... `verdict_not_applicable`, `no_certificate` | Answered questions by the verdict of their Answer Certificate |
| `exports`, `export_rows`, `explanations` | Follow-ups |
| `blocked` | Questions refused by a safety rule |
| `refused` | Requests turned away at the door (quotas, no tenant), with the folded ones counted |
| `people` | Distinct people |
| `tokens_input`, `tokens_output` | Model tokens, where the record has them |

Records without a tenant appear under `(none)`, so the totals add up to the trail. In a CSV a
value that a spreadsheet would run as a formula is written as text.

## Audit and usage

Every audit record of a tenant's request carries `tenant_id`. An installation created with
1.38.0 or later has the column (`db/schema*.sql`). An older engine table keeps recording without
it, the tenant goes to the audit file only, and the log says so once
(`audit_tenant_column_missing`); add the column as an administrator on the engine database:

MySQL:

```sql
ALTER TABLE rosie_audit_log ADD COLUMN tenant_id VARCHAR(64) NULL, ADD KEY idx_rosie_audit_tenant (tenant_id, created_at);
```

PostgreSQL:

```sql
ALTER TABLE rosie_audit_log ADD COLUMN IF NOT EXISTS tenant_id VARCHAR(64) NULL; CREATE INDEX IF NOT EXISTS idx_rosie_audit_tenant ON rosie_audit_log (tenant_id, created_at);
```

SQL Server:

```sql
ALTER TABLE rosie_audit_log ADD tenant_id NVARCHAR(64) NULL; CREATE INDEX idx_rosie_audit_tenant ON rosie_audit_log (tenant_id, created_at);
```

An installation without tenancy never writes the column, so it needs none of this.

## Branding

You may give the assistant **your own name and icon**, for the whole installation
(`project.json` `"brand": { "assistantName", "icon" }`, icon path relative to the context
directory) or per tenant (`"brand"` in the tenants file, icon path relative to that file):

```json
{ "tenants": { "acme": { "database": "shop_acme",
                         "brand": { "assistantName": "Acme Analytics", "icon": "brand/acme.svg" } } } }
```

Where they appear:

| Place | With your brand |
|---|---|
| Widget launcher | Your icon (64 px, rounded corners) instead of Rosie |
| Widget header and results screen | Your icon and your name |
| Widget disclaimer line | "Powered by Rosie" at its end |
| Chat page sign-in and error screens | Your icon in place of the mark, "Sign in to <your name>", "Powered by Rosie" in the footer |
| Chat page top bar | Your icon before the project name |

Colours, layout and the Answer Certificate do not change. "Powered by Rosie" is shown whenever
the assistant has a name other than Rosie or an icon, and no setting removes it. A widget or a
page without a brand looks exactly as before.

A tenant sees its own brand: the widget asks `/project-info` with the user's JWT (or the chat
page's session), and the server answers with that tenant's name and icon. The sign-in screen,
shown before anyone is known, carries the installation's brand.

The icon is a file of yours: PNG (square, 32 to 1024 px), SVG (with a `viewBox`, no script,
event handler, `foreignObject` or reference outside the file) or WebP, at most 64 KB. A file
that breaks a rule stops the startup with the reason. Rosie reads it once and serves exactly
those bytes at `/api/assistant/brand/icon/<digest>.<ext>`, with its content type, `nosniff`, a
sandboxing Content-Security-Policy and a one-year cache (the address changes when the file
does). No other file can be fetched through that route.

**Installation-only settings.** `project.json` `theme` (colours) and `brand.hideMascot`, and
`brand.tagline`, keep working for an installation's own owner, in the `project.json` of
`ROSIE_CONTEXT_DIR`, exactly as before ([WIDGET.md](WIDGET.md)). A tenant or partner brand can
never use them: a tenant's `brand` in the tenants file with `theme`, `hideMascot` or `tagline`,
and a tenant's own context directory whose `project.json` has `theme` or `brand.hideMascot`,
stop the startup with a message naming the key.

## Limits of this release

- The privilege check of the read-only account and the masked-column grant check
  (`ROSIE_REQUIRE_COLUMN_GRANTS`) run on the installation's database. Give every tenant database
  the same grants as that one; `npm run doctor` checks the installation's database only.
- `GET /project-info` shows the installation's welcome text and suggestions to every tenant;
  only the name and the icon are the tenant's.
- Scheduled answers are read from the installation's context directory, and a schedule whose
  tenant is answered from another context directory does not run.
- The administration console reads no tenant's data while tenancy is on.
- Quotas count a person as the tenant and the user id together, so user `1` of one tenant and
  user `1` of another have quotas of their own. `npm run subject` finds a person by user id
  alone: if your user ids repeat across tenants, it finds every tenant's person with that id.
  Check the `tenant_id` of what it lists before you erase, or sign user ids that are unique
  across tenants (for example prefixed with the tenant).

## Licence

The terms under which you may embed and resell Rosie are set in your agreement with the
owner of Rosie. They are agreed per partner. To discuss them, write to [andrebentorj@gmail.com](mailto:andrebentorj@gmail.com?subject=Rosie%20OEM).

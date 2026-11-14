# Rosie Context Directory: Example

Each project that uses Rosie must provide its own context directory with these files.
This contract is what the bootstrap wizard (`npm run init`) generates. Use the wizard for a
fast start, or build the files by hand from this template:

```
rosie-context/
├── project.json             # required. Project metadata, welcome message, suggestions
├── system-prompt.txt        # required. LLM system prompt with domain knowledge
├── schema-catalog.json      # required. Database schema and allowlist (scripts/generate-catalog.js)
├── masked-columns.json      # PII columns to exclude from queries. Without it nothing is masked
├── verified-queries.json    # optional. Examples for the model, and approved queries
├── clarification-rules.json # optional. Questions to ask back before the model is called
├── row-policies.json        # optional. Per-user or per-tenant row isolation
└── eval/
    ├── bank.json            # optional. The bank of `npm run evaluate`
    └── questions/           # optional. The bank of `npm run eval` (JSON files)
```

This directory ships the first six. The server refuses to start without the three required
files. `npm run check-context -- --context <dir>` checks a directory against this contract
with no database and no model (`docs/OPERATIONS.md`).

Set the env var `ROSIE_CONTEXT_DIR` to point to your project's context directory.

Example: `ROSIE_CONTEXT_DIR=/path/to/myproject/rosie-context`

## The two evaluation banks

- `eval/bank.json` is a list of `{ "id", "question", "sql" }`, where `sql` is a statement
  whose result is the right answer (add `"ordered": true` when row order is part of it).
  `npm run evaluate` asks each question and compares results; approved queries without
  parameters are added to the bank automatically. `npm run loadtest`, `npm run
  check-context` and the `rosie_evaluation_*` metrics read this file. The wizard does not
  write it. See `docs/EVALUATION.md`.
- `eval/questions/*.json` is the bank the wizard generates, read by `npm run eval`
  (`test/eval/run.js`), which needs a checkout of the repository.

## Project identity in `project.json`

`name` is the name the widget shows. The project id, which answer
handles, the MCP scope (`rosie:<id>:ask`) and the certificate's provenance carry, is
`projectId` when you set it, otherwise `name`, otherwise the directory name, lower-cased
with anything outside letters, digits, `-` and `_` turned into `-`. Set `projectId` when two
projects could end up with the same name. A context reload refuses to change it; that needs
a restart.

## The engine name in `system-prompt.txt`

Write `{{SQL_DIALECT}}` where the prompt names the database engine. It is replaced by
`MySQL`, `PostgreSQL` or `SQL Server` according to `DB_DIALECT`, so one context directory
serves more than one engine.

## Calendar conventions in `project.json` (optional)

Relative time ("this quarter", "last year", "this week") depends on conventions the engine
cannot infer. Declare them and they are rendered into the per-request date block:

```json
"calendar": {
  "fiscalYearStartMonth": 7,        // 1 = calendar year; 7 = FY starts in July
  "weekStartsOn": "Monday",
  "timezone": "America/New_York",   // how date/time columns are stored
  "completedPeriodsOnly": true,     // "last month" = most recent COMPLETED month
  "notes": "Trading weeks run Friday to Thursday."
}
```

## Governance fields in `project.json`

Good AI governance gives every agentic deployment a **named owner,
business purpose, and approval**. Declare them in `project.json`. The server warns at
startup when the block is missing, and exposes it via `GET /project-info`:

```json
{
    "governance": {
        "owner": "Jane Doe (jane.doe@example.com)",
        "businessPurpose": "Self-service analytics over the order data for the ops team",
        "approvedBy": "Head of Data",
        "approvedDate": "2026-07-01",
        "dataSources": ["sales (read-only MySQL)"]
    },
    "aiDisclaimer": "Optional override of the widget's verify-your-outputs notice."
}
```

`aiDisclaimer` (optional) overrides the wording of the widget's AI-output disclaimer;
when absent, the engine's default notice is shown. It is always on.

### Widget theme (optional)

Each project can re-color the widget UI (the mascot keeps its own colors: it is the
brand, not the theme). All keys optional; colors are hex only, radius is `<n>px`:

```json
{
    "theme": {
        "primary": "#0b6bcb",
        "primaryDeep": "#084c93",
        "primarySoft": "#5ba3e8",
        "primaryTint": "#e3effb",
        "radius": "12px"
    }
}
```

Also available: `surface`, `surfaceAlt`, `surfaceDim`, `ink`, `inkSoft`, `inkFaint`,
`border`, `borderSoft`. When a theme is set it also overrides the widget's automatic
dark-mode palette (explicit beats auto).

### White label (optional)

To present the assistant under your own name, add a `brand` block. All keys optional:

```json
{
    "brand": {
        "assistantName": "Ada",
        "tagline": "Data Assistant",
        "hideMascot": true
    }
}
```

`assistantName` replaces "Rosie" in the header, the input placeholder, the default welcome
message and the results window. `tagline` is the small text next to the name (an empty string
removes it). `hideMascot` swaps the mascot for a neutral chat icon in the theme colour. Names
are limited to 40 characters: letters, digits, spaces and `. & ' -`. Anything else is ignored
and the default is shown. Set your own `welcomeMessage` as well if it mentions the name.

## Optional files and fields

### `verified-queries.json` (optional)
Known-correct question→SQL pairs. For each user question, the engine retrieves the most
similar pairs (lexical similarity) and injects them as few-shot examples. Plain entries are
prompt examples only and are never executed, so they cannot bypass the SQL validator.

```json
[
    {
        "question": "Top 5 customers by number of paid orders",
        "sql": "SELECT u.name, COUNT(*) AS paid_orders FROM orders o JOIN users u ON o.uid = u.id WHERE o.status = 'paid' GROUP BY u.id, u.name ORDER BY paid_orders DESC LIMIT 5",
        "note": "optional hint shown to the model alongside the example"
    }
]
```

An entry that names who approved it (`approvedBy`, with an `id` and optionally `params`,
`approvedOn`, `reviewBy`) becomes an **approved query**: the model may run it by id with
different parameter values, never a different statement, and the answer says a person
reviewed it. See `docs/APPROVED_QUERIES.md`.

### Semantic fields in `schema-catalog.json` (all optional)
A compact summary of the catalog (tables, relationships, metrics) is injected into the
system prompt automatically, and each table also carries one line with its
unmasked column names, type family and enum codes (`AI_CATALOG_COLUMNS`, default on). The
model therefore writes SQL from the prompt and calls `describe_table` only for a column's
meaning. Do NOT write "always call describe_table before every query" into your
`system-prompt.txt`: that instruction alone costs one or two model round-trips per question.
Richer fields improve both the summary and the `describe_table` tool output:

- **table.promptHidden** `true`: keeps a rarely asked table (audit shadows, technical logs)
  out of the detailed prompt block to save tokens on every request. The table is only named
  on an `OTHER TABLES` line; it stays fully queryable and visible to `list_tables` and
  `describe_table`. This is prompt shaping only and is NOT a security control: to forbid a
  table, remove it from the catalog.
- **table.synonyms** `["customers", "clients"]`: alternative names users say.
- **column.description**: what the column means.
- **column.synonyms**: alternative names for the column.
- **column.measure** `true`: the column is a value of each record of its table (the total of
  an order, the amount of a payment), to be added once per record. It is what lets the Answer
  Certificate call a total that repeats those records wrong; without it (or a declared metric
  that adds the column up) such a total is reported as two possible readings. Leave it off
  for attributes that are legitimately added once per related row (the length of a track per
  sale). `npm run init` lists the columns that look like one in its report and marks none of
  them: only someone who knows the data can say it. See `docs/CERTIFICATE.md`.
- **column.enumValues** `{ "void": "cancelled by the user", ... }`: meaning of each code
  (essential for status-like columns).
- **catalog.metrics** (root level): named business metrics. A value may be free text
  (`"gross_revenue": "SUM(orders.total) ..."`) or a structured definition with `expression`,
  `table`, `filters`, `dimensions` and `synonyms`, which the Answer Certificate then checks
  answers against. Definitions can be imported from dbt or Cube. See `docs/METRICS.md`.

### Structural fields in `schema-catalog.json`

`npm run generate-catalog` writes these from the database. They can also be written by hand,
and since 1.24.0 a regeneration keeps what you wrote where the database declares nothing.

- **table.primaryKey**: a column name, or a list of names. The fan-out check and the second
  derivation of the Answer Certificate count the records of a table by its key, and they
  use a single-column key. On a table that has none in the catalog (a view, a table with no
  declared key) they have nothing to count by: write the key by hand there.
  `npm run check-context` fails when the key is not a column of the table.
- **catalog.relationships** (root level): a list of `{ "from": "orders.user_id", "to":
  "users.id" }`, each end written `table.column`. Generated from foreign keys. Add by hand
  the ones the database does not declare: the model joins by them, and `query_metric` only
  joins along them.
- **table.approxRowCount**: shown to the model next to the table name.
- **table.domain**, **table.tags**: free labels, returned by the `list_tables` tool.
- **catalog.version** (root level): any string or number. It is recorded in each
  certificate as `catalogVersion`.
- **catalog.alwaysIncludeTables** (root level): a list of table names. It matters only for a
  catalog larger than `AI_SCHEMA_MAX_TABLES`, where each prompt carries only the tables
  selected for the question: these are always among them.

### Data notes: `column.profile` and `column.profileNote`

A model reads "customers with a known phone" and writes `phone IS NOT NULL`, while the table
holds `-` or an empty string for "not recorded". `npm run profile-columns` reads how the
data of each unmasked column records a missing or special value and writes it on the column:

```bash
npm run profile-columns                     # show the notes, change nothing
npm run profile-columns -- --write          # write them into schema-catalog.json
npm run profile-columns -- --table Customer --table Invoice     # only these tables
```

- **column.profileNote**: one line the model sees in braces after the column name, while
  `AI_CATALOG_COLUMNS` and `AI_CATALOG_PROFILE` are on (both are by default).
- **column.profile**: the counts the note was written from.

It reads through the read-only connection and never reads a masked column. Read the notes
before `--write`: they say what the data shows, and "may mean" where they guess. A note you
edit by hand is kept on the next run as long as you remove the `profile` counts next to it.
Reload or restart the server after writing. The notes are derived from your data and are
sent to the model provider with every prompt (`docs/DATA_PRIVACY.md`).

### Masked columns: three places, one rule of precedence

`masked-columns.json` is the usual place: `{ "users": ["email"] }`. The catalog accepts the
same thing in two other places, a `maskedColumns` list on a table and a root-level
`maskedColumns` object shaped like the file. For each table the engine uses the **first**
of these that has entries, and only that one:

1. the table's own `maskedColumns` in `schema-catalog.json`;
2. the table's entry in `masked-columns.json`;
3. the table's entry in the catalog's root `maskedColumns`.

They are not merged. A table that has its own `maskedColumns` ignores what
`masked-columns.json` says about it, so a column listed only in the file is then readable.
Keep each table's masks in one place. Table names are matched without regard to case.

### `row-policies.json` (optional)
Per-user or per-tenant row isolation. Each key is a catalog table, each value a predicate over
that table with `:claims.<name>` for values of the caller's verified identity:

```json
{ "orders": "tenant_id = :claims.tenant_id" }
```

The engine rewrites every reference to the table into an already filtered derived table before
the statement runs, and refuses the question when the identity lacks the claim. Full contract
in `docs/ROW_POLICIES.md`.

### `clarification-rules.json` (optional)
Deterministic, context-driven clarification. Each rule matches the user's question by regex
and, when it fires, Rosie returns clarification chips BEFORE calling the LLM. That
guarantees the question is asked instead of relying on the model to ask it. Rules can only ASK; they never run SQL.

```json
[
    {
        "id": "bare-orders",
        "pattern": "^\\s*(how many|count)\\s+orders?\\s*\\??\\s*$",
        "skipIf": "\\b(paid|open|per |in |status|customer)\\b",
        "question": "Do you mean all orders or only paid ones?",
        "options": ["All orders", "Paid orders only"]
    }
]
```
- **pattern** (required): case-insensitive regex; if it matches, the rule may fire. Keep it
  NARROW (anchor with `^...$`) so only the genuinely ambiguous phrasing triggers.
- **skipIf** (optional): if this matches, the rule is skipped. Use it for disambiguating
  qualifiers (a status, date, entity, "per X" grouping) that make the question unambiguous.
- **question** + **options** (2 to 4): the clarification shown as chips. A question equal to one
  of the options never re-fires (so picking a chip proceeds to a real answer).

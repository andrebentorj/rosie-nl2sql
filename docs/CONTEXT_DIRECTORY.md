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
serves more than one engine. The engine adds its own short `## SQL dialect` note for the
engine in use; on all three it carries the sentence "Write each table alias the same way
every time, in one letter case; aliases that differ only by case are rejected." (MySQL has
this note since 1.31.0.) Do not repeat or contradict it in your prompt.

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
`border`, `borderSoft`. A key that is set overrides the widget's automatic dark-mode
value for that key (explicit beats auto).

How the theme is applied (widget 2026.11.11):

- **A theme that names `surface` decides light or dark for the whole widget.** The keys it
  does not name (inks, borders, the colours of the verdicts) follow that surface and not the
  operating system: a white `surface` on a dark system renders the widget light throughout.
  Without `surface`, the widget follows the system.
- **The header runs from `primary` to `primaryDeep`**, and the results bar and the send
  button use `primaryDeep`. `primarySoft` is still accepted and no longer colours the header.
- **Text on those colours is white, unless white would not be readable.** When `primary` or
  `primaryDeep` leaves white text under 4.5:1, the widget writes dark text on them and says
  so in the browser console.
- The verdict colours (passed, caution, contradicted, not checked, approved) are not
  themable: they mean the same thing in every project.

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
Richer fields improve both the summary and the `describe_table` tool output. Since 1.30.0
`describe_table` sends a fixed list of fields and nothing else: for a column `name`, `type`,
`nullable`, `description`, `synonyms`, `enumValues`, `measure`, `additive`, `profileNote`
and `profileNoteConfirmed`; for the table `name`, `description`, `primaryKey`, `columns`,
`relationships` and `entity`; for a relationship `from`, `to`, `type` and `inferred`. A
field you add to the catalog that is not in these lists does not reach the model.

- **table.entity** `{ "name": "card", "key": "name" }`: says that the thing your users ask
  about is not one row of this table. A card printed in five sets is five rows and one
  card; a customer registered twice is two rows and one customer. `key` is ONE column of the
  table (not masked, not the primary key) whose value identifies the entity. The prompt then
  tells the model what one card is, and the certificate counts, for a total, an average or
  a count over this table, the rows against the distinct values of `key`: more rows than
  entities and the answer is flagged with both counts and, where it can be computed, the
  figure taking each entity once (`entity_grain`, `docs/CERTIFICATE.md`). Declare it only
  where questions are usually about the entity: a question about each row ("how many
  printings") is then flagged too, as the other of two readings.
- **table.promptHidden** `true`: keeps a rarely asked table (audit shadows, technical logs)
  out of the detailed prompt block to save tokens on every request. The table is only named
  on an `OTHER TABLES` line; it stays fully queryable and visible to `list_tables` and
  `describe_table`. This is prompt shaping only and is NOT a security control: to forbid a
  table, remove it from the catalog.
- **table.synonyms** `["customers", "clients"]`: alternative names users say.
- **column.description**: what the column means.
- **column.synonyms**: alternative names for the column.
- **column.additive** `false`: the column is not a quantity that sums across rows (a balance,
  a rate, a percentage, a unit price). The prompt tells the model not to `SUM` it, and an
  answer that does is flagged (`non_additive`). Leave it unset for ordinary amounts.
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
  `npm run check-context` fails when the key is not a column of the table. On SQL Server,
  `COUNT(*)` over a table with masked columns is rewritten to count its key only when the
  key column has `"nullable": false` and is not masked (`npm run generate-catalog` writes
  `nullable` for you; a hand-written key needs it, `docs/SQLSERVER.md`).
- **catalog.relationships** (root level): a list of `{ "from": "orders.user_id", "to":
  "users.id" }`, each end written `table.column`. Generated from foreign keys. Add by hand
  the ones the database does not declare: the model joins by them, and `query_metric` only
  joins along them.
- **table.reviewedView** `true` (since 1.31.0): this table is a view (or a materialized
  view, or since 1.32.0 a PostgreSQL foreign table) in the database, and someone reviewed it and found that it exposes no masked value.
  A column grant on the base tables does not reach through a view, so until a catalog view
  carries this field the masked-column grant check reports the context as "masked by Rosie
  only" (`view_not_reviewed`). Only the boolean `true` counts. It is your statement, not a
  check: Rosie does not read what the view selects. `npm run generate-catalog` keeps it.
- **catalog.reviewedRoutines** (root level, since 1.31.0): `[ "schema.name", ... ]` (on MySQL
  `"database.name"`), the definer routines the read-only account may execute that someone
  reviewed and found return no masked value: PostgreSQL `SECURITY DEFINER` functions and
  procedures, MySQL `SQL SECURITY DEFINER` routines. Until a routine is listed, the grant
  check reports it `object_not_verifiable` and the context is "masked by Rosie only".
  PostgreSQL entries are read as PostgreSQL resolves a name (unquoted parts fold to lower
  case, `"Quoted"` parts keep their case); MySQL entries compare without case and may use
  backquotes. A listing is by name and covers every overload of that exact name, present and
  future. It is your statement, not a check (a listed routine whose body names a masked
  column is still refused; one that uses `*`, a whole-row reference or dynamic SQL is
  accepted with a warning from `npm run grants -- --verify`). Since 1.32.0 entries compare as
  (schema, name) pairs (`"a.b".c` and `a."b.c"` differ), and the list also covers SQL Server
  procedures and functions (exact names, `[dbo].[name]` accepted).
  `npm run generate-catalog` keeps it; `npm run check-context` validates its shape.
- **catalog.reviewedViews** (root level, since 1.32.0): `[ "schema.name", ... ]`, the tables,
  views, materialized views, foreign tables and (SQL Server) synonyms outside the catalog that
  the read-only account can read and that someone reviewed and found return no masked value.
  The grant check is an allowlist: every such object, whatever it reads, is
  `object_not_verifiable` until it is listed or the account's access to it is taken away.
  Names are read like `reviewedRoutines` (SQL Server: exact, brackets accepted). A synonym for
  a masked catalog table is reported as reading it even when listed. A listed object whose dependencies still show a masked read is reported
  with `reviewed: true` and accepted. Your statement, not a check; `npm run check-context`
  validates it ("Reviewed views").
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
- **column.profile**: the counts the note was written from. It also holds
  `generatedNote`, the note the script last generated, and `hypotheses`, the kinds of guess
  in it (`zero`, `placeholder`). The model never sees `profile`.
- **column.profileNoteConfirmed**: set it to `true` when you, who know the data, stand behind
  the column's note. The script never writes this field and never changes the note of a
  column that has it. The model then sees the note as `{confirmed: ...}` and is told to
  follow it as written.

**A generated note is an observation, not a rule.** The data shows counts and shapes: "0 in
23% of rows, the smallest value above 0 is 14". It cannot show what the 0 means: a weight of
0 is not recorded, a discount of 0 is a real zero. Where a note guesses, it says "may mean"
and says that it depends on the question. Every run ends with the list of those guesses, by
`table.column`. Go through it once with someone who knows the data, and for each one:

- the guess is right: rewrite the note as a fact ("0 means not recorded, leave those rows
  out of averages") and set `"profileNoteConfirmed": true`;
- the guess is wrong: rewrite it ("0 is a real zero: no discount") and confirm it, or delete
  `profileNote` when there is nothing to say;
- you do not know: leave it. The model reads it as a hypothesis.

**A rerun is safe.** The script replaces a note only while it still equals
`profile.generatedNote`. A note you edited, deleted, wrote yourself or confirmed is kept, the
counts beside it are refreshed, and the run reports how many notes it kept and what the data
now reads for each. You do not need to remove `profile` to protect an edit. The catalog is
written to a temporary file and renamed over the old one after checking that it parses and
still has every table and column, so a failed run leaves the catalog as it was.

It reads through the read-only connection and leaves masked columns out of the statements
it runs. Reload or restart
the server after writing. The notes are derived from your data, they can carry values of it
(a placeholder, a code), and they are sent to the model provider with every prompt
(`docs/DATA_PRIVACY.md`).

### Masked columns: three places, added together

`masked-columns.json` is the usual place: `{ "users": ["email"] }`. The catalog accepts the
same thing in two other places, a `maskedColumns` list on a table and a root-level
`maskedColumns` object shaped like the file. Since 1.26.0 the three are added together: a
column listed in any of them is masked, and no place can unmask what another one lists.
Table and column names are matched without regard to case. Since 1.30.0 that also holds
inside one place: `"Users": ["email"]` and `"users": ["password"]` in one file mask both
columns (1.29.2 and earlier applied only the first key). Keep one key per table all the same:
`npm run check-context` fails on two keys that differ only by case. The server, the setup
wizard, the vault and `check-context` read the three places through one module, so they
cannot disagree about what is masked.

A mask is Rosie's own refusal. For the database to refuse the column too, use strict mode
(`ROSIE_MASKING_MODE=strict`), in which the read-only user has no privilege on it
(`docs/INSTALL.md`, Step 2; `npm run grants` prints the statements). In express mode, the
default, the account can read it and the validator is the layer. Rosie reports at startup
and every 15 minutes whether it can still read each masked column. A mask entry for a table or column the catalog does not list keeps the
context out of "masked by the database". On SQL Server column grants do not stop `PIVOT`;
see `docs/SQLSERVER.md` for the design that does.

A masked column is left out of everything Rosie describes to the model: the catalog summary,
`describe_table`, relationships that end on it, metrics whose formula, filters, dimensions or
text read it, and verified examples whose SQL reads it or whose question or note writes it as
`table.column`. Text you write yourself (the system prompt, descriptions, clarification
rules) is sent as written. `check-context` fails where that text writes a masked
`table.column` and warns where it contains the column's words; it cannot see a paraphrase.
Do not tell the model which columns are masked.

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

# Metrics and the semantic layer

A business has definitions: revenue counts paid orders only, an active customer bought
something in the period. A model that writes SQL from a schema does not know them, and will
compute "revenue" in whatever way the tables suggest. Rosie lets you declare the definitions
once, puts them in front of the model, and then checks every answer against them.

## Declaring a metric

In `schema-catalog.json`, a metric is either free text (guidance for the model, nothing more)
or a structured definition:

```json
"metrics": {
    "gross_revenue": {
        "description": "Money from paid orders.",
        "expression": "SUM(orders.total)",
        "table": "orders",
        "filters": [ "orders.status = 'paid'" ],
        "dimensions": [ "orders.created_at", "orders.country" ],
        "synonyms": [ "revenue", "turnover" ]
    },
    "legacy_note": "free text still works and is shown to the model as is"
}
```

| Field | Meaning |
|---|---|
| `expression` | Required. The SQL that computes the metric. Must contain at least one aggregate. |
| `table` | The table the metric is computed on. |
| `filters` | Conditions that must always apply. Each is one predicate. |
| `dimensions` | Columns it makes sense to break the metric down by. Shown to the model; not enforced. |
| `synonyms` | Other names people use. Matched as whole words, like the name itself. |
| `description` | Shown to the model after the definition. |

A structured metric whose expression, table or filters cannot be parsed stops the server at
startup: a definition the engine cannot check is worse than none, because people would
believe it is being enforced.

### A ratio of two metrics

An average ticket, a share, units per order: one declared metric divided by another.

```json
"metrics": {
    "revenue":        { "expression": "SUM(orders.total)", "table": "orders" },
    "order_count":    { "expression": "COUNT(orders.id)", "table": "orders" },
    "paid_orders":    { "expression": "COUNT(orders.id)", "table": "orders", "filters": [ "orders.status = 'paid'" ] },
    "units":          { "expression": "SUM(order_items.qty)", "table": "order_items" },

    "average_ticket": { "ratio": { "numerator": "revenue", "denominator": "order_count" }, "synonyms": [ "ticket" ] },
    "paid_share":     { "ratio": { "numerator": "paid_orders", "denominator": "order_count" } },
    "units_per_order": { "ratio": { "numerator": "units", "denominator": "order_count" } }
}
```

A ratio has no `expression`, `table` or `filters` of its own: they come from the two metrics
it names, each of which must be a plain metric with a `table`. It may carry `description`,
`synonyms` and `dimensions`. A ratio of a ratio is refused at startup, as is a side that is
not a declared metric.

How it is computed, by `query_metric`:

- **Same table and same required filters** (`average_ticket`): one statement that divides the
  two aggregates over the same rows.
- **Different filters or different tables** (`paid_share`, `units_per_order`): each side is
  computed on its own, with its own table, joins and required filters, and the two results
  are joined on the breakdown columns. Each aggregate runs over its own rows only, so a
  ratio cannot be inflated by the other side's join. The denominator drives the join: where
  it has no rows the ratio has no value.
- The division never truncates to an integer, and a denominator of zero gives no value
  instead of an error.
- A breakdown or a filter must be reachable, many-to-one, from **both** sides. Otherwise the
  tool refuses and says which side could not reach it.
- A filter asked for in the question applies to both sides.

One limit. When the two sides are computed separately and broken down by a column that
holds nulls, the rows where that column is null show no ratio.

The certificate checks each side on its own: a side that needed a join is probed for
repeated rows like any other statement, so a compiled ratio normally comes back with
`CHECKS_PASSED` and not merely with the promise that it was built safely.

## What the engine does with it

1. **Prompt.** Each metric is rendered into the catalog block of the system prompt with its
   expression, table, required filters and synonyms.
2. **Answer Certificate.** When a question names a metric, by name or synonym, the final
   query is compared with the definition and the certificate gains a `metric` assertion:
   - **pass**: the query computes the declared aggregate(s), reads the declared table and
     applies every required filter;
   - **warn**: it does not, and the detail says which part is missing. The answer is then
     flagged, and the mismatch is handed back to the model once for a corrected query
     (`docs/CERTIFICATE.md`, The second attempt).
3. **Compilation.** A metric that declares a `table` can also be computed by the engine
   itself: see the next section.

The comparison is structural, not textual. Aliases, table qualifiers, identifier quoting,
letter case and the order of operands in `a * b` or `a + b` do not matter. A required filter
must be ANDed with the rest of the WHERE clause; the same condition under an `OR` does not
count. The definition may sit inside a CTE or be wrapped in `ROUND(...)`.

Two details that keep the check honest:

- **Shared words.** Several metrics may answer to the same word, such as revenue by invoice
  and revenue by line item. The answer passes when it follows any of the metrics the question
  names.
- **Specific beats general.** If a question says "US invoiced amount" and both
  `us_invoiced_amount` and `invoiced_amount` exist, only the longer name counts. The general
  metric cannot vouch for a query that dropped the filter of the specific one.

The check proves the query follows the declared definition. It does not prove the definition
is right, and it says nothing when a question names no metric.

## Compiled metrics: the engine writes the query

Checking a query after the model wrote it catches a wrong formula. Not letting the model
write the formula is stronger, and it is what semantic layers do. For a metric that declares
its `table`, the model has a tool, `query_metric`, where it says only **which** metric, by
**which** dimensions and under **which** filters:

```json
{ "metric": "line_revenue",
  "group_by": [ "Genre.Name", "Invoice.InvoiceDate:year" ],
  "filters": [ { "column": "Invoice.BillingCountry", "op": "in", "value": [ "USA", "Canada" ] } ],
  "order": "metric_desc", "limit": 10 }
```

The engine builds the statement:

- **The formula** is the declared `expression`, and the declared `filters` are always applied.
- **Joins** are added by the engine, and only along many-to-one relationships leading away
  from the metric's table (a foreign key pointing at a primary key, from the catalog's
  `relationships`). Each row of the metric's table therefore meets at most one row of every
  joined table, so a compiled metric cannot be inflated by a join. `LEFT JOIN` is used, so a
  row with no match still counts.
- **A dimension that would repeat rows is refused.** Asking for `revenue`
  (`SUM(Invoice.Total)`) by `Genre.Name` needs the invoice lines, which repeat each invoice;
  the tool answers that the dimension is not reachable and the model picks `line_revenue`
  instead, or writes the query. Two equally short paths to a table are refused as ambiguous.
- **Dimensions** are `Table.Column`; a date column may carry a grain (`:year`, `:quarter`,
  `:month`). If the metric declares `dimensions`, only those are accepted.
- **Filters** take `=`, `!=`, `>`, `>=`, `<`, `<=`, `in` and `between`, with numbers or text
  bound as literals. Masked columns cannot be used.

The compiled statement then runs like any other: validator, row cap, row policies, read-only
pool, Answer Certificate. The answer carries `metric` (name, formula, base table, dimensions,
filters, joins) and the widget shows "Declared metric: revenue = SUM(Invoice.Total)".

What it does not do: metrics over a join declared inside the metric, self-referencing
tables, ratios of ratios, and custom time windows ("trailing 12 months"). For those
the model writes the query and the checks above apply.

**When the tool is offered.** Only when the question names a declared metric, by its name or
a synonym, and only for the metrics it names. This is deliberate and measured: offered on
every question, the default model used it for questions that ask for no metric and accuracy
on our suite fell from 190 to 183 of 194; gated this way it rose to 191, with all 28
compiled answers correct (`docs/BENCHMARK.md`). The consequence is that synonyms matter:
a metric people call "sales" or "turnover" should list those words.

## Importing from dbt or Cube

If the definitions already live in a semantic layer, import them instead of retyping them:

```bash
npm run import-semantic -- --dbt path/to/dbt/models              # dry run, prints the result
npm run import-semantic -- --dbt path/to/dbt/models --write      # updates the catalog
npm run import-semantic -- --cube path/to/cube/model --write
```

| Option | Meaning |
|---|---|
| `--dbt <path>` | File or directory of dbt Semantic Layer YAML (`semantic_models`, `metrics`). |
| `--cube <path>` | File or directory of Cube YAML (`cubes`). |
| `--catalog <file>` | Catalog to update. Default: `$ROSIE_CONTEXT_DIR/schema-catalog.json`. |
| `--table-map a=b,...` | Semantic model or cube name to catalog table, when the names differ. |
| `--overwrite` | Replace metrics already in the catalog. Default: keep them. |
| `--write` | Write the catalog. Default: dry run. |

What is translated:

| Source | Becomes |
|---|---|
| dbt `simple` metric over a `sum`, `count`, `count_distinct`, `average`, `min`, `max` or `sum_boolean` measure | `expression` and `table` |
| dbt `ratio` metric over two measures of one table, with no filter of their own | One expression, `numerator / denominator` |
| Any other dbt `ratio` metric (sides in different tables, a side that names a metric, a filter on one side) | A ratio of two declared metrics. A side that is a bare measure or carries a filter becomes a metric of its own, named `<ratio>_numerator` or `<ratio>_denominator`, so its filter stays on its side |
| dbt filter written with `{{ Dimension('entity__name') }}` | A required filter on the real column |
| dbt `label`, Cube `title` | A synonym |
| dbt entities: a `foreign` entity in one model and the `primary` entity of the same name in another | A many-to-one relationship in the catalog, and the primary key of a table that declares none |
| dbt and Cube dimensions | The `dimensions` of each metric: those of its own model or cube first, then those of every model or cube it reaches through many-to-one joins |
| Cube `sum`, `count`, `count_distinct`, `avg`, `min`, `max` measure with its `filters` | `expression`, `table` and required filters |
| Cube `number` measure written as one measure of the cube divided by another (`{a} / {b}`) | A ratio of the two |
| Cube `joins` with `many_to_one` or `one_to_many` (`belongs_to`, `has_many`) and a single-column equality | A many-to-one relationship in the catalog, from the side that holds the key |

Relationships and primary keys are only ever added: a relationship the catalog already has is
left alone, and a table that declares a primary key keeps it. The dry run lists each one as
`+ join:` or `+ primary key:` before anything is written. Time granularity needs nothing
imported: any date column can be broken down by `:year`, `:quarter` or `:month`.

What is skipped, each with a message: dbt `derived`, `cumulative` and `conversion` metrics;
`median` and percentile measures; filters that use `TimeDimension`, `Entity` or `Metric`
templates; Cube `number` measures in any other form, and measures that reference members of
another cube; Cube joins that are many-to-many or not a single-column equality; cubes defined
by a SQL query instead of a table; anything on a table that is not in the catalog. A metric whose
filter cannot be translated is skipped entirely. Importing it without the filter would turn
a restricted definition into a broader one and then enforce the wrong thing.

Every imported metric is compiled the way the engine compiles it at startup, and dropped
with a message if it does not compile.

`demo/dbt/models/chinook_semantic.yml` and `demo/cube/model/chinook.yml` are small working
examples over the demo database:

```bash
npm run import-semantic -- --dbt demo/dbt/models --catalog contexts/demo/schema-catalog.json
```

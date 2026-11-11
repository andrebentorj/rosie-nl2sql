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

## What the engine does with it

1. **Prompt.** Each metric is rendered into the catalog block of the system prompt with its
   expression, table, required filters and synonyms.
2. **Answer Certificate.** When a question names a metric, by name or synonym, the final
   query is compared with the definition and the certificate gains a `metric` assertion:
   - **pass**: the query computes the declared aggregate(s), reads the declared table and
     applies every required filter;
   - **warn**: it does not, and the detail says which part is missing. The answer is then
     never reported as verified.

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
| dbt `ratio` metric | `numerator / denominator` |
| dbt filter written with `{{ Dimension('entity__name') }}` | A required filter on the real column |
| dbt `label`, Cube `title` | A synonym |
| Cube `sum`, `count`, `count_distinct`, `avg`, `min`, `max` measure with its `filters` | `expression`, `table` and required filters |

What is skipped, each with a message: dbt `derived`, `cumulative` and `conversion` metrics;
`median` and percentile measures; filters that use `TimeDimension`, `Entity` or `Metric`
templates; Cube measures of type `number` or that reference other members; cubes defined by a
SQL query instead of a table; anything on a table that is not in the catalog. A metric whose
filter cannot be translated is skipped entirely. Importing it without the filter would turn
a restricted definition into a broader one and then enforce the wrong thing.

Every imported metric is compiled the way the engine compiles it at startup, and dropped
with a message if it does not compile.

`demo/dbt/models/chinook_semantic.yml` and `demo/cube/model/chinook.yml` are small working
examples over the demo database:

```bash
npm run import-semantic -- --dbt demo/dbt/models --catalog contexts/demo/schema-catalog.json
```

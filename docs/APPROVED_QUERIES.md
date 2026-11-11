# Approved queries

The Answer Certificate checks the structure of a calculation. It cannot tell whether a
question was understood (`docs/CERTIFICATE.md`). An approved query covers the other half: it
is a statement a person reviewed for a question, which Rosie runs unchanged when that
question is asked again, and says so on the answer.

It is the same idea as a verified query repository or a trusted asset in the large analytics
platforms, on the database you already have.

## What it looks like

An entry of `verified-queries.json` in the context directory becomes approved when it names
who approved it:

```json
{
    "id": "revenue-by-country-year",
    "question": "Revenue for a country in a year",
    "sql": "SELECT SUM(Total) AS revenue FROM Invoice WHERE BillingCountry = :country AND YEAR(InvoiceDate) = :year",
    "params": {
        "country": { "type": "string", "description": "Billing country, as stored: USA, Brazil, ..." },
        "year": { "type": "integer" }
    },
    "approvedBy": "Dana Reyes, Finance",
    "approvedOn": "2026-11-11",
    "reviewBy": "2027-05-11"
}
```

| Field | Required | Meaning |
|---|---|---|
| `id` | yes | Lower-case letters, digits, `-` or `_`. Unique in the file |
| `question`, `sql` | yes | As for any verified query |
| `approvedBy` | yes | Who reviewed it. Shown on the answer |
| `approvedOn` | no | Date of the review, `YYYY-MM-DD`. Shown on the answer |
| `reviewBy` | no | After this date the entry stops being approved and goes back to being an example |
| `params` | no | The values a caller may supply, each with a `type`: `string`, `integer`, `number` or `date` |

Write parameters as `:name` in the SQL, outside quotes. Every `:name` must be declared and
every declared parameter must be used, or the entry is not approved.

## How it is used

The entries most similar to a question are shown to the model, as verified queries always
were. Approved ones are listed with their id and parameters, and the model has one more
tool, `run_approved_query(id, params)`. When the question is the question of an approved
query, with at most different values (another country, another year), the model calls the
tool instead of writing SQL.

The model chooses **which** approved query and **which values**. It cannot change the
statement. Values are checked against their declared type and written as literals:

| Type | Accepted |
|---|---|
| `integer` | Up to 18 digits, optional minus sign |
| `number` | The same with up to 10 decimals |
| `date` | `YYYY-MM-DD` |
| `string` | 1 to 200 characters, no control characters, no backslash; single quotes are doubled |

The resulting statement then goes through the same path as any other: the validator and its
allowlist, masked columns, the row cap, row policies, the read-only pool, and the Answer
Certificate. An approved query is not a way around any check; a row policy still restricts
its rows to the caller's.

## What the user sees

- **Widget:** "Approved query, reviewed by Dana Reyes, Finance on 2026-11-11", above the
  certificate lines.
- **HTTP API:** `approval` in the `/ask` response, with `id`, `question`, `approvedBy`,
  `approvedOn`, `reviewBy` and the `params` used. `null` when the answer was not an approved
  query.
- **MCP:** the same object in `structuredContent`, and a sentence in the text block.
- **Audit:** the SQL that ran is recorded as for any answer.

## When an entry stops being approved

At startup each entry is checked, and one that fails is used as a plain example and logged
(`event: approved_query_demoted`), never run as approved:

- no valid `id`, or an id used twice;
- a parameter without a valid type, used and not declared, or declared and not used;
- `reviewBy` in the past;
- the statement no longer validates against the catalog (a dropped column, a renamed table):
  the entry is removed, as for any verified query.

## What it does not do

- It does not match questions by itself: the model decides whether a question is the one an
  approved query answers. A wrong choice is possible, and the answer shows which approved
  question was used so a reader can tell.
- There is no review screen yet: the queue below is a command, entries are edited in the
  file, and the context is loaded at startup.

## From a wrong answer to an approved query

```bash
npm run review                     # answers rated down, or CONTRADICTED, FLAGGED or NOT_CHECKED, last 7 days
npm run review -- --days 30
npm run review -- --approve <messageId> --id revenue-by-country --by "Dana Reyes, Finance" --review-by 2027-06-30
```

The queue lists each answer with its question, its SQL and why it is there. The usual cycle:
find the wrong answer, ask the question again in a way that gets the right statement (or
write it), check the result, then approve **that** answer by its message id. `--approve`
validates the statement against the catalog and appends it to `verified-queries.json` with
your name and today's date. Add `params` by hand where values should vary, and restart the
server.

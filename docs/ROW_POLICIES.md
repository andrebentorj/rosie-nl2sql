# Row policies

Row policies make Rosie answer each user from that user's rows only: one tenant, one region,
one sales team. The filter is enforced by the engine on every statement. It does not depend
on the model, on the prompt or on the question.

Policies read the caller's **verified** identity, which comes from a signed token: see
[IDENTITY.md](IDENTITY.md). With only the static `ROSIE_API_TOKEN` there are no claims, so a
context that declares policies refuses every question about a restricted table. That is the
intended behaviour: isolation without a verified identity would be decoration.

## Declare the policies

Add `row-policies.json` to the context directory. Each key is a table of the catalog; each
value is a predicate over that table's own columns, with `:claims.<name>` where a value of
the caller's identity goes.

```json
{
    "orders":    "tenant_id = :claims.tenant_id",
    "customers": "tenant_id = :claims.tenant_id",
    "tickets":   "tenant_id = :claims.tenant_id AND region IN (:claims.regions)"
}
```

Keys that start with `_` or `$` (`_comment`, `$schema`) are ignored. A predicate may combine
comparisons with `AND`, `OR`, `IN`, `IS NULL` and literals. It may
not call functions, contain a subquery or name a column of another table. A policy that
breaks these rules, names a table or column that is not in the catalog, or uses no claim at
all stops the server at startup.

## How it is enforced

Immediately before a statement runs, every reference to a restricted table is replaced by a
derived table that is already filtered:

```sql
-- what the model wrote (and what the user, the audit log and the answer handle see)
SELECT c.name, SUM(o.total) FROM orders o JOIN customers c ON c.id = o.customer_id GROUP BY c.name

-- what the database receives for a caller whose tenant_id is acme
SELECT c.name, SUM(o.total)
FROM (SELECT * FROM orders WHERE tenant_id = 'acme') AS o
JOIN (SELECT * FROM customers WHERE tenant_id = 'acme') AS c ON c.id = o.customer_id
GROUP BY c.name
```

This happens at every depth: joins, self-joins, subqueries in any clause, derived tables and
CTE bodies. There is no syntax inside a SELECT that reaches the table behind a derived table,
so a query that asks for another tenant (`WHERE tenant_id = 'globex'`, `OR 1 = 1`) simply
runs against the caller's rows and finds nothing else.

The same path serves everything that reads data: `execute_query`, `sample_rows`,
`lookup_values`, `validate_query`, CSV and MCP exports, and the probes of the Answer
Certificate. A certificate therefore verifies a number against the caller's rows, never
against the whole table.

## Fail closed

Nothing runs, and the model is told not to retry, when:

- the request has no verified identity, or the identity lacks a claim the policy needs;
- a claim is empty, or a list claim is an empty list;
- a claim value falls outside a narrow alphabet (letters, digits, space and `_ . @ : + -`, at
  most 128 characters, starting with a letter or a digit, no `--`). Tenant ids, UUIDs, slugs
  and e-mail addresses pass; quotes, backslashes, semicolons and control characters do not;
- a CTE takes the name of a restricted table;
- the statement cannot be parsed for rewriting.

Numbers are written as numbers and booleans as booleans. A list is accepted only inside
`IN (...)`, with at most 200 elements.

## What it is not

- **Not a replacement for database security.** On PostgreSQL, native row-level security is a
  good second layer under this one. On any engine, the read-only user should not have access
  to tables that no user of Rosie may read.
- **Not column security.** Use `masked-columns.json` for that.
- **Not applied to the catalog.** Table and column names, descriptions and enum codes are
  the same for every caller.

## Proof

`npm run test:security` includes attempts to read another tenant's rows, run on MySQL and
PostgreSQL, plus the identity-value and startup checks (section 23 of the battery), and a
separate group that checks the rewritten statement on SQL Server.
`npm run test:postgres` and `npm run test:mysql` run two tenants against a real database and
check that the same question returns different numbers and a certificate derived from each
tenant's rows. The PostgreSQL suite also drives the HTTP API with signed tokens: the claim in
the token decides which rows an export returns.

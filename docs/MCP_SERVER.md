# Running Rosie as an MCP server

> Status: local stdio transport is complete (typed output, resources, prompt, quotas,
> cancellation). The remote HTTP transport verifies JWTs from your identity provider; its
> static-token mode remains a preview for local trials.

Rosie speaks the Model Context Protocol, so any MCP client (Claude Desktop, Claude Code,
Cursor, an internal agent) can ask questions about a project's database through the same
governed pipeline the widget uses. This is a **second front door onto the same engine**, not
a second engine: every request still passes the prompt-injection filter, the intent guards,
the AST validator, the schema allowlist, the PII masking, the read-only pool and the audit
trail.

## What is exposed, and what is not

**Profile A (this phase):**

| Tool | What it does |
|---|---|
| `ask_data` | A question in plain language. Runs the whole Rosie pipeline internally and returns the narrative answer, the result rows, the exact SQL that ran, the Answer Certificate and a signed handle for follow-ups. |
| `explain_answer` | Describes, in words, what the query behind a previous answer did. Takes that answer's handle. |
| `export_answer` | Re-runs a previous answer's query past the display cap to fetch more rows. Takes the handle, re-validates the SQL, and caps the result (default 1000, max 5000). |
| `list_available_data` | Catalog summary: tables, what they are about, relationships, declared metrics. Descriptions and synonyms only, never raw DDL. |

`explain_answer` and `export_answer` authorize with the **signed handle**, never with a
message id: the handle carries the validated SQL, its owner and a 15-minute expiry, and is
verified cryptographically. A handle belonging to another user is refused even though it is
perfectly well signed. Every rejection reads the same to the caller regardless of cause
(expired, wrong owner, bad signature), because a specific reason is an oracle for probing
handles you do not own; the real reason is logged server-side.

The export cap is deliberately far below the HTTP export's 50k: these rows land in a
language model's context, where a huge result is both expensive and harmful to reason over.
Unbounded retrieval belongs to the Tasks extension.

**Deliberately not exposed:** `execute_query`, `validate_query`, `sample_rows`,
`lookup_values`, `describe_table`, `get_relationships`, `list_tables`.

The reason is the point of the whole design. An MCP client hands the tool loop to *someone
else's* model. Rosie's own orchestrator is what supplies the system prompt, filters the
question, blocks write and sensitive intent, injects verified examples, fires the
clarification rules and runs the intent post-check. Publishing `execute_query` would give an
external agent a SQL channel with none of that, leaving only the validator and the read-only
user: five of seven defenses gone, in a setting where indirect prompt injection is *more*
likely, because the calling agent may have read an untrusted document earlier in its own
context. One high-level tool that runs the full pipeline is strictly more defensible than
what the browser widget does today, because the external model never sees SQL it can shape.

The low-level surface (Profile B) is designed but not built: it is meant to be granted per
person with a recorded justification, and there is no per-user identity to grant it against
until the OAuth work lands. Building it now would hand those tools to everyone holding the
shared token.

`masked-columns.json` is never exposed, as a tool or a resource. It is the PII map.

## What a client sees (protocol surface)

- **Typed results.** Every tool declares an `outputSchema`, so `tools/list` tells a client
  exactly what `structuredContent` carries. For `ask_data` that includes the **Answer
  Certificate** as a typed object (`verdict` enum `CONTRADICTED | FLAGGED | CROSS_CHECKED |
  NOT_CHECKED | CHECKS_PASSED | NOT_APPLICABLE`, `assertions[]` with `id`/`verdict`/`detail`,
  `skipped[]` with the checks that could not run; see `docs/CERTIFICATE.md`) and a `status` field:
  `answered`, `clarification` (put `options` to the user and ask again) or `blocked`
  (a governance rule refused the question; `reason` says which class, never the rule).
  SDK clients validate the result against the schema automatically.
- **Instructions.** The server publishes standing guidance that clients inject into the
  agent's context: ask in plain language, call `list_available_data` first when unsure,
  how to treat each verdict (CONTRADICTED is "this total is inflated, do not present it as a fact"; nothing is ever "verified"), no memory between
  calls, handles expire in 15 minutes.
- **Resources.** `rosie://catalog` (the catalog summary, `text/markdown`) and
  `rosie://project` (project card, `application/json`) can be pinned into context without
  a tool round-trip. Nothing else is a resource.
- **Prompt.** `checked_answer(question)` asks through `ask_data` and relays the figure with
  its verdict honestly; a one-click demonstration of what the certificate is for.
- **Argument validation.** Arguments are checked against each tool's `inputSchema` before
  any handler runs (the low-level SDK `Server` does not do this by itself); invalid
  arguments and unknown tools are protocol errors (`-32602`), not tool results.
- **Progress.** When the client sends a `progressToken`, `ask_data` streams progress
  notifications (planning, each executed query, verification) so a 40-second answer never
  looks like a hang and the client's request timeout keeps resetting.

## Connecting Claude Desktop

Add Rosie to the client's MCP configuration (on macOS,
`~/Library/Application Support/Claude/claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "rosie": {
      "command": "node",
      "args": ["/absolute/path/to/rosie/server/mcp/stdio-server.js"]
    }
  }
}
```

That is the whole configuration when the repository's `.env` is already set up: the entry
point loads it by absolute path, so no secret has to be copied into the client config. Use
absolute paths for both `node` and the script (`which node` if the client cannot find it),
and restart the client after editing.

To point one client at a DIFFERENT project than `.env` names, override just that variable —
anything already in the environment wins over the file:

```json
{
  "mcpServers": {
    "rosie-sales": {
      "command": "node",
      "args": ["/absolute/path/to/rosie/server/mcp/stdio-server.js"],
      "env": { "ROSIE_CONTEXT_DIR": "/absolute/path/to/sales/rosie-context" }
    }
  }
}
```

Locally you can also run `npm run mcp`, which is the same entry point; it speaks JSON-RPC on
stdin/stdout and is only useful when driven by a client.

## Operational notes

**stdout belongs to the protocol.** All logging goes to stderr. A single stray `console.log`
in the engine would corrupt the JSON-RPC stream, which is why the entry point redirects
stdout logging before requiring anything.

**The security hold applies here too.** The read-only user's privileges are probed at
startup, and the probe is awaited *before* the transport connects, so a client never gets a
window where every call is refused for a reason that is really "not checked yet". If the
probe fails, every tool call returns an explanatory error instead of an answer. The same is
true of the audit policy: under `ROSIE_AUDIT_POLICY=fail-closed`, tool calls are refused
while no audit sink can record them.

**The probe keeps running.** The privilege probe re-runs on `ROSIE_PERMCHECK_INTERVAL_MS`
(default 15 min) in this process too, because a desktop client keeps it alive for days and a
one-shot check would make a later `GRANT` invisible until restart.

**Quotas and concurrency.** `ask_data`, `explain_answer` and `export_answer` pass through the
same hourly/daily quotas and in-flight cap as the HTTP route (`AI_MAX_QUERIES_PER_HOUR`,
`AI_MAX_QUERIES_PER_DAY`, `AI_CONCURRENCY_LIMIT`), keyed by the caller identity. A refused
call is an `isError` result that says why in plain words. `list_available_data` and the
resources are free. State is per process; replicas share nothing yet.

**Cancellation.** A client that cancels (`notifications/cancelled`) or closes the pipe aborts
the provider loop through the same signal an HTTP socket close uses; nothing keeps spending
on an answer nobody will read.

**Identity.** Over stdio the trust boundary is the machine, so there is no authentication
step; `ROSIE_MCP_USER` only labels the audit rows and owns the signed handles. Handles are
also bound to the **project id**, so two stdio entries that share `ROSIE_HANDLE_SECRET` but
serve different projects refuse each other's handles. `explain_answer` and `export_answer`
write audit rows like `/ask` does (`status` `explain` / `export`, plus `_rejected` /
`_error` variants). Remote transport is a separate phase because it needs real OAuth
identity before anything is exposed beyond localhost.

**Clarifications.** When a question is genuinely ambiguous, `ask_data` returns
`status: "clarification"` with the question and its options (also rendered as text). The
protocol has a native pattern for this (MRTR, `resultType: "input_required"`) that the
published SDK does not implement yet; adopting it is on the roadmap.

## Remote transport

`npm run mcp:http` starts the same tool surface over **Streamable HTTP**, stateless (no MCP
session ids; signed handles carry all cross-call state) and behind **bearer authentication
with audience binding**: `requireBearerAuth({ verifier, requiredScopes, expectedResource })`
from the SDK, with `expectedResource` = `ROSIE_MCP_HTTP_PUBLIC_URL`, so a token minted for
another resource is refused. Identity for quotas, audit rows and handle
ownership is the verified token's `clientId`, never a tool argument.

There are two verifiers:

- **JWT (production).** With `ROSIE_JWT_ISSUER` configured (see `docs/IDENTITY.md`), tokens are
  verified against the provider's keys: signature, issuer, audience, expiry. The verified
  subject is the identity for quotas, audit and handle ownership, and the token's claims are
  what row policies filter by. `/.well-known/oauth-protected-resource` (RFC 9728) names the
  issuer when `ROSIE_MCP_HTTP_PUBLIC_URL` is set. `ROSIE_MCP_HTTP_REQUIRED_SCOPE` can require a
  scope the provider grants.
- **Static token (preview).** Without JWT configuration, `ROSIE_MCP_HTTP_TOKEN` stands in for
  an identity provider: one identity, no claims. Keep it to local trials.

In both modes the server binds `127.0.0.1` and belongs behind a TLS proxy. The JWT path is
covered by the identity unit tests and was exercised by hand against the running transport;
it does not yet have an automated end-to-end test of its own.

`server/mcp/server-factory.js` builds the configured `Server` for both transports, so
stdio and HTTP cannot drift apart in handlers, instructions or capabilities.

## Testing

```bash
npm run test:mcp            # protocol, tool surface, gating and handle ownership — free, no LLM
node test/mcp/run.js --live # adds ask → explain → export against the real DB and model
```

The suite pins the handle-signing secret on both sides and passes the environment through to
the spawned server. That detail is load-bearing: the SDK's stdio transport forwards only a
short allowlist of variables, so with different secrets every handle would be rejected for a
bad signature and the ownership assertions would pass without proving anything. Ownership is
therefore established in two steps — a handle minted for the server's identity must verify,
and only then is the same handle, minted for someone else, shown to be refused.

The suite drives the real server with the real SDK client over stdio, so a wire-format or
handshake regression fails there rather than in someone's editor. It also asserts the two
disclosure rules above: no low-level tool in `tools/list`, and no masked-column map or raw
DDL in `list_available_data` or `rosie://catalog`. Since 2026-11 it further checks the
`outputSchema`/`status` contract, the instructions, both resources, the prompt, argument
validation, a `blocked` outcome (write intent, refused before any model call), handles bound
to another project, the quota governor (a second server is spawned with an hourly quota
of 1 and must refuse the second governed call), and the Streamable HTTP preview (a third
process: 401 without or with a wrong token, full tool surface and a successful call with it).

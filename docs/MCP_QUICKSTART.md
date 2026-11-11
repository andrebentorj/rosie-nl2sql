# Connecting Rosie to Claude Desktop — step by step

> Practical walkthrough. The reference (what each tool does, what is deliberately not
> exposed, how authorization works) is in `docs/MCP_SERVER.md`.

Time needed: about five minutes. You need the repository working locally with a `.env` that
already runs `npm start` successfully. If `npm start` works, this will too.

---

## The short way

```bash
npm run doctor                                   # checks the installation
npm run mcp:config -- --install claude-desktop   # adds Rosie to Claude Desktop (backs up the file)
```

Quit Claude Desktop completely and reopen it. For Claude Code, Cursor or another client,
`npm run mcp:config` prints the entry and the command to paste, with the paths of this
checkout already filled in.

The steps below do the same by hand and explain each part.

## Step 1 — Confirm the server starts

From the repository root:

```bash
npm run test:mcp
```

Expect `ALL 16 MCP CASES PASS`. This drives the real server the same way Claude Desktop
will, so if it passes, the problem in any later step is configuration, not the server.

## Step 2 — Find your node binary

Claude Desktop does not always inherit your shell's `PATH`, which is the single most common
reason a server fails to start. Get the absolute path:

```bash
which node
```

Typical answers: `/usr/local/bin/node`, `/opt/homebrew/bin/node`, or something under
`~/.nvm/versions/node/…/bin/node` if you use nvm. Copy whatever it prints.

## Step 3 — Edit the Claude Desktop configuration

Open the file (create it if it does not exist):

- **macOS:** `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Windows:** `%APPDATA%\Claude\claude_desktop_config.json`

Paste this, replacing the `command` with what Step 2 printed and the path with your
repository location:

```json
{
  "mcpServers": {
    "rosie": {
      "command": "/usr/local/bin/node",
      "args": ["/absolute/path/to/rosie/server/mcp/stdio-server.js"]
    }
  }
}
```

That is the whole thing. No secrets go here: the server loads the repository's `.env` by
absolute path, so the OpenAI key, the database credentials and `ROSIE_CONTEXT_DIR` all come
from the file you already maintain.

If the file already had other servers in it, add only the `"rosie"` block inside the
existing `mcpServers` object rather than pasting a second one.

## Step 4 — Restart Claude Desktop completely

Quit the application from the menu (⌘Q on macOS), do not just close the window. It only
reads the configuration at startup.

## Step 5 — Check that Rosie is connected

Open the tools or connectors list in Claude Desktop. You should see **rosie** with four
tools: `ask_data`, `explain_answer`, `export_answer` and `list_available_data`.

---

## What to try first

Work up in this order; each one shows something different.

**1. Discovery, no cost and no database load:**

> What data does Rosie have available?

Exercises `list_available_data`. You should get the project description and its tables.

**2. A straightforward question:**

> How many orders are paid?

Exercises the full pipeline. The answer arrives with an Answer Certificate: the coverage
check reports what fraction of rows the filter kept.

**3. A question whose premise is wrong** — this is where Rosie differs from a plain SQL bot:

> How many orders have status 'archived'?

There is no such status. Instead of a confident "zero", the certificate says the value does
not exist and lists the ones that do.

**4. The silent-failure case, the one worth showing other people:**

> What is the total credit limit of customers, counting every order?

The join repeats each customer once per order, so the naive sum is inflated many times over. Rosie marks the total as
contradicted and reports both derivations.

**5. Follow-ups on any answer:**

> Explain how you got that.
> Now give me the full list of rows.

These use `explain_answer` and `export_answer`, which authorize with the signed handle from
the original answer rather than re-running anything the client asks for.

## The second derivation

The cross-check behind the `CROSS_CHECKED` and `CONTRADICTED` verdicts is on by default and
spends one extra read-only query on the answers it applies to. `AI_CROSS_CHECK=false` in
`.env` turns it off. What each verdict means, and what the certificate does not check, is in
`docs/CERTIFICATE.md`.

An installation created before the cross-check existed lacks one column in the certificate
table; if the engine reports it missing, add it:

```sql
ALTER TABLE rosie_answer_certificate ADD COLUMN cross_check TEXT NULL AFTER provenance;
```

---

## If something does not work

**"Server disconnected" or Rosie never appears.** Almost always the `node` path from Step 2.
Try the absolute path, and confirm the script path is right by running it by hand: the
command below should print a startup line and then sit waiting for input (Ctrl+C to stop).

```bash
/usr/local/bin/node /absolute/path/to/rosie/server/mcp/stdio-server.js
```

**Every question is refused with a security message.** The read-only database user is being
probed at startup and failing. The startup line printed by the command above says which
check failed. The same hold applies to the web widget, so `npm start` would refuse too.

**"DB_DATABASE env var is required".** The server could not find the repository's `.env`.
Check that the path in `args` points at the real `server/mcp/stdio-server.js` and that
`.env` sits at the repository root.

**Answers work but the checks never appear.** `AI_ANSWER_CHECKS` is set to `false`
somewhere. It defaults to on.

**Logs.** The server writes everything to stderr, which Claude Desktop collects at
`~/Library/Logs/Claude/mcp-server-rosie.log`. That file is the first place to look for any
failure not covered above.

---

## A note on what this is and is not

This is the **local** phase: over stdio the trust boundary is your machine, so there is no
authentication step and `ROSIE_MCP_USER` only labels the audit rows. Exposing Rosie to other
people over the network needs the remote transport and real OAuth identity, which is a
separate phase and depends on your identity provider.

Everything you ask through Claude Desktop is audited exactly like a question asked in the
widget, including the certificate, so the governance trail does not have a hole in it.

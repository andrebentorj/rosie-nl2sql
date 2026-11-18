# Rosie in Slack

People ask Rosie in Slack (1.37.0): by mentioning it in a channel, in a direct message, or with
the `/rosie` command. The answer goes through the same checks as the widget and the chat page,
as the person who asked: the validator and its allowlist, the masked columns, the row policies
for that person, the strict masking gate, the Answer Certificate, the audit trail and the
per-person quota. The verdict is the first line of every answer, before any figure, with what
was checked and what was not.

Scheduled answers can also be posted to a Slack channel
([SCHEDULED_ANSWERS.md](SCHEDULED_ANSWERS.md#slack-1370)).

Off by default. With `ROSIE_SLACK` unset Rosie serves neither Slack path and makes no call to
Slack.

## In six steps

1. **Create the app from the manifest.** Open
   `deploy/slack/manifest.json` (in the release's deploy kit), replace `rosie.example.com` with
   your Rosie address in its three URLs (events, interactivity, the command), and create the app at api.slack.com/apps: *Create New
   App*, *From an app manifest*, pick the workspace, paste the JSON. Slack must reach that
   address over https; Rosie behind the reverse proxy of [deploy/](../deploy/README.md) is enough.
2. **Install it** to the workspace (*Install App*). A workspace administrator may have to
   approve it.
3. **Copy two values** into `.env`: *Basic Information*, *Signing Secret*, into
   `ROSIE_SLACK_SIGNING_SECRET`; *OAuth & Permissions*, *Bot User OAuth Token* (`xoxb-...`), into
   `ROSIE_SLACK_BOT_TOKEN`.
4. **Name the workspace and who may ask:**

   ```
   ROSIE_SLACK=true
   ROSIE_SLACK_WORKSPACES=T0123ABCD          # the Workspace ID; required
   ROSIE_SLACK_ALLOWED_CHANNELS=C0123ABCD    # where Rosie answers (unset: any channel it is in)
   ROSIE_SLACK_ALLOWED_DOMAINS=example.com   # who may ask (or ROSIE_SLACK_ALLOWED_USERS)
   ```

   The Workspace ID is in the workspace's URL (`app.slack.com/client/T0123ABCD/...`), and
   `npm run doctor -- --online` prints it. A channel's ID is at the bottom of its details.
5. **Restart Rosie**, then run `npm run doctor -- --online`: it calls `auth.test` once with the
   token (nothing is posted) and says whether the token works, belongs to that workspace and
   carries every scope below.
6. **Verify the Request URL.** In the app's *Event Subscriptions*, Slack shows the Request URL as
   verified once Rosie answered its challenge (signed with the secret). *Interactivity &
   Shortcuts* must show `https://<your Rosie address>/slack/interactions` (the manifest sets it):
   that is where the buttons arrive. If it does not, the
   address is not reachable over https, or the signing secret is wrong (the log says
   `slack_signature_bad_signature`). Then invite Rosie to the channels where people will ask:
   `/invite @Rosie`.

## What the app may do, and why

The manifest asks for these bot scopes and nothing else:

| Scope | Why |
|---|---|
| `app_mentions:read` | Receive a mention of Rosie in a channel |
| `im:history` | Receive the direct messages people send to Rosie |
| `chat:write` | Answer in the thread or in the asker's direct messages with Rosie (the app's Messages tab), and post scheduled answers to a channel Rosie is a member of |
| `users:read` | Needed by `users:read.email` |
| `users:read.email` | Turn the member ID of a message into the e-mail address Rosie identifies the person by |
| `files:write` | Attach the CSV file of a result with more rows than the message shows |
| `commands` | The `/rosie` command. Remove it, and the `slash_commands` block, if you do not want the command |

Rosie cannot read channel history (it receives only messages that mention it and its own direct
messages), cannot join channels by itself, and cannot post in a channel it was not invited to.
`npm run doctor -- --online` warns about any scope the token carries beyond these.

The events are `app_mention` and `message.im`. Interactivity is on, at `/slack/interactions`, for
the buttons of an answer and of a clarification; it needs no scope of its own.

## Who asked, and who may ask

The person is never taken from the text of a message. Every request from Slack is checked
against the signing secret before any of it is read, so the member ID in it was put there by
Slack. Rosie then asks Slack for that member's account (`users.info`, with the bot token) and
takes the **e-mail address** from it. From there the chat page's rules apply
([CHAT_PAGE.md](CHAT_PAGE.md)):

- **User id:** the e-mail address in lower case. The same person signing in to the chat page
  gets the same id, so the audit trail, the row policies and the quota see one person.
- **Claims** for row policies ([ROW_POLICIES.md](ROW_POLICIES.md)): `email`, `name`,
  `slack_user_id`, `slack_team_id`, and the short names of `ROSIE_JWT_CLAIM_MAP` whose source is
  one of those (`{"mail":"email"}` gives `mail`). Slack has no groups or roles: a row policy that
  reads `groups` or `roles` finds nothing and refuses, as for any identity without them. Write
  policies that Slack users must pass against `email`.

Refused, each with a short, polite reply that only that person sees (and a `refused` audit row
with the reason `slack_<reason>`):

| Reason | When |
|---|---|
| `workspace_not_allowed` | The request, or the member's account, is not of `ROSIE_SLACK_WORKSPACES` (for the request itself no reply is posted) |
| `external_member` | A member of another organisation in a channel shared through Slack Connect (below) |
| `channel_not_allowed` | A channel that is not in `ROSIE_SLACK_ALLOWED_CHANNELS` (when set) |
| `direct_messages_off` | A direct message while `ROSIE_SLACK_DIRECT_MESSAGES=false` |
| `person_not_allowed` | Not on `ROSIE_SLACK_ALLOWED_USERS` and not in a domain of `ROSIE_SLACK_ALLOWED_DOMAINS` (when either is set) |
| `guest_account` | A single- or multi-channel guest, unless `ROSIE_SLACK_ALLOW_GUESTS=true` |
| `account_deactivated`, `not_a_person` | A deactivated account; a bot, an app or Slackbot |
| `no_email`, `email_unconfirmed` | Slack gave no address, or marks it unconfirmed (`is_email_confirmed: false`) |
| `missing_scope` | The token lacks `users:read.email`: add it and reinstall the app |
| `slack_not_asker` | A button pressed by someone other than the person the answer was given to |
| `slack_signature_*` | The request was not signed with the secret, or is more than five minutes old (401, no reply) |

**Shared channels (Slack Connect): people from other organisations are refused.** In a channel
shared with another company, a message from one of its members reaches Rosie through your
workspace, but the member's account belongs to theirs. Rosie checks the account's own workspace:
its `team_id`, or on Enterprise Grid one of the workspaces it belongs to
(`enterprise_user.teams`, or the organisation ID when that is what `ROSIE_SLACK_WORKSPACES`
lists). It must be in `ROSIE_SLACK_WORKSPACES`, and an account Slack marks as a stranger
(`is_stranger`) is refused whatever it names, even with an address in an allowed domain. The
event's workspace counts only for an account that names no workspace of its own.

**The e-mail address is trusted as your workspace keeps it.** An address Slack marks unconfirmed
(`is_email_confirmed: false`) is refused. When Slack leaves that field out (it does for some
accounts and workspaces), the address is accepted as it stands: Rosie then relies on your
workspace's e-mail policy. If members may change their profile e-mail without confirming it,
one of them could take another person's address and be answered as that person; keep e-mail
changes confirmed or managed by your identity provider.

`ROSIE_SLACK_ALLOWED_USERS` takes member IDs (`U0123ABCD`) and e-mail addresses. A person passes
when they are on that list **or** in a domain of `ROSIE_SLACK_ALLOWED_DOMAINS`. With neither set,
every member of the workspace may ask (guests excepted), and startup and doctor warn.

A member's account is kept in memory for `ROSIE_SLACK_IDENTITY_CACHE_SECONDS` (300). A
deactivated member is refused at most that long after the change; the allow-lists are applied
to every question.

## What people see

**In a channel:** mention Rosie, `@Rosie how many orders did we have last month?`. Where the
answer goes depends on `ROSIE_SLACK_CHANNEL_ANSWERS` (below): in the thread, or privately. In
the thread, a short "Working on it" message appears at once and is replaced by the answer. To
follow up, reply in the same thread and mention Rosie again (Rosie only receives messages that
mention it in channels): the follow-up carries the earlier turns of that thread and starts from
the previous answer's statement.

**In a direct message:** write the question. Rosie answers in the message's thread; reply in
that thread to follow up. A new message outside the thread starts a new conversation.

**`/rosie <question>`:** answered privately, in the channel where it was typed. Nobody else
sees the question or the answer. There is no follow-up and no file.

### Who sees an answer given in a channel

An answer is computed with the asking person's access: their row policies decide which rows it
counts. In a thread, everyone in the channel would see it. So:

| `ROSIE_SLACK_CHANNEL_ANSWERS` | Where an answer asked in a channel goes |
|---|---|
| `auto` (default) | **Privately when the context has row policies** (`row-policies.json`), in the thread when it has none. Checked at every question, so it follows a context reload |
| `thread` | In the thread, for everyone in the channel. `npm run doctor` warns when the context has row policies |
| `private` | Privately, always |

Privately means: a short note in the thread that only the asker sees ("I sent you the answer in
a direct message ..."), and the answer itself in the asker's direct messages with Rosie, starting
with where it was asked. A direct message rather than an "only visible to you" (ephemeral)
message, because a direct message is kept (an ephemeral message is gone on reload and on the
person's other devices), can be replaced when the answer is ready, carries the CSV file (an
ephemeral message cannot), keeps its buttons working and takes follow-ups in its thread. It uses
the `chat:write` scope the app already has. A direct message is always answered in place.

### The answer

The layout approved for 1.37.0, as Block Kit inside an attachment whose colour bar carries the
verdict's tone (the widget's own tones: red for CONTRADICTED; amber for FLAGGED, NOT CHECKED and
the holds; grey for clean verdicts). No emoji.

1. The verdict, its word in bold: `*CONTRADICTED.* This figure counts some records more than
   once. Do not use it as a fact.` with the recomputed figure, `*FLAGGED.*`, `*NOT CHECKED.*`,
   or for a clean result `*Checks passed.*` / `*Cross checked.*` with the sentence of
   [CERTIFICATE.md](CERTIFICATE.md) that says what it does not show. Never "verified" or
   "correct".
2. The answer in words.
3. The first `ROSIE_SLACK_ROWS_INLINE` rows (10) as preformatted text.
4. A grey line: the number of rows (with more rows: "10 of 25 rows shown, all of them in the CSV
   file below"), who asked, the approval or the declared metric, the chat page when it is on.
5. A divider, then **Warnings**, **What was checked**, **What was not checked** (three items, then
   "and N more, under Show the statement").
6. Buttons: **Show the statement**, **Explain**, **Helpful**, **Not helpful** (below).
7. With more rows than the message shows, a CSV file with all of them (the verdict rows on top,
   as the widget's export) in the same thread.

A **clarification** shows its options as buttons ("Or reply in this thread with your own words";
in a channel, mentioning Rosie). A **refusal** (not allowed, quota, Rosie busy) and a **hold**
(security hold, masked columns not confirmed, audit unavailable: amber bar, the code and a
reference) go only to the person. A refusal of the question by the engine (a write request, a
protected column) is the widget's sentence, in the thread.

**Escaping.** Every value from the data, the model, the question or a person's name that goes
into formatted text (the sections, the grey lines) is escaped, so it cannot become a link or a
mention, and `@channel` in an answer notifies nobody. The rows are sent as rich-text literal
text and the button labels as plain text, which Slack shows as typed and never reads as a
mention or a link.

**Plain-text fallback.** Every message carries its whole plain-text form (the verdict line
first, the rows, the checks, the statement) for notifications, screen readers and clients that
do not draw blocks: as `text` on a message whose blocks are at the top level (a refusal), as the
attachment's `fallback` on a message drawn in an attachment. Such a message has no top-level
`text`, because Slack draws a top-level `text` as the body of a message without top-level blocks,
which would print the answer twice.

### The buttons

Every press goes to `/slack/interactions`, is checked against the signing secret and the
workspace, and is acknowledged at once. Then:

- **The person pressing is identified again** (`users.info`, the allow-lists, the channel rules),
  exactly as for a question.
- **Only the person the answer was given to may use its buttons.** On a channel message everyone
  sees the buttons; anyone else gets a polite "only the person who asked can do this" (audited as
  `slack_not_asker`) and nothing is shown. The button carries the asker's member ID and the
  answer's signed handle, and the handle must also verify for the person pressing: a handle is
  never accepted for someone it was not minted for. Handles last 15 minutes; after that the
  buttons say the answer is no longer available.
- **Show the statement:** the statement that ran and every "not checked" item, only to that
  person. No engine call.
- **Explain:** what `POST /explain` does: the gates, the explain quota (shared with the route,
  per person), the statement validated against the catalog as it is now, the model's
  explanation, an audit row (`status` `explain`, user agent `slack (<workspace>/<channel>)`).
  Shown only to that person, marked as written by the model and not checked.
- **Helpful / Not helpful:** recorded like `POST /feedback` (`rosie_feedback`), under the verified
  person, with the question and statement of that answer; the feedback quota is shared with the
  route. A short thank-you only that person sees.
- **A clarification option:** asked as that person's next question, in the same thread (or
  direct message), exactly as if they had typed it.

## The same governed path

A question from Slack goes through `server/chat-platform/governed-ask.js`, which applies the
widget route's gates in the route's order and then calls the same orchestrator entry
(`processQuestion`) the route calls:

1. the assistant switched on (`AI_ASSISTANT_ENABLED`);
2. the security hold (the read-only account's privilege check);
3. the masked-columns hold (`ROSIE_REQUIRE_COLUMN_GRANTS`, on in strict masking mode);
4. the audit policy (`ROSIE_AUDIT_POLICY=fail-closed` refuses while nothing can record);
5. the person's quota: `AI_MAX_QUERIES_PER_HOUR` and `AI_MAX_QUERIES_PER_DAY`, counted in the
   same store and under the same key (`u:<user id>`) as the widget's `/ask` for a verified
   person, so a person asking in the chat page and in Slack has one quota
   (`ROSIE_RATE_LIMIT_STORE=database` shares it across replicas); Explain and the ratings count
   in the route's `/explain` and `/feedback` quotas the same way;
6. a slot of `AI_CONCURRENCY_LIMIT`, shared with the route;
7. the question: length, injection filter, write and sensitive requests, the model loop, the
   validator, masking, the row policies for the verified claims, the certificate.

A refusal at a gate is counted on `/metrics` (`route="slack"`) and audited like the route's.
Every question has its audit row and certificate row like any other, with:

| Column | Value |
|---|---|
| `user_id` | the person's e-mail address, lower case |
| `user_agent` | `slack (<workspace>/<channel>)`: how to find Slack's questions in the trail |
| `session_id` | `slack:<workspace>` |
| `conversation_id` | `slack:<channel>:<thread>` |
| `ip_address` | empty: the request comes from Slack, not from the person |

## Delivery and retries

Rosie acknowledges each request at once and answers afterwards (Slack gives an app 3 seconds).
A retried event (the same `event_id`) is answered once per process. With several replicas behind
one URL a retry could reach another one; Rosie acknowledges in milliseconds, so Slack has no
reason to retry, but the guarantee is per process.

The conversation of a thread (its last `ROSIE_SLACK_THREAD_TURNS` turns, 6, per person) is in the
memory of the process that answered it. With several replicas, a follow-up that reaches another
replica starts without that history; the answer is still governed and audited the same way.

What a button needs beyond its own value (the handle, when it is too long for a button, and the
full "not checked" list) is kept in memory for 15 minutes, per process. The handle itself is in
the button whenever it fits, so a press that reaches another replica still works.

A Slack call that fails is logged (`slack_send_failed`, with Slack's error code) and not
repeated for a question: the person can ask again. A scheduled delivery to Slack is retried on
`429`, `5xx` and timeouts ([SCHEDULED_ANSWERS.md](SCHEDULED_ANSWERS.md#failures-and-retries)).

## One workspace

The app is installed in one workspace, with one bot token. An Enterprise Grid organisation where
the same app is installed in several workspaces can list them all in `ROSIE_SLACK_WORKSPACES`.
Distributing the app to other organisations (OAuth installation with a token per workspace) is
not in this release.

## What leaves, and what is kept

- **To Slack:** the answers, in the threads where they were asked or in the asker's direct
  messages: the narrative, the first rows, the checks, a CSV file with all rows when they do not
  fit, and in the fallback text and on "Show the statement" the statement. The buttons carry the
  asker's member ID and the answer handle (it signs the question and the statement; no rows).
  Slack keeps them under your workspace's retention settings; erasure in Rosie
  (`npm run subject`) does not reach them. Record this flow in your record of processing
  ([COMPLIANCE.md](COMPLIANCE.md)).
- **From Slack:** the message text (the question), the member ID, and from `users.info` the
  member's account (name, e-mail address, guest and deactivated flags).
- **Kept by Rosie:** the audit and certificate rows, as for every question. In memory only: the
  member's account for `ROSIE_SLACK_IDENTITY_CACHE_SECONDS`, the last turns of each thread for
  24 hours, the IDs of recent events for 10 minutes, and per answer its handle and "not checked"
  list for 15 minutes. Nothing about Slack is written to disk
  beyond the audit trail.
- **To the model provider:** the question and the thread's earlier turns, as for the widget; for
  Explain, the statement and its question. Not the person's address or Slack IDs.

## Settings

| Setting | Default | What it does |
|---|---|---|
| `ROSIE_SLACK` | `false` | Answer in Slack |
| `ROSIE_SLACK_SIGNING_SECRET` | | The app's Signing Secret; every request is checked with it. Required |
| `ROSIE_SLACK_BOT_TOKEN` | | The Bot User OAuth Token, `xoxb-...`. Required. A user token (`xoxp-`) or an app-level token (`xapp-`) refuses startup |
| `ROSIE_SLACK_WORKSPACES` | | Workspace IDs that may ask. Required |
| `ROSIE_SLACK_ALLOWED_CHANNELS` | any | Channel IDs where Rosie answers and where schedules may post |
| `ROSIE_SLACK_DIRECT_MESSAGES` | `true` | Answer direct messages |
| `ROSIE_SLACK_ALLOWED_USERS` | | Member IDs or e-mail addresses that may ask |
| `ROSIE_SLACK_ALLOWED_DOMAINS` | | E-mail domains that may ask |
| `ROSIE_SLACK_ALLOW_GUESTS` | `false` | Guest accounts may ask |
| `ROSIE_SLACK_CHANNEL_ANSWERS` | `auto` | `auto`: privately when the context has row policies, in the thread otherwise; `thread`: for everyone in the channel; `private`: in the asker's direct messages |
| `ROSIE_SLACK_THREAD_TURNS` | `6` | Earlier turns of a thread given with a follow-up (0 to 10) |
| `ROSIE_SLACK_ROWS_INLINE` | `10` | Rows in the message (1 to 50); more go as a CSV file |
| `ROSIE_SLACK_IDENTITY_CACHE_SECONDS` | `300` | How long a member's account is kept (0 to 3600) |
| `ROSIE_SLACK_API_URL` | `https://slack.com/api` | Slack's Web API; change it only for a test double |

Refuse startup while `ROSIE_SLACK=true`: no signing secret or one shorter than 32 characters, a
token that is not a bot token, no workspace, an ID that is not one (a channel name instead of its
ID), an e-mail domain that is not a domain, a Web API URL that is not https (plain http only to
localhost outside production), a channel-answers mode other than auto, thread or private.
Warned at startup and by doctor: no allow-list of people, no allow-list of channels, guests
allowed, a Web API URL other than Slack's. Warned by doctor: answers in the thread
(`ROSIE_SLACK_CHANNEL_ANSWERS=thread`) while the context has row policies.

## Checking it

```bash
npm run doctor                 # settings, Request URLs, who may ask, where channel answers go (no call to Slack)
npm run doctor -- --online     # also auth.test: the token, its workspace, its scopes
```

The tests run against a local fake of Slack's Web API (`test/lib/fake-slack.js`), with no model
and no database: `node test/unit/slack.js`.

## Troubleshooting

| What you see | Why |
|---|---|
| Slack says the Request URL did not respond | Rosie is not reachable over https at that address, or `ROSIE_SLACK` is off |
| Request URL verification fails, the log says `slack_signature_bad_signature` | The signing secret in `.env` is not the app's |
| `slack_signature_stale_timestamp` in the log | This server's clock is more than five minutes off (use NTP) |
| Rosie does not react to a mention | It is not in the channel (`/invite @Rosie`), or the event subscription lacks `app_mention` |
| Everyone is told "Rosie could not read your e-mail address" | The token lacks `users:read.email`; add it and reinstall |
| An answer has no CSV file | The token lacks `files:write`, or it came from `/rosie` (ask in a direct message for the file) |
| Pressing a button does nothing | *Interactivity* is off or points elsewhere: it must be `<your Rosie address>/slack/interactions` |
| "That answer is no longer available" on a button | More than 15 minutes passed (the handle's lifetime), or another replica answered and the handle did not fit in the button: ask again |
| People in a channel get "I sent you the answer in a direct message" | The context has row policies and `ROSIE_SLACK_CHANNEL_ANSWERS` is `auto` (the default): intended |
| A scheduled post fails with `not_in_channel` | Invite Rosie to that channel |

## What it does not do (this release)

- No follow-up in a channel thread without a mention.
- No installation in other organisations (OAuth distribution).
- No Socket Mode: Slack must reach Rosie over https.
- Thread history and event de-duplication are per process.

# Rosie in Microsoft Teams

People ask Rosie in a personal chat with the bot, or in a channel or group chat it was set up
for, and get the same governed answer as the widget: the validator and its allowlist, the
masked columns, the row policies for the person who asked, the strict gate, the Answer
Certificate and the audit trail. Every answer starts with its verdict. Scheduled answers can be
posted to a Teams conversation too.

Off unless `ROSIE_TEAMS=true`. While it is off nothing answers at `/api/messages`.

## What people see

An answer is one **Adaptive Card**, in this order:

1. **The verdict band**, across the top of the card: CONTRADICTED in red (Teams' attention
   colour), FLAGGED, NOT CHECKED and any verdict Rosie does not know in yellow, the clean
   verdicts (CHECKS PASSED, CROSS CHECKED, NOT APPLICABLE) in grey, each with its sentence. A
   clean verdict is never called "verified" or "correct": the checks look at the structure of a
   calculation, not at whether it is the figure the question meant ([CERTIFICATE.md](CERTIFICATE.md)).
2. The answer's sentence.
3. The first `ROSIE_TEAMS_MESSAGE_ROWS` rows (10 by default) as a table, numbers right aligned,
   and how many rows there are.
4. For a caveat (CONTRADICTED, FLAGGED, NOT CHECKED): the warnings, what was checked and what was
   not, in the card. For a clean verdict they are behind **What was checked**.
5. Buttons: **Statement** (the statement that ran), **Get all rows as CSV** (when not every row
   fits), **Helpful** and **Not helpful**.

A clarification is a card with one button per option; a refusal a grey "NOT ANSWERED" band with
the reason; a hold a yellow "PAUSED" band with the code and whom to tell. Teams shows the first
line of the plain text below in notifications.

**Nothing from your data is read as formatting.** Cells, the statement, warnings and the options
are plain text runs that Teams does not read as Markdown, so a value cannot make a link, a
mention, bold text or a list. The answer's sentence keeps the engine's bold and loses any link
or mention syntax. Web addresses in any text are broken with an invisible space so Teams does not
make them clickable. The plain form (below) holds to the same rule: every value, display name and
model sentence in it has its link, image, HTML, mention, heading and emphasis syntax broken with an
invisible space, so it reads the same and nothing in it acts; only Rosie's own words (the verdict
line, the headings) are bold.

**The plain form.** The card carries its plain text as its fallback, and when Teams refuses a
card the plain text is sent instead; `ROSIE_TEAMS_CARDS=false` sends only the plain text. It is
the same content, verdict line first (in bold for a caveat), the rows in a fixed-width block,
the checks and the statement; a clarification has numbered options, answered with the number or
the words. In the plain form, a personal chat gets the CSV file offered right away instead of
the button.

**More rows than fit.** *Get all rows as CSV* offers the file in the clicker's personal chat
(Teams asks them to accept it; the file goes to their OneDrive). The CSV carries the verdict on
its first row, as the widget's export does. Only the person the offer was made for can accept
it, within 15 minutes. In the plain form in a channel thread, the message gives the chat page's
address when the chat page is on ([CHAT_PAGE.md](CHAT_PAGE.md)).

Rosie keeps, per conversation **and** person, the last `ROSIE_TEAMS_HISTORY_TURNS` questions and
answers (3 by default) and the last answer's signed handle, so "and by month?" works. They are kept in the
memory of the process for 30 minutes and written nowhere. Someone else in the same channel
starts fresh.

## Before you start

- Rosie reachable from the internet over **https**, at a public host name (the Bot Framework
  posts to it). Behind a reverse proxy, set `TRUST_PROXY`.
- Someone who can register an application in **Microsoft Entra ID** and create an **Azure Bot**
  resource in an Azure subscription, and a Teams administrator who can upload a custom app.
## Set up

### 1. Register the bot's application in Entra ID

Azure portal > Microsoft Entra ID > App registrations > New registration:

- Name: `Rosie` (what your administrators see).
- Supported account types: **Accounts in this organizational directory only (Single tenant)**.
- No redirect URI.

Then:

- Copy the **Application (client) ID** (`ROSIE_TEAMS_APP_ID`) and the **Directory (tenant) ID**
  (`ROSIE_TEAMS_TENANT`) from the Overview page.
- Certificates & secrets > New client secret. Copy its **Value** at once (`ROSIE_TEAMS_APP_SECRET`;
  not the Secret ID). Note when it expires: on that day the bot stops answering, and
  `npm run doctor -- --online` says so.
- Only if you will use `ROSIE_TEAMS_ALLOWED_GROUPS`, or a row policy reads `groups`: API
  permissions > Add a permission > Microsoft Graph > **Application** permissions >
  `GroupMember.Read.All`, then **Grant admin consent**. Rosie then asks Graph which groups a
  person is in. Without groups, the app needs no API permission at all.

### 2. Create the Azure Bot resource

Azure portal > Create a resource > **Azure Bot**:

- Type of App: **Single Tenant**. Creation type: **Use existing app registration**, with the
  app id and tenant id of step 1.
- Pricing tier: Standard channels (Teams) are free.

Then, in the bot resource:

- Configuration > **Messaging endpoint**: `https://<your Rosie host>/api/messages`.
- Channels > **Microsoft Teams** > accept the terms > Apply (Microsoft Teams Commercial).

### 3. Configure Rosie

```bash
ROSIE_TEAMS=true
ROSIE_TEAMS_APP_ID=<Application (client) ID>
ROSIE_TEAMS_APP_SECRET=<client secret value>
ROSIE_TEAMS_TENANT=<Directory (tenant) ID>
# who may ask (any of these; with none, anyone in your directory who can install the app):
ROSIE_TEAMS_ALLOWED_DOMAINS=example.com
# ROSIE_TEAMS_ALLOWED_USERS=dana@example.com,lee@example.com
# ROSIE_TEAMS_ALLOWED_GROUPS=<group object id>
```

Restart, then `npm run doctor -- --online`: it fetches the Bot Framework's and your directory's
signing keys and asks Entra ID for the bot's app token, which proves the app id, the secret and
the tenant (nothing is sent to Teams). Without `--online` doctor checks the settings only.

### 4. Make and upload the Teams app

```bash
npm run teams:manifest -- --host rosie.example.com --developer "Example Ltd" \
  --website https://example.com --privacy https://example.com/privacy --terms https://example.com/terms
```

This writes `rosie-teams.zip` (manifest.json and the two icons) for the bot of
`ROSIE_TEAMS_APP_ID`. The icons are `deploy/teams/color.png` (192x192) and
`deploy/teams/outline.png` (32x32, white on transparent), exported from Rosie's mark,
`site/assets/rosie-still.svg`: the colour icon as drawn, the outline from the same shapes (the
face screen as a cut-out, the parts that are only light or colour left out). `--color` and
`--outline` replace them; icons of other sizes are refused. Upload the package in the **Teams
admin center** > Teams apps > Manage apps > Upload new app, and allow it in your app permission
policy; or, for a trial, in Teams > Apps > Manage your apps > Upload an app (custom app upload
must be allowed for you). Then add the app to a personal chat, or to a team.

The manifest declares the bot for personal chats, teams and group chats, and `supportsFiles`
(for the CSV file in personal chats). Rosie answers in channels and group chats only where
`ROSIE_TEAMS_CHANNELS` lists them (below), whatever the manifest allows.

## Who may ask, and where

**Who.** The person is taken from the activity the Bot Framework signed, never from the message
text: their Entra ID object id and tenant, then their **user principal name** from the Bot
Connector's member info (the member's `email` field is not used: Microsoft documents the mail
attribute as unverified). From there the rules are the chat page's with Microsoft
([CHAT_PAGE.md](CHAT_PAGE.md#who-the-person-is)):

- the user id is the UPN in lower case, or the object id with `ROSIE_JWT_USER_CLAIM=oid` (an id
  nobody can rename). The same person on the chat page and in Teams is one user id: one quota,
  one subject for `npm run subject`;
- row policies read `email` (the UPN), `oid`, `tid`, `name`, `preferred_username`, `upn`, the
  short names of `ROSIE_JWT_CLAIM_MAP`, and `groups` when groups are read (only the allowed
  groups the person is in; all of them when a row policy reads `groups`);
- the lists that are set must all pass: the tenant (your own, or one of
  `ROSIE_TEAMS_ALLOWED_TENANTS`), `ROSIE_TEAMS_ALLOWED_DOMAINS` (the UPN's domain), and, when
  either is set, `ROSIE_TEAMS_ALLOWED_USERS` (listed by UPN) **or** `ROSIE_TEAMS_ALLOWED_GROUPS`
  (a member of one of the groups).

Anyone else gets a polite refusal naming whom to tell (`supportContact` of `project.json`), once
in ten minutes per conversation, and the refusal is recorded.

**Where.** Personal chats, unless `ROSIE_TEAMS_PERSONAL=false`. Channels and group chats only
when `ROSIE_TEAMS_CHANNELS` lists them: a channel id, a team id (every channel of the team) or a
group chat id. In a channel the bot answers only when mentioned (Teams sends it nothing else).
A conversation is personal only when Teams says so: one with no type, or a type Rosie does not
know (a meeting chat), is treated as a channel and needs to be listed.

**How the two rules combine.** First *where*: a channel or group chat that `ROSIE_TEAMS_CHANNELS`
does not list gets no answer at all (none is listed by default, so out of the box Rosie answers
only in personal chats). Then, in a listed one, *who reads it*, below.

**Who reads an answer asked in a channel.** An answer is made with the access of the person who
asked. When the context has **row policies** ([ROW_POLICIES.md](ROW_POLICIES.md)), two people can
get different rows for one question, so by default (`ROSIE_TEAMS_CHANNEL_ANSWERS=auto`) the answer
goes **only to the person who asked**, in a personal chat with the bot that Rosie opens (Teams has
no message that only one member of a channel sees). The thread gets one line, "I sent the answer
to <name> in a personal chat", with nothing of the answer in it; if the personal chat cannot be
opened, the thread says so and the answer is posted nowhere. Without row policies the answer goes
in the thread, for everyone there. `ROSIE_TEAMS_CHANNEL_ANSWERS=thread` or `private` forces one or
the other; with `thread` and row policies, startup and `npm run doctor` warn that one person's rows
are shown to all.

To find a channel's id: in Teams, the channel's **...** > Get link to channel. The link holds it,
URL-encoded: `19%3a...%40thread.tacv2` is `19:...@thread.tacv2`. A team's id is the id of its
General channel.

**Quotas.** The person's hourly and daily quotas of `AI_MAX_QUERIES_PER_HOUR` and
`AI_MAX_QUERIES_PER_DAY` are the same counters as the widget's `/ask` and Slack's (key
`u:<user id>`, in the engine database for every replica with `ROSIE_RATE_LIMIT_STORE=database`),
and a Teams question takes one of the `AI_CONCURRENCY_LIMIT` slots the route uses
(`server/lib/ask-limits.js`). Asking on the chat page and in Teams draws from one quota.

**Buttons.** By the same rules as Slack's: whoever clicks is identified again from the activity
the Bot Framework signed, like the author of a message, passes the same allow-lists, and acts
only on their own answer (its signed handle names them). *Helpful* and
*Not helpful* (`{ "rosieAction": "feedback", "messageId", "rating": "up"|"down" }`) are recorded
like `POST /feedback`, with the question and statement of the answer's signed handle, on the
person's feedback quota; *Explain* (`"explain"`) is `POST /explain`'s path (its quota, the
statement validated again, an audit row); *Statement* (`"statement"`) shows the statement of the
person's own answer; *Get all rows as CSV* (`{ "rosieAction": "csv", "messageId" }`) offers the file in the clicker's personal
chat; an option of a clarification (`{ "rosieOption", "messageId" }`) is asked as the next
question. A click on someone else's answer is refused and recorded (`teams_button_not_owner`).
Answers are remembered for buttons for an hour, in memory.

## Scheduled answers in Teams

A schedule can post to a Teams conversation ([SCHEDULED_ANSWERS.md](SCHEDULED_ANSWERS.md#teams)):

```json
{ "type": "teams", "conversation": "19:abc123...@thread.tacv2" }
```

The bot must be installed in that team or chat. The message is verdict first, like every other.
It goes to `ROSIE_TEAMS_SERVICE_URL` (`https://smba.trafficmanager.net/teams/`, Microsoft's
global address for messages a bot starts), with the bot's app token.

## Multi-tenant

`ROSIE_TEAMS_APP_TYPE=multi`, with a multi-tenant app registration and Azure Bot, serves the
directories of `ROSIE_TEAMS_ALLOWED_TENANTS` and no other; Rosie refuses to start with `multi`
and an empty list (it would mean every Microsoft 365 organisation). The app token is then asked
for at `botframework.com`. Each directory still passes the person lists above, and groups need
`GroupMember.Read.All` consented in each.

## Security

What is checked on every `POST /api/messages`, before anything in the activity is acted on
(`server/teams/auth.js`):

- a `Bearer` JWT signed with **RS256** by a key of the **Bot Framework**
  (`https://login.botframework.com/v1/.well-known/openidconfiguration`, issuer
  `https://api.botframework.com`) that is **endorsed for `msteams`**, with a `serviceUrl` claim
  equal to the activity's; or by a key of **your directory** (issuer
  `https://login.microsoftonline.com/<tenant>/v2.0` or `https://sts.windows.net/<tenant>/`),
  issued **to this bot's app** (`azp` or `appid` equal to `ROSIE_TEAMS_APP_ID`, which only a
  holder of the bot's secret can obtain) in your tenant, as an **app-only** token: a delegated
  token (a person's sign-in to the same app registration, with `scp`, or `idtyp` "user") is
  refused;
- `aud` equal to `ROSIE_TEAMS_APP_ID`, `exp` not past and `iat` not in the future, five minutes
  of clock tolerance (the Bot Framework's);
- the activity's channel is `msteams`, and its `serviceUrl` is https on a host of
  `ROSIE_TEAMS_CONNECTOR_HOSTS` (`smba.trafficmanager.net` and its subdomains by default): the
  bot's app token is only ever sent there.

A request that fails gets 401 (403 when the signing key is not endorsed for Teams, as the Bot
Framework documents), is logged (`teams_auth_refused`, with a short reason, never token content)
and recorded as a refusal of route `teams`. The signing keys are cached for at most 24 hours, so
a key Microsoft withdraws stops being accepted within a day; an unknown key id refetches them at
most once in five minutes. Tokens of the Bot Framework Emulator (issued in Microsoft's
`botframework.com` directory) are not accepted: test with Teams itself.

Then the person, and the question through the governed door shared with Slack
(`server/chat-platform/governed-ask.js`): the holds of the `/ask` route in its order (the
assistant switched off, the read-only account's privilege check, the strict gate of
`ROSIE_REQUIRE_COLUMN_GRANTS`, the audit policy), the quota and a concurrency slot, then the same
entry as the widget's `/ask` (`orchestrator.processQuestion`). A follow-up starts from the
previous answer's signed handle, verified for the person asking. The audit row of a Teams question has the user agent
`rosie-teams/1 (personal|channel|groupChat)` and a conversation id `teams:<hash>`; a CSV file
sent is recorded in the audit file as `rosie_teams_file`.

What Rosie keeps of a conversation: the history and the last answer's handle in memory (above),
answers for buttons for an hour, a CSV file offer for 15 minutes, the person's identity for 10
minutes. Nothing is written but the
audit trail. Several replicas work (any replica can answer any activity), but a follow-up or a
file accepted on another replica than the one that answered starts without that memory: the
person asks again.

No Bot Framework SDK is used: the token checks use `jose`, the connector calls are four REST
calls (`server/teams/connector.js`).

## Settings

| Setting | Default | What it does |
|---|---|---|
| `ROSIE_TEAMS` | `false` | the bot on |
| `ROSIE_TEAMS_APP_ID` | | Application (client) ID |
| `ROSIE_TEAMS_APP_SECRET` | | client secret value |
| `ROSIE_TEAMS_TENANT` | | Directory (tenant) ID |
| `ROSIE_TEAMS_APP_TYPE` | `single` | `single` or `multi` |
| `ROSIE_TEAMS_ALLOWED_TENANTS` | | directories of a multi-tenant bot (required with `multi`) |
| `ROSIE_TEAMS_ALLOWED_DOMAINS` | | UPN domains that may ask |
| `ROSIE_TEAMS_ALLOWED_USERS` | | UPNs that may ask |
| `ROSIE_TEAMS_ALLOWED_GROUPS` | | group object ids whose members may ask |
| `ROSIE_TEAMS_PERSONAL` | `true` | answer in personal chats |
| `ROSIE_TEAMS_CHANNELS` | none | channels, teams and group chats where it answers |
| `ROSIE_TEAMS_CARDS` | `true` | answers as Adaptive Cards; `false`: the plain text only |
| `ROSIE_TEAMS_CHANNEL_ANSWERS` | `auto` | channel answers: `auto` (privately when the context has row policies), `thread`, `private` |
| `ROSIE_TEAMS_MESSAGE_ROWS` | `10` | rows in a message (1 to 50) |
| `ROSIE_TEAMS_HISTORY_TURNS` | `3` | earlier turns kept (0 to 10) |
| `ROSIE_TEAMS_SERVICE_URL` | `https://smba.trafficmanager.net/teams/` | connector of scheduled deliveries |
| `ROSIE_TEAMS_CONNECTOR_HOSTS` | `smba.trafficmanager.net` | hosts that may receive the app token |

Rosie refuses to start on: a missing app id, secret or tenant, one that is not a GUID, `multi`
without its list, a single-tenant bot listing another tenant, a channel id, group, user or domain
that is not one, a service URL off the connector hosts or not https, a loopback connector host
in production.

## Troubleshooting

| What you see | Why |
|---|---|
| The bot never answers; the log has `teams_auth_refused` | The reason says which check: `claim_aud` (the Azure Bot uses another app id than `ROSIE_TEAMS_APP_ID`), `key_not_endorsed` (not a Teams activity), `service_url_not_allowed` (a connector host outside `ROSIE_TEAMS_CONNECTOR_HOSTS`, such as a government cloud: add its host), `expired` or `claim_iat` (this machine's clock). |
| Nothing in the log at all | The messaging endpoint of the Azure Bot does not reach Rosie: check the address, https, and the proxy. |
| `teams_reply_failed` with 401 | The app token is not accepted by the connector: `npm run doctor -- --online`; an expired secret is the usual cause. |
| "Rosie could not confirm who you are with Teams just now" | The member lookup failed (`teams_identity_failed` in the log); usually the app token, as above. |
| "You are not on the list of people who may ask Rosie" | The log line `teams_not_allowed` has the reason: `domain_not_allowed`, `not_listed_or_in_group`, `tenant_not_allowed`. |
| A channel gets "Rosie answers only in the chats and channels it was set up for" | Add its id to `ROSIE_TEAMS_CHANNELS`. |

## Not built

- Commands in the manifest's command list, and message extensions.
- Sovereign clouds (GCC High, DoD, China): their login and connector hosts differ; the
  connector host can be listed, the login host cannot be changed yet.

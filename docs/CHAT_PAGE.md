# The chat page with sign-in

Since 1.35.0 Rosie can serve its own chat page. People open Rosie's address, sign in with
Microsoft Entra ID, Google Workspace or any OpenID Connect provider, and ask their questions
on a full page. Every question runs as the person who signed in: row policies filter by their
claims, and the audit trail records their name. Nobody has to build or host anything.

The page is off by default. Embedding the widget in your own application
([INSTALL.md](INSTALL.md), Step 6) keeps working as before, and both can run at once.

## What people see

- **Sign-in**: the Rosie mark, "Sign in to Rosie" (the assistant name, when `project.json`
  sets `brand.assistantName`), the project's `description`, one button per configured
  provider in the order Microsoft, Google, SSO, and a footer with the project's `fullName`
  and `supportContact`.
- **Signed in**: a slim top bar with the project name, the person's name and address (the
  address is dropped on phones) and "Sign out", and the chat under it, filling the page.
- **Paused**: when Rosie holds questions (a security hold), the chat shows its usual hold
  card with the code and whom to tell.
- **Not allowed** (`E_SIGNIN_NOT_ALLOWED`): the provider confirmed who the person is, but
  they are not on the allow-list. The page names the account used, whom to ask, the code, and
  offers "Sign in with another account".
- **Sign-in failed** (`E_SIGNIN_FAILED`): the provider did not confirm who the person is (or
  the sign-in took longer than 10 minutes, or was started in another browser). The page offers
  "Try again" and whom to tell.

### When the sign-in screen is skipped

The sign-in screen is shown only when it is needed. Opening the page without a session:

1. **Right after signing out** (10 minutes): the sign-in screen, always. Nothing happens by
   itself, so a person who signed out stays signed out.
2. **First visit**: one **silent sign-in**. Rosie asks the provider with `prompt=none`: a
   person already signed in at the provider (in Microsoft 365, in Google Workspace) is signed
   in to Rosie without a click and without seeing anything. With several providers this is
   tried only with the one this browser used last (remembered for 30 days). If the provider
   answers `login_required`, `interaction_required`, `consent_required` or
   `account_selection_required`, Rosie goes on to step 3 or 4, with no error page. Any other
   provider error is `E_SIGNIN_FAILED`. A silent success goes through the same checks as any
   sign-in, the allow-lists included.
3. **Exactly one provider configured**: straight to it, interactively (no sign-in screen).
   `ROSIE_CHAT_AUTO_REDIRECT=false` turns this step off and shows the screen.
4. **Otherwise**: the sign-in screen.

The silent attempt is made at most once per browser every 5 minutes, and never again from
the provider's answer, so it cannot loop. When the session ends while the page is open, the
page reloads `/` and these same steps apply, so a person still signed in at the provider is
usually back without noticing.

| Setting | Default | Meaning |
|---|---|---|
| `ROSIE_CHAT_AUTO_REDIRECT` | `true` | With exactly one provider, go straight to it instead of showing the sign-in screen |

Three small cookies carry this: `rosie_silent` (a silent attempt was made, 5 minutes),
`rosie_signed_out` (10 minutes) and `rosie_provider` (the provider used last, 30 days). With
https they have the `__Host-` prefix. They hold no identity and only choose the first screen.

Set `supportContact` in `project.json` (a string, or `{ "name": ..., "contact": ... }`): it is
shown on the sign-in page before anyone signs in, and on both error pages. Without it they say
"the person who looks after Rosie at your company".

## Set up

1. Decide the address people will open, for example `https://rosie.example.com`. Rosie must
   be reachable there over https, usually behind your reverse proxy (`TRUST_PROXY=1`). The
   redirect URL to register at the provider is that address plus `/auth/callback`.
2. Register Rosie at one or more providers (below) and note the client id and secret.
3. Set in `.env`:

   ```bash
   ROSIE_CHAT_PAGE=true
   ROSIE_CHAT_PUBLIC_URL=https://rosie.example.com
   ROSIE_CHAT_SESSION_SECRET=<openssl rand -base64 48>
   ROSIE_CHAT_ALLOWED_DOMAINS=example.com
   # and the provider settings below
   ```

4. Run `npm run doctor`. Its "Chat page" section prints the redirect URL to register and
   fetches each provider's discovery document (no sign-in, no token exchange).
5. Restart, open the address, sign in.

Startup refuses a configuration it cannot use safely, with one line per problem: a public URL
that is not https (plain http is accepted only for `localhost` outside production, for
trials), a session secret under 32 characters, a provider with only some of its settings, a
Microsoft tenant that is not one directory, or Google without `ROSIE_CHAT_ALLOWED_DOMAINS`.

### Microsoft Entra ID

1. In the Microsoft Entra admin center, App registrations, New registration. Supported
   account types: "Accounts in this organizational directory only" (single tenant).
2. Redirect URI: platform **Web**, `https://rosie.example.com/auth/callback`.
3. Certificates & secrets: New client secret. Copy its **Value** (not its id).
4. Token configuration (optional): add the `email` optional claim to the ID token, and the
   `groups` claim if you allow by group. With many groups, prefer **app roles** (App roles,
   then assign people or groups in Enterprise applications) and list the role values in
   `ROSIE_CHAT_ALLOWED_GROUPS`: a person in more than 200 groups gets no `groups` claim and is
   refused with the reason `groups_overage`.

```bash
ROSIE_OIDC_MICROSOFT_TENANT=<Directory (tenant) ID, a GUID>
ROSIE_OIDC_MICROSOFT_CLIENT_ID=<Application (client) ID>
ROSIE_OIDC_MICROSOFT_CLIENT_SECRET=<the secret's Value>
```

Only your own directory can sign in: the tenant must be the GUID (`common`, `organizations`
and `consumers` are refused) and every ID token's `tid` is checked against it.

The person's address is the tenant's sign-in name, `preferred_username`, else `upn`, in
lower case: the default user id, the `email` row policies read, and what the domain list is
checked against. The `email` claim is never used: Microsoft documents it as unverified (it is
the directory's mail attribute, which can be edited), so a person whose mail attribute says
`ceo@example.com` and who signs in as `mallory@example.com` is `mallory@example.com` in Rosie.
For an id nobody can rename, set `ROSIE_JWT_USER_CLAIM=oid` (the object id; the domain list
still reads the sign-in name).

### Google Workspace

1. In the Google Cloud console, APIs & Services, OAuth consent screen: user type **Internal**
   (only accounts of your Workspace organisation).
2. Credentials, Create credentials, OAuth client ID, type **Web application**. Authorized
   redirect URI: `https://rosie.example.com/auth/callback`.

```bash
ROSIE_OIDC_GOOGLE_CLIENT_ID=<...apps.googleusercontent.com>
ROSIE_OIDC_GOOGLE_CLIENT_SECRET=<client secret>
ROSIE_CHAT_ALLOWED_DOMAINS=example.com
```

Google needs the domain list: without it any Google account in the world could sign in, so
startup refuses. Both the verified address (`email_verified` must be true) and the Workspace
domain claim `hd` must be in the list, so a personal Google account that happens to use a
company address is refused. Google sends no groups in the ID token; use the domain list.

### Any OpenID Connect provider (Okta, Auth0, Keycloak, Ping, ...)

Register a web application with the authorization code flow, PKCE allowed, a client secret,
and the redirect URL above.

```bash
ROSIE_OIDC_ISSUER=https://login.example.com          # exactly the provider's issuer
ROSIE_OIDC_CLIENT_ID=<client id>
ROSIE_OIDC_CLIENT_SECRET=<client secret>
ROSIE_OIDC_LABEL=Okta                                # the button reads "Sign in with Okta"
```

Rosie reads `<issuer>/.well-known/openid-configuration`; the document must name the same
issuer, and its endpoints must be https. The provider must offer PKCE with S256 and client
secret authentication (Basic or in the form). For groups, configure the provider to put a
`groups` (or `roles`) claim in the ID token.

The address counts only when the token says `email_verified: true` (the boolean or the
string `"true"`). An absent claim, `false` or `"false"` do not count: the user id is then
`sub`, no `email` is passed to row policies, and the domain list refuses the sign-in
(`no_verified_email`). Some providers never send the claim (some ADFS and federated
set-ups). For them, `ROSIE_OIDC_EMAIL_UNVERIFIED_OK=true` accepts the address when the claim
is absent (an explicit false is still refused). **This is a risk**: at a provider where people
can set or change their own address without verification, a person could sign in as a
colleague, pass the domain list and read what the colleague's row policies allow. Turn it on
only when the provider alone controls the address. Startup and `npm run doctor` warn while it
is on.

## Who may sign in

| Setting | Meaning |
|---|---|
| `ROSIE_CHAT_ALLOWED_DOMAINS` | Comma-separated e-mail domains. The person's verified address must be in one of them. Required with Google |
| `ROSIE_CHAT_ALLOWED_GROUPS` | Comma-separated values; one of them must be in the person's `groups` or `roles` claim (Entra group object ids or app role values, Okta group names, ...) |

With both set, both must pass. With neither, anyone the provider authenticates may sign in,
which is reasonable only when the provider itself restricts who can use the application (an
Entra app with "assignment required", an Okta app assigned to a group); startup and doctor
say so.

Being allowed to sign in gives no access to data beyond what the context already allows:
the catalog, the masked columns and the [row policies](ROW_POLICIES.md) still decide what any
question can read.

## Who the person is

The verified ID token becomes the identity of every request the page makes, through the same
path as a verified JWT ([IDENTITY.md](IDENTITY.md)):

- **User id**: the verified address in lower case (Google: `email`; Microsoft:
  `preferred_username`, else `upn`, never `email`; SSO: `email` with `email_verified` true), or
  `sub` when there is none. Set `ROSIE_JWT_USER_CLAIM` (for example `oid` at Microsoft) to use
  another claim; its value must be a string, or the sign-in fails (`no_user_id`), so a list or
  an object can never give several people one id. That claim is taken as it is, with no
  verification: `ROSIE_JWT_USER_CLAIM=email` makes the raw `email` claim the user id whether
  the provider verified it or not (the domain list still reads only the verified address), so
  do not set it to `email` at a provider whose addresses people can change. This is the id in the audit trail, the owner of answer handles, the
  key of the question quota and the subject of `npm run subject`.
- **Name**: `ROSIE_JWT_NAME_CLAIM` (default `name`), shown in the top bar.
- **Claims for row policies**: `sub`, `iss`, `email` (the verified address only),
  `email_verified`, `name`, `preferred_username`, `upn`, `oid`, `tid`, `hd`, `groups`,
  `roles`, every claim named by `ROSIE_JWT_CLAIM_MAP` (and its short names) and the user and
  name claims. A policy that needs a claim the token does not carry refuses, as with a JWT.
- **Groups and roles** are kept whole only when a row policy reads them (directly or under a
  short name of the map) or `ROSIE_JWT_CLAIM_MAP` names them. Otherwise only the values listed in `ROSIE_CHAT_ALLOWED_GROUPS` are
  kept: who may sign in is decided on the whole token first, so a person in hundreds of
  groups can sign in and the session stays small.

What the page or a request body says about the user (`user-id`, `user: { id }`) is ignored:
the session cookie is the only source.

## Sessions

| Setting | Default | Meaning |
|---|---|---|
| `ROSIE_CHAT_SESSION_SECRET` | none, required | Seals the cookies (32 characters or more). Changing it signs everyone out |
| `ROSIE_CHAT_SESSION_HOURS` | 8 | Longest a sign-in lasts, 1 to 168 |
| `ROSIE_CHAT_IDLE_MINUTES` | 60 | A session unused this long ends, 5 to 1440 |

The session is a cookie sealed by the server (AES-256-GCM), not an id into a server-side
store: any replica can serve any request with no shared store, and the claims are readable
neither by page scripts (HttpOnly) nor in a copy of the cookie jar. The cookie is
`__Host-rosie_session` (Secure, path `/`, no domain) on https, SameSite=Lax, and it holds the
verified claims needed, never a token from the provider. Rosie keeps no provider token at all
and uses no refresh token: when the session ends, the person signs in again (at the provider
this is usually one click).

Sign-out ends the session in that browser and on that process at once. A copy of the cookie
taken before sign-out would stay valid on another replica until its idle or absolute limit;
set the idle limit accordingly, or change the secret to end every session.

**Sign-out ends Rosie's session, not the provider's.** The person stays signed in at Microsoft,
Google or your provider, so once the 10-minute "signed out" marker is gone, opening the page
signs the same person in again silently. That is how single sign-on behaves; on a shared
computer, sign out at the provider too. `ROSIE_CHAT_PROVIDER_SIGNOUT=true` does that from
Rosie's "Sign out": when the provider's discovery document publishes an
`end_session_endpoint`, the browser goes there after Rosie's sign-out, with Rosie's client id
and `post_logout_redirect_uri` set to the page's address (register `<ROSIE_CHAT_PUBLIC_URL>/`
as a post-logout redirect URL at the provider). Off by default.

| Setting | Default | Meaning |
|---|---|---|
| `ROSIE_CHAT_PROVIDER_SIGNOUT` | `false` | Sign-out also signs the person out at the provider, when it publishes `end_session_endpoint` |

A person whose groups are read whole by a row policy and who is in very many groups may still
not fit in a cookie (about 4 KB). That sign-in is refused with `E_SIGNIN_FAILED` and the
reason `session_too_large` in the log: use app roles, or have the provider send only the
groups assigned to the application.

When the session ends while the page is open, the next question answers 401 and the page
goes back to the sign-in screen.

## Security

- Authorization code flow with PKCE (S256), `state` and `nonce`; the sign-in is bound to the
  browser that started it by a sealed cookie valid for 10 minutes.
- The ID token is verified by Rosie: signature against the provider's published keys
  (asymmetric algorithms only), issuer, audience, `azp` (whenever present, and required with
  several audiences),
  expiry with 30 seconds of tolerance, nonce, and the tenant at Microsoft.
- The static `ROSIE_API_TOKEN` never reaches the browser; the page carries no `api-token`.
- CSRF: the cookie is SameSite=Lax; an API call with it must come from the page's exact
  origin and carry `X-Rosie-CSRF` with a token bound to the session (the page puts it in
  `<meta name="rosie-csrf-token">`, the widget sends it on its same-origin calls); sign-out
  is a form with the same token. Anything else is 403 `E_CSRF` and a `chat_csrf_refused` security event.
- The pages carry no inline script and a strict Content-Security-Policy (`script-src 'self'`,
  `frame-ancestors 'none'`), are never cached and cannot be framed. The widget is the
  backend's own copy (`/api/assistant/widget.js`).
- Limits: sign-in steps happen before anyone is verified, so they are counted per address:
  300 a minute for `/auth/login` and `/auth/callback` (one sign-in takes 2 to 4 steps, and the
  token call only happens with the browser's own sealed cookie and state), 30 a minute for
  callbacks that fail the cookie or state check, and 30 sign-outs a minute. Questions from the
  page are counted per signed-in person (`AI_MAX_QUERIES_PER_HOUR`, default 100, and
  `AI_MAX_QUERIES_PER_DAY`, default 1,000, per person), so an office behind one address does
  not share one quota. **Behind a reverse proxy `TRUST_PROXY=1` is required**: without it every
  request has the proxy's address and the per-address limits count everyone together.
- With the page on, the API never answers without a valid session, a valid
  `ROSIE_API_TOKEN` or a valid JWT, in any environment: without a configured token or JWT,
  any request that is not the page's session (with or without an `Authorization` header) is
  401 `E_SIGNIN_REQUIRED`, also in development. Requests with a configured bearer token
  (embedded widgets, scripts) follow their own rules as before, and startup warns that only
  the page can call the API when neither is configured.

## Monitoring and audit

Every sign-in, refusal, failure and sign-out is a log event, a count in
`rosie_signin_total{outcome}`, and an audit file record of kind `rosie_signin` with the
person, the provider, the reason, the address and a correlation id. The events and their
reasons are listed in [MONITORING.md](MONITORING.md#chat-page-sign-in-events-135). With
`ROSIE_AUDIT_POLICY=fail-closed`, a sign-in that cannot be recorded is refused.

## Routes

| Route | What it does |
|---|---|
| `GET /` | The chat page when signed in; otherwise a silent sign-in, the only provider or the sign-in page ("When the sign-in screen is skipped") |
| `GET /auth/login/microsoft`, `/google`, `/sso` | Starts a sign-in; `?switch=1` asks the provider to let the person choose another account, `?silent=1` makes it silent (`prompt=none`) |
| `GET /auth/callback` | Where the provider sends the browser back |
| `POST /auth/logout` | Signs out (form with the page's token) |
| `GET /chat/page.css`, `/chat/page.js`, `/chat/rosie-still.svg` | The page's own files |

None of them exists while `ROSIE_CHAT_PAGE` is not true.

## Not built

- No local passwords and no user database; no SAML (use an OIDC bridge in your provider).
- No user management screen: who may sign in is the provider plus the two allow-lists.
- No refresh tokens, and no provider token stored or forwarded.
- One project per page, as for the widget.

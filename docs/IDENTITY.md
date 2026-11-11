# Verified identity (JWT)

By default Rosie authenticates the host application with one static token
(`ROSIE_API_TOKEN`). That proves who is calling the API and nothing about the person asking
the question. To know the person, and to filter rows by tenant or team with
[row policies](ROW_POLICIES.md), configure JWT verification: the host application sends the
user's own access token, Rosie verifies it against the identity provider's public keys, and
the token's claims become the identity of the request.

## Configure

```bash
ROSIE_JWT_ISSUER=https://login.example.com/
ROSIE_JWT_AUDIENCE=rosie
ROSIE_JWT_JWKS_URL=https://login.example.com/.well-known/jwks.json
```

| Variable | Meaning |
|---|---|
| `ROSIE_JWT_ISSUER` | Expected `iss`. Setting it turns JWT verification on. |
| `ROSIE_JWT_AUDIENCE` | Expected `aud`. Required: without it a token issued for another service would be accepted. Comma-separated for several. |
| `ROSIE_JWT_JWKS_URL` | Where the provider publishes its keys. Must be https. Keys are cached and re-fetched when a new key id appears. |
| `ROSIE_JWT_JWKS` | Alternative to the URL: the public key set itself, as JSON. For air-gapped installs. |
| `ROSIE_JWT_ALGORITHMS` | Default `RS256,ES256,PS256,EdDSA`. |
| `ROSIE_JWT_USER_CLAIM` | Claim used as the user id. Default `sub`. |
| `ROSIE_JWT_NAME_CLAIM` | Claim used as the display name. Default `name`. |
| `ROSIE_JWT_CLAIM_MAP` | JSON mapping a short name to a namespaced claim, e.g. `{"tenant_id":"https://example.com/tenant"}`. Row policies use short names. |

A configuration that sets the issuer without an audience or without keys, lists a symmetric
algorithm, points at a plain-http key URL or puts a private key in `ROSIE_JWT_JWKS` stops the
server at startup.

## What is checked on every request

- The signature, against the provider's public keys.
- `iss`, `aud` and `exp` (30 seconds of clock tolerance). A token with no `exp` is rejected.
- The algorithm, against the allowlist. `none` and the `HS*` family are never accepted: with a
  shared secret, anyone able to verify a token could also mint one. This also closes the
  algorithm-confusion attack, where a token is "signed" with the public key used as an HMAC
  secret.
- The user id claim must be present.

Verification is done by the [`jose`](https://github.com/panva/jose) library.

## How the two kinds of token coexist

| Request carries | JWT configured | Result |
|---|---|---|
| A JWT | yes | Verified. Valid: the request runs as that user, with claims. Invalid: 401. It never falls back to the static token. |
| The static token | yes or no | Accepted when it matches `ROSIE_API_TOKEN`. No claims, so row policies refuse restricted tables. |
| Nothing | JWT only, no static token | 401. |

With a verified identity, the `user` object the widget may send in the request body is
ignored: the token says who the user is. Answer handles are bound to the verified user id, so
a handle minted for one user cannot be redeemed by another, and whatever it runs is filtered
by the redeemer's own claims.

## Sending the token from the widget

Pass the user's access token as the widget's `api-token` attribute, refreshed by the host
application when it rotates. The token must be issued for the audience configured above.

## MCP over HTTP

With JWT configured, `npm run mcp:http` verifies the same tokens instead of the static
`ROSIE_MCP_HTTP_TOKEN`, the verified claims become the session identity, and
`/.well-known/oauth-protected-resource` (RFC 9728) advertises the issuer when
`ROSIE_MCP_HTTP_PUBLIC_URL` is set. Set `ROSIE_MCP_HTTP_REQUIRED_SCOPE` to require a scope the
provider grants; by default any valid token for the audience may ask. The stdio transport has
no network identity and therefore no claims.

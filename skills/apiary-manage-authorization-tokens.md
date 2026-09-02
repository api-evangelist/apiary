---
name: apiary-manage-authorization-tokens
description: Mint, list, regenerate and revoke Apiary authorization tokens over the API using HTTP Basic account credentials.
api: apiary:apiary-api
operations:
  - createAuthorizationToken
  - listAuthorizationTokens
  - deleteAuthorizationToken
generated: '2026-09-02'
method: generated
source: openapi/apiary-apiary-api-openapi.yml
---

# Manage Apiary authorization tokens

## Read this first

These three operations authenticate with **HTTP Basic using the account email and
password**, not with a token. That means an agent running this skill holds the
account credentials, not a scoped credential — which is a materially different trust
level from every other Apiary skill. Prefer the web UI at
<https://login.apiary.io/tokens> unless token rotation genuinely has to be automated.

Two hard constraints from Apiary's own contract:

- **This resource does not work for users who are part of IDCS-controlled teams**
  (Oracle Identity Cloud Service). Those accounts must use the web UI.
- **The token's description is its identifier.** Maximum 30 characters, unique per
  account, and `deleteAuthorizationToken` addresses a token by it. Choose descriptions
  a program can generate deterministically.

## Mint a token — `createAuthorizationToken`

```
POST https://api.apiary.io/authorization
Authorization: Basic <base64(email:password)>
Content-Type: application/x-www-form-urlencoded

tokenDescription=ci-publish-bot&tokenRegenerate=false
```

`201` returns the only copy of the secret you will ever get:

```json
{ "token": "…", "tokenDescription": "ci-publish-bot", "tokenUrl": "https://api.apiary.io/authorization/ci-publish-bot" }
```

Store it immediately. `listAuthorizationTokens` never returns token values.

Failures are `400` with one of:
- `Token Description Missing` — you omitted the field.
- `Token Description Length Greater Than 30` — shorten it.
- `Token Description Already Exists` — see rotation below.

## List tokens — `listAuthorizationTokens`

```
GET https://api.apiary.io/authorization
Authorization: Basic <base64(email:password)>
```

`200` returns `{"tokens": [{"tokenDescription": "…", "tokenUrl": "…"}]}` — descriptions
and URLs only, deliberately. No pagination, no creation date, no last-used date, so
you cannot tell a live token from an abandoned one from this response.

## Rotate a token

There is no separate rotate operation. Re-post the same description with
`tokenRegenerate=true`:

```
POST https://api.apiary.io/authorization
Authorization: Basic <base64(email:password)>
Content-Type: application/x-www-form-urlencoded

tokenDescription=ci-publish-bot&tokenRegenerate=true
```

`201` returns the **new** value. The old value stops working at that moment — there is
no overlap window and no grace period. Deploy the new value before you rotate, or
accept the gap.

## Revoke — `deleteAuthorizationToken`

```
DELETE https://api.apiary.io/authorization
Authorization: Basic <base64(email:password)>
Content-Type: application/x-www-form-urlencoded

tokenDescription=ci-publish-bot
```

`204` no content. This is the one clean reversal in the Apiary API: it exactly undoes
`createAuthorizationToken`, at any time, with no stated window.

The token may also be addressed as a path segment with the description
percent-encoded, e.g. `DELETE /authorization/ci-publish-bot`.

## Errors

All three operations return `{"error": "<string>"}`:

- `401 Unauthorized` — Basic credentials missing or rejected. If the account is in an
  IDCS-controlled team this will not succeed no matter what you send.
- `403 Transport Layer Security Required` — you used http.
- `400 Token Description …` — as above.

Apiary's published enum also contains `Token Creation Failed`, `Token Deletion Failed`
and `Token Retrieval Failed`, but binds none of them to an HTTP status, so treat any
unexpected status carrying one of those strings as a server-side failure and retry
with backoff.

## What a token can do

Everything. Tokens carry no scopes and are account-wide: any token can fetch and
publish every API Project the account owns. Apiary says so directly — *"a token is
like a password."* There is no read-only token, so a bot that only needs
`fetchBlueprint` still holds publish rights over the whole account.

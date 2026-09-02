---
name: apiary-fetch-api-description
description: Read the published API description document (API Blueprint or Swagger/OpenAPI) for an Apiary API Project, discovering the project subdomain first if you only know the project name.
api: apiary:apiary-api
operations:
  - getMe
  - listMyApis
  - listTeamApis
  - fetchBlueprint
generated: '2026-09-02'
method: generated
source: openapi/apiary-apiary-api-openapi.yml
---

# Fetch an Apiary API description document

Read-only. This is the safe half of the Apiary API and the one worth automating.

## Before you start

You need a personal token from <https://login.apiary.io/tokens>. It is unscoped and
account-wide — it can publish over every API Project the account owns, so treat it
as a password.

Base URL is `https://api.apiary.io`. TLS is mandatory; a plaintext request returns
403 `{"error": "Transport Layer Security Required"}`.

**The header name changes between steps.** Steps 1–3 use
`Authorization: Bearer <token>`. Step 4 uses the legacy `Authentication: Token <token>`
header — a different header, not a different value. Setting one Authorization header
globally will fail step 4 with no useful message.

## Steps

### 1. Identify yourself and your teams — `getMe`

```
GET https://api.apiary.io/me
Authorization: Bearer <token>
```

Returns `userId`, `userName`, `userApisUrl` and `teams[]`, each with `teamId`,
`teamName` and `teamApisUrl`. Keep the `teamId` values; there is no other operation
that lists teams.

### 2. List the projects you can reach — `listMyApis`

```
GET https://api.apiary.io/me/apis
Authorization: Bearer <token>
```

Returns `apis[]` with `apiName`, `apiSubdomain`, `apiDocumentationUrl` and four
booleans (`apiIsPrivate`, `apiIsPublic`, `apiIsTeam`, `apiIsPersonal`). Read one of
each pair; they are complements.

There is **no pagination** — no limit, offset or cursor parameter, and no Link
header. You get the whole array in one response.

### 3. If you need a specific team's projects — `listTeamApis`

```
GET https://api.apiary.io/me/teams/{teamId}/apis
Authorization: Bearer <token>
```

An unknown or inaccessible `teamId` returns 404 `{"error": "Team ID Invalid"}`.

### 4. Fetch the document — `fetchBlueprint`

```
GET https://api.apiary.io/blueprint/get/{apiSubdomain}
Authentication: Token <token>
Content-Type: application/json; charset=utf-8
```

Note the header. Success is 200 with:

```json
{ "error": false, "message": "", "code": "FORMAT: X-1A\nHOST: http://api.example.com/\n\n# Example API\n\nIntroduction." }
```

The document source is in `code`. It may be API Blueprint or Swagger/OpenAPI — the
project decides, and this response does not tell you which. Sniff it: an API
Blueprint document opens with a `FORMAT:` line, a Swagger/OpenAPI document parses as
YAML or JSON with a `swagger` or `openapi` key.

## Errors you will actually hit

On steps 1–3 (`{"error": "<string>"}`):

- `401 Token Invalid` — also carries `WWW-Authenticate: Bearer error="invalid_token"`. Mint a new token.
- `403 Transport Layer Security Required` — you used http.
- `404 Team ID Invalid` — step 3 only.

On step 4 the envelope **changes shape** to `{"error": <boolean>, "message": "<text>"}`:

- `500 {"error": true, "message": "Internal Error."}`
- `503 {"error": true, "message": "Infrastructure problem; please retry in a while."}`

Do not write a single error handler that reads `body.error` as a message — on step 4
it is a boolean. See `errors/apiary-problem-types.yml`.

## Retries

No `Retry-After` header is published. On a 503, back off and retry — this operation
is a read, so retrying is safe. Retrying `publishBlueprint` is not; see the publish
skill.

## Cost of a full walk

`2 + T + P` calls for T teams and P projects. There is no bulk read and no field
expansion. Cache aggressively — the underlying documents change rarely.

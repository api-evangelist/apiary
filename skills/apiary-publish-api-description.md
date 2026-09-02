---
name: apiary-publish-api-description
description: Publish an updated API description document to an existing Apiary API Project safely — take a backup first, because there is no server-side undo.
api: apiary:apiary-api
operations:
  - listMyApis
  - fetchBlueprint
  - publishBlueprint
generated: '2026-09-02'
method: generated
source: openapi/apiary-apiary-api-openapi.yml
---

# Publish to an Apiary API Project

**This is a destructive write.** `publishBlueprint` replaces the currently published
revision of a customer's documentation and mock server. There is no restore operation
in the Apiary API and no idempotency key. Do not skip step 1.

## 1. Back up the current revision first — MANDATORY

```
GET https://api.apiary.io/blueprint/get/{apiSubdomain}
Authentication: Token <token>
```

Persist the `code` field somewhere you control before going further.

Apiary does keep version history — every project has an Atom feed at
`https://<apiSubdomain>.docs.apiary.io/feed` and each entry links to a diffing UI —
but the documented rollback procedure is human: *"If you want to rollback, find the
version you are looking for, copy it to the editor and save the project."* No
retention window is stated. Your own backup is the only rollback an agent can perform.

## 2. Validate locally before you publish

The Apiary API has no server-side dry-run parameter. The rehearsal step is the CLI,
which validates without contacting Apiary and without a token:

```
gem install apiaryio
apiary preview     # reads ./apiary.apib or ./swagger.yaml, validates, opens a preview
```

Publishing an invalid document is how a project ends up with broken documentation and
a broken mock server at the same time — the mock server is regenerated from whatever
you publish.

## 3. Publish — `publishBlueprint`

```
POST https://api.apiary.io/blueprint/publish/{apiSubdomain}
Authentication: Token <token>
Content-Type: application/json; charset=utf-8

{ "code": "FORMAT: X-1A\nHOST: http://api.example.com/\n\n# Example API\n\nIntroduction." }
```

Note the header is `Authentication`, not `Authorization` — Apiary calls this group
legacy. Success is `201` with an empty JSON object `{}`.

## 4. Confirm

Re-run step 1 and diff the returned `code` against what you sent. The 201 body tells
you nothing, so this is the only confirmation available. You can also watch
`https://<apiSubdomain>.docs.apiary.io/feed` for a new entry.

## Retry rules — read before you write one

`publishBlueprint` is **not idempotent** and Apiary publishes no `Idempotency-Key`
header.

- On `500 {"error": true, "message": "Internal Error."}` or
  `503 {"error": true, "message": "Infrastructure problem; please retry in a while."}`
  the outcome is genuinely ambiguous: the publish may or may not have landed.
- Do **not** blind-retry. Call `fetchBlueprint` first and compare. If your document is
  already published, stop. If it is not, and nothing else changed in the meantime,
  retry once.
- If the fetched document is neither yours nor your backup, a human or another
  process published between your attempts. Stop and escalate — retrying will silently
  overwrite them.

No `Retry-After` is published, so choose your own backoff.

## Never do this unattended

- Publishing to a project you did not enumerate from `listMyApis` in this same run.
- Publishing without the step 1 backup in hand.
- Calling `createApiProject` to "start clean" — it is not idempotent, a taken
  `desiredName` is silently reassigned to a generated `domain`, and **there is no
  delete operation in the API**. Cleanup requires a human in the web UI, and Apiary
  states that deletion "can't be undone".

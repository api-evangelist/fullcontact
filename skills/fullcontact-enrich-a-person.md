---
name: Enrich a person with FullContact
description: >-
  Resolve a person from any identifier you already hold — email, phone, social
  URL, or name plus postal address — and return a unified profile from
  FullContact's identity graph. Covers both call paths: the hosted MCP tool and
  the REST endpoint behind it.
api: openapi/fullcontact-enrich-api-openapi.yml
operations: [enrichPerson]
mcp_tools: [fullcontact_enrich_person]
generated: '2026-08-14'
method: generated
source: >-
  Grounded in mcp/fullcontact-mcp-tools.json (live tools/list),
  https://docs.fullcontact.com/docs/tool-reference-enrich_person and
  https://docs.fullcontact.com/docs/multi-field-request.
---

# Enrich a person with FullContact

## Which path to use

There are two ways to make the same call. Pick one:

| Path | Endpoint | Use when |
|---|---|---|
| MCP tool | `fullcontact_enrich_person` on `https://api.fullcontact.com/v3/mcp` | You are an MCP client. The server returns a compact, agent-shaped payload and a normalized error envelope. |
| REST | `POST https://api.fullcontact.com/v3/person.enrich` (`enrichPerson`) | You are calling HTTP directly, or you need request properties the MCP tool does not expose (`confidence`, `dataFilter`, `webhookUrl`, `permission`). |

The MCP tool is a thin wrapper over `enrichPerson`. Everything documented for the REST endpoint applies unchanged.

## Authenticate

One credential for both paths:

```
Authorization: Bearer <FULLCONTACT_API_KEY>
```

The key must have the `person.enrich` entitlement. There is no OAuth flow and no separate MCP credential. A key without the entitlement returns `403 forbidden` — that is an account-team fix, not a retry.

## Build the input

Supply at least one identifier. More identifiers means a materially higher match rate.

- `email` — plain text, or an MD5 / SHA-1 / SHA-256 hash
- `phone` — E.164, e.g. `+13035551234`
- `linkedin` — profile URL
- `twitter` — username with or without `@`, or the profile URL
- `name` — full name. **Never send `name` alone.** It is required whenever you send `location`.
- `location` — object of `addressLine1`, `city`, `region`, `regionCode`, `postalCode`, `country`. **Must** be paired with `name`.

Sending `location` without `name` returns `invalid_input`. That is the single most common validation failure on this tool.

REST-only request properties worth knowing (`POST /v3/person.enrich` body):

- `confidence` — `LOW` (60%) / `MED` (80%) / `HIGH` (95%, default) / `MAX` (98%). Raise it for accuracy, lower it for coverage.
- `dataFilter` + `dataFilterLogic` (`AND` / `OR`, default `OR`) — restrict the response to named Insights Bundles.
- `maxMaids`, `maxEmailHashes` (max 20), `hashedEmailTypeFilter` (`MD5` / `SHA1` / `SHA256`).
- `webhookUrl` — go asynchronous; see the companion skill.

## Read the response

A hit returns a Person Summary: `fullName`, `ageRange`, `gender`, `location`, `title`, `organization`, `twitter`, `linkedin`, `bio`, `avatar`, and a `details` object.

**Treat every field as optional.** `details` is entitlement-shaped, not schema-shaped — which keys appear depends on the Insights Bundles enabled on the calling account, so the same call returns different keys for different customers. Never index blindly into `details`.

If you sent only a hashed identifier, FullContact omits clear-text PII (name, plain email, phone, social handles) from the response by design. That is a privacy control, not a miss.

## Handle failures

`not_found` is **not** an error. It means the graph has no match. Log it and continue.

| Code | Retry? | What to do |
|---|---|---|
| `not_found` | no | Normal miss. Move on. |
| `invalid_input` | no | Fix the input — usually `location` without `name`. |
| `unauthorized` | no | Missing/malformed `Authorization` header. |
| `forbidden` | no | Key lacks `person.enrich`. Escalate to a human. |
| `rate_limit_exceeded` | yes | Sleep `retry_after_seconds`, then retry. |
| `upstream_unavailable` / `upstream_timeout` | yes | Back off and retry. |
| `internal_error` | no | Capture `request_id`, open a support ticket. |

On REST, the envelope is `{"status": <int>, "message": "<string>"}` — the same HTTP status can carry different meanings, so read the `message`, not just the code. On `503` honor the `Retry-After` header.

## Respect the limits

Default ceiling is 6,000 requests per minute per account. FullContact does not reject at the edge — it **delays** your request up to 1000ms to keep you inside the limit and reports the delay in `X-FullContact-RateDelay`. You only get a `429` when the needed delay would exceed 1000ms, and that `429` is itself delayed by the maximum, so an immediate retry is safe.

Ignore `X-Rate-Limit`, `X-Rate-Limit-Remaining` and `X-Rate-Limit-Reset` — FullContact documents them as static legacy values kept for old client libraries.

## Billing rule that changes agent design

Every successful call — and every `not_found` and `invalid_input` — counts as one billable `person.enrich`. There is **no idempotency key**: a retried enrichment re-bills. Deduplicate on your side before you call, and never blind-retry a non-retryable error.

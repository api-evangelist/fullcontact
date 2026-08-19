---
name: Bulk enrich with FullContact webhook callbacks
description: >-
  Run enrichment at volume without holding a connection open per record. Supply
  a webhookUrl on the request, take the 202, and reconcile the asynchronous
  POST that FullContact delivers when the lookup completes.
api: openapi/fullcontact-enrich-api-openapi.yml
operations: [enrichPerson]
generated: '2026-08-14'
method: generated
source: >-
  Grounded in https://docs.fullcontact.com/docs/webhooks,
  https://docs.fullcontact.com/docs/request-properties and
  https://docs.fullcontact.com/docs/rate-limiting.
---

# Bulk enrich with FullContact webhook callbacks

## When to use this

Use the synchronous path (see `fullcontact-enrich-a-person.md`) for one record in an interactive loop. Use this skill when you are enriching a list and do not want to hold a request open per record.

FullContact has no batch endpoint. The asynchronous path is per-request: you still call `enrichPerson` once per person, but you hand FullContact a callback URL instead of waiting for the body.

## Make the call asynchronous

Add `webhookUrl` to the request body of `POST https://api.fullcontact.com/v3/person.enrich`:

```bash
curl -X POST https://api.fullcontact.com/v3/person.enrich \
  -H "Authorization: Bearer $FULLCONTACT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"email":"person@example.com","webhookUrl":"https://yourdomain.com/fc?identifier=12345&segment=prospect"}'
```

FullContact answers immediately:

```json
{"status": 202, "message": "Queued for search. Your response will be sent to your webhook shortly at https://yourdomain.com/fc"}
```

A `202` means queued, not matched. Do not treat it as a hit.

## Correlate the callback

**There is no correlation id in the payload.** The documented pattern is to put your own identifiers in query parameters on the `webhookUrl` itself — they survive the round trip. Multiple parameters are supported:

```
https://yourdomain.com/fc?identifier=12345&segment=prospect&batch=2026-08-14
```

Encode whatever you need to reconcile against your own row. If you send the same `webhookUrl` for every record you will not be able to tell the callbacks apart.

## Receive the callback

FullContact sends an HTTP `POST` with a JSON body that mirrors the synchronous response — the same Person Summary shape, with the same entitlement-dependent `details` object.

Your endpoint must:

1. Return `200` quickly. Do the work off the request path.
2. Be idempotent. Assume a callback can arrive more than once.
3. Tolerate a miss — a completed lookup with no match is still a delivered callback.

FullContact publishes **no signing scheme or shared secret** for these callbacks, and **no documented retry or dead-letter policy**. Treat the endpoint as publicly reachable: put an unguessable path segment or your own token in the `webhookUrl` query string, and validate it on arrival.

## Track usage per batch

Add a `Reporting-Key` header to attribute the billed events to this job:

```
Reporting-Key: batch-2026-08-14
```

This is how you see, later, where a given month's billed matches came from.

## Pace the batch

The default account ceiling is 6,000 requests per minute. FullContact absorbs mild overrun by delaying requests up to 1000ms and reporting the delay in `X-FullContact-RateDelay` — so a rising `X-FullContact-RateDelay` across your batch is your early warning to slow down *before* you start collecting `429`s.

If you do get a `429`, it has already been delayed by the maximum possible latency, so you can retry immediately and safely.

Every queued record bills as one `person.enrich` — including the ones that come back with no match. Deduplicate the input list before you start; there is no idempotency key to protect you.

## Test it first

FullContact's documented local setup is `ngrok` fronting a `netcat` listener:

```bash
ngrok http 1337
while true ; do echo -e "HTTP/1.1 200 OK\n\n $(date)" | nc -l 1337; test $? -gt 128 && break ; done
```

Point `webhookUrl` at the ngrok forwarding address and send a single record before you run the list.

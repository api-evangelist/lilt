---
name: Subscribe to LILT webhooks and handle event payloads
description: >-
  Register webhook configurations for job, project and instant-translation
  events, then correctly identify and process each delivered payload.
api: openapi/lilt-openapi-original.yml
generated: '2026-07-19'
method: generated
operations:
  - webhooksCreate
  - webhooksGetMany
  - webhooksGet
  - webhooksUpdate
---

# Subscribe to LILT webhooks

Webhooks let you stop polling job, project and instant-translation state. LILT
POSTs a JSON payload to your URL when a subscribed event fires. Webhook
configuration lives under `/v3`, not `/v2`.

## Authentication

Organization API key as both HTTP Basic username and password against
`https://api.lilt.com`.

## Steps

1. **List what already exists.** Call `webhooksGetMany`
   (`GET /v3/connectors/configuration/webhooks`) before creating anything —
   there is no idempotency key, so a blind create will produce a duplicate
   subscription and duplicate deliveries. Deleted configurations are not
   returned.

2. **Create one configuration per event type.** Call `webhooksCreate`
   (`POST /v3/connectors/configuration/webhooks`) with `webhookName`,
   `webhookUrl` and `eventType` (`create_webhook_options`). Valid event types:

   | Event type | Fires when |
   |---|---|
   | `JOB_UPDATE` | a job is updated |
   | `JOB_DELIVER` | a job is delivered |
   | `PROJECT_UPDATE` | a project is updated |
   | `PROJECT_DELIVER` | a project is delivered |
   | `INSTANT_TRANSLATE_COMPLETED` | an instant file translation succeeds |
   | `INSTANT_TRANSLATE_FAILED` | an instant file translation fails |

   LILT's own recommendation: point each event type at a **distinct endpoint**.
   That removes the need to infer the event type from payload shape.

3. **Read one back or amend it.** `webhooksGet`
   (`GET /v3/connectors/configuration/webhooks/{id}`) fetches a single
   configuration. `webhooksUpdate`
   (`PUT /v3/connectors/configuration/webhooks/{id}`) is a partial update — only
   the fields you send are changed.

4. **Handle the payload.** Instant-translate events carry an explicit
   `eventType` field. Job and project events **do not** — if you multiplexed
   several event types onto one endpoint you must infer the type from shape:

   | Payload contains | Event |
   |---|---|
   | `isDelivered: 1` and `deliveredAt` | `JOB_DELIVER` |
   | `isDelivered: 0` and a job `name` | `JOB_UPDATE` |
   | a project `name` and `due`, no `isDelivered` | `PROJECT_UPDATE` |
   | only `OrganizationId` and `id` | `PROJECT_DELIVER` |

## Rules

- **There is no signature to verify.** LILT publishes no webhook signing scheme,
  no shared secret and no timestamp header. Treat the payload as unauthenticated
  input: use an unguessable callback URL, and re-read the authoritative state
  through the API (`getJob`, `monitorFileTranslation`) before acting on anything
  consequential. Never trust a webhook body alone to trigger a payment,
  deletion, or downstream publish.

- **There is no published retry or delivery guarantee.** Assume at-least-once
  delivery at best and possible loss at worst. Make your handler idempotent on
  `id`, and keep a reconciliation poll as a backstop for anything critical.

- **Payload fields are a subset.** The webhook body carries identifiers and a
  little metadata, not the full object. Fetch the entity by id when you need
  more than `id`, `name`, `due` and delivery state.

- See `asyncapi/lilt-webhooks.yml` for the full captured event catalog. LILT
  publishes no AsyncAPI document.

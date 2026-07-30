---
name: Translate a file with LILT machine translation
description: >-
  Upload a source file to LILT, start adaptive machine translation on it, poll
  until the translation completes, and download the translated result. This is
  the "instant" MT path — no human linguist stage.
api: openapi/lilt-openapi-original.yml
generated: '2026-07-19'
method: generated
operations:
  - uploadFile
  - batchTranslateFile
  - monitorFileTranslation
  - downloadFile
  - getLanguages
---

# Translate a file with LILT

## Authentication

Every call needs the organization API key, sent as **HTTP Basic with the key as
both username and password** against `https://api.lilt.com`. The `key` query
parameter also works but LILT documents it as development-only — do not use it
in production. An API key requires the Business plan and is issued in the LILT
web app under **Manage > API keys**.

## Steps

1. **Confirm the language pair is supported.** Call `getLanguages`
   (`GET /v2/languages`). It returns `source_to_target` (which target languages
   each source supports) and `code_to_name`. Language codes are IETF BCP 47.
   Fail fast here rather than uploading a file for an unsupported pair.

2. **Upload the source file.** Call `uploadFile` (`POST /v2/files`). Request
   parameters go in the **query string**, not the body. The response is a
   `SourceFile` — keep its `id`. A File at this stage is unprocessed: it is not
   attached to a project or a memory.

3. **Start machine translation.** Call `batchTranslateFile`
   (`POST /v2/translate/file`) referencing the uploaded file id. The response
   includes an `id` used to monitor and download the translation.

   To translate several files at once, pass their ids via the `fileId` query
   parameter — `?fileId=1,2,3` or repeated `?fileId=1&fileId=2`. This endpoint
   does **not** accept multiple source files in the multipart body; each file
   must already be uploaded.

4. **Poll for completion.** Call `monitorFileTranslation`
   (`GET /v2/translate/file`). At least one query filter must be provided. The
   returned `TranslationInfo` carries `status` and, on failure, `errorMsg`.
   Alternatively subscribe to the `INSTANT_TRANSLATE_COMPLETED` and
   `INSTANT_TRANSLATE_FAILED` webhooks so you do not have to poll — see
   `asyncapi/lilt-webhooks.yml`.

5. **Download the result.** Call `downloadFile` (`GET /v2/translate/files`)
   with the translation id.

## Rules

- **Respect the rate limits.** `POST /v2/translate/file` is subject to two
  independent per-organization ceilings: **300 requests/minute** and
  **2,500,000 characters/minute**. On a 429, read `X-RateLimit-Reset`, sleep
  that many seconds plus jitter, and retry — never tight-loop. Cap retries
  (LILT's own examples use five attempts). See
  `rate-limits/lilt-rate-limits.yml`.

- **Batch instead of looping.** Sending one request per file burns the request
  ceiling. Reference multiple already-uploaded file ids in a single
  `batchTranslateFile` call. Batching does not reduce character count, so spread
  very large documents across windows.

- **Writes are not idempotent.** LILT publishes no idempotency key. If a
  `uploadFile` or `batchTranslateFile` call times out, do **not** blindly
  re-send — query for the resource first, or you will create duplicates. See
  `conventions/lilt-conventions.yml`.

- **Errors are plain JSON.** The envelope is `{"message": "..."}`; there is no
  RFC 9457 problem detail. The 429 response is the exception and adds
  `{"error": "rate_limit_exceeded"}`. A `403` means the key lacks permission for
  that file; a `410` means the file was deleted and must be re-uploaded. See
  `errors/lilt-problem-types.yml`.

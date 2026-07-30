---
name: Build and query a LILT translation memory and termbase
description: >-
  Create a translation memory for a language pair, import existing TM and
  termbase files, add and correct segments, query for leverage matches, and
  export the termbase.
api: openapi/lilt-openapi-original.yml
generated: '2026-07-19'
method: generated
operations:
  - createMemory
  - getMemory
  - updateMemory
  - importMemoryFile
  - queryMemory
  - createSegment
  - getSegment
  - updateSegment
  - deleteSegmentFromMemory
  - exportTermbase
  - downloadTermbase
---

# Build and query a LILT translation memory

A **Memory** collects source/target sentences for one language pair (for example
English→French). Its data trains the MT system, populates the translation memory
and updates the lexicon. Memories are private to the account — data is not
shared across organizations.

## Authentication

Organization API key as both HTTP Basic username and password against
`https://api.lilt.com`.

## Steps

1. **Create the memory.** Call `createMemory` (`POST /v2/memories`) with the
   source and target language. Keep the returned `id`. The `Memory` object
   carries `srclang`/`trglang`, `srclocale`/`trglocale`, `is_processing` and a
   `version`.

2. **Import existing assets.** Call `importMemoryFile`
   (`POST /v2/memories/import`). Request parameters are passed as a **JSON
   object in the `LILT-API` header**. Supported formats:
   - TM data: `*.tmx`, `*.sdltm`, `*.sdlxliff` (with custom filters), `*.xliff`,
     `*.tmq`
   - Termbase data: `*.csv`, `*.tbx`

   Import is asynchronous — poll `getMemory` (`GET /v2/memories`) and wait for
   `is_processing` to clear before relying on the contents.

3. **Add or correct segments.** `createSegment` (`POST /v2/segments`) adds a
   source/target pair to a Memory or a Document. `updateSegment`
   (`PUT /v2/segments`) rewrites the target and updates the Memory with it.
   `getSegment` (`GET /v2/segments`) reads one back.

4. **Query for leverage.** Call `queryMemory` (`GET /v2/memories/query`) to run
   a translation-memory lookup. Results are `TranslationMemoryEntry` objects
   with `source`, `target`, `score` and `metadata` — `score` is the match
   quality you use to decide exact vs fuzzy vs new.

5. **Export the termbase.** Call `exportTermbase`
   (`POST /v2/memories/termbase/export`) to generate the CSV, then
   `downloadTermbase` (`GET /v2/memories/termbase/download`) to retrieve it.
   Export before download, in that order.

## Rules

- **Deletion is destructive and asymmetric.** `deleteSegment`
  (`DELETE /v2/segments`) removes a segment from *memory* but **not** from a
  document. `deleteSegmentFromMemory` (`DELETE /v2/memories/segment`) is the
  memory-scoped delete. `deleteMemory` (`DELETE /v2/memories`) destroys the
  memory outright. Require explicit human confirmation before any of these.

- **A 401 or 404 on a Memory usually means permissions, not a bad id.** The
  docs call this out directly: a Memory created via the web app under a
  different account must be explicitly shared with the API user.

- **No idempotency key.** A retried `createMemory` or `createSegment` creates a
  duplicate. Read before re-writing.

- **Pagination is offset-based.** Where a list endpoint paginates it uses
  `start` (default 0) and `limit`; there is no cursor and no `has_more`
  envelope. See `conventions/lilt-conventions.yml`.

- **The `n` parameter on the translate endpoints is deprecated.** Do not use
  "return top n translations" in new integrations — see
  `lifecycle/lilt-lifecycle.yml`.

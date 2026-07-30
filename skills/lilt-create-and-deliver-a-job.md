---
name: Create, track and deliver a verified translation job
description: >-
  Run the full human-verified localization flow — pick a workflow template,
  create a job, add documents, pretranslate from translation memory, watch
  leverage stats, then export, download and deliver the job.
api: openapi/lilt-openapi-original.yml
generated: '2026-07-19'
method: generated
operations:
  - getWorkflowTemplates
  - createJob
  - createProject
  - uploadDocument
  - pretranslateDocuments
  - getJob
  - getJobLeverageStats
  - exportJob
  - downloadJob
  - deliverJob
---

# Create, track and deliver a LILT job

A **Job** is a collection of Projects. LILT fans a job out into one Project per
language pair, and each Project is bound to exactly one Memory. Projects hold
Documents; Documents hold Segments. See `data-model/lilt-data-model.yml`.

## Authentication

Organization API key as both HTTP Basic username and password against
`https://api.lilt.com`. Business plan required.

## Steps

1. **Get the workflow templates.** Call `getWorkflowTemplates`
   (`GET /v2/workflows/templates`). This returns the workflow templates owned by
   the team with their `id`s. You need the id to bind a language pair to a
   specific workflow when creating the job. Do not hardcode a template id —
   template ids are per-team.

2. **Create the job.** Call `createJob` (`POST /v2/jobs`). The job carries its
   language pairs; each `LanguagePair` accepts `trgLang`/`trgLocale`,
   `dueDate`, `memoryId`, `pretranslate`, `autoAccept`, `workflowTemplateId`
   and `workflowStageAssignments`. Since March 2026 the API supports the full
   set of job-submission fields including job-level instructions and custom
   metadata. Keep the returned job `id`.

3. **Create projects if you are managing them explicitly.** `createProject`
   (`POST /v2/projects`) creates a Project under the job for one language pair,
   associated with one Memory.

4. **Add documents.** Call `uploadDocument` (`POST /v2/documents/files`).
   Request parameters are passed as a **JSON object in the `LILT-API` request
   header** — not as a JSON body. For large files use the S3-backed upload lane
   instead (`initiateS3Upload`, or `initiateMultipartUpload` →
   `signUploadPart` → `completeMultipartUpload`).

5. **Pretranslate from memory.** Call `pretranslateDocuments`
   (`POST /v2/documents/pretranslate`) with an **array of document ids** in one
   request body. Only documents that are not currently importing/exporting and
   not already pretranslating are processed; the response always reflects the
   current state of every requested document, so read it rather than assuming
   success.

6. **Track progress.** Call `getJob` (`GET /v2/jobs/{jobId}`) for job data plus
   stats, and `getJobLeverageStats` (`GET /v2/jobs/{jobId}/stats`) for the TM
   leverage breakdown (`sourceWords`, `exactWords`, `fuzzyWords`, `newWords`) —
   this is what tells you how much of the job is new work versus memory reuse.
   You can also subscribe to the `JOB_UPDATE` webhook instead of polling.

7. **Export, then download.** Call `exportJob` (`GET /v2/jobs/{jobId}/export`)
   to prepare the files — use `?type=files` to export the translated documents.
   Only then call `downloadJob` (`GET /v2/jobs/{jobId}/download`). Downloading
   before a successful export of the same job id will fail.

8. **Deliver.** Call `deliverJob` (`POST /v2/jobs/{jobId}/deliver`) to set the
   job to delivered and all its projects to done. This fires the `JOB_DELIVER`
   webhook.

## Rules

- **Export before download.** `downloadJob` requires a prior `exportJob` on the
  same job id. Likewise `downloadDocument` returns **502 "File in
  pretranslation"** if an export or pretranslation is still running — poll the
  document's `export_in_progress` / `is_pretranslating` flags first.

- **Pretranslation is rate limited.** `POST /v2/documents/pretranslate` is
  capped at 300 requests/minute and 2,500,000 characters/minute per
  organization. Batch document ids into one call rather than looping per
  document. On 429, honour `X-RateLimit-Reset` with jitter.

- **Lifecycle is not reversible the way you might assume.** `deleteJob` deletes
  every project and document in the job **and** all segments from the job's
  translation memories. Prefer `archiveJob` (reversible via `unarchiveJob`) or
  `reactivateJob` after delivery. Treat `deleteJob` and `deleteMemory` as
  destructive and require explicit human confirmation before calling them.

- **Writes are not idempotent.** There is no idempotency key. A timed-out
  `createJob` may have succeeded — call `retrieveAllJobs` and match on name
  before retrying.

- **Memory permissions are a common 401/404 cause.** A Memory created in the web
  app under a different account must be explicitly shared before the API key can
  read it.

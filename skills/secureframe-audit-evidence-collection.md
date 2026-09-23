---
name: secureframe-audit-evidence-collection
description: Attach an evidence file to a Secureframe compliance test, from staging the upload through confirming the test's state. Use when an auditor or agent needs to put a document on the record for a control.
api: Secureframe Public API
base_url: https://api.secureframe.com
operations:
  - companyTestsIndex
  - companyTestsShow
  - fileUploadsCreate
  - companyTestsEvidencesCreate
  - evidencesIndex
mcp_tools:
  - list_tests
  - get_test
  - create_file_upload
  - create_test_evidence
  - list_evidences
---

# Attaching evidence to a Secureframe test

Evidence is the currency of a compliance audit. This flow puts a file on a specific
test so it shows up as proof against the controls that test backs.

## 1. Find the test

`companyTestsIndex` — `GET /tests`. Narrow with `q` (Lucene syntax) and page with
`page` / `per_page`. Use `include` and `relationships` to pull the related controls
back in the same response instead of walking them afterwards.

Read the test's current state with `companyTestsShow` — `GET /tests/{id}` — before you
change anything.

## 2. Stage the upload

`fileUploadsCreate` — `POST /file_uploads`. The bytes do NOT go through this API.
Send `filename`, `byte_size` and `checksum`; you get back `url`, a `headers` object,
and an `id`.

Declare `byte_size` and `checksum` from the file you are actually about to send —
storage rejects the PUT if they disagree. Files must be 32 MB or smaller, and a larger
`byte_size` is refused here, in step 1, before you spend anything on the transfer.

## 3. PUT the bytes

Send the raw file contents to the returned `url`, replaying every entry in `headers`
unaltered. Raw bytes — not base64, not multipart, not wrapped in JSON.

**Two clocks are running, and they are not the same clock.** The `url` stops being
accepted 15 minutes after staging (`url_expires_at`); the `id` stays redeemable for an
hour (`id_expires_at`). If you stage a batch of uploads up front, every PUT must land
inside that first 15 minutes.

## 4. Attach it

`companyTestsEvidencesCreate` — `POST /tests/{test_id}/evidences`, passing the staged
`id` as `upload_id`. The bytes must already be in storage; an `upload_id` whose PUT
never happened is refused rather than attached empty.

Each `upload_id` is redeemable **once**. Attaching the same file to a second test means
staging it again from step 2.

## 5. Confirm

`evidencesIndex` — `GET /evidences` — to see the attachment on the record.

## Rules that will bite you

- **No idempotency key.** `POST /tests/{test_id}/evidences` has no `Idempotency-Key`
  header. If the call times out and you retry, you create a second evidence record.
  Re-read with `evidencesIndex` before retrying a create, not after.
- **500 requests per minute per IP address**, 429 on exhaustion — and no
  `Retry-After` or `X-RateLimit-*` header comes back, so you have no budget signal.
  Back off with jitter on 429.
- **403 is not 401.** A valid key still returns 403 when the RBAC role of the user it
  belongs to cannot see Tests. Check the role in Console -> Personnel -> Personnel
  settings -> Roles rather than regenerating the key.
- No bulk endpoint exists — one object per request.

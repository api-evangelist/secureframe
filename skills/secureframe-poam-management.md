---
name: secureframe-poam-management
description: Manage the Plan of Action & Milestones register for a NIST 800-171 / CMMC assessment in Secureframe — list, create, update and discard POA&M items. Use for Defense-tier compliance work.
api: Secureframe Public API
base_url: https://api.secureframe.com
operations:
  - poamItemsIndex
  - poamItemsShow
  - poamItemsCreate
  - poamItemsUpdate
  - poamItemsDiscard
mcp_tools:
  - list_poam_items
  - get_poam_item
  - create_poam_item
  - update_poam_item
  - discard_poam_item
---

# Working the POA&M register

A POA&M item is the NIST SP 800-171 §3.12.2 record of a known gap and the plan to close
it. In a CMMC assessment it is the artifact the assessor reads next to the SSP, so
every write here lands in something an assessor will see.

## Survey what is open

`poamItemsIndex` — `GET /poam_items`. Search parameters documented on the operation
include `discarded` (true/false) and `due_date`. Page with `page` / `per_page`; use `q`
for Lucene-syntax search and `sort` to order.

Start by listing with `discarded=false` — the default view is not guaranteed to exclude
discarded items, and a discarded item read as open will misstate the register.

## Read before you write

`poamItemsShow` — `GET /poam_items/{id}`. There is no conditional-request or ETag
support on this API, so a read-modify-write race is invisible: two agents updating the
same item both succeed and the last one wins silently.

## Create

`poamItemsCreate` — `POST /poam_items`.

**This create is not idempotent.** No `Idempotency-Key` header exists on any Secureframe
operation. If the request times out, list with a `q` matching what you were creating
before you retry — a duplicate POA&M item is a defect in an assessment record, not just
noise in a database.

## Update

`poamItemsUpdate` — `PUT /poam_items/{id}`. This is a PUT, not a PATCH: send the full
representation. PUT is naturally idempotent, so a timed-out update is safe to repeat.

## Retire an item

`poamItemsDiscard` — `PUT /poam_items/{id}/discard`.

Discard is a **soft** removal: `GET /poam_items` accepts `discarded=true`, so the item
stays retrievable afterwards. But **Secureframe publishes no un-discard operation and no
retention window.** There is no documented way to reverse a discard through the API, and
nothing states how long a discarded item is kept. Treat discard as one-way from the
API's point of view, and confirm with a human before discarding an item that an
assessment references.

## Related surfaces

The SSP side of the same assessment lives under `/ssp_reports`,
`/ssp_report_sections`, `/ssp_report_section_blocks` and
`/ssp_report_assessment_objectives`, with the duty-assignment matrix under `/ssp_duties`
and `/ssp_duty_roles`. Note that the eight `DELETE /ssp_*` operations are **hard**
deletes with no published restore path — unlike discard here.

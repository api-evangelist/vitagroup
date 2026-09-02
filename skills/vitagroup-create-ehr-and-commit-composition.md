---
generated: '2026-09-02'
method: generated
source: openapi/vitagroup-hip-ehrbase-openehr.json
name: create-ehr-and-commit-composition
description: >-
  Create an openEHR electronic health record on HIP EHRbase and commit a first clinical
  composition against an operational template, without creating a duplicate record for
  a patient who already has one.
api: vitagroup:hip-ehrbase-openehr
base_url: https://sandkiste.ehrbase.org/ehrbase
operations:
  - getEhrBySubject
  - createEhr
  - createEhrWithId
  - getTemplatesClassic
  - getWebTemplate
  - getTemplateExample
  - createComposition
  - getComposition
---

# Create an EHR and commit a composition

Every operationId below was read from `openapi/vitagroup-hip-ehrbase-openehr.json`.

## Before you start

- Authentication depends on how the operator started the server: `NONE`, HTTP Basic, or
  an OAuth 2.0 bearer JWT. The published spec declares **no** securityScheme, so you
  cannot learn this from the contract — see `authentication/vitagroup-authentication.yml`.
  The public sandbox runs with `NONE`.
- You cannot invent composition JSON. openEHR content is constrained by an
  **operational template** that must already be loaded on the server.

## Step 1 — do not create a duplicate EHR

`getEhrBySubject` — `GET /rest/openehr/v1/ehr?subject_id=...&subject_namespace=...`

An EHR is unique on the `(subject_id, subject_namespace)` pair. If you skip this and
call `createEhr` for a patient who already has a record, the server answers **409
Conflict** — documented as "Unable to create a new EHR due to a conflict with an
already existing EHR with the same subject id, namespace pair".

- `200` — reuse this `ehr_id`. Stop; do not create.
- `404` — no record for this subject. Continue.

## Step 2 — create the EHR

Two choices, and they differ in a way that matters for retries:

- `createEhr` — `POST /rest/openehr/v1/ehr`. The server assigns the `ehr_id`. **Not
  idempotent**: a retried request after a timeout creates a second EHR.
- `createEhrWithId` — `PUT /rest/openehr/v1/ehr/{ehr_id}`. You choose the UUID. This is
  the retry-safe option, and it is the closest thing this API has to an idempotency
  key. Generate a UUID yourself, keep it, and reuse it on retry.

**Prefer `createEhrWithId`.** There is no `Idempotency-Key` header on this API
(`conventions/vitagroup-conventions.yml`), so a caller-chosen identifier is the only
protection you have against a duplicate record.

Send `openEHR-AUDIT_DETAILS` with committer and change description — it is persisted
with the CONTRIBUTION and is how the change is attributed later.

## Step 3 — find the template you are writing against

`getTemplatesClassic` — `GET /rest/openehr/v1/definition/template/adl1.4`

Returns the ADL 1.4 operational templates loaded on this instance. ADL 2 templates are
at `getTemplatesNew` (`/definition/template/adl2`).

Take the `template_id` you need. If the template you want is not there, it has to be
uploaded first (`createTemplateClassic`) — that is an operator task, not something to
guess at.

## Step 4 — learn the shape before you build the payload

This is the step agents skip, and it is the reason their compositions get rejected.

- `getWebTemplate` — `GET /rest/openehr/v1/definition/template/adl1.4/{template_id}/webtemplate`
  returns the simplified data template: the flat field paths, their types, and their
  cardinalities.
- `getTemplateExample` — `GET .../{template_id}/example` returns a **valid example
  instance** for that template.

Build your composition by modifying the example. Do not assemble openEHR JSON from
scratch.

## Step 5 — commit the composition

`createComposition` — `POST /rest/openehr/v1/ehr/{ehr_id}/composition`

- `Content-Type`: canonical `application/json` or `application/xml`, or one of the
  simplified formats — `application/openehr.wt.flat.schema+json` or
  `application/openehr.wt.structured.schema+json`. The flat format is far easier to
  produce and is what most application code posts.
- `Prefer: return=representation` makes the response echo the stored composition;
  `return=minimal` returns only the `Location` and `ETag`.
- `openEHR-AUDIT_DETAILS` — attribute the commit.
- **Keep the `ETag`.** It holds the `version_uid`, and you need it for any later update
  or delete.

Responses:

- `201` — created. `Location` and `ETag` carry the new version.
- `400` — "the request URL or body could not be parsed or has invalid content". This is
  almost always template validation. Diff your payload against `getTemplateExample`.
- `404` — the EHR does not exist. You are writing to the wrong `ehr_id`.

`createComposition` is **not idempotent**. A retry after a timeout commits a second
composition. Before retrying, read back with an AQL query or `getComposition` and check
whether the first attempt landed.

## Step 6 — verify

`getComposition` — `GET /rest/openehr/v1/ehr/{ehr_id}/composition/{versioned_object_uid}`

Add `version_at_time` to read the state at a past instant.

## What you can take back

A composition delete is **logical**: it creates a new version whose lifecycle state is
deleted, and prior versions stay readable by `version_uid` and by `version_at_time`. So
a mistaken commit is recoverable in the sense that the record is auditable and the
content is still there.

The Admin API (`/rest/admin/**`) is different — those deletes are physical and there is
no reversal. Do not reach for them.

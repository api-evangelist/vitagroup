---
generated: '2026-09-02'
method: generated
source: >-
  openapi/vitagroup-hip-ehrbase-openehr.json,
  openapi/vitagroup-hip-ehrbase-enterprise.yml,
  openapi/vitagroup-hip-ehrbase-admin.json
name: update-and-reverse-safely
description: >-
  Update versioned openEHR objects on HIP EHRbase without losing a concurrent writer's
  change, and know exactly which actions can be taken back and which are terminal.
api: vitagroup:hip-ehrbase-openehr
base_url: https://sandkiste.ehrbase.org/ehrbase
operations:
  - updateComposition
  - updateEhrStatus
  - updateDirectory
  - deleteComposition
  - getComposition
  - rollback
  - mergeEhrs
  - deleteEhr
---

# Update safely, and know what you can take back

Every operationId below was read from the published vitagroup specs.

## The concurrency contract

Four operations declare an `If-Match` header: `updateComposition`, `updateEhrStatus`,
`updateDirectory`, `deleteComposition`. It is the lost-update guard, and it is the most
important safety property this API has.

1. Read the object. Keep the `ETag` — it carries the `version_uid`, in the form
   `{versioned_object_uid}::{system_id}::{version_number}`.
2. Send that value in `If-Match` on your update or delete.
3. On `412 Precondition Failed`, the header did not match the latest server-side
   version. The `412` response returns the **current** `version_uid` in `Location` and
   `ETag`.

On a `412`: re-read, re-apply your change to the current version, resubmit. **Never
strip `If-Match` to get the write through** — that is exactly the lost update the
precondition exists to stop, and in a clinical record it silently discards somebody
else's documentation.

Set `openEHR-AUDIT_DETAILS` on every write. It is persisted with the CONTRIBUTION and is
what makes the change attributable afterwards.

## What is reversible

**Composition delete — recoverable.** `deleteComposition`
(`DELETE /rest/openehr/v1/ehr/{ehr_id}/composition/{preceding_version_uid}`) is a
*logical* delete in openEHR: it creates a new version with lifecycle state "deleted".
The prior version stays readable by `version_uid` and by `version_at_time`, so the
content is still there and the deletion is auditable.

**Contribution rollback — reversible, enterprise only.** `rollback`
(`POST /plugin/transaction-management/ehr/{ehr_id}/contribution/{contribution_id}/rollback`)
reverses a whole contribution:

- Objects are rolled back serially, in inverse order, in one database transaction.
- If any object raises a 500 or 501, the entire compensation is itself rolled back.
- On success the contribution row is deleted.
- A second rollback on the same `ehr_id` blocks until the first finishes or times out.
- Tenant-bound when multi-tenancy is enabled.

**No window is published.** vitagroup documents *that* rollback works and *how*, but
states no time limit inside which it must be called. Do not assume one, and do not
assume it is unbounded either.

This operation exists only in the HIP EHRbase enterprise build. On the open-source
server there is no rollback at all.

## What is terminal

Treat all of the following as irreversible:

- **Every `/rest/admin/**` delete** — `deleteEhr`, `deleteComposition`,
  `deleteDirectory`, `deleteStoredQuery`, `deleteTemplate`, `deleteAllTemplates`. The
  Admin API sits outside the openEHR versioning model and removes data physically.
- **`mergeEhrs`** (`POST /rest/admin/ehr/merge`). It moves every contribution,
  composition and item tag from the source EHR to the target, deletes the source EHR and
  its EHR_STATUS, and discards the source folders. The provider's own documentation
  says: *"Even if unmerging data is stored for each merge, the unmerge operation is not
  possible at the moment."* An `unmerge_data` parameter exists to retain the data for a
  future unmerge, but **no unmerge endpoint is published.**

An agent should refuse to call any of these without explicit human confirmation naming
the specific `ehr_id`.

## Things that do not exist — do not look for them

- **No dry-run.** No preview or validate-only flag on any write. The nearest thing is
  template validation: an invalid composition is rejected with `400`, so malformed
  clinical data is caught — but there is no way to rehearse a write.
- **No `Idempotency-Key`.** Retry safety comes from caller-chosen identifiers
  (`createEhrWithId`) and from `If-Match`, not from a dedup window.
- **No `Sunset` or `Deprecation` headers.** Breaking changes are announced only in
  `UPDATING.md`. Release 2.30.0 changed six response shapes with no runtime signal.
- **`deleteContribution` is dead.** It is still in the Admin contract and is not marked
  `deprecated`, but it returns `501` — "Contribution delete is not supported since
  2.0.0". Use `rollback` instead.

## References

- `conventions/vitagroup-conventions.yml` — full reversibility and concurrency detail
- `errors/vitagroup-problem-types.yml` — the 412 / 409 / 422 / 501 semantics
- https://docs.ehrbase.org/docs/EHRbase/Enterprise-Features/Transaction-Compensation
- https://docs.ehrbase.org/docs/EHRbase/Enterprise-Features/Merge-EHR

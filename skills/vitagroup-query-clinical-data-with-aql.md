---
generated: '2026-09-02'
method: generated
source: openapi/vitagroup-hip-ehrbase-openehr.json
name: query-clinical-data-with-aql
description: >-
  Read clinical data out of HIP EHRbase using the Archetype Query Language — ad-hoc
  queries, stored queries, paging, and point-in-time retrieval.
api: vitagroup:hip-ehrbase-openehr
base_url: https://sandkiste.ehrbase.org/ehrbase
operations:
  - executeAdHocQuery
  - executeAdHocQuery_1
  - getStoredQueryList
  - getStoredQueryList_1
  - getStoredQueryVersion
  - executeStoredQuery
  - executeStoredQuery_1
  - putStoredQuery
  - getTemplatesClassic
  - getWebTemplate
---

# Query clinical data with AQL

HIP EHRbase has almost no per-endpoint filter parameters. There is one query language
over the whole record instead: **AQL**, the Archetype Query Language — a combination of
XPath-style archetype navigation and SQL-style projection and filtering.

Every operationId below was read from `openapi/vitagroup-hip-ehrbase-openehr.json`.

## Step 1 — check what is already stored

`getStoredQueryList` — `GET /rest/openehr/v1/definition/query`

Stored queries are named, versioned AQL registered on the server. Since EHRbase 2.30.0
each entry also carries a `q` attribute with the plain AQL text, so you can read what a
stored query actually does before running it.

If one of these answers your question, use it — a stored query is reviewed, named and
stable, and you avoid embedding query text in a client.

## Step 2 — know your archetypes before you write AQL

AQL navigates archetype paths. You cannot guess them.

- `getTemplatesClassic` — `GET /rest/openehr/v1/definition/template/adl1.4` lists the
  loaded templates.
- `getWebTemplate` — `GET .../{template_id}/webtemplate` gives you the field paths in
  that template.

Write your `CONTAINS` and `SELECT` clauses against those paths.

## Step 3 — run an ad-hoc query

`executeAdHocQuery_1` — `POST /rest/openehr/v1/query/aql`

POST is the one to use. The GET form (`executeAdHocQuery`, `q` query parameter) puts the
whole query in the URL, which breaks on any non-trivial AQL.

```
POST /rest/openehr/v1/query/aql
Content-Type: application/json

{
  "q": "SELECT c/uid/value, c/context/start_time/value FROM EHR e CONTAINS COMPOSITION c WHERE e/ehr_id/value = $ehrId",
  "query_parameters": { "ehrId": "..." },
  "offset": 0,
  "fetch": 100
}
```

Use `query_parameters` rather than string-concatenating values into the query.

`400 Bad Request` on this endpoint means the AQL did not parse or referenced a path that
does not resolve.

## Step 4 — page through results

`offset` and `fetch` are the paging controls (`fetch` is the page size). There is **no
cursor and no total count** in the response envelope, so:

- Ask for `fetch + 1` rows, or
- Keep incrementing `offset` by `fetch` until a page comes back short.

Do not assume a stable total. The record is append-only and other writers may be
committing while you page.

## Step 5 — read the past

Most read operations accept `version_at_time` (an ISO-8601 instant): `getComposition`,
`getEhrStatusVersionByTime`, `getFolderInDirectoryVersionAtTime`,
`retrieveVersionOfCompositionByTime`, `retrieveVersionOfEhrStatusByTime`.

This is the property that makes openEHR different from a normal CRUD API: history is a
first-class read axis, not an audit-log side channel. A `404` on a versioned read often
means "nothing existed at that instant", not "this object does not exist" — retry
without `version_at_time` to tell the two apart.

## Step 6 — promote a good query to a stored query

`putStoredQuery` — `PUT /rest/openehr/v1/definition/query/{qualified_query_name}/{version}`

Use a reverse-domain qualified name and an explicit version. Then call it by name with
`executeStoredQuery` / `executeStoredQuery_1`.

## Notes

- **No rate limits.** Nothing is published and no `X-RateLimit-*`, `RateLimit-*` or
  `Retry-After` header was observed on a live response, and no operation declares a
  `429`. An expensive AQL query will simply take as long as it takes — bound it with
  `fetch` yourself. See `rate-limits/vitagroup-rate-limits.yml`.
- **Errors are unstructured.** No RFC 9457 problem+json anywhere in this API; branch on
  the HTTP status. See `errors/vitagroup-problem-types.yml`.
- Reference: https://docs.ehrbase.org/docs/EHRbase/Explore/AQL/Introduction

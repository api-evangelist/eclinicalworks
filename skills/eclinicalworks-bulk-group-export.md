---
name: eClinicalWorks bulk Group $export
description: Run a SMART Backend Services population-health export against an eClinicalWorks tenant using Group/{id}/$export and consume the NDJSON output.
api: conformance/eclinicalworks-fhir-capabilitystatement.json
operations:
  - export
  - Group.search-type
generated: '2026-08-14'
method: generated
source: https://fhir.eclinicalworks.com/ecwopendev/documentation/getting-started/backend/patient-access
---

# Bulk-export a patient cohort from eClinicalWorks

`export` is the only system-level operation the eCW FHIR Facade declares
(`http://hl7.org/fhir/uv/bulkdata/OperationDefinition/Global-i-export`). `Group` supports `search-type`
only. Everything below comes from the Bulk Patient Access Specification and the Backend Authentication
page on the eClinicalWorks developer portal.

## 1. Register asymmetric credentials

Backend Services is asymmetric-only. Register a reachable JWKS URL on the eCW Dev Portal; eClinicalWorks
must be able to fetch it and it must be allow-listed. **Only RS384 is accepted.**

## 2. Get a system token

```
POST {token_endpoint}
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion=<one-time JWT signed RS384>
&scope=system/Patient.read system/Group.read
```

Take `token_endpoint` from `{base}/.well-known/smart-configuration`. Set `iss` and `sub` to your
`client_id`, `aud` to the token URL, `exp` within five minutes of `iat`, and include the matching `kid`.
Mint a fresh assertion for every token call — never reuse one.

**`system/Group.read` must be in every bulk token request.** It must NOT be included for the Backend
Single Patient API. Getting this wrong returns `invalid_grant`.

## 3. Create the cohort

The Group is created and enabled inside the EMR by the practice, then enabled for your bulk app. The
`group_id` is shown on the developer portal. You cannot create a Group over the API.

## 4. Kick off the export

```
GET {base}/Group/{group_id}/$export
Prefer: respond-async
Accept: application/fhir+json
Authorization: Bearer <access_token>
```

Optional parameters: `_type` (comma-separated resource types), `_since` (ISO instant or date),
`_outputFormat` (defaults to `application/fhir+ndjson`).

Useful shapes:

- Everything: `GET {base}/Group/{group_id}/$export`
- Clinical subset: `?_type=AllergyIntolerance,Goal,CarePlan`
- Referenced admin resources: `?_type=Practitioner,Location`
- Provenance for one type: `?_type=Practitioner,Provenance`; Provenance alone returns it for every type.

Note: querying an admin resource returns *all* data in the EMR for that resource, not just the cohort.
`CareTeam` may pull in `RelatedPerson`; `DocumentReference` always pulls in `Binary`.

An identical bulk request is rejected while one is already in progress — this is the closest thing to an
idempotency guarantee on this API. There is no idempotency key.

## 5. Poll

`202 Accepted` returns `content-location: {base}/$export-poll-location?job_id={id}` and `retry-after`
(documented example: `120`).

```
GET {base}/$export-poll-location?job_id={id}
```

- `202` + `x-progress: 6% Completed` + `retry-after: 120` — still running. Honour `retry-after`.
- `200` — done. Body carries `transactionTime`, `request`, `requiresAccessToken: true`, `output[]` and `error[]`.
- `401` — token invalid/expired, or the group id is invalid or unauthorized.
- `404` — invalid `job_id`.

Cancel with `DELETE {base}/$export-poll-location?job_id={id}`.

## 6. Download

Each `output[]` entry is `{type, url}` pointing at an NDJSON file — one JSON resource per line.
`requiresAccessToken: true` means you must present the bearer token on the file download too. Errors
arrive as `OperationOutcome` NDJSON in `error[]`.

## 7. Budget the quota

The 250 requests/minute per-practice-code limit covers `/token` as well as FHIR calls. Bulk is the right
tool precisely because it turns a population read into a handful of requests instead of thousands — do
not fan out per-patient reads across a cohort.

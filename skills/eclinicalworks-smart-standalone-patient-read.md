---
name: eClinicalWorks SMART standalone patient read
description: Authorize a patient-facing app with SMART on FHIR Standalone Launch against an eClinicalWorks or healow tenant and read the patient's clinical record.
api: conformance/eclinicalworks-fhir-capabilitystatement.json
operations:
  - Patient.search-type
  - Patient.read
  - Condition.search-type
  - AllergyIntolerance.search-type
  - MedicationRequest.search-type
  - Observation.search-type
  - Immunization.search-type
  - DocumentReference.search-type
  - Binary.read
generated: '2026-08-14'
method: generated
source: https://fhir4.eclinicalworks.com/fhir/r4/{practice_code}/metadata + https://fhir4.eclinicalworks.com/fhir/r4/{practice_code}/.well-known/smart-configuration
---

# Read a patient's record from eClinicalWorks / healow

Every step below is grounded in the live FHIR R4 CapabilityStatement and SMART configuration the eCW FHIR
Facade 1.6 serves. There is no OpenAPI for this API — `/metadata` is the contract.

## 0. Resolve the tenant

There is no global base URL. Each practice has its own:

- Provider-facing: `https://fhir4.eclinicalworks.com/fhir/r4/{practice_code}`
- Patient-facing (healow): `https://fhir4.healow.com/fhir/r4/{practice_code}`

Resolve the practice's base URL from the published endpoint directory
(`https://connect4.healow.com/apps/api/v1/fhir/activated_clinical_endpoints`, a ~15 MB FHIR Bundle of
`Organization` + `Endpoint` resources) or from `https://fhir.eclinicalworks.com/ecwopendev/external/practiceList`.
Never hard-code a base URL for more than one customer.

## 1. Read the tenant's SMART configuration

```
GET {base}/.well-known/smart-configuration
```

Take `authorization_endpoint`, `token_endpoint` and `capabilities` from the response — do not hard-code
them. Both facades currently point at `https://oauthserver.eclinicalworks.com/oauth/oauth2/{authorize,token}`,
and both advertise `launch-standalone`, `permission-v1`, `permission-v2`, `sso-openid-connect` and
`client-confidential-asymmetric`.

## 2. Authorize (Standalone Launch)

Request `launch/patient` so the patient is selected at authorization time, plus the resource scopes you
actually need. `code_challenge_methods_supported` is `["S256"]` — PKCE is required for public clients.

Example scope string (all 486 supported scopes are listed in `scopes/eclinicalworks-scopes.yml`):

```
launch/patient openid fhirUser offline_access
patient/Patient.rs patient/Condition.rs patient/AllergyIntolerance.rs
patient/MedicationRequest.rs patient/Observation.rs patient/Immunization.rs
patient/DocumentReference.rs
```

Ask for the narrowest scope set that satisfies the use case. Requesting an unsupported or malformed scope
returns `invalid_grant` (HTTP 400).

## 3. Read the record

Send `Authorization: Bearer <access_token>` on every call. Only these interactions exist — the facade
declares `read` and `search-type` and nothing else for clinical resources:

| Goal | Call | Declared search parameters |
|---|---|---|
| Patient demographics | `GET {base}/Patient/{id}` | — (`read`) |
| Find a patient | `GET {base}/Patient?...` | `_id, birthdate, family, gender, given, identifier, name, page, phone` |
| Problems / diagnoses | `GET {base}/Condition?patient={id}` | `_id, category, encounter, onset-date, page, patient` |
| Allergies | `GET {base}/AllergyIntolerance?patient={id}` | `_id, clinical-status, encounter, patient` |
| Medications | `GET {base}/MedicationRequest?patient={id}` | `_id, authoredon, encounter, intended-dispenser, intent, page, patient, status` |
| Labs / vitals | `GET {base}/Observation?patient={id}&category=...` | `_id, _since, category, code, date, encounter, identifier, page, patient, status` |
| Immunizations | `GET {base}/Immunization?patient={id}` | `_id, date, encounter, page, patient, status` |
| Documents / C-CDA | `GET {base}/DocumentReference?patient={id}` | `_id, category, contenttype, date, encounter, page, patient, type` |
| Document payload | `GET {base}/Binary/{id}` | — (`read` only, no search) |

There is no `history`, no `vread`, no `Subscription` and no `_include`. Do not attempt them.

## 4. Page results

Follow `Bundle.link[relation=next]`. Use `_count` to request smaller pages. Several resources also declare
a literal `page` search parameter. Do not construct offsets yourself.

## 5. Respect the rate limit

250 requests per minute **per practice code**, covering FHIR resource calls, `/authorize` and `/token`
together. The bucket resets on the wall-clock minute. Exhaustion returns **HTTP 429** and blocks every
request from your application for the rest of that minute. No `Retry-After` and no `RateLimit-*` headers
are returned, so back off to the next minute boundary and count locally. See
`rate-limits/eclinicalworks-rate-limits.yml`.

## 6. Handle errors

- `401` — token invalid or expired. Refresh with `offline_access`; do not retry the same token.
- `429` — rate limited, see above.
- Read/search failures come back as a FHIR `OperationOutcome` (`application/fhir+json`), not RFC 9457.
- OAuth failures use the OAuth 2.0 error envelope: `invalid_client` (401), `unsupported_grant_type`,
  `invalid_grant`, `invalid_request` (400), `access_denied` (500). Full list in
  `errors/eclinicalworks-error-codes.yml`.

## 7. Know the version floor

A tenant on an older eClinicalWorks build will answer "operation not supported for the selected scope".
USCDI v1 works on 11.52.305C / 12.0.1 / 12.0.2 / 12.0.3; USCDI v3 needs 12.0.2(04000407) or
12.0.3(04009267) and above. Treat scope support as a per-tenant capability, discovered at runtime.

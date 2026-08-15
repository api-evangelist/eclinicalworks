---
name: healow RPM vendor integration
description: Integrate a remote-patient-monitoring device vendor with healow — receive signed device orders, acknowledge them, and push FHIR R4 observations back.
api: asyncapi/eclinicalworks-healow-rpm-webhooks.yml
operations:
  - order.create
  - order.cancel
  - order.notification
  - observation.ingest
  - observation.ingest.cgm
generated: '2026-08-14'
method: generated
source: https://connect4.healow.com/apps/jsp/dev/r4/fhirRpmVendorDocumentation.jsp
---

# Integrate an RPM device vendor with healow

This is the only bidirectional surface eClinicalWorks publishes. healow calls endpoints **you** host; you
call endpoints healow hosts. Payloads are FHIR R4 on both sides.

## 1. Register on the healow Dev Portal

Register as an RPM Vendor. You must supply:

- Organization details and a primary contact (receives all status email)
- An **active NDA with healow** — the form will not advance without it
- FDA-approved device models and supported health metrics
- A public **JWKS endpoint** (used to authenticate your pushes)
- A **data pull endpoint** healow calls to retrieve RPM data

Submissions move Draft → Approval Requested → Approved/Rejected. On approval you get a **client ID**.

## 2. Authenticate your pushes to healow

```
POST https://connect4.healow.com/apps/api/v1/fhir/tracker/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion=<JWT signed RS384, kid from your JWKS>
&scope=system/Observation.create system/Device.create
```

`iss` and `sub` = your `client_id`, `aud` = the token URL, `exp` within five minutes of `iat`. The
documented response returns `expires_in: 30000`. On `invalid_client`, build a fresh assertion and retry —
never reuse an expired one.

## 3. Verify healow's calls to you

Every inbound call carries:

```
X-Client-Id: <vendorClientId>
X-Server-Signature: Base64(HMAC_SHA256(<request-body>, <vendor-client-secret>))
```

For `DELETE` (cancel) the signature is computed over an **empty** body. Reject anything that does not
verify.

## 4. Accept an order

`POST {orderApiUrl}` with a FHIR `Bundle` (`type: collection`) containing a `Patient` and a
`DeviceRequest` (`status: active`, `intent: order`). The order number lives at
`DeviceRequest.identifier[system=https://healow.com/fhir/tracker/order].value`. The device name is coded
under `https://healow.com/fhir/tracker/device_name`.

Respond with FHIR `Parameters`:

```json
{"resourceType":"Parameters","parameter":[
  {"name":"order_number","valueString":"..."},
  {"name":"status","valueCode":"received"},
  {"name":"message","valueString":"Order received for fulfillment"}]}
```

healow also accepts a `DeviceRequest`, a `Bundle` containing one, or scalar JSON order fields. It treats
`failed`, `error`, `rejected` and `exception` as unsuccessful and stores your `message` for
troubleshooting.

## 5. Handle cancellation

`DELETE {orderApiUrl}/{orderNumber}`, no body. The practice can cancel only **within 60 minutes** of
order creation; outside that window healow will not call you at all. Return `200` on success, `4xx` when
cancellation is not allowed (e.g. `order_already_shipped`), `5xx` for transient failures.

## 6. Notify order lifecycle

```
POST https://connect4.healow.com/apps/api/v1/fhir/tracker/order/notifications
Content-Type: application/fhir+json
Authorization: Bearer <access_token>
```

Include the order number so healow can reconcile. Lifecycle states: `received`, `pending`, `shipped`,
`delivered`, `exception`.

## 7. Push readings

```
POST https://connect4.healow.com/apps/api/v1/fhir/tracker/Observation
Content-Type: application/fhir+json
```

Send a `Bundle` with `type: collection` (preferred) or `transaction`; single `Observation`s without a
Bundle are still supported. Embed `Device` resources so each `Observation` has a resolvable reference.
**`Bundle.identifier` must carry your healow-issued client ID under system
`https://healow.com/fhir/tracker/vendor`** — that is what routes the readings to the right integration.
The response is an `OperationOutcome`.

Supported reading types: blood pressure, blood glucose, weight/BMI, pulse oximetry, temperature, heart
rate, steps, distance, calories, sleep, and CGM. CGM summary reports go to
`POST /apps/api/v1/fhir/tracker/Observation/cgm`.

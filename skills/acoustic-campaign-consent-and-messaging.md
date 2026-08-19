---
name: Manage consent and send a channel message in Acoustic Campaign
description: Get an OAuth token against the right regional pod, establish a contact identity by channel, read and set consent, then send an SMS or push through the channels API — respecting the ten-concurrent-request cap.
api: openapi/acoustic-campaign-rest-swagger-index.json
generated: '2026-08-13'
method: generated
source: openapi/acoustic-campaign-databases-swagger.json + openapi/acoustic-campaign-channels-swagger.json + https://developer.goacoustic.com/acoustic-campaign/reference/basics
operations:
  - 'POST /oauth/token'
  - 'PUT /databases/{databaseId}/establishidentity/{channel}-{qualifier}/{destination}'
  - 'GET /databases/{databaseId}/contactbychannel/{channel}-{qualifier}/{destination}'
  - 'GET /databases/{databaseId}/consent/{channel}-{qualifier}/{destination}'
  - 'PUT /databases/{databaseId}/consent/{channel}-{qualifier}/{destination}'
  - 'POST /databases/{databaseId}/contacts'
  - 'PATCH /databases/{databaseId}/contacts/{contactId}'
  - 'POST /channels/sms/sends'
  - 'GET /channels/sms/sends/{transactionId}/status'
  - 'POST /channels/push/sends'
  - 'GET /channels/{channel}/publishedmessages'
---

# Manage consent and send a channel message in Acoustic Campaign

Grounded in the live Swagger 1.1 service description Acoustic serves from every Campaign pod
at `/restdoc`, harvested into `openapi/acoustic-campaign-*-swagger.json`. Swagger 1.1 has no
`operationId`; it has a `nickname` per operation, which is **not unique** in this document —
`sms-to-contacts` labels four different operations and `databases-put` labels two. Bind on
`METHOD /path`, never on the nickname.

## 1. Get on the right pod

Every organization lives on exactly one pod, and the pod is part of the host name. There is no
global host and no redirect between them.

| Pod | Host |
|---|---|
| 1–5, 9 | `api-campaign-us-1` … `us-5`, `us-6`.goacoustic.com |
| 6 | `api-campaign-eu-1.goacoustic.com` |
| 7 | `api-campaign-ap-2.goacoustic.com` |
| B | `api-campaign-ap-3.goacoustic.com` |
| 8 | `api-campaign-ca-1.goacoustic.com` |

REST base is `https://<pod>/rest`. XML API is `https://<pod>/XMLAPI`. The interactive Swagger
UI for your pod is at `https://<pod>/restdoc/`.

## 2. Authenticate

```
POST https://<pod>/oauth/token
grant_type=refresh_token&client_id=…&client_secret=…&refresh_token=…
```

Then `Authorization: Bearer {access_token}` on every call.

- Token lifetime is **4 hours**; Acoustic recommends refreshing at **3 hours**.
- **Reuse the token.** Do not mint one per request — see the concurrency rule below.
- Credentials are bound to the issuing Organization only.
- There are **no scopes**. The token carries the organization's full API grant, so treat it as
  a high-privilege secret.
- The legacy `jsessionid` session model still exists and is capped at 20 active sessions per
  organization. Do not build on it.

## 3. The concurrency rule — read this before you parallelise

**Ten concurrent authenticated requests per organization.** Not per user, not per list, not
per API. The 11th in flight is **rejected outright**: Acoustic states it does not hold or
queue the request and will not retry when a thread frees.

There is no `RateLimit-*` header, no `Retry-After` and no documented status code for this, so
you cannot detect it from the response — you have to bound it yourself. Run a semaphore of at
most 10 (in practice 6–8, to leave headroom for other integrations on the same org) and
implement your own exponential backoff.

Do not embed these credentials in a mobile app distributed to end users: Acoustic calls this
out explicitly — you lose concurrency control and leak the org's credentials.

## 4. Establish identity before you touch consent

`PUT /databases/{databaseId}/establishidentity/{channel}-{qualifier}/{destination}`

`channel` is the messaging channel (e.g. SMS), `qualifier` distinguishes the provider or
program, and `destination` is the addressable value (phone number, email). This resolves or
creates the channel identity that consent and sends both key off.

Look up what already exists first with
`GET /databases/{databaseId}/contactbychannel/{channel}-{qualifier}/{destination}`.

## 5. Read consent, then set it

```
GET /databases/{databaseId}/consent/{channel}-{qualifier}/{destination}
PUT /databases/{databaseId}/consent/{channel}-{qualifier}/{destination}
```

Per-contact variants take a trailing `/{contactId}`.

**Always GET before PUT.** There is no idempotency key anywhere in the Acoustic estate and no
conditional-update token on this resource, so a retried `PUT` after an ambiguous response will
overwrite whatever landed in between. Read the current state and reconcile.

Consent is a regulated record. If you are operating under GDPR, the access and erasure paths
are first-class: `POST /databases/{databaseId}/gdpr_access`,
`POST /databases/{databaseId}/gdpr_erasure`, then poll
`GET /gdpr_jobs/{gdprJobId}/status` and read `GET /gdpr_jobs/{gdprJobId}/response`.

## 6. Send

- **SMS to contacts** — `POST /channels/sms/sends`. Poll
  `GET /channels/sms/sends/{transactionId}/status` for the outcome; the send is asynchronous
  and the POST response is a receipt, not a delivery confirmation.
- **SMS with external consent** — `POST /channels/sms/externalconsentsends`, when consent was
  captured outside Acoustic.
- **Push to contacts** — `POST /channels/push/sends`, or
  `POST /channels/push/sends/useAppFrequency` to respect the app's frequency cap.
- **Send a published message to a contact source** —
  `POST /channels/push/publishedmessages/{publishedMessageId}/sendjobs`. List what is
  publishable with `GET /channels/{channel}/publishedmessages`.
- **Inline content to a segment** — `POST /channels/push/sendjobs`.
- **Rich / in-app content** — `POST /channels/push/richcontent` then
  `GET /channels/push/richcontent/{richContentId}`; `POST /channels/push/inappcontent`.

## 7. Handle errors properly

The REST surface returns ordinary HTTP statuses. From the published response-code reference:

- `400` — malformed argument (e.g. "Argument was not a valid email").
- `404` — `Database not found` when `databaseId` is wrong, `No contact found` when
  `contactId` is wrong. **These are different failures — do not collapse them.**
- `422` — field-level errors, one per offending column, in the form
  `{columnName}: {columnValue} is not a valid …` (boolean must be Yes/No, dates must be
  `yyyy-MM-dd`, country/selection values must be in the column's allowed set).
- `500` — including the case where a locally valid email is on the system blocked list.

The XML API returns a numeric `FaultCode` from a 116-entry registry with the detail in
`FaultString`; the full catalogue is in `errors/acoustic-campaign-error-codes.yml`. Watch for
`145` (session expired/invalid) if you are on the legacy auth path.

## What this API will not tell you

- No `securityDefinitions` in the service description — the auth model above comes from the
  docs, not from the contract.
- No rate-limit or retry headers.
- No idempotency key on any write.
- No deprecation or sunset headers, and no published retirement date for the legacy session
  model it tells you to stop using.

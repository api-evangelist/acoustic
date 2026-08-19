---
name: Publish an article in Acoustic Content
description: Authenticate to a tenant, upload an image asset, create a content item against a content type, validate it, promote it and its dependencies to ready, and start a publishing job.
api: openapi/acoustic-content-openapi-original.json
generated: '2026-08-13'
method: generated
source: openapi/acoustic-content-openapi-original.json + https://developer.goacoustic.com/acoustic-content/reference/get-started
operations:
  - 'POST /login/v1/basicauth'
  - 'POST /authoring/v1/resources'
  - 'POST /authoring/v1/assets'
  - 'GET /authoring/v1/types/{id}/new-content'
  - 'POST /authoring/v1/content'
  - 'POST /authoring/v1/types/{id}/validate'
  - 'GET /authoring/v1/references/outgoing/{classification}/{id}'
  - 'POST /authoring/v1/changes/status/ready'
  - 'POST /publishing/v1/jobs'
  - 'GET /publishing/v1/jobs/current-job/status'
---

# Publish an article in Acoustic Content

> **Operation identifiers.** The Acoustic Content contract carries **no `operationId`
> on any of its 178 operations**. Every step below is therefore identified by
> `METHOD /path`, which is what actually appears in the spec. Do not invent an
> operationId; bind on method and path.

## 1. Resolve the tenant base URL

Every call is `{API URL}/{path}` where `{API URL} = https://{DomainName}/api/{ContentHubId}` —
for example `https://content-us-1.content-cms.com/api/12345678-9abc-def0-1234-56789abcdef0`.
The domain is regional (`content-us-1`, `content-eu-4`, …) and the path segment is the
tenant's Content Hub ID. There is no shared host: **never** hardcode one tenant's base URL.

If you do not have it, log in and read the `x-ibm-dx-tenant-base-url` response header.

## 2. Authenticate

`POST /login/v1/basicauth` (or `GET`) with HTTP Basic.

- Acoustic ID: `user@example.com` + password.
- API key: user id is the literal string `AcousticAPIKey`, password is the key value.

The response sets a cookie carrying `x-ibm-dx-user-auth`. Send that on every subsequent
authoring call, as the cookie or as the `x-ibm-dx-user-auth` header. Also record
`x-ibm-dx-tenant-id` / `x-cms-tenant-id` from the response headers.

Anonymous reads are possible on `/delivery/v1/*` only. Everything in this skill is authoring
and requires the token. Roles: creating assets and content needs `editor` or higher;
retiring or deleting non-draft items needs `manager` or `admin`.

## 3. Upload the image

Two shapes, both real:

- **Two-step** — `POST /authoring/v1/resources` with the binary, then
  `POST /authoring/v1/assets` with `{"resource": "<resource-id>", "path": "/…", "name": "…",
  "status": "draft"}`. The `path` **must** start with a leading slash.
- **One-step** — `POST /authoring/v1/assets` as `multipart/form-data` with a `resource` part
  (set `Content-Type` and the `filename` on the `Content-Disposition` of that part) and an
  optional `data` part carrying the JSON above.

If an asset already exists at that `path`, it is **overwritten**. Check first with
`GET /authoring/v1/assets/record?path=…` if that is not what you want.

## 4. Create the content item

1. `GET /authoring/v1/types` (or `GET /authoring/v1/types/by-path`) to find the content type.
2. `GET /authoring/v1/types/{id}/new-content` to get a **skeleton** content item for that type.
   Fill the skeleton — do not construct the element block by hand.
3. `POST /authoring/v1/content` with the filled document.

Element values are keyed by the element `key` defined on the type. Image, file and video
elements reference the asset you created in step 3.

## 5. Validate before promoting

`POST /authoring/v1/types/{id}/validate` with the content item.

Draft items are **allowed** to save with validation errors, but they cannot leave `draft`
until those errors are resolved — that is error `invalid.item.20001`. Validating here turns a
later bulk failure into an immediate, addressable one.

You can also fetch the type's JSON Schema at `GET /authoring/v1/types/{id}/schema` and
validate locally.

## 6. Resolve dependencies, then promote

**This is the step that fails if you skip it.** A draft content item that references a draft
asset cannot go `ready` alone.

1. `GET /authoring/v1/references/outgoing/content/{id}` to list what this item depends on.
2. `POST /authoring/v1/changes/status/ready` with **every** draft item in the graph:

```json
{ "ids": [ { "id": "content:<content-id>" }, { "id": "asset:<asset-id>" } ] }
```

Identifiers may be given as `{"id": "...", "classification": "content"}` or as the combined
uid `{"id": "content:..."}`. You may name the change set with `"name": "…"`.

**Success is `204` with no body.** Any other response is a partial or total failure and
carries the outcome arrays — read them, do not just check the status:

| array | meaning |
|---|---|
| `successful` | promoted |
| `skipped` | would have succeeded; blocked by another item's failure |
| `userErrors` | you must fix something |
| `genericErrors` | server-side failure, nothing for you to fix on that item |
| `missing` | dependencies you did not include in the request |
| `messages` | per-uid error object: `uid`, `id`, `classification`, `key`, `code`, `parameters` |

Handle these codes explicitly (full catalogue in `errors/acoustic-problem-types.yml`):

- `20000` `mismatched.revs` — you sent a stale `rev`. **Re-fetch the item, read `currentRev`,
  retry.** This is the closest thing Acoustic Content has to optimistic concurrency; there is
  no idempotency key, so a blind retry after a partial success is not safe.
- `20001` `invalid.item` — go back to step 5.
- `20200` / `20201` `missing.dependencies` — add the uids listed in `parameters.missing` to the
  request and retry.
- `20100` `dependencies.failed` — fix the dependency named in `parameters.dependencies` first.
- `20010` `insufficient.permissions` — the caller's role is too low.

## 7. Publish

`POST /publishing/v1/jobs` starts an asynchronous job. Poll
`GET /publishing/v1/jobs/current-job/status`. Only one current job is addressable at a time.

If `autoPublishEnabled` is set on the default site revision
(`GET /publishing/v1/site-revisions/default`), promotion to `ready` publishes on its own —
but the flag change itself takes up to 30 seconds to be picked up.

## 8. Verify on the delivery surface

`GET /delivery/v1/content/{id}` anonymously, or `GET /delivery/v1/search?q=…`. Restricted
items only appear on `/mydelivery/v1/*` with an authenticated token.

## Rules that apply to every call

- **No idempotency key exists.** Nothing in this API is safe to blind-retry after an ambiguous
  response. Re-read state (`GET` the item, check `rev`) before retrying a write.
- **Revisions are the concurrency token.** Carry `rev` on updates; a mismatch is `20000`.
- **Conditional GETs are supported** on assets, content and categories — use them.
- **No rate-limit headers.** The API returns no `RateLimit-*` or `Retry-After`. The only
  published cap on the Acoustic estate is Campaign's ten concurrent requests per organization;
  Content publishes none. Keep concurrency modest and back off on 5xx.
- **Bulk delivery reads cap at 25 ids.** `POST /delivery/v1/content/bulk_retrieve` silently
  drops ids past the 25th rather than erroring — page your own requests.

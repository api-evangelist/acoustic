---
name: Read and render Acoustic Content on the delivery surface
description: Query the anonymous delivery API for published content, resolve reference graphs into a rendering context, use the contextual filters (proximity, language, similarity), and know when to switch to the authenticated mydelivery twin.
api: openapi/acoustic-content-openapi-original.json
generated: '2026-08-13'
method: generated
source: openapi/acoustic-content-openapi-original.json
operations:
  - 'GET /delivery/v1/search'
  - 'GET /delivery/v1/contextualsearch'
  - 'GET /delivery/v1/content/{id}'
  - 'POST /delivery/v1/content/bulk_retrieve'
  - 'GET /delivery/v1/rendering/context/{id}'
  - 'GET /delivery/v1/rendering/search'
  - 'GET /delivery/v1/rendering/render/content/{id}'
  - 'GET /delivery/v1/sites/{siteId}'
  - 'GET /delivery/v1/sites/{siteId}/pages/{pageId}'
  - 'GET /delivery/v1/resources/{fileIdentifier}'
  - 'POST /mydelivery/v1/content/bulk_retrieve'
  - 'GET /mydelivery/v1/content/{id}'
---

# Read and render Acoustic Content on the delivery surface

> Bind on `METHOD /path`. This contract has **no `operationId` on any operation**.

## Delivery vs mydelivery — pick the right twin

Almost every delivery route exists twice:

| route | auth | returns |
|---|---|---|
| `/delivery/v1/…` | **anonymous** | published, unrestricted items |
| `/mydelivery/v1/…` | authenticated | the same, **plus restricted items** |

Start on `/delivery/v1/*`. Switch to `/mydelivery/v1/*` only when you need restricted content,
and then you need the `x-ibm-dx-user-auth` token from
`POST /login/v1/basicauth` (see `authentication/acoustic-authentication.yml`).

## Search

`GET /delivery/v1/search` is a Solr-style query surface: `q`, `fq`, `fl`, `rows`, `sort`.

```
q=type:article&fl=id,name,description&rows=25&sort=lastModified desc
```

`GET /delivery/v1/contextualsearch` adds three filters that the plain search does not have.
They are opt-in via `filter=`:

- `filter=accept-language` — you **must** also send the `accept-language` request header with
  an ordered list (`en,de;q=0.5`). The service walks the list and stops at the first language
  that returns hits, then reports which one it used in the `content-language` **response**
  header. Read that header; do not assume you got your first choice.
- `filter=proximity` — with `distance` (km, defaults to 5) and either an explicit
  `position=lat,lon` or, if omitted, the location read from the `X-Akamai-Edgescape` header.
- `filter=similar` — with `similar-source-id` and `similar-source-classification`. Ranks by a
  computed `score` over shared tags (AI-generated, user-added, or image metadata). The source
  item itself is **excluded** from the results. Default sort is `score desc`; you can override
  with `sort=score asc` or `sort=lastModified asc`.

Filters compose — proximity + language + similar in one query is documented and supported.

## Fetch items

- One item: `GET /delivery/v1/content/{id}`.
- Many: `POST /delivery/v1/content/bulk_retrieve` with `{"ids": [...], "fields": [...]}`.
  **Hard cap of 25 ids.** Past the 25th, the extra ids are silently ignored — no error, no
  warning. Page in batches of 25 yourself. Missing ids are skipped rather than erroring, and
  requested fields that do not exist are simply absent from the response. Omit `fields` to get
  the whole item. `ids` must not be empty.

## Rendering contexts — the part that saves you N+1 calls

`GET /delivery/v1/rendering/context/{id}` returns the delivery content structure with
**reference elements already resolved**, recursively. Use this instead of walking `reference`
elements yourself.

**Cycles are real and are handled, not rejected.** Referenced content may link back to itself
through intermediate items. When the service detects a cycle it stops substituting and emits a
marker property with key `$$CYCLE` whose value is the content ID that cycles. Your renderer
must treat `$$CYCLE` as a terminal: look that ID up in the parent hierarchy you already have,
and never follow it as if it were a new reference — that is the loop the API just protected
you from.

Related:

- `GET /delivery/v1/rendering/search` — the same query surface as delivery search, but returns
  rendering contexts. `fl` is ignored and only `classification:content` is supported.
- `GET /delivery/v1/rendering/type/{id}` — aggregated type information.
- `GET /delivery/v1/rendering/render/content/{id}` — server-side rendered output.

## Sites and pages

- `GET /delivery/v1/sites/{siteId}` (and `/delivery/v2/sites/{siteId}`) for site metadata plus
  the page hierarchy.
- `GET /delivery/v1/sites/{siteId}/pages/{pageId}` for one page.
- `GET /delivery/v1/sites/{siteId}/pages/by-parent/{parentPageId}` for direct children; pass
  the literal `@top` as the parent id to get the top-level pages.

## Binary resources

`GET /delivery/v1/resources/{fileIdentifier}`. Acoustic fronts these with Akamai Image
Manager, so transformation and optimisation happen at the edge rather than through API
parameters.

## Caching and freshness

- Conditional requests are supported on assets, content and categories — send
  `If-None-Match` and honour `304`.
- The Personalization library caches `rulesData` and `caricatureData` in browser LocalStorage
  on a **one-hour** refresh, so a newly published personalization rule can take up to an hour
  to affect a returning visitor. That is a documented property of the library, not a bug in
  your integration.

## No runtime budget signal

Acoustic Content publishes no rate limits and returns no `RateLimit-*` or `Retry-After`
headers. There is nothing to read at runtime — cache aggressively, batch with
`bulk_retrieve` (25 at a time), and prefer one `rendering/context` call over a fan-out of
per-reference fetches.

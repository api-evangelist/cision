---
name: Pull media coverage from a CisionOne Mention Stream
description: >-
  Retrieve every media mention matching a CisionOne Mention Stream over a date
  range, paging correctly within Cision's hard 5000-result ceiling and its
  10-request-per-minute budget.
api: openapi/cision-cisionone-openapi.yml
operations:
  - getStreams
  - getMentions
generated: '2026-08-13'
method: generated
source: openapi/cision-cisionone-openapi.yml
---

# Pull media coverage from a CisionOne Mention Stream

The CisionOne API is read-only and everything hangs off a **Mention Stream** — a
saved boolean search created by a human in the CisionOne product. You cannot create
one from the API. Always start by listing the streams the token can see.

Base URL: `https://api.cision.one`

## Authentication

Send the API token on the `X-Auth-Token` header on every request. The token is
minted by an account administrator in CisionOne Settings; there is no self-service
issuance.

```
X-Auth-Token: <token>
```

- `401` — token missing, malformed or invalid. Re-issue it.
- `403` — token is valid but the account is **not entitled to API access**. This is a
  commercial gate, not a permissions problem. Stop and tell the user to contact their
  Cision account team; retrying will never succeed.

## Step 1 — find the stream (`getStreams`)

```
GET /public/api/v2/streams?format=json
```

Returns an array of streams. Match on `label` and keep the integer `id`. The seven
`*Content` booleans (`onlineContent`, `printContent`, `tvContent`, `radioContent`,
`socialContent`, `podcastContent`, `magazineContent`) tell you which media classes
that stream actually covers — do not report an absence of TV coverage from a stream
whose `tvContent` is `false`.

`archived: true` streams still return but are no longer being fed.

## Step 2 — page the mentions (`getMentions`)

```
GET /public/api/v2/mentions/{streamId}
  ?filter[range][after]=2026-01-01T00:00:00Z
  &filter[range][before]=2026-03-31T23:59:59Z
  &pagination[page]=1
  &pagination[page_size]=500
  &sort[field]=timestamp
  &sort[order]=desc
  &format=json
```

Required on every call, with no defaults you can lean on:

- `filter[range][after]` and `filter[range][before]` — both required. **The span must
  not exceed 366 days.** A wider window is a `400`, not a truncated result.
- `pagination[page]` (starts at **1**) and `pagination[page_size]` (max 5000).
- `format` — required here. Omitting it is a `400`, not a JSON default.

Optional: `sort[field]` is one of `advertisement_rate`, `audience`,
`domain_authority`, `impact_score`, `sentiment`, `source.name`, `timestamp`,
`word_count`; `sort[order]` is `asc` or `desc`.

### The ceiling that will bite you

`pagination[page]` x `pagination[page_size]` **cannot exceed 5000 mentions**. Cross
it and you get a `400`. There is no cursor, no `next` link and no total count, so:

1. Page until an empty array comes back, or until you hit 5000.
2. If a stream has more than 5000 mentions in your window, **split the window into
   shorter date ranges** and pull each separately. That is the only way to get past
   the ceiling.

Use `page_size=500` and 10 pages rather than a small page size — you are budgeting
requests, not rows.

## Step 3 — respect the rate limit

10 requests per minute, keyed on **both** your IP address and your API key. Two
processes behind the same NAT share one budget.

- On `429`, back off. There is no `Retry-After` and no `RateLimit-*` header to steer
  by, so sleep at least 60 seconds and resume.
- Steady state: roughly **one request every six seconds**. Build the sleep in; do not
  discover the limit by hitting it.

## Reading a mention

Every item carries `id`, `type` (e.g. `onlineArticle`, `radioClip`), `timestamp`
(epoch milliseconds), `publishedAt`, `medium` (`Online`/`Print`/`TV`/`Radio`),
`title_summary`, `author`, `source`, `url`, `internalLink` (a deep link into the
Cision product), location fields, `languageCode`, `wordCount`, `sentiment`,
`keywordCounts`, `audience`, `advertisingValue` (AVE), `impactScore`,
`domainAuthority`, `excerpt` and a `social` object of per-network share counts.

Broadcast items additionally carry `transcript`, `archivedLink`,
`localViewershipAudience`, `nationalViewershipAudience`, `localViewershipAdValue`
and `nationalViewershipAdValue`. Do not assume these exist on online items.

Two published quirks to handle defensively:

- `url` is **absent for tweet mentions**. Fall back to `internalLink`.
- `impactScore` is typed as a `number` in the schema but the provider's own example
  returns an array of `{score, grade}`. Accept both shapes.

## Gotchas

- `404` on `getMentions` means the `streamId` is not visible to your organisation —
  re-run `getStreams`.
- `format=csv` returns the same fields as a CSV body. Choose it for bulk export; JSON
  for anything you need to parse per-field.
- Readership for online (non-broadcast, non-print) content is deliberately zeroed
  under Cision's licensing agreement with its data supplier. It is not a bug and
  there is no parameter that turns it on.

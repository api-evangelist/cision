---
name: Report earned media metrics for a CisionOne stream
description: >-
  Pull the aggregated statistics summary for a CisionOne Mention Stream — advertising
  value equivalency, audiences by media type and the sentiment histogram — and turn
  it into a period-over-period earned media report without over-reading the numbers.
api: openapi/cision-cisionone-openapi.yml
operations:
  - getStreams
  - getStreamStats
generated: '2026-08-13'
method: generated
source: openapi/cision-cisionone-openapi.yml
---

# Report earned media metrics for a CisionOne stream

Use `getStreamStats` when you want the totals. It aggregates server-side, so one
request replaces thousands of `getMentions` pages — which matters a great deal
against a 10 request/minute budget.

Base URL: `https://api.cision.one`. Auth: `X-Auth-Token` header (see
`authentication/cision-authentication.yml`).

## Step 1 — resolve the stream (`getStreams`)

```
GET /public/api/v2/streams?format=json
```

Take the integer `id` for the stream you are reporting on.

## Step 2 — get the summary (`getStreamStats`)

```
GET /public/api/v2/streams/{streamId}/stats
  ?filter[range][after]=2026-04-01T00:00:00Z
  &filter[range][before]=2026-06-30T23:59:59Z
  &format=json
```

All three query parameters are **required**. The date span must not exceed 366 days.

## Step 3 — read the response

```
streamId, streamLabel, after, before
media[]                  # the media types present, e.g. Online, Print, TV, Radio
advertisingValues[]      # per media label: {count, total, currency}
audiencesByType[]        # per media label: {label, value}
sentimentAggregation[]   # histogram buckets with {from, to, doc_count}
```

The sentiment histogram uses five named buckets: **Negative**, **Trending Negative**,
**Balanced**, **Trending Positive**, **Positive**. Report those labels as Cision
defines them; do not collapse them into a positive/negative binary, because
"Balanced" and the two "Trending" buckets are distinct editorial states in Cision's
model.

`advertisingValues[].values.count` is the number of items in that media class and
`.total` is the summed Advertising Value Equivalency. Divide only if you intend to
report an average AVE per item, and say so.

## Step 4 — period over period

To compare quarters, call `getStreamStats` once per window rather than once with a
wide window. Two calls, two summaries, subtract. Keep both windows the same length
or the totals are not comparable.

Against the rate limit this is the cheap path: a year of monthly reporting is 12
requests, versus potentially hundreds of `getMentions` pages.

## What not to claim

- **Readership / audience for online content is zeroed by contract.** Cision's
  agreement with its data supplier prevents it from returning per-item audience for
  online items via the API. `audiencesByType` for online media reflects that. Never
  report an online audience of 0 as a real measurement — say the figure is not
  available through the API.
- **AVE is a modelled estimate**, not spend. Cision documents how it is computed
  ("How Ad Value Equivalency (AVE) Works"). Label it as an equivalency figure.
- The API exposes no outlet or journalist entity — `source` is a plain string. You
  cannot join these numbers to the Cision media contacts database through the API.

## Errors

| Status | Meaning | What to do |
|---|---|---|
| 400 | Date range exceeds 366 days, or `format` missing | Narrow the window; always send `format` |
| 401 | Token missing or invalid | Re-issue from CisionOne Settings |
| 403 | Account not entitled to API access | Stop; escalate to the Cision account team |
| 404 | `streamId` not visible to this organisation | Re-run `getStreams` |
| 429 | Over 10 requests/minute (keyed on IP **and** key) | Sleep 60s; no `Retry-After` is published |

No error carries a response body schema. Branch on the status code alone.

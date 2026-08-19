---
name: Authenticate and export from the Cision Communications Cloud API
description: >-
  Exchange Cision Communications Cloud credentials for an access token, list
  searches, and export mentions and stats — including retrieving the real readership
  values that are zeroed in the response body and delivered instead via a five-minute
  presigned URL.
api: https://api.trendkite.com
operations:
  - POST /api/login
  - GET /api/v2.2/searches
  - GET /api/v2.2/totalmentions
  - GET /api/v2.2/stats
generated: '2026-08-13'
method: generated
source: https://cision.atlassian.net/wiki/spaces/CSM/pages/25764989843/Settings+-+Cision+API
---

# Authenticate and export from the Cision Communications Cloud API

This is Cision's **other** API — the Next Generation Cision Communications Cloud
surface, served on `https://api.trendkite.com`, the host Cision inherited with its
2019 acquisition of TrendKite. It is documented by Cision on Cision's own help site
and is a separate product from CisionOne. Do not mix the two: different base host,
different path prefix (`/api/v2.2`), different pagination convention, different
operations.

Cision publishes **no OpenAPI** for this surface. Everything below comes from
Cision's own developer documentation.

## Step 1 — get a token

```bash
curl -X POST -H "Content-Type: application/json" \
  -d '{"username":"<cision-username>","password":"<cision-password>"}' \
  https://api.trendkite.com/api/login
```

The JSON response carries an `access_token` string. There is no published expiry, no
refresh flow and no client registration — these are the user's platform login
credentials, so treat them as high-value secrets and never log the request body.

Every subsequent call sends the token on the `X-Auth-Token` header:

```
X-Auth-Token: <access_token>
```

CORS is supported. JSONP is explicitly not.

## Step 2 — list searches

```
GET /api/v2.2/searches
```

Optional `shared=true` also returns searches shared with the API user. Returns
`{searches: [{id, title, taxonomy}]}` where `taxonomy` is e.g. `COMPANY` or `CUSTOM`.
Keep the `id` — it is the `s` parameter everywhere else.

## Step 3 — export mentions

```
GET /api/v2.2/totalmentions
  ?s=<searchId>
  &range-start=2026-01-01T00:00:00.000-0000
  &range-end=2026-03-31T23:59:59.000-0000
  &page-num=0
  &page-size=100
  &sort=asc
  &format=json
```

- `s`, `range-start` and `range-end` are required.
- **Pagination starts at 0 here**, not 1 — the opposite of the CisionOne API. Default
  `page-num=0`, default `page-size=100`.
- `sort` is `asc` or `desc` (default descending). `format` is `csv` or `json`
  (default json).
- `ga-id` optionally joins Google Analytics data onto the mentions, if that GA account
  is attached to the authenticated user.

Each item carries `adEquivalency`, `author`, `date`, `impactScore`, `link`,
`mediaOutlet`, `mediaType`, `sentiment`, `seoImpact`, `shares`, per-network social
counts, `title`, `url`, location fields, and — when a GA integration is attached —
`sessions`, `pageViews`, `bounceRate`, `goalCompletions`, `goalValue`,
`transactionRevenue` and friends.

## Step 4 — the readership side-channel

`readership` in the JSON body is **always 0** for online articles. That is deliberate:
Cision's agreement with its data supplier prevents returning per-item online
readership in the API response.

The real values are delivered out of band. Read the **`x-additional-info` response
header**: it holds a presigned URL to a CSV containing the same rows plus the true
readership value.

```
x-additional-info: https://<bucket>.s3.amazonaws.com/<id>.csv?X-Amz-...&X-Amz-Expires=299&...
```

**The URL expires in five minutes.** So:

1. Read the header off the response — do not discard headers.
2. Download the CSV immediately, in the same execution.
3. If it expired, do not retry the download. Re-call `/api/v2.2/totalmentions` to mint
   a fresh URL — and remember that costs you another request against the rate limit.

Never persist or log that URL; it is a bearer credential for the duration.

## Step 5 — aggregate stats

```
GET /api/v2.2/stats?s=<searchId>&range-start=...&range-end=...&type=all
```

`type` is one of `all`, `ave`, `readership`, `totalMentions`, `socialShares`. The
response is an array with `status`, `searchId`, `searchName`, `startDate`, `endDate`
and a `data[]` block carrying `ave` (per media type plus `totalAVE`), `readership`
(per media type plus `totalReadership`), `totalMentions`, and a `socialShares` object
broken down by network.

Pass a narrow `type` when you only need one metric — it is the same request cost, but
a much smaller payload.

## Rate limit

10 requests per minute across all endpoints, with a `429` on exhaustion. No
`Retry-After` and no `RateLimit-*` headers. Self-throttle to one request every six
seconds. The Master Subscription Agreement makes respecting Cision's API rate limits
a contractual obligation, so this is not merely a technical courtesy.

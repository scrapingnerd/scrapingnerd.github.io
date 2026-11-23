---
layout: post
title: "One Endpoint for Every TikTok URL: Notes on Ultimate TikTok Scraper's Server Mode"
date: 2026-11-24 07:10:00 +0700
permalink: /apify/tiktok-scraper-ultimate-server-mode/
categories: [Data Engineering, Analysis]
tags: [Web Scraping, Apify, Server Actors, Standby, TikTok, Node.js, Caching]
author: Scraping Nerd
description: "How the server mode of novi/tiktok-scraper-ultimate routes any TikTok URL or keyword through GET /videos, what that design costs and saves, and a Node.js client with caching, request coalescing, 402 handling and 502 backoff."
---

In [yesterday's post on Server Actors](/apify/agent-shaped-scraping-server-actors/) I listed [Ultimate TikTok Scraper](https://apify.com/novi/tiktok-scraper-ultimate?fpr=7hce1m) as a batch-only Actor. As of today it also runs in server mode (Apify Standby). I think its API design is worth a closer look, because it handles a problem most TikTok integrations deal with badly: working out what kind of link a user just pasted.

## One route, many URL types

The server lives at `https://novi--tiktok-scraper-ultimate.apify.actor` and has a single endpoint, `GET /videos`. You pass exactly one of these:

- `url`: any tiktok.com URL the Actor supports. That covers a video, a profile, a hashtag, a music page, a search page, or home/trending. Short links work too.
- `keyword`: a search term such as `podcast`.

The other parameters:

| Param | Meaning |
|---|---|
| `limit` | 1 to 100, default 20 |
| `location` | 2-letter country, default US (IR, AQ, KP, TD unsupported) |
| `sortType` | Search and hashtag only: DEFAULT, RELEVANCE, MOST_LIKED, MOST_RECENT |
| `dateRange` | Search and hashtag only: DEFAULT, ALL_TIME, YESTERDAY, WEEK, MONTH, THREE_MONTH, SIX_MONTH |

A 200 response is always a JSON array of video objects, the same shape as the batch run's dataset items.

Most TikTok APIs give you one endpoint per resource type. The caller then has to parse the URL, decide whether it's a video, a profile or a tag, pull out the ID or handle, and call the matching route. Short links make this worse, because you can't tell what a short link points to until you follow the redirect.

Here the Actor does that classification on the server. For a Slack bot, a browser extension backend, or an agent tool described as "give me data for this TikTok link", the client shrinks to one HTTP call. The agent case benefits most. An LLM is decent at copying a URL into a parameter and much less reliable at choosing between six similar tools.

The cost of one call is harder to predict, though. One URL can cost very different amounts depending on its type. A video link returns one item. A profile or hashtag link returns up to `limit` items. Each is charged per returned item (`result-item`) plus the same per-task events a normal run uses (a single-video lookup is `single-video-query`, for example). Since the client no longer knows the URL type, it also can't predict the cost unless you set `limit` deliberately. If users paste arbitrary links, choose a `limit` that you're happy to pay for on every request, because a profile link will always come back with that many.

## Errors

| Status | Body | What to do |
|---|---|---|
| 400 | `{ "errors": [...] }` or `{ "error": "Unsupported TikTok URL..." }` | Bad input. Show it to the user and don't retry. |
| 402 | | Charge limit reached. Stop sending requests and alert someone. |
| 404 | | Unknown path. This is a bug in your client. |
| 502 | `{ "error": "Upstream request failed" }` | Retry with backoff. |

Compare this with [Advanced Search TikTok API](/apify/advanced-search-tiktok-api-server-mode/) returns 200 with `meta.chargeLimitReached` when it hits the charge limit, while this Actor fails the whole request. If one client talks to both, write the "budget exhausted" logic once and have both checks feed into it.

## Trying it with curl

URL-encode the `url` parameter. `curl -G --data-urlencode` handles that:

```bash
BASE=https://novi--tiktok-scraper-ultimate.apify.actor

# a profile's latest videos
curl -sS -G --max-time 300 -H "Authorization: Bearer $APIFY_TOKEN" "$BASE/videos" \
  --data-urlencode "url=https://www.tiktok.com/@gordonramsayofficial" \
  --data-urlencode "limit=20" | jq length

# a hashtag, most liked
curl -sS -G --max-time 300 -H "Authorization: Bearer $APIFY_TOKEN" "$BASE/videos" \
  --data-urlencode "url=https://www.tiktok.com/tag/duet" \
  --data-urlencode "sortType=MOST_LIKED" | jq '.[0]'

# a keyword, last week
curl -sS -G --max-time 300 -H "Authorization: Bearer $APIFY_TOKEN" "$BASE/videos" \
  --data-urlencode "keyword=podcast" --data-urlencode "location=US" \
  --data-urlencode "dateRange=WEEK" | jq length
```

## A Node.js client with caching and coalescing

Link-lookup traffic has a lot of repetition. The same viral video gets pasted into five Slack channels within an hour, and an agent fetches the same profile again three turns after the first time. Two cheap layers handle most of that:

1. A TTL cache keyed on the full request, so a repeat inside the window costs nothing.
2. Request coalescing (often called single-flight): if an identical request is already in progress, wait for its promise and don't start a second one. Without this, five people pasting a link in the same second still produce five paid requests, because the cache has nothing in it until the first one returns.

Node 18+ with the built-in `fetch`:

```js
// tiktokLookup.mjs
const BASE = 'https://novi--tiktok-scraper-ultimate.apify.actor';
const TOKEN = process.env.APIFY_TOKEN;
const TTL_MS = 10 * 60 * 1000;          // pick what "fresh enough" means for you

const cache = new Map();    // key -> { at, data }
const inflight = new Map(); // key -> Promise
let budgetExhausted = false;

export class ChargeLimitError extends Error {}
export class BadInputError extends Error {}

function cacheKey(params) {
  const p = { ...params };
  if (p.url) {
    const u = new URL(p.url);
    u.hostname = u.hostname.toLowerCase();
    // drop share/tracking query strings, but keep them on search pages
    if (!u.pathname.startsWith('/search')) u.search = '';
    u.hash = '';
    p.url = u.toString().replace(/\/$/, '');
  }
  return JSON.stringify(Object.keys(p).sort().map((k) => [k, p[k]]));
}

async function callOnce(params, { retries = 3 } = {}) {
  const qs = new URLSearchParams(params);
  for (let attempt = 0; ; attempt++) {
    let res;
    try {
      res = await fetch(`${BASE}/videos?${qs}`, {
        headers: { Authorization: `Bearer ${TOKEN}` },
        signal: AbortSignal.timeout(300_000),
      });
    } catch (err) {
      if (attempt >= retries) throw err;            // network error / timeout
    }
    if (res?.ok) return res.json();
    if (res?.status === 402) {
      budgetExhausted = true;
      throw new ChargeLimitError('Apify charge limit reached');
    }
    if (res?.status === 400) {
      const body = await res.json().catch(() => ({}));
      throw new BadInputError(body.error ?? JSON.stringify(body.errors));
    }
    if (res && res.status !== 502) {
      throw new Error(`Unexpected ${res.status}: ${await res.text()}`);
    }
    if (attempt >= retries) throw new Error('Upstream request failed after retries');
    const delay = 1000 * 2 ** attempt + Math.random() * 500;
    await new Promise((r) => setTimeout(r, delay));
  }
}

export async function lookup(params) {
  if (budgetExhausted) throw new ChargeLimitError('budget exhausted, not calling');
  if (!!params.url === !!params.keyword) {
    throw new BadInputError('pass exactly one of url or keyword');
  }
  const key = cacheKey(params);
  const hit = cache.get(key);
  if (hit && Date.now() - hit.at < TTL_MS) return hit.data;
  if (inflight.has(key)) return inflight.get(key);

  const p = callOnce(params)
    .then((data) => { cache.set(key, { at: Date.now(), data }); return data; })
    .finally(() => inflight.delete(key));
  inflight.set(key, p);
  return p;
}

// example
const [a, b] = await Promise.all([
  lookup({ url: 'https://www.tiktok.com/@billieeilish/video/7050551461734042926' }),
  lookup({ url: 'https://www.tiktok.com/@billieeilish/video/7050551461734042926?is_from_webapp=1' }),
]);
console.log(a.length, a === b); // second call joined the first one
```

Some notes on the code.

URL normalization is deliberately cautious. Removing the query string makes two share links for the same video hit the same cache entry. Search pages keep it, though, because the search term is in the query. Short links stay as they are. You can't know where a short link points without resolving it, and resolving it client-side would rebuild the routing the Actor already does. The result is that a short link and its full URL get separate cache entries. I'm fine with that.

Errors aren't cached. A 400 for an unsupported URL could be cached as a negative result, but I'd rather have users see the message each time than cache a mistake.

`budgetExhausted` acts as a process-wide circuit breaker. After a 402, every later request would fail the same way, so the client stops making them. In a real service you'd reset it on a timer or when an operator raises the limit, and you'd keep the flag somewhere shared if more than one process is running.

The timeout is 300 s, matching Apify's limit for a server-mode request. The first request after an idle stretch can be slower because Apify has to start a container (containers stay warm for about 300 s of idle time). If your users notice the delay, measure it: log the latency of the first call after an idle period separately from warm calls and see whether the gap matters for your use.

## When to use a batch run

Server mode keeps each request small on purpose. A server-mode request has to finish within about 5 minutes, which is why `limit` stops at 100 and why there's one `url` per request. Switch to the normal input run when you need:

- more than 100 items for a profile, hashtag or search,
- many URLs in a single job (you could fan out requests, but a batch run gives you one dataset and one place where failures show up),
- `customMapFunction`, which server mode doesn't support on this Actor,
- a stored dataset as the record of what you collected.

So I use server mode for "someone pasted a link, show them the data" and the batch run for "collect everything for these 200 accounts every night". The OpenAPI description for `/videos` is on the Actor's **Server** tab in Apify Console, and pricing is on the [Store page](https://apify.com/novi/tiktok-scraper-ultimate?fpr=7hce1m).

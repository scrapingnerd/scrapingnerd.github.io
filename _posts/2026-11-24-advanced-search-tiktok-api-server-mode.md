---
layout: post
title: "Advanced Search TikTok API in Server Mode: Partial Results, Fan-Out and Payload Shaping"
date: 2026-11-24 07:00:00 +0700
permalink: /apify/advanced-search-tiktok-api-server-mode/
categories: [Data Engineering, Analysis]
tags: [Web Scraping, Apify, Server Actors, Standby, TikTok, Python, asyncio]
author: Scraping Nerd
description: "A data engineer's look at the server mode of novi/advanced-search-tiktok-api: why each request is capped, how meta.partial and meta.chargeLimitReached should change your client, and how to fan out many keywords without losing track of what came back."
---

Yesterday I wrote that [none of the TikTok Actors I use run as servers](/apify/agent-shaped-scraping-server-actors/). That's no longer true: three novi TikTok Actors got server mode, and two of them were on yesterday's list. This post covers the one I use most for keyword work, [Advanced Search TikTok API](https://apify.com/novi/advanced-search-tiktok-api?fpr=7hce1m), which now has a server mode (Apify Standby, which Apify now calls Server Actors).

I'm not going to repeat the getting-started material. I care about the contract: what one request can and can't promise, what the response tells you when it falls short, and how to build a client that works within those limits.

## What changes compared with a batch run

In a normal run you send input, the Actor runs until it's done, and the videos land in a dataset you read later. In server mode the Actor keeps an HTTP server running at `https://novi--advanced-search-tiktok-api.apify.actor`. You send a request with your Apify token and the videos come back in the response body. You don't start a run and you don't poll anything.

That convenience comes with three costs.

1. Nothing is stored. Results exist only in the HTTP response. If you want history, diffs or an audit trail, your code has to write them somewhere.
2. There's a deadline. Apify requires a server-mode request to finish within about 5 minutes (300 s). The Actor sets its own deadline lower, at about 3.5 minutes, so it can still return what it has before the platform cuts the connection.
3. Cold starts. The first request after a quiet period can be slower because Apify has to start a container. Containers stay warm for about 300 s of idle time. If you want a number for your own traffic, time the first request after ten idle minutes and then a few back-to-back requests, and compare the medians and the worst cases.

The batch run still works the same as before. For anything large or scheduled, it's still what I'd use.

## The endpoint

`GET /` returns usage info. The real work happens at `/search`, which accepts both GET and POST.

GET takes query parameters:

| Param | Meaning |
|---|---|
| `keyword` | Repeatable: `keyword=nike&keyword=adidas` |
| `keywords` | Comma-separated alternative |
| `limit` | 30 to 500 videos per keyword, default 30 |
| `region` | 2-letter code, default GB (IR, AQ, KP unsupported) |
| `sortType` | 0 relevance, 1 most liked, 2 most recent; default 0 |
| `publishTime` | ALL_TIME, YESTERDAY, WEEK, MONTH, THREE_MONTH, SIX_MONTH; default ALL_TIME |
| `includeSearchTerms` | Adds a `__search_terms` object to each video: `keyword`, `sort_type`, `publish_time`, `region` |
| `fetchUserBio` | Adds `author.bio`; slower, and charged extra as `user-bio-lookup` per unique author |

POST takes a JSON body with the same fields as the Actor input (`keywords` as an array, max 10), plus one field GET doesn't have: `customMapFunction`.

The limits per request: at most 10 keywords, at most 500 videos per keyword, up to 4 keywords fetched in parallel, and no `isUnlimited`. Every one of these follows from the 5-minute deadline. If one keyword can take a while to page through, 10 keywords at 500 each with 4 running at a time may not fit in 3.5 minutes. That's why the response has a `meta` block.

## Reading `meta`

A 200 response looks like this:

```json
{
  "items": [ ... ],
  "meta": {
    "keywords": [{ "keyword": "nike", "count": 100 }],
    "limitPerKeyword": 100,
    "partial": false,
    "timedOut": false,
    "chargeLimitReached": false
  }
}
```

The two flags call for different handling, and treating a 200 as plain success throws that information away.

`partial: true, timedOut: true` means the Actor hit its internal deadline and returned what it had. The items are real and you've paid for them, so keep them. Then compare `meta.keywords[].count` against `limitPerKeyword` to see which keywords fell short and send those again in a smaller request. One caveat: a count below the limit can also mean TikTok had no more results for that keyword and time window. If a keyword comes up short again on its own, I treat that as its real total.

`chargeLimitReached: true` means you hit your max charge limit. This Actor still returns 200 with partial results. The other two TikTok Actors that got server mode today return 402 in the same situation. Retrying makes no sense here. Stop sending new requests until someone raises the limit, and keep what arrived.

The errors are simple. A 400 with `{ "error": "..." }` means bad input (too many keywords, invalid JSON) and should never be retried. A 500 with `{ "error": "Failed to retrieve search list." }` is worth a couple of retries with backoff.

## A quick curl check

```bash
curl -sS --max-time 300 \
  -H "Authorization: Bearer $APIFY_TOKEN" \
  "https://novi--advanced-search-tiktok-api.apify.actor/search?keyword=nike&keyword=adidas&limit=30&region=US&sortType=2&publishTime=WEEK" \
  | jq '.meta'
```

Put the token in the header. The `?token=` query parameter works too, but URLs end up in proxy logs and shell history.

## Making the payload smaller with `customMapFunction`

The items are raw TikTok "aweme" objects, the same as dataset items. They're large: play addresses, music, author info and statistics. If an agent only needs an ID, the caption and a few counters, sending all of that through a tool call wastes bandwidth and tokens. `customMapFunction` (POST only) runs a JS function on each video before it's returned:

```bash
curl -sS --max-time 300 -X POST \
  -H "Authorization: Bearer $APIFY_TOKEN" \
  -H "Content-Type: application/json" \
  "https://novi--advanced-search-tiktok-api.apify.actor/search" \
  -d '{
    "keywords": ["running shoes", "marathon training"],
    "limit": 30,
    "region": "US",
    "sortType": 2,
    "publishTime": "WEEK",
    "customMapFunction": "(o) => ({ id: o.aweme_id, desc: o.desc, created: o.create_time, author: o.author?.unique_id, plays: o.statistics?.play_count, likes: o.statistics?.digg_count })"
  }'
```

Use it to reshape items. It doesn't filter. You're charged per returned item, and mapping doesn't change how many items come back, so it won't lower the count. To compare payload sizes yourself, run the same request with and without the function and pipe both into `wc -c`.

## Fanning out many keywords

Monitoring jobs often track more than 10 keywords. A client has to:

- split the keyword list into chunks of 10 or fewer,
- run a few chunks at once, but not too many (each request is already fetching up to 4 keywords in parallel on the server),
- retry 5xx with backoff, and never retry 400,
- re-queue the short keywords from a timed-out chunk on their own,
- stop starting new chunks once any response reports `chargeLimitReached`.

Here's that client in Python with `asyncio` and `httpx`:

```python
import asyncio, os, random
import httpx

BASE = "https://novi--advanced-search-tiktok-api.apify.actor"
TOKEN = os.environ["APIFY_TOKEN"]

def slim(o: dict) -> dict:
    # trim client-side so the __search_terms object stays available
    return {"id": o.get("aweme_id"), "desc": o.get("desc"),
            "created": o.get("create_time"),
            "author": (o.get("author") or {}).get("unique_id"),
            "plays": (o.get("statistics") or {}).get("play_count"),
            "keyword": (o.get("__search_terms") or {}).get("keyword")}

class ChargeLimit(Exception):
    pass

async def search(client, keywords, limit, stop: asyncio.Event, retries=3):
    body = {"keywords": keywords, "limit": limit, "region": "US",
            "sortType": 2, "publishTime": "WEEK",
            "includeSearchTerms": True}
    for attempt in range(retries + 1):
        if stop.is_set():
            raise ChargeLimit()
        try:
            r = await client.post(f"{BASE}/search", json=body)
        except httpx.TimeoutException:
            r = None
        if r is not None and r.status_code == 400:
            raise ValueError(r.json().get("error"))      # bad input: don't retry
        if r is not None and r.status_code == 200:
            return r.json()
        if attempt == retries:
            if r is not None:
                r.raise_for_status()
            raise RuntimeError("request timed out")
        await asyncio.sleep(2 ** attempt + random.random())  # 500/502/timeout

async def run(all_keywords, limit=100, concurrency=3):
    stop = asyncio.Event()
    sem = asyncio.Semaphore(concurrency)
    results, shortfalls = [], []

    async def worker(chunk):
        async with sem:
            data = await search(client, chunk, limit, stop)
        meta = data["meta"]
        results.extend(slim(o) for o in data["items"])
        if meta.get("chargeLimitReached"):
            stop.set()
        if meta.get("timedOut"):
            for k in meta["keywords"]:
                if k["count"] < meta["limitPerKeyword"]:
                    shortfalls.append(k["keyword"])

    chunks = [all_keywords[i:i + 10] for i in range(0, len(all_keywords), 10)]
    timeout = httpx.Timeout(300.0, connect=10.0)
    headers = {"Authorization": f"Bearer {TOKEN}"}
    async with httpx.AsyncClient(timeout=timeout, headers=headers) as client:
        outcomes = await asyncio.gather(*(worker(c) for c in chunks),
                                        return_exceptions=True)
        # second pass: retry timed-out keywords in smaller requests
        if shortfalls and not stop.is_set():
            small = [shortfalls[i:i + 3] for i in range(0, len(shortfalls), 3)]
            outcomes += await asyncio.gather(*(worker(c) for c in small),
                                             return_exceptions=True)

    errors = [o for o in outcomes if isinstance(o, Exception)]
    return results, errors, stop.is_set()

if __name__ == "__main__":
    kws = ["nike", "adidas", "puma", "asics", "hoka", "on running", "new balance",
           "brooks", "saucony", "mizuno", "salomon", "reebok"]
    items, errors, capped = asyncio.run(run(kws))
    print(len(items), "items;", len(errors), "failed chunks; charge limit hit:", capped)
```

A few of the choices in that code need explaining.

`includeSearchTerms` is on because once results from several keywords are merged into one list, `__search_terms.keyword` is what tells you which keyword produced which video. Group and filter on that field. The object also carries the `sort_type`, `publish_time` and `region` the request used, which helps if you later mix results from differently configured requests. I trim the items in Python here, not with `customMapFunction`, so that the `__search_terms` object is guaranteed to still be there when the client groups by it. If you'd rather trim on the server, check on a small request that your mapped output still includes the keyword before you rely on it.

Concurrency is 3 requests. With up to 4 keywords running per request on the server side, that's already 12 keyword fetches in flight. I'd only raise it after checking that error rates and timeouts stay flat. The right number depends on your traffic, so measure it.

The retry pass uses chunks of 3. A timeout tells you the chunk was too heavy for the deadline, so sending the same 10 keywords again will probably time out again.

Watch for duplicates. A video that matches two keywords comes back twice. You pay for both copies, so dedupe on `aweme_id` (`id` in the trimmed output) before storing. If you need to know every keyword a video matched, collect the `keyword` values per ID while you dedupe so you don't throw them away.

## Caching and when to stop using server mode

Search results for a keyword over a `WEEK` window don't change much from one minute to the next. If several users of a dashboard or several agent turns ask about the same keyword, put a cache keyed on the full parameter set (keywords, limit, region, sortType, publishTime) in front of the client, with a TTL that fits how fresh you need the data. Every cache hit is an item you don't pay for again and a request that can't time out.

I switch to a batch run when:

- one keyword needs more than 500 videos, or I want `isUnlimited`,
- the job is scheduled and I want the dataset kept as the record of what was seen that day,
- the keyword list is so long that the fan-out spends more effort on chunking, retries and shortfall tracking than on the data.

Server mode fits interactive, bounded questions such as "what's been posted about these five terms this week", called by a person or an agent who is waiting for the answer. Charges are per returned item (plus `user-bio-lookup` when `fetchUserBio` is on). See the [Store page](https://apify.com/novi/advanced-search-tiktok-api?fpr=7hce1m) for current pricing. The full OpenAPI description is on the Actor's **Server** tab in Apify Console.

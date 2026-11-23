---
layout: post
title: "Polling Creators Without Overpaying: TikTok Profile Videos API in Server Mode"
date: 2026-11-24 07:20:00 +0700
permalink: /apify/tiktok-profile-videos-api-server-mode/
categories: [Data Engineering, Analysis]
tags: [Web Scraping, Apify, Server Actors, Standby, TikTok, Python, Influencer Marketing]
author: Scraping Nerd
description: "The server mode of novi/tiktok-user-api turns 'latest videos for this creator' into one GET request. Here is how per-item charging changes polling, why 404 should be cached, and an asyncio worker pool for checking a creator roster."
---

The third TikTok Actor that got server mode this week is the simplest of the three. [TikTok Profile Videos API](https://apify.com/novi/tiktok-user-api?fpr=7hce1m) (slug `tiktok-user-api`) answers one question: what has this account posted recently? Because it's so narrow, the interesting part is how you call it repeatedly, not how you call it once. That's what this post covers.

For background on what server mode (Apify Standby, now called Server Actors) is and where it fits next to batch runs, see [yesterday's analysis](/apify/agent-shaped-scraping-server-actors/). The short version: you send an HTTP request, the videos come back in the response, and nothing is written to a dataset.

## The contract

Base URL: `https://novi--tiktok-user-api.apify.actor`. One endpoint, `GET /posts`.

Give exactly one way to identify the account:

- `username`, e.g. `nike`
- `userId`, e.g. `6714563164218679553`
- `url`, a profile URL such as `https://www.tiktok.com/@nike`

Then `limit`, from 2 to 100 with a default of 20. You get back a JSON array of the account's latest videos as raw TikTok aweme objects, the same shape as the batch run's dataset items. The fields I use most are `aweme_id`, `desc`, `create_time`, `author.unique_id` and the `statistics` counters (`play_count`, `digg_count`, `comment_count`, `share_count`).

The status codes:

| Status | Meaning | Retry? |
|---|---|---|
| 200 | Array of videos | |
| 400 | `{ "error": ... }` / `{ "errors": [...] }`, bad input | No |
| 402 | Charge limit reached | No. Stop everything. |
| 404 | `{ "error": "User not found or could not be resolved." }` | No. Cache it. |
| 502 | Upstream failure | Yes, with backoff |

```bash
curl -sS -G --max-time 300 -H "Authorization: Bearer $APIFY_TOKEN" \
  "https://novi--tiktok-user-api.apify.actor/posts" \
  --data-urlencode "username=nike" --data-urlencode "limit=5" \
  | jq '.[] | {aweme_id, desc, create_time, plays: .statistics.play_count}'
```

## Per-item charging changes how you poll

The endpoint has no "since" parameter. Each call returns the latest `limit` videos, and you're charged per returned item (`result-item`). Polling therefore costs `limit × polls`, however many of those videos you've already seen.

Take a typical influencer-marketing check: "has this creator posted the sponsored video yet?" If you poll each creator with `limit=100`, most of what you pay for is old videos you already have. The new post will be at or near the top of the list, so a small limit (the minimum is 2) answers the question for a fraction of the items. I'd split the work into two kinds of call:

- A vetting call, made once per creator when they enter your pipeline. Use a large `limit` (up to 100) to get enough history for engagement stats.
- Monitoring calls, made repeatedly. Use a small `limit`, compare `aweme_id`s against what you've stored, and only process the new ones.

One edge case: if a creator posts more than `limit` videos between two polls, a small limit will miss some. Choose the monitoring limit based on how often you poll and how often the creator posts. A creator who posts several times a day needs a bigger limit or more frequent polls. Your own data will show you which, if you log how many of each poll's videos were new.

## Why 404 should be cached

A 404 here isn't a broken route. It's the Actor telling you the account can't be resolved: the account may have been renamed or deleted, or someone mistyped the handle when entering it. Retrying in a loop won't fix any of those. In a polling system, an unhandled 404 becomes a permanent error that shows up every cycle and hides the errors that actually need attention.

Treat it as data. Mark the creator `unresolved`, store the time, and skip them for a while (a day, say) before trying again. If your roster can store the numeric `userId`, prefer it to the username, because a creator can change their handle.

## An asyncio worker pool for a creator roster

The Actor takes one account per request, so a roster of creators means fanning out requests. A queue with a fixed number of workers works better here than `gather` over everything. It caps concurrency, lets the whole pool stop cleanly on a 402, and makes it easy to re-queue a creator after a 502.

```python
import asyncio, os, random, time
import httpx

BASE = "https://novi--tiktok-user-api.apify.actor"
TOKEN = os.environ["APIFY_TOKEN"]
WORKERS = 4
MONITOR_LIMIT = 5
MAX_ATTEMPTS = 4

# your persistence layer; dicts here for brevity
seen_ids: dict[str, set[str]] = {}      # creator key -> aweme_ids already stored
unresolved: dict[str, float] = {}       # creator key -> time of last 404

def creator_params(c: dict) -> dict:
    # prefer the stable numeric id when you have it
    if c.get("userId"):
        return {"userId": c["userId"]}
    return {"username": c["username"]}

async def fetch_posts(client, params, limit):
    r = await client.get(f"{BASE}/posts", params={**params, "limit": limit})
    if r.status_code == 200:
        return r.json()
    if r.status_code == 404:
        raise LookupError(r.json().get("error"))
    if r.status_code == 402:
        raise PermissionError("charge limit reached")
    if r.status_code == 400:
        raise ValueError(r.text)
    r.raise_for_status()   # 502 and anything else -> HTTPStatusError

async def worker(client, queue: asyncio.Queue, stop: asyncio.Event, out: list):
    while True:
        creator, attempt = await queue.get()
        key = creator.get("userId") or creator["username"]
        try:
            if stop.is_set():
                continue
            if time.time() - unresolved.get(key, 0) < 86_400:
                continue                      # 404'd within the last day
            try:
                videos = await fetch_posts(client, creator_params(creator), MONITOR_LIMIT)
            except LookupError:
                unresolved[key] = time.time()
                continue
            except PermissionError:
                stop.set()                    # 402: drain the queue
                continue
            except ValueError as e:
                print(f"bad input for {key}: {e}")
                continue
            except (httpx.HTTPStatusError, httpx.TransportError):
                if attempt + 1 < MAX_ATTEMPTS:
                    await asyncio.sleep(2 ** attempt + random.random())
                    queue.put_nowait((creator, attempt + 1))
                else:
                    print(f"giving up on {key}")
                continue

            known = seen_ids.setdefault(key, set())
            new = [v for v in videos if v.get("aweme_id") not in known]
            known.update(v.get("aweme_id") for v in new)
            if videos and len(new) == len(videos) and known - {v.get("aweme_id") for v in new}:
                print(f"{key}: every returned video was new, consider a larger limit")
            out.extend({"creator": key, **v} for v in new)
        finally:
            queue.task_done()

async def poll_roster(roster: list[dict]):
    queue: asyncio.Queue = asyncio.Queue()
    for c in roster:
        queue.put_nowait((c, 0))
    stop, out = asyncio.Event(), []
    timeout = httpx.Timeout(300.0, connect=10.0)
    headers = {"Authorization": f"Bearer {TOKEN}"}
    async with httpx.AsyncClient(timeout=timeout, headers=headers) as client:
        tasks = [asyncio.create_task(worker(client, queue, stop, out))
                 for _ in range(WORKERS)]
        await queue.join()
        for t in tasks:
            t.cancel()
    return out, stop.is_set()

if __name__ == "__main__":
    roster = [{"username": "nike"}, {"userId": "6714563164218679553"},
              {"username": "this-handle-probably-does-not-exist-123"}]
    new_videos, capped = asyncio.run(poll_roster(roster))
    for v in new_videos:
        print(v["creator"], v["aweme_id"], (v.get("desc") or "")[:60])
    print("charge limit hit:", capped)
```

A few notes:

- The sleep before a retry runs inside the worker, so that worker is blocked during backoff while the others keep going. With 4 workers that's fine. For a large roster you'd schedule the retry with a delay and keep the worker free.
- The "every returned video was new" warning is the overflow check from the polling section. If all `MONITOR_LIMIT` videos are unseen and the creator already has stored history, you may have missed some posts, so run one vetting-size call for that creator.
- To check for a sponsored post, filter `new` on `desc` for the campaign hashtag or the brand handle before you notify anyone.
- The two dicts stand in for a real store. In production I'd use a table keyed by `(creator, aweme_id)` so a duplicate insert simply fails.

## Cold starts and agents

The first request after an idle stretch can be slower because Apify has to start a container (containers stay warm for about 300 s of idle time). A roster poll every 15 minutes may hit a cold container on its first request (unless other traffic kept it warm) and a warm one after that, so your latency logs may show one slow request followed by normal ones. An agent tool like "what has @x posted lately" sees whatever state the container is in when the user asks. If that matters, log first-call latency separately from warm-call latency before deciding it's a problem.

For agents I'd keep `limit` small and leave the full aweme objects out of the model's context. Pull out ID, caption, timestamp and counts in your tool wrapper. This Actor has no server-side mapping option, so the reshaping has to happen in your code.

## Limits, and when to switch to batch

Each request covers one account and returns at most 100 videos. Both caps exist because a server-mode request has to finish within about 5 minutes. For a deep history pull on one creator, or a nightly sweep of hundreds of accounts where you want a dataset as the record, use the normal input run. It still works the same as before.

Server mode fits the high-frequency, low-volume end of the problem: vetting a creator while someone waits on the page, checking whether a post went live, answering an agent's question. Pricing is per returned item and listed on the [Store page](https://apify.com/novi/tiktok-user-api?fpr=7hce1m). The OpenAPI spec for `/posts` is on the Actor's **Server** tab in Apify Console.

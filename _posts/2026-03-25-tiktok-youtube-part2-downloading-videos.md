---
layout: post
title: "Part 2: Asynchronous video ingestion and connection pooling"
date: 2026-03-25 08:00:00 +0700
categories: [Systems Engineering, Video Automation, Python]
tags: [apify, httpx, asyncio, python]
permalink: /tiktok/tiktok-youtube-part2-downloading-videos/
---

This is Part 2 of our pipeline series. (See the [Architecture Overview](/tiktok/youtube-news-channel-tiktok-data/) for context.) After normalizing the dataset payload from the [Advanced TikTok Search API Actor](https://apify.com/novi/advanced-search-tiktok-api?fpr=7hce1m), our next challenge is physically ingesting the `.mp4` payloads from TikTok's CDN endpoints.

> **Series Navigation**
> - [Part 1: Interfacing with the Apify Python SDK](/tiktok/tiktok-youtube-part1-searching-api/)
> - **Part 2: Asynchronous video ingestion and connection pooling** ← You are here
> - [Part 3: LLM script synthesis and FFmpeg concatenation](/tiktok/tiktok-youtube-part3-ai-narration/)
> - [Part 4: OAuth2 authentication and YouTube Data API uploads](/tiktok/tiktok-youtube-part4-scheduling-publishing/)

## Engineering constraints 

Downloading 100+ videos synchronously causes substantial I/O blocking. Conversely, unrestrained parallel requests will exhaust file descriptors and trigger remote rate limiters (resulting in `403 Forbidden` or `429 Too Many Requests` responses from the CDN). We need bounded concurrency. 

We use `httpx` for modern Python HTTP connection pooling, wrapped in an `asyncio.Semaphore` to throttle active multiplexed TLS streams.

```bash
pip install httpx asyncio
```

## Bounded concurrency downloader implementation

This script reads the normalized data lake JSON produced in Part 1, builds a target manifest, deduplicates operations using a `set` lookup (using `aweme_id` as the primary key), and initiates pooled GET requests to the `source_url`.

```python
# ingest.py

import asyncio
import json
import logging
import sys
from pathlib import Path

import httpx

# Throttle concurrent TCP connections to the CDN
MAX_CONCURRENT_RQS = 8
TIMEOUT_SEC = 45.0
DATA_LAKE_PATH = Path("data/raw")
INGEST_PATH = Path("data/media_assets")

logging.basicConfig(level=logging.INFO, format="%(levelname)s: %(message)s")

def build_manifest(input_json: Path) -> dict:
    """Parses extract data and yields a deduplicated manifest dict."""
    with open(input_json, "r") as f:
        records = json.load(f)

    manifest = {}
    for r in records:
        uid = r["aweme_id"]
        source_url = r["media"]["source_url"]
        
        # Guard against malformed records
        if not uid or not source_url:
            continue
            
        manifest[uid] = {
            "url": source_url,
            "topic": r["query_context"].replace(" ", "_"),
            "target_dir": INGEST_PATH / r["query_context"].replace(" ", "_"),
            "target_filename": f"{uid}.mp4"
        }
            
    return manifest

async def stream_to_disk(
    client: httpx.AsyncClient, 
    semaphore: asyncio.Semaphore, 
    item_id: str, 
    job: dict
):
    """Downloads a video stream using bounded concurrency and exponential backoff."""
    job["target_dir"].mkdir(parents=True, exist_ok=True)
    out_file = job["target_dir"] / job["target_filename"]
    
    # Idempotent execution
    if out_file.exists():
        logging.debug(f"[{item_id}] Cache hit. Skipping.")
        return "cache"

    async with semaphore:
        for attempt in range(3):
            try:
                # TikTok CDNs require a realistic User-Agent
                headers = {
                    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/115.0.0.0 Safari/537.36",
                    "Referer": "https://www.tiktok.com/"
                }
                
                async with client.stream("GET", job["url"], headers=headers, timeout=TIMEOUT_SEC) as resp:
                    resp.raise_for_status()
                    
                    with open(out_file, "wb") as f:
                        async for chunk in resp.aiter_bytes(chunk_size=1024 * 128): # 128KB chunks
                            f.write(chunk)
                            
                return "ok"
            except (httpx.HTTPError, httpx.TimeoutException) as e:
                logging.warning(f"[{item_id}] Network fault {e}. Retry {attempt+1}/3")
                await asyncio.sleep(2 ** attempt)  # Exponential backoff
        
    return "failed"

async def orchestrate_ingestion(manifest: dict):
    # Enforces absolute limit on active connections
    semaphore = asyncio.Semaphore(MAX_CONCURRENT_RQS)
    
    # httpx.AsyncClient maintains a persistent connection pool (Keep-Alive)
    async with httpx.AsyncClient(http2=True, follow_redirects=True) as client:
        tasks = [
            stream_to_disk(client, semaphore, uid, job)
            for uid, job in manifest.items()
        ]
        
        # Await completion of all ingestion coroutines
        results = await asyncio.gather(*tasks)
        
    oks = results.count("ok")
    caches = results.count("cache")
    fails = results.count("failed")
    logging.info(f"Ingestion complete: OK={oks}, Cached={caches}, Failed={fails}")

if __name__ == "__main__":
    latest_extract = sorted(DATA_LAKE_PATH.glob("extract_*.json"))[-1]
    logging.info(f"Mounting manifest from {latest_extract.name}")
    
    manifest = build_manifest(latest_extract)
    asyncio.run(orchestrate_ingestion(manifest))
```

## Infrastructure design notes

- **Chunked Writes (`aiter_bytes`)**: Reading arbitrary 50MB files securely into memory scales poorly. Passing physical streams into file byte streams avoids buffer constraints.
- **Connection Pooling**: Reusing the TLS connections to the same host over `http2` minimizes handshake bottlenecks, improving performance exponentially versus naive `requests.get()` inside a `ThreadPoolExecutor`.
- **Deduplication**: By building a manifest dictionary indexed on `aweme_id`, multiple query strategies yielding the same video naturally deduplicate.

In [Part 3: LLM script synthesis and FFmpeg concatenation](/tiktok/tiktok-youtube-part3-ai-narration/), we will utilize `ffmpeg-python` bindings and ChatGPT to assemble these raw `.mp4` chunks into a single chronological presentation.

> **Series Navigation**
> - [Part 1: Interfacing with the Apify Python SDK](/tiktok/tiktok-youtube-part1-searching-api/)
> - **Part 2: Asynchronous video ingestion and connection pooling** ← You are here
> - [Part 3: LLM script synthesis and FFmpeg concatenation](/tiktok/tiktok-youtube-part3-ai-narration/)
> - [Part 4: OAuth2 authentication and YouTube Data API uploads](/tiktok/tiktok-youtube-part4-scheduling-publishing/)

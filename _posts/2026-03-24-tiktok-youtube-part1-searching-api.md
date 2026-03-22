---
layout: post
title: "Part 1: Interfacing with the Apify Python SDK for TikTok Extraction"
date: 2026-03-24 08:00:00 +0700
categories: [Systems Engineering, Video Automation, Python]
tags: [apify, tiktok-api, python, etl]
permalink: /tiktok/tiktok-youtube-part1-searching-api/
---

This is Part 1 of our pipeline architecture series. (See the [Architecture Overview](/tiktok/youtube-news-channel-tiktok-data/) for context.) Here, we implement the extraction layer using the `apify-client` SDK to interface with the [Advanced TikTok Search API Actor](https://apify.com/novi/advanced-search-tiktok-api?fpr=7hce1m).

> **Series Navigation**
> - **Part 1: Interfacing with the Apify Python SDK** ← You are here
> - [Part 2: Asynchronous video ingestion and connection pooling](/tiktok/tiktok-youtube-part2-downloading-videos/)
> - [Part 3: LLM script synthesis and FFmpeg concatenation](/tiktok/tiktok-youtube-part3-ai-narration/)
> - [Part 4: OAuth2 authentication and YouTube Data API uploads](/tiktok/tiktok-youtube-part4-scheduling-publishing/)

## Infrastructure dependencies

We rely on the official Apify SDK to manage API requests and dataset pagination to Apify's infrastructure.

```bash
pip install apify-client python-dotenv
```

Store your `APIFY_TOKEN` securely in environment variables or a `.env` file. Never hardcode access tokens in version control.

## Schema configurations and targeted queries

The actor strictly defines an input schema. When extracting data for news events, geographical relevance (`region`) and recency (`publishTime`) are critical query parameters.

```python
# config.py

import os
from dotenv import load_dotenv

load_dotenv()

APIFY_TOKEN = os.environ.get("APIFY_TOKEN")
if not APIFY_TOKEN:
    raise EnvironmentError("Missing APIFY_TOKEN boundary condition.")

# Input schema matching the Advanced TikTok Search API
SEARCH_PAYLOADS = [
    {
        "keyword": "Ukraine frontline",
        "region": "UA",
        "sortType": 2,       # 2 maps to "Most recent"
        "publishTime": "YESTERDAY",
        "limit": 50,
    },
    {
        "keyword": "France protests",
        "region": "FR",
        "sortType": 1,       # 1 maps to "Most liked"
        "publishTime": "WEEK",
        "limit": 30,
    }
]
```

## Executing the extraction pipeline

To separate raw HTTP responses from our pipeline's expected schema, we implement a normalizer (`normalize_payload`) that extracts only the specific fields required downstream (e.g., `aweme_id`, `video_url_no_watermark`, `desc`). Storing the entire raw payload from the actor wastes disk space and database I/O bandwidth.

```python
# extract.py

import json
from datetime import datetime
from pathlib import Path

from apify_client import ApifyClient
from config import APIFY_TOKEN, SEARCH_PAYLOADS

ACTOR_ID = "novi/advanced-search-tiktok-api"
DATA_LAKE_PATH = Path("data/raw")

def normalize_payload(raw_item: dict, original_query: str) -> dict:
    """Normalizes the raw JSON schema into a consistent pipeline format."""
    stats = raw_item.get("statistics", {})
    play_addr = raw_item.get("video", {}).get("play_addr", {})

    return {
        "aweme_id": raw_item.get("aweme_id"),
        "description": raw_item.get("desc", ""),
        "region": raw_item.get("region"),
        "timestamp_unix": raw_item.get("create_time"),
        "metrics": {
            "views": stats.get("play_count", 0),
            "likes": stats.get("digg_count", 0),
            "shares": stats.get("share_count", 0),
        },
        "media": {
            # Extract the first available CDN URL; prioritizing the watermark-free address
            "source_url": play_addr.get("url_list", [None])[0],
            "duration_ms": raw_item.get("video", {}).get("duration"),
        },
        "query_context": original_query
    }

def fetch_datasets():
    client = ApifyClient(APIFY_TOKEN)
    normalized_results = []

    for payload in SEARCH_PAYLOADS:
        print(f"Triggering execution for: {payload['keyword']}")
        
        # Blocking call: waits for the actor to complete execution on Apify
        run = client.actor(ACTOR_ID).call(run_input=payload)
        
        # Paginate through the associated dataset
        items_iterator = client.dataset(run["defaultDatasetId"]).iterate_items()
        
        for item in items_iterator:
            normalized_results.append(normalize_payload(item, payload['keyword']))
            
    return normalized_results

if __name__ == "__main__":
    results = fetch_datasets()
    
    DATA_LAKE_PATH.mkdir(parents=True, exist_ok=True)
    out_file = DATA_LAKE_PATH / f"extract_{datetime.now().strftime('%Y%m%d%H%M')}.json"
    
    with open(out_file, "w") as f:
        json.dump(results, f, separators=(',', ':')) # Minified JSON
        
    print(f"Extracted {len(results)} records to local storage.")
```

By explicitly flattening nested hierarchies into `metrics` and `media` dictionaries, we decouple the actor's API shape from our internal domain model. 

In [Part 2: Asynchronous video ingestion and connection pooling](/tiktok/tiktok-youtube-part2-downloading-videos/), we write a concurrent parser using `asyncio` and `httpx` to actually fetch the `.mp4` payloads from the retrieved CDN URLs.

> **Series Navigation**
> - **Part 1: Interfacing with the Apify Python SDK** ← You are here
> - [Part 2: Asynchronous video ingestion and connection pooling](/tiktok/tiktok-youtube-part2-downloading-videos/)
> - [Part 3: LLM script synthesis and FFmpeg concatenation](/tiktok/tiktok-youtube-part3-ai-narration/)
> - [Part 4: OAuth2 authentication and YouTube Data API uploads](/tiktok/tiktok-youtube-part4-scheduling-publishing/)

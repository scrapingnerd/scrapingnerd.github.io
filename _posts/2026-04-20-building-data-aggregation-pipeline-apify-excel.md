---
layout: post
title: "Merging Multiple Apify Dataset Runs into a Single Excel File with Python"
permalink: /merging-multiple-apify-runs-excel-python/
---

# Merging Multiple Apify Dataset Runs into a Single Excel File with Python

Large-scale TikTok scraping jobs on Apify are never a single operation. Subdividing work across multiple **Actor runs** is the correct pattern—it avoids **pagination ceilings**, circumvents actor timeouts, and parallelizes throughput. The bottleneck is what comes next: you have N distinct **Datasets** scattered across your Apify account and need a unified table for analysis.

This is a data engineering problem. This post walks through the Python implementation to solve it: chain multiple `novi/fast-tiktok-api` runs, normalize the nested JSON output, and write the merged result to `.xlsx`.

---

## 1. Dependencies

Install the three packages required. Nothing more.

```bash
pip install pandas apify openpyxl
```

* **`apify`**: The official SDK for the Apify REST API.
* **`pandas`**: DataFrame operations and JSON normalization.
* **`openpyxl`**: Excel write engine used by Pandas.

---

## 2. Implementation

### Authentication

Instantiate the Apify client. Assign your token directly for local scripts; pull from an environment variable in automated pipelines.

```python
import pandas as pd
from apify_client import ApifyClient

APIFY_API_TOKEN = "your_token_here"  # or os.getenv("APIFY_API_TOKEN")
ACTOR_ID = "novi/fast-tiktok-api"

client = ApifyClient(APIFY_API_TOKEN)
```

### Dispatching Runs

Loop over your keyword list. Wrap every call in `try/except`—a single failed run must not abort the entire pipeline.

```python
queries = ["dance", "coding"]
dataset_ids = []

for query in queries:
    print(f"Dispatching run: '{query}'...")
    try:
        run = client.actor(ACTOR_ID).call(run_input={
            "type": "SEARCH",
            "keyword": query,
            "limit": 10
        })
        dataset_ids.append((query, run['defaultDatasetId']))
        print(f"  → Dataset: {run['defaultDatasetId']}")
    except Exception as e:
        print(f"  ✗ Failed ({query}): {e}")
```

### Fetching and Normalizing

TikTok API responses are deeply nested JSON. Using `pd.DataFrame()` directly produces columns containing raw dict objects. Use **`pd.json_normalize()`** to flatten the structure into a proper columnar format before concatenation.

```python
dataframes = []

for query, dataset_id in dataset_ids:
    print(f"Fetching dataset: {dataset_id}...")
    items = list(client.dataset(dataset_id).iterate_items())

    if items:
        df = pd.json_normalize(items)   # Flatten nested JSON
        df["source_query"] = query      # Tag for lineage tracking
        dataframes.append(df)
        print(f"  → {len(df)} records")
```

### Merging and Exporting

```python
if dataframes:
    merged_df = pd.concat(dataframes, ignore_index=True)
    output_file = "./Merged_TikTok_Data.xlsx"
    merged_df.to_excel(output_file, index=False, engine="openpyxl")
    print(f"\n✅ Done! {len(merged_df)} records → {output_file}")
else:
    print("⚠️  No data extracted.")
```

The `index=False` argument drops Pandas' default integer index—without it, Excel will have a redundant leading column.

---

## 3. Full Script

```python
# Prerequisites: pip install pandas apify openpyxl

import pandas as pd
from apify_client import ApifyClient

# ── Config ────────────────────────────────────────────────────────────────────
APIFY_API_TOKEN = "your_token_here"
ACTOR_ID = "novi/fast-tiktok-api"

client = ApifyClient(APIFY_API_TOKEN)

# ── Input ─────────────────────────────────────────────────────────────────────
queries = ["dance", "coding"]
dataset_ids = []

# ── Dispatch Actor Runs ───────────────────────────────────────────────────────
for query in queries:
    print(f"Triggering Run: query='{query}'...")
    try:
        run = client.actor(ACTOR_ID).call(run_input={"type": "SEARCH", "keyword": query, "limit": 10})
        dataset_ids.append((query, run['defaultDatasetId']))
        print(f"  → Dataset ID: {run['defaultDatasetId']}")
    except Exception as e:
        print(f"  ✗ Actor error ({query}): {e}")

# ── Data Collection and Normalization ─────────────────────────────────────────
dataframes = []
for query, dataset_id in dataset_ids:
    print(f"Fetching data from dataset: {dataset_id}...")
    items = list(client.dataset(dataset_id).iterate_items())

    if items:
        df = pd.json_normalize(items)  # Flatten nested JSON
        df['source_query'] = query
        dataframes.append(df)
        print(f"  → {len(df)} records fetched")

# ── Merge and Export ──────────────────────────────────────────────────────────
if dataframes:
    merged_df = pd.concat(dataframes, ignore_index=True)
    output_file = "./Merged_TikTok_Data.xlsx"
    merged_df.to_excel(output_file, index=False, engine="openpyxl")
    print(f"\n✅ Done! {len(merged_df)} records saved to: {output_file}")
else:
    print("⚠️  No data extracted.")
```

---

## 4. Scaling Beyond This Script

This script handles ad-hoc workloads cleanly. To push it into production:

* **Parallel dispatch:** `client.actor().call()` is blocking. Switch to `asyncio` + `apify-client-async` to fire all runs concurrently instead of sequentially.
* **Database sink:** Excel caps at ~1M rows and is slow to write at scale. Replace `to_excel` with an insert into **PostgreSQL** via `psycopg2` or a bulk upload to **BigQuery** for analytical workloads.
* **Retry logic:** High-frequency dispatching hits HTTP 429 errors. Add exponential backoff using the `tenacity` library to automatically retry on transient failures.

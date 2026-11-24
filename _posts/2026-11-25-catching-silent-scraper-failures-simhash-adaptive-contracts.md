---
layout: post
title: "Catching Silent Scraper Failures: SimHash Soft-Block Clustering, Adaptive Relocation, and Drift Gates"
date: 2026-11-25 00:15:00 +0700
permalink: /data-engineering/catching-silent-scraper-failures-simhash-adaptive-contracts/
categories: [Data Engineering, Web Scraping, Analysis]
tags: [Web Scraping, SimHash, Deduplication, Schema Drift, Scrapling, Data Contracts, Python, Apify]
author: Scraping Nerd
description: "HTTP 200 does not mean valid data. How to catch soft-blocks and layout breaks before they poison your database using 64-bit SimHash band indexing, structural element relocation, and batch schema-drift gates."
---

An HTTP 403 or 500 is easy to notice: your retry queue backs up and the dashboard turns red. The failure mode that ruins downstream analytics and RAG indexes returns `200 OK`. It looks like success at the transport layer while delivering a Cloudflare "Just a moment..." page with a rotating Ray ID, a login redirect, or a redesigned product card where `price` silently evaluates to `null`.

Standard defensive plumbing misses both cases. Exact SHA-256 hashes fail as soon as a page injects a dynamic timestamp or request nonce, and static CSS selectors fail silently when a frontend deploy renames hashed Tailwind classes. To stop broken scrapes before they hit PostgreSQL or ClickHouse, I pair three lightweight checks in code: volatile-token-normalized 64-bit SimHash with band indexing, structural DOM relocation with drift warnings, and a batch quarantine gate.

## Defense 1: 64-bit SimHash for near-duplicates and soft-block clusters

In their 2007 paper [Detecting Near-Duplicates for Web Crawling](https://research.google/pubs/detecting-near-duplicates-for-web-crawling/), Manku, Jain, and Das Sarma showed how 64-bit SimHash fingerprints turn document similarity into a bitwise Hamming distance check, with `k <= 3` catching near-duplicates across billions of pages without $O(N)$ comparisons when split across permutation tables or fixed bit bands. Splitting a 64-bit integer into four 16-bit bands works by the pigeonhole principle: if two 64-bit fingerprints differ by at most 3 bits, at least one of the four 16-bit bands must match identically.

While benchmarking this in Python (3-word shingles hashed via 8-byte `blake2b`), I ran into a failure mode that rarely gets mentioned in SimHash tutorials: document length and ephemeral IDs skew small pages badly. I measured Hamming distances across three document lengths after appending an 8-token footer (`session nonce 998172 rendered 2026-11-25T00:14:02Z`):

| Test case | Word count | Raw Hamming distance | Normalized Hamming distance | SHA-256 match |
|---|---|---|---|---|
| Short product card + dynamic footer | 45 | 12 | 8 | No |
| Medium description + dynamic footer | 185 | 5 | Not tested | No |
| Full product page + dynamic footer | 399 | 3 | 2 | No |
| Distinct product pages (chair vs USB-C dock) | 399 vs 114 | Not tested | 35 | No |
| 50 short WAF interstitials (`cf-99a000..049`) | 25 | 4 to 16 | 0 | No |

Look closely at that last row. Soft-block challenge pages are often short, around 25 to 60 words, with a unique Ray ID (`cf-99a042`) or IP address stamped into every reply. With raw 3-word shingles, one rotating hyphenated ID mutated three overlapping shingles out of roughly 22, pushing raw Hamming distances up to `16` and slipping completely past a `k <= 3` threshold. Replacing timestamps, IPv4 addresses, and hex or alphanumeric IDs before shingling collapsed all 50 interstitials to Hamming distance `0`, while leaving genuinely distinct pages separated by `35` bits.

```python
from collections import Counter, defaultdict
import hashlib
import re


def normalize_volatile_tokens(text: str) -> str:
    s = re.sub(r"\d{4}-\d{2}-\d{2}[tT ]\d{2}:\d{2}:\d{2}\S*", " tokts ", text)
    s = re.sub(r"\b\d{1,3}(?:\.\d{1,3}){3}\b", " tokip ", s)
    s = re.sub(
        r"\b(?:[a-z]{1,4}-[0-9a-f]{4,}|[0-9a-f]{8,}|\d{6,})\b",
        " tokid ",
        s,
        flags=re.IGNORECASE,
    )
    return s


def simhash64(text: str, ngram: int = 3) -> int:
    tokens = re.findall(r"[a-z0-9]+", text.lower())
    if not tokens:
        return 0
    if len(tokens) < ngram:
        features = tokens
    else:
        features = [
            " ".join(tokens[i : i + ngram]) for i in range(len(tokens) - ngram + 1)
        ]

    acc = [0] * 64
    for feat, weight in Counter(features).items():
        h = int.from_bytes(
            hashlib.blake2b(feat.encode("utf-8"), digest_size=8).digest(),
            "big",
        )
        for bit in range(64):
            acc[bit] += weight if ((h >> bit) & 1) else -weight

    fp = 0
    for bit in range(64):
        if acc[bit] >= 0:
            fp |= 1 << bit
    return fp


class SimHashBandIndex:
    def __init__(self, max_hamming: int = 3):
        self.max_hamming = max_hamming
        self.bands = [defaultdict(list) for _ in range(4)]
        self.records = {}

    @staticmethod
    def _keys(fp: int) -> list[int]:
        return [(fp >> (i * 16)) & 0xFFFF for i in range(4)]

    def query(self, fp: int) -> list[tuple[str, int]]:
        seen = set()
        hits = []
        for idx, key in enumerate(self._keys(fp)):
            for doc_id in self.bands[idx].get(key, ()):
                if doc_id in seen:
                    continue
                seen.add(doc_id)
                dist = (fp ^ self.records[doc_id]).bit_count()
                if dist <= self.max_hamming:
                    hits.append((doc_id, dist))
        return hits

    def add(self, doc_id: str, fp: int) -> None:
        self.records[doc_id] = fp
        for idx, key in enumerate(self._keys(fp)):
            self.bands[idx][key].append(doc_id)
```

In a single-threaded benchmark on 500 documents of 399 words each, pure Python 3-shingle hashing completed in 692.94 ms (`722` docs/sec). That is plenty fast enough to run inline inside a batch validation worker, and it catches something useful: if 40 out of 50 distinct URLs in one crawl suddenly collide at Hamming distance `<= 2`, you have a soft-block wave even though every HTTP status says `200`. Pairing that signal with proxy session retirement (the pattern behind [Crawlee's SessionPool](https://crawlee.dev/docs/guides/session-management)) lets workers drop burned proxies right away instead of polling through a block wall.

## Defense 2: structural relocation without LLM latency

When a storefront ships a CSS refactor and renames `class="price-tag"` to `class="tw-price-81bc"`, throwing every page at an LLM fallback adds latency and inference cost. Libraries such as [Scrapling](https://github.com/D4Vinci/Scrapling) handle this in their [adaptive scraping mode](https://scrapling.readthedocs.io/en/latest/parsing/adaptive.html) by saving a structural fingerprint of the target node on healthy runs and scoring candidate elements by structural similarity when the primary selector misses.

A compact structural scorer built on Python's `html.parser` shows how this works using five cues:

1. Tag name (`span`, `div`, `p`).
2. Direct parent tag and trailing ancestor path.
3. Attribute key names and readable tokens from attribute values (`sku`, `price`).
4. Sibling tag distribution inside the same container.
5. Text format shape (such as matching `^\$\d+(\.\d{2})?$`, without hardcoding the dollar value itself).

When I tested this against a refactored product card where `class="price-tag"` changed to `"tw-price-81bc"`, `data-testid="sku-price"` became `data-metric="sku-price-display"`, a new `<div class="new-wrapper">` wrapped the row, and the stock badge moved after the price, the static selector returned `failed`, while structural scoring recovered `"$349.00"` with a similarity score of `0.725`.

Adaptive relocation works best as a temporary fallback, because silent fuzzy matches carry their own risk. Every time relocation triggers, attach `drift_warning: true` and the similarity score to the output record. If relocation fires on a handful of runs after a Friday deploy, you still get clean data over the weekend while alerting engineering to update the canonical selector on Monday. If you let fuzzy relocation run unmonitored for months, a page adding a "Suggested Add-on: $29.00" badge can eventually outscore the real item price.

## Defense 3: batch data contracts and dead-letter quarantine

Row-level checks (such as validating types and positive ranges in [Pydantic](https://docs.pydantic.dev/latest/)) stop malformed individual records, yet they still let a partially broken crawl trickle bad rows into production tables. Putting a batch gate right in front of the `INSERT` or `COPY` transaction checks two aggregate ratios across the whole micro-batch before anything commits:

- `soft_block_cluster_ratio`: fraction of distinct URLs in the batch whose normalized SimHash matches an earlier URL in the same batch within Hamming distance `<= 3`.
- `invalid_price_ratio`: fraction of rows missing a valid positive numeric price.

```python
import json
import sqlite3
import time


def validate_and_commit_batch(
    batch_rows: list[dict],
    conn: sqlite3.Connection,
    dlq_path: str,
    max_null_ratio: float = 0.20,
    max_soft_block_ratio: float = 0.30,
) -> dict:
    total = len(batch_rows)
    if total == 0:
        return {"committed": False, "reason": "empty_batch"}

    batch_index = SimHashBandIndex(max_hamming=3)
    cluster_hits = 0
    for idx, row in enumerate(batch_rows):
        fp = simhash64(normalize_volatile_tokens(row["raw_text"]))
        if batch_index.query(fp):
            cluster_hits += 1
        batch_index.add(f"row_{idx}", fp)

    soft_block_ratio = cluster_hits / total
    bad_prices = sum(
        1
        for r in batch_rows
        if r.get("price") is None
        or not isinstance(r.get("price"), (int, float))
        or r["price"] <= 0
    )
    null_ratio = bad_prices / total

    reasons = []
    if soft_block_ratio > max_soft_block_ratio:
        reasons.append(f"soft_block_cluster_ratio={soft_block_ratio:.2f} > {max_soft_block_ratio:.2f}")
    if null_ratio > max_null_ratio:
        reasons.append(f"invalid_price_ratio={null_ratio:.2f} > {max_null_ratio:.2f}")

    if reasons:
        with open(dlq_path, "a", encoding="utf-8") as fh:
            fh.write(json.dumps({
                "quarantined_at": time.time(),
                "reasons": reasons,
                "batch_size": total,
                "sample_urls": [r["url"] for r in batch_rows[:3]],
            }) + "\n")
        return {"committed": False, "quarantined": True, "reasons": reasons}

    for r in batch_rows:
        conn.execute(
            "INSERT OR REPLACE INTO validated_prices VALUES (?, ?, ?, ?)",
            (r["url"], r["sku"], float(r["price"]), r["extraction_mode"]),
        )
    conn.commit()
    return {"committed": True, "quarantined": False, "rows_written": total}
```

One limit on `soft_block_cluster_ratio`: only apply page-text clustering across URLs that ought to carry distinct items, such as separate product pages. When a job pulls paginated search results or short template variants of one product, stick to null-rate and structural checks so legitimate variants do not trigger a false block alarm.

## Applying the contract gate to mixed web and social feeds

In my end-to-end test, a clean 5-row batch of distinct products committed cleanly to SQLite (`rows_written: 5`). A second 5-row poisoned batch containing five distinct URLs returning Cloudflare-style challenge text with rotating Ray IDs and `price: None` tripped both gates (`soft_block_cluster_ratio=0.80 > 0.30` and `invalid_price_ratio=1.00 > 0.20`), routed a diagnostic entry to `quarantine.jsonl`, and left the production SQLite table untouched at 5 rows.

That same contract boundary pays off when you pull social sentiment alongside storefront data. Even managed extractors such as [TikTok Comment API](https://apify.com/novi/tiktok-comment-api?fpr=7hce1m), [TikTok Scraper](https://apify.com/xtdata/tiktok-scraper?fpr=7hce1m), and [Twitter - X.com Scraper](https://apify.com/xtdata/twitter-x-scraper?fpr=7hce1m) sit on top of third-party mobile and GraphQL schemas that rename nested fields or return empty arrays during upstream rate limits. Passing every dataset pull through a shared batch gate (checking item-level null rates, expected nested keys, and duplicate text clusters before appending to your warehouse) ensures one upstream schema change gets quarantined in JSONL before it leaves empty columns in Monday's report.

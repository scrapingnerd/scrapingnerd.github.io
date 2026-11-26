---
layout: post
title: "Measuring X (Twitter) Audience Overlap from Follower Samples: Bias, Scan Caps and Cost"
date: 2026-11-26 08:40:00 +0700
permalink: /apify/x-twitter-audience-overlap-follower-sampling/
categories: [Data Engineering, Analysis]
tags: [Web Scraping, Apify, Twitter, X.com, Followers, Network Analysis, Python, Sampling]
author: Scraping Nerd
description: "Notes on comparing X audiences with scraped follower lists: why the first N followers are not a random sample, which overlap metric survives truncated lists, how a filter's match rate sets the scan cap, and what filter-before-charge pricing does to the cost model."
---

"How much do the audiences of account A and account B overlap?" sounds like a set intersection. On X, though, you almost never have the full sets. A 2M-follower account would take hours to page through and cost real money, so in practice you compare samples. How you take the sample matters more than people expect.

These notes use [Twitter X.com Followers & Following Scraper](https://apify.com/xtdata/twitter-x-followers-following-scraper?fpr=7hce1m), a new Apify actor that returns follower and following lists as full profiles and filters on the server. Most of the reasoning applies to any follower-list source.

## The first N followers are not a random sample

A followers list is a paginated feed, and X serves it roughly newest-first. Ask for `maxItems: 1000` and you get the 1,000 accounts that followed most recently, not 1,000 drawn uniformly from the whole audience.

That has three consequences:

1. **Recent followers skew toward recent events.** If account A went viral last week, its newest 1,000 followers are mostly people who came in through that post. That's a real audience, but not a representative one.
2. **Bots and fresh accounts are over-represented.** Spam waves follow in bursts, and so do newly created accounts. The `createdAt` field shows this: plot account age for the sample and you'll often see a spike in the last few months.
3. **Overlap measured this way is "overlap among recent followers."** Two accounts that shared an audience five years ago and drifted apart will look less similar than they are. Two accounts that both got picked up by the same news cycle will look more similar.

For "who is paying attention right now", recent followers are exactly what you want. For long-run audience similarity they aren't, and since the list can't be sampled randomly, the honest move is to label the metric for what it is.

## Pick an overlap metric that survives truncation

Say you have samples S_A and S_B, each capped at n, from accounts with full audiences of size N_A and N_B.

**Jaccard** `|S_A ∩ S_B| / |S_A ∪ S_B|` behaves badly here. Its value depends on n and on how unequal the account sizes are. A small account's full list against a large account's newest 1,000 is not comparable with two equal-sized samples.

**Containment** `|S_A ∩ S_B| / |S_A|` asks a narrower question: what share of A's sampled followers also follow B? That's easier to reason about, and it's what marketing questions usually mean anyway ("how many of my followers also follow the competitor?").

The catch is that `S_B` is truncated too, so a follower of both may simply fall outside B's sample. Truncate only one side and you avoid this. Take a sample of A, then test each sampled user against the *full* B, for example by checking whether B appears in that user's **following** list. That flips the question from "scrape B's 2M followers" to "scrape the following lists of 500 people", which is usually far cheaper because most people follow a few hundred accounts.

The actor supports both directions in one run (`getFollowers`, `getFollowing`), and every row carries `type` and `sourceUser`, so the joins are mechanical.

## A minimal overlap script

```python
from collections import defaultdict
from apify_client import ApifyClient
import os

client = ApifyClient(os.environ["APIFY_TOKEN"])
ACTOR = "xtdata/twitter-x-followers-following-scraper"

def followers(handles, n):
    run = client.actor(ACTOR).call(run_input={
        "usernames": handles, "getFollowers": True, "maxItems": n,
    })
    by_source = defaultdict(set)
    for row in client.dataset(run["defaultDatasetId"]).iterate_items():
        by_source[row["sourceUser"]].add(row["id"])
    return by_source

s = followers(["account_a", "account_b"], 1000)
a, b = s["account_a"], s["account_b"]
print(f"containment(A in B) = {len(a & b) / len(a):.3f}  "
      f"(both sides truncated, treat as a lower bound)")
```

Compare by `id`, not `username`. Handles change, and IDs don't. `sourceUser` echoes your input exactly, so the dict keys match the strings you passed.

## The scan cap, as arithmetic

The server-side filters (`minFollowers`, `maxFollowers`, `onlyVerified`, `onlyWithEmail`, `onlyWithWebsite`) drop rows before delivery. A filter with match rate p needs, on average, `maxItems / p` scanned profiles to fill `maxItems`. To keep a strict filter from paging through a celebrity's entire audience, the actor stops each list after `maxScanned` profiles, which defaults to `maxItems × 5`.

The default therefore assumes p ≥ 0.2. Some reference points:

- `minFollowers: 100` on an established account's followers: often well above 0.2, so the default is fine.
- `onlyWithEmail: true`: emails in bios are rare, so p is usually far below 0.2 and the default cap will cut you off.
- Combining filters multiplies their match rates (roughly, since they aren't independent).

Measure p cheaply before a big run. Set a small `maxItems` and a fixed `maxScanned`, for example 50 and 1,000. If the log reports `scan limit reached` with k delivered, then p ≈ k / 1000. Then set `maxScanned ≈ 1.3 × maxItems / p` for the real run. The 1.3 covers the variance and the fact that match rates drift as you go deeper into older followers.

Keep in mind that a deep scan pushes the sample further into the past. That partly offsets the recency bias from the first section, but in a way you don't control: a filtered sample spans a time window that depends on p.

## Filter-before-charge changes the cost model

The actor charges per **delivered** profile ($0.0005 per follower and $0.001 per followed account on Apify's paid plans, with no start fee). Scanned-but-filtered profiles are free. So:

```
cost  ≈ delivered × price
time  ≈ scanned / page_rate ≈ delivered / (p × page_rate)
```

Cost depends only on what you keep, while runtime grows as p shrinks. Many scrapers bill for scanned rows, which pushes you toward loose filters and local cleanup. Here the cheap strategy is the opposite: make the filter as strict as you mean it, give it a generous `maxScanned`, and accept a slower run. Set `maxTotalChargeUsd` on the run as a ceiling. The actor stops cleanly when the next item would exceed it.

One more cost detail matters for the containment trick. Following rows cost twice as much as follower rows. Testing 500 sampled users against full following lists of ~300 each is 150k following rows, which is not cheap. If you only need "does user X follow B?", cap `maxItems` on the following side, accept a small false-negative rate for users who follow thousands of accounts, and report that rate.

## A quick sanity checklist

- Report n, the sampling direction (newest-first), and the filters with every overlap number.
- Compare by `id`.
- Drop `protected` accounts from denominators. Their lists are invisible, and they cost nothing, so they come back as zero rows rather than errors.
- Check the `createdAt` distribution of each sample before trusting a spike in overlap. Two accounts hit by the same bot wave will "overlap" heavily.
- If a list ends before `maxItems` without a scan-limit message, X served fewer users than the profile count claims. That happens, and it's not a sampling choice you made.

For finding the accounts to compare in the first place, [X.com Twitter User Search Scraper](https://apify.com/xtdata/twitter-x-user-search-scraper?fpr=7hce1m) works by keyword. To check that the "overlap" is about the same topic, pull recent posts from the shared users with [Twitter - X.com Scraper](https://apify.com/xtdata/twitter-x-scraper?fpr=7hce1m).

---
layout: post
title: "Polite, Fast Web Crawlers: RFC 9309 robots.txt Matching, Min-Heap Frontiers, and AIMD Rate Control"
date: 2026-11-25 08:25:00 +0700
permalink: /data-engineering/polite-web-crawlers-rfc9309-robots-aimd-frontier/
categories: [Data Engineering, Web Scraping, Systems]
tags: [Web Scraping, RFC 9309, robots.txt, AIMD, Rate Limiting, Python, Crawler Architecture, Apify]
author: Scraping Nerd
description: "Why Python's urllib.robotparser fails modern RFC 9309 rules on sites like GitHub, and how to pair a longest-match evaluator with a per-host min-heap frontier and AIMD Retry-After rate control."
---

Most scraper failures under load come from scheduling bugs inside the crawler loop. A fixed worker pool firing concurrent requests at one host trips `429 Too Many Requests` walls within seconds, while a single global sleep timer slows down every fast host whenever one slow site stalls.

Polite, high-throughput crawlers separate three concerns: accurate [RFC 9309](https://www.rfc-editor.org/rfc/rfc9309.html) `robots.txt` evaluation, per-host scheduling via a min-heap frontier, and adaptive rate control using Additive Increase / Multiplicative Decrease (AIMD) paired with [RFC 9110](https://www.rfc-editor.org/rfc/rfc9110.html) `Retry-After` parsing.

## Where Python's urllib.robotparser diverges from RFC 9309

Many Python scrapers rely on [`urllib.robotparser`](https://docs.python.org/3/library/urllib.robotparser.html) without realizing it predates RFC 9309's 2022 standardization of longest-match precedence, `*` wildcards, and `$` end-of-path anchors.

To measure how much this matters on live sites, I parsed three public `robots.txt` files:

| Host | Size (bytes) | User-agent lines | `Allow` rules | `Disallow` rules | Wildcard or `$` rules | `Crawl-delay` lines |
|---|---|---|---|---|---|---|
| `https://github.com/robots.txt` | 30,190 | 16 | 34 | 855 | 859 | 2 |
| `https://en.wikipedia.org/robots.txt` | 28,275 | 34 | 4 | 460 | 0 | 1 |
| `https://docs.python.org/robots.txt` | 517 | 1 | 0 | 22 | 0 | 0 |

GitHub's `robots.txt` uses `859` wildcard or `$`-anchored rules across `889` total `Allow` and `Disallow` lines. Running a seven-vector RFC 9309 compliance suite against Python 3.9's `urllib.robotparser` exposed four mismatches:

| Rule snippet | Test path | RFC 9309 expected | `urllib.robotparser` | RFC 9309 matcher |
|---|---|---|---|---|
| `Disallow: /search` then `Allow: /search/public` | `/search/public?q=test` | Allow (`True`) | Disallow (`False`) | Allow (`True`) |
| `Disallow: /private$` | `/private` | Disallow (`False`) | Allow (`True`) | Disallow (`False`) |
| `Disallow: /*.json$` | `/api/items.json` | Disallow (`False`) | Allow (`True`) | Disallow (`False`) |
| `Disallow: /tie` then `Allow: /tie` | `/tie/page` | Allow (`True`) | Disallow (`False`) | Allow (`True`) |

Under RFC 9309 Section 2.2.2, the longest matching rule wins regardless of line order, and `Allow` breaks equal-length ties. Because `urllib.robotparser` stops at the first matching prefix and treats `*` and `$` as literal characters, it can either skip permitted subtrees or crawl disallowed wildcard endpoints.

Here is a compact RFC 9309 matcher in standard-library Python that handles longest-match precedence, wildcards, `$` anchors, and user-agent specificity:

```python
import re


class RFC9309Matcher:
    def __init__(self, robots_txt: str):
        self.groups = []
        cur_agents = []
        cur_rules = []
        seen_rule = False

        for raw_line in robots_txt.splitlines():
            line = raw_line.split("#", 1)[0].strip()
            if not line or ":" not in line:
                continue
            field, value = line.split(":", 1)
            field = field.strip().lower()
            value = value.strip()

            if field == "user-agent":
                if seen_rule:
                    self.groups.append((cur_agents, cur_rules))
                    cur_agents = []
                    cur_rules = []
                    seen_rule = False
                cur_agents.append(value.lower())
            elif field in ("allow", "disallow"):
                if not cur_agents:
                    continue
                seen_rule = True
                if field == "disallow" and value == "":
                    cur_rules.append(("allow", "/"))
                else:
                    cur_rules.append((field, value))

        if cur_agents:
            self.groups.append((cur_agents, cur_rules))

    @staticmethod
    def _compile_pattern(pattern: str) -> re.Pattern:
        anchored = pattern.endswith("$")
        body = pattern[:-1] if anchored else pattern
        regex = ".*".join(re.escape(seg) for seg in body.split("*"))
        if anchored:
            regex += "$"
        return re.compile("^" + regex)

    def _select_rules(self, user_agent: str) -> list:
        ua = user_agent.lower()
        best_len = -1
        best_rules = []
        wildcard_rules = []
        for agents, rules in self.groups:
            for ag in agents:
                if ag == "*":
                    wildcard_rules.extend(rules)
                elif ag and ua.startswith(ag):
                    if len(ag) > best_len:
                        best_len = len(ag)
                        best_rules = list(rules)
                    elif len(ag) == best_len:
                        best_rules.extend(rules)
        return best_rules if best_len >= 0 else wildcard_rules

    def can_fetch(self, user_agent: str, path: str) -> bool:
        if not path.startswith("/"):
            path = "/" + path
        best_match_len = -1
        best_allow = True

        for directive, pattern in self._select_rules(user_agent):
            if self._compile_pattern(pattern).match(path):
                plen = len(pattern)
                is_allow = directive == "allow"
                if plen > best_match_len or (plen == best_match_len and is_allow):
                    best_match_len = plen
                    best_allow = is_allow
        return best_allow
```

One nuance from RFC 9309: `Crawl-delay` is outside the standard, though sites like Wikipedia and GitHub still publish it. If you choose to honor `Crawl-delay`, parse it alongside `Allow` and `Disallow` and use it as the floor interval for that host's frontier queue.

## Per-host min-heap frontiers and AIMD rate control

Once a URL passes `robots.txt` filtering, route it into a per-host deque and track each host's `next_allowed` timestamp inside a min-heap (`heapq`). When a host triggers a `429` or returns [`Retry-After`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/429), only that host moves further down the heap while workers keep draining other hosts at full speed.

Inside each host controller, AIMD replaces manual rate guessing with closed-loop feedback (the same general idea behind TCP congestion control and adaptive autoscalers like [Crawlee's blocking controls](https://crawlee.dev/docs/guides/avoid-blocking)):

- Every clean window of `200 OK` responses nudges the host's rate up linearly (`rps += 1.0`).
- A single `429 Too Many Requests` response cuts the host's rate in half (`rps *= 0.5`), parses `Retry-After` whether the origin sends integer delta-seconds (`"1"`) or an RFC 9110 HTTP-date (`"Wed, 25 Nov 2026 02:00:10 GMT"`), and delays the next pop from that host until the cooldown expires.

```python
from collections import deque
from datetime import timezone
from email.utils import parsedate_to_datetime
import heapq
import time
import urllib.error
import urllib.parse
import urllib.request


def parse_retry_after(value: str, now_ts: float = None) -> float:
    if not value:
        return 0.0
    value = value.strip()
    if value.isdigit():
        return float(value)
    try:
        dt = parsedate_to_datetime(value)
        if dt.tzinfo is None:
            dt = dt.replace(tzinfo=timezone.utc)
        now = now_ts if now_ts is not None else time.time()
        return max(0.0, dt.timestamp() - now)
    except (TypeError, ValueError):
        return 0.0


class AIMDHostController:
    def __init__(
        self,
        initial_rps: float = 20.0,
        min_rps: float = 2.0,
        max_rps: float = 30.0,
        additive_step: float = 1.0,
        beta: float = 0.5,
    ):
        self.rps = initial_rps
        self.min_rps = min_rps
        self.max_rps = max_rps
        self.additive_step = additive_step
        self.beta = beta
        self.next_allowed = 0.0
        self.ok_streak = 0

    def on_success(self, now: float) -> None:
        self.ok_streak += 1
        if self.ok_streak >= max(1, int(self.rps)):
            self.rps = min(self.max_rps, self.rps + self.additive_step)
            self.ok_streak = 0
        self.next_allowed = max(self.next_allowed, now + (1.0 / self.rps))

    def on_throttle(self, now: float, retry_after_sec: float = 0.0) -> None:
        self.ok_streak = 0
        self.rps = max(self.min_rps, self.rps * self.beta)
        cooldown = max(retry_after_sec, 1.0 / self.rps)
        self.next_allowed = max(self.next_allowed, now + cooldown)


def crawl_with_aimd_frontier(urls: list) -> dict:
    queues = {}
    controllers = {}
    for url in urls:
        host = urllib.parse.urlparse(url).netloc
        queues.setdefault(host, deque()).append(url)
        if host not in controllers:
            controllers[host] = AIMDHostController()

    heap = [(0.0, host) for host in queues]
    heapq.heapify(heap)
    completed = 0
    throttled = 0

    while heap:
        ready_at, host = heapq.heappop(heap)
        now = time.monotonic()
        if ready_at > now:
            time.sleep(ready_at - now)
        if not queues[host]:
            continue

        url = queues[host].popleft()
        ctrl = controllers[host]
        try:
            with urllib.request.urlopen(url, timeout=10) as resp:
                resp.read()
                completed += 1
                ctrl.on_success(time.monotonic())
        except urllib.error.HTTPError as err:
            if err.code == 429:
                throttled += 1
                ra = parse_retry_after(err.headers.get("Retry-After", ""))
                ctrl.on_throttle(time.monotonic(), retry_after_sec=ra)
                queues[host].appendleft(url)
            else:
                raise

        if queues[host]:
            heapq.heappush(heap, (ctrl.next_allowed, host))

    return {"completed": completed, "throttled": throttled}
```

## Benchmark: naive worker pool vs AIMD frontier

I tested both strategies against a local HTTP server enforcing a sliding-window ceiling of `12` requests per second (`15` ms origin latency, returning HTTP `429` with `Retry-After: 1` whenever the window filled) across `60` URLs:

| Strategy | Total requests sent | `200 OK` | `429` rejections | Wasted traffic share | Wall time |
|---|---|---|---|---|---|
| Fixed pool (8 workers, 80 ms retry) | 426 | 60 | 366 | 85.9% | 4.18 s |
| AIMD + `Retry-After` frontier (starts at 20 req/s) | 61 | 60 | 1 | 1.6% | 7.33 s |

In a permissive local harness that never escalates repeated `429`s into a ban, the fixed 8-worker pool finished faster (`4.18` s vs `7.33` s) by brute-forcing `426` requests to get `60` pages back. On a real WAF or metered residential proxy, firing `366` rejected requests in four seconds burns proxy traffic and promotes your IP subnet from a soft throttle into a hard `403` block or CAPTCHA challenge. The AIMD frontier took one `429` probe while starting above the server's unknown limit, cut its pace in half, honored `Retry-After: 1`, and finished all `60` pages in `61` total requests (1.6% overhead).

## Choosing where to run your own frontier

A polite RFC 9309 + AIMD frontier works well across open documentation sites, government registries, news archives, and multi-domain e-commerce crawls where you control the user-agent identity and want predictable, low-error throughput.

Closed social platforms operate under tighter rules: their `robots.txt` policies restrict general crawling, rate limits track per-session tokens alongside IP reputation, and `Retry-After` headers are often omitted during silent throttling. When a data pipeline needs social metrics alongside open web crawls, keep your custom frontier on open sites and route social collection through managed pay-per-event APIs such as [TikTok Hashtag API](https://apify.com/novi/tiktok-hashtag-api?fpr=7hce1m), [TikTok Trend API](https://apify.com/novi/tiktok-trend-api?fpr=7hce1m), or [TikTok Scraper](https://apify.com/xtdata/tiktok-scraper?fpr=7hce1m). That keeps your crawler compliant where `robots.txt` governs access and avoids maintaining custom session rotation on platforms that reject standard HTTP bots.

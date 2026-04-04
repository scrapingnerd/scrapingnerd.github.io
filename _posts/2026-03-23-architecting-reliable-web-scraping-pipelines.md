---
layout: post
title: "Architecting Reliable Web Scraping Pipelines: From HTTP to DB"
permalink: /architecture/architecting-reliable-web-scraping-pipelines/
categories: [Architecture, Data Pipeline]
tags: [playwright, postgresql, python]
---

Building an enterprise-grade web scraping application means leaving behind single-run scripts and architecting a resilient data pipeline. This guide explores the technical lifecycle of building a production-ready web scraper, using a modern price tracker as our architectural baseline.

## Data Modeling and Scope Definition

Before writing extraction logic, you must carefully define your data model and assess the target environment.

### 1. Schema Design
Define a strict schema for the extracted information. For a price tracker, the minimal relational schema could look like:
* **`product_sku`** (VARCHAR): A unique identifier, ideally the raw SKU.
* **`product_name`** (VARCHAR): The raw string of the item.
* **`price_cents`** (INTEGER): Storing prices as integer cents avoids floating-point inaccuracies.
* **`scraped_at`** (TIMESTAMP WITH TIME ZONE): The exact time of the scrape.

### 2. Payload Assessment
Analyze the network traffic of your target using browser developer tools (Network tab).
* **Static Payloads (SSR):** If data is in the initial HTML payload, lean on high-performance async HTTP clients.
* **Dynamic Payloads (CSR):** If data loads via subsequent XHR APIs or relies on React/Vue hydration, you will need headless browser automation to execute the JavaScript scope.

### 3. Compliance and Politeness
Respect operational boundaries to mitigate risks:
* **`robots.txt`**: Adhere to permitted paths and `Crawl-delay` directives.
* **Rate Limiting**: Implement asynchronous token buckets (e.g., `aiolimiter` in Python) to control outbound concurrency.
* **Terms of Service**: Avoid authenticating unless explicitly allowed, and do not extract PII.

## The Modern Extraction Stack

Switching from legacy scripts to a production stack is necessary for speed and evasion.

### Handling Static Contexts
* **Async HTTP/2:** Opt for **`httpx`** over `requests`. True multiplexing via HTTP/2 helps avoid basic protocol-level fingerprinting.
* **AST Parsing:** Use **`selectolax`** or **`lxml`** for DOM traversal. They wrap fast C-level parsers, handling massive HTML payloads with minimal memory overhead.

### Handling Dynamic Contexts
* **Headless Browsers:** Use **Playwright**. It offers better auto-waiting and network interception capabilities compared to Selenium.
* **Evasion Techniques:** Vanilla headless browsers fail against modern WAFs (Datadome, Cloudflare). You must patch them:
  
  ```python
  from playwright.async_api import async_playwright
  from playwright_stealth import stealth_async

  async def fetch_dynamic_content(url):
      async with async_playwright() as p:
          browser = await p.chromium.launch(headless=True)
          page = await browser.new_page()
          await stealth_async(page) # Masks navigator.webdriver
          await page.goto(url)
          content = await page.content()
          await browser.close()
          return content
  ```

## Logic: Extraction and Normalization

### 1. Network Evasion
If making direct HTTP requests, your networking layer needs careful tuning:
* Use tools like **`curl_cffi`** to spoof TLS fingerprints (JA3/JA4) to match Chrome or Safari.
* Implement **Proxy Rotation**. A robust pipeline uses a rotating pool of Residential IPs, retrying failed requests (HTTP 403, 429) via a different node automatically.

### 2. Robust DOM Traversal
Brittle selectors break pipelines. Use resilient **XPath** or **CSS** queries.
* Avoid: `div.content > span.price-text-3x2` 
* Prefer: `//*[@data-testid='price-container']//span` or similar semantic markers.

### 3. Normalization logic
Transform raw strings into strict types before DB insertion:
```python
import re

def normalize_price_to_cents(raw_string: str) -> int:
    """Converts a string like '$1,299.99' to 129999"""
    match = re.search(r'[\d,]+\.\d{2}', raw_string)
    if not match:
        raise ValueError("No valid price found")
    clean_float = float(match.group().replace(',', ''))
    return int(clean_float * 100)
```

## Persistence and Orchestration

### 1. The Database Layer
* **PostgreSQL:** The gold standard for structured scraping. Use `INSERT ... ON CONFLICT (product_sku) DO UPDATE` (Upsert) patterns to keep track of the latest prices effortlessly.
* **MongoDB/JSONB:** Use document stores or Postgres JSONB columns if extracting highly variable metadata (e.g., differing spec sheets across e-commerce categories).

### 2. Pipeline Orchestration
* **Scheduling:** While Linux `cron` is fine for side projects, production requires tools like **Apache Airflow** or **Prefect** to manage run DAGs, retries, and backfills. For a fully managed experience, explore our collection of pre-built [scraping tools](/tools/) that handle scheduling, proxy rotation, and evasion automatically.
* **Alerting:** Implement a threshold check post-insertion. If `new_price < threshold`, dispatch an asynchronous event to a **Slack Webhook** or email service (SendGrid/AWS SES) to alert stakeholders.

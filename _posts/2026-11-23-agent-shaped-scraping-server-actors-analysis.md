---
layout: post
title: "The Agent-Shaped Web-Data Stack: What Apify's Server Actors Say About Where Scraping Is Going"
date: 2026-11-23 07:20:00 +0700
permalink: /apify/agent-shaped-scraping-server-actors/
categories: [Data Engineering, Analysis]
tags: [Web Scraping, Apify, Server Actors, AI Agents, TikTok, Twitter, API Security]
author: Scraping Nerd
description: "An analyst's read on Apify's Server Actors: what the request/response model changes, where batch TikTok and X scrapers still win, how I'd measure the difference, and why agents holding tokens raise the stakes."
---

# The Agent-Shaped Web-Data Stack: What Apify's Server Actors Say About Where Scraping Is Going

For about a decade, a scraper has been a job. You hand it input, it runs for a while, and it leaves rows in a table you query later. On November 12 Apify announced general availability for Server Actors, and it's the clearest sign I've seen that a large platform now expects a different caller on the other end: an AI agent that wants one answer, right now, inside a tool call.

I think that changes how scraping is priced and how it fails, and it changes who holds the keys. Not all of those changes are as friendly as the launch posts make them sound.

## The numbers, and how much weight they can carry

The growth figures came from the company at its first user conference, Run, and were reported by [VMblog](https://vmblog.com/news/apify-unveils-new-ai-agent-capabilities-and-server-actors-to-support-growing-demand/). Apify says Actor executions went from 50 million a week at the end of 2025 to 267 million a week in October 2026. It says the Store holds more than 100,000 Actors from more than 4,500 developers, and that creators earn more than $1.7M a month. It also says its plugin in Claude's directory was among the 50 most-used as of October 31, with more than 57,000 lifetime installs and over 12,000 active accounts.

Nobody has audited any of that, so I read it the way I'd read any vendor's conference slide. The plugin numbers are more useful than they look, though. Going from 57,000 installs to 12,000 active accounts tells you most people who try an agent integration drop it, and the ones who stay are a smaller, more serious group. And if agents really are calling scrapers many times for small answers, instead of humans scheduling a few large runs, weekly executions is the metric you'd expect to balloon.

CEO Jan Curn put the bet this way: "As AI code generation drives the cost of building software and workflows toward zero, discovery and distribution become the real challenge." Put that next to Apify AI, which the company says can now "execute code, orchestrate multiple Actors, and build entirely new Actors using natural-language instructions." If anyone can generate a scraper, Apify wants to be where agents go to find one and call it.

## From job-plus-dataset to request/response

According to [Apify's own explainer](https://blog.apify.com/introducing-server-actors/), Server Actors are a rename of what used to be called Actor Standby mode. A regular Actor writes results to a dataset (the blog's example is scraping 1,000 Facebook pages overnight). A Server Actor returns results in the HTTP response and stores nothing.

The [docs](https://docs.apify.com/platform/actors/running/server-actors) set out the contract. Each Server Actor gets a URL of the form `https://<username>--<actor-name>.apify.actor`, you authenticate with a bearer token, and the platform strips that token before the request reaches the Actor's code. The first response has to arrive within 5 minutes. If no run can take your request within 2 minutes, you get HTTP 429. Your account's rate limit comes back in an `X-RateLimit-Limit` header.

The older way to get data back synchronously is [`run-sync-get-dataset-items`](https://docs.apify.com/api/v2/act-run-sync-get-dataset-items-post). It starts a full run, waits up to 300 seconds, and returns 408 if the run takes longer. So you could always fake request/response on Apify. You paid for a cold run each time and hoped it finished inside the window.

I suspect people will underrate what statelessness costs them. With no dataset, there's no record of what the agent saw. If an agent tells you a creator's last ten videos averaged some number of views, you can't check that unless your own application logged the response. A chatbot answering a passing question can live with that. A number you'll have to defend in a client report can't.

The pricing models push in different directions. Multi-tenant Server Actors are pay-per-event only and, per Apify, don't count against your memory or concurrency limits. Single-tenant ones bill platform usage, idle time included, plus events. Multi-tenant is the cheap default for spiky agent traffic; single-tenant is what you pick when you need isolation and will pay for a warm process sitting around. The Store now has "Multi Tenant Server" and "Single Tenant Server" categories, and creators can expose MCP through `webServerMcpPath` and an OpenAPI description through `webServerSchema`, which lets an agent work out what the endpoint does without a human reading the README.

## Where batch Actors still win

Most of my work is TikTok and X data, and none of the Actors I lean on for it are Server Actors. [xtdata/tiktok-scraper](https://apify.com/xtdata/tiktok-scraper?fpr=7hce1m) covers trending, music and user videos, [xtdata/tiktok-comment-scraper](https://apify.com/xtdata/tiktok-comment-scraper?fpr=7hce1m) pulls comments and replies, and [xtdata/twitter-x-scraper](https://apify.com/xtdata/twitter-x-scraper?fpr=7hce1m) and [novi/tiktok-scraper-ultimate](https://apify.com/novi/tiktok-scraper-ultimate?fpr=7hce1m) round it out. All of them are batch Actors, called as async runs or through run-sync, and I don't see that as a gap.

Bulk is the obvious reason. Pulling every comment under a viral video, or a few thousand tweets for a keyword, is a job with a natural end and a large output. Pushing that through a single HTTP response with a 5-minute deadline on the first byte gains you nothing.

Freshness windows are a less obvious one. The X scraper takes `start` and `end` dates plus a `sort` of Top, Latest or Both, and [novi/advanced-search-tiktok-api](https://apify.com/novi/advanced-search-tiktok-api?fpr=7hce1m) takes a `publishTime` window from YESTERDAY up to SIX_MONTH. Those inputs are built for "what happened in this window" questions, which you answer once, store, and compare against next week's window. That comparison only works if you kept last week's dataset. A batch run leaves one you can re-query, diff and hand to a colleague, and for trend work on TikTok or X I'd take that over a first result that arrives a few seconds sooner.

Prices at the time of writing follow the same split. On the free tier the batch scrapers charge per item: $0.0004 per video on xtdata/tiktok-scraper, $0.0003 per comment, $0.0003 per video on tiktok-scraper-ultimate, and $0.005 per tweet on the X scraper, which drops to $0.00025 on GOLD and above. Apify's own multi-tenant servers charge per call: $0.005 per result for [apify/web-search](https://apify.com/apify/web-search?fpr=7hce1m) and $0.0015 per fetch for [apify/web-fetch](https://apify.com/apify/web-fetch?fpr=7hce1m). The first set is priced for volume and the second for lookups.

## How I'd measure the difference

I haven't run this benchmark, so there are no results here, only the method I'd use to get a real answer on latency and cost for TikTok search.

Start with a fixed set of keywords, around 20, mixing popular and niche terms. On the batch side, call novi/advanced-search-tiktok-api through `run-sync-get-dataset-items` with the minimum `limit` of 30, a fixed `region` and `sortType`, and a `publishTime` of YESTERDAY. On the server side, none of the TikTok Actors I use run as servers, so the fair version uses a multi-tenant server such as apify/web-search on equivalent queries as a reference for the request/response model, clearly labeled as a different data source. Run each keyword at several times of day, both cold and back-to-back.

For each call, record time to first byte, total wall time, HTTP status (408s on the batch side, 429s on the server side), result count, and the charged events from the billing view. Report the p95 next to the median, because an agent waiting on a tool call is stalled by the slow tail. If someone ships a real TikTok search Server Actor, swap it in and the comparison becomes direct.

## Agents holding tokens

Every one of these endpoints takes an Apify API token. Once that token goes into an agent's tool configuration, copies of it end up in prompts, logs, plugin settings and anything else the agent framework touches, and each copy is one more place it can leak from.

Apify's November incident makes a reasonable case study. [BleepingComputer](https://www.bleepingcomputer.com/news/security/apify-rotates-api-tokens-prompts-2fa-resets-after-breach/) reported unauthorized access on November 9 and 10, detected the next day at 12:13 UTC. Apify rotated API tokens and Apify Proxy passwords, reset sessions, required users to reset 2FA, and revoked GitHub OAuth connections, affecting around 80,000 customers. Reports differ on whether customer data was accessed: the initial report said there was no sign platform data was affected, and later secondary coverage claims some was. There's no post-mortem yet, so check [status.apify.com](https://status.apify.com) before repeating either version.

Whatever the final account says, a forced rotation hurts most when one token sits in a dozen agent configs. [Scoped tokens](https://blog.apify.com/scoped-api-tokens/), available since September 2024, limit the damage. You can restrict them per account or per resource, down to Run, Build or Read permissions on a single dataset or key-value store, from Console under API & Integrations with "Limit token permissions." Server Actors accept them, so an agent that only needs to call one Actor can hold a token that does nothing else.

## What this means for TikTok and X analytics

My read is that the stack is splitting in two. On top sits an agent-facing layer: stateless, fast to answer, priced per call, good for "what is this account" or "fetch this page." Underneath, batch Actors keep doing windowed pulls of posts, comments and tweets into datasets that feed dashboards.

People will judge freshness by the top layer, because that's where they ask questions. Whether they can trust the answers depends on the bottom layer, where the dated record lives. The setups I'd bet on run agents for lookups through servers, keep scheduled batch Actors writing history underneath, and give each side its own scoped token so a leak on one doesn't take out the other.

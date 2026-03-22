---
layout: post
title: "Part 4: OAuth2 authentication and YouTube Data API uploads"
date: 2026-03-27 08:00:00 +0700
categories: [Systems Engineering, Video Automation, Python]
tags: [youtube-api, python, oauth2, automation, cron]
permalink: /tiktok/tiktok-youtube-part4-scheduling-publishing/
---

This is the final component of our distributed pipeline. Having successfully managed the ETL process (Extract from TikTok, Transform via LLM/FFmpeg, Load to disk in [Part 3](/tiktok/tiktok-youtube-part3-ai-narration/)), we now map the normalized `.mp4` chunks into a destination platform: the YouTube Data API v3. 

> **Series Navigation**
> - [Part 1: Interfacing with the Apify Python SDK](/tiktok/tiktok-youtube-part1-searching-api/)
> - [Part 2: Asynchronous video ingestion and connection pooling](/tiktok/tiktok-youtube-part2-downloading-videos/)
> - [Part 3: LLM script synthesis and FFmpeg concatenation](/tiktok/tiktok-youtube-part3-ai-narration/)
> - **Part 4: OAuth2 authentication and YouTube Data API uploads** ← You are here

## Architecture and dependency context

We require Google's official client binding for proper OAuth authorization flows (`google-auth-oauthlib`) and the discovery API (`google-api-python-client`) for endpoint definitions.

```bash
pip install google-auth google-auth-oauthlib google-api-python-client
```

## Managing OAuth2 credentials lifecycle

The primary challenge of uploading via YouTube API is maintaining the `access_token` lifecycle without continuous human execution. First, we securely request permissions against the local loopback interface, storing a serialized binary object (`.pickle`) containing the persistent `refresh_token`.

```python
# google_auth_handler.py
import pickle
from pathlib import Path

from google.auth.transport.requests import Request
from google_auth_oauthlib.flow import InstalledAppFlow
from googleapiclient.discovery import build

SCOPES = ["https://www.googleapis.com/auth/youtube.upload"]
TOKEN_CACHE = Path("credentials/youtube_token.pickle")
CLIENT_SECRET = Path("credentials/client_secret.json")

def bootstrap_oauth_service():
    """Initializes the YouTube API interface, rotating tokens automatically."""
    creds = None
    
    # Load previously serialized tokens to avoid continuous user authorization
    if TOKEN_CACHE.exists():
        with open(TOKEN_CACHE, "rb") as token:
            creds = pickle.load(token)

    if not creds or not creds.valid:
        # Transparently solicit a new access_token if the short-lived token expired
        if creds and creds.expired and creds.refresh_token:
            creds.refresh(Request())
        else:
            # Cold-start loopback auth
            flow = InstalledAppFlow.from_client_secrets_file(
                str(CLIENT_SECRET), SCOPES
            )
            creds = flow.run_local_server(port=8090)

        # Serialize
        TOKEN_CACHE.parent.mkdir(parents=True, exist_ok=True)
        with open(TOKEN_CACHE, "wb") as token:
            pickle.dump(creds, token)

    return build("youtube", "v3", credentials=creds)
```

## Resumable media chunks and standardizing the upload API

Video uploads fail unpredictably. Standard POST bodies break down over 5MB. Thus, we utilize `MediaFileUpload` to configure chunked, resumable POST transfers.

```python
# sync_youtube.py
from googleapiclient.http import MediaFileUpload
from google_auth_handler import bootstrap_oauth_service

def transmit_artifact(youtube_service, video_filepath: str, metadata: dict):
    """Executes a resumable file transmission of an `.mp4` artifact."""
    
    # Define request payload using YouTube's Data Schema
    request_body = {
        "snippet": {
            "title": metadata.get("title"),
            "description": metadata.get("description"),
            "tags": metadata.get("tags", []),
            "categoryId": "25" # Explicitly cast to 'News & Politics' category
        },
        "status": {
            "privacyStatus": "private", # Stage immediately, authorize later
            "selfDeclaredMadeForKids": False
        }
    }

    # Transmit via 10 MB contiguous chunks
    chunk_stream = MediaFileUpload(
        video_filepath,
        mimetype="video/mp4",
        resumable=True,
        chunksize=10_485_760 
    )

    request = youtube_service.videos().insert(
        part="snippet,status",
        body=request_body,
        media_body=chunk_stream
    )

    # Poll server transmission rate
    response = None
    while response is None:
        status, response = request.next_chunk()
        if status:
            print(f"Transmission progress: {int(status.progress() * 100)}%")

    print(f"Artifact persisted: https://youtu.be/{response['id']}")
```

## Distributed orchestration vs generic cron logic

While a monolithic `bash` script or basic `cron` daemon can wrap this four-step pipeline, serious deployments rely on containerized execution.

**Apify Webhooks**: Using Apify as a task queue, you can bind a web endpoint to the [Advanced TikTok Search API Actor](https://apify.com/novi/advanced-search-tiktok-api?fpr=7hce1m) directly. Post-execution, the webhook triggers a downstream CI/CD worker (e.g., GitHub Actions or AWS Lambda) which executes this normalized codebase sequentially. 

By pushing the extraction layer onto a managed platform and keeping the transform and load steps within a localized microservice container, you fully isolate volatile web scraping states from stable application logic.

> **Series Navigation**
> - [Part 1: Interfacing with the Apify Python SDK](/tiktok/tiktok-youtube-part1-searching-api/)
> - [Part 2: Asynchronous video ingestion and connection pooling](/tiktok/tiktok-youtube-part2-downloading-videos/)
> - [Part 3: LLM script synthesis and FFmpeg concatenation](/tiktok/tiktok-youtube-part3-ai-narration/)
> - **Part 4: OAuth2 authentication and YouTube Data API uploads** ← You are here

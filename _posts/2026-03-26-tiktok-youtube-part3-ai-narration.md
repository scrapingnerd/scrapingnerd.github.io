---
layout: post
title: "Part 3: LLM script synthesis and FFmpeg concatenation"
date: 2026-03-26 08:00:00 +0700
categories: [Systems Engineering, Video Automation, Python]
tags: [openai, ffmpeg, python, pipeline]
permalink: /tiktok/tiktok-youtube-part3-ai-narration/
---

This is Part 3 of our pipeline series. (See the [Architecture Overview](/tiktok/youtube-news-channel-tiktok-data/) for context.) Having isolated independent `.mp4` chunks in our local filesystem from [Part 2](/tiktok/tiktok-youtube-part2-downloading-videos/), we must now parse metadata via OpenAI and concatenate the sequence using FFmpeg.

> **Series Navigation**
> - [Part 1: Interfacing with the Apify Python SDK](/tiktok/tiktok-youtube-part1-searching-api/)
> - [Part 2: Asynchronous video ingestion and connection pooling](/tiktok/tiktok-youtube-part2-downloading-videos/)
> - **Part 3: LLM script synthesis and FFmpeg concatenation** ← You are here
> - [Part 4: OAuth2 authentication and YouTube Data API uploads](/tiktok/tiktok-youtube-part4-scheduling-publishing/)

## Engineering dependencies

We interface with OpenAI's asynchronous API wrapper and the command-line FFmpeg application. FFmpeg operations are delegated via Python's `subprocess` for raw control over transcode flags.

```bash
pip install openai
```
*Ensure the FFmpeg binary is installed on your OS environment (`sudo apt install ffmpeg` or `brew install ffmpeg`).*

## Prompt engineering the LLM component

We construct a narrative bridging the disparate `.mp4` clips by pushing the accompanying `desc` metadata to an LLM context window.

```python
# synthesize_script.py
import json
import os
from openai import OpenAI

SYSTEM_PROMPT = """
You are a factual, un-biased news synthesis engine. Given a JSON array of localized multimedia captions originating from social media on a specific geographic event, synthesize a tight 60-second broadcast script. Return raw text without markdown headers, HTML, commentary, or social media references.
"""

def synthesize_broadcast(api_key: str, topic: str, context: list[str]) -> str:
    """Invokes OpenAI ChatGPT to generate continuous editorial."""
    client = OpenAI(api_key=api_key)
    
    # Prune elements beyond context threshold
    pruned_context = context[:20] 
    
    payload = f"Topic Context: {topic}\n\nIngestion Metadata:\n" 
    payload += "\n".join(f"- {c}" for c in pruned_context)
    
    resp = client.chat.completions.create(
        model="gpt-4o-mini",
        temperature=0.3, # Low variance
        messages=[
            {"role": "system", "content": SYSTEM_PROMPT},
            {"role": "user", "content": payload}
        ]
    )
    return resp.choices[0].message.content

def synthesize_audio(api_key: str, script: str, out_path: str):
    """Invokes the neural TTS engine."""
    client = OpenAI(api_key=api_key)
    with client.audio.speech.with_streaming_response.create(
        model="tts-1",
        voice="onyx",
        input=script,
    ) as response:
        response.stream_to_file(out_path)
```

By strictly managing the `temperature` parameter and heavily biasing the systemic prompt against colloquialisms, we extract objective signals from unstructured social streams.

## Demuxing and concatenating via FFmpeg

With `narration.mp3` isolated, we construct an FFmpeg intermediate `.txt` concatenation file, enforce standardization via H.264 profiles (`libx264`), and map the audio stream natively, skipping arbitrary format transcodings.

```python
# build_artifacts.py
import subprocess
from pathlib import Path

def concat_and_mux(video_paths: list[Path], audio_path: Path, out_path: Path):
    """
    Standardizes aspect ratios, drops source audio channels, concatenates video tracks, 
    and muxes the secondary audio track.
    """
    # 1. Build FFmpeg concat instruction file
    concat_txt = Path("concat.txt")
    with open(concat_txt, "w") as f:
        for vp in video_paths:
            f.write(f"file '{vp.absolute()}'\n")

    # 2. Transcode parameters to normalize resolutions to 1080p 16:9
    tmp_vid = Path("tmp_vid.mp4")
    subprocess.run([
        "ffmpeg", "-y", "-f", "concat", "-safe", "0",
        "-i", str(concat_txt),
        "-c:v", "libx264", "-crf", "24", "-preset", "veryfast",
        "-r", "30", "-an", # Drop source audio track (-an)
        "-vf", "scale=1920:1080:force_original_aspect_ratio=decrease,pad=1920:1080:(ow-iw)/2:(oh-ih)/2",
        str(tmp_vid)
    ], check=True)

    # 3. Mux audio and video tracks, truncate at shortest stream
    subprocess.run([
        "ffmpeg", "-y",
        "-i", str(tmp_vid), "-i", str(audio_path),
        "-c:v", "copy", "-c:a", "aac", "-b:a", "192k",
        "-shortest",
        str(out_path)
    ], check=True)

    # Teardown artifacts
    concat_txt.unlink()
    tmp_vid.unlink()
```

This structural normalization guarantees that vertical TikTok codecs are padded appropriately without mutating hardware profiles, minimizing the container rendering overhead.

In [Part 4: OAuth2 authentication and YouTube Data API uploads](/tiktok/tiktok-youtube-part4-scheduling-publishing/), we discuss orchestrating the final stage via the `google-api-python-client`.

> **Series Navigation**
> - [Part 1: Interfacing with the Apify Python SDK](/tiktok/tiktok-youtube-part1-searching-api/)
> - [Part 2: Asynchronous video ingestion and connection pooling](/tiktok/tiktok-youtube-part2-downloading-videos/)
> - **Part 3: LLM script synthesis and FFmpeg concatenation** ← You are here
> - [Part 4: OAuth2 authentication and YouTube Data API uploads](/tiktok/tiktok-youtube-part4-scheduling-publishing/)

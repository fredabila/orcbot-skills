---
name: elevenlabs
description: >
  Use this skill whenever an agent needs to interact with ElevenLabs in any way.
  Covers all ElevenLabs REST API capabilities: text-to-speech (TTS), streaming audio,
  voice cloning (instant and professional), speech-to-text transcription, voice design
  (create voices from text prompts), managing the voice library, and handling long-form
  content like audiobooks or articles. Trigger this skill whenever the user mentions
  ElevenLabs, text-to-speech, TTS, voice cloning, voice generation, audio synthesis,
  speech generation, transcription, or anything voice/audio-AI related — even casually.
  Don't try to wing ElevenLabs tasks from memory; always consult this skill first.
---

# ElevenLabs Skill

A complete guide for agents to use the **ElevenLabs REST API** for voice generation, cloning, transcription, and voice design. No SDK required — all examples use `requests` (Python) or plain `curl`.

---

## 0. Authentication & Setup

All requests require your API key in the `xi-api-key` header.

```python
import requests
import os

API_KEY = os.environ["ELEVENLABS_API_KEY"]  # never hardcode
BASE_URL = "https://api.elevenlabs.io/v1"

HEADERS = {
    "xi-api-key": API_KEY,
    "Content-Type": "application/json",
}
```

Get your API key: **elevenlabs.io → Developers → API Keys**

---

## 1. Models Reference

| Model ID | Best For | Notes |
|----------|----------|-------|
| `eleven_v3` | Highest quality, most expressive | Latest flagship model |
| `eleven_flash_v2_5` | Real-time / low latency (~75ms) | Best for streaming |
| `eleven_multilingual_v2` | Multilingual, nuanced | 32 languages |
| `eleven_turbo_v2_5` | Balanced speed + quality | Good general purpose |
| `eleven_english_sts_v2` | Speech-to-speech (voice changer) | English only |
| `scribe_v1` | Speech-to-text transcription | State-of-the-art STT |

List all available models programmatically:
```python
resp = requests.get(f"{BASE_URL}/models", headers=HEADERS)
for model in resp.json():
    print(model["model_id"], "-", model["name"])
```

---

## 2. Text-to-Speech (TTS)

### Basic TTS — save to file

```python
def text_to_speech(text: str, voice_id: str, output_path: str, model_id="eleven_v3"):
    url = f"{BASE_URL}/text-to-speech/{voice_id}"
    payload = {
        "text": text,
        "model_id": model_id,
        "voice_settings": {
            "stability": 0.5,           # 0.0–1.0: lower = more expressive
            "similarity_boost": 0.75,   # 0.0–1.0: higher = closer to original voice
            "style": 0.0,               # 0.0–1.0: style exaggeration (v3+)
            "use_speaker_boost": True,  # enhances similarity
        },
        "output_format": "mp3_44100_128",  # see formats below
    }
    resp = requests.post(url, json=payload, headers=HEADERS)
    resp.raise_for_status()
    with open(output_path, "wb") as f:
        f.write(resp.content)
    print(f"Saved: {output_path}")
    # Cost tracking from response headers:
    print(f"Characters used: {resp.headers.get('x-character-count')}")

# Example
text_to_speech("Hello, world! This is ElevenLabs.", "21m00Tcm4TlvDq8ikWAM", "output.mp3")
```

### Output formats

| Format | Quality | Plan Required |
|--------|---------|---------------|
| `mp3_22050_32` | Low | Free |
| `mp3_44100_64` | Standard | Free |
| `mp3_44100_128` | Good | Starter+ |
| `mp3_44100_192` | High | Creator+ |
| `pcm_44100` | Lossless | Pro+ |
| `ulaw_8000` | Telephony (Twilio) | Starter+ |

### Streaming TTS (real-time)

Use `/stream` endpoint for real-time playback — starts audio before full generation completes.

```python
def stream_tts(text: str, voice_id: str, output_path: str, model_id="eleven_flash_v2_5"):
    url = f"{BASE_URL}/text-to-speech/{voice_id}/stream"
    payload = {
        "text": text,
        "model_id": model_id,
        "voice_settings": {"stability": 0.5, "similarity_boost": 0.75},
        "output_format": "mp3_44100_128",
    }
    with requests.post(url, json=payload, headers=HEADERS, stream=True) as resp:
        resp.raise_for_status()
        with open(output_path, "wb") as f:
            for chunk in resp.iter_content(chunk_size=4096):
                f.write(chunk)
    print(f"Streamed to: {output_path}")
```

### SSML — controlling speech delivery

ElevenLabs supports `<break>` and `<phoneme>` tags inline in text:

```python
text = """
Welcome to the future of voice AI.
<break time="1.5s" />
Let me explain how this works.
<phoneme alphabet="ipa" ph="ˈɛləvənlæbz">ElevenLabs</phoneme> uses neural networks.
"""
```

---

## 3. Long-form Content (Audiobooks / Articles)

For content over ~2,500 characters, split into chunks with context overlap for natural continuity.

```python
def chunk_text(text: str, max_chars=2000) -> list[str]:
    """Split text at sentence boundaries."""
    import re
    sentences = re.split(r'(?<=[.!?])\s+', text)
    chunks, current = [], ""
    for sentence in sentences:
        if len(current) + len(sentence) <= max_chars:
            current += " " + sentence
        else:
            if current:
                chunks.append(current.strip())
            current = sentence
    if current:
        chunks.append(current.strip())
    return chunks

def long_form_tts(text: str, voice_id: str, output_dir: str, model_id="eleven_multilingual_v2"):
    import os, time
    os.makedirs(output_dir, exist_ok=True)
    chunks = chunk_text(text)
    files = []

    for i, chunk in enumerate(chunks):
        # Pass previous/next text for continuity
        prev_text = chunks[i-1][-500:] if i > 0 else None
        next_text = chunks[i+1][:500] if i < len(chunks)-1 else None

        url = f"{BASE_URL}/text-to-speech/{voice_id}"
        payload = {
            "text": chunk,
            "model_id": model_id,
            "voice_settings": {"stability": 0.6, "similarity_boost": 0.8},
            "previous_text": prev_text,
            "next_text": next_text,
        }
        resp = requests.post(url, json=payload, headers=HEADERS)
        resp.raise_for_status()

        path = f"{output_dir}/chunk_{i:03d}.mp3"
        with open(path, "wb") as f:
            f.write(resp.content)
        files.append(path)
        print(f"  ✅ Chunk {i+1}/{len(chunks)}: {path}")
        time.sleep(0.3)  # be gentle with rate limits

    return files

# Merge all chunks into one file (requires ffmpeg)
def merge_audio(files: list[str], output: str):
    import subprocess
    list_file = "/tmp/filelist.txt"
    with open(list_file, "w") as f:
        for fp in files:
            f.write(f"file '{fp}'\n")
    subprocess.run(["ffmpeg", "-f", "concat", "-safe", "0", "-i", list_file, "-c", "copy", output])
    print(f"Merged: {output}")
```

---

## 4. Voice Library Management

### List your voices

```python
def list_voices():
    resp = requests.get(f"{BASE_URL}/voices", headers=HEADERS)
    resp.raise_for_status()
    for voice in resp.json()["voices"]:
        print(f"{voice['name']:30} | ID: {voice['voice_id']} | Category: {voice['category']}")

list_voices()
```

### Get a specific voice

```python
def get_voice(voice_id: str):
    resp = requests.get(f"{BASE_URL}/voices/{voice_id}", headers=HEADERS)
    return resp.json()
```

### Search shared voice library (10k+ voices)

```python
def search_voices(query: str, gender=None, age=None, accent=None, page_size=10):
    params = {"search": query, "page_size": page_size}
    if gender: params["gender"] = gender         # "male" | "female"
    if age: params["age"] = age                  # "young" | "middle_aged" | "old"
    if accent: params["accent"] = accent         # "american" | "british" | etc.
    
    resp = requests.get(f"{BASE_URL}/voices/shared", params=params, headers=HEADERS)
    for v in resp.json().get("voices", []):
        print(f"{v['name']:30} | {v['voice_id']} | {v.get('description','')[:60]}")
```

### Delete a voice

```python
def delete_voice(voice_id: str):
    resp = requests.delete(f"{BASE_URL}/voices/{voice_id}", headers=HEADERS)
    resp.raise_for_status()
    print(f"Deleted voice: {voice_id}")
```

---

## 5. Voice Cloning

### Instant Voice Clone (IVC) — fast, from short clips

Good quality from 1–3 minutes of clean audio. Available on all paid plans.

```python
def instant_voice_clone(name: str, audio_files: list[str], description="") -> str:
    """
    audio_files: list of paths to .mp3/.wav files
    Returns: voice_id of the new cloned voice
    """
    url = f"{BASE_URL}/voices/add"
    files = [("files", (f, open(f, "rb"), "audio/mpeg")) for f in audio_files]
    data = {"name": name, "description": description}
    
    headers_no_ct = {"xi-api-key": API_KEY}  # no Content-Type — let requests set multipart
    resp = requests.post(url, data=data, files=files, headers=headers_no_ct)
    resp.raise_for_status()
    voice_id = resp.json()["voice_id"]
    print(f"Cloned voice '{name}' — ID: {voice_id}")
    return voice_id

# Tips for best results:
# - Use clear audio with no background noise or music
# - 1–5 minute samples work best
# - Consistent tone/emotion across samples
# - Multiple short clips often better than one long clip
```

### Update a cloned voice

```python
def update_voice(voice_id: str, name: str = None, new_audio_files: list[str] = None):
    url = f"{BASE_URL}/voices/{voice_id}/edit"
    data = {}
    if name: data["name"] = name
    files = []
    if new_audio_files:
        files = [("files", (f, open(f, "rb"), "audio/mpeg")) for f in new_audio_files]
    
    headers_no_ct = {"xi-api-key": API_KEY}
    resp = requests.post(url, data=data, files=files, headers=headers_no_ct)
    resp.raise_for_status()
    print(f"Updated voice: {voice_id}")
```

---

## 6. Voice Design (Create from Text Prompt)

Generate entirely new synthetic voices from a text description — no audio samples needed.

```python
def design_voice_previews(prompt: str, gender: str, age: str, n_previews=3) -> list[dict]:
    """
    Step 1: Generate preview voices from a description.
    gender: "male" | "female" | "neutral"
    age: "young" | "middle_aged" | "old"
    Returns list of {generated_voice_id, audio_base64}
    """
    url = f"{BASE_URL}/text-to-voice/design"
    payload = {
        "voice_description": prompt,
        "text": "Hello! This is a preview of the voice you just created.",
        "gender": gender,
        "age": age,
        "loudness": 0,           # -1.0 to 1.0
        "quality": 1,            # 0 (more variety) to 1 (higher quality)
        "auto_generate_text": False,
        "enhance_voice_description": True,  # let AI enrich the prompt
    }
    resp = requests.post(url, json=payload, headers=HEADERS)
    resp.raise_for_status()
    return resp.json()["previews"]

def save_voice_from_preview(generated_voice_id: str, name: str, description="") -> str:
    """
    Step 2: Save a preview as a permanent voice in your library.
    Returns: voice_id
    """
    url = f"{BASE_URL}/text-to-voice"
    payload = {
        "voice_name": name,
        "voice_description": description,
        "generated_voice_id": generated_voice_id,
    }
    resp = requests.post(url, json=payload, headers=HEADERS)
    resp.raise_for_status()
    voice_id = resp.json()["voice_id"]
    print(f"Saved designed voice '{name}' — ID: {voice_id}")
    return voice_id

# Full voice design workflow:
previews = design_voice_previews(
    prompt="A calm, authoritative British male narrator with a deep, warm tone",
    gender="male",
    age="middle_aged"
)

# Save preview audio to listen to them
import base64
for i, preview in enumerate(previews):
    audio_bytes = base64.b64decode(preview["audio_base64_pcm"])
    with open(f"preview_{i}.mp3", "wb") as f:
        f.write(audio_bytes)
    print(f"Preview {i}: ID = {preview['generated_voice_id']}")

# Pick one and save it permanently
voice_id = save_voice_from_preview(previews[0]["generated_voice_id"], "British Narrator")
```

---

## 7. Speech-to-Text (Transcription)

Uses **Scribe v1** — ElevenLabs' state-of-the-art transcription model with speaker diarization and timestamps.

```python
def transcribe_audio(audio_path: str, language_code=None, diarize=False) -> dict:
    """
    audio_path: path to audio file (.mp3, .wav, .m4a, etc.)
    language_code: e.g. "en", "fr", "de" (auto-detected if None)
    diarize: if True, labels who is speaking
    Returns: full transcription response dict
    """
    url = f"{BASE_URL}/speech-to-text"
    headers_no_ct = {"xi-api-key": API_KEY}
    
    data = {"model_id": "scribe_v1"}
    if language_code:
        data["language_code"] = language_code
    if diarize:
        data["diarize"] = "true"
    
    with open(audio_path, "rb") as f:
        files = {"file": (audio_path, f, "audio/mpeg")}
        resp = requests.post(url, data=data, files=files, headers=headers_no_ct)
    
    resp.raise_for_status()
    return resp.json()

# Usage
result = transcribe_audio("interview.mp3", diarize=True)
print(result["text"])  # full transcript

# With speaker labels (if diarize=True)
for word in result.get("words", []):
    print(f"[{word.get('speaker_id', '?')}] {word['text']} ({word['start']:.2f}s)")
```

---

## 8. Voice Settings Tuning Guide

| Setting | Low (0.0) | High (1.0) | Recommended |
|---------|-----------|------------|-------------|
| `stability` | More expressive, variable | More consistent, stable | 0.4–0.6 for dialogue; 0.6–0.8 for narration |
| `similarity_boost` | More creative interpretation | Closer to original voice | 0.7–0.85 |
| `style` | Neutral delivery | Exaggerated style | 0.0–0.3 (higher can distort) |
| `use_speaker_boost` | Off | On | `true` for cloned voices |

---

## 9. Best Practices

- **Voice selection matters most** — spend time testing voices; it impacts quality more than any setting
- **Use Flash v2.5 for real-time** — 75ms latency, ideal for agents and live interactions
- **Use Multilingual v2 for quality** — best for audiobooks, podcasts, polished content
- **Chunk long content** — pass `previous_text` and `next_text` for seamless continuity between chunks
- **Cache voice IDs** — store them in a config file; don't re-fetch every run
- **Monitor character usage** — read `x-character-count` from response headers to track costs
- **For voice cloning** — use clean, noise-free audio; multiple short clips beat one long clip
- **Rate limit awareness** — add `time.sleep(0.3)` between bulk requests; 429 = too many concurrent requests

---

## 10. Common Popular Voice IDs

| Voice | ID | Style |
|-------|-----|-------|
| Rachel | `21m00Tcm4TlvDq8ikWAM` | Calm, female, American |
| Drew | `29vD33N1CtxCmqQRPOHJ` | Well-rounded, male |
| Clyde | `2EiwWnXFnvU5JabPnv8n` | War veteran, male, American |
| Charlotte | `XB0fDUnXU5powFXDhCwa` | Swedish, female |
| Adam | `pNInz6obpgDQGcFmaJgB` | Deep, male, American |

Find more via `GET /v1/voices` or the ElevenLabs voice library UI.

---

## 11. Troubleshooting

| Error | Meaning | Fix |
|-------|---------|-----|
| `401 Unauthorized` | Bad or missing API key | Check `xi-api-key` header |
| `429 Too Many Requests` | Rate limit hit | Add `time.sleep()` between requests |
| `422 Unprocessable Entity` | Bad request body | Check required fields and formats |
| `voice_id not found` | Wrong voice ID | Use `GET /v1/voices` to list valid IDs |
| Robotic/distorted audio | Bad voice settings | Lower `style`, raise `stability` |
| Cloned voice sounds off | Poor input audio | Use cleaner samples, remove background noise |
| Audio cuts off mid-sentence | Chunk too long | Keep chunks under 2,500 characters |

# Vertex AI Integration Guide

**Project:** Bibo YouTube Video Generator
**Extracted:** 2026-02-22

---

## Overview

VidGen uses multiple Vertex AI services for AI-powered video generation. This document captures integration patterns, model selection strategies, and best practices discovered during development.

---

## Services Used

| Service | Purpose | Models Used |
|---------|---------|-------------|
| **Gemini** | Script generation, metadata, image prompts | 2.5 Flash (default), 2.5 Pro (optional) |
| **Imagen** | AI image generation | imagen-3.0-generate-001 |
| **Cloud TTS** | Script narration | Studio voices (en-US-Studio-O/M) |
| **Cloud Speech-to-Text** | YouTube transcription | Long-running recognition |
| **Cloud Storage** | Large audio file handling | For files >10MB |

---

## Authentication Pattern

### Development (Local)

**Application Default Credentials (ADC):**
```bash
gcloud auth application-default login
```

**Verification:**
```python
from google.cloud import speech

try:
    client = speech.SpeechClient()
    print("✅ Auth OK")
except Exception as e:
    print(f"❌ Auth failed: {e}")
```

### Production (Deployed)

**Service Account JSON:**
```bash
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/service-account.json"
```

**Required Permissions:**
- Vertex AI User
- Speech-to-Text Admin
- Text-to-Speech Admin
- Storage Object Admin

---

## Gemini Integration

### Initialization

```python
import vertexai
from google.ai import generativelanguage as genai
import os

PROJECT_ID = os.getenv("VERTEX_AI_PROJECT")
LOCATION = os.getenv("VERTEX_AI_LOCATION", "us-central1")

vertexai.init(project=PROJECT_ID, location=LOCATION)
```

**Environment Variables:**
- `VERTEX_AI_PROJECT` - Google Cloud project ID
- `VERTEX_AI_LOCATION` - Region (default: us-central1)

### Model Selection Strategy

**Gemini 2.5 Flash (Default):**
- **Use for:** Script generation, metadata, image prompts
- **Strengths:** Fast, cost-effective, consistent output
- **Output quality:** ~1,045 words for script (tested Feb 11, 2026)
- **Cost:** ~$0.01-0.02 per generation

**Gemini 2.5 Pro (Optional Upgrade):**
- **Use for:** Complex topics, higher quality needed
- **Strengths:** Better reasoning, more nuanced output
- **Cost:** ~5x more expensive than Flash
- **When to use:** User requests better quality, topic is complex

**Model Configuration:**
```python
generation_config = {
    "temperature": 0.7,        # Creativity (0.0-1.0)
    "top_p": 0.95,             # Nucleus sampling
    "top_k": 40,               # Top-k sampling
    "max_output_tokens": 8192, # Max response length
}

model = genai.GenerativeModel(
    model_name="gemini-2.5-flash",
    generation_config=generation_config
)
```

### Prompt Engineering Patterns

**Script Generation (Summarization):**
```python
prompt = f"""You are an expert YouTube script writer. Transform the following transcript into an engaging video script.

REQUIREMENTS:
- Length: 800-1200 words
- Style: Conversational, engaging
- Structure: Hook → Main points → Call-to-action
- Tone: [informative/entertaining/educational]

TRANSCRIPT:
{transcript_text}

OUTPUT FORMAT:
[Engaging script text, ready for narration]
"""
```

**Image Prompt Generation:**
```python
prompt = f"""Generate image prompts for a video based on this script.

REQUIREMENTS:
- {num_images} prompts needed
- Each prompt: Detailed visual description
- Style: Cinematic, high-quality, photorealistic
- Aspect ratio: 16:9 (landscape)

SCRIPT:
{script_text}

OUTPUT FORMAT:
Return ONLY a JSON array:
[
  {{"prompt": "Detailed image description 1"}},
  {{"prompt": "Detailed image description 2"}},
  ...
]
"""
```

**Metadata Generation:**
```python
prompt = f"""Generate YouTube metadata for this video script.

SCRIPT:
{script_text}

OUTPUT FORMAT (JSON):
{{
  "titles": [
    "Main title option 1",
    "Main title option 2",
    "Main title option 3"
  ],
  "description": "Full video description with keywords",
  "hashtags": ["tag1", "tag2", "tag3", "tag4", "tag5"]
}}
"""
```

### Response Handling

**Text Extraction:**
```python
response = model.generate_content(prompt)
text_response = response.text
```

**JSON Parsing:**
```python
import json
import re

response_text = response.text

# Extract JSON from markdown code blocks if present
json_match = re.search(r'```json\n(.*?)\n```', response_text, re.DOTALL)
if json_match:
    json_str = json_match.group(1)
else:
    json_str = response_text

data = json.loads(json_str)
```

### Error Handling

**Common Errors:**
```python
from google.api_core import exceptions

try:
    response = model.generate_content(prompt)
except exceptions.ResourceExhausted:
    # Rate limit hit
    print("Rate limit exceeded. Wait and retry.")
except exceptions.InvalidArgument as e:
    # Bad prompt or config
    print(f"Invalid request: {e}")
except Exception as e:
    # Other errors
    print(f"Gemini error: {e}")
```

**Retry Strategy (Not Currently Implemented):**
```python
import time

def generate_with_retry(prompt, max_retries=3):
    for attempt in range(max_retries):
        try:
            return model.generate_content(prompt)
        except exceptions.ResourceExhausted:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt  # Exponential backoff
                time.sleep(wait_time)
            else:
                raise
```

---

## Imagen Integration

### Model Setup

```python
from vertexai.preview.vision_models import ImageGenerationModel

model = ImageGenerationModel.from_pretrained("imagen-3.0-generate-001")
```

### Image Generation Pattern

```python
def generate_image(prompt: str, output_path: str) -> None:
    """Generate single image with Imagen."""

    model = ImageGenerationModel.from_pretrained("imagen-3.0-generate-001")

    # Generate
    images = model.generate_images(
        prompt=prompt,
        number_of_images=1,          # Always 1 for consistency
        aspect_ratio="16:9",          # YouTube landscape
        safety_filter_level="block_some",  # Default safety
        person_generation="allow_adult"    # Allow people in images
    )

    # Save
    images[0].save(output_path)
```

**Parameters:**
- `prompt` - Detailed visual description
- `number_of_images` - Always 1 (generate multiple via loop)
- `aspect_ratio` - "16:9" for YouTube, "1:1" for Instagram, "9:16" for Shorts
- `safety_filter_level` - "block_few" | "block_some" | "block_most"
- `person_generation` - "dont_allow" | "allow_adult"

### Prompt Best Practices

**Good prompts:**
```
"A serene mountain landscape at sunset, golden hour lighting,
photorealistic, cinematic composition, 4K quality, professional photography"
```

**Bad prompts:**
```
"mountains"  # Too vague
"a mountain with a person" # May trigger safety filter
```

**Pattern:** Detailed + style + quality markers

### Batch Generation Pattern

```python
def generate_images_batch(prompts: List[str], output_dir: str) -> None:
    """Generate multiple images sequentially."""

    model = ImageGenerationModel.from_pretrained("imagen-3.0-generate-001")

    for i, prompt in enumerate(prompts):
        output_path = f"{output_dir}/image_{i:04d}.png"

        print(f"Generating image {i+1}/{len(prompts)}...")

        try:
            images = model.generate_images(
                prompt=prompt,
                number_of_images=1,
                aspect_ratio="16:9"
            )

            images[0].save(output_path)

        except Exception as e:
            print(f"Error generating image {i}: {e}")
            # Continue with next image
            continue
```

**Note:** Sequential generation (not parallel) to avoid rate limits.

**Estimated time:** ~5-10 seconds per image.

### Cost Considerations

**Imagen 3.0 Pricing (as of Feb 2026):**
- ~$0.04 per image
- For 1-minute video (6 images @ 10s each): ~$0.24
- For 2-minute video (12 images): ~$0.48

**Cost optimization:**
- Generate fewer images (1 per 15s instead of 10s)
- Reuse images across similar videos
- Cache generated images (not currently implemented)

---

## Cloud Text-to-Speech Integration

### Voice Selection

**Studio Voices (Recommended):**
```python
VOICES = {
    "en-US-Studio-O": "Female, professional",
    "en-US-Studio-M": "Male, professional"
}
```

**Quality comparison:**
- **Standard voices:** Robotic, cheap
- **WaveNet voices:** Better, mid-price
- **Studio voices:** Most natural, premium
- **Neural2 voices:** Latest, not tested

### TTS Pattern (with Chunking)

```python
from google.cloud import texttospeech

def synthesize_speech(text: str, output_file: str, voice: str) -> None:
    """Generate audio with chunking for long text."""

    client = texttospeech.TextToSpeechClient()

    # TTS has 5000 byte limit - chunk if needed
    chunks = split_text_into_chunks(text, max_bytes=4500)

    audio_segments = []

    for chunk in chunks:
        synthesis_input = texttospeech.SynthesisInput(text=chunk)

        voice_params = texttospeech.VoiceSelectionParams(
            language_code="en-US",
            name=voice
        )

        audio_config = texttospeech.AudioConfig(
            audio_encoding=texttospeech.AudioEncoding.MP3,
            speaking_rate=1.0,  # Normal speed
            pitch=0.0           # Normal pitch
        )

        response = client.synthesize_speech(
            input=synthesis_input,
            voice=voice_params,
            audio_config=audio_config
        )

        audio_segments.append(response.audio_content)

    # Combine audio segments
    combined = b''.join(audio_segments)

    with open(output_file, 'wb') as f:
        f.write(combined)
```

### Text Chunking Strategy

```python
def split_text_into_chunks(text: str, max_bytes: int = 4500) -> List[str]:
    """Split text respecting paragraph boundaries."""

    paragraphs = text.split('\n\n')
    chunks = []
    current_chunk = ""

    for para in paragraphs:
        # Check byte size (not character count)
        test_chunk = current_chunk + "\n\n" + para if current_chunk else para

        if len(test_chunk.encode('utf-8')) > max_bytes:
            # Would exceed limit - save current, start new
            if current_chunk:
                chunks.append(current_chunk.strip())
            current_chunk = para
        else:
            current_chunk = test_chunk

    if current_chunk:
        chunks.append(current_chunk.strip())

    return chunks
```

**Key points:**
- Measure in **bytes**, not characters (API limit is bytes)
- Respect paragraph boundaries (don't split mid-paragraph)
- Leave buffer (4500 for 5000 limit accounts for UTF-8 multi-byte chars)

### Audio Duration Calculation

```python
from pydub import AudioSegment

def get_audio_duration(audio_file: str) -> float:
    """Get audio duration in seconds."""

    audio = AudioSegment.from_mp3(audio_file)
    duration_seconds = len(audio) / 1000.0

    return duration_seconds
```

**Used for:** Determining how many images to generate (1 per 10 seconds).

---

## Cloud Speech-to-Text Integration

### Long-Running Recognition Pattern

```python
from google.cloud import speech

def transcribe_audio(audio_file: str) -> str:
    """Transcribe audio (handles large files)."""

    client = speech.SpeechClient()

    # Check file size
    file_size = os.path.getsize(audio_file)

    if file_size > 10_000_000:  # >10MB
        # Upload to GCS
        gcs_uri = upload_to_gcs(audio_file)
        audio = speech.RecognitionAudio(uri=gcs_uri)
    else:
        # Use content directly
        with open(audio_file, 'rb') as f:
            content = f.read()
        audio = speech.RecognitionAudio(content=content)

    config = speech.RecognitionConfig(
        encoding=speech.RecognitionConfig.AudioEncoding.FLAC,
        sample_rate_hertz=16000,
        language_code="en-US",
        enable_automatic_punctuation=True,
        model="video"  # Optimized for video content
    )

    # Long-running operation
    operation = client.long_running_recognize(config=config, audio=audio)

    print("Transcription in progress...")
    response = operation.result(timeout=3600)  # 1 hour max

    # Combine all transcript segments
    transcript = " ".join([
        result.alternatives[0].transcript
        for result in response.results
    ])

    return transcript
```

**Key points:**
- Use `long_running_recognize` for audio >1 minute
- Generous timeout (videos can be long)
- GCS upload for files >10MB (API requirement)
- FLAC encoding for best quality

### GCS Upload Pattern

```python
from google.cloud import storage

def upload_to_gcs(local_file: str) -> str:
    """Upload file to GCS, return URI."""

    bucket_name = os.getenv("GOOGLE_STT_BUCKET")
    client = storage.Client()
    bucket = client.bucket(bucket_name)

    blob_name = f"audio/{Path(local_file).name}"
    blob = bucket.blob(blob_name)

    blob.upload_from_filename(local_file)

    gcs_uri = f"gs://{bucket_name}/{blob_name}"

    return gcs_uri
```

**Cleanup (not currently implemented):**
```python
# Delete from GCS after transcription
blob.delete()
```

---

## Model Selection Guidelines

### When to Use Gemini 2.5 Flash

✅ **Use for:**
- Script generation (most cases)
- Metadata generation
- Image prompt generation
- High-volume/batch processing
- Cost-sensitive projects

### When to Upgrade to Gemini 2.5 Pro

✅ **Use for:**
- Complex technical topics
- Nuanced creative writing
- When Flash output quality insufficient
- User explicitly requests higher quality
- Final polish for important videos

### Cost Comparison

| Task | Flash Cost | Pro Cost | Flash Time | Pro Time |
|------|-----------|----------|------------|----------|
| Script (1000 words) | ~$0.01 | ~$0.05 | ~3s | ~5s |
| Metadata | ~$0.005 | ~$0.02 | ~2s | ~3s |
| Image prompts (6) | ~$0.01 | ~$0.05 | ~3s | ~5s |
| **Total per video** | ~$0.025 | ~$0.12 | ~8s | ~13s |

**Recommendation:** Default to Flash, let user upgrade if needed.

---

## Rate Limiting & Quotas

### Current Approach

**No explicit rate limiting** - rely on Google's built-in limits.

**Observed limits (Feb 2026):**
- Gemini: ~60 requests/minute (not hit in testing)
- Imagen: ~10 images/minute (sequential generation stays under)
- TTS: ~100 requests/minute (not hit)
- Speech-to-Text: Long-running (no per-minute limit)

### Future Improvements

**If hitting rate limits:**
```python
import time
from functools import wraps

def rate_limit(calls_per_minute):
    """Decorator to rate limit function calls."""
    min_interval = 60.0 / calls_per_minute
    last_call = [0.0]

    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_call[0]
            left_to_wait = min_interval - elapsed

            if left_to_wait > 0:
                time.sleep(left_to_wait)

            result = func(*args, **kwargs)
            last_call[0] = time.time()

            return result
        return wrapper
    return decorator

@rate_limit(calls_per_minute=10)
def generate_image(prompt, output):
    # Imagen call
    pass
```

---

## Context Caching (Not Yet Implemented)

**Potential optimization:**
```python
# Cache static portions of prompts
from google.ai.generativelanguage import CachedContent

# Create cached content
cached_system_prompt = CachedContent.create(
    model="gemini-2.5-flash",
    contents=[{
        "role": "system",
        "parts": [{"text": "You are an expert YouTube script writer..."}]
    }],
    ttl=3600  # Cache for 1 hour
)

# Use cached content
model = genai.GenerativeModel(
    model_name="gemini-2.5-flash",
    cached_content=cached_system_prompt
)
```

**Benefit:** Reduce tokens billed for repeated system prompts.

---

## Key Takeaways

1. **Single ecosystem wins:** Google Cloud services integrate seamlessly
2. **Flash is good enough:** Default to cheaper/faster model
3. **Chunk long text:** TTS and API limits require chunking
4. **GCS for large files:** Speech-to-Text requires GCS for >10MB
5. **Sequential over parallel:** Avoid rate limits with sequential processing
6. **Studio voices worth it:** Quality matters for published content
7. **Long timeouts:** Speech-to-Text can take 10-30 minutes

---

_Extracted: 2026-02-22_

# Vertex AI Integrations

## What This Is For

All Vertex AI modalities in one reference: text generation, image generation, TTS, STT, context caching, and the Files API for live context upload. Auth setup for local dev and Cloud Run.

---

## Quick Reference — Auth Setup

### Local Development (ADC)

```bash
gcloud auth application-default login
```

Verify it works:
```python
import vertexai
vertexai.init(project="your-project", location="us-central1")
print("Auth OK")
```

### Production (Cloud Run — Service Account)

Grant the service account these roles:
```bash
PROJECT="your-project"
SA_EMAIL="my-sa@${PROJECT}.iam.gserviceaccount.com"

# Vertex AI user
gcloud projects add-iam-policy-binding $PROJECT \
  --member="serviceAccount:${SA_EMAIL}" --role="roles/aiplatform.user"

# For TTS
gcloud projects add-iam-policy-binding $PROJECT \
  --member="serviceAccount:${SA_EMAIL}" --role="roles/cloudtexttospeech.admin"

# For STT
gcloud projects add-iam-policy-binding $PROJECT \
  --member="serviceAccount:${SA_EMAIL}" --role="roles/speech.admin"
```

No credentials file needed in Cloud Run — uses workload identity automatically.

---

## SDK Choice: google.genai vs vertexai

Two SDK options. Know which to use:

| SDK | Import | Use For |
|-----|--------|---------|
| `google.genai` | `from google import genai` | ADK agents, File Search API, Gemini via Vertex |
| `vertexai` | `import vertexai` | Imagen, VidGen-style non-ADK generation |

**ADK always uses `google.genai` internally.** Set `GOOGLE_GENAI_USE_VERTEXAI=1` to route it through Vertex AI instead of the public Gemini API.

For non-ADK code (standalone generation scripts, VidGen pipeline), use `vertexai`.

---

## Gemini Text Generation

### Standard (Non-ADK)

```python
import os
import vertexai
from vertexai.generative_models import GenerativeModel

vertexai.init(
    project=os.environ["GOOGLE_CLOUD_PROJECT"],
    location=os.environ.get("GOOGLE_CLOUD_LOCATION", "us-central1")
)

model = GenerativeModel(
    model_name="gemini-2.5-flash",
    generation_config={
        "temperature": 0.7,
        "top_p": 0.95,
        "max_output_tokens": 8192,
    }
)

response = model.generate_content("Explain ADK in one paragraph.")
print(response.text)
```

### JSON Response Parsing

Gemini often wraps JSON in markdown code blocks. Strip them:

```python
import json
import re

def parse_json_response(response_text: str) -> dict:
    """Extract JSON from Gemini response, handling markdown code blocks."""
    # Try to extract from ```json ... ``` block
    match = re.search(r'```(?:json)?\n?(.*?)\n?```', response_text, re.DOTALL)
    if match:
        return json.loads(match.group(1))
    # Fall back to parsing the whole response
    return json.loads(response_text)
```

### Gemini SDK (google.genai) — for File Search API + Context Caching

```python
import os
os.environ["GOOGLE_GENAI_USE_VERTEXAI"] = "1"

from google import genai

client = genai.Client()

response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="Explain the difference between RAG and context caching.",
)
print(response.text)
```

---

## Model Selection Guide

| Task | Model | Notes |
|------|-------|-------|
| Most tasks | `gemini-2.5-flash` | Default. Fast, cheap, good quality. |
| Complex reasoning | `gemini-2.5-pro` | ~5x cost. For nuanced analysis or difficult instructions. |
| Image generation | `imagen-3.0-generate-001` | Current production model. |
| Image editing | `imagen-3.0-capability-001` | For in/outpainting, editing. |

**Rule:** Default to Flash. Upgrade to Pro when Flash quality is insufficient for the specific task.

---

## Imagen — Image Generation

```python
from vertexai.preview.vision_models import ImageGenerationModel
import vertexai
import os

vertexai.init(project=os.environ["GOOGLE_CLOUD_PROJECT"], location="us-central1")

def generate_image(prompt: str, output_path: str, aspect_ratio: str = "16:9") -> None:
    """Generate one image with Imagen 3."""
    model = ImageGenerationModel.from_pretrained("imagen-3.0-generate-001")

    images = model.generate_images(
        prompt=prompt,
        number_of_images=1,
        aspect_ratio=aspect_ratio,          # "16:9", "1:1", "9:16", "4:3", "3:4"
        safety_filter_level="block_some",   # "block_few" | "block_some" | "block_most"
        person_generation="allow_adult",    # "dont_allow" | "allow_adult"
    )

    images[0].save(output_path)
```

### Batch Generation (Sequential — Avoids Rate Limits)

```python
from pathlib import Path

def generate_images_batch(prompts: list[str], output_dir: str) -> list[str]:
    """Generate multiple images sequentially. ~5-10s per image."""
    model = ImageGenerationModel.from_pretrained("imagen-3.0-generate-001")
    output_paths = []
    Path(output_dir).mkdir(parents=True, exist_ok=True)

    for i, prompt in enumerate(prompts):
        output_path = f"{output_dir}/{i:04d}.png"
        try:
            images = model.generate_images(
                prompt=prompt,
                number_of_images=1,
                aspect_ratio="16:9",
            )
            images[0].save(output_path)
            output_paths.append(output_path)
            print(f"Generated {i+1}/{len(prompts)}: {output_path}")
        except Exception as e:
            print(f"Failed image {i}: {e}")
            continue  # Skip and continue batch

    return output_paths
```

### Prompt Best Practices

```
# Good: detailed, style markers, quality terms
"A futuristic cityscape at night, neon lights reflecting on wet streets,
cinematic composition, photorealistic, 4K quality, wide angle"

# Bad: too vague
"a city"

# Triggers safety filter (avoid):
"person with weapon", "explicit content"
```

---

## Cloud TTS — Text to Speech

### Voice Reference

| Tier | Example Voices | Quality | Cost |
|------|---------------|---------|------|
| Standard | `en-US-Standard-A` through `J` | Robotic | Cheapest |
| WaveNet | `en-US-Wavenet-A` through `J` | Better | Mid |
| Neural2 | `en-US-Neural2-A` through `J` | Good | Mid |
| Studio | `en-US-Studio-O` (F), `en-US-Studio-Q` (M) | Most natural | Premium |
| Chirp3-HD | `en-US-Chirp3-HD-Aoede` (F), `en-US-Chirp3-HD-Puck` (M), etc. | Highest quality | Most expensive |

**Production default:** `en-US-Studio-O` (female) or `en-US-Studio-Q` (male). Best quality-to-cost ratio for published content.

### TTS with Chunking (Required for Long Text)

Google TTS has a 5000-byte limit per request. For scripts over ~4000 characters, chunk by paragraph.

```python
from google.cloud import texttospeech

def synthesize_speech(text: str, output_file: str, voice_name: str = "en-US-Studio-O") -> None:
    """Generate MP3 audio from text. Handles long text via chunking."""
    client = texttospeech.TextToSpeechClient()
    chunks = split_text_into_chunks(text, max_bytes=4500)
    audio_segments = []

    for chunk in chunks:
        synthesis_input = texttospeech.SynthesisInput(text=chunk)
        voice = texttospeech.VoiceSelectionParams(
            language_code="en-US",
            name=voice_name,
        )
        audio_config = texttospeech.AudioConfig(
            audio_encoding=texttospeech.AudioEncoding.MP3,
            speaking_rate=1.0,
            pitch=0.0,
        )
        response = client.synthesize_speech(
            input=synthesis_input,
            voice=voice,
            audio_config=audio_config,
        )
        audio_segments.append(response.audio_content)

    with open(output_file, "wb") as f:
        f.write(b"".join(audio_segments))


def split_text_into_chunks(text: str, max_bytes: int = 4500) -> list[str]:
    """Split text at paragraph boundaries, respecting byte limit."""
    paragraphs = text.split("\n\n")
    chunks = []
    current = ""

    for para in paragraphs:
        candidate = current + "\n\n" + para if current else para
        if len(candidate.encode("utf-8")) > max_bytes:
            if current:
                chunks.append(current.strip())
            current = para
        else:
            current = candidate

    if current:
        chunks.append(current.strip())

    return chunks
```

**Why bytes, not characters:** Multi-byte UTF-8 characters (accents, symbols) are 2-4 bytes each. Measure bytes, not len().

### Full Voice List (Current)

**Neural2 / WaveNet (cheaper):**
`en-US-Neural2-A/C/D/E/F/G/H/I/J`, `en-US-Wavenet-A/B/C/D/E/F/G/H/I/J`

**Studio (recommended):**
`en-US-Studio-O` (FEMALE), `en-US-Studio-Q` (MALE)

**Chirp3-HD (premium):**
`en-US-Chirp3-HD-Aoede` (F), `en-US-Chirp3-HD-Puck` (M), `en-US-Chirp3-HD-Charon` (M), `en-US-Chirp3-HD-Kore` (F), `en-US-Chirp3-HD-Fenrir` (M), `en-US-Chirp3-HD-Leda` (F), and 24 more.

---

## Cloud STT — Speech to Text

### Long-Running Recognition Pattern

For audio longer than 1 minute, use async long-running recognition.

```python
from google.cloud import speech
import os

def transcribe_audio(audio_file: str) -> str:
    """Transcribe audio file. Handles large files via GCS."""
    client = speech.SpeechClient()
    file_size = os.path.getsize(audio_file)

    if file_size > 10_000_000:  # > 10MB
        gcs_uri = upload_to_gcs(audio_file)
        audio = speech.RecognitionAudio(uri=gcs_uri)
    else:
        with open(audio_file, "rb") as f:
            audio = speech.RecognitionAudio(content=f.read())

    config = speech.RecognitionConfig(
        encoding=speech.RecognitionConfig.AudioEncoding.FLAC,
        sample_rate_hertz=16000,
        language_code="en-US",
        enable_automatic_punctuation=True,
        model="video",  # Optimized for video/podcast content
    )

    operation = client.long_running_recognize(config=config, audio=audio)
    print("Transcribing... (can take 10-30 minutes for long audio)")
    response = operation.result(timeout=3600)

    return " ".join(
        result.alternatives[0].transcript
        for result in response.results
    )
```

### GCS Upload (Required for Audio > 10MB)

```python
from google.cloud import storage
from pathlib import Path

def upload_to_gcs(local_file: str, bucket_name: str = None) -> str:
    """Upload file to GCS, return gs:// URI."""
    bucket_name = bucket_name or os.environ["GOOGLE_STT_BUCKET"]
    client = storage.Client()
    bucket = client.bucket(bucket_name)
    blob_name = f"audio/{Path(local_file).name}"
    blob = bucket.blob(blob_name)
    blob.upload_from_filename(local_file)
    return f"gs://{bucket_name}/{blob_name}"
```

---

## Multimodal Inputs — PDF and Image Analysis

Pass PDFs and images directly to Gemini for analysis. No OCR needed.

```python
import os
os.environ["GOOGLE_GENAI_USE_VERTEXAI"] = "1"

from google import genai
from google.genai import types

client = genai.Client()

def analyze_pdf(pdf_path: str, question: str) -> str:
    """Send PDF directly to Gemini for analysis."""
    with open(pdf_path, "rb") as f:
        pdf_bytes = f.read()

    response = client.models.generate_content(
        model="gemini-2.5-flash",
        contents=[
            types.Part.from_bytes(data=pdf_bytes, mime_type="application/pdf"),
            types.Part.from_text(text=question),
        ],
    )
    return response.text


def analyze_image(image_path: str, question: str) -> str:
    """Send image directly to Gemini for analysis."""
    import mimetypes
    mime_type, _ = mimetypes.guess_type(image_path)

    with open(image_path, "rb") as f:
        image_bytes = f.read()

    response = client.models.generate_content(
        model="gemini-2.5-flash",
        contents=[
            types.Part.from_bytes(data=image_bytes, mime_type=mime_type),
            types.Part.from_text(text=question),
        ],
    )
    return response.text
```

---

## Vertex Context Caching

Cache expensive context (large system prompts, reference documents) across many calls. Reduces latency and cost for repeated queries with the same context.

```python
import os
os.environ["GOOGLE_GENAI_USE_VERTEXAI"] = "1"

from google import genai
from google.genai import types
import datetime

client = genai.Client()

# Create a cached content object (cache the large context)
with open("reference_doc.txt") as f:
    large_document = f.read()

cache = client.caches.create(
    model="gemini-2.5-flash",
    contents=[
        types.Content(
            role="user",
            parts=[types.Part.from_text(text=large_document)]
        )
    ],
    system_instruction="You are an expert at analyzing this document.",
    ttl=datetime.timedelta(hours=1),   # Cache TTL
)

print(f"Cache created: {cache.name}")

# Use the cache for multiple queries
def query_with_cache(question: str, cache_name: str) -> str:
    response = client.models.generate_content(
        model="gemini-2.5-flash",
        contents=question,
        config=types.GenerateContentConfig(cached_content=cache_name),
    )
    return response.text

answer1 = query_with_cache("Summarize section 3.", cache.name)
answer2 = query_with_cache("What are the key metrics?", cache.name)
# Both calls use cached context — only charged once for the large doc
```

**When to use:** System prompts over 1000 tokens that are reused across many requests. Knowledge base documents that many users query. Reduces both latency (context not re-ingested) and cost (cached tokens billed at lower rate).

---

## Google Files API — Live Context Upload

Upload files at runtime for per-session document analysis. Different from File Search API (RAG) — this makes the file directly available to the model for one session.

```python
import os
os.environ["GOOGLE_GENAI_USE_VERTEXAI"] = "1"

from google import genai
from google.genai import types

client = genai.Client()

def analyze_uploaded_file(file_path: str, question: str) -> str:
    """Upload file and ask Gemini about it."""
    import mimetypes
    mime_type, _ = mimetypes.guess_type(file_path)

    uploaded_file = client.files.upload(
        path=file_path,
        mime_type=mime_type,
    )

    # Wait for processing
    import time
    while uploaded_file.state.name == "PROCESSING":
        time.sleep(1)
        uploaded_file = client.files.get(name=uploaded_file.name)

    if uploaded_file.state.name != "ACTIVE":
        raise ValueError(f"File upload failed: {uploaded_file.state}")

    # Use in generation
    response = client.models.generate_content(
        model="gemini-2.5-flash",
        contents=[
            types.Part.from_uri(
                file_uri=uploaded_file.uri,
                mime_type=mime_type,
            ),
            types.Part.from_text(text=question),
        ]
    )

    # Clean up
    client.files.delete(name=uploaded_file.name)

    return response.text
```

**Files API vs File Search API:**
- **Files API:** Upload file → pass directly to model → model "reads" it in context. Good for one-off document analysis.
- **File Search API (RAG):** Upload to a store → model retrieves relevant chunks on query. Good for large knowledge bases with many documents.

---

## Audio Duration Calculation

Needed for video pipelines to determine how many images to generate.

```python
from pydub import AudioSegment

def get_audio_duration_seconds(audio_file: str) -> float:
    audio = AudioSegment.from_mp3(audio_file)
    return len(audio) / 1000.0

# Usage: generate 1 image per 10 seconds of audio
duration = get_audio_duration_seconds("narration.mp3")
num_images = max(1, int(duration / 10))
```

---

## Error Handling

```python
from google.api_core import exceptions

try:
    response = model.generate_content(prompt)
except exceptions.ResourceExhausted:
    # Rate limit hit — back off and retry
    import time
    time.sleep(60)
    response = model.generate_content(prompt)
except exceptions.InvalidArgument as e:
    # Bad request — don't retry
    raise ValueError(f"Invalid prompt or parameters: {e}")
except exceptions.ServiceUnavailable:
    # Transient error — retry with backoff
    import time
    time.sleep(10)
    response = model.generate_content(prompt)
```

---

## Gotchas

**1. vertexai.init() must be called before any vertexai usage**
Always call `vertexai.init(project=..., location=...)` before creating models. Without it, calls fail with auth or project errors.

**2. GOOGLE_GENAI_USE_VERTEXAI before google.genai imports**
Same as ADK — set env var before importing the library.

**3. TTS byte limit, not character limit**
The API limit is 5000 bytes. UTF-8 encodes non-ASCII characters as 2-4 bytes. `len(text)` underestimates. Use `len(text.encode('utf-8'))`.

**4. STT requires FLAC or LINEAR16 for long-running recognition**
MP3 is not supported for long-running recognition. Convert to FLAC first: `ffmpeg -i input.mp3 -ar 16000 output.flac`.

**5. Imagen generates one image per call — loop for multiple**
`number_of_images=1` is the only reliable option. To get 6 images, call generate_images() 6 times in a loop. Don't try to batch — rate limits hit at higher counts.

**6. Files API state must be ACTIVE before use**
Upload is async. Poll until `state.name == "ACTIVE"` before passing the file to generate_content.

---

## What's Next

- RAG with File Search API → `rag-pipelines.md`
- Deploy to Cloud Run with Vertex AI permissions → `cloud-run-deployment.md`
- ADK agents using Vertex modalities → `adk-agents-fundamentals.md`

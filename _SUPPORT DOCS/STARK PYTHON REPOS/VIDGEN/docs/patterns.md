# VidGen Patterns

**Project:** Bibo YouTube Video Generator
**Extracted:** 2026-02-22

---

## Overview

This document captures repeating patterns observed across the VidGen codebase. These patterns represent conventions, best practices, and common approaches that should be replicated in similar projects.

---

## Import Patterns

### Standard Imports

Every module follows consistent import organization:

```python
# Standard library (alphabetical)
import json
import os
from pathlib import Path
from typing import Dict, List, Optional

# Third-party (alphabetical)
from google.cloud import aiplatform, speech, texttospeech
from google.ai import generativelanguage as genai
import vertexai

# Local imports (relative)
from src.logger import log_message
from src.utils.config import load_config
```

**Pattern:** Standard → Third-party → Local, all alphabetized within groups.

---

## Configuration Management

### Config Loading Pattern

```python
# src/utils/config.py
def load_config(config_file: str = "config/config.json") -> Dict:
    """Load configuration from JSON file."""
    with open(config_file, 'r') as f:
        return json.load(f)

# Usage in modules
config = load_config()
default_model = config["models"]["gemini-2.5-flash"]["name"]
```

**Pattern:** Centralized config loader, JSON-based, defaults in config file.

### Environment Variables

```python
# Always check for required env vars at module load
import os

GOOGLE_APPLICATION_CREDENTIALS = os.getenv("GOOGLE_APPLICATION_CREDENTIALS")
if not GOOGLE_APPLICATION_CREDENTIALS:
    raise EnvironmentError("GOOGLE_APPLICATION_CREDENTIALS not set")
```

**Pattern:** Fail fast if required env vars missing.

---

## File Path Handling

### Path Construction Pattern

```python
from pathlib import Path

def get_project_path(project_name: str) -> Path:
    """Get path to project directory."""
    return Path("projects") / project_name

def get_stage_file(project_name: str, stage: str) -> Path:
    """Get path to stage output file."""
    project_path = get_project_path(project_name)

    stage_files = {
        "transcript": "0_transcript.txt",
        "script": "1_summary.txt",
        "audio": "2_audio.mp3",
        "prompts": "3_image_prompts.json",
        "metadata": "4_metadata.json",
        "images": "5_images",
        "video": "6_final_video.mp4"
    }

    return project_path / stage_files[stage]
```

**Pattern:**
- Use `Path` over string concatenation
- Centralize path logic (no hardcoded paths scattered)
- Dictionary mapping for stage → filename convention

---

## Directory Creation Pattern

```python
from pathlib import Path

def ensure_directory_exists(directory: Path) -> None:
    """Create directory if it doesn't exist."""
    directory.mkdir(parents=True, exist_ok=True)

# Usage
output_dir = Path("projects") / project_name / "5_images"
ensure_directory_exists(output_dir)
```

**Pattern:** Always create directories before writing files, use `parents=True, exist_ok=True`.

---

## Vertex AI Patterns

### Initialization Pattern

```python
import vertexai
from google.cloud import aiplatform

# Initialize Vertex AI (once per module)
PROJECT_ID = os.getenv("VERTEX_AI_PROJECT")
LOCATION = os.getenv("VERTEX_AI_LOCATION", "us-central1")

vertexai.init(project=PROJECT_ID, location=LOCATION)
```

**Pattern:** Initialize at module level, use env vars for project/location.

### Gemini Generation Pattern

```python
from google.ai import generativelanguage as genai

def generate_with_gemini(
    prompt: str,
    model: str = "gemini-2.5-flash",
    temperature: float = 0.7
) -> str:
    """Generate content with Gemini model."""

    # Configure model
    generation_config = {
        "temperature": temperature,
        "top_p": 0.95,
        "top_k": 40,
        "max_output_tokens": 8192,
    }

    # Create model instance
    model_instance = genai.GenerativeModel(
        model_name=model,
        generation_config=generation_config
    )

    # Generate
    response = model_instance.generate_content(prompt)

    return response.text
```

**Pattern:**
- Function wraps model call
- Config dict for generation parameters
- Returns plain text (unwrapped)

### Imagen Generation Pattern

```python
from vertexai.preview.vision_models import ImageGenerationModel

def generate_image(prompt: str, output_path: str) -> None:
    """Generate image with Imagen."""

    model = ImageGenerationModel.from_pretrained("imagen-3.0-generate-001")

    images = model.generate_images(
        prompt=prompt,
        number_of_images=1,
        aspect_ratio="16:9"
    )

    images[0].save(output_path)
```

**Pattern:**
- Load pretrained model
- Single image generation (for consistency)
- Fixed aspect ratio (16:9 for YouTube)

---

## Audio Processing Patterns

### TTS Pattern (Chunked for Long Text)

```python
from google.cloud import texttospeech

def synthesize_speech(text: str, output_file: str, voice: str) -> None:
    """Convert text to speech, chunking if necessary."""

    client = texttospeech.TextToSpeechClient()

    # Chunk text (TTS has 5000 byte limit)
    chunks = split_text_into_chunks(text, max_bytes=4500)

    audio_segments = []

    for chunk in chunks:
        synthesis_input = texttospeech.SynthesisInput(text=chunk)

        voice_params = texttospeech.VoiceSelectionParams(
            language_code="en-US",
            name=voice
        )

        audio_config = texttospeech.AudioConfig(
            audio_encoding=texttospeech.AudioEncoding.MP3
        )

        response = client.synthesize_speech(
            input=synthesis_input,
            voice=voice_params,
            audio_config=audio_config
        )

        audio_segments.append(response.audio_content)

    # Combine segments
    combined_audio = combine_audio_segments(audio_segments)

    with open(output_file, 'wb') as f:
        f.write(combined_audio)
```

**Pattern:**
- Check text length, chunk if needed
- Process chunks sequentially
- Combine audio segments
- Single output file

### Text Chunking Strategy

```python
def split_text_into_chunks(text: str, max_bytes: int = 4500) -> List[str]:
    """Split text into chunks respecting paragraph boundaries."""

    paragraphs = text.split('\n\n')
    chunks = []
    current_chunk = ""

    for para in paragraphs:
        # Check if adding paragraph exceeds limit
        if len((current_chunk + para).encode('utf-8')) > max_bytes:
            if current_chunk:
                chunks.append(current_chunk.strip())
            current_chunk = para
        else:
            current_chunk += "\n\n" + para if current_chunk else para

    if current_chunk:
        chunks.append(current_chunk.strip())

    return chunks
```

**Pattern:**
- Respect paragraph boundaries (don't split mid-paragraph)
- Measure in bytes (not characters) - API limit is bytes
- Leave buffer (4500 bytes for 5000 byte limit)

---

## Speech-to-Text Patterns

### Long-Running Recognition Pattern

```python
from google.cloud import speech

def transcribe_audio(audio_file: str) -> str:
    """Transcribe audio file (handles large files)."""

    client = speech.SpeechClient()

    # Check file size
    file_size = os.path.getsize(audio_file)

    if file_size > 10_000_000:  # >10MB
        # Upload to GCS and use URI
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
    )

    # Long-running operation
    operation = client.long_running_recognize(config=config, audio=audio)

    print("Waiting for transcription...")
    response = operation.result(timeout=3600)  # 1 hour max

    # Combine all transcripts
    transcript = " ".join([
        result.alternatives[0].transcript
        for result in response.results
    ])

    return transcript
```

**Pattern:**
- Check file size, GCS upload if >10MB
- Use long-running recognize (not synchronous)
- Poll with generous timeout
- Combine all result segments into single text

---

## MoviePy 2.x Patterns

### Video Composition Pattern

```python
from moviepy.editor import AudioFileClip, ImageClip, concatenate_videoclips
from moviepy import vfx

def compose_video(images_dir: str, audio_file: str, output_file: str) -> None:
    """Compose video from images and audio."""

    # Load audio
    audio = AudioFileClip(audio_file)
    audio_duration = audio.duration

    # Load images
    image_files = sorted(Path(images_dir).glob("*.png"))

    # Calculate duration per image
    duration_per_image = audio_duration / len(image_files)

    # Create video clips
    clips = []
    for img_file in image_files:
        clip = (ImageClip(str(img_file))
                .with_duration(duration_per_image)
                .with_effects([vfx.Resize(height=1080)]))
        clips.append(clip)

    # Concatenate
    video = concatenate_videoclips(clips, method="compose")

    # Set audio
    video = video.with_audio(audio)

    # Write output
    video.write_videofile(
        output_file,
        fps=24,
        codec="libx264",
        audio_codec="aac"
    )
```

**Pattern (MoviePy 2.x API):**
- `.with_duration()` instead of `.set_duration()`
- `.with_audio()` instead of `.set_audio()`
- `.with_effects([])` instead of chaining `.fx()`
- Import effects from `moviepy.vfx`

---

## JSON Handling Patterns

### JSON Read/Write Pattern

```python
import json
from pathlib import Path

def read_json(file_path: Path) -> Dict:
    """Read JSON file."""
    with open(file_path, 'r', encoding='utf-8') as f:
        return json.load(f)

def write_json(file_path: Path, data: Dict) -> None:
    """Write JSON file."""
    with open(file_path, 'w', encoding='utf-8') as f:
        json.dump(data, f, indent=2, ensure_ascii=False)
```

**Pattern:**
- Always specify `encoding='utf-8'`
- Always use `indent=2` for readability
- Use `ensure_ascii=False` for international characters

### JSON Structure for Image Prompts

```json
{
  "prompts": [
    {
      "timestamp": 0.0,
      "duration": 10.5,
      "prompt": "A serene mountain landscape..."
    },
    {
      "timestamp": 10.5,
      "duration": 10.5,
      "prompt": "A bustling city street..."
    }
  ],
  "total_duration": 42.0,
  "images_count": 4
}
```

**Pattern:** Array of prompt objects with timestamp/duration for sync with audio.

---

## Logging Patterns

### Session Logging Pattern

```python
# src/logger.py
from datetime import datetime
from pathlib import Path

def log_message(project_name: str, message: str, level: str = "INFO") -> None:
    """Log message to project-specific log file."""

    log_file = Path("projects") / project_name / "pipeline.log"

    timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    log_entry = f"[{timestamp}] [{level}] {message}\n"

    with open(log_file, 'a', encoding='utf-8') as f:
        f.write(log_entry)

    # Also print to console
    print(log_entry.strip())
```

**Pattern:**
- Per-project log file
- Timestamp + level + message
- Append mode (don't overwrite)
- Mirror to console for real-time feedback

### Usage Pattern

```python
from src.logger import log_message

log_message(project_name, "Starting script generation", "INFO")
log_message(project_name, f"Using model: {model}", "INFO")

try:
    result = generate_script(...)
    log_message(project_name, "Script generated successfully", "SUCCESS")
except Exception as e:
    log_message(project_name, f"Error: {e}", "ERROR")
    raise
```

**Pattern:** Log start, config, success/error at each stage.

---

## Error Handling Patterns

### Try/Except with Logging

```python
def process_stage(project_name: str, stage: str) -> None:
    """Process a pipeline stage with error handling."""

    log_message(project_name, f"Starting {stage}", "INFO")

    try:
        # Stage processing
        result = execute_stage(project_name, stage)

        log_message(project_name, f"{stage} completed", "SUCCESS")

    except Exception as e:
        log_message(project_name, f"{stage} failed: {e}", "ERROR")
        raise  # Re-raise for caller to handle
```

**Pattern:**
- Log before processing
- Try/except around main logic
- Log success or error
- Re-raise exceptions (don't swallow)

---

## Streamlit Patterns

### Page Structure Pattern

```python
import streamlit as st
from pathlib import Path

# Add project root to path (for imports)
import sys
project_root = Path(__file__).resolve().parent.parent.parent
if str(project_root) not in sys.path:
    sys.path.insert(0, str(project_root))

from app.state import get_approval_status, set_approval
from src.module import process_function

# Page config
st.set_page_config(page_title="Stage Name", layout="wide")

# Check project selected
if "current_project" not in st.session_state:
    st.error("No project selected. Go to Home.")
    st.stop()

project = st.session_state.current_project

# Check prerequisites
if not prerequisite_met(project):
    st.warning("Complete previous stage first.")
    st.stop()

# Main UI
st.title("Stage Name")

# Action button
if st.button("Process"):
    with st.spinner("Processing..."):
        process_function(project)
    st.success("Done!")
    st.rerun()  # Refresh to show results
```

**Pattern:**
- Path manipulation at top
- Check session state
- Check prerequisites
- Main action with spinner
- Rerun after state change

### State Management Pattern

```python
# app/state.py
def get_approval_status(project: str, stage: str) -> bool:
    """Check if stage is approved."""
    config_file = Path("projects") / project / "config.json"

    if not config_file.exists():
        return False

    config = read_json(config_file)
    return config.get("approved", {}).get(stage, False)

def set_approval(project: str, stage: str, approved: bool) -> None:
    """Set approval status for stage."""
    config_file = Path("projects") / project / "config.json"

    config = read_json(config_file) if config_file.exists() else {}

    if "approved" not in config:
        config["approved"] = {}

    config["approved"][stage] = approved

    write_json(config_file, config)
```

**Pattern:** Centralize state operations, work with config.json directly.

---

## Testing Patterns

### Mocking External APIs (Unit Tests)

```python
# tests/unit/test_summarization.py
from unittest.mock import Mock, patch
import pytest

@patch('src.summarization.genai.GenerativeModel')
def test_summarize_transcript(mock_model):
    """Test script generation with mocked Gemini."""

    # Mock response
    mock_response = Mock()
    mock_response.text = "Generated script content"
    mock_model.return_value.generate_content.return_value = mock_response

    # Call function
    result = summarize_transcript("input.txt", "output.txt")

    # Assertions
    assert mock_model.called
    assert "Generated script content" in result
```

**Pattern:**
- Mock at module level (`@patch('module.Class')`)
- Mock returns expected structure
- Assert both call and result

### Integration Test Pattern

```python
# tests/integration/test_vertex_ai.py
import pytest
from google.cloud import aiplatform

@pytest.mark.integration
def test_gemini_flash_generation():
    """Test actual Gemini API call."""

    # Requires valid credentials
    # May incur costs

    prompt = "Write a haiku about testing."

    response = generate_with_gemini(prompt, model="gemini-2.5-flash")

    assert response
    assert len(response) > 0
```

**Pattern:**
- Mark with `@pytest.mark.integration`
- Document cost/credential requirements
- Test real API behavior

---

## Key Patterns Summary

| Pattern | Purpose | Example |
|---------|---------|---------|
| **Path objects** | Cross-platform paths | `Path("projects") / project / "file.txt"` |
| **Config-driven** | Avoid hardcoding | `config["models"][model_name]` |
| **Chunking** | Handle API limits | Split text at 4500 bytes |
| **File-based state** | Zero database | Check file existence = stage complete |
| **Module independence** | Reusable functions | Each `src/*.py` is standalone |
| **GCS fallback** | Large file handling | Upload to GCS if >10MB |
| **Operation polling** | Long-running APIs | `operation.result(timeout=3600)` |
| **MoviePy 2.x** | Video composition | `.with_duration()`, `.with_effects()` |
| **Logging everywhere** | Debuggability | Log start/success/error for each stage |
| **Fail fast** | Early error detection | Raise if env vars missing |

---

_Extracted: 2026-02-22_

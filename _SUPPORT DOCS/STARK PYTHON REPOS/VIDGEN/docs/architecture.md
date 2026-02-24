# VidGen Architecture

**Project:** Bibo YouTube Video Generator (VidGen)
**Extracted:** 2026-02-22

---

## Overview

VidGen is a CLI-based video generation pipeline that transforms YouTube videos (or raw text) into new video content using AI services. The system uses a **file-based state management** approach with sequential stage processing.

**Key Characteristic:** Zero database - all state lives in filesystem.

---

## System Architecture

### High-Level Flow

```
Input (YouTube URL or Text)
    ↓
0. Transcription (YouTube → Text)
    ↓
1. Summarization (Text → Script)
    ↓
2. Text-to-Speech (Script → Audio)
    ↓
3. Metadata Generation (Script → Titles/Descriptions)
    ↓
4. Image Prompting (Script + Audio Duration → Prompts)
    ↓
5. Image Creation (Prompts → PNG Files)
    ↓
6. Video Composition (Audio + Images → MP4)
    ↓
Final Output (Shareable Video)
```

---

## File Structure

```
project-bibo-youtube-v2/
├── main.py                      # CLI orchestrator
├── src/                         # Pipeline modules
│   ├── transcription.py         # YouTube → text
│   ├── summarization.py         # Text → script
│   ├── text_to_speech.py        # Script → audio
│   ├── metadata_generation.py   # Script → metadata
│   ├── image_prompting.py       # Script → image prompts
│   ├── image_creation.py        # Prompts → images
│   ├── video_composition.py     # Audio + images → video
│   ├── logger.py                # Session logging
│   └── utils/
│       └── config.py            # Configuration management
├── config/
│   └── config.json              # Model/voice configurations
├── projects/                    # Output directory
│   └── {project_name}/
│       ├── config.json          # Approval tracking
│       ├── 0_transcript.txt
│       ├── 1_summary.txt
│       ├── 2_audio.mp3
│       ├── 3_image_prompts.json
│       ├── 4_metadata.json
│       ├── 5_images/*.png
│       ├── 6_final_video.mp4
│       └── pipeline.log
├── app/                         # Streamlit GUI wrapper
│   ├── main.py
│   ├── state.py                 # GUI state management
│   ├── components/
│   │   └── sidebar.py
│   └── pages/                   # One page per stage
│       ├── 1_inputs.py
│       ├── 2_script.py
│       ├── 3_audio.py
│       ├── 4_metadata.py
│       ├── 5_images.py
│       └── 6_video.py
└── tests/
    ├── unit/                    # Mocked tests
    └── integration/             # Real API tests
```

---

## Core Design Decisions

### 1. File-Based State Management

**Decision:** Use filesystem for all state (no database)

**Rationale:**
- **Simplicity:** No DB setup, migrations, connection pooling
- **Portability:** Project folders are self-contained, can be zipped/shared
- **Debuggability:** State is human-readable (text, JSON, audio files)
- **Resume-ability:** Can restart from any stage by checking file existence
- **Backup/Recovery:** Copy folder = full backup

**Implementation:**
```python
# State is implicit in file existence
def stage_is_complete(project_name: str, stage: str) -> bool:
    stage_file = get_stage_file_path(project_name, stage)
    return os.path.exists(stage_file)
```

**Trade-offs:**
- ✅ Zero infrastructure overhead
- ✅ Easy to understand and debug
- ❌ No query capabilities (can't ask "all projects with approved scripts")
- ❌ No concurrent access protection (single-user assumption)

---

### 2. Sequential Pipeline Architecture

**Decision:** Stages must complete in order (no parallelization)

**Rationale:**
- Each stage depends on previous stage output
- Linear flow is easier to understand and debug
- User approval gates ensure quality before proceeding

**Dependencies:**
```
Starter (YouTube URL or Text)
    ↓
Script (requires: transcript or text)
    ↓
Audio + Metadata (requires: script) ← Can run in parallel
    ↓
Images (requires: audio for duration + script)
    ↓
Video (requires: images + audio)
```

**Implementation:**
```python
PIPELINE_STAGES = [
    {"name": "transcript", "file": "0_transcript.txt", "prereq": None},
    {"name": "script", "file": "1_summary.txt", "prereq": "transcript"},
    {"name": "audio", "file": "2_audio.mp3", "prereq": "script"},
    {"name": "metadata", "file": "4_metadata.json", "prereq": "script"},
    {"name": "images", "file": "5_images/", "prereq": "audio"},
    {"name": "video", "file": "6_final_video.mp4", "prereq": "images"}
]
```

---

### 3. Approval Tracking via JSON

**Decision:** Store approval state in `projects/{name}/config.json`

**Rationale:**
- Separates state (file exists) from approval (user reviewed)
- Allows regeneration without losing approval
- Simple JSON structure, easy to read/write
- Per-project config keeps approval local

**Structure:**
```json
{
  "project_name": "my_project",
  "approved": {
    "script": true,
    "audio": true,
    "metadata": false,
    "images": false,
    "video": false
  },
  "voice": "en-US-Studio-O",
  "model": "gemini-2.5-flash"
}
```

**Usage in GUI:**
```python
# Check approval before allowing next stage
if not get_approval_status(project, "script"):
    st.error("Cannot generate audio. Script not approved.")
    return
```

---

### 4. Module Independence

**Decision:** Each `src/*.py` module is independently executable

**Rationale:**
- CLI can call modules directly
- GUI can call same modules (no duplication)
- Testing is easier (can test modules in isolation)
- Debugging is simpler (can run single stage)

**Implementation:**
Each module has standard signature:
```python
# src/summarization.py
def summarize_transcript(
    input_file: str,
    output_file: str,
    model: str = "gemini-2.5-flash"
) -> None:
    """Standalone function, file in → file out"""
    pass
```

**Benefits:**
- CLI calls: `summarize_transcript("0_transcript.txt", "1_summary.txt")`
- GUI calls: Same function, same parameters
- Tests can mock file I/O, test logic independently

---

### 5. Streamlit as Thin Wrapper

**Decision:** GUI (Streamlit) is control surface only, no logic

**Rationale:**
- Pipeline logic lives in `src/`, not in UI
- UI cannot drift from CLI behavior
- Easier to test (logic is in pure Python, not UI code)
- Can swap UI frameworks without touching pipeline

**Pattern:**
```python
# app/pages/2_script.py (Streamlit)
if st.button("Generate Script"):
    # Call src module directly
    from src.summarization import summarize_transcript

    summarize_transcript(
        input_file=f"projects/{project}/0_transcript.txt",
        output_file=f"projects/{project}/1_summary.txt"
    )

    st.success("Script generated!")
```

**No logic in UI:**
- ❌ No AI calls in Streamlit code
- ❌ No file format transformations in UI
- ❌ No business rules in UI
- ✅ Only: trigger functions, display results, update approval state

---

## Data Flow

### File Naming Convention

```
projects/{project_name}/
├── 0_transcript.txt        # Stage 0: YouTube transcription
├── 1_summary.txt           # Stage 1: Script generation
├── 2_audio.mp3             # Stage 2: Text-to-speech
├── 3_image_prompts.json    # Stage 3: Image prompt generation
├── 4_metadata.json         # Stage 4: Title/description
├── 5_images/               # Stage 5: Generated images
│   ├── image_0000.png
│   ├── image_0001.png
│   └── ...
└── 6_final_video.mp4       # Stage 6: Composed video
```

**Pattern:** Numbered prefixes indicate stage order.

**Note:** Numbers don't strictly match stage count (3 vs 4 reversed for historical reasons).

---

### State Transitions

```
[No Files]
    ↓ Paste text OR YouTube URL
[0_transcript.txt exists]
    ↓ Generate script
[1_summary.txt exists]
    ↓ User approves script
[1_summary.txt + approved=true]
    ↓ Generate audio
[2_audio.mp3 exists]
    ↓ User approves audio
[2_audio.mp3 + approved=true]
    ↓ Generate image prompts
[3_image_prompts.json exists]
    ↓ Generate images
[5_images/*.png exist]
    ↓ User approves images
[5_images/*.png + approved=true]
    ↓ Compose video
[6_final_video.mp4 exists]
    ↓ User approves video
[COMPLETE]
```

---

## Integration Points

### External Services

1. **Vertex AI (Gemini)**
   - Purpose: Script generation, metadata, image prompts
   - Models: gemini-2.5-flash (default), gemini-2.5-pro (high quality)
   - Authentication: ADC or service account JSON

2. **Vertex AI (Imagen)**
   - Purpose: Image generation
   - Model: imagen-3.0-generate-001
   - Authentication: Same as Gemini

3. **Google Cloud Text-to-Speech**
   - Purpose: Script narration
   - Voices: Studio voices (en-US-Studio-O, en-US-Studio-M)
   - Authentication: Same as Gemini

4. **Google Cloud Speech-to-Text**
   - Purpose: YouTube transcription
   - API: Long-running recognize (up to 480 minutes)
   - Authentication: Same as Gemini

5. **Google Cloud Storage**
   - Purpose: Large audio file uploads (>10MB) for transcription
   - Bucket: Configured in `.env`
   - Authentication: Same as Gemini

6. **yt-dlp**
   - Purpose: Download YouTube audio
   - Integration: Command-line tool via subprocess

7. **MoviePy**
   - Purpose: Video composition
   - Version: 2.x (different API than 1.x)

---

## Configuration Management

### Global Config (`config/config.json`)

```json
{
  "models": {
    "gemini-2.5-flash": {...},
    "gemini-2.5-pro": {...}
  },
  "voices": {
    "en-US-Studio-O": "Female",
    "en-US-Studio-M": "Male"
  },
  "imagen": {
    "model": "imagen-3.0-generate-001"
  }
}
```

**Purpose:**
- Model selection dropdown options
- Voice selection dropdown options
- System-wide defaults

### Project Config (`projects/{name}/config.json`)

```json
{
  "project_name": "my_project",
  "approved": {
    "script": false,
    "audio": false,
    ...
  },
  "voice": "en-US-Studio-O",
  "model": "gemini-2.5-flash"
}
```

**Purpose:**
- Per-project approval state
- Per-project voice/model preferences
- Created automatically when project is created

---

## Error Handling Patterns

### Long-Running API Calls

**Problem:** Speech-to-Text can take 10-30 minutes for long videos.

**Solution:** Use operation polling with timeout.

```python
operation = client.long_running_recognize(...)
response = operation.result(timeout=3600)  # 1 hour max
```

### Large File Handling

**Problem:** Audio files >10MB cannot be sent directly to Speech-to-Text API.

**Solution:** Auto-upload to GCS, transcribe from bucket.

```python
if file_size > 10_000_000:
    gcs_uri = upload_to_gcs(audio_file)
    audio = speech.RecognitionAudio(uri=gcs_uri)
else:
    content = read_audio_file(audio_file)
    audio = speech.RecognitionAudio(content=content)
```

### API Rate Limiting

**Current:** No explicit rate limiting (relies on Google's built-in limits).

**Future:** Could add exponential backoff retry logic.

---

## Scalability Considerations

### Current Limitations

1. **Single-threaded:** Pipeline runs one stage at a time
2. **Single-project:** CLI processes one project at a time
3. **No queue:** No job queue for background processing
4. **No caching:** Regenerating script means re-calling Gemini (no cache)
5. **Local filesystem:** Projects stored locally (not cloud-backed)

### Potential Improvements

1. **Parallel metadata + audio:** These stages are independent
2. **Background jobs:** Use Celery or Cloud Tasks for long operations
3. **Response caching:** Cache LLM responses (with cache key = prompt hash)
4. **Cloud storage:** Store projects in GCS for multi-machine access
5. **Batch processing:** Process multiple projects concurrently

**Current design choice:** Optimize for simplicity, not scale (single-user assumption).

---

## Testing Strategy

See `testing_strategy.md` for details.

**Summary:**
- Unit tests: Mock external APIs
- Integration tests: Real APIs (cost/time considerations)
- Manual testing: Full pipeline with real YouTube videos

---

## Deployment Model

**Current:** Local Python execution
- User runs `python main.py` or `streamlit run app/main.py`
- Dependencies installed via pip
- Requires local Google Cloud credentials

**Future (Potential):**
- Package as desktop app (PyInstaller)
- Deploy Streamlit to Cloud Run (for web access)
- API wrapper (FastAPI) for programmatic access

---

## Key Takeaways

1. **File-based state** eliminates database complexity
2. **Sequential pipeline** ensures data dependencies are met
3. **Module independence** enables both CLI and GUI usage
4. **Approval gates** give user control over quality
5. **Vertex AI integration** centralizes on Google ecosystem
6. **Streamlit wrapper** provides UI without duplicating logic

**This architecture prioritizes:**
- ✅ Simplicity over scalability
- ✅ Debuggability over performance
- ✅ User control over automation
- ✅ Filesystem over database
- ✅ Sequential over parallel

---

_Extracted: 2026-02-22_

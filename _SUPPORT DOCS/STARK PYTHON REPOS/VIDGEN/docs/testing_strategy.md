# VidGen Testing Strategy

**Project:** Bibo YouTube Video Generator
**Extracted:** 2026-02-22

---

## Overview

VidGen uses a hybrid testing approach:
- **Unit tests** with mocked external dependencies (fast, free, offline)
- **Integration tests** with real API calls (slow, costly, requires credentials)
- **Manual end-to-end testing** for full pipeline validation

---

## Test Structure

```
tests/
├── unit/                          # Mocked tests (default)
│   ├── test_transcription.py
│   ├── test_summarization.py
│   ├── test_text_to_speech.py
│   ├── test_metadata_generation.py
│   ├── test_image_prompting.py
│   ├── test_image_creation.py
│   └── test_video_composition.py
├── integration/                   # Real API tests (marked)
│   ├── test_vertex_ai.py
│   ├── test_speech_services.py
│   └── test_full_pipeline.py
└── conftest.py                    # Pytest fixtures
```

---

## Unit Testing Approach

### Philosophy

**Mock all external dependencies:**
- Vertex AI API calls
- Google Cloud TTS/STT
- File I/O (where appropriate)
- yt-dlp subprocess calls
- MoviePy operations

**Test business logic only:**
- Prompt construction
- Response parsing
- Error handling
- Data transformations

### Example: Summarization Test

```python
# tests/unit/test_summarization.py
from unittest.mock import Mock, patch, mock_open
import pytest
from src.summarization import summarize_transcript

@patch('src.summarization.genai.GenerativeModel')
@patch('builtins.open', new_callable=mock_open, read_data="Test transcript")
def test_summarize_transcript(mock_file, mock_model):
    """Test script generation with mocked Gemini."""

    # Mock Gemini response
    mock_response = Mock()
    mock_response.text = "Generated script content"
    mock_model.return_value.generate_content.return_value = mock_response

    # Call function
    summarize_transcript(
        input_file="input.txt",
        output_file="output.txt",
        model="gemini-2.5-flash"
    )

    # Assertions
    mock_model.assert_called_once()
    mock_file.assert_called()

    # Check file was written with generated content
    handle = mock_file()
    handle.write.assert_called_with("Generated script content")
```

**Key patterns:**
- `@patch` decorator mocks external dependencies
- `mock_open` handles file operations
- Assert both calls and results

### Example: TTS Test (Chunking Logic)

```python
# tests/unit/test_text_to_speech.py
from src.text_to_speech import split_text_into_chunks

def test_split_text_respects_byte_limit():
    """Test that chunking respects byte limit."""

    # Create text that would exceed limit
    long_text = "Lorem ipsum. " * 500  # ~6500 bytes

    chunks = split_text_into_chunks(long_text, max_bytes=4500)

    # Should split into multiple chunks
    assert len(chunks) > 1

    # Each chunk should be under limit
    for chunk in chunks:
        assert len(chunk.encode('utf-8')) <= 4500

def test_split_text_respects_paragraphs():
    """Test that chunking doesn't split paragraphs."""

    text = "Paragraph 1.\n\nParagraph 2.\n\nParagraph 3."

    chunks = split_text_into_chunks(text, max_bytes=100)

    # Each chunk should end with complete paragraph
    for chunk in chunks:
        # Should not end mid-word
        assert not chunk.endswith(" ")
```

**Focus:** Test the chunking algorithm without calling real TTS API.

### Example: JSON Parsing Test

```python
# tests/unit/test_metadata_generation.py
import json
from src.metadata_generation import parse_metadata_response

def test_parse_metadata_from_json():
    """Test parsing metadata from JSON response."""

    response_text = '''
    ```json
    {
      "titles": ["Title 1", "Title 2"],
      "description": "Video description",
      "hashtags": ["tag1", "tag2"]
    }
    ```
    '''

    result = parse_metadata_response(response_text)

    assert "titles" in result
    assert len(result["titles"]) == 2
    assert result["description"] == "Video description"
    assert len(result["hashtags"]) == 2

def test_parse_metadata_handles_plain_json():
    """Test parsing when response is plain JSON (no markdown)."""

    response_text = '{"titles": ["Title"], "description": "Desc", "hashtags": ["tag"]}'

    result = parse_metadata_response(response_text)

    assert "titles" in result
```

**Focus:** Test JSON extraction/parsing logic without calling Gemini.

---

## Integration Testing Approach

### Philosophy

**Test real API behavior:**
- Verify actual API responses
- Test authentication flow
- Validate model outputs
- Catch API changes early

**Mark as integration tests:**
```python
@pytest.mark.integration
def test_real_api():
    # Real API call here
    pass
```

**Run explicitly:**
```bash
# Skip integration tests by default
pytest tests/unit/  # Fast, free

# Run integration tests explicitly
pytest -m integration  # Slow, costs money
```

### Example: Gemini Integration Test

```python
# tests/integration/test_vertex_ai.py
import pytest
from src.summarization import summarize_transcript
from pathlib import Path

@pytest.mark.integration
def test_gemini_flash_generates_script(tmp_path):
    """Test real Gemini API call for script generation."""

    # Create test input
    input_file = tmp_path / "transcript.txt"
    input_file.write_text("This is a test transcript about AI.")

    output_file = tmp_path / "script.txt"

    # Real API call (costs money!)
    summarize_transcript(
        input_file=str(input_file),
        output_file=str(output_file),
        model="gemini-2.5-flash"
    )

    # Assertions
    assert output_file.exists()

    output_text = output_file.read_text()
    assert len(output_text) > 100  # Should generate substantial content
    assert "AI" in output_text  # Should reference input topic

@pytest.mark.integration
def test_imagen_generates_image(tmp_path):
    """Test real Imagen API call."""

    output_file = tmp_path / "test_image.png"

    from src.image_creation import generate_single_image

    # Real API call (costs ~$0.04)
    generate_single_image(
        prompt="A serene mountain landscape",
        output_path=str(output_file)
    )

    # Assertions
    assert output_file.exists()
    assert output_file.stat().st_size > 10000  # Should be substantial PNG
```

**Important:**
- ✅ Mark with `@pytest.mark.integration`
- ✅ Document that test costs money
- ✅ Use `tmp_path` fixture for file operations
- ✅ Test real behavior (API responses, file outputs)

### Example: Speech-to-Text Integration Test

```python
@pytest.mark.integration
def test_speech_to_text_transcription(tmp_path):
    """Test real Speech-to-Text API (long-running)."""

    # Use small test audio file to minimize cost
    test_audio = "tests/fixtures/short_audio.flac"  # 10 seconds

    from src.transcription import transcribe_audio

    # Real API call (costs ~$0.006 for 10 seconds)
    # May take 30-60 seconds to complete
    transcript = transcribe_audio(test_audio)

    # Assertions
    assert transcript
    assert len(transcript) > 10  # Should have content
    # Check for expected words if using known test audio
```

**Cost consideration:** Use small test files to minimize cost.

---

## Test Fixtures

### pytest Configuration

```python
# tests/conftest.py
import pytest
from pathlib import Path

@pytest.fixture
def sample_transcript():
    """Sample transcript for testing."""
    return """
    This is a sample YouTube transcript.
    It contains multiple sentences.
    We can use this for testing script generation.
    """

@pytest.fixture
def sample_script():
    """Sample script for testing TTS."""
    return """
    Welcome to this video about AI.

    Today we'll explore how artificial intelligence is changing the world.

    Let's dive in!
    """

@pytest.fixture
def sample_image_prompts():
    """Sample image prompts for testing."""
    return [
        {"prompt": "A futuristic city skyline", "duration": 10.5},
        {"prompt": "An AI robot working", "duration": 10.5},
        {"prompt": "Data visualization graphics", "duration": 10.5}
    ]

@pytest.fixture
def mock_config():
    """Mock configuration for testing."""
    return {
        "models": {
            "gemini-2.5-flash": {
                "name": "gemini-2.5-flash",
                "temperature": 0.7
            }
        },
        "voices": {
            "en-US-Studio-O": "Female"
        }
    }
```

---

## Manual Testing Strategy

### Full Pipeline Test

**Process:**
1. Create new project via CLI or GUI
2. Paste sample text or YouTube URL
3. Run through all stages:
   - Generate script
   - Approve script
   - Generate audio
   - Approve audio
   - Generate metadata
   - Generate image prompts
   - Generate images (5-10 min)
   - Compose video
   - Download/review final video

**What to check:**
- Each stage produces expected output
- Approval gates work (can't proceed without approval)
- Files are created in correct locations
- Video quality is acceptable
- Audio syncs with images
- No crashes or errors

**Test projects:**
- Short video (~1 min) - Fast iteration
- Long video (~5 min) - Edge cases
- Different topics - Model quality
- Different voices - TTS quality

### Regression Testing

**When making changes:**
1. Run unit tests (`pytest tests/unit/`)
2. Run integration tests (`pytest -m integration`) if API-related
3. Full manual pipeline test on known-good project
4. Compare output quality to baseline

---

## What We Test vs Don't Test

### ✅ What We Test

**Unit Tests:**
- Prompt construction logic
- Response parsing (JSON extraction)
- Text chunking algorithm
- File path generation
- Error handling paths
- Data transformations

**Integration Tests:**
- Real API connectivity
- Authentication works
- Model outputs are reasonable
- File uploads succeed (GCS)

**Manual Tests:**
- Full pipeline end-to-end
- User workflows (GUI)
- Output quality (subjective)

### ❌ What We Don't Test (Yet)

**Not tested:**
- Streamlit GUI pages (UI testing is complex)
- MoviePy video composition (assumes library works)
- yt-dlp download (assumes tool works)
- Edge cases (extremely long videos, unusual formats)
- Performance benchmarks
- Security (API key exposure, etc.)

**Future improvements:**
- GUI tests with Streamlit testing library
- Performance tests (track pipeline duration)
- Load tests (multiple concurrent projects)
- Security audits (credential handling)

---

## CI/CD Integration (Not Yet Implemented)

### Potential GitHub Actions Workflow

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - name: Set up Python
        uses: actions/setup-python@v2
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pytest pytest-cov

      - name: Run unit tests
        run: pytest tests/unit/ -v --cov=src

  integration-tests:
    runs-on: ubuntu-latest
    # Only run on main branch (to avoid excessive API costs)
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v2

      - name: Set up Python
        uses: actions/setup-python@v2
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run integration tests
        env:
          GOOGLE_APPLICATION_CREDENTIALS: ${{ secrets.GCP_CREDENTIALS }}
        run: pytest -m integration -v
```

**Key points:**
- Unit tests run on every push (fast, free)
- Integration tests only on main branch (slow, costly)
- Credentials via GitHub Secrets

---

## Test Coverage Goals

**Current (estimated):**
- Core modules: ~60% coverage
- Edge cases: ~20% coverage
- Integration: ~30% coverage

**Target:**
- Core modules: 80% coverage (unit + integration)
- Edge cases: 50% coverage
- Integration: 50% coverage (key workflows)

**Run coverage:**
```bash
pytest tests/unit/ --cov=src --cov-report=html
open htmlcov/index.html
```

---

## Testing Cost Estimation

### Integration Test Costs (per run)

| Test | API Calls | Approx Cost |
|------|-----------|-------------|
| Gemini generation | 3-5 calls | $0.05-0.10 |
| Imagen generation | 1-2 images | $0.04-0.08 |
| TTS synthesis | 1-2 calls | $0.02-0.05 |
| Speech-to-Text | 1 call (short audio) | $0.01 |
| **Total per run** | | **~$0.12-0.24** |

**Monthly cost (daily CI runs):** ~$3.60-7.20

**Recommendation:** Run integration tests sparingly (main branch only, or nightly).

---

## Key Learnings

1. **Mock external APIs for unit tests** - Fast, free, reliable
2. **Integration tests are expensive** - Run selectively
3. **Manual testing still crucial** - Subjective quality matters
4. **Test business logic, not libraries** - Don't test MoviePy, test our usage
5. **Fixtures improve test quality** - Reusable test data
6. **Mark integration tests explicitly** - Easy to run/skip
7. **Cost awareness** - Integration tests cost real money

---

## Testing Workflow

### Developer Workflow

```bash
# 1. Write feature code
vim src/new_feature.py

# 2. Write unit tests
vim tests/unit/test_new_feature.py

# 3. Run unit tests (fast feedback)
pytest tests/unit/test_new_feature.py -v

# 4. Run all unit tests
pytest tests/unit/ -v

# 5. If touching API integration, run integration tests
pytest -m integration -v

# 6. Manual test via GUI/CLI
python main.py
```

### Pre-Release Workflow

```bash
# 1. All unit tests
pytest tests/unit/ -v --cov=src

# 2. All integration tests
pytest -m integration -v

# 3. Full manual pipeline test
# - Create project
# - Run all stages
# - Verify output quality

# 4. Regression test
# - Run on known-good project
# - Compare output to baseline
```

---

_Extracted: 2026-02-22_

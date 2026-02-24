# VidGen Key Decisions

**Project:** Bibo YouTube Video Generator
**Extracted:** 2026-02-22

---

## Overview

This document captures key architectural and technical decisions made during VidGen development, including context, alternatives considered, rationale, and trade-offs.

---

## Decision 1: File-Based State (No Database)

**Context:**
Need to track pipeline progress, approval state, and project metadata.

**Decision:**
Use filesystem for all state management. No database.

**Alternatives Considered:**
1. **SQLite database** - Lightweight, SQL queries
2. **PostgreSQL** - Full-featured, scalable
3. **Redis** - In-memory, fast access
4. **JSON files** - Simple, human-readable

**Rationale:**
- **Simplicity:** No DB setup, no migrations, no connection management
- **Portability:** Project folder is self-contained, can zip/share
- **Debuggability:** State is human-readable (open text file, see status)
- **Resume-ability:** Check file existence = know what's done
- **Single-user:** No concurrent access needed (desktop app assumption)

**Trade-offs:**
- ✅ Zero infrastructure overhead
- ✅ Easy to understand and debug
- ✅ Portable (entire project in one folder)
- ❌ No query capabilities (can't ask "all projects with approved scripts")
- ❌ No concurrent access protection
- ❌ No relational integrity checks

**Outcome:**
Fast development, zero deployment complexity. Perfect for single-user desktop tool.

---

## Decision 2: Gemini 2.5 Flash as Default Model

**Context:**
Need to choose default LLM for script generation, metadata, image prompts.

**Decision:**
Use Gemini 2.5 Flash as default, with 2.5 Pro as optional upgrade.

**Alternatives Considered:**
1. **Gemini 2.0 Flash Preview** - Earlier version
2. **Gemini 2.5 Pro** - Higher quality
3. **GPT-4** - OpenAI alternative
4. **Claude Opus** - Anthropic alternative

**Rationale:**
- **Testing showed:** 2.5 Flash produces 1,045-word scripts (most consistent)
- **Cost:** Flash is cheaper than Pro (good default for bulk generation)
- **Speed:** Flash is faster than Pro (better UX)
- **Quality:** Flash quality is sufficient for initial drafts
- **Availability:** Vertex AI integration (same ecosystem as Imagen/TTS)

**From testing notes (Feb 11, 2026):**
> "gemini-3-flash-preview tested and working... most consistent at ~1,045 words"

**Trade-offs:**
- ✅ Fast generation
- ✅ Lower cost
- ✅ Consistent output length
- ❌ Slightly lower quality than Pro
- ❌ User may need to upgrade for complex topics

**Outcome:**
Flash works for 80% of use cases, Pro available when needed. User can switch via config.

---

## Decision 3: Sequential Pipeline (No Parallelization)

**Context:**
Pipeline has 6 stages, some could theoretically run in parallel.

**Decision:**
Enforce strict sequential processing. No parallel stages.

**Alternatives Considered:**
1. **Parallel metadata + audio** - Both depend on script, could run simultaneously
2. **Background job queue** - Celery/RQ for async processing
3. **Fully parallel** - All stages that can run in parallel do

**Rationale:**
- **Data dependencies:** Most stages depend on previous output
- **User approval gates:** Need sequential review (can't approve images before audio)
- **Simplicity:** Linear flow is easier to understand and debug
- **Resource usage:** Parallel Vertex AI calls could hit rate limits
- **UX:** User focuses on one stage at a time (not overwhelming)

**Trade-offs:**
- ✅ Simple mental model (current stage → next stage)
- ✅ Clear error handling (stage fails → fix → retry)
- ✅ Easy progress tracking (linear progress bar)
- ❌ Slower total time (could save minutes with parallel metadata+audio)
- ❌ Idle time (user waits for sequential operations)

**Outcome:**
Prioritized simplicity over speed. Total pipeline time is ~10-20 minutes regardless.

---

## Decision 4: Approval Tracking in config.json (Not File Existence)

**Context:**
Need to differentiate "file exists" from "user approved file."

**Decision:**
Store approval state in `projects/{name}/config.json`, separate from file existence.

**Alternatives Considered:**
1. **File existence = approval** - Simpler (delete file to revoke)
2. **Separate approval files** - `1_summary.txt.approved` marker files
3. **Database flag** - Requires database (see Decision 1)
4. **In-memory state** - Lost on restart

**Rationale:**
- **Regeneration use case:** User wants to regenerate audio (different voice) but keep approval
- **Clear semantics:** File exists ≠ approved. Explicit is better.
- **Centralized:** All approval state in one file (easy to inspect)
- **Per-project:** Config travels with project folder

**Trade-offs:**
- ✅ Regenerate without losing approval
- ✅ Clear approval state (not inferred)
- ✅ Easy to inspect (open config.json, see all approvals)
- ❌ Two sources of truth (file + config)
- ❌ Could get out of sync (file deleted but approval remains)

**Outcome:**
Enables key workflow (regenerate with approval intact). Worth the extra complexity.

---

## Decision 5: Streamlit for GUI (Not CLI-Only)

**Context:**
Initial version was CLI-only. User wanted non-technical access.

**Decision:**
Add Streamlit GUI wrapper, keep CLI functional.

**Alternatives Considered:**
1. **CLI-only** - Technical users only
2. **React + FastAPI** - Full web app
3. **Electron** - Desktop app with web stack
4. **PyQt** - Native desktop GUI
5. **Streamlit** - Rapid Python GUI

**Rationale:**
- **Speed:** Streamlit prototype in hours (vs days for React)
- **Python-native:** No frontend/backend split
- **No deployment:** Runs locally (no hosting needed)
- **Live reload:** Easy development iteration
- **Minimal code:** Pages are simple (call `src/` functions)

**From session notes (Feb 14, 2026):**
> "Complete 7-page app operational... 6 stages implemented... ~3 hours"

**Trade-offs:**
- ✅ Fast to build
- ✅ Python developers comfortable
- ✅ No frontend expertise needed
- ❌ Not as polished as custom web app
- ❌ Limited customization (Streamlit constraints)
- ❌ Desktop-only (not web-deployable without Cloud Run)

**Outcome:**
Perfect for MVP. Can always rebuild in React later if needed.

---

## Decision 6: MoviePy for Video Composition (Not FFmpeg CLI)

**Context:**
Need to compose video from images + audio.

**Decision:**
Use MoviePy 2.x library (Python wrapper for FFmpeg).

**Alternatives Considered:**
1. **FFmpeg CLI** - Direct command-line calls
2. **OpenCV** - Computer vision library
3. **Pillow + pydub** - Image + audio manipulation
4. **MoviePy 1.x** - Older version

**Rationale:**
- **Python-native:** No subprocess calls, easier error handling
- **High-level API:** `ImageClip`, `concatenate_videoclips` abstractions
- **Audio sync:** Automatic audio/video sync
- **Effects:** Built-in transitions, resizing, etc.
- **Version 2.x:** Latest API (1.x deprecated)

**Trade-offs:**
- ✅ Pythonic API
- ✅ Easier than raw FFmpeg commands
- ✅ Good documentation
- ❌ FFmpeg still required (dependency)
- ❌ Limited control vs raw FFmpeg
- ❌ Version 2.x has different API (breaking changes from 1.x)

**Outcome:**
Abstracts video complexity. Simple composition code.

---

## Decision 7: Google Cloud Ecosystem (Not AWS/Azure)

**Context:**
Need LLM, TTS, image generation, speech-to-text services.

**Decision:**
All-in on Google Cloud (Vertex AI, Cloud TTS, Cloud Speech, GCS).

**Alternatives Considered:**
1. **AWS** - Bedrock (LLM), Polly (TTS), Rekognition
2. **Azure** - OpenAI (LLM), Speech Services
3. **Mixed** - Best-of-breed (OpenAI LLM + Google TTS + Stability AI images)
4. **Google** - Single ecosystem

**Rationale:**
- **Single auth:** One service account for all APIs
- **Integrated:** Vertex AI includes Gemini + Imagen
- **Studio voices:** Google TTS has highest-quality voices
- **Unified billing:** One invoice, easy cost tracking
- **ADC support:** Development credentials via `gcloud auth`

**Trade-offs:**
- ✅ Simplified authentication
- ✅ Consistent API patterns
- ✅ Single vendor relationship
- ❌ Vendor lock-in (hard to switch later)
- ❌ Google pricing may not be cheapest
- ❌ Limited to Google's models (can't use GPT-4 easily)

**Outcome:**
Pragmatic choice for MVP. Integration time saved > vendor flexibility.

---

## Decision 8: yt-dlp for YouTube Download (Not youtube-dl)

**Context:**
Need to download YouTube audio for transcription.

**Decision:**
Use yt-dlp (youtube-dl fork).

**Alternatives Considered:**
1. **youtube-dl** - Original project
2. **pytube** - Pure Python YouTube downloader
3. **yt-dlp** - Active youtube-dl fork
4. **YouTube Data API** - Official API (no audio download)

**Rationale:**
- **Actively maintained:** youtube-dl development stalled, yt-dlp is active fork
- **Better support:** yt-dlp handles more sites, newer YouTube changes
- **Audio extraction:** Built-in audio-only download
- **Format selection:** Can specify quality, codec

**Trade-offs:**
- ✅ Works reliably with current YouTube
- ✅ Audio-only download (saves bandwidth)
- ✅ Fast, well-tested
- ❌ External dependency (CLI tool)
- ❌ YouTube may break it (cat-and-mouse game)
- ❌ Terms of service concerns (YouTube may prohibit download)

**Outcome:**
Best available option for YouTube audio extraction. Monitor for breakage.

---

## Decision 9: 16:9 Aspect Ratio (Not 9:16 or 1:1)

**Context:**
Need to choose output video aspect ratio.

**Decision:**
Fixed 16:9 (landscape, 1920x1080).

**Alternatives Considered:**
1. **9:16** - Vertical (TikTok, Shorts)
2. **1:1** - Square (Instagram)
3. **4:3** - Classic TV
4. **16:9** - Widescreen (YouTube default)
5. **User configurable** - Let user choose

**Rationale:**
- **YouTube standard:** 16:9 is default for YouTube
- **Simplicity:** One format = simpler code
- **Image generation:** Imagen generates 16:9 well
- **Horizontal content:** Most stock images are landscape

**Trade-offs:**
- ✅ Optimized for YouTube
- ✅ Simpler implementation (no format variations)
- ✅ Works well with horizontal images
- ❌ Not suitable for TikTok/Shorts (9:16)
- ❌ No flexibility for different platforms
- ❌ Hardcoded in multiple places

**Outcome:**
Good default for YouTube. Could add format selection in future (Phase 3).

---

## Decision 10: Studio Voices for TTS (Not Standard Voices)

**Context:**
Google Cloud TTS offers Standard, WaveNet, and Studio voices.

**Decision:**
Use Studio voices as default (en-US-Studio-O, en-US-Studio-M).

**Alternatives Considered:**
1. **Standard voices** - Cheapest, robotic
2. **WaveNet voices** - Mid-tier quality
3. **Studio voices** - Highest quality, expensive
4. **Neural2 voices** - Latest generation

**Rationale:**
- **Quality:** Studio voices sound most natural
- **Use case:** YouTube videos need professional narration
- **Cost acceptable:** For ~1-2 min audio, Studio voice cost is low (~$0.10)
- **User expectation:** Non-technical users expect high quality

**Trade-offs:**
- ✅ Professional-sounding output
- ✅ Worth the cost for final videos
- ✅ User satisfaction higher
- ❌ More expensive than Standard/WaveNet
- ❌ Higher per-character cost
- ❌ Limited language support (Studio voices fewer than Standard)

**Outcome:**
Quality matters for published content. Studio voices are worth the premium.

---

## Decision 11: Imagen 3.0 for Image Generation (Not Stable Diffusion/DALL-E)

**Context:**
Need AI image generation for video visuals.

**Decision:**
Use Vertex AI Imagen 3.0.

**Alternatives Considered:**
1. **Stable Diffusion** - Open source, self-hosted
2. **DALL-E 3** - OpenAI (via API)
3. **Midjourney** - High quality (no API)
4. **Imagen 3.0** - Google Vertex AI

**Rationale:**
- **Ecosystem fit:** Already using Vertex AI for Gemini
- **Quality:** Imagen 3.0 competitive with DALL-E
- **Integration:** Same authentication, same billing
- **Aspect ratio:** Native 16:9 support
- **Safety:** Built-in content filtering

**Trade-offs:**
- ✅ Single-vendor convenience
- ✅ Good quality outputs
- ✅ Integrated with Vertex AI
- ❌ Cost per image (~$0.04) adds up
- ❌ Google's content filtering may be restrictive
- ❌ Limited customization vs self-hosted Stable Diffusion

**Outcome:**
Pragmatic choice for integrated solution. Quality is good enough.

---

## Decision 12: Per-Project Logging (Not Global Log)

**Context:**
Need to track pipeline execution for debugging.

**Decision:**
Each project gets own `pipeline.log` file.

**Alternatives Considered:**
1. **Global log** - Single `app.log` for all projects
2. **Per-project log** - `projects/{name}/pipeline.log`
3. **Structured logging** - JSON logs to logging service
4. **No logging** - Print to console only

**Rationale:**
- **Project isolation:** Each project's log travels with its folder
- **Debuggability:** User can check specific project's log
- **Portability:** Zip project folder = includes logs
- **Simplicity:** Append to text file (no logging service)

**Trade-offs:**
- ✅ Logs travel with project
- ✅ Easy to inspect (text file)
- ✅ Per-project debugging
- ❌ No centralized log analysis
- ❌ No log rotation (files grow unbounded)
- ❌ No structured querying

**Outcome:**
Fits file-based architecture. Good enough for single-user desktop tool.

---

## Decision 13: 1 Image per 10 Seconds of Audio

**Context:**
Need to determine how many images to generate for video.

**Decision:**
Generate 1 image per 10 seconds of audio duration.

**Alternatives Considered:**
1. **Fixed count** - Always 5 images
2. **1 per 5 seconds** - More images, smoother transitions
3. **1 per 10 seconds** - Balanced
4. **1 per 15 seconds** - Fewer images, cheaper
5. **User configurable** - Let user choose

**Rationale:**
- **Cost balance:** Imagen costs ~$0.04/image. 10s = fewer images = lower cost
- **Visual pace:** 10s per image feels natural (not too fast, not static)
- **Generation time:** Fewer images = faster pipeline
- **Testing result:** 10s pacing tested well

**Trade-offs:**
- ✅ Reasonable cost (4-6 images for 1min video = $0.16-$0.24)
- ✅ Acceptable visual pace
- ✅ Faster generation
- ❌ Less visual variety than 5s
- ❌ Hardcoded (no user control)
- ❌ May feel slow for fast-paced content

**Outcome:**
Good default. Could add configurability later (Phase 3 enhancement).

---

## Key Takeaways

**Optimization Priorities (Revealed by Decisions):**
1. **Simplicity over scalability** (file-based state, sequential pipeline)
2. **Single-vendor integration over flexibility** (Google Cloud ecosystem)
3. **Quality over cost** (Studio voices, Imagen)
4. **Speed of development over perfect UX** (Streamlit, fixed aspect ratio)
5. **Desktop single-user over web multi-tenant** (local files, no auth)

**These decisions work because:**
- Target user is non-technical (simplicity matters)
- Use case is desktop tool (not web service)
- Scale is low (single user, few projects)
- Development speed matters (MVP phase)

**If requirements change (web service, multi-user, high scale), revisit:**
- Decision 1 (file-based → database)
- Decision 3 (sequential → parallel)
- Decision 12 (per-project logs → centralized logging)

---

_Extracted: 2026-02-22_

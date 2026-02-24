# APP BRIEF: Pepper's Video Generation Rig

> **Status:** LOCKED (v1)
> **Version:** 1.1
> **Author:** Architect Agent
> **Date:** 2026-01-31

---

## 1. Mission Statement

A desktop-class Streamlit application that transforms text content (from YouTube transcripts or raw text dumps) into fully produced documentary-style videos with AI-generated narration, visuals, and metadata. Built for internal use by Tony, Coach, and VA Joane — not a consumer product.

---

## 2. Hero Action

**"Paste text → Click through tabs → Download finished video"**

User provides source content, approves each pipeline stage with human-in-the-loop verification, and receives a publish-ready MP4 with title, description, and hashtags.

---

## 3. User & Auth Scope

- **Target Audience:** Internal team (Tony, Coach, Joane)
- **Auth Method:** None (local app, single user)
- **Roles:** Single role — full access to all features

---

## 4. Complexity Class

- **App Type:** Pipeline Orchestrator (sequential stages with dependencies)
- **AI Required:** Yes — LLM (script, metadata, prompts), TTS (audio), Image Gen (visuals)
- **Real-time:** No — batch processing per stage

---

## 5. Domain Concepts (Core Objects)

| Object   | Key Fields                                  | Lifecycle                        | Storage                            |
| -------- | ------------------------------------------- | -------------------------------- | ---------------------------------- |
| Project  | name, created_at, profile, status           | Created → In Progress → Complete | `projects/{name}/`                 |
| Script   | content, word_count, approved               | Generated → Reviewed → Approved  | `1_summary.txt`                    |
| Audio    | file_path, duration_seconds, approved       | Generated → Previewed → Approved | `2_audio.mp3`                      |
| Metadata | titles[], description, hashtags, approved   | Generated → Reviewed → Approved  | `3_metadata.txt`                   |
| ImageSet | prompts[], images[], count                  | Generated → Reviewed → Approved  | `4_image_prompts.txt`, `5_images/` |
| Video    | file_path, duration, resolution             | Composed → Complete              | `6_final_video.mp4`                |
| Profile  | name, aspect_ratio, word_target, dimensions | Static config                    | In-app config                      |
| Log      | timestamp, stage, message, level            | Append-only                      | `pipeline.log`                     |

---

## 6. Pipeline Flow

```
[INPUT STAGE]
    ├── YouTube URL → Transcriber → transcript
    └── Text Drop → raw text
            ↓
[SCRIPT STAGE]
    Script Writer → 1_summary.txt → [Preview] → [Approve/Redo]
            ↓
    ┌───────┴───────┐
    ↓               ↓
[METADATA]      [AUDIO]
3_metadata.txt  2_audio.mp3
[Preview]       [Play/Preview]
[Approve/Redo]  [Approve/Redo]
                    ↓
              [IMAGES]
              (depends on audio duration)
              5_images/
              [Preview Gallery]
              [Approve/Redo]
                    ↓
              [VIDEO]
              (depends on audio + images)
              6_final_video.mp4
              [Play/Preview]
              [Download]
```

---

## 7. Scope Locks

### ✅ In Scope (v1):

- YouTube URL transcription (existing, working)
- Text drop input (new)
- Script generation with human approval
- Metadata generation (titles, description, hashtags)
- Audio generation via Google Cloud TTS
- Image generation via Vertex AI Imagen
- Video composition via MoviePy
- Human-in-the-loop at every stage (Preview → Approve/Redo)
- Editable system instructions per stage
- Profile selector (only YouTube Standard enabled)
- Project-based file storage
- Basic progress indicator + log file access
- Audio playback preview
- Single video at a time

### 🚫 Out of Scope (v1):

1. Windows packaging / installer
2. Batch processing / queue
3. YouTube Shorts, Instagram, TikTok profiles (locked in UI)
4. Live scrolling log panel (Vercel-style)
5. Duration slider / word count calculator
6. Thumbnail generation
7. Translation layer
8. Multi-user / auth
9. Cloud deployment
10. Auto-upload to YouTube

---

## 8. Tech Stack

| Layer         | Technology                                  |
| ------------- | ------------------------------------------- |
| Frontend      | Streamlit                                   |
| Runtime       | Python 3.11+                                |
| LLM           | Vertex AI Gemini 2.0 Flash / 2.5 Pro        |
| TTS           | Google Cloud Text-to-Speech                 |
| Image Gen     | Vertex AI Imagen 4.0                        |
| Video         | MoviePy + FFmpeg                            |
| Transcription | OpenAI Whisper (existing) OR Vertex AI (v2) |
| Credentials   | Service account JSON (bundled)              |

---

## 9. State Management

**Pattern:** File-based state with explicit approval tracking. No database.

- Each project = folder under `projects/`
- Each stage writes numbered output file
- **Approval state tracked in `config.json`** — file existence alone does NOT imply approval
- Tab enablement determined by: file exists AND `approved: true` in config
- Resumable: restart app, pick project, continue from last completed stage

**File Structure:**

```
projects/{project_name}/
├── 0_transcript.txt       # (if YouTube input)
├── 1_summary.txt          # Script
├── 2_audio.mp3            # Narration
├── 3_metadata.txt         # Titles/desc/hashtags
├── 4_image_prompts.txt    # Visual prompts
├── 5_images/              # Generated PNGs
│   ├── 001.png
│   ├── 002.png
│   └── _image_log.json
├── 6_final_video.mp4      # FINAL OUTPUT
├── pipeline.log           # Timestamped execution log
└── config.json            # Project settings + approval states
```

**config.json Approval Tracking:**

```json
{
  "project_name": "ai-trends-2026",
  "profile": "youtube_standard",
  "created_at": "2026-01-31T13:00:00Z",
  "approvals": {
    "script": true,
    "metadata": true,
    "audio": true,
    "images": false,
    "video": false
  }
}
```

---

## 10. Configuration

### Profiles (v1)

```json
{
  "youtube_standard": {
    "name": "YouTube Standard",
    "enabled": true,
    "aspect_ratio": "16:9",
    "resolution": [1920, 1080],
    "word_target": [900, 1000],
    "seconds_per_image": 12
  },
  "youtube_shorts": {
    "name": "YouTube Shorts",
    "enabled": false,
    "aspect_ratio": "9:16",
    "resolution": [1080, 1920],
    "word_target": [150, 200],
    "seconds_per_image": 5
  }
}
```

### System Instructions

- Stored as editable text per stage
- Defaults loaded from `config/prompts/`
- User edits saved to project's `config.json`

---

## 11. Success Definition

**Demo in 3 steps:**

1. Tony pastes article text into Starter tab
2. Tony clicks through each tab, approving outputs
3. Tony downloads `6_final_video.mp4` ready for YouTube upload

**Metrics (manual for v1):**

- Time from input to final video
- Number of "Redo" clicks per stage
- Video successfully plays in VLC

---

## 12. Failure Modes to Prevent

1. **Lost work** → File-based state ensures resumability
2. **Bad script propagates** → Human approval gate before proceeding
3. **API failures silent** → Log file + error display in status panel
4. **Wrong aspect ratio** → Profile locks all dimensions together
5. **Runaway costs** → One video at a time, manual approval at each stage
6. **Accidental overwrite of approved content** → Confirmation dialog required before regenerating any approved artifact

---

## 13. Planning State

| Item                             | Type          | Notes                       |
| -------------------------------- | ------------- | --------------------------- |
| Streamlit for v1                 | 🔒 Decision   | Locked                      |
| File-based state                 | 🔒 Decision   | Locked                      |
| Human-in-the-loop                | 🔒 Decision   | Locked                      |
| Gemini via Vertex                | 🔒 Decision   | Locked                      |
| Google Cloud TTS                 | 🔒 Decision   | Locked                      |
| YouTube Standard only            | 🔒 Decision   | Other profiles locked in UI |
| Approval tracking in config.json | 🔒 Decision   | Locked                      |
| Whisper for transcription        | ⚠️ Assumption | May swap to Vertex in v2    |
| 900-1000 word target             | ⚠️ Assumption | May add slider in v2        |

---

## 14. Handoff Checklist

- [x] Designer receives: `APP_BRIEF.md` + `UI_SPEC.md`
- [x] Engineer receives: `APP_BRIEF.md` + existing codebase
- [ ] All system instruction defaults documented
- [ ] Vertex credentials confirmed working

---

## ✅ CHANGES MADE (3 Surgical Augments)

| #   | Change                                                            | Location                     |
| --- | ----------------------------------------------------------------- | ---------------------------- |
| 1   | `seconds_per_image`: 20 → **12**                                  | Section 10 (Profiles)        |
| 2   | Added explicit approval tracking in `config.json`                 | Section 9 (State Management) |
| 3   | Added Failure Mode #6: "Accidental overwrite of approved content" | Section 12                   |

Also bumped version to **1.1** and status to **LOCKED (v1)**.

---

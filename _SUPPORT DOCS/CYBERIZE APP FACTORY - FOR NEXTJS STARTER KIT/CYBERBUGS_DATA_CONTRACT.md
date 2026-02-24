# CYBERBUGS_DATA_CONTRACT.md

> **Purpose:**
> This document defines the **shape of the data** used by the CyberBugs application.
>
> It is **not** a database schema.
> It is **not** Supabase-specific.
>
> This is an agreement between **frontend and backend** so both evolve toward the same target.

---

## 1. Why This Document Exists

We are building the UI first using mock data.

To avoid redoing work later, we must agree upfront on:

* what data exists
* how entities relate
* what fields are required
* what fields are optional

This document is the **source of truth** for data shape during:

* UI development
* mock data creation
* service layer design
* backend implementation later

If it's not defined here, **UI should not assume it exists**.

---

## 2. Core Entities

CyberBugs has the following core entities:

1. App
2. App Version
3. Bug
4. Bug Attachment
5. Bug Comment (optional in v1)
6. User (comes from Supabase Auth)

---

## 3. Entity Definitions

### 3.1 App

Represents a software application being tracked.

**Fields**

* `id` (string, required)
* `name` (string, required)
* `description` (string, optional)
* `is_active` (boolean, required)

**Audit Fields**

* `created_at` (timestamp, required)
* `created_by` (user_id, required)

**Notes**

* An App can have many Versions
* An App can have many Bugs

---

### 3.2 App Version

Represents a deployment/version of an App.

**Fields**

* `id` (string, required)
* `app_id` (string, required)
* `version_label` (string, required)
  * examples: `v1.0.3`, `2025.01.02`, `build-109`
* `release_notes` (string, optional)
* `deployed_at` (timestamp, optional)

**Audit Fields**

* `created_at` (timestamp, required)
* `created_by` (user_id, required)

**Notes**

* Versions are added by Admin or SuperAdmin
* Bugs may optionally reference a version

---

### 3.3 Bug

Represents a reported issue.

**Core Fields**

* `id` (string, required)
* `app_id` (string, required)
* `version_id` (string, optional)
* `title` (string, required)
* `description` (string, optional)

**Reproduction Fields**

* `steps_to_reproduce` (string, optional) — numbered steps to recreate the bug
* `video_url` (string, optional) — Loom, ClickUp Clip, or YouTube link
* `expected_behavior` (string, optional) — what should happen
* `actual_behavior` (string, optional) — what actually happens

**Status Fields**

* `status` (enum, required)
  * `open`
  * `in_progress`
  * `blocked`
  * `resolved`
  * `closed`

* `severity` (enum, required)
  * `low`
  * `medium`
  * `high`
  * `critical`

* `environment` (enum, optional)
  * `production`
  * `staging`
  * `development`

**Reporter Fields**

* `reported_by` (user_id, required)

**Audit Fields**

* `created_at` (timestamp, required)
* `updated_at` (timestamp, optional)

**Notes**

* Testers can create bugs
* Only Admin / SuperAdmin can change status to `resolved` or `closed`
* `video_url` is for external screen recordings (Loom, ClickUp Clip, etc.)
* Screenshots and file uploads use Bug Attachment (see 3.4)

---

### 3.4 Bug Attachment

Represents a file (screenshot, document, log file) attached to a bug report.

**Fields**

* `id` (string, required)
* `bug_id` (string, required)
* `file_url` (string, required) — URL from Supabase Storage
* `file_name` (string, required) — original filename
* `file_type` (string, optional) — MIME type (e.g., `image/png`, `application/pdf`)
* `file_size` (number, optional) — size in bytes
* `uploaded_by` (user_id, required)

**Audit Fields**

* `created_at` (timestamp, required)

**Storage Notes**

* Files are uploaded to **Supabase Storage**
* `file_url` is the public or signed URL returned by Supabase
* Supported types: images (png, jpg, gif), documents (pdf, txt), logs (txt, log)

---

### 3.5 Bug Comment (Optional – Phase 2)

Represents discussion or updates on a bug.

**Fields**

* `id` (string, required)
* `bug_id` (string, required)
* `comment` (string, required)
* `commented_by` (user_id, required)

**Audit Fields**

* `created_at` (timestamp, required)

---

### 3.6 User (Auth-backed, mocked in Demo Mode)

Users come from Supabase Auth in production. During Demo Mode, users are mocked locally.

**Fields (used by app)**

* `id` (string, required)
* `email` (string, required)
* `display_name` (string, optional) — shown in UI (e.g., "Tony Stark")
* `avatar_url` (string, optional) — profile picture URL
* `role` (enum, required)
  * `member`
  * `admin`
  * `super_admin`

**Notes**

* Role is currently stored in Supabase Auth metadata (`user_metadata.role`)
* UI and routing rely on this role for RBAC
* RLS enforcement will come in Production Mode

---

## 4. Relationships Summary

| Parent | Relationship | Child |
|--------|--------------|-------|
| App | has many | Versions |
| App | has many | Bugs |
| Bug | belongs to | App |
| Bug | optionally belongs to | Version |
| Bug | reported by | User |
| Bug | has many | Attachments |
| Bug | has many | Comments |

---

## 5. Required Audit Rules (Non-Negotiable)

Every persistent entity must include:

* `created_at` (timestamp)
* `created_by` (user_id) — except Bug Attachment which uses `uploaded_by`

This applies even during mock data creation.

---

## 6. Derived Fields (Computed at Runtime)

These fields are **NOT stored**. They are calculated by the service layer or UI.

### App (Computed)

* `total_bugs` — count of all bugs for this app
* `open_bugs` — count of bugs with status = open
* `critical_bugs` — count of bugs with severity = critical
* `health_score` — percentage of resolved/closed bugs

### Dashboard (Computed)

* `bugs_by_severity` — aggregation for pie chart
* `bugs_by_status` — aggregation for status breakdown
* `bugs_over_time` — aggregation for trend chart
* `recent_activity` — latest bug reports and status changes

### User (Computed)

* `bugs_reported` — count of bugs reported by this user

---

## 7. What This Contract Does NOT Decide

This document does **not** define:

* database tables or column types
* indexes or foreign keys
* Supabase RLS policies
* API routes or endpoints
* validation rules or libraries
* error handling

Those are implementation details handled in later phases.

---

## 8. Rules for Frontend Development

* UI must only use fields defined in this document
* Mock data must match this contract exactly
* Service layer functions must return data shaped according to this contract
* No UI component should invent new fields
* Derived fields must be computed, not stored in mock data

---

## 9. Mock Data Examples

### App Example

```json
{
  "id": "app-001",
  "name": "DockBloxx",
  "description": "Container management platform for shipping logistics",
  "is_active": true,
  "created_at": "2025-12-01T10:00:00Z",
  "created_by": "user-admin-001"
}
```

### App Version Example

```json
{
  "id": "ver-001",
  "app_id": "app-001",
  "version_label": "v2.1.0",
  "release_notes": "Added bulk container import feature",
  "deployed_at": "2025-12-10T14:00:00Z",
  "created_at": "2025-12-10T13:45:00Z",
  "created_by": "user-admin-001"
}
```

### Bug Example

```json
{
  "id": "bug-001",
  "app_id": "app-001",
  "version_id": "ver-001",
  "title": "Login button unresponsive on mobile",
  "description": "Users report the login button does nothing when tapped on iOS Safari.",
  "steps_to_reproduce": "1. Open app on iPhone\n2. Navigate to login page\n3. Enter credentials\n4. Tap Login button\n5. Nothing happens",
  "video_url": "https://www.loom.com/share/abc123xyz",
  "expected_behavior": "User should be logged in and redirected to dashboard",
  "actual_behavior": "Button does not respond to tap, no error shown",
  "status": "open",
  "severity": "high",
  "environment": "production",
  "reported_by": "user-tester-001",
  "created_at": "2025-12-15T14:30:00Z",
  "updated_at": null
}
```

### Bug Attachment Example

```json
{
  "id": "attach-001",
  "bug_id": "bug-001",
  "file_url": "https://your-project.supabase.co/storage/v1/object/public/attachments/bug-001/screenshot.png",
  "file_name": "screenshot.png",
  "file_type": "image/png",
  "file_size": 245000,
  "uploaded_by": "user-tester-001",
  "created_at": "2025-12-15T14:31:00Z"
}
```

### User Example (Mock)

```json
{
  "id": "user-tester-001",
  "email": "tony@cyberize.io",
  "display_name": "Tony Stark",
  "avatar_url": "https://api.dicebear.com/7.x/avataaars/svg?seed=tony",
  "role": "member"
}
```

```json
{
  "id": "user-admin-001",
  "email": "jarvis@cyberize.io",
  "display_name": "J.A.R.V.I.S.",
  "avatar_url": "https://api.dicebear.com/7.x/avataaars/svg?seed=jarvis",
  "role": "admin"
}
```

---

## 10. Change Policy

If UI or product needs change:

1. **Update this document first**
2. Then update mock data
3. Then update service layer types
4. Then update UI components
5. Backend implementation comes last

This prevents drift between frontend and backend.

---

## 11. Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025-12-31 | Initial contract |
| 1.1 | 2026-01-01 | Added Reproduction Fields, Bug Attachment, Derived Fields, Mock Examples |

---

### END OF DOCUMENT

---

# MBMCnotes — Backend Requirements (Approved)

## 1. Vision
Web app for college. Students/teachers from different faculties/subjects share notes and assignments. Core loop: normal user submits material in current directory -> verifier approves/rejects -> approved material visible to all in that directory.

Backend-only scope. No frontend decisions in this doc.

## 2. Actors
1. **Normal User** (student or teacher, no teacher role distinction)
   - Register/login with email + password
   - Must verify email
   - Can self-serve password reset via email
   - Selects faculty, browses Semester -> Subject -> Folder
   - Submits notes/assignments, views own history

2. **Verifier / Mod** (teacher or trusted student, granted by admin)
   - Same account as normal user + `is_verifier=True`
   - Two switchable modes: `normal` | `moderator`
   - In moderator mode: views scoped pending queue with directory, uploader, content, files, approves/rejects
   - Password reset: manual only. Must ask admin. No email reset flow.

3. **Admin (Repo Owner)**
   - Only creator of Faculty / Semester / Subject / Folder
   - Grants/revokes verifier flag
   - Grants/revokes moderation scopes
   - Manual password reset for verifiers
   - Monitors queue usage

## 3. Information Architecture (fixed)
```
Faculty -> Semester -> Subject -> Folder[type=notes|assignments] -> Submission
```
- No user-created folders.
- Each Subject auto-has exactly 2 Folders: `notes`, `assignments`.
- Published view = `Submission(status=approved)` in that Folder.
- Submit button context: "Submit Notes" in notes Folder, "Submit Assignment" in assignments Folder. API: `POST /folders/:id/submissions`.

## 4. Functional Requirements

### 4.1 Auth
- `register(email, password, faculty_id?)` -> creates unverified user, sends verification email with token/link, 24h expiry.
- `verify-email(token)` -> `is_verified=True`.
- `resend-verification` rate-limited.
- `login` only if `is_verified=True`. Returns JWT.
- `forgot-password / reset-password` for normal users only. If `is_verifier=True`, endpoint returns `403 + CONTACT_ADMIN`.
- Verifier login uses same endpoint. Mode defaults to `normal` on login.

### 4.2 Directory Browsing (read-only)
- `GET /faculties`, `GET /faculties/:id/semesters`, `GET /semesters/:id/subjects`, `GET /subjects/:id/folders`, `GET /folders/:id/items?status=approved`
- `PATCH /me {faculty_id}` to change faculty.
- All browsing requires auth + verified email.

### 4.3 Submissions
Approved limits:
- Allowed types: `pdf, jpg/jpeg, png, webp, docx` only. No video.
- Per-file max: `image 20MB, pdf 100MB, docx 20MB`.
- Per-submission max: `5 files, 100MB total`.
- Validation by MIME + extension + magic bytes. Reject on mismatch.
- Required fields: `title, description?, folder_id (from URL), files[]`.
- Creates `Submission(status=pending)` + `File` rows.
- Uploader can list `GET /me/submissions` with status + reject reason.

### 4.4 Pending Queue Cap (4GB)
- Definition: `SUM(File.size WHERE Submission.status=pending) <= 4GB`.
- Approved/published files do NOT count. Rejected files stop counting at decision time (see §4.6).
- Enforcement in `transaction.atomic()` with row-lock on singleton `QueueCounter`.
- If `pending_bytes + new_bytes > 4GB`: reject entire submission, save nothing, return `503 {code: QUEUE_FULL, message: "Server queue full, please try again after some time"}`.
- No partial saves.

### 4.5 Moderation
- Mode switch: `POST /me/mode {mode}`. 403 if `is_verifier=False`. Mode stored as JWT claim + server-side check.
- Queue: `GET /moderation/queue?folder_id=&subject_id=&search=` returns only `pending` where `folder_id IN (my scopes)`. Includes full directory path `faculty/semester/subject/folder_type`, uploader email/id, title, description, file URLs/metadata, submitted_at.
- Detail: `GET /moderation/submissions/:id` (scope-checked).
- Actions: `POST /moderation/submissions/:id/approve`, `POST /moderation/submissions/:id/reject {reason (required)}`.
- Both actions: set `reviewer_id, reviewedAt`, write `ModerationLog`, update `QueueCounter`, move/delete files (see §4.6). Atomic.
- Reject reason required, visible to uploader for 3 days.

### 4.6 Storage Lifecycle
- Two roots: `pending/` (queue) and `published/` (main storage).
- On approve: filesystem move `pending/* -> published/*`, update `File.storage_key`, `Submission.status=approved`. Queue bytes freed.
- On reject: `Submission.status=rejected, rejectedAt=now(), purgeAt=now()+3days`. Queue bytes freed immediately. Row + files remain readable to uploader/reviewer for 3 days, then hard-deleted by daily purge job.
- Purge job deletes `File` objects from disk + DB, then `Submission` rows where `status=rejected AND purgeAt <= now()`.

### 4.7 Verifier Scope (allowlist)
- Enforcement at Folder level via join table, logically a list per verifier.
- `ModeratorScope(user_id, folder_id)` unique pair.
- Helpers for admin to bulk-grant whole Subject/Faculty (bulk-insert Folder rows), but enforcement stays Folder-level.
- Queue + approve/reject both check scope. No scope = empty queue.

### 4.8 Admin-only Structure
- `POST/PATCH/DELETE /admin/faculties|semesters|subjects` restricted to `is_staff`.
- Creating a Subject auto-creates its 2 Folders.
- `POST /admin/verifiers/grant|revoke`, `POST /admin/scopes/grant|revoke`, `POST /admin/verifiers/reset-password`.

## 5. Non-functional
- Passwords: Argon2/bcrypt. Rate-limit login, verify, reset.
- JWT short access + longer refresh. Revoke on password change.
- Email: SMTP for MVP, async send (thread/Celery later).
- Storage abstraction S3-ready via django-storages. MVP local disk ok.
- Audit: `ModerationLog` immutable.

## 6. Out of scope (MVP)
- No teacher role, no comments/ratings, no full-text search, no versioning, no appeal flow beyond resubmit after purge.

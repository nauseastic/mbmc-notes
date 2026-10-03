# MBMCnotes — Data Model and API (Django)

Stack: `Django 5 + DRF + SimpleJWT + PostgreSQL + django-storages + SMTP`. Local disk `MEDIA_ROOT/pending|published` for MVP, S3 swap later.

## 1. Models

```python
User(id, email unique, password_hash, is_verified bool default False,
     is_verifier bool default False, faculty FK null, is_staff, created_at)

EmailVerificationToken(id, user FK, token unique, expires_at, used_at null)
PasswordResetToken(id, user FK, token unique, expires_at, used_at null)  # normal users only

Faculty(id, name unique, code unique, created_at)
Semester(id, faculty FK, number int, label e.g. "Sem 3", Unique(faculty, number))
Subject(id, semester FK, name, code, Unique(semester, code))
Folder(id, subject FK, type Char[notes|assignments], Unique(subject, type))

Submission(id, folder FK, uploader FK, title, description,
           status Char[pending|approved|rejected] default pending,
           reject_reason null, reviewer FK null, reviewed_at null,
           rejected_at null, purge_at null, created_at)
  Index(folder, status, created_at)

File(id, submission FK related_name files, storage_key, original_name,
     mime, size bigint, created_at)

ModeratorScope(id, user FK, folder FK, granted_by FK, granted_at)
  Unique(user, folder)
  Index(user, folder)

QueueCounter(id=1 singleton, pending_bytes bigint default 0, updated_at)
  # avoids SUM over File table on every submit under lock

ModerationLog(id, submission FK, reviewer FK, action[approve|reject],
              reason null, created_at)  # append-only
```

Why single `Submission` table with status instead of separate `Note/Assignment` tables:
- Notes vs assignments distinguished by `folder.type`, no duplicate schema.
- Approve = status flip + file move, no cross-table copy.
- Published query = `filter(folder=..., status=approved)`.

Why `ModeratorScope` rows (list) not single string:
- One row per `(user, folder)` = logically a list per verifier, e.g. `[Physics/notes, Physics/assignments]`.
- FK survives renames, string path `BSc-CS/Sem3/Physics/notes` breaks on rename.
- Enforceable: `Folder.id IN (SELECT folder_id FROM scope WHERE user=me)`.
- Revocable per-directory, auditable via `granted_by/at`.

## 2. Validation Matrix

| Check | Rule |
|---|---|
| Extension+MIME | `pdf: application/pdf`, `jpg/jpeg: image/jpeg`, `png: image/png`, `webp: image/webp`, `docx: application/vnd.openxmlformats-officedocument.wordprocessingml.document` |
| Magic bytes | Verify header (e.g. `%PDF`, ZIP for docx) |
| Per-file | image 20MB, docx 20MB, pdf 100MB |
| Per-submission | <=5 files, total <=100MB |
| Queue | `pending_bytes + new_total <= 4GB`, else 503 QUEUE_FULL |

## 3. API Surface (DRF)

Auth:
```
POST /api/auth/register
POST /api/auth/verify-email
POST /api/auth/resend-verification
POST /api/auth/login
POST /api/auth/refresh
POST /api/auth/forgot-password   # 403 if is_verifier
POST /api/auth/reset-password    # 403 if is_verifier
GET  /api/me
PATCH /api/me {faculty_id}
POST /api/me/mode {mode: normal|moderator}  # 403 if not verifier
```

Browse:
```
GET /api/faculties
GET /api/faculties/:id/semesters
GET /api/semesters/:id/subjects
GET /api/subjects/:id/folders
GET /api/folders/:id/items  # approved only
```

Submissions:
```
POST /api/folders/:id/submissions  # multipart title, description, files[]
GET  /api/me/submissions
GET  /api/me/submissions/:id
```

Moderation (requires `IsVerifier + mode==moderator + scope`):
```
GET  /api/moderation/queue?folder_id=&subject_id=&ordering=
GET  /api/moderation/submissions/:id
POST /api/moderation/submissions/:id/approve
POST /api/moderation/submissions/:id/reject {reason}
```

Admin (`IsAdminUser`):
```
CRUD /api/admin/faculties|semesters|subjects
POST /api/admin/verifiers/grant {user_id}
POST /api/admin/verifiers/revoke {user_id}
POST /api/admin/verifiers/reset-password {user_id, new_password}
POST /api/admin/scopes/grant {user_id, folder_ids[]}
POST /api/admin/scopes/grant-subject {user_id, subject_id}
POST /api/admin/scopes/revoke {user_id, folder_ids[]}
GET  /api/admin/queue-stats {pending_bytes, pending_count, limit}
```

## 4. Permissions
- `IsVerified`: all authed endpoints require `is_verified`.
- `IsModeratorMode`: checks JWT `mode==moderator` + `is_verifier`.
- `HasFolderScope`: object-level, `ModeratorScope.exists(user, submission.folder)`.
- File serving: signed URLs or X-Accel-Redirect, never public static. Check browse permission (verified) or scope (pending).

## 5. QueueCounter Concurrency
- `SELECT ... FOR UPDATE` on `QueueCounter(id=1)` inside `transaction.atomic()` for submit, approve, reject.
- Submit: `if counter.pending_bytes + new_bytes > 4GB: raise QUEUE_FULL` else `counter += new_bytes`.
- Decision: `counter -= submission_total` (clamp >=0). Freed at decision, not at purge.

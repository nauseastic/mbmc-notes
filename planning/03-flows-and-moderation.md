# MBMCnotes — Flows and Moderation

## 1. Register / Verify / Login
1. `POST /register {email, password, faculty_id?}` -> hash (Argon2), `is_verified=False`, create `EmailVerificationToken(24h)`, send link.
2. User clicks link -> `POST /verify-email {token}` -> validate expiry + unused -> `is_verified=True`.
3. `POST /login` -> if not verified: `403 EMAIL_NOT_VERIFIED`. Else JWT `access (mode=normal default) + refresh`.
4. Rate-limit: 5 login/min/IP, 3 resend/hour/user.

## 2. Password Reset
Normal user:
1. `POST /forgot-password {email}` -> always `200 OK` (no enumeration), create token if user exists and `is_verifier=False`, send email.
2. `POST /reset-password {token, new_password}` -> validate, set password, mark used, revoke JWTs.

Verifier:
1. Same endpoint returns `403 {code: CONTACT_ADMIN}` if `is_verifier=True`.
2. Verifier contacts admin offline -> admin `POST /admin/verifiers/reset-password` -> new password communicated offline.

## 3. Browse Flow
1. User `GET /faculties` -> select -> `PATCH /me {faculty_id}` (optional persist).
2. `GET /faculties/:id/semesters` -> select semester.
3. `GET /semesters/:id/subjects` -> select subject.
4. `GET /subjects/:id/folders` -> returns 2 rows: `notes`, `assignments`.
5. `GET /folders/:id/items` -> paginated approved submissions + file metadata + download URLs.

## 4. Submit Flow (current directory)
1. User in Folder view clicks Submit -> `POST /folders/:fid/submissions` with `title, description, files[]`.
2. Server: auth + verified -> validate Folder exists -> validate file count/MIME/magic/per-file/per-total (422 on fail with per-file errors).
3. Queue check (atomic):
   ```
   BEGIN; LOCK QueueCounter;
   new_bytes = sum(f.size);
   IF counter.pending_bytes + new_bytes > 4GB: ROLLBACK; 503 QUEUE_FULL;
   ELSE: create Submission(pending) + File(pending/...) rows; counter += new_bytes; COMMIT;
   ```
4. Response: `201 {submission_id, status: pending}` or `503 {code: QUEUE_FULL, message: "Server queue full, please try again after some time"}`. Frontend shows that message verbatim. Nothing saved on 503.
5. Uploader tracks via `GET /me/submissions`.

## 5. Moderation Flow (core)
Precondition: `is_verifier=True`, scope rows exist, mode=moderator.

1. Login (mode=normal) -> `POST /me/mode {moderator}` -> new access token with `mode=moderator`.
2. `GET /moderation/queue` -> SQL: `Submission(status=pending AND folder IN my_scopes) ORDER BY created_at`. Each item includes `directory_path` (joined Faculty/Semester/Subject/Folder), `uploader {id, email}`, `title, description, files [{name, mime, size, url}]`.
3. Open detail -> preview/download via scoped signed URL.
4. Approve:
   ```
   BEGIN; check scope; LOCK submission + counter;
   move files pending/ -> published/ on disk;
   update File.storage_key; Submission(status=approved, reviewer, reviewed_at);
   counter -= total; insert ModerationLog(approve); COMMIT;
   ```
   Appears immediately in `GET /folders/:id/items`.
5. Reject:
   ```
   BEGIN; check scope; Submission(status=rejected, reason required, rejected_at=now, purge_at=now+3d);
   counter -= total; insert ModerationLog(reject); COMMIT;
   ```
   Files stay 3 days for transparency, then purged. Uploader sees `rejected + reason` until purge.

Mode switch back: `POST /me/mode {normal}` -> moderator APIs 403 again.

## 6. Verifier Grant Flow
1. User asks admin offline.
2. Admin `POST /admin/verifiers/grant {user_id}` -> `is_verifier=True`.
3. Admin `POST /admin/scopes/grant {user_id, folder_ids[]}` or `grant-subject` helper.
4. Verifier can now switch modes. No self-signup for verifier.

## 7. Purge Job (rejected, 3 days)
- Daily cron: `python manage.py purge_rejected`.
- Query: `Submission(status=rejected, purgeAt <= now())`.
- For each: delete files from disk, delete `File` rows, delete `Submission` row, log.
- Idempotent, batched (e.g. 500/batch). Failures logged, retried next run.
- Approved rows never purged.

## 8. Error Codes
- `422 VALIDATION_ERROR` + per-file details (bad type, over per-file/total limit, too many files).
- `503 QUEUE_FULL` queue cap.
- `403 NOT_VERIFIED / NOT_VERIFIER / WRONG_MODE / NO_SCOPE / CONTACT_ADMIN`.
- `404` for out-of-scope detail (don't leak existence — return 404 instead of 403 on detail).

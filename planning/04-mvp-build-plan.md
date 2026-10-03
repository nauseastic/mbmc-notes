# MBMCnotes — MVP Build Plan (Backend Only)

## 1. Stack (final)
- `Django 5, DRF, SimpleJWT, PostgreSQL, django-storages, SMTP`
- Storage: `MEDIA_ROOT/pending|published` local for MVP, S3-ready interface.
- Email: Django SMTP backend, sync send MVP, move to background thread/Celery if slow.
- Purge: `management command + cron (daily)`. No Celery/Redis for MVP.
- Auth hashing: Argon2.

## 2. Project Layout
```
config/settings.py, urls.py
apps/accounts/ (User, tokens, auth views, IsVerified)
apps/catalog/ (Faculty, Semester, Subject, Folder, admin CRUD)
apps/submissions/ (Submission, File, QueueCounter, submit/browse APIs)
apps/moderation/ (ModeratorScope, queue, approve/reject, mode middleware)
apps/adminops/ (grant/revoke, reset verifier, queue-stats, purge command)
core/storage.py (pending/published paths, move helper, MIME+magic validation)
core/permissions.py
```

## 3. Milestones
1. **M1 Auth + Catalog (week 1)**: custom User, verify/reset tokens, SMTP, Faculty/Semester/Subject/Folder CRUD (admin-only), auto-create 2 Folders on Subject create, seed script for initial faculties.
2. **M2 Submit + Queue (week 1-2)**: multipart submit, validation matrix (20/100/20MB, 5 files/100MB total), QueueCounter + atomic QUEUE_FULL 503, `My Submissions`, approved listing.
3. **M3 Moderation (week 2)**: `is_verifier`, mode switch, ModeratorScope + bulk helpers, scoped queue/detail, atomic approve (move) / reject (purgeAt+3d), ModerationLog, verifier-reset block.
4. **M4 Hardening (week 3)**: rate limits, signed file URLs, purge command + cron, queue-stats, tests (auth, limits, queue race, scope leak, purge), Postgres + media backup.

## 4. Key Implementation Notes
- Singleton `QueueCounter(id=1)` created via migration. All submit/decide paths `select_for_update()` it.
- File move on approve must be filesystem-atomic per file + DB transaction; on failure rollback and leave in pending.
- Never trust extension: check `content_type + python-magic`.
- Detail endpoint returns 404 (not 403) for out-of-scope to avoid ID enumeration.
- JWT `mode` claim: `normal` default, `moderator` only if `is_verifier`. Middleware re-checks DB flag each request.

## 5. Test Checklist
- Register -> verify -> login -> browse hierarchy.
- Unverified blocked, verifier reset blocked with CONTACT_ADMIN.
- Upload pdf/image/docx pass, video/exe fail, over-limit fail, 6 files fail, >100MB total fail.
- Fill queue to 4GB (mock counter) -> next submit 503, nothing saved.
- Verifier sees only scoped folders, cannot approve out-of-scope (404).
- Approve moves files + frees bytes + visible in folder. Reject sets purgeAt, frees bytes, deleted after 3d job.
- Race: parallel submits don't exceed cap (counter lock test).

## 6. Risks
- 4GB lock contention under burst: acceptable for college scale; mitigate with short transaction (validate files before lock).
- Local disk fills: monitor `pending_bytes` via `/admin/queue-stats` + disk alerts. Published storage needs separate quota later.
- Email deliverability: use college SMTP or Brevo/SES, SPF/DKIM.

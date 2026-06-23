# Job Application Curator — Building Workshop Task List

Each task has a clear deliverable, acceptance criteria, and a verification command.
Built from the 8-slice backend build + the frontend prototype.

---

## Task 0 — Frontend prototype (complete)
**Deliverable:** Single-file React app (`JobCuratorApp.jsx`) covering the full user flow: landing → OTP auth → 5-step intake → dashboard with job matches, ATS resume tips, cover letter generation, and application tracker.
**Acceptance criteria:**
- All intake steps (preferences, skills interview, documents, links, review) navigate correctly
- Skills typeahead suggests from a lexicon as the user types
- Dashboard shows job cards with % match, apply links, and a resume/cover-letter panel
- Tracker persists applications through status stages (Saved → Applied → Interviewing → Offer → Rejected)
- App runs in a plain Chrome tab with no build step
**Verification:** `open index.html` in Chrome → walk landing → OTP demo → intake → dashboard → tracker without errors

---

## Task 1 — Project scaffold + database
**Deliverable:** FastAPI app with `/health`, Postgres schema from the data model, SQLAlchemy models, Alembic migrations, docker-compose for local Postgres, and a virtual environment.
**Acceptance criteria:**
- `/health` returns `{"status": "ok"}` with 200
- `alembic upgrade head` creates all 7 tables (users, otp_codes, preferences, skills, documents, external_links, applications) with correct columns, enums, indexes, and triggers
- `alembic check` reports no drift between models and migration
- `.env` is gitignored; `.env.example` has placeholders only
- Two smoke tests pass: health check + migration cycle
**Verification:**
```bash
alembic upgrade head && alembic downgrade base && alembic upgrade head
alembic check
pytest tests/test_smoke.py -v
curl http://localhost:8000/health
```

---

## Task 2 — Email OTP authentication + JWT
**Deliverable:** `POST /api/auth/request-otp`, `POST /api/auth/verify`, `GET /api/auth/me` with email OTP flow, JWT issuance, and an auth dependency for protected routes.
**Acceptance criteria:**
- OTP codes are stored hashed (HMAC-SHA256), expire in 10 min, and are single-use
- `request-otp` always returns 200 (no email-existence leak); rate-limited per email + IP
- `verify` uses timing-safe comparison; locks the code after 5 failed attempts
- JWT signed with HS256; app refuses to boot without `JWT_SECRET`
- Dev mode logs the OTP code (never emails it); production refuses to boot without SMTP
- `GET /api/auth/me` returns 401 with no/bad/expired token
- All 5 required cases pass: happy path, expired code, wrong code, attempt cap, no-token 401
**Verification:**
```bash
pytest tests/test_auth.py -v   # 11 tests
# Live: start uvicorn, POST /api/auth/request-otp, read code from server log,
# POST /api/auth/verify → get JWT, GET /api/auth/me with Bearer token → 200
```

---

## Task 3 — Profile CRUD (the keystone record)
**Deliverable:** `GET /api/profile` and `PUT /api/profile` — transactional upsert across preferences, skills, and external_links; explicit `completed` flag; draft vs. complete distinction.
**Acceptance criteria:**
- `GET` returns 404 for a new user; assembles all tables into the camelCase contract shape
- `PUT` writes preferences + skills (full-replace) + links in a single transaction — partial failure rolls back entirely
- `completed=true` requires LinkedIn (URL-shaped) + ≥1 target role; drafts (`completed=false`) have no mandatory fields
- `linkedin_url` is nullable so "Save & finish later" works before the Links step
- `profileComplete` from `/auth/verify` and `/auth/me` reads the `completed` flag
- User A cannot read or write User B's profile
**Verification:**
```bash
pytest tests/test_profile.py -v   # 9 tests including atomicity + user-scoping
# Live round-trip: PUT profile → GET profile → confirm identical (salary int↔str,
# sponsorship bool↔"yes"/"no", skills sorted)
```

---

## Task 4 — Document upload, list, delete
**Deliverable:** `POST /api/documents` (multipart), `GET /api/documents`, `DELETE /api/documents/{id}` with byte-sniffing validation and a local filesystem storage adapter.
**Acceptance criteria:**
- Upload validates format from actual bytes (PDF magic / DOCX zip+`word/document.xml`), not client Content-Type → 415 for anything else
- Files stored under a server-generated key (`{user_id}/{uuid}.{ext}`) — client filename never touches the storage key
- Uploads over 10 MB rejected with 413
- DELETE removes the DB row first (committed), then the object — a failed object delete leaves a harmless orphan, never a dangling row
- User B cannot see or delete User A's documents
- Storage adapter is behind a Protocol interface — S3 drops in later without endpoint changes
**Verification:**
```bash
pytest tests/test_documents.py -v   # 7 tests
# Live: upload real PDF → list → DELETE → confirm 0 files on disk
# Spoof test: send text bytes with Content-Type: application/pdf → confirm 415
```

---

## Task 5 — Live job matching (server-side AI + web search)
**Deliverable:** `GET /api/jobs` — server-side Gemini call with Google Search grounding returning scored job listings; per-user 15-minute cache; explicit refresh.
**Acceptance criteria:**
- `GEMINI_API_KEY` stays server-side; never appears in any response or log line
- Model call is behind a patchable seam — test suite never hits the live API
- JSON output parsed defensively (whole → fenced → first-`[`...last-`]`); each entry validated with Pydantic; malformed entries dropped
- Returns `score` (0–100) and `matched` (skill overlap) per listing
- Model error/timeout → 502/504 so the frontend's sample fallback triggers
- Missing API key → 502 (not 500)
- Results cached 15 min per user; `?refresh=1` forces a new search
- No profile → 400
**Verification:**
```bash
pytest tests/test_jobs.py -v   # 10 tests, seam patched, no live calls
# Live (requires GEMINI_API_KEY in .env):
# GET /api/jobs with Bearer token → real current listings with applyUrl fields
```

---

## Task 6 — ATS resume tips + cover letter (server-side AI)
**Deliverable:** `POST /api/optimize` — server-side Gemini call returning `{atsTips: string[], coverLetter: string}` tailored to a specific job + the user's profile.
**Acceptance criteria:**
- Same key-safety and seam patterns as Task 5
- Prompt built from profile (preferences + skills) + job fields; absent job fields omitted gracefully (including `job: {}`)
- Resume `parsed_text` included in the prompt if available; falls back to profile only (the seam where real resume parsing slots in)
- Output validated with Pydantic (`extra="ignore"` drops hallucinated keys); missing required field → 502
- No caching (user-triggered, per-job)
- All error paths return generic messages — upstream text never echoed
**Verification:**
```bash
pytest tests/test_optimize.py -v   # 12 tests, seam patched
# Live (requires GEMINI_API_KEY):
# POST /api/optimize with a job object → {atsTips: [...], coverLetter: "..."}
```

---

## Task 7 — Application tracker CRUD
**Deliverable:** `GET`, `POST`, `PATCH`, `DELETE` on `/api/applications` — the "Excel but better" tracker backing the dashboard's tracker tab.
**Acceptance criteria:**
- `POST` upserts on `(user_id, company, title)` — re-tracking the same role updates, never duplicates
- `appliedDate` defaults to today when `status=applied` and no date supplied
- `PATCH` only touches fields present in the payload (`exclude_unset`)
- `DELETE` returns 204; 404 if not the user's
- User B cannot see, patch, or delete User A's applications
- camelCase I/O per the contract (`applyUrl`, `match`, `appliedDate`)
**Verification:**
```bash
pytest tests/test_applications.py -v   # 8 tests
# Live: POST → upsert same title+company (list stays at 1) → PATCH notes →
# DELETE → empty list
```

---

## Task 8 — Frontend repoint + end-to-end proof
**Deliverable:** Vite React app under `frontend/` with the prototype repointed at the real backend; CORS configured; all browser-side AI calls removed.
**Acceptance criteria:**
- Direct browser Anthropic/Gemini calls are deleted — key stays server-side
- JWT stored in localStorage; any 401 clears token and returns to landing
- `profileComplete` from `/auth/verify` routes new users to intake, returning users to dashboard
- OTP: client-side code generation removed; dev hint points at server logs
- Documents upload on drop and delete on remove (real `POST`/`DELETE /api/documents`)
- Skills `string[]` ↔ API `[{skill, source}]`, sponsorship `"yes"/"no"` ↔ `bool|null`, salary `string` ↔ `int|null` all round-trip correctly
- Jobs from `/api/jobs` use server `score`/`matched` directly; `scoreJob` kept only for offline sample fallback (weights aligned)
- Tracker uses lowercase API enum values with display labels
- 502/504 on jobs → sample fallback with note; 502/504 on optimize → modal error
- Frontend builds (`npm run build`) with no errors
**Verification:**
```bash
pytest -v   # 62 tests, all green
npm --prefix frontend run build   # clean build
# Full live E2E (requires GEMINI_API_KEY):
# localhost:5173 → sign up → OTP from server log → intake → complete profile →
# dashboard shows real job listings → Resume + cover letter generates → Apply →
# confirm → tracker shows application → PATCH status → DELETE
```

---

## Summary

| Task | Endpoints / Deliverable | Tests | Commit |
|------|------------------------|-------|--------|
| 0 | Frontend prototype | manual | — |
| 1 | Scaffold + DB | 2 | ed0c40f |
| 2 | Auth (OTP + JWT) | 11 | 4a13f14 |
| 3 | Profile CRUD | 9 | be194bf |
| 4 | Documents | 7 | 8efe39a |
| 5 | Jobs (AI + search) | 10 | 98dcd3a |
| 6 | Optimize (AI) | 12 | 9ac6c32 |
| 7 | Tracker CRUD | 8 | 3b45393 |
| 8 | Frontend repoint | 3 CORS | 6684c14 |
| **Total** | **15 endpoints** | **62** | **8 commits** |

**One remaining step:** add `GEMINI_API_KEY` to `.env` and run the full live E2E walk in Task 8 to prove the real AI calls end-to-end.

## Execution Log

<!-- Plan deviations, implementation calls, and gate evidence go here as the plan runs. -->

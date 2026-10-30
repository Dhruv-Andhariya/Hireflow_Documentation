# HireFlow AI — Project Map (simple guide)

One file that explains **the whole project in simple words**: what it does, every folder and its role, and how the
folders work together in each workflow.

> Other docs go deeper on one topic: [SCREENING_ROUND.md](SCREENING_ROUND.md) (AI voice round),
> [M06_M09_PUBLISH_AND_APPLY.md](M06_M09_PUBLISH_AND_APPLY.md) (publish + apply), [TEAM_GUIDE.md](TEAM_GUIDE.md)
> (setup + git rules), [HireFlow_AI_Documentation.md](HireFlow_AI_Documentation.md) (full product spec, module
> numbers M01–M33).

---

## 1. What is HireFlow?

HireFlow is a **hiring platform that uses AI** to do the boring parts of hiring for a company's HR team.

In one line: **HR types in a job opening → AI writes the job description → it is posted online → students apply with
their resume → AI reads and scores every resume → good students get an email and talk to an AI interviewer → HR reads
the results.**

### Who uses it

| Person | What they do | Where |
|---|---|---|
| **HR Member** | Creates jobs, edits/approves JDs, publishes, looks at applicants | HR dashboard |
| **Company Admin** | Everything an HR Member does + company profile, team invites, LinkedIn connection | HR dashboard |
| **Platform Admin** | Looks after *all* companies (suspend / activate). Created only from the command line | HR dashboard `/admin` |
| **Student / candidate** | Sees jobs, applies, verifies phone, uploads resume, takes the AI voice round, can delete their data | Careers site |

Many companies can use the same HireFlow (it is **multi-tenant**). Each company only ever sees its own data.

---

## 2. The full hiring flow (the story)

```text
 HR side (HR dashboard)                                    Student side (careers site)
 ─────────────────────                                     ───────────────────────────
 1. Sign up / log in (email code or Google)
 2. Register company + fill company profile
 3. Create a job (job form + minimum ATS score)
 4. AI writes the JD  ──► HR edits ──► HR approves
 5. Publish ──► careers page + LinkedIn post  ───────────► 6. Student opens the "Apply here" link
                                                            7. Verifies phone (OTP), fills form, uploads resume
                                                               └─ "Thanks for applying" email
 8. AI reads + scores the resume in the background (ATS)
      score ≥ minimum → SHORTLISTED  ──► shortlist email with a personal link ──► 9. AI voice screening:
      score <  minimum → NOT SHORTLISTED ──► polite rejection email                  6 spoken questions
      could not read   → NEEDS HR REVIEW (never auto-rejected)                       answered by voice
10. HR opens "Shortlisted" page: score, reasons, transcript + AI notes of the voice round, review status
```

Everything up to step 10 is **built and working**. Test rounds (coding, aptitude, SQL…), proctoring, interview
scheduling, final evaluation and analytics are **planned but not started** (empty folders with a README).

---

## 3. The running pieces (architecture)

When you run the project, **4 programs** run on your laptop, plus some helper services:

```text
   ┌─────────────────────┐       ┌─────────────────────┐
   │  HR dashboard       │       │  Careers site       │      ◄── what people open in the browser
   │  Next.js  :3000     │       │  Next.js  :3300     │
   └─────────┬───────────┘       └─────────┬───────────┘
             │  /api/v1/...                │  /api/v1/public/... (proxied, students only)
             ▼                             ▼
         ┌───────────────────────────────────────┐
         │  API  (NestJS, TypeScript)  :4000     │      ◄── the "brain": all rules, all data
         └──┬──────────────┬─────────────┬───────┘
            │              │             │ HTTP
            ▼              ▼             ▼
      PostgreSQL       S3 bucket    ┌─────────────────────────────┐
      (Neon cloud      (resume      │  AI worker (Python FastAPI) │ ──► Google Gemini
       or Docker)       files)      │  :8000                      │     (text, audio, voice)
                                    └─────────────────────────────┘
      Email: Gmail SMTP or Mailpit (fake inbox :8025)   LinkedIn API (company page posts)
```

**Golden rules of this design**

1. Browsers only talk to the **API**. They never talk to the database, the AI worker or Gemini directly.
2. The **API** owns all data and all business rules. It is the only program that writes to the database.
3. The **AI worker** only does AI work (write JD, read resume, transcribe voice, speak). It stores nothing.
4. The **shape of data** sent between the parts is defined once in `packages/contracts` (TypeScript, Zod) and copied
   in `workers/ai-worker/ai_worker/schemas.py` (Python, Pydantic).

---

## 4. Top-level folder map

```text
hireflow/
├── apps/                  ◄ programs people open or call
│   ├── api/               ✅ NestJS backend (the brain)                     :4000
│   ├── hr-dashboard/      ✅ Next.js website for HR                         :3000
│   ├── careers-site/      ✅ Next.js public job pages, apply form, AI round :3300
│   ├── candidate-app/     ⬜ empty — future candidate portal for test rounds
│   └── admin-console/     ⬜ empty — future admin app
├── workers/               ◄ background programs
│   ├── ai-worker/         ✅ Python FastAPI — every AI task                  :8000
│   ├── workflows/         ⬜ future orchestrator (moves a candidate step to step)
│   └── activities-ts/     ⬜ future side-effect tasks for the orchestrator
├── packages/              ◄ shared code, imported by apps (never run alone)
│   ├── contracts/         ✅ data shapes (Zod schemas) shared by everyone
│   ├── ui/ config/ email-templates/ round-sdk/ proctor-sdk/ rounds/*   ⬜ placeholders
├── prompts/               ✅ the AI prompts, as versioned YAML files
├── infra/                 ◄ Docker (local services) + future cloud deployment
├── scripts/               ◄ helper scripts (daily sync, public tunnel, demo)
├── seed/                  ◄ demo data / sample resumes
├── docs/                  ◄ documentation (this file)
├── .github/               ◄ CI, code owners, PR template
├── .vscode/               ◄ editor settings and tasks
└── root files             ◄ package.json, turbo.json, pnpm-workspace.yaml, .env.example …
```

✅ = real working code · 🟡 = partial · ⬜ = only a README describing the plan

---

## 5. Root files — what each one does

| File | Role |
|---|---|
| `package.json` | Root commands: `pnpm dev` (start all JS apps), `pnpm check` (build + lint + typecheck + test), `pnpm sync`, `pnpm infra:up` / `infra:down`, `pnpm format` |
| `pnpm-workspace.yaml` | Tells pnpm which folders are JavaScript packages (`apps/*`, `packages/*`, …) |
| `pnpm-lock.yaml` | Exact versions of every JS library. Never edit by hand |
| `turbo.json` | Turborepo: runs `build / dev / lint / typecheck / test` in all packages, in the right order |
| `.env.example` | List of every setting (database URL, Gemini key, email, LinkedIn, feature flags…). Copy to `.env` and fill in. **`.env` is never committed** |
| `.editorconfig`, `.prettierrc` | Same formatting for everybody |
| `.gitignore`, `.gitattributes` | What git ignores, line endings |
| `README.md` | Short intro + quick start + timeline |
| `AGENTS.md` | Auto-written note for AI coding assistants (written by Turborepo) |

**Tooling in one sentence:** the JavaScript/TypeScript side is a **monorepo** managed by **pnpm** (installs) and
**Turborepo** (runs tasks); the Python AI worker is managed separately by **uv**.

---

## 6. `apps/api/` — the backend (the brain)

**Tech:** NestJS (TypeScript) + Prisma (database library) + PostgreSQL. All routes start with `/api/v1`.

### 6.1 Inside the folder

| Path | Role |
|---|---|
| `src/main.ts` | Starts the server on port 4000, sets the `/api/v1` prefix, CORS, proxy trust |
| `src/app.module.ts` | The list of all modules. Each new module is added here with one line |
| `src/health.controller.ts` | `GET /api/v1/health` — "is the API alive?" |
| `src/common/` | Shared helpers used by all modules (see 6.2) |
| `src/modules/` | **One folder per business area** (see 6.3) |
| `src/cli/` | Command-line tools: create the first Platform Admin, make someone a Company Admin, move old resumes to S3 |
| `prisma/schema/` | The database design, **one `.prisma` file per area** (see section 10) |
| `prisma/schema/migrations/` | SQL files that create/change tables, in date order. Applied with `pnpm --filter api db:deploy` |
| `prisma.config.ts` | Tells Prisma to read `DATABASE_URL` from the root `.env` |
| `tests/` | Vitest tests, one sub-folder per module. `in-memory.ts` / `fakes.ts` let tests run without a real database or AI |

### 6.2 `src/common/` — shared helpers

| File | Role |
|---|---|
| `prisma.service.ts` | The one database connection everybody uses |
| `env.ts` | Loads the root `.env` file |
| `api-error.ts` | Same error format everywhere + request body validation with the Zod contracts |
| `feature-flags.ts` | Reads `FEATURE_*=on/off` so a whole module can be switched off without code changes |
| `secret-box.ts` | AES-256-GCM encryption (used for LinkedIn tokens) |
| `uuidv7.ts` | Creates time-ordered unique ids |

### 6.3 `src/modules/` — one folder per business area

**How every module is built (same pattern everywhere):**

```text
xxx.controller.ts   ◄ the URL endpoints: reads the request, checks login/permission, calls the service
xxx.service.ts      ◄ the business rules ("is the job approved?", "is the score above the minimum?")
xxx.store.ts /      ◄ the only place that talks to the database (always filtered by the company / tenant_id)
xxx.repository.ts
xxx.module.ts       ◄ wires the pieces together for NestJS
index.ts            ◄ the module's "front door": other modules may import ONLY from here
README.md           ◄ what the module does, its endpoints, what is not done yet
```

| Module | Status | Role in simple words | Key files |
|---|---|---|---|
| `auth/` | ✅ | **Who are you, and what may you do?** Sign-up (with email code), login (+ 2FA code by email), Google sign-in, forgot/reset password, sessions in an httpOnly cookie, roles + permissions, team invitations | `auth.controller.ts`, `auth.service.ts`, `auth.guard.ts` (checks every request), `permissions.ts`, `access.ts`, `google-oauth.ts`, `team.*`, `sessions.service.ts` |
| `tenants/` | ✅ | **Companies.** Register a company, company profile (about, website…), Platform Admin can suspend/activate. `tenant-context.ts` gives every request "which company is this user from" | `companies.*`, `company-profile.*`, `tenant-context.ts` |
| `jobs/` | ✅ | **Job + AI job description.** Save the job form, ask the AI worker to write a JD, keep every version, HR edits, approval | `jobs.controller.ts`, `jobs.service.ts`, `jobs.repository.ts`, `ai-worker.client.ts` |
| `publisher/` | ✅ | **Post the approved job online.** Careers page + LinkedIn company page, preview, retry, close/reopen/delete, "Connect LinkedIn" (OAuth), update apply links when the public URL changes | `publisher.service.ts`, `post-formatter.ts`, `channels/careers-site.adapter.ts`, `channels/linkedin.adapter.ts`, `linkedin-connect.*`, `jd-checks.ts` |
| `applications/` | ✅ | **Students applying.** Public job data for the careers site, phone OTP, resume upload with checks (real PDF/DOCX, 5 MB, virus scan), duplicate check, "thanks" email, HR applicant list + resume download, student "view/delete my data" | `public-careers.controller.ts`, `applications.service.ts`, `otp/`, `resume/`, `email/`, `abuse/`, `hr-applications.*`, `my-data.service.ts` |
| `ats/` | ✅ | **AI resume screening + decision emails.** Minimum score per job, background loop that sends each resume to the AI worker, stores score and reasons, shortlisted / not shortlisted, HR override, automatic shortlist + rejection emails, creates the personal screening link | `ats-processor.service.ts`, `ats-worker.client.ts`, `screening-settings.*`, `decision.*`, `auto-rejection.ts`, `decision-email.sender.ts`, `emails/decision-emails.ts`, `candidates.*` |
| `screening/` | ✅ | **AI voice interview (round 1).** Checks the personal link, consent, sends each spoken answer to the AI worker, saves the transcript, plays the next question, creates notes for HR at the end | `flow.ts` (the 6 fixed steps + limits), `screening.service.ts` (student side), `hr-screening.service.ts` (HR side), `screening-worker.client.ts`, `speech-cache.ts` |
| `audit/` | 🟡 | **Diary of important actions** (login, JD approved, published, resume downloaded, screening done…). Write-only, no screen yet | `audit.service.ts` |
| `rounds-config/` | ⬜ | Future: which test rounds a job has, order, cut-offs | README only |
| `round-engine/` | ⬜ | Future: runs the test rounds using `packages/round-sdk` plugins | README only |
| `proctoring/` | ⬜ | Future: cheating detection during tests | README only |
| `scheduler/` | ⬜ | Future: book interview slots (calendar) | README only |
| `evaluator/` | ⬜ | Future: score round attempts with reasons | README only |
| `feedback/` | ⬜ | Future: report cards / feedback for candidates | README only |
| `notifications/` | ⬜ | Future: one central email/WhatsApp sender (today each module sends its own emails) | README only |
| `analytics/` | ⬜ | Future: hiring funnel numbers | README only |

**Security built into every module:** the global `AuthGuard` blocks every route that does not say who may use it;
every database query is filtered by the logged-in user's company; another company's id simply answers "404 not
found"; public (student) routes are rate-limited per IP.

---

## 7. `apps/hr-dashboard/` — the HR website

**Tech:** Next.js 15 (React) App Router. Port 3000. Calls the API at `NEXT_PUBLIC_API_URL`.

| Path | Role |
|---|---|
| `src/app/` | **One folder = one page (URL)** — see table below |
| `src/components/` | Reusable UI pieces: forms, panels, tables, buttons (`ui.tsx`), `session.tsx` (who is logged in) |
| `src/lib/` | Logic with no UI: `api.ts` (calls the API), `ats-api.ts`, `publisher-api.ts`, `screening-api.ts`, form helpers (`job-form.ts`, `company-form.ts`, `jd.ts`), `roles.ts`, `safe-next.ts` (safe redirect after login) |
| `tests/` | Vitest tests for `src/lib/` |

| Page (URL) | What HR does there |
|---|---|
| `/signup`, `/login`, `/forgot-password`, `/reset-password/[token]` | Account and login |
| `/invite/[token]` | Accept a team invitation |
| `/register-company` | Create a new company (you become its Company Admin) |
| `/company` | Edit the company profile (used by the AI when writing JDs) |
| `/team` | Invite teammates, change roles, disable members |
| `/jobs`, `/jobs/new` | List jobs, create a job (form + minimum ATS score) |
| `/jobs/[id]` | Generate the JD with AI, edit it, see versions, validate and approve |
| `/publish` | Preview and publish approved jobs, connect LinkedIn, retry / close / reopen / delete posts |
| `/applications` | All applicants per job, download resumes |
| `/shortlisted` | ATS results: Shortlisted / Not shortlisted / Needs HR review / Being screened; override decisions, send emails, read the AI voice-round transcript (`ai-screening-panel.tsx`) |
| `/admin`, `/admin/companies/[id]` | Platform Admin: all companies |

**How a page works:** `page.tsx` (the route) → shows a client component (e.g. `jobs-list.tsx`) → it calls a function in
`src/lib/*-api.ts` → that calls the API with the login cookie and a CSRF header.

---

## 8. `apps/careers-site/` — the public website for students

**Tech:** Next.js 15. Port 3300. No login. This is what the "Apply here" link opens.

| Path | Role |
|---|---|
| `src/app/page.tsx` | Home page |
| `src/app/[company]/page.tsx` | All live jobs of one company |
| `src/app/[company]/jobs/[job]/` | Full job description + `apply-form.tsx` |
| `src/app/[company]/my-data/` | Student sees or deletes their own data (proved by a fresh phone OTP) |
| `src/app/screening/[token]/` | **The AI voice interview room** (`screening-room.tsx`) |
| `src/components/phone-verify.tsx` | The phone OTP box |
| `src/lib/api.ts`, `screening-api.ts` | Calls to the API |
| `src/lib/recorder.ts` | Records the microphone (and refuses silent recordings) |
| `src/lib/apply-form.ts`, `phone-proof.ts`, `client-ip.ts`, `screening-ui.ts` | Form checks, OTP proof, real-IP helper, screening screen states |
| `src/middleware.ts` | Sets the student's real IP header safely (so rate limits cannot be faked) |
| `next.config.ts` | **Proxy:** the browser calls `/api/v1/public/*` on the careers site, which forwards it to the API. Only public routes are forwarded — HR routes are never reachable from here. Also adds privacy headers on the screening page |
| `src/content.ts` | Fixed texts of the site |
| `tests/` | Vitest tests |

---

## 9. `workers/` — background programs

### 9.1 `workers/ai-worker/` — all AI work (Python)

**Tech:** Python 3.12, FastAPI, uv. Port 8000. Swagger page: `http://localhost:8000/docs`. Only the API calls it.

| Endpoint | Used by | What it does |
|---|---|---|
| `POST /v1/jd/generate` | `jobs` module | Job form + company profile → full job description (JSON) |
| `POST /v1/jd/validate` | `publisher` module | Checks the JD has title, summary, responsibilities, requirements (rule checks) |
| `POST /v1/resume/screen` | `ats` module | Resume file + JD → parsed resume + ATS score with reasons. Scanned PDFs are read as images (OCR fallback) |
| `POST /v1/screening/turn` | `screening` module | One spoken answer → transcript + "ask a follow-up or move on?" (+ answer to the student's question) |
| `POST /v1/screening/speak` | `screening` module | Text of a question → spoken audio (WAV) |
| `POST /v1/screening/extract` | `screening` module | Full transcript → structured notes for HR (no score, no decision) |
| `GET /health` | anyone | "Is the worker alive?" |

| Folder / file | Role |
|---|---|
| `ai_worker/main.py` | All the endpoints above |
| `ai_worker/config.py` | Reads settings (model names, API key, time budgets) from the root `.env` |
| `ai_worker/schemas.py` | Data shapes — **must match `packages/contracts`** |
| `ai_worker/llm_gateway/` | "One door to the AI": `provider.py` (common interface), `gemini.py` (Gemini, with fallback models and retries), `prompts.py` (loads `prompts/*.yaml`), `errors.py` |
| `ai_worker/jd/` | JD writer (`generator.py`) |
| `ai_worker/parsing/` | Gets text out of a resume (`extract.py` for PDF/DOCX, `ocr.py` for scanned PDFs) |
| `ai_worker/scoring/` | ATS scoring of a resume against the JD (`ats.py`) |
| `ai_worker/screening/` | Voice interview logic: transcribe, decide follow-up, extract notes (`interview.py`, `models.py`) |
| `ai_worker/speech/` | Text-to-speech (`gemini_tts.py`) behind a `SpeechProvider` interface |
| `ai_worker/evaluation/`, `feedback/` | ⬜ empty, for future rounds |
| `tests/` | Pytest. `fakes.py` = fake AI, so **tests never call the real Gemini** |
| `pyproject.toml`, `uv.lock` | Python libraries |

### 9.2 `workers/workflows/` and `workers/activities-ts/` — ⬜ not started

The future "conductor" (Temporal) that moves a job and a candidate from step to step automatically. Today that glue
lives inside the API modules (for example the `ats` background loop).

---

## 10. The database (PostgreSQL)

Defined in `apps/api/prisma/schema/`, one file per area. Every company-owned table has a `tenant_id`.

| File | Tables | What they keep |
|---|---|---|
| `base.prisma` | — | Prisma setup (database connection, generator) |
| `tenants.prisma` | `tenants`, `company_profiles` | Companies and their profile |
| `auth.prisma` | `users`, `memberships`, `sessions`, `invitations`, `password_resets`, `login_challenges`, `signup_challenges` | People, their role in each company, logins, email codes |
| `jobs.prisma` | `jobs`, `jd_versions`, `job_approvals` | Job form, every JD version, who approved which version |
| `publisher.prisma` | `publisher_job_listings`, `job_posts`, `publisher_channel_settings`, `channel_credentials` | What is published where, per-channel status, encrypted LinkedIn tokens |
| `applications.prisma` | `candidates`, `applications`, `resumes`, `resume_files`, `consents`, `applications_phone_otp_challenges` | Students, their applications, resume records (the file itself is in S3), consents, OTPs |
| `ats.prisma` | `job_screening_settings`, `ats_evaluations`, `candidate_emails`, `screening_invites` | Minimum score, score + reasons + decision, decision emails, personal screening links (only a hash is stored) |
| `screening.prisma` | `screening_sessions`, `screening_turns`, `screening_results` | AI voice round: session, every question/answer transcript, final notes |
| `audit.prisma` | `audit_logs` | Diary of important actions |

**Changing the database:** edit the `.prisma` file → `pnpm --filter api db:migrate` (creates a new folder in
`migrations/`) → commit it → teammates run `pnpm --filter api db:deploy` (or `pnpm sync`).

The team database is on **Neon** (cloud PostgreSQL). Local Docker PostgreSQL also works for offline work.

---

## 11. `packages/` — shared code

| Package | Status | Role |
|---|---|---|
| `contracts/` | ✅ | **The most important shared package.** Zod schemas for every data shape: `schemas/auth.ts`, `company.ts`, `job.ts`, `publisher.ts`, `application.ts`, `resume.ts`, `ats.ts`, `screening.ts`, and `events/` (event envelope for the future event bus). The API checks requests against them; the dashboards use the same types. Change = small PR, merged the same day |
| `ui/` | ⬜ | Future shared React components |
| `config/` | ⬜ | Future shared ESLint / TS / Tailwind config |
| `email-templates/` | ⬜ | Future React Email templates (today templates live in each module's `emails/` file) |
| `round-sdk/` | ⬜ | Future `RoundPlugin` interface every test round implements |
| `rounds/*` | ⬜ | Future rounds: `aptitude`, `coding`, `sql`, `website`, `gaming`, `communication`, `group-discussion` |
| `proctor-sdk/` | ⬜ | Future browser-side cheating detection |

---

## 12. Supporting folders

| Folder | Role |
|---|---|
| `prompts/` | Every AI instruction as a versioned YAML file. Never write prompts inside code. Change the wording → new version file. Current: `jd_generate.v3`, `resume_screen.v1`, `resume_transcribe.v1` (scanned PDFs), `screening_turn.v1`, `screening_transcribe.v1`, `screening_extract.v1` |
| `infra/docker/` | `docker-compose.yml`: PostgreSQL (pgvector), Redis, **Mailpit** (fake inbox at :8025). Opt-in: MinIO (`--profile storage`), ClamAV virus scanner (`--profile security`), Temporal (`--profile workflow`) |
| `infra/k8s/`, `infra/terraform/` | ⬜ future production deployment |
| `scripts/sync.sh` | `pnpm sync`: git pull + install JS and Python libraries + apply DB migrations. Run every day |
| `scripts/start-public.ps1` | Starts Docker + a **Cloudflare tunnel** to the careers site, writes the public URL into `CAREERS_SITE_URL`, restarts the apps. Needed so LinkedIn posts and emails have a link that works outside your laptop |
| `scripts/register-autostart.ps1` | Runs `start-public.ps1` at every Windows login |
| `scripts/demo-m06-m09/` | A demo script for publish + apply, and SQL for a read-only resume-reader login |
| `seed/` | Demo data and sample resumes (`seed/resumes/`) |
| `docs/` | Documentation |
| `.github/` | `workflows/ci.yml` (on every PR: `pnpm check` + Python `ruff` + `pytest`), `CODEOWNERS` (who reviews which folder), PR template |
| `.vscode/` | Recommended extensions, editor settings, tasks |

---

## 13. Workflows — how the folders work together

Each workflow below shows **which folder does which step**.

### A. HR signs up and sets up the company

```text
hr-dashboard /signup ──► api auth: POST signup ──► email code sent (Gmail / Mailpit) ──► code entered ──► user created
hr-dashboard /login  ──► api auth: password ok ──► (2FA) code by email ──► session cookie set
hr-dashboard /register-company ──► api tenants: tenant + Company Admin membership created
hr-dashboard /company ──► api tenants: company profile saved
hr-dashboard /team ──► api auth team: invitation email ──► teammate opens /invite/[token] ──► joins as HR Member
```

### B. Job → AI job description → approval

```text
hr-dashboard /jobs/new        HR fills the job form + minimum ATS score
   └─► api jobs               saves the job (min score is saved privately in the ats module)
hr-dashboard /jobs/[id]       HR clicks "Generate"
   └─► api jobs               loads job form + company profile
       └─► ai-worker /v1/jd/generate ──► prompts/jd_generate.v3.yaml ──► Gemini ──► JD (JSON, checked + cleaned)
   ◄── api jobs               saves it as a new row in jd_versions
HR edits (new version each time) ──► validates ──► approves ──► job_approvals row, job locked
```

### C. Publish

```text
hr-dashboard /publish         HR picks an approved job ──► Preview ──► Publish
   └─► api publisher          checks: approved? minimum ATS score set? JD valid (ai-worker /v1/jd/validate)?
                              post-formatter builds the post (exactly 2 links: Apply here + company website)
                              all channels in parallel:
                                ├─ careers-site adapter ──► job is now LIVE on careers site /<company>/jobs/<job>
                                └─ linkedin adapter     ──► LinkedIn company page post (company's own encrypted token)
                              audit log + per-channel status (retry if one failed)
```

### D. Student applies

```text
careers-site /<company>/jobs/<job>     student reads the JD
   phone-verify.tsx ──► api applications /public/otp/send + /verify ──► signed phone proof
   apply-form.tsx   ──► api applications POST /public/applications (form + resume)
                         checks: job LIVE? phone proven? real PDF/DOCX ≤ 5 MB? virus scan? already applied?
                         resume file ──► S3 bucket    record ──► resumes / applications tables
                         "thanks for applying" email ──► background email queue
HR: hr-dashboard /applications ──► list + resume download (audited)
```

### E. AI resume screening (ATS) + decision emails — runs by itself in the background

```text
api ats  ats-processor loop (every ATS_POLL_MS)
   finds new applications ──► creates an ats_evaluations row ──► claims it (never twice)
   └─► ai-worker /v1/resume/screen   resume + the JD version the student applied to
         parsing/ (text, or OCR for scanned PDFs) ──► scoring/ats.py ──► Gemini
   ◄── parsed resume + score + reasons
   compare with job's minimum score:
      score ≥ min ──► SHORTLISTED      ──► auto shortlist email + screening_invites (personal link, 7 days)
      score < min ──► NOT_SHORTLISTED  ──► auto rejection email
      error       ──► retry 1, 5, 15, 60 min ──► after 5 tries: NEEDS_REVIEW (HR decides, never auto-rejected)
HR: hr-dashboard /shortlisted ──► sees everyone, can override the decision (until an email was sent)
```

The automatic shortlist email is only sent when the screening link is **public** (Cloudflare tunnel or real
deployment). With a `localhost` link, HR presses "send" by hand.

### F. AI voice screening (round 1)

```text
student clicks the link in the shortlist email ──► careers-site /screening/<token>
   └─► api screening: token hash ──► invite ──► application (checks: valid, not expired)
   consent + microphone
   loop (6 steps: Introduction, Education, Background, Location, Availability, Candidate questions):
      api GET  /public/screening/speech ──► ai-worker /v1/screening/speak ──► Gemini TTS ──► question plays
      recorder.ts records the answer ──► api POST /public/screening/answer
         └─► ai-worker /v1/screening/turn: 1) transcribe audio only  2) decide follow-up from text
      api saves the turn (transcript) ──► next question (limits enforced by api screening/flow.ts)
   COMPLETED ──► background: ai-worker /v1/screening/extract ──► notes saved in screening_results
HR: hr-dashboard /shortlisted ──► candidate ──► "AI screening": transcript + notes + review status
```

Raw audio is **never stored**, only the text. The round gives information to HR — **no score, no decision**.

### G. Student deletes their data (DPDP)

```text
careers-site /<company>/my-data ──► fresh phone OTP ──► api applications my-data
   view: shows everything stored about the student
   delete: removes candidate, applications, resumes (also from S3), consents, screening rows
```

### H. A developer's day

```text
pnpm sync            ◄ pull + install + migrate
pnpm dev             ◄ API + HR dashboard + careers site
uv run python -m ai_worker   (inside workers/ai-worker)
... code ... tests ...
pnpm check           ◄ before every PR (+ uv run pytest if Python changed)
PR ──► CI runs the same checks ──► CODEOWNERS review ──► merge
```

---

## 14. How one request travels through the code

Example: HR clicks **"Generate JD"**.

```text
1. apps/hr-dashboard/src/app/jobs/[id]/job-review.tsx      button click
2. apps/hr-dashboard/src/lib/api.ts                        POST /api/v1/jobs/:id/jd/generate (cookie + CSRF header)
3. apps/api/src/modules/auth/auth.guard.ts                 logged in? has permission jobs.write?
4. apps/api/src/modules/jobs/jobs.controller.ts            reads the request
5. apps/api/src/modules/jobs/jobs.service.ts               rules: job exists in my company, not locked
6. apps/api/src/modules/jobs/ai-worker.client.ts           POST http://localhost:8000/v1/jd/generate
7. workers/ai-worker/ai_worker/main.py → jd/generator.py   builds the prompt from prompts/jd_generate.v3.yaml
8. workers/ai-worker/ai_worker/llm_gateway/gemini.py       calls Gemini, returns JSON
9. apps/api/src/modules/jobs/jobs.repository.ts            saves a new jd_versions row
10. apps/api/src/modules/audit/audit.service.ts            writes "jd.generated" to audit_logs
11. back to the dashboard ──► jd-panel.tsx shows the new JD
```

Almost every feature follows this pattern: **page → lib/api → controller → service → store/repository → database,
and when AI is needed: service → worker client → AI worker → Gemini.**

---

## 15. Run it locally

```powershell
cp .env.example .env                    # fill DATABASE_URL, GEMINI_API_KEY, … (ask the lead)
npx pnpm@9.15.0 install
docker compose -f infra/docker/docker-compose.yml up -d     # Redis, Mailpit (+ local Postgres)
npx pnpm@9.15.0 --filter api db:deploy                      # create / update tables
npx pnpm@9.15.0 dev                                         # API :4000, HR :3000, careers :3300
cd workers/ai-worker; uv sync; uv run python -m ai_worker   # AI worker :8000
```

| Open | URL |
|---|---|
| HR dashboard | http://localhost:3000 |
| Careers site | http://localhost:3300 |
| API health | http://localhost:4000/api/v1/health |
| AI worker docs | http://localhost:8000/docs |
| Fake email inbox | http://localhost:8025 |

Public link for LinkedIn / emails / phones: `powershell -ExecutionPolicy Bypass -File scripts\start-public.ps1`.

**Tests:** `pnpm check` (all TypeScript) · `pnpm --filter api test` (API only) · `cd workers/ai-worker; uv run pytest`.

---

## 16. "Where do I change…?" cheat sheet

| I want to change… | Go to |
|---|---|
| A page in the HR dashboard | `apps/hr-dashboard/src/app/<page>/` |
| A page students see | `apps/careers-site/src/app/` |
| A business rule / an endpoint | `apps/api/src/modules/<area>/*.service.ts` / `*.controller.ts` |
| A database table | `apps/api/prisma/schema/<area>.prisma` + new migration |
| The shape of data sent between parts | `packages/contracts/src/schemas/` **and** `workers/ai-worker/ai_worker/schemas.py` |
| What the AI is told to do | `prompts/*.yaml` (new version file) |
| Which AI model is used | `.env` (`LLM_MODEL`, `LLM_FALLBACK_MODELS`, `SCREENING_TTS_MODEL`) — never in code |
| How the AI is called | `workers/ai-worker/ai_worker/llm_gateway/` |
| Email wording | `apps/api/src/modules/ats/emails/decision-emails.ts`, `applications/email/confirmation-email.ts`, `auth/*-email.ts` |
| The voice interview questions / limits | `apps/api/src/modules/screening/flow.ts` + `SCREENING_*` in `.env` |
| A new posting channel (e.g. Naukri) | `apps/api/src/modules/publisher/channels/` — new adapter + one line in `channels/index.ts` |
| Switch a feature off | `.env`: `FEATURE_PUBLISHER`, `FEATURE_APPLICATIONS`, `FEATURE_SCREENING`, `FEATURE_ATS`, `AUTO_*_EMAILS` |
| Local services (DB, mail…) | `infra/docker/docker-compose.yml` |

---

## 17. What is built and what is next

| Area | Status |
|---|---|
| Login, 2FA, Google sign-in, teams, roles | ✅ |
| Multi-company (tenants), company profile | ✅ |
| Job form → AI JD → versions → approval | ✅ (validator = rule checks only, no AI bias check yet) |
| Publish to careers page + LinkedIn | ✅ |
| Apply link, phone OTP, resume upload to S3, thanks email, DPDP view/delete | ✅ (OTP shown on screen in dev — no SMS provider yet) |
| AI resume parsing + ATS score + shortlist / rejection emails | ✅ |
| AI voice screening round + HR transcript view | ✅ |
| Audit log | 🟡 write only, no screen |
| Test rounds, proctoring, scheduling, evaluation, feedback, notifications hub, analytics | ⬜ |
| Candidate app, admin console, workflow orchestrator, event bus | ⬜ |
| Permanent deployment (Vercel / Render + Neon + S3 + real email) | ⬜ — today the public link is a temporary Cloudflare tunnel |

---

## 18. Small glossary

| Word | Meaning |
|---|---|
| **JD** | Job description |
| **ATS** | Applicant Tracking System — here: the AI score that says how well a resume matches the JD |
| **Tenant** | One company using HireFlow |
| **Monorepo** | Many apps and libraries in one git repository |
| **Module** | One business area inside the API (`apps/api/src/modules/<name>`) |
| **Contract** | The agreed shape of a piece of data (`packages/contracts`) |
| **Prompt** | The written instruction given to the AI (`prompts/*.yaml`) |
| **Migration** | A SQL file that changes the database tables |
| **OTP** | One-time password (the 6-digit code sent to a phone or email) |
| **Feature flag** | An on/off switch in `.env` for a whole feature |
| **Tunnel** | A temporary public URL that points to your laptop (Cloudflare) |
| **DPDP** | India's data protection law — students can see and delete their data |
| **M01…M33** | Module numbers from the full spec ([HireFlow_AI_Documentation.md](HireFlow_AI_Documentation.md)) |

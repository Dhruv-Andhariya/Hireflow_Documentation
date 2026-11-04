# HireFlow Project Guide

This guide explains how HireFlow is built, folder by folder, in simple words. It covers:

- what each folder and file does
- which libraries we use and why
- how the parts connect to each other
- how the main features were built, step by step (for example the camera and tab-switch protection)

Read sections 1 to 4 first to see the big picture. After that, jump to the part you are working on.

> Short form used in this guide: **M05, M12, M13...** are the module numbers from the product spec
> (`docs/HireFlow_AI_Documentation.md`). Every API route starts with `/api/v1`.

---

## Contents

1. [What HireFlow does](#1-what-hireflow-does)
2. [The big picture: the 7 running parts](#2-the-big-picture-the-7-running-parts)
3. [Libraries we use (tech stack)](#3-libraries-we-use-tech-stack)
4. [Folder map](#4-folder-map)
5. [How the parts talk to each other](#5-how-the-parts-talk-to-each-other)
6. [The hiring journey, step by step, with the files involved](#6-the-hiring-journey-step-by-step-with-the-files-involved)
7. [Backend: `apps/api`](#7-backend-appsapi)
8. [Database: Prisma and PostgreSQL](#8-database-prisma-and-postgresql)
9. [Frontends: the three Next.js apps](#9-frontends-the-three-nextjs-apps)
10. [Python workers (AI)](#10-python-workers-ai)
11. [Shared code: `packages/` and `prompts/`](#11-shared-code-packages-and-prompts)
12. [How the main features were built](#12-how-the-main-features-were-built)
13. [Running it locally, settings and tests](#13-running-it-locally-settings-and-tests)
14. [Placeholders and known gaps](#14-placeholders-and-known-gaps)

---

## 1. What HireFlow does

HireFlow is an **AI hiring platform for companies**. A company's HR team:

1. writes a job (the AI drafts the job description),
2. publishes it to its careers page and LinkedIn,
3. receives applications (resume upload with phone verification),
4. lets the AI score every resume (ATS),
5. sends shortlisted students through **rounds** that HR chooses and orders. The round types are:
   - AI voice screening
   - Aptitude and English test
   - DSA coding
   - SQL
   - Group Discussion with an AI moderator
   - AI Technical Interview
   - HR interview
6. reviews every result and **makes the decision**.

**The AI only advises. HR always decides.** You will see this rule in many places in the code. Some examples:

- An AI score is stored as advice only.
- A terminated test waits for HR instead of rejecting anyone automatically.
- HR must give a written reason when it overrides a result.

HireFlow is **multi-company** ("multi-tenant"). Every database table carries a `tenant_id`, and the API always takes the company from the logged-in session, never from the request. Another company's IDs simply return 404.

---

## 2. The big picture: the 7 running parts

```
                         Browser
     ┌──────────────────────┬───────────────────────┬──────────────────────┐
     │  HR Dashboard :3000  │  Careers Site :3300   │  Candidate App :3001 │
     │  (HR + admins)       │  (students: apply,    │  (students: DSA/SQL  │
     │                      │  screening, tests,    │  coding test)        │
     │                      │  GD, AI interview)    │                      │
     └──────────┬───────────┴───────────┬───────────┴──────────┬───────────┘
                │ cookie session        │ link token (header)  │ link token (URL)
                ▼                       ▼                      ▼
          ┌─────────────────────────────────────────────────────────┐
          │        API (NestJS) :4000   /api/v1/...                 │
          │  auth · jobs · publisher · applications · ats ·         │
          │  screening · rounds · round-engine · dsa · sql ·        │
          │  group-discussion · ai-interview · assessments          │
          └───┬──────────────┬──────────────┬──────────────┬────────┘
              │ Prisma       │ HTTP JSON    │ HTTP         │ SMTP / HTTP
              ▼              ▼              ▼              ▼
        PostgreSQL     AI worker      GD worker      Mailpit/Gmail, S3,
        (Neon or       (Python)       (Python)       LinkedIn, Piston,
        Docker)        :8000          :8010          LiveKit, ClamAV
                         │              │
                         ▼              ▼
                   Google Gemini   faster-whisper, Piper TTS,
                   (LLM + TTS)     Gemini, LiveKit agent
```

| # | Part | Folder | Language / framework |
|---|---|---|---|
| 1 | API (the brain: all rules, all data) | `apps/api` | TypeScript, NestJS 11 |
| 2 | HR Dashboard | `apps/hr-dashboard` | Next.js 15, React 19 |
| 3 | Careers Site (all student pages except coding) | `apps/careers-site` | Next.js 15, React 19 |
| 4 | Candidate App (DSA / SQL coding page) | `apps/candidate-app` | Next.js 15, React 19 |
| 5 | AI worker (JD, resume scoring, voice screening, AI interview) | `workers/ai-worker` | Python, FastAPI |
| 6 | GD worker + AI moderator (group discussion) | `workers/gd-agent` | Python, FastAPI, LiveKit Agents |
| 7 | Database | `apps/api/prisma` | PostgreSQL via Prisma |

**The golden rule:** the frontends only *show* things and *report* what the browser saw. The API owns the real data:

- the clock
- the questions
- the answers
- the scores
- the warning counts

A candidate's browser can never send a score, a deadline, a warning count or another person's ID.

---

## 3. Libraries we use (tech stack)

### Whole repository
| Library | What it is used for |
|---|---|
| **pnpm 9** (workspaces) | One install for all JavaScript/TypeScript projects (`pnpm-workspace.yaml`) |
| **Turborepo 2** (`turbo.json`) | Runs `build`, `dev`, `test`, `typecheck` across all apps in the right order |
| **TypeScript 5** | Every JS project |
| **Prettier 3** | Code formatting (`.prettierrc`: single quotes, width 100) |
| **Zod 3** | Shape checking of every request and response (in `packages/contracts`) |
| **Vitest 2** | Tests in all TypeScript projects |
| **uv** | Python package manager for the two Python workers |
| **GitHub Actions** (`.github/workflows/ci.yml`) | On every pull request: `pnpm check` (build, lint, typecheck, test), plus `ruff` and `pytest` for the AI worker |

### Backend (`apps/api`)
| Library | Used for |
|---|---|
| `@nestjs/*` 11 | Web framework: modules, controllers, services, guards |
| `@prisma/client` / `prisma` 6.19 | Database access and migrations |
| `nodemailer` 6.9 | Sending every email (Gmail or any SMTP; Mailpit locally) |
| `@aws-sdk/client-s3` | Storing resumes in S3-compatible storage |
| `livekit-server-sdk` | Group discussion video rooms (access tokens, starting the AI moderator) |
| `multer` (through `@nestjs/platform-express`) | Uploads: resumes, voice answers |
| `zod` | Request validation (`parseBody()`), shared with the frontends |
| Node built-ins | `crypto` (token hashing, scrypt passwords, AES-GCM), `node:sqlite` (SQL sandbox), `fetch` (calls to the AI worker, Piston, LinkedIn) |

### Frontends
| Library | Where | Used for |
|---|---|---|
| `next` 15.5, `react` 19.1 | all three | Pages (App Router in `src/app`) |
| `tailwindcss` 4 (`@tailwindcss/postcss`) | all three | Styling; there is no `tailwind.config`, just `@import "tailwindcss"` |
| `@hireflow/contracts` | HR dashboard, careers site | Shared types and checks with the API |
| `livekit-client`, `@livekit/components-react` | HR dashboard, careers site | Live group discussion video room |
| `@mediapipe/tasks-vision` 0.10.21 | careers site, candidate app | **Counting faces in the webcam, on the device** (protection) |
| Browser APIs (no library) | careers site, candidate app | `getUserMedia` (camera/mic), `MediaRecorder` (voice answers), `AudioContext` (mic level), Web Speech API (live transcript preview), Fullscreen API, `visibilitychange`/`blur`/`focus` events (tab switching), Permissions API |

### Python workers
| Library | Worker | Used for |
|---|---|---|
| `fastapi`, `uvicorn`, `pydantic` | both | HTTP server and data checks |
| `google-genai` | both | Google **Gemini**: text generation, transcription and text-to-speech |
| `pyyaml` | ai-worker | Loading prompts from `prompts/*.yaml` |
| `pypdf`, `python-docx` | ai-worker | Reading resume text |
| `faster-whisper`, `av` (PyAV), `numpy` | gd-agent | Transcribing GD audio on the CPU, mixing the recordings |
| `livekit-agents`, `piper-tts` | gd-agent | The AI moderator that joins the GD room and speaks (offline voice) |
| `pytest`, `ruff`, `httpx` | both | Tests and linting |

### Outside services
| Service | Used by | Why |
|---|---|---|
| **PostgreSQL** (Neon in the cloud, or Docker `pgvector/pgvector:pg17` locally) | API | All data |
| **Google Gemini** | AI worker, GD worker | All AI: writing, scoring, transcribing, speaking |
| **Piston** (Docker, `infra/piston`) | API `dsa` module | Runs students' Python / Java / C++ code safely |
| **LiveKit** | API, GD worker, frontends | Real-time group discussion video and audio |
| **Mailpit** (Docker) or Gmail | API | Emails (Mailpit catches them locally on port 8025) |
| **S3 / MinIO**, **ClamAV** | API | Resume storage and virus scanning (both optional) |
| **LinkedIn API**, **Google sign-in**, **Cloudflare Turnstile** | API | Publishing jobs, "Continue with Google", CAPTCHA |
| **cloudflared** tunnels | `scripts/start-public.ps1` | Public links for students during local demos |

---

## 4. Folder map

```
hireflow/
├── apps/
│   ├── api/                  NestJS backend (section 7) + Prisma database (section 8)
│   ├── hr-dashboard/         HR website (section 9.1)
│   ├── careers-site/         Student website (section 9.2)
│   ├── candidate-app/        DSA / SQL coding page (section 9.3)
│   └── admin-console/        EMPTY placeholder (admin pages live in hr-dashboard /admin)
├── packages/
│   ├── contracts/            THE shared types + Zod schemas (the only package with code)
│   └── config, ui, email-templates, proctor-sdk, round-sdk, rounds/*   README-only placeholders
├── workers/
│   ├── ai-worker/            Python AI service (section 10.1)
│   ├── gd-agent/             Python GD transcription / evaluation / AI moderator (section 10.2)
│   └── workflows/, activities-ts/   README-only (Temporal, planned)
├── prompts/                  Versioned AI prompts (YAML), loaded by the AI worker
├── infra/
│   ├── docker/docker-compose.yml   Postgres, Redis, Mailpit (+ optional MinIO, ClamAV, Temporal)
│   ├── piston/               Settings + start script for the code runner
│   └── k8s/, terraform/      README-only
├── scripts/                  start-public.ps1 (whole stack + tunnels), sync.sh, demos
├── docs/                     Feature documents (this guide; see docs/README.md)
├── seed/                     Sample resumes
└── .env.example              Every setting, with comments
```

**Module rule:** each folder has a README that names an owner. Other code imports a module **only through its `index.ts`**, never its inner files or tables.

---

## 5. How the parts talk to each other

### 5.1 Three kinds of login

| Who | How they are recognised | Where |
|---|---|---|
| **HR / Company Admin** | httpOnly session cookie. The database stores only the hash of the token. Every write must also send the header `x-requested-with: hireflow` (CSRF protection) | `apps/api/src/modules/auth/*` |
| **Platform Admin** | The same session, plus the `@PlatformAdminOnly()` route rule | `auth/access.ts` |
| **Student (candidate)** | A **personal link token** from their email. The database stores only its SHA-256 hash. The token works for exactly one application and one round, and it expires | each round's `link.ts` |

The student's token travels in a header, never in an API URL, so it stays out of server logs:

| Round | Header |
|---|---|
| AI voice screening | `x-screening-token` |
| Online test | `x-assessment-token` |
| AI interview | `x-interview-token` |
| Group discussion | `x-gd-token` |

The coding page is the one exception: its token is in the URL path (`/dsa/:token`).

Every API route **must** declare who may call it, or the global `AuthGuard` blocks it ("fails closed"). The rules are:

- `@Public()` for anyone
- `@Authenticated()` for any logged-in user
- `@PlatformAdminOnly()` for platform admins
- `@RequirePermission('jobs.read')` (or another permission) for company members

The roles are `COMPANY_ADMIN` and `HR_MEMBER`, and the permission matrix is in `auth/permissions.ts`.

### 5.2 Frontend to API
- **HR dashboard** calls the API directly (`NEXT_PUBLIC_API_URL`) with `credentials: 'include'`.
- **Careers site** and **candidate app** call their *own* address (`/api/v1/public/...` or `/api/v1/dsa/...`). The `rewrites()` rule in `next.config.ts` forwards these calls to the API. This means:
  - one public tunnel URL is enough
  - HR routes can never be reached through the student sites
- The request and response shapes come from `@hireflow/contracts`. If the API changes a shape, TypeScript breaks the frontend build, which catches the mistake early.

### 5.3 API to AI worker
- Plain HTTP JSON calls to `AI_WORKER_URL` (default `http://localhost:8000`). There is one small client file per module, for example `ai-interview-worker.client.ts` and `ats-worker.client.ts`.
- Every call has a time limit (the `*_TIMEOUT_MS` settings). If the AI fails, the API **falls back to rules**:
  - an interview plan is built by rule
  - the next question comes from the plan
  - the report is marked "failed, retry"

  An interview or test never gets stuck because of the AI.
- The Python Pydantic models (`ai_worker/schemas.py`, `interview/models.py`) are **hand-written copies** of `packages/contracts`. The pull request template reminds you to change both together.

### 5.4 API to GD worker and LiveKit
- The API calls `GD_WORKER_URL` (`:8010`) for `/gd/transcribe`, `/gd/evaluate` and `/gd/mix`, sending the header `x-worker-secret`.
- The API creates LiveKit access tokens. When a group starts, it *dispatches* the `gd-moderator` agent into the room.

### 5.5 Background loops (no queue server)
There is no Redis queue or Temporal yet. Each module that needs background work has a **worker class** with a `setInterval` loop. The loop is switchable with an env setting and is always off in tests.

| Loop | File | What it does |
|---|---|---|
| ATS | `ats/ats-processor.service.ts` | Scores new resumes |
| Decision emails | `ats/decision-email.sender.ts` | Sends shortlist / reject emails |
| Round engine | `round-engine/round-engine.worker.ts` | Sends test invites, times out tests, retries grading, sends outcome emails |
| Group discussion | `group-discussion/gd.worker.ts` | Pools, invites, the live timeline, transcription and evaluation |
| AI interview | `ai-interview/ai-interview.worker.ts` | Invites, timeouts, reports, retrying HR decisions |

Running several API processes at once is safe. The loops use these patterns so that work is never done twice:

- unique keys
- conditional updates (`updateMany` with a `where` on the old state)
- `FOR UPDATE SKIP LOCKED` claims

### 5.6 How rounds hand over to each other
- `rounds-config` stores HR's rounds in HR's order.
- When a student passes a round, **`round-engine`** decides the next step (`online-test.ts → nextStep()`):
  - an online test: the round engine itself sends the invite
  - a DSA round: also run by the round engine
  - a **round module** (Group Discussion, AI Interview, SQL): the module *connects* at startup through `RoundModules.connect('AI_INTERVIEW')` and polls `waitingForModule()`
  - nothing left, or a fail: an outcome email
- Round modules send their result back with `RoundEngineService.recordModuleResult()`. The round engine then queues the next step exactly once.

---

## 6. The hiring journey, step by step, with the files involved

| Step | What happens | Backend module | Frontend page | AI used |
|---|---|---|---|---|
| 1. Company signs up | HR account, company profile, team invitations | `auth`, `tenants` | `/signup`, `/register-company`, `/team` | none |
| 2. Job + JD (M01–M03) | HR fills the job form; the AI writes the JD; HR edits (versions) and approves | `jobs` → AI worker `/v1/jd/generate` | `/jobs/new`, `/jobs/[id]` | Gemini + `prompts/jd_generate.v3.yaml` |
| 3. Rounds (M05) | HR picks and orders the rounds, writes or AI-generates questions, sets pass rules | `rounds-config` → `/v1/rounds/questions` | publish dialog → rounds editor | `round_questions.v1.yaml` |
| 4. Publish (M06–M07) | Posts to the careers site and LinkedIn | `publisher` | `/publish` | JD check `/v1/jd/validate` |
| 5. Apply (M08–M09) | Student opens the job page, verifies the phone (OTP), uploads a resume (virus scan, S3) | `applications` | careers `/[company]/jobs/[job]` | none |
| 6. ATS (M10–M11) | Resume is parsed and scored in the background; HR sees the shortlist and decides; emails go out | `ats` → `/v1/resume/screen` | `/shortlisted` | `resume_screen.v1.yaml` (+ OCR for scanned PDFs) |
| 7. AI voice screening (M12) | Student talks to an AI about education, availability... HR reads the notes | `screening` → `/v1/screening/*` | careers `/screening/[token]` | Gemini transcription + TTS |
| 8. Online tests (M13) | Aptitude & English and other multiple-choice tests: timer, autosave, webcam, **warnings** | `round-engine`, `assessments` | careers `/assessment/[token]` | none (graded by rules) |
| 9. DSA / SQL coding | Code editor; code runs in Piston; SQL runs in a sandbox | `dsa`, `sql`, `round-engine` | candidate-app `/dsa/[token]` | none |
| 10. Group discussion (M18) | Live video group with an AI moderator; Whisper transcript; AI rubric; HR confirms | `group-discussion` → GD worker, LiveKit | careers `/gd/[token]`, HR `/group-discussion/*` | Whisper, Gemini, Piper |
| 11. AI technical interview | Spoken interview on resume, job and scenarios; report; **warnings**; HR decides | `ai-interview` → `/v1/interview/*` | careers `/interview/[token]` | Gemini (plan, transcribe, next question, report, TTS) |
| 12. Review | HR sees everything per student and can override with a reason | `assessments` | `/assessments`, `/assessments/[applicationId]` | none |

---

## 7. Backend: `apps/api`

### 7.1 How it starts
1. `src/main.ts` loads the repo-root `.env` (`common/env.ts`).
2. It creates the Nest app, sets the prefix `api/v1`, turns on CORS, and listens on `API_PORT` (default 4000).
3. `src/app.module.ts` lists every module. `AuthGuard` is registered globally.

### 7.2 Shared helpers: `src/common/`
| File | What it does |
|---|---|
| `feature-flags.ts` | `FEATURE_<NAME>=off` turns a module's routes off (they answer 404 `FEATURE_DISABLED`). Default: on |
| `prisma.service.ts` | One database client for the whole app |
| `api-error.ts` | The standard error body `{ statusCode, code, message }`; `parseBody()` checks a body with Zod |
| `secret-box.ts` | AES-256-GCM encryption for stored OAuth tokens (`ENCRYPTION_KEY`) |
| `uuidv7.ts` | Time-ordered IDs for every row |

### 7.3 Inside a typical module
Most modules follow this pattern. `ai-interview` is used as the example:

```
ai-interview/
├── index.ts                         the ONLY thing other modules import
├── ai-interview.module.ts           wires everything together (Nest providers)
├── public-interview.controller.ts   routes for students   (/public/interview/*)
├── hr-interview.controller.ts       routes for HR         (/ai-interview/*)
├── ai-interview.service.ts          the rules for students (uses the store + worker client)
├── ai-interview-hr.service.ts       the rules for HR
├── ai-interview.store.ts            ALL database access (abstract class + Prisma version)
├── ai-interview-worker.client.ts    HTTP client to the Python AI worker
├── ai-interview.worker.ts           background loop
├── flow.ts / protection.ts          PURE rules (no database, no HTTP), easy to test
├── emails.ts, link.ts               email text; link token create/hash
└── README.md
```

**Why have an abstract "store"?** Tests swap the Prisma store for an **in-memory store** (`tests/*/fakes.ts`). That is why 650+ API tests run in seconds without a database.

### 7.4 Every module in one line
| Module | Purpose | Main routes | Talks to |
|---|---|---|---|
| `auth` | Login, sessions, sign-up, 2FA codes, Google sign-in, password reset, team and invitations, global guard | `/auth/*`, `/company/members`, `/company/invitations` | nodemailer, Google OIDC |
| `tenants` | Companies, company profile, platform admin, "who is the current company" | `/companies`, `/company-profile`, `/admin/companies` | — |
| `audit` | Append-only log of important actions (`audit_logs`) | — | used by everyone |
| `jobs` | Jobs, AI JD generation, versions, approval | `/jobs/*` | AI worker |
| `publisher` | Publish to the careers site and LinkedIn, LinkedIn OAuth | `/publisher/*` | LinkedIn API, AI worker |
| `applications` | Public job pages, OTP, apply, resume storage and virus scan, "my data", HR list, **mailer** | `/public/:company/jobs`, `/public/applications`, `/applications/*` | S3, ClamAV, Turnstile, nodemailer |
| `ats` | Resume parsing and scoring loop, shortlist, HR decisions, decision emails | `/ats/*` | AI worker |
| `screening` | AI voice screening | `/public/screening/*`, `/screening/*` | AI worker (turn, speak, extract) |
| `rounds-config` | HR's rounds, questions, pass rules; AI question ideas; `RoundModules` registry | `/rounds/jobs/:jobId` | AI worker |
| `assessments` | Round results, HR overrides, the Assessments board | `/assessments/*` | written by round-engine and modules |
| `round-engine` | Online tests: invites, attempts, autosave, grading, next step, emails, **test protection** | `/public/assessment/*`, `/round-engine/*` | assessments, rounds-config |
| `dsa` | Coding page backend: timer, runs code on Piston, scoring, coding-page proctoring | `/dsa/:token/*`, `/dsa-hr/*` | Piston, round-engine, sql |
| `sql` | SQL question bank and safe SQL sandbox (`node:sqlite` in a child process) | `/sql-hr/questions/*` | — |
| `group-discussion` | GD pools, groups, live rooms, recordings, transcription, AI evaluation, HR confirm | `/public/gd/*`, `/group-discussion/*` | LiveKit, GD worker |
| `ai-interview` | AI technical interview, report, HR decision, **interview protection** | `/public/interview/*`, `/ai-interview/*` | AI worker, round-engine |
| `analytics`, `evaluator`, `feedback`, `notifications`, `proctoring`, `scheduler` | **README-only placeholders** | — | — |

---

## 8. Database: Prisma and PostgreSQL

- **Connection:** `DATABASE_URL`. The team uses **Neon** (cloud Postgres); locally you can use Docker Postgres on port 5432 or 5433.
- **Schema:** split into one file per area in `apps/api/prisma/schema/`. `prisma.config.ts` points Prisma at this folder.

| File | Main tables |
|---|---|
| `auth.prisma` | User, Session, Membership, Invitation, PasswordReset, LoginChallenge, SignupChallenge |
| `tenants.prisma` | Tenant, CompanyProfile |
| `jobs.prisma` | Job, JdVersion, JobApproval |
| `publisher.prisma` | PublisherJobListing, JobPost, PublisherChannelSetting, ChannelCredential |
| `applications.prisma` | Candidate, Application, Resume, ResumeFile, Consent, PhoneOtpChallenge |
| `ats.prisma` | JobScreeningSettings, AtsEvaluation, CandidateEmail, ScreeningInvite |
| `screening.prisma` | ScreeningSession, ScreeningTurn, ScreeningResult |
| `rounds.prisma` | JobRoundConfig, JobRound, RoundQuestion, RoundResult |
| `round_engine.prisma` | RoundInvite, RoundAttempt, ProctorEvent, **RoundProtectionIncident**, RoundOutcomeEmail, DsaProctorSession |
| `sql_bank.prisma` | SqlQuestion |
| `group_discussion.prisma` | GdEvent, GdBatch, GdParticipant, GdRecording, GdTranscript, GdEvaluation, ... |
| `ai_interview.prisma` | AiInterviewInvite, AiInterviewSession, AiInterviewTurn, AiInterviewEvent, AiInterviewReport, **AiInterviewViolation**, **AiInterviewProtectionSetting** |
| `audit.prisma` | AuditLog |

- **Migrations:** `apps/api/prisma/schema/migrations/`, 31 folders in date order. Apply them with `cd apps/api && npx prisma migrate deploy`. New ones are only **additive**: new tables, or new columns with defaults.
- **Rules:**
  - every table that belongs to a company has `tenant_id`
  - IDs are UUIDv7
  - deleting an application deletes everything about it (`ON DELETE CASCADE`), which is how the candidate's "delete my data" works
  - announce schema changes to the team before running `migrate dev`

---

## 9. Frontends: the three Next.js apps

All three use Next.js 15 App Router (`src/app`), React 19 and Tailwind 4.

### 9.1 `apps/hr-dashboard` (port 3000): for HR and admins
- **Login handling:** `components/session.tsx` loads `/auth/me`, sends you to `/login` when you are logged out, and shows the menu based on your role.
- **API clients** (`src/lib/*-api.ts`): one file per backend module, for example `ai-interview-api.ts`, `round-engine-api.ts` and `gd-api.ts`. All of them use the `request()`/`call()` helper (cookie + CSRF header).
- **Main pages:**
  - Jobs and JD: `/jobs`, `/jobs/new`, `/jobs/[id]`
  - Publishing, applicants and shortlist: `/publish`, `/applications`, `/shortlisted`
  - Results: `/assessments` (the board), `/assessments/[applicationId]` (everything about one student)
  - Group discussion: `/group-discussion/*` (including the live room)
  - Admin pages: `/sql-questions`, `/company`, `/team`, `/admin`
- **Important components:**
  - `rounds/rounds-editor.tsx`, `round-card.tsx` and `question-editor.tsx` for the rounds setup
  - `rounds/interview-protection-settings.tsx` for the AI interview's warning rules
  - `ai-interview-panel.tsx` for the interview report, transcript, incidents and decision
  - `online-test-panel.tsx` for an online test's attempt, proctoring log, **test protection**, and the integrity decision

### 9.2 `apps/careers-site` (port 3300): for students
- `next.config.ts`:
  - forwards `/api/v1/public/*` to the API
  - adds privacy headers on token pages (no referrer, no cache, noindex)
  - sets a **`Permissions-Policy`** per page, so camera / microphone / fullscreen are allowed only where they are needed
- `src/middleware.ts` sets the real client IP header used by the API's rate limits (`lib/client-ip.ts`).
- **Pages:**

| Page | Main file | What it is |
|---|---|---|
| `/[company]`, `/[company]/jobs/[job]` | `apply-form.tsx` | Job list, job page, apply form (OTP, resume upload with progress) |
| `/[company]/my-data` | | See or delete your data |
| `/screening/[token]` | `screening-room.tsx` | AI voice screening |
| `/assessment/[token]` | `exam-room.tsx` | Online test with protection |
| `/interview/[token]` | `interview-room.tsx` | AI technical interview with protection |
| `/gd/[token]` | `gd-lobby.tsx`, `gd-room.tsx` | Group discussion |

- **`src/lib/` helpers:**

| File | Job |
|---|---|
| `recorder.ts` | Records the microphone with `MediaRecorder`, picks a format the browser supports (WebM-Opus or MP4), measures loudness |
| `camera.ts` | Starts the webcam preview, explains camera errors, watches for revoked permission |
| `live-transcript.ts` | Live words while you speak (browser Web Speech API; preview only, never saved) |
| `interview-ui.ts` | Interview screen states, `EndOfAnswer` (detects when you stopped talking), error texts |
| `assessment-ui.ts` | Test logic: answers, question palette, timer, local draft |
| **`interview-protection.ts`** | `ProtectionMonitor` (tab / window / fullscreen → incidents), `FaceEpisodeTracker` (several faces → incident), warning texts |
| **`face-counter.ts`** | Loads MediaPipe's face detector and counts faces in the video |
| `*-api.ts` | One client per round; the link token goes in a header |
| `gd-recorder.ts` | Records your GD audio in 5-second pieces and uploads them in order |

- `scripts/copy-face-detector.mjs` runs before `dev` and `build`. It copies the MediaPipe WASM files into `public/mediapipe/` (git-ignored). The face model `public/models/blaze_face_short_range.tflite` is committed.

### 9.3 `apps/candidate-app` (port 3001): DSA / SQL coding page
- One page: `/dsa/[token]` (`assessment.tsx`). It has:
  - a dependency-free code editor (`code-editor.tsx`)
  - the problem panel and test results
  - its own proctoring (`proctoring.tsx`: fullscreen, tab, copy/paste blocking, key shortcuts)
  - a webcam with MediaPipe face checks (`webcam.tsx`)
- It does not use `@hireflow/contracts`; the types are copied by hand in `lib/dsa-api.ts`.
- `next.config.ts` forwards only `/api/v1/dsa/*` and allows long requests, because code runs can wait in a queue.

### 9.4 `apps/admin-console`
This folder is empty. The platform admin pages are in the HR dashboard under `/admin`.

---

## 10. Python workers (AI)

### 10.1 `workers/ai-worker` (FastAPI, port 8000)
- **Start:** `cd workers/ai-worker && uv sync && uv run python -m ai_worker`. API docs are at `http://localhost:8000/docs`.
- **Settings:** `config.py` reads the repo-root `.env`.
- **How it is organised:**
  - `main.py` holds every route.
  - Each route uses one feature folder.
  - The AI is reached through two small "provider" interfaces, so the model can be swapped:
    - `llm_gateway/` for text: `GeminiProvider`, which asks Gemini for **JSON that matches a schema**, retries on server errors and tries fallback models
    - `speech/` for voice: `GeminiSpeechProvider`, which turns text into WAV audio
- **Prompts:** loaded from `prompts/*.yaml` with `llm_gateway/prompts.py`. Student text is escaped (`escape_data`), so a resume or an answer cannot inject instructions.

| Route | Feature folder | What it does |
|---|---|---|
| `POST /v1/jd/generate`, `/v1/jd/validate` | `jd/` | Write the JD / check it |
| `POST /v1/resume/screen` | `parsing/` (pypdf, python-docx, OCR) + `scoring/ats.py` | Read the resume, then score it with a **fixed formula**: skills 40, experience 20, projects 20, achievements 10, education 10 |
| `POST /v1/screening/turn`, `/speak`, `/extract` | `screening/` | Voice screening: transcribe the answer, choose a follow-up, speak, write notes for HR |
| `POST /v1/interview/plan`, `/turn`, `/speak`, `/report` | `interview/` | AI interview (section 12.2) |
| `POST /v1/rounds/questions` | `rounds/` | Question ideas for HR |

There is no separate speech-to-text library: the audio is sent to Gemini, which writes the transcript.

**Tests:** about 115 `pytest` tests. A safety fixture makes any test fail if it tries to call the real Gemini. Run them with `uv run pytest`.

### 10.2 `workers/gd-agent` (FastAPI, port 8010, plus a LiveKit agent)
| Command | What it runs |
|---|---|
| `uv run python -m gd_agent api` | The HTTP service: `/gd/transcribe` (faster-whisper on the CPU), `/gd/evaluate` (Gemini rubric), `/gd/mix` (PyAV mixes all tracks into `session.webm`) |
| `uv run python -m gd_agent moderator dev` | The **AI moderator**. It joins the LiveKit room and speaks with Piper (an offline voice) |
| `uv run python -m gd_agent download` | One-time download of the Whisper and Piper models |

- It must run on the **same machine** as the API, because it reads the audio files the API saved (`GD_STORAGE_DIR`).
- The API keeps the GD timeline in the room's metadata, so the discussion continues even if the moderator drops.

---

## 11. Shared code: `packages/` and `prompts/`

### 11.1 `packages/contracts` (`@hireflow/contracts`): the most important shared package
- This is where we write down **"what does this request / response look like?"** once, with Zod. Both the API and the frontends use it.
- **Build:** run `pnpm --filter @hireflow/contracts build` after changing it. The other apps read the built `dist/` folder.

| File | What it defines |
|---|---|
| `auth.ts`, `company.ts`, `job.ts` | Accounts, company profile, job and JD shapes |
| `publisher.ts`, `application.ts`, `resume.ts`, `ats.ts` | Publishing, apply form, parsed resume, ATS |
| `screening.ts` | AI voice screening |
| `rounds.ts`, `assessments.ts` | Round setup, pass rules, results, overrides |
| `round-engine.ts` | Online tests: public test page, events, **test protection**, integrity decision |
| `dsa-bank.ts`, `sql-bank.ts` | Coding and SQL question banks |
| `group-discussion.ts` | GD |
| `ai-interview.ts` | AI interview: statuses, report, **interview protection** (settings, incident types, face policy) |
| **`protection.ts`** | `countProtectionIncident()`: **the one warning counter** used by both the interview and the tests |

### 11.2 `prompts/`
Versioned YAML prompts. Each file has a task, a version, a temperature, the system text and the user text. They are never hard-coded in the Python code.

| Area | Prompt files |
|---|---|
| JD | `jd_generate.v3` (v1 and v2 are kept but not used) |
| Resume | `resume_screen.v1`, `resume_transcribe.v1` |
| Voice screening | `screening_transcribe.v1`, `screening_turn.v1`, `screening_extract.v1` |
| Rounds | `round_questions.v1` |
| AI interview | `interview_plan.v1`, `interview_transcribe.v1`, `interview_turn.v1`, `interview_report.v1` |

### 11.3 Placeholder packages
`config`, `ui`, `email-templates`, `proctor-sdk`, `round-sdk` and `rounds/*` contain only a README. The features they were planned for were built directly inside the API modules and the apps.

---

## 12. How the main features were built

### 12.1 Interview and test **protection** (tab switch, fullscreen, camera, microphone, multiple faces)

**Goal:**

- notice when a student leaves the page or someone else appears on camera
- warn them: Warning 1, then the Final Warning
- end the round on the 3rd incident
- let **HR decide**: a termination is never an automatic rejection

**Where it is used:**

- AI technical interview: `/interview/[token]`
- Online tests (Aptitude & English and the other multiple-choice tests): `/assessment/[token]`

The DSA coding page has its own similar system in the `dsa` module.

#### Step 1. The browser notices things (careers site)
| What | How it is detected | Library / browser API | File |
|---|---|---|---|
| Tab switch, minimise, screen lock | `document.visibilitychange` → `hidden` | Page Visibility API | `lib/interview-protection.ts` (`ProtectionMonitor`) |
| Another window or app | `window.blur` / `focus`; counts only after a **2-second grace**, because clicking the address bar also blurs | DOM events | same |
| Left fullscreen (Escape) | `document.fullscreenchange` | Fullscreen API | same |
| Camera or mic stopped, unplugged, permission revoked | the media track's `ended` event; camera `readyState` checked every 100 ms | `getUserMedia` / MediaStream | `interview-room.tsx`, `exam-room.tsx` |
| More than one person | Counts faces about **once per second** on the device. 2 or more faces for **5 seconds** = one incident; it ends after 3 seconds with one face | **MediaPipe Tasks Vision** `FaceDetector` (BlazeFace short-range model, WASM, CPU) | `lib/face-counter.ts` + `FaceEpisodeTracker` |

**Grouping events into one incident:** one action fires several events. For example, Alt+Tab out of fullscreen fires *blur + fullscreen exit + hidden*, and coming back fires *visible + focus*. `ProtectionMonitor` turns all of that into **one incident with one random `incidentId`**. Coming back never counts.

**Reloads are not incidents:** a reload or closing the page also hides it. To ignore that, the page marks itself "unloading" on `pagehide` / `beforeunload`, and hidden events after that are ignored.

**Privacy:** for faces, only the **number** of faces is used. No image leaves the device, nobody is identified, and there is no gaze or emotion analysis.

The face counter was built like this:

1. Install `@mediapipe/tasks-vision`.
2. Copy its WASM runtime into `public/mediapipe/` with `scripts/copy-face-detector.mjs`, and commit the model file `public/models/blaze_face_short_range.tflite`. Both are served from our own site, with no third-party CDN.
3. Load them only when the round starts, using `FilesetResolver.forVisionTasks()` and `FaceDetector.createFromOptions()`.
4. Run `detectForVideo()` on the `<video>` preview once per second.

#### Step 2. The browser reports, the server decides
- The page sends `POST /public/interview/violation` (or `/public/assessment/violation`) with `{ incidentId, type }` using `fetch` with `keepalive`, so it still arrives while the tab is being hidden.
- Failed sends wait in an **outbox** and are retried later with the *same* ID.
- **The page never sends or decides a warning count.**

#### Step 3. One counter, safe under concurrency (API)
- **Shared rule:** `countProtectionIncident()` in `packages/contracts/src/schemas/protection.ts`:
  - the first qualifying incident gives warning 1
  - the next ones go up to `maxWarnings` (default 2; the last one is the final warning)
  - the one after that **terminates**
  - an incident within **3 seconds** of the last counted one is GROUPED with it
- **Interview side:**
  - `ai-interview/protection.ts`: which types count (devices count only if HR chose "WARN")
  - `protection.service.ts` (`InterviewProtectionService`): checks the link and saves the result
- **Test side:** `round-engine/assessment-protection.ts` + `RoundEngineService.violation()`.
- **Concurrency:** the store runs everything in **one database transaction** that first locks the row with `SELECT ... FOR UPDATE`. Two reports at the same moment are therefore counted one after the other. Database guards back this up:
  - a unique `(session, incidentId)` key, so a re-sent incident is counted once
  - a partial unique index so a warning number cannot be used twice
  - `CHECK (warning_count >= 0)`

#### Step 4. Termination and HR review
**Interview:**

- The session becomes `TERMINATED`, and its integrity status becomes `INTEGRITY_VIOLATION_REVIEW`.
- The page stops the AI voice, the microphone and the camera, and leaves fullscreen.
- Any answer still being processed is refused.
- The answers saved so far are kept, and a report is still written from them.

**Test:**

- The attempt becomes `TERMINATED` with the answers saved so far, and it is graded.
- `progress()` then **holds**: no result email and no next invite.
- The worker retry query skips it.

**HR:**

- Sees the incident list (time, type, warning number, acknowledged or not) and the termination reason:
  - interview: `ai-interview-panel.tsx`
  - online test: `online-test-panel.tsx`
- Must write a reason to decide:
  - interview decision: `POST /ai-interview/.../decision`
  - test decision: `POST /round-engine/.../integrity-decision`
- The decision goes through the round engine once. A decision against the score is saved as HR's audited override.

#### Step 5. Settings
- **AI interview:** per round, set by HR in the rounds editor (`interview-protection-settings.tsx` → `ai_interview_protection_settings`). The settings are:
  - number of warnings
  - tab / window monitoring
  - fullscreen
  - camera and microphone policy (`RECOVER` or `WARN`)
  - faces policy (`WARN`, `RECORD` or `OFF`)
- **Online tests:** server-wide env settings: `ASSESSMENT_PROTECTION`, `ASSESSMENT_MAX_WARNINGS`, `ASSESSMENT_FACE_POLICY`, and the others.
- The rules are **copied onto each interview or attempt when it starts**, so the rules the student saw are the rules enforced.
- Turn everything off with `FEATURE_AI_INTERVIEW_PROTECTION=off` or `ASSESSMENT_PROTECTION=off`.

**Tests:**

- API: `tests/ai-interview/protection.test.ts`, `tests/round-engine/assessment-protection.test.ts`
- Browser logic: `careers-site/tests/interview-protection.test.ts`

### 12.2 AI technical interview (voice)
```
Browser                                  API                              AI worker → Gemini
camera + mic (getUserMedia)
question audio <── GET /speech ───────── cache (SpeechCache) ──────────── /v1/interview/speak (TTS)
MediaRecorder answer ── POST /answer ──> saves once (clientTurnId) ────── /v1/interview/turn
                                         server clock, limits,           1) transcribe audio only
                                         next question (flow.ts)         2) read answer, propose next
end of interview ───────────────────────> background report ──────────── /v1/interview/report
```
- **End of an answer:** `EndOfAnswer` (in `interview-ui.ts`) listens to the microphone level through `AudioContext`. After 2.5 seconds of speech, 4.5 seconds of silence ends the answer. The student can also press "Done answering".
- **Rules in code, not in the AI** (`flow.ts`):
  - at most 2 follow-ups per topic
  - no repeated question
  - no early end before the minimum time
  - a final question in the last 90 seconds
  - the AI's proposal is used only if it passes these rules; otherwise the next planned topic is asked
- **The report score is computed in code** from the AI's per-skill evidence. The AI's recommendation is advice only.
- **Live transcript:** the browser's Web Speech API, shown as a preview only. The saved transcript comes from the server.

### 12.3 AI voice screening
Same voice building blocks as the interview (`recorder.ts`, the TTS cache, Gemini transcription), but a **fixed list of steps**: intro, education, background, location, availability, candidate questions. The AI writes structured notes for HR with **no score**. Files: `screening/flow.ts`, `ai_worker/screening/interview.py`.

### 12.4 Online tests (Aptitude & English and other multiple-choice tests)
- **Start:** `round-engine.service.ts → start()` freezes the rounds and creates the attempt with a **server deadline**.
- **Answers:** the page autosaves every change (`PUT /answers` with a growing `revision`; an older save never overwrites a newer one). It keeps a local draft while offline (`assessment-ui.ts`, localStorage).
- **Finishing:** on submit or timeout, `finish()` → `assessments.record()` grades it with the pass rule → `progress()` queues the next step exactly once (`claimProgress`).
- **Webcam:** only a preview. Camera, tab and fullscreen events are logged for HR (`ProctorEvent`), and warnings come from the protection system in 12.1.

### 12.5 DSA and SQL coding rounds
- **Running code:** the `dsa` module wraps the student's function with a test harness (`evaluation/harness.ts`) and runs it on **Piston**, a self-hosted code runner in Docker:
  - `POST /api/v2/execute`
  - no network for student code
  - time and memory limits (`infra/piston/piston.env`)
  - a concurrency queue: `execution/limiter.ts`
- **SQL:** runs in a separate Node child process using the built-in **`node:sqlite`** (`sql/engine/sql-sandbox.ts`), with a memory cap. The result is compared with HR's reference query.
- **Coding-page proctoring:** `dsa/proctoring/*` with a strike policy (`DSA_PROCTOR_*`). On the page side: `candidate-app/.../proctoring.tsx` and `webcam.tsx`.

### 12.6 Group discussion
1. Students who pass wait in a **pool**. HR schedules groups (`planning.ts`).
2. The API creates **LiveKit** rooms and tokens (`livekit.ts`). Students join from `gd-room.tsx` using `@livekit/components-react`.
3. The **AI moderator** (`gd_agent/moderator.py`, Piper voice) follows the timeline that the API writes into the room metadata (`timeline.ts`).
4. Each student's audio is recorded in 5-second chunks (`gd-recorder.ts`) and uploaded.
5. Afterwards, the GD worker:
   - transcribes with faster-whisper
   - evaluates with a Gemini rubric
   - mixes the session audio
6. HR reviews the ranking, can override, and confirms the selection.

### 12.7 Resume ATS
1. **Upload:** OTP proof, file-type check from the file's bytes, ClamAV scan, then storage in S3 or the database.
2. **Text:** `ats-processor.service.ts` sends the resume to `/v1/resume/screen`. pypdf / python-docx read the text; Gemini reads scanned PDFs (OCR).
3. **Score:** the LLM extracts the *evidence*, and **code computes the score** with a fixed formula. HR sets a private minimum score per job.

---

## 13. Running it locally, settings and tests

### 13.1 First time
```bash
pnpm install                                   # all JS projects (+ prisma generate)
cp .env.example .env                           # then fill DATABASE_URL, GEMINI_API_KEY, mail settings
pnpm infra:up                                  # Docker: Postgres, Redis, Mailpit
cd apps/api && npx prisma migrate deploy       # create / update the tables
cd workers/ai-worker && uv sync                # Python AI worker
```

### 13.2 Every day
```bash
pnpm dev                                       # API :4000, HR :3000, careers :3300, candidate :3001
cd workers/ai-worker && uv run python -m ai_worker      # AI :8000
cd workers/gd-agent && uv run python -m gd_agent api    # GD :8010 (only for GD rounds)
```

- **Mail:** emails appear at `http://localhost:8025` (Mailpit).
- **One-click demo stack on Windows:** `pnpm start:public`. It starts everything and opens public tunnel links for students.
- **Coding rounds:** these also need Piston: `infra/piston/start-piston.ps1`.

### 13.3 Settings (`.env.example`)
Every setting has a comment. The main groups are:

| Group | Settings |
|---|---|
| Database / services | `DATABASE_URL`, `AI_WORKER_URL`, `GD_WORKER_URL`, `PISTON_URL`, `LIVEKIT_*` |
| AI | `GEMINI_API_KEY`, `LLM_MODEL`, `LLM_FALLBACK_MODELS`, `SCREENING_TTS_*`, plus `*_BUDGET_SECONDS` / `*_TIMEOUT_MS` per feature |
| Email | `GMAIL_USER` + `GMAIL_APP_PASSWORD`, or `SMTP_URL` |
| Links sent to students | `CAREERS_SITE_URL`, `ASSESSMENT_URL`, `AI_INTERVIEW_URL`, `CANDIDATE_APP_URL` |
| Background loops | `*_PROCESSING`, `*_POLL_MS`, `AUTO_*_INVITES` |
| Feature switches | `FEATURE_PUBLISHER`, `FEATURE_SCREENING`, `FEATURE_ROUND_ENGINE`, `FEATURE_AI_INTERVIEW`, `FEATURE_AI_INTERVIEW_PROTECTION`, `FEATURE_DSA`, `FEATURE_SQL`, `FEATURE_GROUP_DISCUSSION`, ... |
| Test protection | `ASSESSMENT_PROTECTION`, `ASSESSMENT_MAX_WARNINGS`, `ASSESSMENT_TAB_MONITORING`, `ASSESSMENT_FULLSCREEN_REQUIRED`, `ASSESSMENT_FACE_POLICY` |

### 13.4 Tests
| Where | Command | Notes |
|---|---|---|
| Everything (as CI does) | `pnpm check` | build + lint + typecheck + test |
| API | `pnpm --filter api test` | about 650 tests with in-memory fakes. Tests named `*.db.test.ts` run only with `TEST_DATABASE_URL`; Piston / ClamAV / S3 tests need those services |
| Careers site / HR dashboard | `pnpm --filter careers-site test`, `pnpm --filter hr-dashboard test` | pure logic (monitors, form checks, API clients) |
| Contracts | `pnpm --filter @hireflow/contracts test` | |
| AI worker | `cd workers/ai-worker && uv run pytest` | real Gemini only with `RUN_LLM_INTEGRATION=1` |
| GD worker | `cd workers/gd-agent && uv run pytest` | |

---

## 14. Placeholders and known gaps

- **README-only (not built yet):**
  - API modules: `analytics`, `evaluator`, `feedback`, `notifications`, `proctoring`, `scheduler`
  - packages: `config`, `ui`, `email-templates`, `proctor-sdk`, `round-sdk`, `rounds/*`
  - workers: `workflows`, `activities-ts` (Temporal)
  - infra: `infra/k8s`, `infra/terraform`
  - apps: `apps/admin-console` (an empty folder)
- **Background jobs** use simple `setInterval` loops, not a queue server. Rate limiters and the speech cache live in memory, per API process.
- **No authentication between the API and the AI worker.** Keep the AI worker on a private network.
- **Python copies of the contracts** (`ai_worker/schemas.py` and the others) must be updated by hand when `packages/contracts` changes.
- **The candidate app** has no tests and copies its API types by hand.
- **Protection:**
  - the row-lock path and the face counter have not yet been tried against a real database or real webcams
  - browsers cannot see a second device or what was opened in another app
  - see `docs/AI_TECHNICAL_INTERVIEW.md` §9 for more
- **`docs/README.md`** lists three documents that don't exist (`PROJECT_MAP.md`, `JD_WORKFLOW.md`, `MULTI_COMPANY_AUTH.md`) and is missing two that do (`ASSESSMENT_ROUNDS.md`, `M06_M09_PUBLISH_AND_APPLY.md`).

### Where to read more
| Topic | Document |
|---|---|
| Full product spec | `docs/HireFlow_AI_Documentation.md` |
| Team workflow and setup | `docs/TEAM_GUIDE.md`, `docs/FOLDER_STRUCTURE.md` |
| Rounds | `docs/ASSESSMENT_ROUNDS.md`, `docs/APTITUDE_ASSESSMENT_ROUND.md`, `docs/DSA_ASSESSMENT_ROUND.md`, `docs/SQL_ASSESSMENT_ROUND.md`, `docs/GROUP_DISCUSSION_ROUND.md` |
| AI | `docs/SCREENING_ROUND.md`, `docs/AI_TECHNICAL_INTERVIEW.md` |
| Publishing and applying | `docs/M06_M09_PUBLISH_AND_APPLY.md` |
| Each module's details | its own `README.md` |

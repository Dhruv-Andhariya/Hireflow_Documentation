# HireFlow AI — Project Guide

This guide explains **how the whole project works**, folder by folder and file by file. Read parts 1–6 first: they
give the big picture. The later sections go into each folder and list every file.

The guide describes the code on the `develop` branch at commit `a458189` (PR #14 "fix wiring", 5 Nov 2026).
When the code and an older document disagree, the code wins — this guide was written by reading the code.

## Contents

1. [What the project is](#1-what-the-project-is)
2. [The running pieces (processes and ports)](#2-the-running-pieces-processes-and-ports)
3. [One candidate from start to end](#3-one-candidate-from-start-to-end)
4. [How the pieces talk to each other](#4-how-the-pieces-talk-to-each-other)
5. [Repository map](#5-repository-map)
6. [Code patterns used everywhere](#6-code-patterns-used-everywhere)
7. [apps/api — core, database and first-half modules](#appsapi--core-database-and-first-half-modules)
8. [apps/api — screening, rounds and hiring pipeline](#appsapi--screening-rounds-and-hiring-pipeline)
9. [apps/api — round modules: DSA, SQL, group discussion, AI interview](#appsapi--round-modules-dsa-sql-group-discussion-ai-interview)
10. [apps/hr-dashboard and shared packages](#appshr-dashboard-and-shared-packages)
11. [Candidate-facing apps: careers-site, candidate-app, admin-console](#candidate-facing-apps-careers-site-candidate-app-admin-console)
12. [Workers, prompts, scripts, infrastructure and root files](#workers-prompts-scripts-infrastructure-and-root-files)
13. [Where do I look when…](#where-do-i-look-when)

---

## 1. What the project is

HireFlow AI is a hiring platform. It covers the path from **"HR has an opening"** to **"the candidate is selected for
the personal interview"**:

- HR creates a job. AI writes the job description (JD). HR approves it.
- HR chooses the assessment rounds for the job (aptitude, DSA coding, SQL, group discussion, AI technical
  interview, HR interview …) in the order they want.
- The job is published (careers page + LinkedIn). Students apply with a resume.
- AI reads every resume and gives an ATS score. Students above the job's minimum score are shortlisted.
- Shortlisted students do a short AI **voice screening**, then go through HR's rounds one by one. Every invitation,
  result and email happens **automatically** in background loops.
- HR watches everything on the HR dashboard and can override any decision.

It is a **monorepo**: many apps in one Git repository, managed by **pnpm workspaces** and **Turborepo**.

## 2. The running pieces (processes and ports)

```
                       ┌────────────────────────── your laptop ───────────────────────────┐
  HR (browser) ──────► │ hr-dashboard  :3000  (Next.js)                                   │
                       │        │                                                         │
  Student (browser) ─► │ careers-site  :3300  (Next.js) ◄── Cloudflare tunnel (public URL) │
                       │ candidate-app :3001  (Next.js) ◄── Cloudflare tunnel (DSA / SQL)  │
                       │        │  all three call ▼                                       │
                       │ api           :4000  (NestJS, /api/v1)  + background loops       │
                       │        │          │            │                                │
                       │        ▼          ▼            ▼                                │
                       │  ai-worker :8000  gd-agent :8010  Piston :2000 (runs code)       │
                       │  (FastAPI,        (Python, GD       (Docker)                     │
                       │   Gemini)          recording/AI)                                 │
                       └──────────┬───────────────────────────────────────────────────────┘
                                  ▼
             PostgreSQL (Neon cloud, SHARED by the whole team)   Redis :6379   Mailpit :1025/:8025
```

| Process | Port | Folder | Started by |
|---|---|---|---|
| API (NestJS) | 4000 | `apps/api` | `pnpm dev` |
| HR dashboard | 3000 | `apps/hr-dashboard` | `pnpm dev` |
| Careers site (public pages for students) | 3300 | `apps/careers-site` | `pnpm dev` |
| Candidate app (DSA / SQL coding pages) | 3001 | `apps/candidate-app` | `pnpm dev` |
| AI worker (all Gemini calls) | 8000 | `workers/ai-worker` | `uv run python -m ai_worker` |
| GD worker + AI moderator | 8010 | `workers/gd-agent` | `scripts/start-public.ps1` |
| Piston (code runner) | 2000 | `infra/piston` | Docker |
| PostgreSQL | Neon (cloud) or 5432/5433 | `apps/api/prisma` | Neon / Docker |
| Redis, Mailpit | 6379, 1025 / 8025 | `infra/docker` | `pnpm infra:up` |
| Cloudflare quick tunnels | — | `scripts/start-public.ps1` | `pnpm start:public` |

`pnpm start:public` (`scripts/start-public.ps1`) starts **everything** in the right order: Docker services, the
tunnels (and writes their URLs into `.env`), `pnpm dev`, the AI worker and the GD worker.

## 3. One candidate from start to end

Each step names the code that does it. Steps marked ⚙ happen **automatically in a background loop** inside the API.

| # | What happens | Where in the code |
|---|---|---|
| 1 | HR signs up / logs in, registers the company, invites teammates | `apps/api/src/modules/auth`, `tenants` · HR pages `apps/hr-dashboard` |
| 2 | HR fills the job form. The AI worker writes the JD. A rule-based validator checks it. HR edits and approves (each change = new JD version) | `modules/jobs` → `workers/ai-worker` (`/v1/jd/*`, prompts `prompts/jd_generate.*.yaml`) |
| 3 | HR chooses the rounds, their questions and pass marks, and the pipeline rules | `modules/rounds-config`, `modules/pipeline` · `hr-dashboard/src/components/rounds` |
| 4 | HR publishes: careers page + LinkedIn post with the Apply link | `modules/publisher` |
| 5 | Student opens the Apply link, verifies phone (OTP), uploads resume → application saved, "thanks" email | `apps/careers-site` → `modules/applications` |
| 6 ⚙ | Resume is parsed and scored by AI (ATS score) | `modules/ats` (`AtsProcessor`, needs `ATS_PROCESSING=on`) → ai-worker `/v1/resume/screen` |
| 7 ⚙ | Score ≥ job minimum → **shortlist email** with the AI screening link; below → rejection email | `modules/ats` (`DecisionEmailSender`, `AUTO_SHORTLIST_EMAILS`, `AUTO_REJECTION_EMAILS`) |
| 8 | Student does the AI **voice screening** (5–8 min, information only, never a score) | `careers-site /screening/[token]` → `modules/screening` → ai-worker `/v1/screening/*` |
| 9 ⚙ | Screening completed → invitation to the job's **first round** | `modules/round-engine` (`RoundEngineWorker.discover`) |
| 10 | Student takes the round: online test (aptitude/MCQ), DSA or SQL coding, group discussion, AI technical interview … | see "Which module runs which round" below |
| 11 ⚙ | Result recorded and graded → **next step**: next round's invitation, or "cleared all rounds", or rejection email | `round-engine/online-test.ts` `nextStep()` + `RoundEngineService.nextFor()` (+ Stage 2 rule from `pipeline`) |
| 12 | After the last technical round, the Stage 2 overall rule is checked (all passed / total % / both) | `modules/pipeline` + `packages/contracts/src/schemas/pipeline.ts` |
| 13 ⚙ | AI technical interview (only if HR added the round): invite → voice interview → AI report → auto decision by pass mark → "selected for Personal Interview" email | `modules/ai-interview` |
| 14 | HR calls the student for the personal interview, records status (Called / Scheduled / Selected / Rejected) | `modules/pipeline` (`pi_calls`) · `hr-dashboard /personal-interview` |

**Which module runs which round**

| Round type (in the rounds editor) | Run by | Candidate page |
|---|---|---|
| Aptitude, Technical MCQ, Communication (online tests) | `round-engine` itself | careers-site `/assessment/[token]` |
| DSA coding (`CODING`) | `round-engine` invites and times it; `dsa` module runs code on Piston | candidate-app `/dsa/[token]` |
| SQL | `round-engine` too: a SQL problem is a kind of coding problem. `sql` module checks queries in a read-only in-memory SQLite (not Piston) | candidate-app `/dsa/[token]` (same coding page) |
| Group discussion | `group-discussion` module (finds students with its own pool query) | careers-site GD pages + gd-agent |
| AI Technical Interview (`AI_INTERVIEW`) | `ai-interview` module | careers-site `/interview/[token]` |
| Technical / HR interview, custom | HR, outside the system | none — student gets a "next round is run by HR" email |

> **Important:** the technical interview is **not** automatic. It only happens when HR adds an
> "AI Technical Interview" round to the job. If the job has no such round, a student who passes the last round gets
> the "cleared all rounds" email (`CLEARED_ALL`), without an interview link.

## 4. How the pieces talk to each other

**HTTP.** Front-ends call the API at `/api/v1/...`. HR pages use a login cookie (httpOnly) plus a CSRF header on
every write. Candidate pages have no login: they use a **personal link token** (see below). The API calls the AI
worker, gd-agent and Piston over HTTP.

**One shared database.** All state lives in PostgreSQL (Neon). The whole team points at the **same** database, so
every teammate's API sees the same jobs and students. Background loops claim work with
`SELECT … FOR UPDATE SKIP LOCKED`, so two API processes never handle the same row twice.

**Background loops inside the API.** No separate queue server: each loop is a `setInterval` started in
`onApplicationBootstrap()`.

| Loop (class) | Does | On/off switch |
|---|---|---|
| `AtsProcessor` (`modules/ats`) | queue + score new resumes; queues the decision email right after scoring | `ATS_PROCESSING`, `FEATURE_ATS`; what it queues: `AUTO_SHORTLIST_EMAILS`, `AUTO_REJECTION_EMAILS` |
| `DecisionEmailSender` (`modules/ats`) | sends queued shortlist / rejection emails (shortlist = with screening link) | `FEATURE_ATS` only — `ATS_PROCESSING=off` does **not** stop it |
| `RoundEngineWorker` (`modules/round-engine`) | first-round invites after screening, time-out sweep, invitation and outcome emails | `ROUND_ENGINE_PROCESSING`, `AUTO_ROUND_INVITES` (`FEATURE_ROUND_ENGINE=off` only hides routes; the loop keeps running) |
| `GdWorker` (`modules/group-discussion`) | GD pool, auto schedule, reminders, evaluation, auto confirm | `GD_PROCESSING`, `AUTO_GD_*` |
| `AiInterviewWorker` (`modules/ai-interview`) | interview invites, time-out sweep, reports, auto decisions | `AI_INTERVIEW_PROCESSING`, `AUTO_AI_INTERVIEW_*` |
| `PublisherService` (`modules/publisher`) | fix Apply links in posts when the tunnel URL changed | `PUBLISHER_AUTO_REFRESH_LINKS` |

If a loop is off on your laptop (for example `ATS_PROCESSING=off`), that work only happens on a teammate's machine
where it is on.

**Personal links (tokens).** Every candidate link (screening, assessment, DSA, interview, GD) contains a random
token. The database stores only its **SHA-256 hash**. Careers-site pages send the token to the API in a header
(`x-screening-token`, `x-assessment-token`, `x-interview-token`, `x-gd-token`). The candidate-app (coding rounds) is
different: it puts the token in the API path (`/api/v1/dsa/<token>/run`). Links expire (usually 7 days).

**Front-end → API.** The HR dashboard calls `/api/v1` on its **own** address; `next.config.ts` forwards it to the API
(`API_INTERNAL_URL`), so the login cookie stays first-party. Every HR write sends the header
`x-requested-with: hireflow` (CSRF guard). The careers site and candidate app also proxy to the API.

**Public URLs and tunnels.** Students must open links from their own devices, so the careers site and candidate app
are exposed through **Cloudflare quick tunnels**. A quick tunnel gets a **new random URL every time it restarts**.
`start-public.ps1` writes the URL into `.env` (`CAREERS_SITE_URL`, `CANDIDATE_APP_URL`), and the API re-sends links /
edits posts whose old link no longer opens. Emails are never sent with a link that does not open.

**Emails.** Sent by the API's mailer (SMTP / Gmail settings in `.env`; Mailpit at http://localhost:8025 for local
testing). Hiring emails (shortlist, rejection, round invitations, outcomes, interview, GD) are a **database row
first** (queued), then sent by a loop with retries, so nothing is sent twice. Account emails (signup / login codes,
team invites) and the "thanks for applying" email use a small **in-memory** queue inside the API process: if the API
stops, those queued emails are lost.

**No event bus.** Modules do not send events to each other. A module finds new work by **checking the database on a
timer** (for example the ATS loop looks for applications without a score every `ATS_POLL_MS`). Events like
`application.received` are only written to the log.

**Feature flags.** `FEATURE_<NAME>=off` turns a whole module's routes off (`apps/api/src/common/feature-flags.ts`).
Default is on.

## 5. Repository map

```
hireflow/
├── apps/
│   ├── api/                 NestJS backend: all business logic, database, background loops   (section 7–9)
│   │   ├── prisma/          database schema (one .prisma file per area) + migrations
│   │   ├── src/main.ts      starts the server
│   │   ├── src/app.module.ts  lists every module
│   │   ├── src/common/      shared helpers (env, errors, flags, Prisma, public links, encryption)
│   │   ├── src/modules/     one folder per feature module (auth, jobs, ats, round-engine, dsa, …)
│   │   └── tests/           Vitest tests (mostly in-memory fakes, some against a real DB)
│   ├── hr-dashboard/        Next.js app for HR                                                 (section 10)
│   ├── careers-site/        Next.js public site for students: job pages, apply, screening, tests, interview, GD (section 11)
│   ├── candidate-app/       Next.js app for DSA / SQL coding rounds                            (section 11)
│   └── admin-console/       placeholder                                                        (section 11)
├── packages/
│   ├── contracts/           shared zod schemas + pure rules, used by the API AND the front-ends (section 10)
│   └── rounds, ui, config, email-templates, proctor-sdk, round-sdk   EMPTY placeholders (README only) (section 10)
├── workers/
│   ├── ai-worker/           Python FastAPI: every AI call (Gemini)                             (section 12)
│   ├── gd-agent/            Python: group-discussion recording, transcription, AI moderator    (section 12)
│   └── workflows/, activities-ts/   placeholders for later (Temporal)                          (section 12)
├── prompts/                 versioned LLM prompts (YAML) used by the AI worker                 (section 12)
├── scripts/                 start-public.ps1 and helpers                                       (section 12)
├── infra/                   docker-compose, Piston, k8s / terraform placeholders                (section 12)
├── docs/                    feature documents                                                  (section 12)
├── seed/                    sample data (resumes)
├── .env.example             every setting, with comments (copy to .env)
├── package.json, turbo.json, pnpm-workspace.yaml   monorepo setup
```

## 6. Code patterns used everywhere

Once you know these patterns, every API module reads the same way.

**A typical API module** (`apps/api/src/modules/<name>/`):

| File | Role |
|---|---|
| `index.ts` | The module's public entry. Other modules import **only** from here. |
| `<name>.module.ts` | NestJS wiring: imports other modules, lists controllers and providers, reads config from env. |
| `*.controller.ts` | HTTP routes. `hr-*.controller.ts` = HR routes (login + permission). `public-*.controller.ts` = candidate routes (token + rate limit). |
| `<name>.service.ts` | Business logic. |
| `<name>.store.ts` | All database access. An **abstract class** + a `Prisma…Store` implementation. Tests swap in an in-memory store. |
| `<name>.worker.ts` | The background loop (see table in part 4). |
| `flow.ts`, `emails.ts`, `link.ts` | Pure rules (no I/O), email text, link/token helpers. Easy to unit-test. |
| `README.md` | Short module notes. |

**Other patterns**

- **Shared contracts.** Request/response shapes and pure rules (round types, pass rules, pipeline status) live in
  `packages/contracts` as **zod** schemas. The API validates bodies with them and the front-ends use the same types,
  so they never disagree.
- **Multi-tenant.** Every row has a `tenant_id` (the company). Every query is scoped to the logged-in user's company;
  another company's ids answer 404.
- **Audit log.** Important actions are written with `AuditService.record()` (`modules/audit`), ids only, never
  personal content.
- **Idempotent and safe to retry.** Unique keys + conditional updates mean "do it again" never sends a second email
  or creates a second result.
- **IDs** are UUID v7 (`common/uuidv7.ts`), time-ordered.
- **Errors** are JSON `{ statusCode, code, message }` (`common/api-error.ts`), e.g. `409 ROUNDS_LOCKED`.
- **Database schema** is split into one file per area in `apps/api/prisma/schema/*.prisma`. Migrations are
  **additive only** (new tables/columns), because the team shares one database.

**Run it** (details in `docs/TEAM_GUIDE.md`):

```bash
pnpm install
pnpm start:public                 # everything, with public tunnels (Windows)
# or by hand:
pnpm infra:up                     # Docker: Postgres, Redis, Mailpit
pnpm dev                          # API :4000, HR :3000, careers :3300, candidate :3001
cd workers/ai-worker && uv run python -m ai_worker   # AI worker :8000
```

---

## apps/api — core, database and first-half modules

This section covers the NestJS API root, the shared `common` helpers, the Prisma database, the first six
feature modules (`auth`, `tenants`, `jobs`, `audit`, `publisher`, `applications`), the CLI scripts and every test file.
All paths are relative to the repo root.

---

### `apps/api` (root files and boot)

**What it does**
- This is the NestJS 11 backend. It runs on Express and listens on `API_PORT` (default `4000`).
- Every route starts with `/api/v1` (global prefix set in `main.ts`).
- There is no global `ValidationPipe`. Each service checks its request body with a Zod schema from `@hireflow/contracts` (see `parseBody` in `common/api-error.ts`).
- There is no `vitest.config.*` file. Vitest runs with its default settings and finds every `*.test.ts` under `apps/api/tests`.

**How it connects**
- `main.ts` loads the repo-root `.env` first (`loadRootEnv`), so `DATABASE_URL`, `AI_WORKER_URL` etc. work from any folder.
- CORS: only origins in `CORS_ORIGINS` (comma list). When it is empty, only `http://localhost:<any port>` / `127.0.0.1` is allowed. Credentials (cookies) are on.
- `trust proxy` comes from `TRUST_PROXY` (default `loopback`). This decides what `req.ip` is, which the rate limits use.
- `app.module.ts` imports all feature modules. `SqlModule` is imported at the top of the file but is NOT in the `imports` list; it is loaded through `DsaModule` instead.
- `AuthModule` registers a global guard. Every route must declare its access, or it answers 403.

**Main flow (start-up)**
1. `pnpm install` runs `postinstall` → `prisma generate` (builds the Prisma client).
2. `main.ts` loads `.env`, creates the Nest app, sets `trust proxy`, the `api/v1` prefix and CORS.
3. Nest builds `AppModule`: `PrismaModule` (global DB client), `AuditModule` (global), `AuthModule` (global guard), then the feature modules.
4. Some services start background work on boot (e.g. publisher "refresh apply links", ATS poller — the latter is in another section).
5. The app listens on `API_PORT`.

**HTTP endpoints**

| Method | Path | Who | What |
|---|---|---|---|
| GET | `/api/v1/health` | public | Returns `{ status: 'ok', service: 'api' }` |

**Files**

| File | What it does |
|---|---|
| `apps/api/package.json` | Scripts: `dev` (`nest start --watch`), `build`, `start`, `test` (vitest), `db:migrate` / `db:deploy` / `db:studio`, CLI scripts (`admin:create`, `company:assign-admin`, `resumes:to-storage`, `sql:seed`), `test:db` (runs only `multi-company.e2e.test.ts`). `lint` is only an `echo`. Deps: Nest 11, Prisma 6.19, zod, nodemailer, AWS S3 SDK, LiveKit SDK |
| `apps/api/nest-cli.json` | Nest CLI config, source root is `src` |
| `apps/api/tsconfig.json` | TypeScript: Node16 modules, ES2022, strict, decorators on, `src` → `dist` |
| `apps/api/prisma.config.ts` | Prisma CLI config: loads repo-root `.env`, schema is the whole `prisma/schema` folder (multi-file schema) |
| `apps/api/src/main.ts` | Boots the app: loads `.env`, sets trust proxy, `api/v1` prefix, CORS, listens on `API_PORT` |
| `apps/api/src/app.module.ts` | Root module: lists every feature module (one line per intern) and the health controller |
| `apps/api/src/health.controller.ts` | `GET /health`, marked `@Public()` |

---

### `apps/api/src/common`

**What it does**
- Small shared helpers used by all modules: error format, env loading, feature flags, Prisma client, encryption, UUIDv7, tunnel-link checks.
- The README says this folder will hold "guards, filters, event bus/outbox (M30)". Today there is no event bus or outbox here; events are only logged by the modules.

**How it connects**
- `PrismaService` is injected everywhere a module touches the DB.
- `featureFlag(name)` guards are used by `publisher`, `applications`, `ats`, `rounds`, `screening`, `dsa`, `sql`, `round_engine`, `group_discussion`, `ai_interview` controllers. `auth`, `tenants` and `jobs` have NO feature flag.
- `SecretBox` encrypts LinkedIn tokens (publisher). `public-link.ts` is used by modules that email links (screening, AI interview, etc.).

**Files**

| File | What it does |
|---|---|
| `apps/api/src/common/README.md` | Owner info and folder rules (template text; mentions an event bus that does not exist yet) |
| `apps/api/src/common/api-error.ts` | `apiError()` builds the standard error body `{ statusCode, code, message }`; `parseBody()` validates with Zod and adds `issues[]` on 400 |
| `apps/api/src/common/env.ts` | `loadRootEnv()` walks up to the folder with `pnpm-workspace.yaml` and loads its `.env`; `trustProxySetting()` turns `TRUST_PROXY` text into Express's value |
| `apps/api/src/common/feature-flags.ts` | `FEATURE_<NAME>=off/false/0` turns a module's routes off; guard answers `404 FEATURE_DISABLED`. Default is ON |
| `apps/api/src/common/prisma.service.ts` | One shared `PrismaClient` for the whole API (global `PrismaModule`), disconnects on shutdown |
| `apps/api/src/common/public-link.ts` | Detects temporary tunnel URLs (trycloudflare, ngrok) and checks (cached 60 s) that the link still opens before an email with that link is sent |
| `apps/api/src/common/secret-box.ts` | AES-256-GCM encrypt/decrypt with `ENCRYPTION_KEY` (32 bytes, base64 or hex). Format `v1.<iv>.<tag>.<data>` |
| `apps/api/src/common/uuidv7.ts` | Makes time-ordered UUIDv7 ids in Node (PostgreSQL 17 has no generator) |

---

### `apps/api/prisma`

**What it does**
- One PostgreSQL database for the whole API. The schema is split into one `.prisma` file per area (Prisma multi-file schema, needs Prisma ≥ 6.7).
- Almost every table has `tenant_id` (the company). Code always filters by it. There is NO Row Level Security in the database.
- Ids are UUIDv7 (`@default(uuid(7))` or `uuidv7()` in code).
- Tables use snake_case names (`@@map`), models use PascalCase.

**How it connects**
- `pnpm --filter api db:migrate` (dev) creates a migration; `db:deploy` applies them. The CLI reads `DATABASE_URL` from the root `.env`.
- The migrations also create a SQL view `parser_resume_inbox` and a role `hireflow_resume_reader` for the resume parser (they are not in the Prisma models).

**Schema files — models (tables)**

| Schema file | Model → table | What it stores |
|---|---|---|
| `base.prisma` | — | Generator (`prisma-client-js`) and datasource (`postgresql`, `DATABASE_URL`) only |
| `auth.prisma` | `User` → `users` | Staff account: email (lower case), name, scrypt password hash, optional `google_sub`, `is_platform_admin`, `is_active` |
| | `Session` → `sessions` | Server-side login session: SHA-256 of the cookie token, user, current company (`tenant_id`), expiry |
| | `Membership` → `memberships` | User ↔ company link with role (`COMPANY_ADMIN` / `HR_MEMBER`) and status (`ACTIVE` / `DISABLED`) |
| | `Invitation` → `invitations` | One-time "join company" link (hash only), email, role, expiry, accepted/revoked time |
| | `PasswordReset` → `password_resets` | One-time "forgot password" link (hash only) |
| | `LoginChallenge` → `login_challenges` | Login 2FA: hashed challenge token + hashed 6-digit code, attempts |
| | `SignupChallenge` → `signup_challenges` | Sign-up email check: form details (password already hashed) waiting for the code |
| `tenants.prisma` | `Tenant` → `tenants` | A company: name, unique slug, status `ACTIVE` / `SUSPENDED` |
| | `CompanyProfile` → `company_profiles` | One per company: industry, description, website, careers URL, size, culture, benefits… (sent to the AI for JDs) |
| `jobs.prisma` | `Job` → `jobs` | A job: `form_data` (JSON job form), status `DRAFT` / `JD_GENERATED` / `APPROVED`, current JD version, openings, deadline |
| | `JdVersion` → `jd_versions` | Every AI generation or HR edit is a new row (never updated): content JSON, company snapshot, model, prompt version |
| | `JobApproval` → `job_approvals` | HR approval of one JD version (approver name + id, "company details verified", note) |
| `audit.prisma` | `AuditLog` → `audit_logs` | Append-only "who did what": tenant, actor, action, subject, metadata |
| `publisher.prisma` | `PublisherJobListing` → `publisher_job_listings` | One row per published job: slug, `LIVE` / `CLOSED`, which approved version, selection process |
| | `JobPost` → `job_posts` | One post per job per channel: status, external id, URL, attempts, last error. Unique key stops double posting |
| | `PublisherChannelSetting` → `publisher_channel_settings` | Per company per channel: enabled flag + non-secret config (e.g. LinkedIn `organizationId`) |
| | `ChannelCredential` → `channel_credentials` | Per company per channel: encrypted OAuth access/refresh tokens, expiry, scopes, LinkedIn account id/name |
| `applications.prisma` | `Candidate` → `candidates` | One person per company, keyed by verified phone |
| | `Application` → `applications` | One candidate → one job: education, score, links, email typed on this application, `jd_version_id`, confirmation-email time. Unique per (job, candidate) and (job, email) |
| | `Resume` → `resumes` | Resume metadata: name, type, size, sha256, scan result (`CLEAN` only), `storage_key` (S3) |
| | `ResumeFile` → `resume_files` | Resume bytes when no S3 is used (local dev / old rows) |
| | `Consent` → `consents` | What the student agreed to, policy version, time (DPDP) |
| | `PhoneOtpChallenge` → `applications_phone_otp_challenges` | Phone OTP: HMAC of the code, attempts, expiry |
| `ats.prisma` | `JobScreeningSettings`, `AtsEvaluation`, `CandidateEmail`, `ScreeningInvite` | HR's private minimum ATS score per job; one ATS score per application; the one shortlist/rejection email; hashed AI-screening link (covered in the ATS section) |
| `screening.prisma` | `ScreeningSession`, `ScreeningTurn`, `ScreeningResult` | AI voice screening: session state, each Q&A transcript, structured summary for HR |
| `rounds.prisma` | `JobRoundConfig`, `JobRound`, `RoundQuestion`, `RoundResult` | HR's assessment rounds per job (locked once students start), questions (incl. `coding_spec`, `sql_spec`), and each student's result |
| `round_engine.prisma` | `RoundInvite`, `RoundAttempt`, `ProctorEvent`, `RoundProtectionIncident`, `RoundOutcomeEmail`, `DsaProctorSession` | Online test invites (hashed token), attempts (server clock, answers, coding state), proctoring events/incidents, outcome emails, DSA proctoring |
| `pipeline.prisma` | `JobPipelineSettings`, `RoundRubric`, `PiCall` | Stage-2 pass rule per job, GD rubric per round, personal-interview call log |
| `ai_interview.prisma` | `AiInterviewInvite`, `AiInterviewSession`, `AiInterviewViolation`, `AiInterviewProtectionSetting`, `AiInterviewTurn`, `AiInterviewEvent`, `AiInterviewReport` | AI technical voice interview: invite, session, protection incidents and settings, Q&A turns, device events, final report |
| `group_discussion.prisma` | `GdEvent`, `GdBatch`, `GdParticipant`, `GdRecording`, `GdParticipantEvent`, `GdPrivateTalk`, `GdHrActivity`, `GdTranscript`, `GdEvaluation`, `GdEmail` | AI group discussion: drives, groups (LiveKit rooms), participants, audio segments, events, HR actions, transcripts, AI evaluation, email log |
| `sql_bank.prisma` | `SqlQuestion` → `sql_questions` | A company's SQL question bank (schema, datasets, reference query, computed problem) |

**Migrations** (folder `apps/api/prisma/schema/migrations/`, applied in name order)

| Migration folder | What it adds |
|---|---|
| `20261012113526_company_profile_and_jobs` | First tables: `tenants`, `company_profiles`, `jobs`, `jd_versions`, `job_approvals` + job enums |
| `20261013070835_m06_m09_publisher_applications` | Publisher and applications tables (candidates, applications, resumes, resume_files, consents, OTP challenges, listings, job_posts, channel settings) + temporary copy tables; view `parser_resume_inbox` and role `hireflow_resume_reader` |
| `20261013090000_multi_company_auth` | Login and multi-company: `users`, `sessions`, `memberships`, `invitations`, `audit_logs`; `tenants.status`; `created_by_id` / `approved_by_id` columns; composite `(job_id, tenant_id)` foreign keys |
| `20261013105133_m06_m09_audit_events` | Temporary `applications_audit_events` and `publisher_audit_events` tables (dropped two migrations later) |
| `20261013120000_company_careers_url` | `company_profiles.careers_url` |
| `20261014120000_m06_m09_use_core_tables` | Drops the temporary copy/audit tables; publisher/applications now point at `jd_versions`, `job_approvals`, `audit_logs`; view re-created |
| `20261015090000_m07_channel_credentials` | `channel_credentials` (encrypted LinkedIn tokens) |
| `20261019110000_applications_bugfixes` | `applications.email` + backfill + lower-case CHECK + unique (job, email) |
| `20261019130000_password_resets` | `password_resets` |
| `20261019140000_resume_cloud_storage` | `resumes.storage_key`; view now also returns `storage_key` |
| `20261019150000_channel_account` | `channel_credentials.account_id` / `account_name` |
| `20261022120000_login_challenges` | `login_challenges` (login 2FA) |
| `20261022130000_signup_challenges` | `signup_challenges` (sign-up email code) |
| `20261022140000_user_google_sub` | `users.google_sub` (Continue with Google) |
| `20261022150000_application_project_url` | `applications.project_url` |
| `20261026100741_ats_screening` | ATS: `job_screening_settings`, `ats_evaluations`, `candidate_emails`, `screening_invites` |
| `20261029075341_ai_screening` | AI screening: `screening_sessions`, `screening_turns`, `screening_results` |
| `20261030100000_assessment_rounds` | Rounds: `job_round_configs`, `job_rounds`, `round_questions`, `round_results` |
| `20261030120000_round_engine` | Online tests: `round_invites`, `round_attempts`, `round_proctor_events`, `round_outcome_emails` |
| `20261102120000_dsa_round` | `round_attempts.coding_state` + `dsa_proctor_sessions` |
| `20261102130000_group_discussion` | GD base: `gd_events`, `gd_batches`, `gd_participants` |
| `20261102140000_group_discussion_live` | GD live: recordings, participant events, private talks, HR activity, transcripts, evaluations, emails + new columns |
| `20261103100000_screening_invite_link_base` | `screening_invites.link_base` (re-send when tunnel URL died) |
| `20261103120000_dsa_hr_problems` | `round_questions.coding_spec` (HR's own DSA problem) |
| `20261103140000_gd_session_audio` | `gd_batches` columns for one whole-session audio file |
| `20261103150000_round_type_ai_interview` | Enum value `round_type.AI_INTERVIEW` |
| `20261103170000_ai_technical_interview` | AI interview: invites, sessions, turns, events, reports |
| `20261104090000_ai_interview_protection` | AI interview protection: `ai_interview_violations`, `ai_interview_protection_settings`, status `TERMINATED` |
| `20261104100000_sql_round` | `sql_questions` + `round_questions.sql_spec` |
| `20261104120000_ai_interview_face_count` | Violation type `MULTIPLE_FACES` + `multiple_faces_policy` setting |
| `20261104140000_assessment_protection` | Online test protection: `round_protection_incidents`, attempt status `TERMINATED`, integrity columns |
| `20261104160000_pipeline_rules` | `job_pipeline_settings`, `round_rubrics`, `pi_calls` + GD rubric columns |
| `20261104180000_round_skip` | `job_rounds` skip columns (HR can skip a round for a job) |

**Files**

| File | What it does |
|---|---|
| `apps/api/prisma/README.md` | Rules: one schema file per area, `tenant_id` everywhere, UUIDv7, how to handle migration conflicts |
| `apps/api/prisma/schema/base.prisma` | Generator + datasource |
| `apps/api/prisma/schema/auth.prisma` | Users, sessions, memberships, invitations, password resets, login/sign-up challenges |
| `apps/api/prisma/schema/tenants.prisma` | Tenants and company profiles |
| `apps/api/prisma/schema/jobs.prisma` | Jobs, JD versions, approvals |
| `apps/api/prisma/schema/audit.prisma` | Audit log |
| `apps/api/prisma/schema/publisher.prisma` | Listings, posts, channel settings, channel credentials |
| `apps/api/prisma/schema/applications.prisma` | Candidates, applications, resumes, resume files, consents, phone OTPs |
| `apps/api/prisma/schema/ats.prisma` | ATS settings, evaluations, decision emails, screening invites |
| `apps/api/prisma/schema/screening.prisma` | AI screening sessions, turns, results |
| `apps/api/prisma/schema/rounds.prisma` | Round config, rounds, questions, results |
| `apps/api/prisma/schema/round_engine.prisma` | Online test invites, attempts, proctoring, outcome emails, DSA proctoring |
| `apps/api/prisma/schema/pipeline.prisma` | Pipeline settings, rubrics, PI calls |
| `apps/api/prisma/schema/ai_interview.prisma` | AI interview tables |
| `apps/api/prisma/schema/group_discussion.prisma` | Group discussion tables |
| `apps/api/prisma/schema/sql_bank.prisma` | SQL question bank |
| `apps/api/prisma/schema/migrations/*/migration.sql` | 33 migrations, see table above |
| `apps/api/prisma/schema/migrations/migration_lock.toml` | Prisma lock file (provider = postgresql) |

---

### `apps/api/src/modules/auth`

**What it does**
- Login and roles for staff (HR people). Students never log in; they use phone OTP in the `applications` module.
- Sessions are server-side: the cookie `hireflow_session` holds a random 256-bit token; the DB holds only its SHA-256.
- It has sign-up, login with optional email 2FA code, sign-up email verification, "Continue with Google", forgot/reset password, company switch, invitations and team management.
- It registers the global `AuthGuard`, which protects every route in the whole API.

**How it connects**
- Tables: `users`, `sessions`, `memberships`, `invitations`, `password_resets`, `login_challenges`, `signup_challenges`; reads `tenants` / `company_profiles` for invitation emails.
- Uses `AuditService` for every important action (`auth.login`, `auth.login_failed`, `invitation.created`, …).
- Sends email through the `applications` module's mailer (`mailerFromEnv`, SMTP or Gmail). It imports leaf files directly to avoid an import cycle.
- Other modules use its decorators: `@Public()`, `@Authenticated()`, `@PlatformAdminOnly()`, `@RequirePermission('…')`. `tenants` uses `SessionsService` and `TeamService`.
- Env: `SESSION_TTL_HOURS` (12), `INVITATION_TTL_HOURS` (168), `HR_DASHBOARD_URL`, `COOKIE_SECURE`, `LOGIN_2FA`, `SIGNUP_VERIFY_EMAIL`, `LOGIN_CODE_TTL_MINUTES` (10), `LOGIN_CODE_MAX_ATTEMPTS` (5), `LOGIN_MAX_ATTEMPTS`, `SIGNUP_MAX_PER_IP`, `PASSWORD_RESET_TTL_MINUTES` (60), `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_REDIRECT_URI`, `GOOGLE_EXTRA_DASHBOARD_URLS`.

**Main flow**

How each request is checked (`auth.guard.ts`):
1. If the method is not GET/HEAD/OPTIONS, the header `X-Requested-With: hireflow` must be there, else `403 CSRF_CHECK_FAILED`. (This also applies to public student endpoints.)
2. The guard reads the access decorator on the route. No decorator → `403 FORBIDDEN` (fail closed).
3. `@Public` → allowed. Otherwise the session cookie is looked up by hash. No valid session → `401 UNAUTHENTICATED`.
4. The session's company is re-checked on every request: ACTIVE membership, ACTIVE company, and the role must have the permission (`permissions.ts`).
5. The result is stored in `req.auth`. `TenantContext` (tenants module) reads the company id only from here.

Sign-up and new company:
1. `POST /auth/signup` with name, email, password (rate limit 10 per IP per hour).
2. If `SIGNUP_VERIFY_EMAIL` is on (default ON only when `NODE_ENV=production`): details are saved in `signup_challenges`, a 6-digit code is emailed, the answer is `{ codeRequired, challenge, emailHint, expiresAt }`. Then `POST /auth/signup/verify` creates the user.
3. If off: the user is created at once and gets a session cookie.
4. The new user has no company. The dashboard then calls `POST /companies` (tenants module), which makes them `COMPANY_ADMIN`.

Login:
1. `POST /auth/login` with email + password. scrypt always runs (dummy hash for unknown emails), so timing does not leak.
2. Wrong → `401 INVALID_CREDENTIALS` and audit `auth.login_failed`.
3. If `LOGIN_2FA` is on (default ON only in production): a code is emailed (`login_challenges`). Outside production the code is also printed to the API console. Then `POST /auth/login/verify`.
4. A session row is created (in the user's first active company) and the cookie is set.

Google:
1. `GET /auth/google/start` → redirect to Google with state + PKCE (kept in a 10-minute cookie).
2. `GET /auth/google/callback`: known `google_sub` → login; same verified email → link and login; else create a new user with a random password. No email code is asked.
3. Redirect to the dashboard (`/jobs`, `/admin` or `/register-company`).

Invitation:
1. Company Admin calls `POST /company/invitations` → older pending invitations for that email are revoked, a new one is saved, the link is emailed in the background AND returned once in the response.
2. Invitee opens `/invite/:token` on the dashboard → `GET /auth/invitations/:token` (preview).
3. `POST /auth/invitations/accept`: new email → creates the account; existing account → must give its own password. Membership is created (or re-activated) and the user is logged in.

**HTTP endpoints**

| Method | Path | Who | What |
|---|---|---|---|
| GET | `/api/v1/auth/providers` | public | `{ google: true/false }` |
| GET | `/api/v1/auth/google/start?next=` | public (browser) | Redirect to Google |
| GET | `/api/v1/auth/google/callback` | public (browser) | Finish Google login, set cookie, redirect |
| POST | `/api/v1/auth/login` | public | Password login; may answer "code required" |
| POST | `/api/v1/auth/login/verify` | public | 2FA code → session |
| POST | `/api/v1/auth/signup` | public | New account; may answer "code required" |
| POST | `/api/v1/auth/signup/verify` | public | Sign-up code → account + session |
| POST | `/api/v1/auth/logout` | public | Delete session, clear cookie |
| POST | `/api/v1/auth/password/forgot` | public | Always 204; emails a reset link if the account exists |
| POST | `/api/v1/auth/password/reset` | public | `{ token, password }`; logs out all devices |
| GET | `/api/v1/auth/me` | logged in | User, current company + role, all companies |
| POST | `/api/v1/auth/company` | logged in | Switch current company |
| GET | `/api/v1/auth/invitations/:token` | public | Invitation preview |
| POST | `/api/v1/auth/invitations/accept` | public | Accept invitation and log in |
| GET | `/api/v1/company/members` | Company Admin (`team.manage`) | Members + pending invitations |
| POST | `/api/v1/company/invitations` | Company Admin | Create invitation, returns `inviteUrl` |
| DELETE | `/api/v1/company/invitations/:id` | Company Admin | Revoke invitation |
| PATCH | `/api/v1/company/members/:userId` | Company Admin | Change role / status; last active admin cannot be removed |

**Files**

| File | What it does |
|---|---|
| `apps/api/src/modules/auth/README.md` | Roles, request checks, endpoint list, env switches (missing the password endpoints; says invitations are not emailed, but they are) |
| `apps/api/src/modules/auth/access.ts` | Access decorators (`@Public`, `@Authenticated`, `@PlatformAdminOnly`, `@RequirePermission`) and `AuthInfo` / `AuthRequest` types |
| `apps/api/src/modules/auth/auth.controller.ts` | `/auth/*` routes; sets/clears the session cookie; Google redirects |
| `apps/api/src/modules/auth/auth.guard.ts` | Global guard: CSRF header, access check, session lookup, company + permission check |
| `apps/api/src/modules/auth/auth.module.ts` | Wires services, Google config, mailer (`AUTH_MAILER`) and the global `APP_GUARD` |
| `apps/api/src/modules/auth/auth.service.ts` | Login, sign-up, logout, company switch, invitation preview/accept; in-memory rate limits |
| `apps/api/src/modules/auth/cookies.ts` | Read the cookie header; build the `HttpOnly; SameSite=Lax; Secure` session cookie |
| `apps/api/src/modules/auth/email-code.ts` | Shared 6-digit code helpers: make, hash (bound to challenge token), mask email, on/off switch |
| `apps/api/src/modules/auth/google-auth.service.ts` | "Continue with Google": find/link/create the user, start session, choose landing page |
| `apps/api/src/modules/auth/google-oauth.ts` | Google OAuth client: config from env, authorize URL with PKCE, code exchange, ID-token checks, state cookie, safe `next` path |
| `apps/api/src/modules/auth/index.ts` | Public exports of the module (decorators, services, `PERMISSIONS`, `hashPassword`) |
| `apps/api/src/modules/auth/invitation-email.ts` | Builds the "Join <company> on HireFlow" email (text + HTML) |
| `apps/api/src/modules/auth/login-code-email.ts` | Builds the login-code email and the sign-up-code email |
| `apps/api/src/modules/auth/login-code.service.ts` | Login 2FA: create challenge, email code, verify code (5 tries, single use) |
| `apps/api/src/modules/auth/password-reset-email.ts` | Builds the "reset your password" email |
| `apps/api/src/modules/auth/password-reset.service.ts` | Forgot/reset password; defines `AUTH_MAILER` token; ends all sessions after reset |
| `apps/api/src/modules/auth/password.ts` | scrypt hash/verify (N=2^15) with Node crypto; dummy hash for timing |
| `apps/api/src/modules/auth/permissions.ts` | Permission matrix: `company.read/update`, `jobs.read/write`, `jd.generate/edit/approve`, `team.manage` |
| `apps/api/src/modules/auth/rate-limit.ts` | Simple fixed-window in-memory limiter (429 `TOO_MANY_ATTEMPTS`), per API process |
| `apps/api/src/modules/auth/sessions.service.ts` | Create / resolve / destroy sessions, switch company, build the `/auth/me` answer |
| `apps/api/src/modules/auth/signup-code.service.ts` | Sign-up email verification: store pending sign-up, email code, verify |
| `apps/api/src/modules/auth/team.controller.ts` | `/company/*` routes for team management (Company Admin only) |
| `apps/api/src/modules/auth/team.service.ts` | List team, create/revoke invitations (emailed in background), update members (serializable transaction) |
| `apps/api/src/modules/auth/tokens.ts` | Random tokens, SHA-256 hashing, TTLs, `HR_DASHBOARD_URL` |

---

### `apps/api/src/modules/tenants`

**What it does**
- A tenant is a company. This module lets a logged-in user register their own company, lets HR read/save the company profile, and lets the Platform Admin list and suspend companies.
- It provides `TenantContext`: the one place where services ask "which company is this request for, and who is acting?".

**How it connects**
- Tables: `tenants`, `company_profiles`, `memberships` (on register).
- Uses `SessionsService` and `TeamService` from `auth`, and `AuditService`.
- `CompanyProfileService` and `TenantContext` are used by `jobs`, `publisher`, `applications` and many later modules.
- `RequestTenantContext` is request-scoped. It reads the company only from `req.auth` (set by `AuthGuard`), never from URL/body/headers.

**Main flow (company onboarding — "step 1" of the JD flow)**
1. A new user (no company) calls `POST /companies` with the company profile.
2. The service makes a slug from the name (adds a random suffix if taken, up to 5 tries).
3. In one statement it creates the tenant, its profile and a `COMPANY_ADMIN` membership for the caller.
4. Audit `company.registered`; the session is switched to the new company; the new `Session` is returned.
5. Later the admin can update the profile with `PUT /company-profile` (also renames the tenant).

**HTTP endpoints**

| Method | Path | Who | What |
|---|---|---|---|
| POST | `/api/v1/companies` | any logged-in user | Register own company, become Company Admin |
| GET | `/api/v1/company-profile` | `company.read` (admin + HR) | Own company profile, or `404 COMPANY_PROFILE_NOT_FOUND` |
| PUT | `/api/v1/company-profile` | `company.update` (Company Admin) | Create or replace the profile |
| GET | `/api/v1/admin/companies` | Platform Admin | All companies with member/job counts |
| GET | `/api/v1/admin/companies/:id` | Platform Admin | One company + members + pending invitations |
| PATCH | `/api/v1/admin/companies/:id` | Platform Admin | `{ status: ACTIVE / SUSPENDED }` |

**Files**

| File | What it does |
|---|---|
| `apps/api/src/modules/tenants/README.md` | Endpoints and notes (links to `docs/JD_WORKFLOW.md`, which does not exist in the repo) |
| `apps/api/src/modules/tenants/companies.controller.ts` | Two controllers: `POST /companies` (self-service) and `/admin/companies` (Platform Admin) |
| `apps/api/src/modules/tenants/companies.service.ts` | Register company (unique slug), list/get/suspend companies, audit |
| `apps/api/src/modules/tenants/company-profile.controller.ts` | `GET` / `PUT /company-profile` |
| `apps/api/src/modules/tenants/company-profile.repository.ts` | Abstract repo + Prisma version: upsert profile and copy its name to `tenants.name` |
| `apps/api/src/modules/tenants/company-profile.service.ts` | Validate and save profile; `require()` throws `409 COMPANY_PROFILE_REQUIRED` for the jobs module |
| `apps/api/src/modules/tenants/index.ts` | Exports `TenantsModule`, `CompanyProfileService`, `TenantContext`, `Actor` |
| `apps/api/src/modules/tenants/tenant-context.ts` | `TenantContext` (abstract) and request-scoped `RequestTenantContext` |
| `apps/api/src/modules/tenants/tenants.module.ts` | Wires controllers, services and the two abstract providers |

---

### `apps/api/src/modules/jobs`

**What it does**
- HR creates a job (the job form), asks the AI to write the JD, edits it, and approves it.
- Each AI generation and each HR edit is a new immutable row in `jd_versions`. The job points to its "current" version.
- Status: `DRAFT` → `JD_GENERATED` → `APPROVED` (locked). Changing the form sets it back to `DRAFT`.
- HR can also delete a job for good (unpublish + erase applications + delete job).

**How it connects**
- Tables: `jobs`, `jd_versions`, `job_approvals` (via `JobsRepository`).
- Calls the Python AI worker: `POST {AI_WORKER_URL}/v1/jd/generate` (default `http://localhost:8000`, timeout `AI_WORKER_TIMEOUT_MS` = 90 s). It reads the model and prompt version from the `x-llm-model` / `x-prompt-version` response headers.
- Uses `CompanyProfileService` (profile must exist), `TenantContext`, `AuditService`.
- Optional: `ScreeningSettingsService` (ATS module) saves HR's private minimum ATS score sent with `POST /jobs`; `PublisherService.unpublishIfListed` and `JobApplicationsEraser` are used on delete.
- No feature flag. No events are emitted (there is a `TODO(M29)` for `jd.approved`).

**Main flow (job → JD → approve)**
1. `POST /jobs` with the job form (+ optional `screening`). Needs a saved company profile, else `409 COMPANY_PROFILE_REQUIRED`. Job is saved as `DRAFT`.
2. `POST /jobs/:id/jd/generate`: the job form and the saved company profile are sent to the AI worker. The answer is checked with `jdContentSchema`. The "Company Website" link is always taken from the profile, not from the AI. A new `AI` version becomes current (`JD_GENERATED`).
3. `PATCH /jobs/:id/jd` with `{ baseVersionId, content }`: HR edit becomes a new `HR_EDIT` version. If someone else wrote a newer version first → `409 STALE_JD_VERSION` (compare-and-set in the repository).
4. `POST /jobs/:id/approve` with `{ jdVersionId, companyDetailsVerified: true }`: the current version must pass `editedJdSchema` (else `422 JD_INCOMPLETE`). An approval row is written and the job is locked (`APPROVED`).
5. The job is now visible to the `publisher` module (it reads `job_approvals`).

**HTTP endpoints**

| Method | Path | Who | What |
|---|---|---|---|
| POST | `/api/v1/jobs` | `jobs.write` | Create DRAFT job |
| GET | `/api/v1/jobs` | `jobs.read` | List jobs, newest first |
| GET | `/api/v1/jobs/:id` | `jobs.read` | Job + form + all JD versions + approval |
| PATCH | `/api/v1/jobs/:id` | `jobs.write` | Change job form (not after approval) |
| DELETE | `/api/v1/jobs/:id` | `jobs.write` | Delete job for good (any status), with posts and applications |
| POST | `/api/v1/jobs/:id/jd/generate` | `jd.generate` | AI writes a new JD version |
| PATCH | `/api/v1/jobs/:id/jd` | `jd.edit` | HR edit → new version |
| POST | `/api/v1/jobs/:id/approve` | `jd.approve` | Approve current version, lock job |

**Files**

| File | What it does |
|---|---|
| `apps/api/src/modules/jobs/README.md` | Status rules, endpoints, error codes (does not list `DELETE /jobs/:id`; links to missing `docs/JD_WORKFLOW.md`) |
| `apps/api/src/modules/jobs/ai-worker.client.ts` | HTTP client for `/v1/jd/generate`; maps worker errors to safe API errors (`AI_WORKER_TIMEOUT`, `LLM_*` pass-through, `retryAfterSeconds`) |
| `apps/api/src/modules/jobs/index.ts` | Exports `JobsModule` only |
| `apps/api/src/modules/jobs/jobs.controller.ts` | `/jobs` routes with permissions |
| `apps/api/src/modules/jobs/jobs.module.ts` | Imports tenants, ATS settings, publisher, applications; wires repo and AI client |
| `apps/api/src/modules/jobs/jobs.repository.ts` | Prisma access: create, list, get, update form, append version (compare-and-set), approve, delete |
| `apps/api/src/modules/jobs/jobs.service.ts` | Business logic of steps 2-4 and delete; audit `job.created`, `jd.generated`, `jd.edited`, `job.approved`, `job.deleted` |

---

### `apps/api/src/modules/audit`

**What it does**
- One small global service: `AuditService.record()` writes one row to `audit_logs`.
- If the write fails, it only logs an error. It never undoes the action.
- There is no API to read the audit log yet.

**How it connects**
- Global module, so any service can inject `AuditService`. Used by auth, tenants, jobs, publisher, applications and later modules.
- Rule in the code comment: never store passwords, tokens or JD text in metadata.

**Files**

| File | What it does |
|---|---|
| `apps/api/src/modules/audit/README.md` | Short description; list of audited actions; "not yet: read API, AI decision audit, retention" |
| `apps/api/src/modules/audit/audit.service.ts` | `AuditService` (abstract), `PrismaAuditService`, global `AuditModule` |
| `apps/api/src/modules/audit/index.ts` | Exports `AuditModule`, `AuditService`, `AuditEntry` |

---

### `apps/api/src/modules/publisher`

**What it does**
- Takes an **approved** JD and makes it public: a job page on the careers site, plus a post on LinkedIn.
- One post per job per channel. The unique key on `job_posts` stops double posting, even with parallel requests.
- HR can preview, publish, retry, close, reopen, unpublish ("Delete" on the publish page) and refresh apply links.
- Live channels: `careers` (our careers site, always on) and `linkedin`. Naukri, Indeed, Instagram, Facebook, X, WhatsApp are "coming soon" only.

**How it connects**
- Reads (never writes) `jobs`, `jd_versions`, `job_approvals`, `tenants`, `company_profiles`. Writes `publisher_job_listings`, `job_posts`, `publisher_channel_settings`, `channel_credentials`.
- Calls the AI worker validator `POST {AI_WORKER_URL}/v1/jd/validate` (15 s). If it is down, publishing is not blocked; a warning is shown.
- Before publish it asks `ScreeningSettingsService.isSet` (minimum ATS score must be set, when `FEATURE_ATS` is on) and `RoundsConfigService.publicSteps` (rounds must be complete; the round names become the "Selection process", when `FEATURE_ROUNDS` is on).
- LinkedIn REST API (Posts API) with a per-company OAuth token (encrypted with `SecretBox`), or an env fallback token for one tenant.
- Exports `PublicJobsReader` (used by `applications` for the careers site) and `PublisherService` (used by `jobs` on delete).
- Events `job.published` / `job.publish_failed` are only validated and logged (no event bus yet).
- Env: `CAREERS_SITE_URL` (default `http://localhost:3300`), `PUBLISHER_AUTO_REFRESH_LINKS`, `FEATURE_PUBLISHER`, `ENCRYPTION_KEY`, `LINKEDIN_*` (client id/secret, redirect URI, `LINKEDIN_POST_AS`, `LINKEDIN_ACCESS_TOKEN`, `LINKEDIN_FALLBACK_TENANT_ID`, `LINKEDIN_ALLOWED_MEMBER_ID`, `LINKEDIN_API_VERSION`).

**Main flow (publish)**
1. HR opens the Publish page → `GET /publisher/jobs` lists every approved job with its status.
2. `POST /publisher/jobs/:jobId/preview`: loads the approved version, runs basic checks (title, summary, responsibilities, requirements, deadline not passed), the AI validator, ATS-score and rounds checks, and renders the text for each channel. Nothing is saved.
3. `POST /publisher/jobs/:jobId/publish`: same checks; any error → `422 NOT_READY_TO_PUBLISH`. A listing row is saved with slug `<title>-<last 6 hex of job id>` (full id if taken).
4. Every enabled channel runs in parallel: check `blockers` (e.g. no LinkedIn token → `NOT_CONFIGURED`), "claim" the post row (`PUBLISHING`), call the adapter, then mark `PUBLISHED` or `FAILED`. Only "surely not sent" errors are retried (backoff).
5. The careers adapter just records the URL `CAREERS_SITE_URL/<company slug>/jobs/<job slug>`. The careers site then reads the job through `PublicJobsReader`.
6. Every post carries exactly two links: the apply link (`?src=<channel>`) and the company link (website, else careers URL).
7. On API start, posts that still carry an older `CAREERS_SITE_URL` are edited in the background (`refreshLinks`).

LinkedIn connect: `GET /publisher/linkedin/connect` → LinkedIn consent → `GET /publisher/linkedin/callback` saves encrypted tokens → redirect to dashboard `/publish?linkedin=<result>`.

**HTTP endpoints** (all need login; all switched off by `FEATURE_PUBLISHER=off`)

| Method | Path | Who | What |
|---|---|---|---|
| GET | `/api/v1/publisher/channels` | `jobs.read` | Live + coming-soon channels and their state |
| GET | `/api/v1/publisher/jobs` | `jobs.read` | Approved jobs with listing, post status, application count |
| POST | `/api/v1/publisher/jobs/:jobId/preview` | `jobs.read` | Preview posts, errors, warnings |
| POST | `/api/v1/publisher/jobs/:jobId/publish` | `jd.approve` | Publish approved JD to all channels |
| POST | `/api/v1/publisher/jobs/:jobId/retry` | `jd.approve` | Retry failed / not-configured channels |
| POST | `/api/v1/publisher/jobs/:jobId/close` | `jd.approve` | Stop applications |
| POST | `/api/v1/publisher/jobs/:jobId/reopen` | `jd.approve` | Take applications again |
| DELETE | `/api/v1/publisher/jobs/:jobId` | `jd.approve` | Unpublish (remove page + LinkedIn post); job and applications stay |
| POST | `/api/v1/publisher/jobs/:jobId/refresh-links` | `jd.approve` | Edit existing posts to carry the current apply link |
| GET | `/api/v1/publisher/jobs/:jobId/posts` | `jobs.read` | Per-channel status |
| PUT | `/api/v1/publisher/channels/:channel/settings` | `company.update` | Enable/disable channel + non-secret config |
| GET | `/api/v1/publisher/linkedin/connect` | `company.update` | Redirect to LinkedIn consent |
| GET | `/api/v1/publisher/linkedin/callback` | `company.update` | Save LinkedIn token, redirect back |
| DELETE | `/api/v1/publisher/linkedin/connection` | `company.update` | Forget the LinkedIn token |

**Files**

| File | What it does |
|---|---|
| `apps/api/src/modules/publisher/README.md` | Files, endpoints, "add a new platform" steps, known limits |
| `apps/api/src/modules/publisher/channel-credentials.ts` | Save/remove/read encrypted OAuth tokens per company; `null` token = "reconnect needed" |
| `apps/api/src/modules/publisher/channels/careers-site.adapter.ts` | Careers-page "channel": records the job page URL, always ready |
| `apps/api/src/modules/publisher/channels/channel-adapter.ts` | `ChannelAdapter` interface (`render`, `blockers`, `publish`, optional `close` / `remove` / `update`), `ChannelError` |
| `apps/api/src/modules/publisher/channels/index.ts` | Registers live adapters from env; `COMING_SOON_CHANNELS` list |
| `apps/api/src/modules/publisher/channels/linkedin.adapter.ts` | LinkedIn Posts API: create, delete, edit text (3000 char limit), org or member mode, safe retries |
| `apps/api/src/modules/publisher/index.ts` | Exports module, `PublicJobsReader`, `PublisherService`, types |
| `apps/api/src/modules/publisher/jd-checks.ts` | `publisherError()`, basic publish checks, `JdValidatorClient` for `/v1/jd/validate` |
| `apps/api/src/modules/publisher/linkedin-connect.controller.ts` | Connect / callback / disconnect routes (browser redirects) |
| `apps/api/src/modules/publisher/linkedin-connect.service.ts` | Connect flow: state cookie, code exchange, read account, save tokens, audit |
| `apps/api/src/modules/publisher/linkedin-oauth.ts` | LinkedIn OAuth client (authorize URL, token exchange, userinfo) and state cookie helpers |
| `apps/api/src/modules/publisher/post-formatter.ts` | Builds the student-friendly post text (role, location, salary in LPA, who can apply, skills, selection process, deadline); shortening for limited channels |
| `apps/api/src/modules/publisher/public-jobs.reader.ts` | Read-only view for the careers site: live jobs, one job, job titles, public URLs |
| `apps/api/src/modules/publisher/publisher.controller.ts` | HR `/publisher/*` routes |
| `apps/api/src/modules/publisher/publisher.events.ts` | Builds `job.published` / `job.publish_failed` event envelopes and logs them |
| `apps/api/src/modules/publisher/publisher.module.ts` | Wires store, adapters, credentials, OAuth client, validator, config |
| `apps/api/src/modules/publisher/publisher.service.ts` | Core logic: checks, publish in parallel, claim/finish posts, retry, close, reopen, unpublish, refresh links on boot, audit |
| `apps/api/src/modules/publisher/publisher.store.ts` | All DB access, always scoped by `tenant_id`; post "claim" with 5-minute stale takeover |

---

### `apps/api/src/modules/applications`

**What it does**
- Public (no login) API for the careers site: list live jobs, show one job, phone OTP, submit an application with a resume, and the student's "my data" view/delete (DPDP).
- HR side: list applications of a job and download a resume.
- Resume checks: real PDF/DOCX by file content, max 5 MB, virus scan (ClamAV), then stored in S3 (or in the DB in local dev).
- It also owns the shared mailer and email queue that many other modules use.

**How it connects**
- Tables: `candidates`, `applications`, `resumes`, `resume_files`, `consents`, `applications_phone_otp_challenges`; reads `publisher_job_listings` (with `FOR SHARE` lock). Writes `audit_logs` (DPDP actions, resume downloads).
- Uses `PublicJobsReader` (publisher) to find company/job by slug. Reads ATS/screening data for the HR list (`screeningOf`).
- Exports `RESUME_STORAGE`, `MAILER`, `JobApplicationsEraser`, `RateLimiter` for ATS, jobs, screening and others. `auth` imports the mailer files directly.
- External: ClamAV (`CLAMAV_HOST`/`CLAMAV_PORT`), S3 (`RESUME_STORAGE=s3`, `S3_BUCKET`, `S3_REGION`, `S3_ENDPOINT`, `S3_ACCESS_KEY`, `S3_SECRET_KEY`), SMTP (`SMTP_URL` or `GMAIL_USER` + `GMAIL_APP_PASSWORD`, `MAIL_FROM`), Cloudflare Turnstile (`TURNSTILE_SECRET_KEY`).
- Other env: `OTP_PROVIDER` (only `dev` exists), `OTP_SIGNING_SECRET`, `VIRUS_SCANNER` (`clamav` or `dev-eicar-only`), `PUBLIC_RATE_LIMITS`, `FEATURE_APPLICATIONS`.
- New applications are NOT pushed to other modules. `application.received` is only logged. The ATS module finds new applications by polling the DB (`ATS_POLL_MS`).
- The resume parser can read only the view `parser_resume_inbox` through the role `hireflow_resume_reader`.

**Main flow (apply + OTP + resume upload)**
1. Careers site loads the job: `GET /public/:companySlug/jobs/:jobSlug` (approved JD only).
2. Student asks for an OTP: `POST /public/otp/send` (`{ companySlug, phone }`, optional CAPTCHA). Per phone: 30 s gap, 5 per hour, guarded by a Postgres advisory lock. Only an HMAC of the code is stored. With `OTP_PROVIDER=dev` the code comes back in the answer as `devCode` (no SMS provider exists yet).
3. `POST /public/otp/verify` → checks the latest challenge (5 tries) → returns a signed `otpToken` (HMAC, valid 30 minutes, bound to company + phone).
4. `POST /public/applications` (multipart: `companySlug`, `jobSlug`, `application` JSON, `resume` file). The per-IP rate limit runs before the file is read.
5. Service checks: form valid → job is `LIVE` (else `410 JOB_CLOSED`) → `otpToken` matches → file is real PDF/DOCX ≤ 5 MB → virus scan clean (scanner down → `503`, fail closed).
6. Quick duplicate check (same phone or same email for this job → `409 ALREADY_APPLIED`).
7. If S3 is used, the file is uploaded first.
8. One DB transaction (waits up to 30 s for a connection): lock the listing row, check `LIVE` again, dedupe again, upsert candidate by phone, create application, resume record, (resume bytes if no S3), consents. If it fails, the uploaded S3 file is deleted.
9. A "thanks for applying" email is queued in memory (2 at a time, retries temporary errors). On success `confirmation_email_sent_at` is set. Emails to reserved test domains (`example.com`, `.test`, …) are refused.
10. Receipt is returned to the student.

DPDP: `POST /public/my-data` and `/my-data/delete` need a fresh `otpToken`. Delete removes OTPs, resumes (and S3 files), consents, applications and the candidate in one transaction, and cancels queued emails.

**HTTP endpoints**

| Method | Path | Who | What |
|---|---|---|---|
| GET | `/api/v1/public/:companySlug/jobs` | public | Live jobs of a company |
| GET | `/api/v1/public/:companySlug/jobs/:jobSlug` | public | Approved JD for the job page |
| POST | `/api/v1/public/otp/send` | public (rate-limited, CAPTCHA if set) | Send phone OTP |
| POST | `/api/v1/public/otp/verify` | public (rate-limited) | Verify OTP → `otpToken` |
| POST | `/api/v1/public/applications` | public (rate-limited) | Submit application + resume |
| POST | `/api/v1/public/my-data` | public (OTP proof) | See own data |
| POST | `/api/v1/public/my-data/delete` | public (OTP proof) | Erase own data |
| GET | `/api/v1/public/config` | public | `{ captchaRequired }` |
| GET | `/api/v1/applications/jobs/:jobId` | `jobs.read` | All applications of a job (with screening info) |
| GET | `/api/v1/applications/:applicationId/resume` | `jobs.read` | Download resume (attachment, no-store, audited) |

Note: public POSTs still need the `X-Requested-With: hireflow` header (global CSRF check). The HR controller is switched by `FEATURE_PUBLISHER`, not `FEATURE_APPLICATIONS`.

**Files**

| File | What it does |
|---|---|
| `apps/api/src/modules/applications/README.md` | Files, endpoints, parser hand-off, load-test notes, limits (says DPDP audit goes to `applications_audit_events`, but code uses `audit_logs`) |
| `apps/api/src/modules/applications/abuse/captcha.ts` | Cloudflare Turnstile check on OTP send (off when no secret; fails closed when Cloudflare is down) |
| `apps/api/src/modules/applications/abuse/rate-limiter.ts` | Per-IP in-memory limits for all public endpoints (also screening, assessment, GD, interview); `rateLimit()` guard; `PUBLIC_RATE_LIMITS` overrides |
| `apps/api/src/modules/applications/applications.errors.ts` | `applicationsError()` — same error body shape as elsewhere |
| `apps/api/src/modules/applications/applications.module.ts` | Wires services and env-based providers (OTP sender/secret, scanner, storage, mailer, queue, CAPTCHA, limiter) |
| `apps/api/src/modules/applications/applications.service.ts` | Intake: validate, check job/OTP/file/scan, dedupe, upload, save in one transaction, queue email |
| `apps/api/src/modules/applications/email/confirmation-email.ts` | Builds the "thanks for applying" email; `EmailMessage` type; `HF-XXXXXXXX` reference |
| `apps/api/src/modules/applications/email/email-queue.ts` | In-memory background email queue: 2 parallel, retries, cancel by key (lost if API stops) |
| `apps/api/src/modules/applications/email/mailer.ts` | SMTP / Gmail mailer via nodemailer, disabled mailer, safe error reasons, blocks reserved test domains |
| `apps/api/src/modules/applications/hr-applications.controller.ts` | HR routes: list applications, download resume; safe `Content-Disposition` |
| `apps/api/src/modules/applications/hr-applications.service.ts` | HR list (with ATS/screening info) and resume read from S3 or DB; audit `application.resume_downloaded` |
| `apps/api/src/modules/applications/index.ts` | Exports module, storage, mailer, eraser, rate limiter |
| `apps/api/src/modules/applications/job-applications.eraser.ts` | Deletes all applications of a job (used by "Delete job"); removes candidates left with no application |
| `apps/api/src/modules/applications/my-data.service.ts` | DPDP view and erase, proven by OTP token; audit in `audit_logs` |
| `apps/api/src/modules/applications/otp/otp-sender.ts` | `OtpSender` interface; only `DevOtpSender` exists (refused in production) |
| `apps/api/src/modules/applications/otp/otp-token.ts` | Signed phone-proof token (HMAC) and OTP hashing |
| `apps/api/src/modules/applications/otp/otp.service.ts` | Send (limits + advisory lock), verify (5 tries), check proof |
| `apps/api/src/modules/applications/public-careers.controller.ts` | Public `/public/*` routes |
| `apps/api/src/modules/applications/resume/file-check.ts` | Detect PDF/DOCX by magic bytes and ZIP entries, size limit, safe file name |
| `apps/api/src/modules/applications/resume/resume-storage.ts` | S3 storage (key `<tenant>/resumes/<id>.pdf`, SSE-AES256) or DB storage; DB mode refused in production |
| `apps/api/src/modules/applications/resume/virus-scanner.ts` | ClamAV INSTREAM scanner, or `dev-eicar-only` test scanner (refused in production) |

---

### `apps/api/src/cli` (server scripts)

There is no `apps/api/scripts` folder. The server-side scripts live in `src/cli`. Each npm script first runs `nest build`, then the compiled file.

| File | What it does |
|---|---|
| `apps/api/src/cli/create-platform-admin.ts` | `pnpm --filter api admin:create` — the only way to make a Platform Admin (`PLATFORM_ADMIN_EMAIL`, `_NAME`, `_PASSWORD`); promotes an existing user |
| `apps/api/src/cli/assign-company-admin.ts` | `company:assign-admin` — makes an existing user Company Admin of an existing company (`COMPANY_SLUG`, `USER_EMAIL`) |
| `apps/api/src/cli/move-resumes-to-storage.ts` | `resumes:to-storage` — moves resume bytes from `resume_files` to S3, safe to re-run |
| `apps/api/src/cli/seed-sql-questions.ts` | `sql:seed` — adds the 15 starter SQL questions to one company (`COMPANY_SLUG`) or `all` |

---

### `apps/api/tests`

**What it does**
- Vitest tests. Run with `pnpm --filter api test`.
- Files named `*.db.test.ts` / `*.e2e.test.ts` need a real migrated database in `TEST_DATABASE_URL`. Without it they are skipped (`describe.skipIf`).
- Piston tests need a running Piston (`PISTON_URL`); some need local Python/Java/g++. The load test runs only when asked.
- Most unit tests use in-memory fakes instead of Prisma.

**Root folder (`apps/api/tests/`)**

| File | What it covers |
|---|---|
| `cookies.test.ts` | Session cookie `Secure` flag rules and the `LOGIN_2FA` on/off switch |
| `dsa-access.test.ts` | DSA controllers behind the real `AuthGuard` (fail-closed, CSRF), demo tokens |
| `dsa-bank-sync.test.ts` | HR DSA problem list and API problems stay the same |
| `dsa-coding-author.piston.integration.test.ts` | AI-/HR-written coding problems run through live Piston (skipped without it) |
| `dsa-coding-author.test.ts` | Coding problems from AI, from HR's PDF/Word, from CSV; no language names in AI problems |
| `dsa-fixtures.ts` | Helper: known-correct reference solutions per question and language |
| `dsa-http.e2e.test.ts` | DSA HTTP API with real Nest controllers, code run by local toolchains |
| `dsa-local-runner.test.ts` | Full DSA suite with local Python/Java/g++ (checks harnesses and expected outputs) |
| `dsa-piston-load.test.ts` | Opt-in load test: N students submit at once to real Piston |
| `dsa-piston-runner.test.ts` | Piston response mapping, concurrency limiter, `PistonRunner` |
| `dsa-piston.integration.test.ts` | Live Piston tests (skipped if Piston cannot run code) |
| `dsa-proctoring.e2e.test.ts` | DSA proctoring over HTTP |
| `dsa-proctoring.test.ts` | Proctoring consent, events, validation, warning/disqualify policy |
| `dsa-unit.test.ts` | DSA question data, compare helpers, request validation, `DsaService` |
| `google-auth.db.test.ts` | "Continue with Google" with real DB (fake Google token endpoint) |
| `google-oauth-flow.test.ts` | Google sign-in returns through the dashboard address HR used |
| `in-memory.ts` | Helper: in-memory repositories with the same compare-and-set rules as Prisma ones |
| `invitation-email.test.ts` | `buildInvitationEmail` content and escaping |
| `jobs.test.ts` | JD flow steps 1-4 and delete job, with in-memory repos and a fake AI worker |
| `login-2fa.db.test.ts` | Login 2FA by email code over HTTP, real DB |
| `multi-company.e2e.test.ts` | Real `AppModule`: auth, JD flow per company, company isolation, roles, suspension, CSRF, rate limits |
| `password-reset.db.test.ts` | Forgot/reset password, real DB |
| `signup-verify.db.test.ts` | Sign-up email verification over HTTP, real DB |

**`tests/applications/`**

| File | What it covers |
|---|---|
| `abuse.test.ts` | Rate limiter, `PUBLIC_RATE_LIMITS` parsing, Turnstile, feature flags |
| `confirmation-email.test.ts` | Confirmation email content, reserved-domain block, `mailerFromEnv`, Gmail, mail errors |
| `email-queue.test.ts` | `EmailQueue` parallelism, retries, cancel |
| `fixtures.ts` | Helper: builds tiny real ZIP/DOCX/PDF buffers |
| `flow.db.test.ts` | Publish + apply end to end over HTTP with real DB (S3 optional) |
| `hr-applications.test.ts` | HR application list, resume download, `attachmentHeader` |
| `intake.test.ts` | OTP lock, intake conflict handling, DPDP delete with a fake Prisma |
| `otp-token.test.ts` | Phone proof token, OTP hashing, production safety checks |
| `public-careers.http.test.ts` | `POST /public/applications` over HTTP; `TRUST_PROXY` behaviour |
| `resume-checks.test.ts` | File type checks, ClamAV client, scanner selection |
| `resume-storage.test.ts` | `resumeStorageFromEnv`, S3 storage against a real S3 mock (optional) |

**`tests/publisher/`**

| File | What it covers |
|---|---|
| `fixtures.ts` | Helper: sample form/JD/company, in-memory publisher store, credentials and audit |
| `linkedin-connect.test.ts` | `SecretBox`, OAuth state cookie, `LinkedInOAuthClient` |
| `linkedin.adapter.test.ts` | LinkedIn adapter publish/remove/update with fake fetch; text escaping |
| `post-formatter.test.ts` | Post rendering, helpers, `shorten` |
| `publisher.service.test.ts` | Publish, list, preview, close/reopen/audit, channel connections, deadline passed |

**`tests/ats/`**

| File | What it covers |
|---|---|
| `ats-processor.db.test.ts` | ATS background screening with real DB (SKIP LOCKED, unique keys, cascades) |
| `ats-worker.client.test.ts` | `AtsWorkerClient` HTTP client |
| `candidates.db.test.ts` | Shortlisted students lists, real DB |
| `dead-screening-link.test.ts` | Shortlist email never carries a dead tunnel link |
| `decision-emails.test.ts` | Shortlist and rejection email content, screening link |
| `decisions.db.test.ts` | HR decisions and automatic decision emails, real DB |
| `screening-settings.test.ts` | Minimum ATS score settings, never public, env options |
| `seed.ts` | Helper: real-DB fixtures (company, approved jobs, applications with resume) |

**`tests/screening/`**

| File | What it covers |
|---|---|
| `fakes.ts` | Helper: in-memory screening store and fake worker |
| `flow.test.ts` | Screening question flow |
| `screening.service.test.ts` | Screening link, answers, HR view |

**`tests/rounds/`**

| File | What it covers |
|---|---|
| `fakes.ts` | Helper: in-memory rounds-config and assessments stores, fake AI question generator |
| `assessments.service.test.ts` | Recording results, HR results view, grading rules, changing pass criterion and total marks |
| `rounds-config.service.test.ts` | Saving rounds, company isolation, locking, AI question suggestions, public selection process |

**`tests/round-engine/`**

| File | What it covers |
|---|---|
| `fakes.ts` | Helper: in-memory round-engine store and fake mailer |
| `assessment-protection.test.ts` | Online test protection: warnings then termination, duplicates, env settings |
| `dsa-hr-problem.test.ts` | HR's own DSA problem from rounds editor to judging |
| `dsa-round.db.test.ts` | DSA round end to end with real DB |
| `dsa-round.test.ts` | Which rounds are DSA, DSA link and invitation email, candidate run |
| `round-engine.test.ts` | Online rounds, assessment link, autosave, submit/grade/next step, proctoring events |
| `round-skip.test.ts` | HR skips a round; students go to the next non-skipped round |

**`tests/pipeline/`**

| File | What it covers |
|---|---|
| `pipeline-flow.test.ts` | Candidates A-D through the whole pipeline with real services and in-memory stores |

**`tests/group-discussion/`**

| File | What it covers |
|---|---|
| `fakes.ts` | Helper: in-memory GD store and fakes |
| `gd-flow.db.test.ts` | A whole GD of one group with real DB |
| `gd-store.db.test.ts` | GD store queries with real DB |
| `group-discussion.test.ts` | Group sizes, time slots, topics, pool hand-off, auto scheduling |
| `timeline-metrics.test.ts` | Timeline, metrics/transcripts, links, emails, decisions |

**`tests/ai-interview/`**

| File | What it covers |
|---|---|
| `fakes.ts` | Helper: in-memory AI interview store and fakes |
| `ai-interview.test.ts` | Interview link, consent/devices, conversation, reconnects, report and HR decision |
| `flow.test.ts` | Interview config, server clock, next question, plan helpers |
| `protection.test.ts` | Warning rules, protection flow, duplicates and concurrency |

**`tests/sql/`**

| File | What it covers |
|---|---|
| `fakes.ts` | Helper: in-memory SQL bank store and a test SQL sandbox |
| `sql-bank.service.test.ts` | HR manages SQL questions in the DB |
| `sql-camera-proctoring.test.ts` | Camera monitoring of DSA/SQL attempts and alerts |
| `sql-engine.test.ts` | SQL sandbox runs queries and compares results |
| `sql-round.test.ts` | SQL round run by the round engine on the coding page, proctoring rules |
| `sql-wiring.test.ts` | SQL module is part of the API; `/sql-hr/questions` routes |
| `starter-questions.test.ts` | The 15 starter SQL questions are valid |

**`tests/support/`**

| File | What it covers |
|---|---|
| `execution-suite.ts` | Helper: shared DSA end-to-end suite (harness → runner → evaluator → score) |
| `local-runner.ts` | Helper: test-only runner using local toolchains (NOT a sandbox) |

---

## apps/api — screening, rounds and hiring pipeline

This part of the API takes a student from "applied" to "selected". It reads the resume, runs the AI voice
screening, runs the test rounds HR picked, records the results and sends the emails. Most of it runs in
**background loops** inside the API process. There is no event bus and no separate job queue: the database
tables are the queue.

### Overview: the student journey

1. **Apply** (applications module). Nothing else happens in the request, so applying stays fast.
2. **Resume screening** (`ats`). A loop picks up the new application, sends the resume to the Python AI worker and
   gets an ATS score. Score >= the job's minimum ATS score = `SHORTLISTED`, else `NOT_SHORTLISTED`.
3. **Decision email** (`ats`). A shortlist email (with an AI screening link) or a rejection email is queued at once
   and sent by a second loop.
4. **AI voice screening** (`screening`). The student opens the link and answers 6 spoken questions. This round gives
   **no score and no decision**. It only produces a transcript and notes for HR.
5. **First test round** (`round-engine`). When the screening session is `COMPLETED`, the round-engine loop invites
   the student to the job's first round. There is no HR click in between.
6. **Each round** is graded by `assessments` (`grading.ts` is the only place that decides pass / fail). Then
   `nextStep()` in `round-engine/online-test.ts` decides what comes next (see below).
7. **Pipeline rules** (`pipeline`) can stop a student after the last technical round (Stage 2 overall rule), and can
   turn rejection emails off.
8. **Personal interview** (`pipeline`). HR calls the students who cleared everything and records the call status.

**How `nextStep()` works** (`round-engine/online-test.ts`). It gets the job's rounds, the round just finished and
`passed` (HR override wins over the system result):

| Situation | Result |
|---|---|
| Student failed the round | `EMAIL` kind `REJECTED` |
| Passed, and no next round runs (skipped rounds are ignored) | `EMAIL` kind `CLEARED_ALL` |
| Next round is an online test (MCQ / true-false) or a DSA coding round | `INVITE` (new `round_invites` row) |
| Next round is a Group Discussion and `FEATURE_GROUP_DISCUSSION` is on | `GROUP_DISCUSSION` (GD module takes the student, no email) |
| Next round type has a **connected** round module (SQL, AI interview) | `MODULE` (that module takes the student, no email) |
| Anything else (HR interview, custom round, module not connected) | `EMAIL` kind `CLEARED_NEXT_BY_HR` with the next round's name |

`RoundEngineService.nextFor()` wraps this: after a passed round it also asks `PipelineService.failsStage2Overall()`.
If the Stage 2 rule fails, the step becomes `REJECTED`. `applyStep()` then queues the invite or the outcome email.
For `GROUP_DISCUSSION` and `MODULE` steps it writes nothing. Those modules find the student themselves by polling
(`waitingForModule()` or the GD pool).

**Round modules** connect themselves at start with `RoundModules.connect('SQL' | 'AI_INTERVIEW' | ...)`. They call
`roundEngine.waitingForModule(module)` to find students who passed every earlier round, and
`roundEngine.recordModuleResult(tenantId, module, input, hrDecision?)` to save the result and move the student on.
Today GD (`FEATURE_GROUP_DISCUSSION`), DSA (`DSA_ROUNDS`, run by the round engine itself) and AI interview
(`FEATURE_AI_INTERVIEW`) connect. SQL is not connected yet.

### Background loops, env switches and many API processes

Several API processes can share one PostgreSQL database (for example teammates' laptops on Neon). Every loop
therefore **claims** rows with one SQL statement: `UPDATE ... WHERE id IN (SELECT ... FOR UPDATE SKIP LOCKED)
RETURNING ...`. One row is handled by one process only. A claim older than 15 minutes is taken over (the process
that held it probably crashed). Unique keys (one decision email per application, one invite per application + round,
one outcome email per application + round, `progressed_at` claim on an attempt) make every step happen once.

| Loop | Class | Interval | Turned off by |
|---|---|---|---|
| Resume screening | `ats/ats-processor.service.ts` `AtsProcessor` | `ATS_POLL_MS` (5 s) | `ATS_PROCESSING=off`, `FEATURE_ATS=off`, `NODE_ENV=test` |
| Decision emails | `ats/decision-email.sender.ts` `DecisionEmailSender` | `ATS_POLL_MS`, plus a 1-minute shortlist sweep | `FEATURE_ATS=off`, `NODE_ENV=test` (not by `ATS_PROCESSING`) |
| Round engine | `round-engine/round-engine.worker.ts` `RoundEngineWorker` | `ROUND_ENGINE_POLL_MS` (5 s) | `ROUND_ENGINE_PROCESSING=off`, `NODE_ENV=test` / `VITEST` |

| Env var | Default | What it does |
|---|---|---|
| `ATS_PROCESSING` | on | Resume screening loop on / off |
| `ATS_CONCURRENCY` | 2 (max 10) | Resumes screened at the same time per process |
| `ATS_WORKER_TIMEOUT_MS` | 100000 | Wait for the AI worker's `/v1/resume/screen` |
| `AUTO_REJECTION_EMAILS` | on | Queue the rejection email automatically after screening / HR decision / new minimum |
| `AUTO_SHORTLIST_EMAILS` | on | Same for the shortlist email; also turns on the 1-minute "missed shortlists" sweep |
| `AI_SCREENING_URL` / `CAREERS_SITE_URL` | empty | Base of the screening link. Empty = shortlist emails wait |
| `SCREENING_ALLOW_LOCAL_LINKS` | off | Testing only: allow a `localhost` screening link in emails |
| `AUTO_ROUND_INVITES` | on | Invite to the first round as soon as AI screening is `COMPLETED` |
| `ASSESSMENT_URL` / `CAREERS_SITE_URL` | empty | Base of the test link (`/assessment/{token}`) |
| `DSA_ASSESSMENT_URL` / `CANDIDATE_APP_URL` | empty | Base of the DSA coding link (`/dsa/{token}`) |
| `ASSESSMENT_LINK_DAYS` | 7 | How long a test link works after the email is sent |
| `ROUND_ALLOW_LOCAL_LINKS` | off | Testing only: allow `localhost` test links in emails |
| `DSA_ROUNDS` | on | Off = DSA coding rounds are run by HR |
| `ASSESSMENT_REQUIRE_CAMERA`, `ASSESSMENT_PROTECTION`, `ASSESSMENT_MAX_WARNINGS`, `ASSESSMENT_TAB_MONITORING`, `ASSESSMENT_FULLSCREEN_REQUIRED`, `ASSESSMENT_FACE_POLICY` | on / 2 / on / on / WARN | Webcam and warning rules of online tests |
| `FEATURE_ATS`, `FEATURE_SCREENING`, `FEATURE_ROUNDS`, `FEATURE_ROUND_ENGINE` | on | Turn the HTTP routes off (404 `FEATURE_DISABLED`) |

Note: `FEATURE_ROUND_ENGINE=off` only hides the routes. The round-engine worker keeps running until
`ROUND_ENGINE_PROCESSING=off`.

A link is never emailed when it points at `localhost` (or at a dead Cloudflare quick tunnel, for the screening link).
The email stays `QUEUED` without using up a try and is checked again every 5 minutes. A process that has no public
screening link does not even claim shortlist emails, so the process that has the public link sends them.

---

### `apps/api/src/modules/ats`

**What it does.** Resume screening (M10 / M11). It scores every resume against the job with the AI worker. It keeps
HR's private minimum ATS score per job. It stores HR's decisions and sends the shortlist / rejection emails. The ATS
only suggests (`system_decision`); HR's choice (`hr_decision`) wins until a decision email was sent.

**How it connects.**
- Called by: the jobs module and the publisher (through `settings.ts` only) to save and check the minimum score;
  the screening module (through `index.ts`) for `hashScreeningToken`.
- Calls: the Python AI worker `POST /v1/resume/screen` (`AI_WORKER_URL`), resume storage (`RESUME_STORAGE` from
  applications), the mailer (`MAILER` from applications), the audit log.
- Tables: `ats_evaluations`, `job_screening_settings`, `candidate_emails`, `screening_invites` (in
  `prisma/schema/ats.prisma`); reads `applications`, `jd_versions`, `jobs`, `company_profiles`.
- A missing table (migration not applied) gives 503 `ATS_NOT_READY` and pauses the loops for 5 minutes.

**Main flow.**
1. `AtsProcessor.tick()` creates an `ats_evaluations` row (`PENDING`) for every application that has none.
2. It claims due rows (`FOR UPDATE SKIP LOCKED`), loads the resume and the **JD version the student applied to**,
   and calls the AI worker.
3. It saves the score, breakdown and skills, and sets `system_decision` from the job's minimum (null when HR has not
   set one).
4. In the same transaction it queues the automatic email (`auto-rejection.ts`): `SHORTLIST` or `REJECTION` in
   `candidate_emails`.
5. A technical error is never a rejection. It retries after 1, 5, 15, 60 minutes. After 5 tries, or for an
   unreadable file, the row becomes `NEEDS_REVIEW`. An AI quota error (429) pauses the whole loop and does not count a try.
6. `DecisionEmailSender` claims queued emails (10 at a time, sent one by one). For a shortlist email it creates a new
   random token, stores only its SHA-256 in `screening_invites` (valid 7 days from the send time), and puts the link
   in the email. Mail errors are retried 3 times, then `FAILED` (and the unused link is deleted).
7. At start it also re-sends shortlist emails whose link base changed and no longer opens (`resendDeadLinks`), with a
   "New Link" subject.
8. HR can change the decision (`PUT .../decision`), re-run a `NEEDS_REVIEW` resume, or send / re-send emails by hand.
   Changing the minimum score re-sorts every candidate HR has not decided on or emailed yet.

**HTTP endpoints** (all under `/api/v1`, HR session, feature flag `ats`)

| Method | Path | Who | What |
|---|---|---|---|
| GET | `/ats/jobs` | HR `jobs.read` | Approved jobs with screening counts |
| GET | `/ats/jobs/:jobId/candidates?view=` | HR `jobs.read` | Candidates of one view: `shortlisted` (default), `not_shortlisted`, `needs_review`, `pending`, `all` |
| PUT | `/ats/applications/:applicationId/decision` | HR `jobs.write` | `{ decision: SHORTLISTED \| NOT_SHORTLISTED \| null }`; 409 once an email went out |
| POST | `/ats/applications/:applicationId/retry` | HR `jobs.write` | Send a `NEEDS_REVIEW` resume back to screening |
| POST | `/ats/emails` | HR `jobs.write` | `{ kind, applicationIds }`: queue decision emails (or re-send `FAILED` ones) |
| GET | `/ats/jobs/:jobId/settings` | HR `jobs.read` | The job's minimum ATS score |
| PUT | `/ats/jobs/:jobId/settings` | HR `jobs.write` | `{ minAtsScore: 0-100 }`; re-sorts and queues automatic emails |

**Files**

| File | What it does |
|---|---|
| `README.md` | Module notes: minimum score, background screening, decision emails, import rules |
| `ats-processor.service.ts` | `AtsProcessor` loop: queue new applications, claim rows, call the AI worker, save score + decision, retry / `NEEDS_REVIEW`, quota pause; reads `ATS_*` env |
| `ats-worker.client.ts` | `AtsWorkerClient` for `POST /v1/resume/screen`; maps HTTP errors to `AtsWorkerError` (retryable / review / quota) |
| `ats.module.ts` | Nest module: decision + candidates controllers, processor, sender, worker client, screening base URL provider |
| `auto-rejection.ts` | `AUTO_REJECTION_EMAILS` / `AUTO_SHORTLIST_EMAILS` flags and `queueAutoRejections` / `queueAutoShortlists` (insert into `candidate_emails`, one per application) |
| `candidates.controller.ts` | HR routes `GET /ats/jobs` and `GET /ats/jobs/:jobId/candidates` |
| `candidates.service.ts` | Builds the "Shortlisted Students" page data: buckets (shortlisted, not shortlisted, needs review, pending), counts, score order |
| `decision-email.sender.ts` | `DecisionEmailSender` loop: claim and send emails, create screening tokens, hold emails until a public link exists, dead-link re-send, missed-shortlist sweep |
| `decision.controller.ts` | HR routes: set decision, retry screening, queue emails (its comment "never sent automatically" is outdated) |
| `decision.service.ts` | Rules for HR decisions, manual email requests (`whyNot` checks) and retry; `SCREENING_BASE_URL` token |
| `emails/decision-emails.ts` | Text + HTML of the shortlist and rejection emails (agreed wording, no score) |
| `index.ts` | Public entry: `AtsModule`, `hashScreeningToken`, settings exports |
| `screening-link.ts` | Screening link base from `AI_SCREENING_URL` or `CAREERS_SITE_URL/screening/{token}`, token create + SHA-256, 7-day validity |
| `screening-settings.controller.ts` | HR routes `GET / PUT /ats/jobs/:jobId/settings` |
| `screening-settings.module.ts` | Small module for the minimum score only, so jobs and publisher avoid an import circle |
| `screening-settings.service.ts` | Save / read the minimum score, re-sort candidates, `isSet()` publish check; `isMissingTable` / `atsNotReady` helpers |
| `settings.ts` | Second public entry (settings only) for the jobs and publisher modules |

---

### `apps/api/src/modules/screening`

**What it does.** The AI voice screening round (M12) after a shortlist. The student opens the link from the shortlist
email, gives consent and answers six spoken questions. Each answer is transcribed by the AI worker and saved at once.
HR then reads the transcript and a structured summary. This round gives **no score and no decision**.

**How it connects.**
- Called by: the careers site page `/screening/[token]` (public routes) and the HR dashboard (HR routes).
- Uses the token created by `ats` (`screening_invites`, looked up by SHA-256 via `link.ts`).
- Calls the AI worker: `/v1/screening/turn` (transcribe + decide follow-up), `/v1/screening/speak` (question as WAV),
  `/v1/screening/extract` (summary). Base URL `AI_WORKER_URL`.
- Tables: `screening_sessions`, `screening_turns`, `screening_results` (`prisma/schema/screening.prisma`), plus
  `consents` (applications) and `screening_invites` (ats).
- The round engine reads `screening_sessions.status = 'COMPLETED'` to send the first round invite. The assessments
  module reads the session to show the "voice screening" flag.
- Env: `SCREENING_FOLLOW_UPS_PER_QUESTION`, `SCREENING_MAX_CANDIDATE_QUESTIONS`, `SCREENING_MAX_TURNS`,
  `SCREENING_MAX_MINUTES`, `SCREENING_WORKER_TIMEOUT_MS`, `SCREENING_SPEECH_TIMEOUT_MS`, `SCREENING_EXTRACT_TIMEOUT_MS`.

**Main flow.**
1. `GET /public/screening/session` checks the token (shape, hash, not expired) and creates the session
   (`NOT_STARTED`, first question pending).
2. `POST /start` with `{ consent: true }` records consent (`purpose: ai_voice_screening`) and sets `IN_PROGRESS`.
   An `ENDED_EARLY` session can resume.
3. The page plays the question (`GET /speech?seq=`, cached in `SpeechCache`) and uploads the recorded answer
   (`POST /answer`, max 6 MB).
4. The AI worker returns the transcript and "needs follow-up?". `flow.ts` (not the AI) decides the next question:
   steps `INTRO`, `EDUCATION`, `BACKGROUND`, `LOCATION`, `AVAILABILITY`, `CANDIDATE_QUESTIONS`, with follow-up,
   turn and time limits.
5. `saveTurn` is safe against double uploads (`clientTurnId`) and parallel requests (`pendingSeq` check).
6. After the last step the session is `COMPLETED`, the invite is marked used, and the summary is made in the
   background. HR can create the summary again.

**HTTP endpoints** (under `/api/v1`, feature flag `screening`)

| Method | Path | Who | What |
|---|---|---|---|
| GET | `/public/screening/session` | Candidate (header `x-screening-token`) | Screening state; creates the session on first visit |
| POST | `/public/screening/start` | Candidate | `{ consent: true }`; start or resume |
| POST | `/public/screening/answer` | Candidate | Multipart `audio`, `seq`, `clientTurnId`, `durationMs` |
| GET | `/public/screening/speech?seq=` | Candidate | The question as `audio/wav` |
| POST | `/public/screening/end` | Candidate | Stop early (answers kept, can resume) |
| GET | `/screening/jobs/:jobId` | HR `jobs.read` | Every application of the job with a screening link, with display status |
| GET | `/screening/applications/:applicationId` | HR `jobs.read` | Transcript + summary |
| PUT | `/screening/applications/:applicationId/review` | HR `jobs.write` | `{ reviewStatus }` |
| POST | `/screening/applications/:applicationId/summary` | HR `jobs.write` | Create the summary again |

**Files**

| File | What it does |
|---|---|
| `README.md` | Module notes, routes, tables, what is not built yet |
| `flow.ts` | Pure question flow: main questions, follow-up limits, turn / time caps, closing text; `flowLimitsFromEnv` |
| `hr-screening.controller.ts` | HR routes (list per job, detail, review, retry summary) |
| `hr-screening.service.ts` | HR views; `displayStatus` (`EXPIRED`, `INTERRUPTED` after 30 idle minutes, ...) |
| `index.ts` | Public entry: `ScreeningModule`, `SpeechCache` (also used by ai-interview) |
| `link.ts` | Re-exports `hashScreeningToken` from `ats` |
| `public-screening.controller.ts` | Public candidate routes, per-IP rate limits, file upload limits |
| `screening-worker.client.ts` | Client for `/v1/screening/turn`, `/speak`, `/extract`; `ScreeningWorkerError` |
| `screening.errors.ts` | `screeningError()` helper for the common error body |
| `screening.module.ts` | Nest module: controllers, services, Prisma store, worker client, speech cache, flow limits, clock |
| `screening.service.ts` | Candidate side: token check, start, answer, speech, end, background summary; job facts for the AI |
| `screening.store.ts` | All database access (abstract `ScreeningStore` + `PrismaScreeningStore`); lenient job form reader |
| `speech-cache.ts` | In-memory LRU (120 entries) of spoken questions, shares parallel requests |

---

### `apps/api/src/modules/rounds-config`

**What it does.** The round configurator (M05). HR picks the assessment rounds of each job, their order, questions
and pass rules, in the Publish dialog. A config is `DRAFT` or `COMPLETE`. When the first student starts, the rounds
are **locked**: questions and rounds can no longer change (409 `ROUNDS_LOCKED`). Pass rules and "skip round" still
work after the lock. Only the round **names** ever become public (the "Selection process" on the careers page).

**How it connects.**
- Called by: the HR dashboard; the publisher (`publicSteps()` – refuses to publish until `COMPLETE`); the assessments
  module (`forRoundEngine`, `lockForAttempts`, `setPassRule`, `listConfigs`); the round engine (`forRoundEngine`,
  `lockForAttempts`, `setSkip`, `get`); the pipeline module; dsa / sql modules (`jobFactsFor`).
- Calls the AI worker `POST /v1/rounds/questions` for question suggestions (`ROUND_QUESTIONS_WORKER_TIMEOUT_MS`).
- Tables: `job_round_configs`, `job_rounds`, `round_questions` (`prisma/schema/rounds.prisma`).
- `RoundModules` (a registry in memory) says which round modules are connected in this process.
- Shared rules: `packages/contracts/src/schemas/rounds.ts` (`saveRoundsRequestSchema`, `roundsCompletionProblems`,
  `activeRounds`, `ROUND_MODULE_OF`).

**Main flow.**
1. HR saves all rounds at once (`PUT`). `toStored()` keeps known round / question ids and keeps HR's skip.
2. The store replaces every round and question in one transaction, and checks the lock in the same statement.
3. `complete: true` needs at least one round, a pass rule on each, and questions on each round not scored by an
   interviewer.
4. HR can ask the AI for questions (`POST .../generate`). Nothing is saved; broken or duplicate questions are dropped.
5. The first test start or first result calls `lockForAttempts()` and sets `locked_at` once.

**HTTP endpoints** (under `/api/v1`, feature flag `rounds`)

| Method | Path | Who | What |
|---|---|---|---|
| GET | `/rounds/jobs/:jobId` | HR `jobs.read` | Rounds, questions, pass rules, `problems` still missing |
| PUT | `/rounds/jobs/:jobId` | HR `jobs.write` | `{ rounds, complete }`; replaces every round |
| POST | `/rounds/jobs/:jobId/generate` | HR `jobs.write` | AI question suggestions for one round (saves nothing) |

(Skip / unskip routes under `/rounds/jobs/:jobId/rounds/:roundId/...` live in `round-engine/round-skip.controller.ts`.)

**Files**

| File | What it does |
|---|---|
| `README.md` | Module notes: privacy rule, draft vs complete, lock, API for the round engine |
| `index.ts` | Public entry: module, `RoundModules`, `RoundsConfigService`, `passRuleOf`, `toSavedRound`, types |
| `question-generator.client.ts` | Client for the AI worker `POST /v1/rounds/questions`; keeps only well-shaped questions |
| `round-modules.ts` | `RoundModules` registry: `connect()`, `isConnected()`, `connected()`; how a new module plugs in |
| `rounds-config.controller.ts` | HR routes get / save / generate |
| `rounds-config.module.ts` | Nest module (imports only tenants, to avoid cycles); exports service and `RoundModules` |
| `rounds-config.service.ts` | Save, generate, `publicSteps`, `forRoundEngine`, `lockForAttempts`, `setSkip`, `setPassRule`, `jobFactsFor` |
| `rounds-config.store.ts` | All database access; `replace()` in one transaction with a lock check; reads DSA / SQL problem specs |

---

### `apps/api/src/modules/assessments`

**What it does.** Stores the result of each round for each student (`round_results`, one row per application per
round). `grading.ts` is the **only** place where pass / fail is decided. HR sees results on the Assessments page,
can override pass / fail with a reason, and can change a round's pass rule later (all results of that round are then
graded again).

**How it connects.**
- Called by: the round engine (`record()` after every test, `override()` for integrity decisions and AI interview
  decisions); the pipeline module (`resultsFor`, `studentsOf`, `jobTitle`); the HR dashboard.
- Calls: `RoundsConfigService` (rounds, lock, pass rule), the audit log.
- Tables: `round_results`; reads `applications`, `ats_evaluations`, `screening_*`, `round_invites`,
  `round_attempts`, `gd_participants`, `ai_interview_*` to show "where the student is" in a round.
- Shape of a result: `roundResultInputSchema` in `packages/contracts/src/schemas/assessments.ts`.

**Main flow.**
1. `record(tenantId, input)` checks the round belongs to the student's job and the config is `COMPLETE`.
2. `gradeResult()` grades MCQ / true-false answers against the saved correct options, then applies the pass rule:
   `CORRECT_COUNT`, `MARKS` (can be scaled to HR's max), `PERCENTAGE` or `INTERVIEWER_SCORE`. Impossible numbers give
   422 `INVALID_ROUND_RESULT`.
3. The first result locks the rounds. The result is an upsert: sending it again replaces it, but an HR override stays.
4. Outcome = HR override if set, else `systemPassed` (`outcomeOf`). `stageOf` gives `NOT_STARTED`, `IN_PROGRESS`,
   `CLEARED` or `NOT_CLEARED`.
5. "Students who reached the rounds" = shortlisted after resume screening, or anyone with a round result.

**HTTP endpoints** (under `/api/v1`, feature flag `rounds`)

| Method | Path | Who | What |
|---|---|---|---|
| GET | `/assessments/jobs` | HR `jobs.read` | Jobs that have rounds, with counts |
| GET | `/assessments/jobs/:jobId` | HR `jobs.read` | Every student x every round: outcome, score, progress; pass / fail counts |
| GET | `/assessments/applications/:applicationId` | HR `jobs.read` | One student's full review with answers |
| PUT | `/assessments/jobs/:jobId/rounds/:roundId/pass-rule` | HR `jobs.write` | `{ passRule }`; re-grades the round, lists changed students, sends nothing |
| PUT | `/assessments/applications/:applicationId/rounds/:roundId/override` | HR `jobs.write` | `{ passed, reason }` (`passed: null` removes the override) |

**Files**

| File | What it does |
|---|---|
| `README.md` | Module notes and how the round engine calls `record()` (the pass-rule route is not listed there) |
| `assessments.controller.ts` | HR routes above |
| `assessments.module.ts` | Nest module; imports rounds-config; exports `AssessmentsService` |
| `assessments.service.ts` | `record`, HR views, `changePassRule` (re-grade all or nothing), `override`, helpers for pipeline |
| `assessments.store.ts` | All database access; student list with ATS, AI screening and per-round progress text |
| `grading.ts` | `gradeResult`, `outcomeOf`, `stageOf`, `InvalidResultError` |
| `index.ts` | Public entry: module and `AssessmentsService` |

---

### `apps/api/src/modules/round-engine`

**What it does.** Runs the online test rounds (M13): invitation email, secure link, timed and monitored test on the
careers site (`/assessment/[token]`), grading through `assessments`, then the next step. It also runs DSA coding
rounds for the `dsa` module (`/dsa/[token]` on the candidate app). It moves students on after GD / AI interview /
other module results, handles HR's "skip round", and enforces the test protection (warnings, termination, HR review).

**How it connects.**
- Uses: `RoundsConfigService` (rounds, lock, skip), `AssessmentsService` (record, override), `PipelineService`
  (Stage 2 rule, gate, rejection emails on / off), `RoundModules`, `MAILER`, audit.
- Called by: careers site (public routes), HR dashboard, `dsa` module (`openCodingRound`, `startCodingAttempt`,
  `saveCodingProgress`, `finishCodingAttempt`, `continueAfterCoding`), `group-discussion`
  (`continueAfterExternalRound`), `ai-interview` (`waitingForModule`, `recordModuleResult`).
- Tables: `round_invites`, `round_attempts` (incl. `coding_state`, `protection`), `round_proctor_events`,
  `round_protection_incidents`, `round_outcome_emails` (`prisma/schema/round_engine.prisma`); results go to
  `round_results`. Reads `screening_sessions` to find completed screenings.

**Main flow.**
1. **Discover.** The worker runs `screeningsToInvite()`: completed AI screening + job's first non-skipped round is an
   online test (or DSA round) + no invite yet. It queues a `round_invites` row (`AUTO_ROUND_INVITES`).
2. **Send invite.** The worker claims the invite, makes a fresh token (only SHA-256 stored), sends the email. It waits
   (no try used) while the link base is missing or `localhost`. The link works `ASSESSMENT_LINK_DAYS` from sending.
3. **Test.** The student starts (consent). This locks the rounds and creates the attempt with a server deadline. The
   page autosaves (`revision` stops old saves from winning) and reports events and violations.
4. **Finish.** Submit, time-out (`view`, `save` or the worker's `sweepOverdue`), or termination by the warning limit.
5. **Progress** (`progress()`): grade through `assessments.record()`, store `systemPassed`, then claim
   `progressed_at` once, compute `nextFor()` and `applyStep()` (queue invite or outcome email). A terminated attempt
   is graded but **held** until HR's integrity decision. If anything fails, the worker's `recover()` retries.
6. **Outcome email.** The worker sends `REJECTED`, `CLEARED_NEXT_BY_HR` or `CLEARED_ALL`. After an `AI_INTERVIEW`
   round the "selected for personal interview" text is used. Rejection emails are skipped when the pipeline setting
   is off.
7. **Skip round.** HR skips a round; waiting students (invite not started, or in the GD pool) are moved on as if they
   passed the round before. "Unskip" never pulls anybody back.
8. **DSA → DSA / SQL.** After a coding round, the next coding round's invite email is held 10 minutes. The coding
   page can take the student straight on (`continueAfterCoding`).

**HTTP endpoints** (under `/api/v1`)

| Method | Path | Who | What |
|---|---|---|---|
| GET | `/public/assessment/session` | Candidate (header `x-assessment-token`) | Test state; submits a test whose time ran out |
| POST | `/public/assessment/start` | Candidate | `{ consent: true }`; starts the clock (once) |
| PUT | `/public/assessment/answers` | Candidate | Autosave `{ revision, answers }` |
| POST | `/public/assessment/submit` | Candidate | Final answers; graded at once |
| POST | `/public/assessment/events` | Candidate | Proctoring events (camera, tab, full screen, connection) |
| POST | `/public/assessment/violation` | Candidate | `{ incidentId, type }`; server decides warning / termination under a row lock |
| POST | `/public/assessment/violation/ack` | Candidate | `{ warningNumber }`; candidate read the warning |
| GET | `/round-engine/applications/:applicationId` | HR `jobs.read` | Invites, attempts, events, incidents, outcome emails per round |
| POST | `/round-engine/applications/:applicationId/rounds/:roundId/integrity-decision` | HR `jobs.write` | `{ decision: ADVANCE \| REJECT, reason }` for a terminated test |
| POST | `/round-engine/applications/:applicationId/rounds/:roundId/revoke` | HR `jobs.write` | Withdraw the link |
| POST | `/round-engine/applications/:applicationId/rounds/:roundId/resend` | HR `jobs.write` | Re-queue a `FAILED` invitation (new link) |
| POST | `/rounds/jobs/:jobId/rounds/:roundId/skip` | HR `jobs.write` | `{ reason }`; skip the round for the job, move waiting students on |
| POST | `/rounds/jobs/:jobId/rounds/:roundId/unskip` | HR `jobs.write` | The round runs again for students not yet past it |

Public and HR round-engine routes use feature flag `round_engine`; skip / unskip use `rounds`.

**Files**

| File | What it does |
|---|---|
| `README.md` | Module notes, round modules, protection rules, routes |
| `assessment-protection.ts` | Pure protection rules: settings from env, which incidents count, termination reason, page state |
| `coding-round.ts` | Pure DSA rules: coding state per attempt, marks (best submission), answers for assessments, `DSA_ROUNDS`, camera mode |
| `emails.ts` | Invitation, rejection, "cleared" and "selected for personal interview" emails (no score, no pass mark) |
| `hr-round-engine.controller.ts` | HR routes: view, integrity decision, revoke, resend |
| `index.ts` | Public entry: module, service, `roundKind`, coding helpers, `hashAssessmentToken` |
| `link.ts` | Token create + SHA-256, link bases from `ASSESSMENT_URL` / `CAREERS_SITE_URL` and `DSA_ASSESSMENT_URL` / `CANDIDATE_APP_URL` |
| `online-test.ts` | Pure rules: `roundKind`, public questions (no answers), answer checks, **`nextStep()`**, event severity |
| `public-assessment.controller.ts` | Public candidate routes with per-IP rate limit `assessment` |
| `round-engine.module.ts` | Nest module; connects `DSA` to `RoundModules` when `DSA_ROUNDS` is on |
| `round-engine.service.ts` | Candidate flow, `finish` / `progress`, `nextFor` / `applyStep`, module API, skip, coding API, HR views |
| `round-engine.store.ts` | All database access; claim queries with `FOR UPDATE SKIP LOCKED`; raw SQL for "who is waiting" and "who to invite" |
| `round-engine.worker.ts` | Background loop: discover, sweep overdue, recover, send invites, send outcome emails; reads worker env |
| `round-skip.controller.ts` | HR routes skip / unskip |

---

### `apps/api/src/modules/pipeline`

**What it does.** The rules **between** rounds, on top of each round's own pass rule. It holds the Stage 2 overall
rule per job, the "send rejection emails" switch, HR's GD rubric, every student's computed pipeline status, and the
personal-interview (PI) call board. Statuses are computed, never stored. Settings and rubrics freeze with the rounds
(409 `PIPELINE_LOCKED`).

**How it connects.**
- Called by: round engine (`failsStage2Overall`, `gateAllows`, `rejectionEmailsOn`); group-discussion (`rubricOf`,
  `checkRubricScores`, `rejectionEmailsOn`); HR dashboard.
- Calls: `RoundsConfigService`, `AssessmentsService`, audit.
- Tables: `job_pipeline_settings`, `round_rubrics`, `pi_calls` (`prisma/schema/pipeline.prisma`). No settings row =
  defaults (`ALL_ROUNDS_PASSED`, rejection emails on).
- Pure rules: `packages/contracts/src/schemas/pipeline.ts` (`stage2Verdict`, `pipelineStatusOf`, `stage2GateAllows`).

**Main flow.**
1. Stages: 1 Aptitude (`APTITUDE`, `TECHNICAL_MCQ`, `COMMUNICATION`) → 2 Technical (`CODING`, `SQL`,
   `GROUP_DISCUSSION`) → 3 Tech interview (`AI_INTERVIEW`, `TECHNICAL_INTERVIEW`) → 4 Personal interview (`HR_INTERVIEW`).
2. When a student **passes the last technical round** that runs, the Stage 2 rule is checked: `ALL_ROUNDS_PASSED`,
   `TOTAL_PERCENT` or `BOTH`. Fail = `REJECTED` email, status `REJECTED_MODULE2_OVERALL`.
3. `gateAllows()` stops round modules (for example AI interview) from taking a student who failed the Stage 2 rule.
4. The PI board lists students who cleared everything before the HR interview, best overall % first. HR sets the call
   status (`NOT_CALLED`, `CALLED`, `SCHEDULED`, `SELECTED`, `REJECTED`); `SELECTED` / `REJECTED` change the status.

**HTTP endpoints** (under `/api/v1`, feature flag `rounds`)

| Method | Path | Who | What |
|---|---|---|---|
| GET | `/pipeline/jobs/:jobId/settings` | HR `jobs.read` | Stage 2 rule and rejection-email switch |
| PUT | `/pipeline/jobs/:jobId/settings` | HR `jobs.write` | `{ stage2Rule, stage2MinPercent, rejectionEmails }` |
| GET | `/pipeline/jobs/:jobId/rounds/:roundId/rubric` | HR `jobs.read` | GD rubric |
| PUT | `/pipeline/jobs/:jobId/rounds/:roundId/rubric` | HR `jobs.write` | `{ parameters: [{ name, maxMarks }] }` (empty = delete) |
| GET | `/pipeline/jobs/:jobId/statuses` | HR `jobs.read` | Every student's pipeline status and Stage 2 % |
| GET | `/pipeline/jobs/:jobId/personal-interview` | HR `jobs.read` | PI board |
| PUT | `/pipeline/jobs/:jobId/personal-interview/:applicationId` | HR `jobs.write` | `{ status, notes }` |

**Files**

| File | What it does |
|---|---|
| `README.md` | Short notes: routes, tables, who uses which method |
| `index.ts` | Public entry: module, `PipelineService`, `PipelineStore`, `MemoryPipelineStore` |
| `pipeline.controller.ts` | HR routes above |
| `pipeline.module.ts` | Nest module; imports rounds-config and assessments |
| `pipeline.service.ts` | Settings, rubric, Stage 2 checks, gate, statuses, PI board and PI call update |
| `pipeline.store.ts` | Prisma store for settings, rubrics, PI calls, plus an in-memory store for tests |

---

### Stub modules (empty placeholders)

These folders hold **only a `README.md`**. There is no code, no Nest module and no route. They are not imported in
`app.module.ts`. The real work for some of them lives elsewhere today.

| Folder | File | Planned purpose | Where it is done today |
|---|---|---|---|
| `apps/api/src/modules/analytics` | `README.md` | M27: funnel and source stats (v1.0) | Not built; HR pages compute counts themselves (`ats`, `assessments`) |
| `apps/api/src/modules/evaluator` | `README.md` | M22: score screening and round attempts with reasons | `assessments/grading.ts` and the AI worker |
| `apps/api/src/modules/feedback` | `README.md` | M23: feedback and report cards | Not built |
| `apps/api/src/modules/notifications` | `README.md` | M24: email first, WhatsApp later | Each module sends its own email: `ats/decision-email.sender.ts`, `round-engine/round-engine.worker.ts` (through `MAILER` from applications) |
| `apps/api/src/modules/proctoring` | `README.md` | M21: proctor events, WebSocket heartbeat, flags | `round-engine` (events, violations) and the `dsa` module |
| `apps/api/src/modules/scheduler` | `README.md` | M25: PI slot booking with Google Calendar | Not built; PI calls are tracked by hand in `pipeline` |

### Known gaps worth knowing

- An HR override **after** the outcome email does not send a new email or invite (`assessments` override does not
  re-run `nextStep`).
- Rate limits and the speech cache are in memory, so they are per process, not shared between API processes.
- `RoundModules` is also per process: a module is "connected" only in the processes where it started.
- `continueAfterCoding()` looks for the next round by position without skipping skipped rounds (it uses a plain sort,
  not `activeRounds`). If the next round is skipped, the coding page shows `DONE`, but the invite to the later round
  is still emailed.

---

## apps/api — round modules: DSA, SQL, group discussion, AI interview

HR builds a job's selection process in the rounds editor (`rounds-config`). The **round engine** (`apps/api/src/modules/round-engine`) moves each student from round to round. Some round types are run by a separate **round module**. This section covers those modules.

There are two ways a module gets students from the engine:

| Round type (`ROUND_MODULE_OF` in `packages/contracts/src/schemas/rounds.ts`) | Who invites the student | How the result goes back |
|---|---|---|
| `CODING` (DSA) and SQL problems | The **round engine itself**. A round whose questions all pass `isDsaQuestion()` is a "coding round" (`roundKind() = 'CODING'`). The engine sends the email with a link to the candidate app's `/dsa/[token]` page. | The dsa module saves progress on the engine's `round_attempts` row (`EngineSessionStore`). `finishCodingAttempt()` records the result and queues the next email. |
| `AI_INTERVIEW` | The ai-interview worker asks `RoundEngineService.waitingForModule('AI_INTERVIEW')`, then invites. | `RoundEngineService.recordModuleResult(tenantId, 'AI_INTERVIEW', …, hrDecision)` |
| `GROUP_DISCUSSION` | The GD worker finds students with its own SQL query (`GdStore.poolCandidates`) and puts them in a pool. | `AssessmentsService.record()` / `override()`, then `RoundEngineService.continueAfterExternalRound()` |

A module "plugs in" with `RoundModules.connect('<MODULE>')` in its `onModuleInit` (`rounds-config/round-modules.ts`). If a module is not connected, the engine sends the "HR will run this round" email instead. `waitingForModule('DSA')` always returns `[]`, because DSA rounds belong to the engine. DSA is connected by the round engine when `DSA_ROUNDS` is on. No code calls `connect('SQL')`: SQL problems run as coding rounds.

External services used in this section:

| Service | Default address | Used by |
|---|---|---|
| Piston (code runner, Docker) | `PISTON_URL` = `http://127.0.0.1:2000` | dsa (Run / Submit; AI-authored reference solutions) |
| ai-worker (FastAPI, Python) | `AI_WORKER_URL` = `http://localhost:8000` | dsa authoring (`/v1/rounds/coding-problems`), ai-interview (`/v1/interview/plan`, `/turn`, `/speak`, `/report`) |
| gd-agent (FastAPI, Python) | `GD_WORKER_URL` = `http://localhost:8010` | group-discussion (`/gd/transcribe`, `/gd/evaluate`, `/gd/mix`); it also runs the LiveKit moderator bot `gd-moderator` |
| Gemini | (only through the two Python workers) | Problem writing, interview turns, transcription, reports, GD rubric. No module in this section calls Gemini directly. |
| LiveKit (Cloud or own server) | `LIVEKIT_URL`, `LIVEKIT_API_KEY`, `LIVEKIT_API_SECRET` | group-discussion live rooms |
| Email (`MAILER` from applications) | SMTP / Gmail settings | ai-interview invitations; GD invites, reminders, results. DSA/SQL emails are sent by the round engine. |
| Node built-in `node:sqlite` | in a child process | sql sandbox |

---

### `apps/api/src/modules/dsa`

**What it does.** This is the API for the candidate's coding page (`/dsa/[token]` in the candidate app). It shows the problems, keeps the server-side timer, and runs code on Piston. Submit scores code against the sample tests and the hidden tests. It also does browser proctoring (full screen, tab switch, AI camera alerts) and HR's monitoring review. HR can also get coding problems written by AI. The same page runs **SQL problems**; in that case a query runs in the sql module's sandbox, not on Piston.

**How it connects.**
- Imports `RoundEngineModule`, `RoundsConfigModule`, `ApplicationsModule`, `GroupDiscussionModule` (for `GdMailService.studentLink`), `SqlModule` (for `SQL_SANDBOX`), `TenantsModule`.
- Talks to the round engine only through `RoundEngineService`. It calls `openCodingRound`, `startCodingAttempt`, `saveCodingProgress`, `finishCodingAttempt` and `continueAfterCoding`.
- DB tables: `round_attempts` (owned by the engine; the coding state is stored there) and `dsa_proctor_sessions` (one JSON row per invitation, keyed by `inviteRef` = a SHA-256 of the token).
- Problems: the built-in bank (`questions/mock-questions.data.ts`, 3 problems; the HR list is `DSA_BANK` in `packages/contracts/src/schemas/dsa-bank.ts`), HR's own problems (`coding` spec on the round question), or SQL problems (`sql` spec).
- Env, main ones:
  - On/off and mode: `FEATURE_DSA` (routes on/off), `DSA_ROUNDS` (engine runs coding rounds), `DSA_DEMO_MODE`, `DSA_DEMO_TOKENS`, `DSA_ALLOW_DEMO_TOKENS`, `DSA_DEMO_DURATION_MINUTES`, `DSA_SHOW_SCORE` (default off).
  - Piston: `PISTON_URL`, `DSA_RUNNER_CONCURRENCY` (3), `DSA_RUNNER_PER_SUBMISSION` (2), `DSA_RUNNER_MAX_QUEUE` (60), `DSA_RUNNER_QUEUE_TIMEOUT_MS`, `DSA_RUNNER_REQUEST_TIMEOUT_MS`.
  - Run limits: `DSA_RUN_TIMEOUT_MS` (5000), `DSA_RUN_CPU_MS`, `DSA_RUN_MEMORY_MB` (256), `DSA_COMPILE_*`, `DSA_JAVA_RUN_*` (Java gets 8 s and 512 MB).
  - Proctoring: `DSA_PROCTORING`, `DSA_PROCTOR_MAX_STRIKES` (3), `DSA_PROCTOR_CAMERA` (`all` | `sql` | `off`), `DSA_PROCTOR_CAMERA_STRIKES`, `DSA_PROCTOR_CAMERA_COOLDOWN_MS`, `DSA_PROCTOR_REQUIRE_FULLSCREEN`, `DSA_PROCTOR_FULLSCREEN_REENTRY`, `DSA_PROCTOR_*_SCORE`, `DSA_PROCTOR_AUTO_END`, `DSA_PROCTOR_DEDUPE_MS`, `DSA_PROCTOR_MERGE_MS`, `DSA_PROCTOR_MAX_EVENTS`.
  - Other: `DSA_ADMIN_API_KEY`, `AI_WORKER_URL`, `CODING_PROBLEMS_WORKER_TIMEOUT_MS`.
- `DSA_DEMO_MODE=on` swaps in the in-memory resolver and stores (see `dsa.module.ts`).

**Main flow.**
1. The round engine emails the student a link with a token. Every request goes to `/dsa/:token/...`.
2. `DsaService.load()` → `RoundInvitationResolver` → `engine.openCodingRound(token)` → an invitation with the round's questions. `BankQuestionRepository` turns each question into a `DsaQuestion` (bank, HR `stdio` problem, or SQL problem).
3. The page calls `proctor/start` (consent and full screen; the webcam too if the camera is on). Then it calls `start`. The first save creates the engine attempt with `startCodingAttempt`, which starts the clock.
4. **Run** uses only the sample tests. **Submit** uses the sample and hidden tests. The `Evaluator` wraps the code in a harness (`evaluation/harness.ts`) and sends it to `PistonRunner`. It runs the first case alone, so a compile error costs one run. SQL goes to `SqlEvaluator` → `SqlSandbox`.
5. Score = `marks × passed / total`. The best submission counts, and its source is kept for HR (never sent to the browser). Each candidate can run only one job at a time (HTTP 429 `BUSY`).
6. `proctor/events` records violations. When violations go **above** `maxStrikes`, `closeSession(…, 'disqualified')` ends the attempt. The work so far is kept.
7. The attempt ends in one of three ways: `finish`, the deadline (`expireIfNeeded`; the engine's own sweeper also handles closed tabs), or disqualification. Then `finishCodingAttempt` records the result, and the engine sends the next email.
8. `continue` asks the engine what comes next. It can be the next coding round's token (same page), a GD link (`GdMailService.studentLink`), or `DONE`.

**HTTP endpoints** (prefix `/api/v1`).

| Method | Path | Who | What |
|---|---|---|---|
| GET | `/dsa/:token` | candidate (public, token) | Instructions, timer, questions (no hidden tests), progress, proctoring state |
| POST | `/dsa/:token/start` | candidate | Start the server timer (needs proctoring started) |
| POST | `/dsa/:token/run` | candidate | `{questionId, language, code}` against the sample tests |
| POST | `/dsa/:token/submit` | candidate | Judge against all tests and save the best score |
| POST | `/dsa/:token/finish` | candidate | End the attempt |
| POST | `/dsa/:token/continue` | candidate | What comes next after the round (next coding token / GD / DONE) |
| POST | `/dsa/:token/proctor/start` | candidate | Record consent (and full screen) |
| POST | `/dsa/:token/proctor/events` | candidate | Up to 20 integrity events; returns `ended` |
| POST | `/dsa/:token/proctor/end` | candidate | Seal the record (only after finish) |
| GET | `/dsa-hr/applications/:applicationId/proctoring` | HR `jobs.read` | Monitoring record per round (max 300 events) |
| POST | `/dsa-hr/jobs/:jobId/coding-problems/generate` | HR `jobs.write` | AI-written problems (not saved) |
| POST | `/dsa-hr/jobs/:jobId/coding-problems/import` | HR `jobs.write` | Problems from a PDF / DOCX (AI) or CSV file, max 5 MB |
| GET | `/dsa-admin/proctoring/flagged` | service key `x-dsa-admin-key` | Flagged attempts (404 unless `DSA_ADMIN_API_KEY` is set) |
| GET | `/dsa-admin/proctoring/:inviteRef/summary` | service key | One attempt's summary |
| GET | `/dsa-admin/proctoring/:inviteRef/events` | service key | Paged event log |

The candidate routes are `@Public()` and use the rate limit `assessment`. Writes need the CSRF header `x-requested-with: hireflow`.

**Files (top level).**

| File | What it does |
|---|---|
| `README.md` | Owner notes: real vs demo wiring, routes, folder rules (hidden tests never leave `questions/`) |
| `index.ts` | Exports `DsaModule` |
| `dsa.module.ts` | Nest module. Picks the real or demo resolver/stores with `DSA_DEMO_MODE`; binds `CODE_RUNNER` to `PistonRunner` and the authoring client |
| `dsa.controller.ts` | Candidate routes `/dsa/:token/*`, including `continue` (calls `continueAfterCoding` and the GD link) |
| `dsa-hr.controller.ts` | HR monitoring review `/dsa-hr/applications/:id/proctoring`, scoped to the HR user's company |
| `dsa.service.ts` | Core logic: load the session, start, run, submit, finish, timer expiry, scoring, proctoring calls, state for the page (score hidden unless `DSA_SHOW_SCORE=on`) |
| `dsa.dto.ts` | Checks the run/submit body (language, code ≤ 64,000 chars); `dsaError()` helper |
| `dsa.tokens.ts` | `DSA_CLOCK` injection token (tests control time) |

#### `apps/api/src/modules/dsa/authoring`

The rounds editor can ask for coding problems. The AI worker writes them, or they are read from HR's file. For AI problems, the **expected outputs come from running the reference solution (Python) on Piston**, not from the model's text. A problem whose solution fails is dropped. Sentences that name a programming language are removed. Nothing is saved; HR reviews the problems and saves the round. Audit: `rounds.coding_problems_generated` / `_imported`.

| File | What it does |
|---|---|
| `coding-author.client.ts` | HTTP client for the ai-worker `POST /v1/rounds/coding-problems` (`AI_WORKER_URL`, timeout 120 s); maps errors (timeout, 429, 404 = outdated worker) |
| `coding-author.controller.ts` | `generate` and `import` routes (multipart field `file`) |
| `coding-author.service.ts` | Validates the request; CSV parsing (`CODING_CSV_COLUMNS`); runs the reference solutions on Piston; `languageNeutral()`; builds `AuthoredCodingProblem` with `dsaProblemSpecSchema` |

#### `apps/api/src/modules/dsa/execution`

| File | What it does |
|---|---|
| `code-runner.ts` | The `CodeRunner` interface (`runBatch`), the `CODE_RUNNER` token, and the errors `CodeRunnerUnavailableError` / `CodeRunnerBusyError` |
| `limiter.ts` | `ConcurrencyLimiter`: max parallel runs + a FIFO queue with a size limit and a timeout (throws Busy) |
| `piston.runner.ts` | `PistonRunner`: `POST {PISTON_URL}/api/v2/execute` with the limits for each language; `mapPistonResponse()` turns Piston's answer into a verdict (compile error, time limit, memory limit, runtime error) |

#### `apps/api/src/modules/dsa/evaluation`

| File | What it does |
|---|---|
| `evaluator.ts` | Runs the code for each test case and compares outputs (`exact` / `float` 1e-5 / `lines`). Turns Python syntax errors into compile errors. Hidden cases return only their status |
| `harness.ts` | Wraps the candidate's function for the bank problems (Python/Java/C++). Prints the result after a `__HF_RESULT__:` marker. Maps error line numbers back to the candidate's own code. For `stdio` problems the code runs as written |
| `sql-evaluator.ts` | Judges a SQL answer: the sandbox runs it on each dataset, then `compareResults()` checks it. A syntax error is reported once as a compile error; a setup failure is an internal error (not the candidate's fault) |

#### `apps/api/src/modules/dsa/proctoring`

Browser integrity monitoring. Event types: full screen exit, tab switch, blur, copy/paste, devtools, AI camera alerts. Each event has a severity and a weighted score. Events are merged and deduplicated. The status can be clean / warning / review / disqualified. The AI camera (MediaPipe) runs only in the browser; only alert events reach the API (see `docs/SQL_CAMERA_PROCTORING.md`).

| File | What it does |
|---|---|
| `proctoring.controller.ts` | Candidate routes `proctor/start`, `proctor/events`, `proctor/end` (they call `DsaService`) |
| `proctoring-admin.controller.ts` | `/dsa-admin/proctoring/*`, protected by the `x-dsa-admin-key` header (timing-safe compare); 404 when the key is not set |
| `proctoring.service.ts` | `ProctoringService`: consent, record events (server clock is clamped to the attempt window), open/close episodes, summary, candidate state (strikes used); `inviteRefFor(token)` |
| `proctoring.policy.ts` | `policyFromEnv()` (all `DSA_PROCTOR_*` vars), `cameraFor()`, `severityOf()`, `summarize()`, `mergedOutsideMs()`, `scoreOf()` |
| `proctoring.types.ts` | Event types, severity, session/summary shapes, consent versions (`dsa-proctoring-v2`, camera `…-camera-v3`), notice texts |
| `proctoring.dto.ts` | Checks the bodies. A body that tries to name a candidate or assessment is refused. Max 20 events per call; camera duration max 30 min |
| `proctoring.store.ts` | `ProctorStore` interface + `InMemoryProctorStore` (demo) |
| `prisma-proctor.store.ts` | `dsa_proctor_sessions` store. Saves only HR rounds (assessment id `<applicationId>.<roundId>`); `forApplication()` for HR |

#### `apps/api/src/modules/dsa/questions`

| File | What it does |
|---|---|
| `question.types.ts` | `DsaQuestion` (with hidden tests), languages (`python`, `java`, `cpp`, `sql`), harness kinds, `toPublicQuestion()` whitelist (only samples; SQL: tables + sample datasets) |
| `question.repository.ts` | `QuestionRepository` interface + `DSA_QUESTION_REPOSITORY` token |
| `bank-question.repository.ts` | Real repository: maps each round question to a bank, HR or SQL problem under the round's question id and marks; demo = the whole bank |
| `mock-question.repository.ts` | Old fixed repository (always the 3 bank problems); not bound in the module |
| `mock-questions.data.ts` | The problem bank: `dsa-001` Two Sum, `dsa-002` Longest Substring, `dsa-003` Median of Two Sorted Arrays, with starter code and hidden tests |
| `hr-problem.ts` | `fromHrProblem()`: HR's own problem → `stdio` problem (full program reads stdin) + starter code |
| `sql-problem.ts` | `fromSqlProblem()`: the round's copy of a SQL bank question → problem with language `sql`, one test case per dataset |

#### `apps/api/src/modules/dsa/session`

| File | What it does |
|---|---|
| `session.store.ts` | `DsaSession` / `QuestionProgress` types, the `SessionStore` interface, `InMemorySessionStore` (demo) |
| `engine-session.store.ts` | Real store: reads and writes the session on the engine's `round_attempts` row (start / save progress / finish) |
| `invitation.resolver.ts` | `DsaInvitation` type, `InvitationResolver` interface, `DemoInvitationResolver` (`demo-dsa-token`, `DSA_DEMO_TOKENS`; refused in production) |
| `round-invitation.resolver.ts` | Real resolver through `openCodingRound`; `assessmentIdFor()` = `<applicationId>.<roundId>`; only the first name is shown |

---

### `apps/api/src/modules/sql`

**What it does.** This module has two parts. The first is the company's **SQL question bank** for HR. The second is the **SQL sandbox** that runs queries safely. A SQL round has no own page or pipeline. A round question that holds a SQL problem counts as a coding problem, so the round engine and the dsa coding page run it like a DSA round. When HR saves a question, the API runs it, so expected results are computed and never typed by hand.

**How it connects.**
- Imports only `TenantsModule`. Exports `SQL_SANDBOX` and `SqlBankService`. The dsa module imports it for the sandbox.
- DB: `sql_questions` (`prisma/schema/sql_bank.prisma`). Rounds keep their own copy in `round_questions.sql_spec`, so edits and deletes never break a round.
- Sandbox: one `node -e` child process per run, using `node:sqlite`. No env is passed to it. It is killed at the time limit.
- Env: `FEATURE_SQL`, `SQL_RUNNER_CONCURRENCY` (4), `SQL_RUNNER_HEAP_MB` (64), `SQL_RUNNER_MAX_QUEUE` (100), `SQL_RUNNER_QUEUE_TIMEOUT_MS`, `SQL_RUNNER_STARTUP_MS` (5000). The camera is controlled by `DSA_PROCTOR_CAMERA` in the dsa module.
- CLI: `COMPANY_SLUG=<slug|all> pnpm --filter api sql:seed` adds the starter questions.
- Audit: `sql.question_*`.

**Main flow.**
1. HR writes a question: the schema, datasets (sample or hidden), a reference query, and an "order matters" flag. `POST /check` runs it without saving.
2. `buildProblem()` calls `SqlSandbox.inspect()`. The schema and every dataset must load, and the reference query must succeed on each one. The tables, columns, sample rows and expected results are read from the real database. If it fails: 422 `SQL_QUESTION_NOT_RUNNABLE` with `issues`.
3. HR picks active questions in the rounds editor. The round stores a copy (`sql_spec`).
4. A candidate submits a query on the coding page. `sqlStatements()` makes sure it is exactly one statement. For each dataset, the worker creates a fresh in-memory SQLite DB, loads the data, then locks it read-only (`query_only` + an authorizer that allows only SELECT/read/function/recursive). It returns at most 1000 rows.
5. `compareResults()` compares the result as a table: column count and names, values (numbers within 1e-6, NULL only equals NULL), duplicates, and row order only if required.

**HTTP endpoints** (prefix `/api/v1`, all HR, company from the session).

| Method | Path | Who | What |
|---|---|---|---|
| GET | `/sql-hr/questions` | HR `jobs.read` | The bank list (no datasets) |
| GET | `/sql-hr/questions/:id` | HR `jobs.read` | One question with datasets, expected results, reference query |
| POST | `/sql-hr/questions/check` | HR `jobs.write` | Run a question without saving |
| POST | `/sql-hr/questions` | HR `jobs.write` | Create (422 if it does not run) |
| PUT | `/sql-hr/questions/:id` | HR `jobs.write` | Replace |
| PATCH | `/sql-hr/questions/:id/active` | HR `jobs.write` | `{ active }`; inactive questions cannot be picked for new rounds |
| DELETE | `/sql-hr/questions/:id` | HR `jobs.write` | Delete (rounds keep their copy) |
| POST | `/sql-hr/questions/starter` | HR `jobs.write` | Add the 15 starter questions the company does not have yet |

**Files.**

| File | What it does |
|---|---|
| `README.md` | What is SQL-specific, the routes, env vars, tests |
| `index.ts` | Public exports: `SqlModule`, `SQL_SANDBOX`, `SqlSandbox`, `compareResults`, `formatTable`, `SqlBankService` |
| `sql.module.ts` | Nest module: controller, bank service, Prisma store, sandbox from env |
| `sql-hr.controller.ts` | HR routes `/sql-hr/questions*` behind `FEATURE_SQL` |
| `bank/build-problem.ts` | Runs HR's draft in the sandbox and builds the `SqlProblemSpec` (tables, sample data, expected results), or returns readable problems |
| `bank/sql-bank.service.ts` | CRUD, check, activate/deactivate, starter import (by `seedKey`, safe to repeat), audit |
| `bank/sql-bank.store.ts` | `SqlBankStore` abstract + `PrismaSqlBankStore` on `sql_questions` (always filtered by company) |
| `bank/starter-questions.ts` | 15 starter questions (5 easy / 5 medium / 5 hard): only the input; the expected output is computed |
| `engine/sql-sandbox.ts` | `SqlSandbox`: spawns the worker process, applies the concurrency limit (reuses dsa's `ConcurrencyLimiter`), kills it at the time limit, `query()` / `inspect()`, readable error messages |
| `engine/worker-source.ts` | The plain JS program run by `node -e`: fresh SQLite DB per dataset, read-only lock, row cap, heap limit |
| `engine/statements.ts` | Splits SQL at real statement ends (not inside strings or comments); `singleStatementProblem()` |
| `engine/compare.ts` | `cellsEqual`, `compareResults` (table compare with a reason), `formatTable` (text table) |

---

### `apps/api/src/modules/group-discussion`

**What it does.** This module runs `GROUP_DISCUSSION` rounds; the round's questions are the topics. Students who passed every earlier round wait in the round's **pool**. HR, or the system on its own, schedules an **event** that splits the pool into **groups** (batches). Each group has its own time slot, LiveKit room and topic. Each group runs live with an AI moderator bot. After the GD, each student's own-mic recording is transcribed with Whisper and scored with an LLM rubric. Then the selection is confirmed, automatically or by HR.

**How it connects.**
- Imports `TenantsModule`, `RoundsConfigModule`, `ApplicationsModule`, `AssessmentsModule`, `PipelineModule`, `RoundEngineModule`. It connects `GROUP_DISCUSSION` in `onModuleInit` when the feature is on. It exports `GdMailService`, which the dsa `continue` route uses.
- **Getting students:** `GdStore.poolCandidates()` is a raw SQL query. It finds students where every earlier, not-skipped round has a passed result (HR override first), or AI screening is `COMPLETED` when GD is the first round. Then `addToPool` adds them to `gd_participants`. The round engine's `nextStep()` returns `GROUP_DISCUSSION`, so no "HR will contact you" email is sent.
- **Giving results back:** `GdResultsService.confirm()` calls `assessments.record()` (interviewer score = AI overall scaled to the pass rule, or HR's rubric marks). It calls `assessments.override()` when the final decision differs. Then it calls `engine.continueAfterExternalRound()` for selected students. GD sends its own "selected" / rejection emails.
- DB (`prisma/schema/group_discussion.prisma`): `gd_events`, `gd_batches`, `gd_participants`, `gd_recordings`, `gd_participant_events`, `gd_private_talks`, `gd_hr_activity`, `gd_transcripts`, `gd_evaluations`, `gd_emails`.
- External: LiveKit (tokens, rooms, room metadata, agent dispatch `gd-moderator`), gd-agent on :8010 (Whisper on the CPU, Gemini rubric, ffmpeg mix), local disk for audio (`GD_STORAGE_DIR`, default `~/.hireflow/gd-recordings`), email.
- Env:
  - On/off and loops: `FEATURE_GROUP_DISCUSSION`, `GD_PROCESSING`, `GD_POLL_MS` (10 s), `GD_LIVE_POLL_MS` (1 s).
  - Automation: `AUTO_GD_SCHEDULE` (on), `AUTO_GD_MIN_STUDENTS` (6), `AUTO_GD_LEAD_MINUTES` (30), `AUTO_GD_CONFIRM` (on), `GD_RETENTION_DAYS` (90).
  - Workers and services: `GD_WORKER_URL`, `GD_WORKER_TIMEOUT_MS` (15 min), `GD_WORKER_SECRET`, `LIVEKIT_*`, `GD_STORAGE_DIR`.
  - Links: `GD_URL` / `CAREERS_SITE_URL`, `HR_DASHBOARD_URL`, `GD_ALLOW_LOCAL_LINKS` / `ROUND_ALLOW_LOCAL_LINKS`, `ENCRYPTION_KEY` (seals the link token so the reminder can reuse it).
- Rate limits `gd`, `gd-audio`. There are many audit actions (`gd.*`), and every HR action is also written to `gd_hr_activity`.

**Main flow.**
1. **Pool.** The slow loop in `gd.worker.ts` (every ~10 s) calls `fillPools()` → `gd_participants` (status WAITING).
2. **Schedule.** HR calls `POST rounds/:roundId/events`. Or `autoSchedule()` runs when at least 6 students wait; the first group starts 30 min later, and a failed attempt is retried after 30 min. The pool is split into groups with back-to-back slots (`planGroupSizes` / `planGroupSlots` in contracts). Topics go round the list (`assignTopics`), and each group gets a secret room name.
3. **Emails.** `GdMailService.queueDue()` / `sendDue()`: an invitation with a personal link (`/gd/{token}`; the token is made at send time), a reminder 15 min before, and HR's live link. Rows are claimed with `FOR UPDATE SKIP LOCKED`. Retries come after 1, 5 and 15 min.
4. **Live.** The live loop (~1 s) calls `GdLiveService.tickLive()`. It opens rooms at the start time and dispatches the moderator bot. After the grace minutes it locks the room (no-shows become ABSENT). Then it runs the phases INTRO → THINKING → DISCUSSION → SUMMARY → HR_QUESTIONS → CLOSE (`timeline.ts`). The state is pushed as LiveKit room metadata. HR speaking pauses the discussion timer. HR can start private talks, admit late students, or reschedule them.
5. **Recording.** Each student's browser records their own mic and uploads chunks (`recordings/:id/chunks`) to disk.
6. **Evaluate.** `GdResultsService.processNext()` claims a finished batch. It calls Whisper per recording, then `metrics.ts` builds one timeline and the speaking metrics. Private talks are excluded unless HR includes them. Then it calls `/gd/evaluate` per student. On failure it retries up to 5 times (after 2, 10, 30, 60 min).
7. **Confirm.** If `AUTO_GD_CONFIRM` is on, there is no GD rubric, and every evaluation exists, `autoConfirm()` runs. Otherwise HR clicks Confirm. Results are recorded and the next round / emails follow. Then the whole session audio is mixed (`/gd/mix` → `session.webm`).
8. **Retention.** Every hour, audio and transcripts older than `GD_RETENTION_DAYS` are deleted. Scores and decisions stay.

**HTTP endpoints** (prefix `/api/v1`).

| Method | Path | Who | What |
|---|---|---|---|
| GET | `/group-discussion/rounds` | HR `jobs.read` | GD rounds of the company with pool counts |
| GET | `/group-discussion/rounds/:roundId` | HR `jobs.read` | Round overview: pool, events, groups |
| POST | `/group-discussion/rounds/:roundId/events` | HR `jobs.write` | Schedule an event (split the pool into groups) |
| POST | `/group-discussion/events/:eventId/cancel` | HR `jobs.write` | Cancel an event that has not started |
| PUT | `/group-discussion/batches/:batchId/topic` | HR `jobs.write` | Change a group's topic |
| GET | `/group-discussion/batches/:batchId/live` | HR `jobs.read` | HR's LiveKit connection + live state |
| POST | `/group-discussion/batches/:batchId/speak` | HR `jobs.write` | HR starts/stops speaking (pauses/resumes the timer, HR tile visible) |
| POST | `/group-discussion/batches/:batchId/private-talks` | HR `jobs.write` | Start a private talk with one student |
| POST | `/group-discussion/private-talks/:talkId/end` | HR `jobs.write` | End a private talk |
| POST | `/group-discussion/batches/:batchId/participants/:participantId/admit` | HR `jobs.write` | Admit a late student |
| POST | `/group-discussion/participants/:participantId/reschedule` | HR `jobs.write` | Move an absent student to another group (new link) |
| POST | `/group-discussion/batches/:batchId/recordings` | HR `jobs.write` | Start HR's own audio recording |
| POST | `/group-discussion/recordings/:recordingId/chunks` | HR `jobs.write` | Upload an HR audio chunk |
| GET | `/group-discussion/batches/:batchId/results` | HR `jobs.read` | Transcripts, metrics, evaluations, decisions |
| GET | `/group-discussion/batches/:batchId/session-audio` | HR `jobs.read` | Stream the mixed session audio |
| POST | `/group-discussion/batches/:batchId/session-audio` | HR `jobs.write` | Rebuild the mixed audio |
| PUT | `/group-discussion/batches/:batchId/students/:participantId/decision` | HR `jobs.write` | HR override (selected / not) |
| PUT | `/group-discussion/batches/:batchId/students/:participantId/rubric` | HR `jobs.write` | HR rubric marks (when the job has a GD rubric) |
| PUT | `/group-discussion/private-talks/:talkId/include` | HR `jobs.write` | Include or exclude a private talk in the evaluation |
| POST | `/group-discussion/batches/:batchId/reprocess` | HR `jobs.write` | Run the evaluation again |
| POST | `/group-discussion/batches/:batchId/confirm` | HR `jobs.write` | Confirm the selection (once) |
| GET | `/public/gd/session` | student (header `x-gd-token`) | Lobby data |
| POST | `/public/gd/join` | student | Consent, then a LiveKit token (also used to reconnect) |
| GET | `/public/gd/state` | student | Live state |
| GET | `/public/gd/private` | student | Connection to a private talk with HR |
| POST | `/public/gd/events` | student | Client events (mic, connection…) |
| POST | `/public/gd/recordings` | student | Start an own-mic recording |
| POST | `/public/gd/recordings/:recordingId/chunks` | student (rate limit `gd-audio`) | Upload an audio chunk |

**Files.**

| File | What it does |
|---|---|
| `README.md` | Overview, file list, routes, flags, audit actions, folder rules |
| `index.ts` | Public exports: `GroupDiscussionModule`, `isGroupDiscussionRound`, `GdMailService`, `GdStudentLink`, `gdWorkerConfigFromEnv` |
| `group-discussion.module.ts` | Nest module; binds the LiveKit, storage and gd-agent client from env; connects `GROUP_DISCUSSION` |
| `planning.ts` | Pure: is it a GD round, its topics, a topic per group (round-robin with offset), random room names |
| `timeline.ts` | Pure: phase plan, `start`, `step` (no drift), `pause`/`resume` when HR speaks |
| `metrics.ts` | Pure: phase windows, removes private-talk lines, speaking metrics (talk time, words, fillers, interruptions), transcripts for the evaluator |
| `link.ts` | Student token (32 bytes, SHA-256 stored), `x-gd-token` header, link base from `GD_URL` / `CAREERS_SITE_URL`, HR live link |
| `emails.ts` | Invitation, reminder, HR live link and result emails (IST times; no scores or topic) |
| `gd.service.ts` | Scheduling: list rounds, overview, create/cancel event, change a topic, `scheduleAutomatically` |
| `gd-live.service.ts` | Student lobby/join/state/recording/events/private talk; HR live controls; `tickLive()` opens/locks rooms and runs phases |
| `gd-mail.service.ts` | The `gd_emails` queue and sender with retries; `studentLink()` for the coding page |
| `gd-results.service.ts` | Transcribe + evaluate pipeline, results view, overrides, rubric, include private talks, reprocess, auto/HR confirm, session audio, retention |
| `gd.worker.ts` | Two timers: live (~1 s) and slow (pools, auto schedule, emails, evaluation, hourly retention); pauses 5 min if tables are missing |
| `gd.store.ts` | `GdStore` abstract + `PrismaGdStore`: pool query, add to pool, rounds, participants, events (transaction; a student can never be in two groups), topics |
| `livekit.ts` | `LiveKitGateway`: join tokens, create rooms, room metadata, hide/show HR, dispatch the `gd-moderator` agent, remove a participant |
| `storage.ts` | `LocalGdAudioStorage`: chunk files `<tenant>/<batch>/<recording>/<seq>.chunk`, delete folders |
| `gd-worker.client.ts` | HTTP client for gd-agent `/gd/transcribe`, `/gd/evaluate` (checked with zod), `/gd/mix`; `x-worker-secret` header |
| `hr-gd.controller.ts` | HR routes `/group-discussion/*` |
| `public-gd.controller.ts` | Student routes `/public/gd/*` (token in the header) |

---

### `apps/api/src/modules/ai-interview`

**What it does.** This module runs `AI_INTERVIEW` rounds ("AI Technical Interview" in the round editor). Students who passed the earlier rounds get an email with a personal link to `/interview/[token]` on the careers site. There they have a spoken interview: the AI asks questions out loud (TTS) and the student answers with their voice. The questions are based on the job description, the student's parsed resume and HR's own questions. After the interview, the AI writes a report. A decision (automatic by pass mark, or by HR) is sent back to the round engine. The module also has "protection": tab, focus and full screen warnings, with termination after too many warnings.

**How it connects.**
- Imports `ApplicationsModule` (mailer, rate limits), `TenantsModule`, `RoundsConfigModule`, `RoundEngineModule`. Uses `SpeechCache` from the screening module. It connects `AI_INTERVIEW` in `onModuleInit` when `FEATURE_AI_INTERVIEW` is on.
- **Getting students:** the worker calls `engine.waitingForModule('AI_INTERVIEW', { limit: 100 })`, then `store.queueInvite()`.
- **Giving results back:** `AiInterviewHrService.record()` calls `engine.recordModuleResult(tenantId, 'AI_INTERVIEW', { applicationId, roundId, interviewerScore, answers, notes, … }, { passed, reason, actor })`. A decision that differs from the score is saved as HR's override. The engine sends the next invitation or the outcome email (only once).
- DB (`prisma/schema/ai_interview.prisma`): `ai_interview_invites`, `_sessions`, `_turns`, `_events`, `_reports`, `_violations`, `_protection_settings`.
- External: ai-worker :8000 → `/v1/interview/plan` (topic plan), `/turn` (transcribe the audio + propose the next question), `/speak` (WAV), `/report`. The worker uses Gemini. Email goes through `MAILER`.
- Env:
  - On/off and loop: `FEATURE_AI_INTERVIEW`, `FEATURE_AI_INTERVIEW_PROTECTION`, `AI_INTERVIEW_PROCESSING`, `AI_INTERVIEW_POLL_MS` (10 s).
  - Automation: `AUTO_AI_INTERVIEW_INVITES` (on), `AUTO_AI_INTERVIEW_DECISIONS` (on).
  - Links: `AI_INTERVIEW_URL` / `CAREERS_SITE_URL`, `AI_INTERVIEW_LINK_DAYS` (7), `ROUND_ALLOW_LOCAL_LINKS`.
  - AI worker: `AI_WORKER_URL`, `INTERVIEW_PLAN_TIMEOUT_MS` (55 s), `INTERVIEW_TURN_TIMEOUT_MS` (28 s; must stay under the careers-site proxy's 30 s), `SCREENING_SPEECH_TIMEOUT_MS`, `INTERVIEW_REPORT_TIMEOUT_MS` (115 s).
- Rate limits `interview`, `interview-answer`.

**Main flow.**
1. The worker tick (every 10 s) does these steps in order: `discover()` → `sendInvites()` → `sweep()` → `reports()` → `autoDecide()` → `recover()`.
2. **Invite.** Each send attempt makes a fresh token; only its SHA-256 is stored. The email is held (not failed) while there is no public link. On start, `resendDeadLinks()` re-sends invitations whose link points to an old tunnel address that no longer opens.
3. **Open.** The page sends the token in the header `x-interview-token`. On the first visit, `ensureSession()` copies the round config and the protection settings, so later HR edits do not change a running interview.
4. **Interview.** `start` → `ensurePlan()` (AI plan; `fallbackPlan()` if the AI is down). For each `answer`, the audio (≤ 8 MB) goes to `/turn`. `flow.nextQuestion()` decides if the AI's question is allowed (time phase, follow-up limit, no repeats); if not, it uses the plan. `speech?seq=` returns the question as WAV. The server owns the clock (pauses are capped; 60 s grace for a last answer).
5. **Protection.** The page reports incidents with its own id. `InterviewProtectionService` and the store decide, under a row lock, whether it is a new warning. Going over `maxWarnings` (default 2) means `TERMINATED`: the answers are kept, a report is made, and HR must review it.
6. **End.** The interview ends by `end`, by the time running out, or by `sweep()` for an idle page. Then `generateReport()` calls `/report`.
7. **Decide.** `autoDecide()` uses the AI score against the pass mark (`interviewerScore()`), but never for TERMINATED interviews. Otherwise HR decides on the dashboard, and needs a reason when overruling the score or deciding a terminated interview. `record()` hands the result to the engine. `recover()` retries if that step was interrupted.

**HTTP endpoints** (prefix `/api/v1`).

| Method | Path | Who | What |
|---|---|---|---|
| GET | `/public/interview/session` | candidate (header `x-interview-token`) | Page data: status, current question, clock, protection state |
| POST | `/public/interview/start` | candidate | Consent + start |
| POST | `/public/interview/answer` | candidate (multipart: `audio`, `seq`, `clientTurnId`, `durationMs`) | One spoken answer → the next question (safe to retry) |
| GET | `/public/interview/speech?seq=` | candidate | The question as `audio/wav` |
| POST | `/public/interview/repeat` | candidate | Count a "repeat the question" (limited) |
| POST | `/public/interview/pause` | candidate | Pause (device lost) |
| POST | `/public/interview/resume` | candidate | Resume |
| POST | `/public/interview/events` | candidate | Client events for HR's log |
| POST | `/public/interview/violation` | candidate | `{ incidentId, type, detail? }` → warning / termination |
| POST | `/public/interview/violation/ack` | candidate | `{ warningNumber }` acknowledged |
| POST | `/public/interview/end` | candidate | End early |
| GET | `/ai-interview/applications/:applicationId` | HR `jobs.read` | Transcript, events, integrity, report, decision per round |
| POST | `/ai-interview/applications/:applicationId/rounds/:roundId/decision` | HR `jobs.write` | Advance / reject (only once) |
| POST | `/ai-interview/applications/:applicationId/rounds/:roundId/report` | HR `jobs.write` | Retry a failed report |
| GET | `/ai-interview/jobs/:jobId/rounds/:roundId/protection` | HR `jobs.read` | Protection settings of the round |
| PUT | `/ai-interview/jobs/:jobId/rounds/:roundId/protection` | HR `jobs.write` | Save the protection settings |

**Files.**

| File | What it does |
|---|---|
| `README.md` | Overview, files, tables, routes, flags, audit actions |
| `index.ts` | Exports `AiInterviewModule` |
| `ai-interview.module.ts` | Nest module; binds the Prisma store, worker client, `SpeechCache`, worker config; connects `AI_INTERVIEW` |
| `flow.ts` | Pure rules: config snapshot, server clock (`clockOf`, pauses, 90 s final phase), `nextQuestion`, `fallbackPlan`, greeting/closing, `vocabulary`, `interviewerScore` |
| `ai-interview.service.ts` | Candidate side: `open` by token hash, start/resume/pause/events/repeat/end, `answer`, `speech`, `ensurePlan`, `generateReport`, `sweepTime` |
| `ai-interview-hr.service.ts` | HR view, `decide`, `autoDecide`, `retryReport`, `record` (→ `recordModuleResult`), `hrAnswers()` for the Assessments page |
| `protection.ts` | Pure protection rules: `qualifies`, `classifyViolation` (grouping within 3 s, device dedup, one counter), `accepts`, `protectionState`, termination text |
| `protection.service.ts` | `InterviewProtectionService`: report and acknowledge violations; get/save HR settings per round |
| `ai-interview.worker.ts` | Background loop (invite, send, sweep, reports, auto decide, recover); email retries after 1/5/15 min; dead-link resend at startup |
| `ai-interview.store.ts` | `AiInterviewStore` abstract + `PrismaAiInterviewStore`: all DB access (invites, sessions, turns with a sequence check, violations under a row lock, reports, decisions) |
| `ai-interview-worker.client.ts` | Typed HTTP client for the ai-worker `/v1/interview/plan`, `/turn`, `/speak`, `/report`, with a timeout for each call |
| `link.ts` | Token (32 bytes, SHA-256 stored), `x-interview-token`, link base from `AI_INTERVIEW_URL` / `CAREERS_SITE_URL`, `isPublicLink` |
| `emails.ts` | The invitation email (names the round just cleared, duration, link expiry; no scores) |
| `public-interview.controller.ts` | Candidate routes `/public/interview/*` |
| `hr-interview.controller.ts` | HR routes `/ai-interview/*` |

---

## apps/hr-dashboard and shared packages

### apps/hr-dashboard

**What it does**

This is the web app for HR people. It runs on port 3000 (`pnpm --filter hr-dashboard dev`).
HR uses it to:

- sign up, log in, register a company, and invite team members;
- fill the company profile and create jobs;
- let the AI write the job description (JD), edit it, and approve it;
- set up the assessment rounds and publish the job (careers site, LinkedIn);
- see applications, shortlist students, and follow every round (online tests, DSA/SQL coding, group discussion, AI interview, personal interview).

A Platform Admin also uses it (`/admin`) to see and suspend companies.

Every page is a thin server `page.tsx` that only reads route params / query and renders one client component (`'use client'`). All data loading happens in the browser.

**How it connects**

- **Only the HireFlow API.** The browser never talks to the AI worker or Gemini. No AI keys exist in this app.
- **Base URL.** `src/lib/api.ts` uses `NEXT_PUBLIC_API_URL`, and if it is empty, the relative path `/api/v1`.
- **Proxy.** `next.config.ts` rewrites `/api/v1/:path*` to `API_INTERNAL_URL` (default `http://localhost:4000/api/v1`). So the browser calls the dashboard's own origin. The cookie stays first-party and there is no CORS. This also works through a Cloudflare tunnel (`allowedDevOrigins: *.trycloudflare.com`).
  Note: the app `README.md` says the default is `http://localhost:4000/api/v1`. The code actually defaults to `/api/v1` through the proxy.
- **Auth.** The API sets an **httpOnly session cookie**. The front-end never sees a token. Every request sends `credentials: 'include'`.
- **CSRF.** Every request sends the header `x-requested-with: hireflow`. The API rejects writes without it. File uploads (`uploadHrChunk`, `importCodingProblems`) add this header by hand.
- **Responses are checked.** `request()` in `api.ts` parses each answer with the zod schema from `@hireflow/contracts`. A bad shape becomes `ApiError('INVALID_RESPONSE')`. Error codes are turned into friendly text (the `FRIENDLY` map).
- **401 handling.** Any 401 (except `/auth/*`) fires the window event `hireflow:unauthenticated`. `SessionProvider` then sends the user to `/login?next=...`.
- **Session gate (`src/components/session.tsx`).** It loads `GET /auth/me`. Public pages: `/login`, `/signup`, `/invite/*`, `/forgot-password`, `/reset-password/*`. A user with no company is sent to `/register-company`. A Platform Admin with no company is sent to `/admin`. A suspended company sees a notice. `/team` is for Company Admins only. This gate is only for display; the API checks every permission again and takes the company from the session.
- **Live video.** The GD live page uses LiveKit (`@livekit/components-react`, `livekit-client`). The API returns the LiveKit URL and token.
- **Second API client.** `lib/publisher-api.ts` has a generic `call()` helper (same cookie + CSRF rules). All the newer `*-api.ts` files use `call()` from there, not `request()` from `api.ts`.

**Pages**

The header menu shows: Jobs, New job, Publish, Applications, Shortlisted Students, Assessments (a drop-down menu per job), Company profile, Team (Company Admin), Companies (Platform Admin). Other pages are opened from links.

| Route | File | What HR does there | API calls |
|---|---|---|---|
| `/` | `src/app/page.tsx` | Redirects to `/jobs`. | - |
| `/login` | `src/app/login/page.tsx`, `login-form.tsx` | Log in with email + password. If login 2FA is on, a second step asks for the 6-digit email code. "Continue with Google" button. Reads `?next`, `?reset=done`, `?oauth_error`. After login goes to `next`, or `/jobs`, `/admin` or `/register-company`. | `POST /auth/login`, `POST /auth/login/verify`, `GET /auth/providers`, browser link to `/auth/google/start` |
| `/signup` | `src/app/signup/page.tsx`, `signup-form.tsx` | A new HR person creates an account (step A). With email verification on, a code step follows. | `POST /auth/signup`, `POST /auth/signup/verify`, Google button |
| `/register-company` | `src/app/register-company/page.tsx`, `register-company.tsx` | Step B: fill the company profile. The company is created and the user becomes its Company Admin. | `POST /companies` |
| `/forgot-password` | `src/app/forgot-password/page.tsx`, `forgot-password-form.tsx` | Enter email to get a reset link. Same answer whether the account exists or not. | `POST /auth/password/forgot` |
| `/reset-password/[token]` | `src/app/reset-password/[token]/page.tsx`, `reset-password-form.tsx` | Choose a new password from the emailed link. Logs out every device. | `POST /auth/password/reset` |
| `/invite/[token]` | `src/app/invite/[token]/page.tsx`, `accept-invitation.tsx` | Join a company from an invitation: new users create an account, existing users confirm their password. | `GET /auth/invitations/:token`, `POST /auth/invitations/accept` |
| `/admin` | `src/app/admin/page.tsx`, `companies-admin.tsx` | Platform Admin: list all registered companies. | `GET /admin/companies` |
| `/admin/companies/[id]` | `src/app/admin/companies/[id]/page.tsx`, `company-detail.tsx` | Platform Admin: one company's status and team; suspend / activate. Jobs and JDs are not shown on purpose. | `GET /admin/companies/:id`, `PATCH /admin/companies/:id` |
| `/team` | `src/app/team/page.tsx`, `team-manager.tsx` | Company Admin: see members and invitations, invite a member, revoke an invite, change role, disable access. | `GET /company/members`, `POST /company/invitations`, `DELETE /company/invitations/:id`, `PATCH /company/members/:userId` |
| `/company` | `src/app/company/page.tsx`, `company-editor.tsx` | Step 1: company profile. Company Admin edits; HR Member sees it read-only. `?next=` says where to go after save. | `GET /company-profile`, `PUT /company-profile` |
| `/jobs` | `src/app/jobs/page.tsx`, `jobs-list.tsx` | List of jobs with status (Draft / Awaiting approval / Approved). Delete a job. Warns when the company profile is missing. | `GET /jobs`, `GET /company-profile`, `DELETE /jobs/:id` |
| `/jobs/new` | `src/app/jobs/new/page.tsx`, `new-job.tsx` | Step 2: job form. The saved company profile is shown. HR also sets the private "Minimum ATS score". On save, opens `/jobs/[id]?generate=1`. | `GET /company-profile`, `POST /jobs` (form + `screening`) |
| `/jobs/[id]` | `src/app/jobs/[id]/page.tsx`, `job-review.tsx` | Steps 3 + 4: generate the JD (auto when `?generate=1`), edit it (new version), regenerate, version history + restore, edit job requirements, verify company details, approve. Also edit the minimum ATS score. | `GET /jobs/:id`, `GET /company-profile`, `POST /jobs/:id/jd/generate`, `PATCH /jobs/:id/jd`, `PATCH /jobs/:id`, `POST /jobs/:id/approve`, `GET`/`PUT /ats/jobs/:id/settings` |
| `/publish` | `src/app/publish/page.tsx`, `publish-board.tsx` | Every approved job. Open the publish dialog (set rounds, preview posts, publish). Retry, close / reopen applications, take posts down, update apply links, delete the job. Company Admin: turn channels on/off, connect / disconnect LinkedIn (`?linkedin=` after the OAuth callback). | `GET /publisher/jobs`, `GET /publisher/channels`, `POST /publisher/jobs/:id/{retry,close,reopen,refresh-links}`, `DELETE /publisher/jobs/:id`, `PUT /publisher/channels/:channel/settings`, link to `/publisher/linkedin/connect`, `DELETE /publisher/linkedin/connection`, `DELETE /jobs/:id`, plus the dialog calls (see `publish-dialog.tsx`) |
| `/applications` | `src/app/applications/page.tsx`, `applications-board.tsx` | Pick a published job, see everyone who applied, see the ATS screening at a glance, download resumes. | `GET /publisher/jobs`, `GET /applications/jobs/:jobId`, resume download link |
| `/shortlisted` | `src/app/shortlisted/page.tsx`, `shortlisted-board.tsx`, `shortlisted.ts` | Per job, tabs: Shortlisted / Not shortlisted / Needs HR review / Being screened / All. Preview resume in the page, set HR's decision, retry screening, send shortlist / rejection emails (HR clicks; the ATS only suggests). Shows the AI voice screening panel. | `GET /ats/jobs`, `GET /ats/jobs/:id/candidates?view=`, `PUT /ats/applications/:id/decision`, `POST /ats/applications/:id/retry`, `POST /ats/emails`, resume blob download, plus `ai-screening-panel` calls |
| `/assessments` | `src/app/assessments/page.tsx`, `assessments-board.tsx`, `gd-recordings.tsx`, `assessment-styles.ts` | Per job (`?job=`, optional `?round=`): every student after AI screening, ATS score, AI screening status, result in each round, pass / fail counts, pipeline stage and Stage 2 %. CSV export. Links to the student page, to `/sql-questions`, and to GD results. | `GET /assessments/jobs`, `GET /assessments/jobs/:jobId`, `GET /pipeline/jobs/:jobId/statuses`, `GET /group-discussion/rounds/:roundId` |
| `/assessments/[applicationId]` | `src/app/assessments/[applicationId]/page.tsx`, `student-review.tsx` | One student in one place: resume score, AI screening notes, every round with answers, online test details, AI interview report. HR can override pass / fail with a reason. | `GET /assessments/applications/:id`, `PUT /assessments/applications/:id/rounds/:roundId/override`, `GET /round-engine/applications/:id`, plus panel calls |
| `/sql-questions` | `src/app/sql-questions/page.tsx`, `sql-bank-board.tsx` | The company's SQL question bank: write schema, datasets and reference query; the server runs it and shows the tables and expected results before save. Activate / deactivate, delete, add starter questions. (Not in the menu; linked from Assessments.) | `GET`/`POST /sql-hr/questions`, `GET`/`PUT`/`DELETE /sql-hr/questions/:id`, `POST /sql-hr/questions/check`, `PATCH /sql-hr/questions/:id/active`, `POST /sql-hr/questions/starter` |
| `/group-discussion` | `src/app/group-discussion/page.tsx`, `gd-board.tsx` | Per GD round (`?round=`): students waiting in the pool, a form to split them into groups with time slots and topics, scheduled events, cancel an event, change a group's topic. Links to live and results pages. | `GET /group-discussion/rounds`, `GET /group-discussion/rounds/:id`, `POST /group-discussion/rounds/:id/events`, `POST /group-discussion/events/:id/cancel`, `PUT /group-discussion/batches/:id/topic` |
| `/group-discussion/live/[batchId]` | `src/app/group-discussion/live/[batchId]/page.tsx`, `live-dashboard.tsx` | HR watches one group live (LiveKit video grid). Camera / mic, "Speak to group" (timer pauses, HR's voice is recorded), private talk with one student, admit or move absent students. Side panel: attendance, talk time, flags, activity log. | `GET .../batches/:id/live`, `POST .../batches/:id/speak`, `POST .../batches/:id/private-talks`, `POST /group-discussion/private-talks/:id/end`, `POST .../batches/:id/participants/:pid/admit`, `POST /group-discussion/participants/:pid/reschedule`, `POST .../batches/:id/recordings`, `POST /group-discussion/recordings/:id/chunks` |
| `/group-discussion/results/[batchId]` | `src/app/group-discussion/results/[batchId]/page.tsx`, `results-board.tsx` | Results of one group: session audio, ranked students with AI scores and evidence quotes, HR decision override, rubric marks, include / exclude private talks, reprocess, rebuild audio, and "Confirm selection" (results recorded, emails sent). | `GET .../batches/:id/results`, `PUT .../students/:pid/decision`, `PUT .../students/:pid/rubric`, `PUT /group-discussion/private-talks/:id/include`, `POST .../batches/:id/reprocess`, `POST .../batches/:id/session-audio` (rebuild), `GET .../batches/:id/session-audio`, `POST .../batches/:id/confirm` |
| `/personal-interview` | `src/app/personal-interview/page.tsx`, `pi-board.tsx` | Per job (`?job=`): only students who cleared the technical interview, best score first, all round scores, Stage 2 total, interview remarks. HR sets call status and notes. CSV export. | `GET /assessments/jobs`, `GET /pipeline/jobs/:id/personal-interview`, `PUT /pipeline/jobs/:id/personal-interview/:applicationId` |

(`...` = `/group-discussion`.)

**Other files in `src/app`**

| File | What it does |
|---|---|
| `src/app/layout.tsx` | Root layout. Wraps every page in `SessionProvider`. Sets the page title. |
| `src/app/globals.css` | Tailwind v4 import and colour tokens (`--accent`, `--danger`, ...) with a dark-mode version. |
| `src/app/favicon.ico` | Browser tab icon. |
| `src/app/assessments/assessment-styles.ts` | Shared badge styles and labels for round outcomes and stages; `scoreText()`. |
| `src/app/assessments/gd-recordings.tsx` | Under a GD round on Assessments: session recording of every finished group, with a link to its results. |
| `src/app/shortlisted/shortlisted.ts` | Pure helpers: the tab list (`VIEWS`), `emailKindFor()`, `canEmail()` (when HR may send a decision email). |

**Components** (`src/components`)

| File | What it does |
|---|---|
| `session.tsx` | `SessionProvider`: loads `/auth/me`, login redirect, role gate, header with menu, company switcher (`POST /auth/company`) and logout (`POST /auth/logout`). Exports `useSession`, `useRefreshSession`, `isCompanyAdmin`. |
| `assessments-menu.tsx` | The "Assessments" drop-down in the header. Shows one job's process step by step: AI screening, then only the rounds HR chose. Remembers the last job in `localStorage`. Calls `GET /assessments/jobs`, `GET /rounds/jobs/:id`. |
| `ui.tsx` | Small UI kit: `Field`, `Button`, `Card`, `StatusBadge`, `ErrorNotice`, `Spinner`, `inputClass`, `describedBy`. |
| `google-button.tsx` | "Continue with Google". Hidden unless `GET /auth/providers` says Google is set up. Also error texts for `?oauth_error=`. |
| `company-profile-form.tsx` | The company profile form (used by `/company` and `/register-company`). |
| `company-card.tsx` | `CompanyFacts`: read-only view of company facts sent to the AI. |
| `job-form-panel.tsx` | The job form (title, skills, salary, work mode...). Used on `/jobs/new` and `/jobs/[id]`. |
| `tag-input.tsx` | Type a value, press Enter or comma to add a tag; pasting "a, b, c" adds three. |
| `screening-score-field.tsx` | "Minimum ATS score" input (0-100). |
| `screening-settings-card.tsx` | Job page card to change the private minimum ATS score at any time. `GET`/`PUT /ats/jobs/:id/settings`. |
| `jd-panel.tsx` | `JdPanel` (view / edit / regenerate the JD, copy as text) and `JdView` (read-only). |
| `version-history.tsx` | List of JD versions (AI drafts and HR edits) to preview and restore. |
| `approval-panel.tsx` | Step 4: verify the company details used in the JD, then approve. |
| `delete-job.tsx` | `confirmAndDeleteJob()` (confirm, then `DELETE /jobs/:id`), `PostsNotRemovedNotice` (posts HR must remove by hand), `dangerButton` style. |
| `invite-link.tsx` | Shows a new invitation link once, to copy. |
| `team-table.tsx` | Table of members and pending invitations (Team page and admin company page). |
| `sql-result-table.tsx` | Small scrollable table for SQL data and results. |
| `ai-screening-panel.tsx` | AI voice screening of one application: status, transcript, structured notes, HR review mark. No score. `GET /screening/applications/:id`, `PUT .../review`, `POST .../summary`. |
| `ai-interview-panel.tsx` | AI technical interview of one student: report, transcript, device events, protection incidents, HR decision (advance / reject; reason needed if it goes against the AI score). `GET /ai-interview/applications/:id`, `POST .../rounds/:roundId/decision`, `POST .../rounds/:roundId/report`. |
| `online-test-panel.tsx` | One online test round for one student: invite email, attempt, proctoring events, DSA browser observations, integrity decision. `POST /round-engine/applications/:id/rounds/:roundId/{revoke,resend,integrity-decision}`, `GET /dsa-hr/applications/:id/proctoring`. |
| `gd/gd-ui.tsx` | GD room parts shared with the careers site: topic banner, phase tracker, timer on server clock, video tiles, `VideoGrid` that fits all tiles on one screen. |
| `gd/session-recording.tsx` | Audio player + download for the whole GD session (`/group-discussion/batches/:id/session-audio`). |
| `publish/publish-dialog.tsx` | Publish flow for one job: edit rounds (save draft or complete), preview the post for every channel, publish, retry. `GET`/`PUT /rounds/jobs/:id`, `POST /publisher/jobs/:id/preview`, `POST .../publish`, `POST .../retry`. Also post status styles. |
| `rounds/rounds-editor.tsx` | List of rounds of one job (add, reorder, remove) plus pipeline settings. |
| `rounds/round-card.tsx` | One round: name, type, description, time limit, pass rule, questions, AI generate, skip. Changing the pass rule after lock: `PUT /assessments/jobs/:id/rounds/:roundId/pass-rule`. |
| `rounds/question-editor.tsx` | One question (MCQ, true/false, text, coding). Can pick a DSA bank problem or a SQL bank question (`GET /sql-hr/questions`, `GET /sql-hr/questions/:id`), or write HR's own DSA problem with test cases. |
| `rounds/ai-generate-panel.tsx` | "Generate with AI" for a round: HR edits and picks suggestions. `POST /rounds/jobs/:id/generate`. |
| `rounds/coding-author-panel.tsx` | DSA problems from the AI or from HR's PDF / Word / CSV file. `POST /dsa-hr/jobs/:id/coding-problems/generate`, `POST /dsa-hr/jobs/:id/coding-problems/import`. |
| `rounds/gd-rubric-settings.tsx` | GD scoring rubric (parameters and max marks). `GET`/`PUT /pipeline/jobs/:id/rounds/:roundId/rubric`. |
| `rounds/interview-protection-settings.tsx` | AI interview protection rules (warnings, device policy). Saved at once. `GET`/`PUT /ai-interview/jobs/:id/rounds/:roundId/protection`. |
| `rounds/pipeline-settings.tsx` | Stage 2 overall rule over DSA / SQL / GD and rejection emails. `GET`/`PUT /pipeline/jobs/:id/settings`. |
| `rounds/round-skip.tsx` | "Skip round" / "Undo skip" for a saved round. `POST /rounds/jobs/:id/rounds/:roundId/{skip,unskip}`. |

**lib** (`src/lib`)

| File | What it does |
|---|---|
| `api.ts` | Main client: `request()`, `ApiError`, friendly error texts, `API_URL`, 401 event. Auth, team, admin, company profile, jobs, JD generate / edit / approve. |
| `publisher-api.ts` | Generic `call()` helper (reused by other clients) and publisher calls: jobs, channels, preview, publish, retry, close, reopen, unpublish, refresh links, LinkedIn, `listJobApplications`, `resumeDownloadUrl`. |
| `ats-api.ts` | ATS: screening settings, screening jobs, candidates by view, decision, retry, decision emails, `fetchResumeBlob` (resume as Blob for in-page preview), `parseMinAtsScore`. |
| `screening-api.ts` | AI voice screening (`/screening/...`). |
| `rounds-api.ts` | Rounds (`/rounds/jobs/:id`), AI questions, skip / unskip, assessments (`/assessments/...`), pass-rule change, DSA coding problem generate / import. |
| `round-engine-api.ts` | Online test per student (`/round-engine/...`) and DSA proctoring review. |
| `ai-interview-api.ts` | AI interview view, decision, retry report, protection settings. |
| `gd-api.ts` | All group discussion calls, HR audio chunk upload, session audio URL. |
| `pipeline-api.ts` | Pipeline settings, GD rubric, statuses, personal interview board; `downloadCsv()` (Excel-friendly CSV). |
| `sql-bank-api.ts` | SQL question bank (`/sql-hr/questions`); `activeSqlQuestions()` caches the active list for the editor. |
| `rounds.ts` | Rounds editor state (no network): draft types, `newRound`, `fromSaved`, `toRequest`, `validate`, pass-mark helpers, `HR_ROUND_TYPES` (Aptitude, Coding, SQL, GD), labels. |
| `job-form.ts` | Job form draft (strings) <-> `JobForm`; validation with the shared `jobFormSchema`. |
| `company-form.ts` | Company draft <-> `CompanyProfile`; validation; `readSnapshot`, `sameCompany` (did the profile change since the JD?). |
| `jd.ts` | JD helpers: version label, section list, `tidyJd`, `jdProblems`, `jdToText`. |
| `gd-recorder.ts` | `ChunkRecorder`: records HR's mic in 5 s chunks and uploads in order with retry. Same as the careers-site recorder. |
| `roles.ts` | Role labels (Company Admin / HR Member). |
| `safe-next.ts` | `safeNext()`: allows only same-site `/path` values for `?next=` (no open redirect). |

**Tests** (`tests/`, Vitest, node environment)

| File | What it does |
|---|---|
| `tests/api.test.ts` | Company profile, job workflow and session client calls (fake `fetch`). |
| `tests/ats-api.test.ts` | `parseMinAtsScore` and the ATS client. |
| `tests/company-form.test.ts` | Company draft validation and snapshot helpers. |
| `tests/jd.test.ts` | JD helpers. |
| `tests/job-form.test.ts` | Job form validation, deadline rules. |
| `tests/publisher-api.test.ts` | Publisher client. |
| `tests/rounds.test.ts` | Rounds editor state and rounds client. |
| `tests/shortlisted.test.ts` | Who can get a decision email; ATS lists / decisions / emails. |
| `tests/sql-bank.test.ts` | SQL round in the editor; SQL bank client. |

**Config files**

| File | What it does |
|---|---|
| `package.json` | Scripts: `dev`/`start` on port 3000, `build`, `lint`, `typecheck`, `test`. Deps: Next 15.5, React 19.1, LiveKit, `@hireflow/contracts`. |
| `next.config.ts` | `/api/v1/*` proxy to `API_INTERNAL_URL`; allows `*.trycloudflare.com` in dev. |
| `tsconfig.json` | TypeScript config; alias `@/*` -> `src/*`. |
| `eslint.config.mjs` | ESLint with `next` config. |
| `postcss.config.mjs` | Tailwind v4 PostCSS plugin. |
| `vitest.config.mts` | Runs `tests/**/*.test.ts`; skips PostCSS in tests. |
| `.gitignore` | Standard Next.js ignores. |
| `README.md` | Short page list and run commands (some parts are older than the code). |

### packages/contracts

**What it does**

The shared "language" of the system. Zod schemas, TypeScript types, limits, and pure rule functions. The API uses them to validate input. The front-ends use them to validate forms and API answers. Built with `tsc` to `dist/` (the package `main` is `dist/index.js`, so it must be built before apps use it; `dist/` is not in git).

**Who uses it**

`apps/api`, `apps/hr-dashboard`, `apps/careers-site`, `apps/candidate-app` (all list `@hireflow/contracts` as a dependency).

**Files**

| File | What it does |
|---|---|
| `package.json` | Name `@hireflow/contracts`; only dependency `zod`; scripts `build`, `dev` (watch), `typecheck`, `test`. |
| `tsconfig.json` | `src` -> `dist`, node16 modules, declarations on. |
| `src/index.ts` | Re-exports every schema file and the events. |
| `src/events/envelope.ts` | `eventEnvelope(type, data)`: the outer shape of every event (`id`, `type`, `version`, `tenant_id`, `occurred_at`, `correlation_id`, `producer`, `data`). |
| `src/events/index.ts` | Event schemas: `job.created`, `application.received`, `ats.scored`, `job.published`, `job.publish_failed`. Only the publisher events are used today (by `apps/api/src/modules/publisher/publisher.events.ts`). |
| `src/schemas/auth.ts` | Roles (`COMPANY_ADMIN`, `HR_MEMBER`), company / membership status, password limits, login / signup (with 2FA code step), session, password reset, invitations, team members, admin company views. |
| `src/schemas/company.ts` | Company profile schema and limits, company sizes, saved profile record. Limits are mirrored in `ai_worker/schemas.py`. |
| `src/schemas/job.ts` | Job form schema and limits, JD content, edited JD, job status, JD versions, approval, job summary / detail, edit / approve requests, JD validation. Limits mirrored in the AI worker. |
| `src/schemas/resume.ts` | Parsed profile and ATS score (M10 output, used by `ats.scored` event). |
| `src/schemas/publisher.ts` | Channels (a channel is any string key, e.g. `careers`, `linkedin`), post status, publish request, preview, publish / unpublish / refresh results, delete-job result, public job (careers site), publishable job, channel info. |
| `src/schemas/application.ts` | Apply form (with phone, consent version), resume limits (PDF and .docx only), OTP send / verify, receipt, "my data" requests, HR view of applications. |
| `src/schemas/ats.ts` | HR-only ATS: private minimum score settings, parsed resume, screen result, decisions, decision email kinds and status, candidate views and counts, screened candidate, screening jobs. |
| `src/schemas/screening.ts` | AI voice screening (M12): fixed question steps, statuses, public session for the candidate link, answers, extraction, HR summary / detail and review status. No score. |
| `src/schemas/rounds.ts` | Round configurator (M05): round types, question kinds, difficulty, pass modes and rules, question and round inputs, completion checks, AI interview rules, save request, saved rounds, skip, AI question generation. Only round names become public. |
| `src/schemas/dsa-bank.ts` | DSA problems: built-in `DSA_BANK`, HR's own problem spec with test cases, `dsa:<id>` topic marker, coding problem generate / import shapes, CSV template and `parseCsv`, proctoring review type. |
| `src/schemas/sql-bank.ts` | SQL question bank: limits, cells, tables, datasets, question input, problem spec (copied into the round), check result, starter import result. |
| `src/schemas/round-engine.ts` | Online test rounds (M13): which round types run online, consent, questions / answers, attempt status, protection settings and incidents, integrity decision, public view (never shows correct answers), proctor events, per-application round engine view. |
| `src/schemas/assessments.ts` | Round results sent by the round engine, outcomes, stages, per-job board rows, student review, HR override, pass-rule change result. HR only. |
| `src/schemas/group-discussion.ts` | GD (M18): settings, group size / slot planning (`planGroupSizes`, `planGroupSlots`), events, batches, topics, LiveKit identities and connection, live state and phases, private talks, audio recording, client events, HR live view, evaluation criteria, results and decisions. |
| `src/schemas/ai-interview.ts` | AI technical interview: statuses, consent, protection settings and violations, questions, transcript, public session, answer evaluation, report dimensions, recommendation, HR view and decision. The AI score is advice only. |
| `src/schemas/protection.ts` | `countProtectionIncident()`: the one shared warning counter (warning 1..N, then terminate; groups incidents within a short window). Used by the API. |
| `src/schemas/pipeline.ts` | Hiring pipeline: stages (Aptitude -> Technical rounds -> Tech interview -> Personal interview), Stage 2 rule and verdict, pipeline settings, GD rubric and totals, pipeline statuses, personal-interview call status and board. |
| `src/schemas/camera-proctoring.ts` | Pure webcam rules: head pose / eye gaze from face points, per-frame classify against a calibrated baseline, grouped alerts with cool-down. Meant for SQL rounds. Today only imported by `apps/api` (`dsa/proctoring/proctoring.types.ts`) and its test. |
| `tests/application.test.ts` | Apply form rules. |
| `tests/auth.test.ts` | Auth schemas. |
| `tests/camera-proctoring.test.ts` | Gaze / alert rules. |
| `tests/company.test.ts` | Company profile schema. |
| `tests/dsa-bank.test.ts` | DSA spec, bank, CSV parsing. |
| `tests/job.test.ts` | Job form schema. |
| `tests/pipeline.test.ts` | Pipeline rules and rubric. |
| `tests/rounds.test.ts` | Round questions and pass rules. |
| `tests/sql-bank.test.ts` | SQL question schema. |

### packages/rounds

**What it does**

**Empty placeholders.** Each folder holds only a `README.md` that says "One RoundPlugin. Implements the interface from round-sdk." No code. The real round logic lives in `apps/api` modules.

**Who uses it**

Nobody.

**Files**

| File | What it does |
|---|---|
| `packages/rounds/aptitude/README.md` | Placeholder for an aptitude round plugin. |
| `packages/rounds/coding/README.md` | Placeholder for a coding round plugin. |
| `packages/rounds/communication/README.md` | Placeholder for a communication round plugin. |
| `packages/rounds/gaming/README.md` | Placeholder for a gaming round plugin. |
| `packages/rounds/group-discussion/README.md` | Placeholder for a GD round plugin. |
| `packages/rounds/sql/README.md` | Placeholder for a SQL round plugin. |
| `packages/rounds/website/README.md` | Placeholder for a website round plugin. |

### packages/round-sdk

**What it does** - **Empty placeholder.** README says it will hold the `RoundPlugin` interface (M13). No code.

**Who uses it** - Nobody.

| File | What it does |
|---|---|
| `packages/round-sdk/README.md` | Plan and folder rules only. |

### packages/ui

**What it does** - **Empty placeholder.** README says "Shared React components (shadcn/ui)". No code. Each app has its own `components/ui.tsx` instead.

**Who uses it** - Nobody.

| File | What it does |
|---|---|
| `packages/ui/README.md` | Plan and folder rules only. |

### packages/config

**What it does** - **Empty placeholder.** README says "Shared ESLint/TS/Tailwind configs". No code. Each app has its own config files.

**Who uses it** - Nobody.

| File | What it does |
|---|---|
| `packages/config/README.md` | Plan and folder rules only. |

### packages/email-templates

**What it does** - **Empty placeholder.** README says "React Email templates (rejection, invite, offer)" (M24). No code. Emails are built inside `apps/api`.

**Who uses it** - Nobody.

| File | What it does |
|---|---|
| `packages/email-templates/README.md` | Plan and folder rules only. |

### packages/proctor-sdk

**What it does** - **Empty placeholder.** README says "Browser proctoring client library" (M21). No code. Proctoring code lives in the apps and in `packages/contracts` (`camera-proctoring.ts`, `protection.ts`).

**Who uses it** - Nobody.

| File | What it does |
|---|---|
| `packages/proctor-sdk/README.md` | Plan and folder rules only. |

---

## Candidate-facing apps: careers-site, candidate-app, admin-console

Candidates (students) never log in. They open **personal links** that HR or the API sends by email. Two Next.js 15 apps serve these links:

- `apps/careers-site` (port 3300): job pages, apply form, "my data", and the token rounds: AI screening, online MCQ test, AI interview, group discussion (GD).
- `apps/candidate-app` (port 3001): the DSA / SQL coding round.
- `apps/admin-console`: empty placeholder.

### How token links work (both apps)

1. The API makes a random token and emails a link like `https://<tunnel>/interview/<token>`.
2. The page file (`page.tsx`) reads `token` from the URL and passes it to a client component. The token is kept **in memory only**.
3. **careers-site:** every API call sends the token in a **header**, never in the API URL:
   `x-screening-token`, `x-assessment-token`, `x-interview-token`, `x-gd-token`.
4. **candidate-app is different:** the token is **in the API URL path**: `/api/v1/dsa/<token>/...`.
5. Every write also sends `x-requested-with: hireflow`. The API uses this header as its CSRF check.
6. `next.config.ts` adds headers on token pages: `Referrer-Policy: no-referrer`, `Cache-Control: no-store`, `X-Robots-Tag: noindex`, and a `Permissions-Policy` that allows only the camera / microphone each page needs.

### How the browser reaches the API (proxy)

- The browser calls **its own site**, not `localhost:4000`. This is needed because the site is opened through a Cloudflare quick tunnel on the candidate's computer.
- careers-site rewrites `/api/v1/public/:path*` to `${API_INTERNAL_URL}/public/:path*`. Only public routes are forwarded; HR routes are not reachable.
- candidate-app rewrites `/api/v1/dsa/:path*` to `${API_INTERNAL_URL}/dsa/:path*`. `/dsa-admin/*` and the code runner are not reachable.
- Server-side rendering (job pages) calls the API directly with `API_INTERNAL_URL` (default `http://localhost:4000/api/v1`).
- `scripts/start-public.ps1` opens one tunnel to :3300 (written to `CAREERS_SITE_URL`). When a code runner is set up (`PISTON_URL`), it opens a second tunnel to :3001 (written to `CANDIDATE_APP_URL`).

---

### apps/careers-site

**What it does**

- Shows a company's live jobs and the full job description (JD), server-rendered on every request.
- Apply form: phone OTP check, details, three consents, resume upload with a progress bar.
- "My data" page (DPDP law): see or delete your own data after a phone OTP.
- Hosts four token rounds: AI voice screening (M12), online MCQ assessment (M13), AI technical interview (camera + mic), and live group discussion (M18, LiveKit video).
- Proctoring on the test and interview: tab switch, window blur, fullscreen exit, and on-device face counting (MediaPipe). The **server** counts warnings; the page only reports incidents and shows the server's warnings.

**How it connects**

- API base in the browser: `/api/v1` (same origin, proxied). Override: `NEXT_PUBLIC_CAREERS_API_URL`.
- `src/middleware.ts` runs on `/api/v1/public/*` before the proxy. It replaces `X-Forwarded-For` with the value of the header named in `TRUSTED_CLIENT_IP_HEADER` (for example `cf-connecting-ip`), or removes it. So a student cannot fake their IP for the API's per-IP rate limits.
- Shared types and zod schemas come from `@hireflow/contracts` (same schemas the API uses).
- Other env: `CAREERS_SITE_URL` (canonical URL in job page metadata), `NEXT_PUBLIC_TURNSTILE_SITE_KEY` (Cloudflare CAPTCHA on OTP send; empty = off).
- GD video uses LiveKit (`@livekit/components-react`, `livekit-client`). The API returns the LiveKit URL and room token.

**Pages**

| Route | File | What the candidate does there | API calls (all via `/api/v1/public/*` proxy, except SSR) |
|---|---|---|---|
| `/` | `src/app/page.tsx` | Static text: "open the job link you received". | none |
| (404) | `src/app/not-found.tsx` | "Page not found" with a link home. | none |
| `/[company]` | `src/app/[company]/page.tsx` | List of the company's LIVE jobs; each links to the job page. | SSR: `GET /public/:company/jobs` |
| `/[company]/jobs/[job]` | `src/app/[company]/jobs/[job]/page.tsx` + `apply-form.tsx` | Reads the JD (facts, responsibilities, skills, selection process). If LIVE, fills the apply form: phone + OTP, details, consents, resume. Sees a receipt at the end. If closed, sees "no longer open". | SSR: `GET /public/:company/jobs/:job`. Browser: `POST /public/otp/send`, `POST /public/otp/verify`, `POST /public/applications` (multipart, XHR for progress) |
| `/[company]/my-data` | `src/app/[company]/my-data/page.tsx` + `my-data-panel.tsx` | Verifies phone (or reuses the proof saved right after applying), sees their applications, can delete them. | `POST /public/otp/send`, `/public/otp/verify`, `POST /public/my-data`, `POST /public/my-data/delete` |
| `/screening/[token]` | `src/app/screening/[token]/page.tsx` + `screening-room.tsx` | AI voice screening. Welcome + consent, mic check, then a loop: hears the question (server TTS), answers out loud, answer is uploaded, next question comes back. Can end early. Refresh continues where it stopped. | Header `x-screening-token`. `GET /public/screening/session`, `POST .../start`, `GET .../speech?seq=N` (WAV), `POST .../answer` (multipart audio), `POST .../end` |
| `/assessment/[token]` | `src/app/assessment/[token]/page.tsx` + `exam-room.tsx` | Online MCQ test. Landing page -> system check (camera preview, fullscreen) -> timed exam with a question palette, "mark for review", autosave -> submit. Warnings pop up from the server; too many = terminated. Camera video is only a preview, never recorded. | Header `x-assessment-token`. `GET /public/assessment/session`, `POST .../start`, `PUT .../answers` (with revision), `POST .../submit`, `POST .../events` (batched every 4 s), `POST .../violation`, `POST .../violation/ack` |
| `/interview/[token]` | `src/app/interview/[token]/page.tsx` + `interview-room.tsx` | AI technical interview. Welcome + consent, device setup (camera + mic), then a voice loop like screening, with a live transcript preview, automatic end-of-answer on silence, "repeat question", pause on device / connection loss, resume after reload. Proctoring as in the assessment. | Header `x-interview-token`. `GET /public/interview/session`, `POST .../start`, `.../resume`, `.../pause`, `.../end`, `.../repeat`, `GET .../speech?seq=N`, `POST .../answer`, `.../events`, `.../violation`, `.../violation/ack` |
| `/gd/[token]` | `src/app/gd/[token]/page.tsx` + `gd-room.tsx`, `gd-lobby.tsx`, `gd-ui.tsx` | Group discussion. Lobby (facts, camera preview, mic meter, consent, countdown) -> live video room (topic, phase tracker, timer, tiles, raise hand) -> maybe a private talk with HR -> thank-you. Own mic is recorded in 5 s chunks and uploaded. | Header `x-gd-token`. `GET /public/gd/session` (polled in lobby), `POST .../join` (returns LiveKit connection), `GET .../state` (polled every 5 s / 3 s), `GET .../private`, `POST .../events`, `POST .../recordings`, `POST .../recordings/:id/chunks` |

**App files (components inside `src/app`)**

| File | What it does |
|---|---|
| `src/app/layout.tsx` | Root layout: header with the site name from `content.ts`. |
| `src/app/globals.css` | Tailwind v4 import and colour variables (`--background`, `--surface`, ...). |
| `src/app/favicon.ico` | Site icon. |
| `src/app/[company]/jobs/[job]/apply-form.tsx` | Client apply form. Uses `PhoneVerify`, validates with `lib/apply-form.ts`, uploads with progress, maps API field errors to inputs, saves the phone proof for the my-data page, handles "job closed meanwhile". |
| `src/app/[company]/my-data/my-data-panel.tsx` | Client panel: phone OTP (or saved proof), shows data, delete with confirm. |
| `src/app/screening/[token]/screening-room.tsx` | Whole screening state machine (phases from `lib/screening-ui.ts`), audio playback, recording, re-send of the same answer on errors. |
| `src/app/assessment/[token]/exam-room.tsx` | Whole MCQ exam (about 1200 lines): camera check, fullscreen, server-clock timer, debounced autosave (700 ms), offline draft, event batching, incident outbox (retried every 10 s), face counting, warning dialog. |
| `src/app/interview/[token]/interview-room.tsx` | Whole AI interview (about 1300 lines): device setup, speak/listen/process loop, `EndOfAnswer` silence detection, live transcript, pause/resume, protection monitor, face counting, warning dialog. |
| `src/app/gd/[token]/gd-room.tsx` | GD page controller: lobby / room / private / done modes, `LiveKitRoom`, polling fallback, audio chunk recorder, client events (JOINED, CAMERA_ON/OFF, MIC_ON/OFF, TAB_HIDDEN, HAND_RAISED, LEFT). |
| `src/app/gd/[token]/gd-lobby.tsx` | Lobby: GD facts, camera preview, mic level meter, consent box, countdown. Join is disabled until camera, voice and consent are OK. |
| `src/app/gd/[token]/gd-ui.tsx` | Shared GD pieces: topic banner, phase tracker, server-clock timer, video tile, "with HR" tile, video grid; reads live state from LiveKit room metadata. |

**Components and content**

| File | What it does |
|---|---|
| `src/components/phone-verify.tsx` | Phone + OTP widget, used by apply form and my-data. Optional Turnstile CAPTCHA. Shows `devCode` when the API returns it (dev mode). |
| `src/components/ui.tsx` | Small UI kit: `Field`, `Button`, `Spinner`, `inputClass`, ARIA helpers (`describedBy`). |
| `src/content.ts` | All user-facing text for job page, form, consents (with `CONSENT_POLICY_VERSION`), my-data. Change wording here. |

**lib**

| File | What it does |
|---|---|
| `src/lib/api.ts` | `API_URL` choice (server vs browser), `ApiError`, `getJob`, `getJobs`, `sendOtp`, `verifyOtp`, `submitApplication` (XHR), `viewMyData`, `deleteMyData`. |
| `src/lib/apply-form.ts` | Form draft type, validation with the shared `applyFormSchema`, resume check (`RESUME_LIMITS`), builds the multipart body. |
| `src/lib/phone-proof.ts` | Keeps the OTP proof in `sessionStorage` (this tab only) so my-data does not ask for a second OTP. |
| `src/lib/client-ip.ts` | Builds a safe `X-Forwarded-For` from the trusted edge header (used by middleware). |
| `src/lib/screening-api.ts` | Screening API client (`x-screening-token`), `sendAnswer` multipart, `newClientTurnId` (makes re-sends idempotent). |
| `src/lib/screening-ui.ts` | Screening phases, answer length limits, error -> message + retry kind (`resend` / `record` / `reload`). |
| `src/lib/recorder.ts` | Mic recording with `MediaRecorder` (WebM-Opus, MP4 fallback), loudness meter, silence peak, mic error messages. |
| `src/lib/assessment-api.ts` | MCQ test API client (`x-assessment-token`), uses `keepalive` on writes. |
| `src/lib/assessment-ui.ts` | Pure exam logic: answer map, palette state, server-clock countdown, local draft in `localStorage` keyed by a SHA-256 hash of the token, error texts. |
| `src/lib/interview-api.ts` | AI interview API client (`x-interview-token`). |
| `src/lib/interview-ui.ts` | Interview phases, pause texts, device errors, `EndOfAnswer` (ends an answer after 4.5 s silence following at least 2.5 s of speech). |
| `src/lib/interview-protection.ts` | `ProtectionMonitor` (turns blur / hidden / fullscreen signals into ONE incident per act, 2 s blur grace), `FaceEpisodeTracker` (2+ faces for 5 s = one incident), warning texts. Used by interview and assessment. |
| `src/lib/live-transcript.ts` | Browser Web Speech API preview of what the candidate says. Never saved. |
| `src/lib/camera.ts` | Camera support checks, error messages, small video-only stream, permission watcher, fullscreen check. |
| `src/lib/face-counter.ts` | MediaPipe face detector on the device; returns only the number of faces. Loads WASM from `/mediapipe/wasm` and model from `/models/`. |
| `src/lib/gd-api.ts` | GD API client (`x-gd-token`). |
| `src/lib/gd-recorder.ts` | `ChunkRecorder`: records own mic track, uploads 5 s chunks in order, retries with back-off. |
| `src/middleware.ts` | Next middleware on `/api/v1/public/*`: fixes `X-Forwarded-For` (see above). |

**Tests** (Vitest, node env, `pnpm --filter careers-site test`)

| File | What it does |
|---|---|
| `tests/apply-form.test.ts` | Form validation: verified form OK, OTP needed, errors on right inputs, empty score, bad `source`, resume checks. |
| `tests/assessment.test.ts` | Answer choosing, mark/clear, palette counts, stale drafts, server-clock timer, error and camera messages. |
| `tests/client-ip.test.ts` | Trusted IP header logic. Also checks confirmation email wording says "queued", not "sent". |
| `tests/interview-protection.test.ts` | Tab switch / blur / fullscreen grouping into single incidents, monitoring off cases, multiple-face episodes. |
| `tests/interview.test.ts` | End-of-answer silence logic, error mapping, device errors, clock format, missing live transcript. |
| `tests/screening.test.ts` | Recording MIME choice, upload names, mic errors, error -> retry kind, answer id format. |

**Config and other files**

| File | What it does |
|---|---|
| `package.json` | Scripts: `dev`/`start` on port 3300, `test` (vitest), `lint`, `typecheck`. `predev`/`prebuild` copy the face detector WASM. |
| `next.config.ts` | API rewrite for `/api/v1/public/*`, security headers per token page, `allowedDevOrigins` for trycloudflare / ngrok. |
| `turbo.json` | Extends root; `typecheck` also depends on `build`. |
| `tsconfig.json` | TypeScript config, `@/*` -> `src/*`. |
| `vitest.config.mts` | Vitest: `tests/**/*.test.ts`, `@` alias, skips PostCSS. |
| `eslint.config.mjs` | ESLint with `next/core-web-vitals` + `next/typescript`. |
| `postcss.config.mjs` | Tailwind v4 PostCSS plugin. |
| `scripts/copy-face-detector.mjs` | Copies `@mediapipe/tasks-vision/wasm` to `public/mediapipe/wasm` (git-ignored), so no CDN is used. |
| `public/models/blaze_face_short_range.tflite` | Committed face detector model (about 230 KB). |
| `README.md` | Short owner notes: pages, files to edit, env vars. Covers only the job/apply pages. |
| `.gitignore` | Standard Next ignores plus `/public/mediapipe/`, `.env*`, `next-env.d.ts`. |

---

### apps/candidate-app

**What it does**

- One route: the DSA / SQL coding round at `/dsa/[token]`.
- Candidate reads problems, picks a language (Python, Java, C++, or SQL for SQL problems), writes code, presses **Run** (sample tests) or **Submit** (sample + hidden tests). Code runs on the backend runner, never in the browser.
- Proctoring: fullscreen required, key shortcuts blocked (copy, paste, devtools, print ...), tab switch / blur reported, and optional AI camera (MediaPipe Face Landmarker on the device). The server counts strikes and can end the round.
- At the end it shows the result (or no score for an HR round) and asks the API what comes next: another coding round, an emailed link, or a group discussion link. It can redirect there automatically.

**How it connects**

- Browser API base: `/api/v1` (same origin). Override: `NEXT_PUBLIC_CANDIDATE_API_URL`. Server-side: `API_INTERNAL_URL`.
- Rewrite: `/api/v1/dsa/:path*` -> `${API_INTERNAL_URL}/dsa/:path*`. `experimental.proxyTimeout` is 240 s, because a Submit can wait in the runner queue for up to about 2 minutes.
- Token goes **in the URL path** of API calls (`/dsa/<token>/run`), not in a header. Writes send `X-Requested-With: hireflow`.
- Code drafts are saved in `sessionStorage` under `hf-dsa:<token>` (raw token as key).
- Types are hand-written in `src/lib/dsa-api.ts` to mirror `apps/api/src/modules/dsa`. Only `CameraEventType`, `CameraRuleEngine`, `FACE_POINTS` come from `@hireflow/contracts`.

**Pages**

| Route | File | What the candidate does there | API calls (via `/api/v1/dsa/*` proxy) |
|---|---|---|---|
| `/dsa/[token]` | `src/app/dsa/[token]/page.tsx` -> `assessment.tsx` | Intro screen (rules, consent, camera check) -> enters fullscreen -> timed coding: problem panel, editor, results panel -> Finish -> result screen -> "what next" (auto-redirect to next round or GD link). | `GET /dsa/:token`, `POST /dsa/:token/proctor/start` (consent), `POST /dsa/:token/start`, `POST /dsa/:token/run`, `POST /dsa/:token/submit`, `POST /dsa/:token/finish`, `POST /dsa/:token/proctor/events`, `POST /dsa/:token/continue` |

**Files in `src/app/dsa/[token]/`**

| File | What it does |
|---|---|
| `page.tsx` | Reads `token` and renders `<Assessment>`. |
| `assessment.tsx` | Main controller: loads state, starts proctoring and the round, language choice, drafts in sessionStorage, run / submit, finish, fullscreen and camera gates. |
| `code-editor.tsx` | Simple editor: a `<textarea>` with a highlighted layer under it (SQL highlighting), no external editor library. |
| `problem-panel.tsx` | Shows title, difficulty, marks, description, formats, constraints, samples, and SQL tables / sample data. |
| `results-panel.tsx` | Shows verdict, passed / total, per-case status, time, memory; hidden cases show only label and status; SQL results as tables. |
| `sql-table.tsx` | Renders a table of rows (max 50 by default). |
| `screens.tsx` | Loading, error, intro (rules + consent + camera check), and result screens. Result screen calls `/continue` and polls while a GD group is forming. |
| `proctoring.tsx` | Fullscreen helpers, `useProctoring` hook (listens to keys, blur, visibility, fullscreen; sends events in a queue; reacts to server strike level and "ended"), `FullscreenGate`, `ProctorNotice`. |
| `proctoring-rules.ts` | `classifyKey`: which key combos are blocked/reported; notice texts per event type. |
| `camera-monitor.tsx` | `useCameraMonitor`: webcam + MediaPipe Face Landmarker; reduces frames to 4 points + eye scores; shared `CameraRuleEngine` turns them into grouped alerts. `CameraCheck`, `CameraTile`, `CameraGate`. **This is the camera code that is used.** |
| `face-model.ts` | Loads Face Landmarker from `/mediapipe/` (CPU), `toMeasures` keeps only needed points; hides MediaPipe "INFO:" console noise. |
| `webcam.tsx` | Older camera proctoring (BlazeFace detector, NO_FACE / MULTIPLE_FACES). **Not imported anywhere: dead code.** |

**Other source files**

| File | What it does |
|---|---|
| `src/lib/dsa-api.ts` | API client and all DSA types (questions, results, proctoring, next step), `languageFor`, `formatClock`. |
| `src/lib/camera.ts` | Camera support checks, messages, `startCamera` (640x480), permission follow / watch. |
| `src/components/ui.tsx` | Small UI kit: `Button`, `Spinner`, `Badge`, `Card`, `inputClass`. |
| `src/app/layout.tsx` | Root layout with "HireFlow AI - Coding Assessment" header. |
| `src/app/globals.css` | Tailwind v4 and colour variables. |
| `src/app/favicon.ico` | Site icon. |

**Config, scripts and assets** (no tests in this app)

| File | What it does |
|---|---|
| `package.json` | `dev`/`start` on port 3001, `typecheck`. `predev`/`prebuild` run both face scripts. No `test` or `lint` script. |
| `next.config.ts` | DSA rewrite, 240 s proxy timeout, security headers on `/dsa/:token*`, `allowedDevOrigins`. |
| `tsconfig.json` | TypeScript config. **Excludes `camera-monitor.tsx` and `face-model.ts` from typecheck.** |
| `postcss.config.mjs` | Tailwind v4 PostCSS plugin. |
| `next-env.d.ts` | Next.js type stub (committed here, ignored in careers-site). |
| `tsconfig.tsbuildinfo` | Empty, committed by mistake. |
| `scripts/copy-face-detector.mjs` | Copies MediaPipe WASM to `public/mediapipe/wasm`. |
| `scripts/prepare-face-model.mjs` | Copies WASM again and downloads `face_landmarker.task` (3.7 MB) from Google storage once, checked against a pinned SHA-256. If offline, camera-monitored rounds cannot start. |
| `public/models/blaze_face_short_range.tflite` | BlazeFace model; only used by the dead `webcam.tsx`. |
| `.gitignore` | Ignores `/public/mediapipe/`. |
| `.gitkeep` | Leftover empty file. |

---

### apps/admin-console

| File | What it does |
|---|---|
| `apps/admin-console/.gitkeep` | Empty placeholder. There is no admin console code yet. The HR UI lives in another app (port 3000). |

---

## Workers, prompts, scripts, infrastructure and root files

This part covers everything outside `apps/` and `packages/`: the two Python workers, the prompt files, the start
scripts, Docker and Piston, CI, editor settings and the root config files.

---

### `workers/ai-worker`

**What it does.** A Python FastAPI service on port **8000**. It does every Gemini call of HireFlow except the GD
evaluation: write a JD, check a JD, read and score a resume, run the AI voice screening, run the AI technical
interview, suggest round questions and write DSA coding problems. It saves **nothing**. It only answers HTTP
requests from the API.

**How it connects.**
- The NestJS API calls it with `AI_WORKER_URL` (default `http://localhost:8000`). Each API module has its own small
  typed client (see the table below).
- It reads the repo-root `.env` at start (`config.load_env()`, finds the root by looking for `pnpm-workspace.yaml`).
  Real environment variables win over `.env`.
- It loads prompts from `prompts/*.yaml` (`PROMPTS_DIR` can change the folder).
- All model calls go through one door: `ai_worker/llm_gateway` (`LLMProvider.generate_json`). Only `gemini` is
  supported. Text-to-speech goes through `ai_worker/speech`.
- Start: `cd workers/ai-worker; uv sync; uv run python -m ai_worker` (uvicorn, `0.0.0.0:8000`, reload on). Swagger:
  `http://localhost:8000/docs`.

**Main flow of one AI request** (same pattern in every endpoint):
1. FastAPI checks the JSON body with Pydantic. A bad body = 422 before any AI call.
2. The endpoint loads the prompt YAML (`load_prompt`, cached) and builds the provider. A missing key only fails here,
   after input validation (`get_llm_provider` returns a factory on purpose).
3. A **time budget** starts (for example 80 s for a JD). Every retry and fallback must fit inside it, so the worker
   always answers before the API stops waiting.
4. User text is put inside `<tags>` and escaped (`escape_data`) so it cannot break out of the prompt.
5. Gemini is called with `response_mime_type=application/json` and a Pydantic `response_schema`.
6. On a 5xx or 429 the gateway tries the next model in `LLM_FALLBACK_MODELS`. 429 is never retried on the same model.
7. The JSON is parsed and validated. Invalid output gets **one** retry (`MAX_ATTEMPTS = 2`), then `LLM_INVALID_OUTPUT`.
8. The code then checks and cleans the answer (drops anything off-limits). Fake content is never returned.
9. Errors become `{"detail": {"code", "message"}}` with a safe message (`llm_gateway/errors.py`): 503
   `LLM_NOT_CONFIGURED`, 429 `LLM_RATE_LIMITED` (+ `Retry-After`), 504 `LLM_TIMEOUT`, 503 `LLM_UNAVAILABLE`, 502
   `LLM_AUTH_FAILED`, 502 `LLM_INVALID_OUTPUT`, 422 `LLM_BLOCKED`. Resume problems = 422 `RESUME_UNREADABLE`, bad
   audio = 422 `AUDIO_UNREADABLE`.

**Endpoints** (all in `ai_worker/main.py`)

| Method | Path | Called by | What |
|---|---|---|---|
| GET | `/health` | health checks, `start-public.ps1` (port check only) | `{status: ok, service: ai-worker}` |
| POST | `/v1/jd/generate` | `apps/api/src/modules/jobs/ai-worker.client.ts` | M02. Job form + optional company profile -> `JdContent`. Prompt `jd_generate.v3`. Headers `X-LLM-Model`, `X-Prompt-Version` |
| POST | `/v1/jd/validate` | `apps/api/src/modules/publisher/jd-checks.ts` (before publish) | M03. **Rules only, no LLM yet**: title, summary, responsibilities, requirements must not be empty |
| POST | `/v1/resume/screen` | `apps/api/src/modules/ats/ats-worker.client.ts` (ATS background loop) | M10 + M11. Base64 resume (PDF/DOCX) + job + JD -> parsed resume + explainable ATS score. Scanned PDF -> OCR fallback. Header `X-Resume-Text-Source: file\|scan` |
| POST | `/v1/screening/turn` | `apps/api/src/modules/screening/screening-worker.client.ts` | M12. One spoken answer (base64 audio) -> transcript + follow-up decision (+ reply to the candidate's question) |
| POST | `/v1/screening/speak` | same screening client | M12. Interviewer text -> `audio/wav` (Gemini TTS). Header `X-TTS-Model` |
| POST | `/v1/screening/extract` | same screening client | M12. Whole transcript -> structured notes for HR (no score, no decision) |
| POST | `/v1/interview/plan` | `apps/api/src/modules/ai-interview/ai-interview-worker.client.ts` | AI technical interview. Resume + job + HR questions -> list of topics |
| POST | `/v1/interview/turn` | same ai-interview client | One spoken answer -> raw + normalised transcript + next question (when `decide` is true) |
| POST | `/v1/interview/speak` | same ai-interview client | Same TTS as the screening |
| POST | `/v1/interview/report` | same ai-interview client | Saved transcript -> evidence-based report for HR (advice only) |
| POST | `/v1/rounds/questions` | `apps/api/src/modules/rounds-config/question-generator.client.ts` | M05. Question suggestions for one round. HR reviews them; nothing saved |
| POST | `/v1/rounds/coding-problems` | `apps/api/src/modules/dsa/authoring/coding-author.client.ts` | DSA round. `generate` new problems or `extract` them from HR's PDF/Word file. Each has a Python reference solution + test inputs; the API runs it on Piston to get the expected outputs |

**Important details per area**
- **JD** (`jd/generator.py`): the job form and the company profile go in two separate data tags. `aboutCompany` is
  forced to `""` when no company is sent.
- **Resume / ATS** (`parsing/`, `scoring/ats.py`): text comes from pypdf / python-docx (first 10 pages, max 20,000
  chars). Under 80 chars = "no text" -> if it is a PDF and `RESUME_OCR_FALLBACK` is on, Gemini reads the page images
  (`resume_transcribe.v1`), and `promptVersion` becomes `resume_screen.v1+resume_transcribe.v1`. The model only
  **reads** the resume. The score is a fixed formula in code: skills 40 (must-have 35 + nice-to-have 5), experience
  20, projects 20, achievements 10, qualifications 10. Same reading = same score.
- **Screening and interview turns** use **two separate calls** on purpose: (1) transcribe from the audio alone (no
  question, no history) so the model cannot "invent" an answer for silence; (2) decide the next step from text only.
  "Thinking" is off by default (`*_THINKING_BUDGET=0`, about 2x faster). If the interview decision fails, the
  transcript is still returned and the API moves to the next planned topic.
- **Interview normalisation** (`interview/normalize.py`): rule-based spelling fixes for tech terms ("post gres" ->
  "PostgreSQL"). The raw transcript is always kept.
- **TTS** (`speech/gemini_tts.py`): model `SCREENING_TTS_MODEL` (default `gemini-3.8-flash-tts`), voice `Kore`, with
  its own fallback list because TTS quotas are small and per model. Output is converted to WAV.

**Files**

| File | What it does |
|---|---|
| `workers/ai-worker/README.md` | How to run, test, lint; endpoint status; config table; JD error codes |
| `workers/ai-worker/pyproject.toml` | Python >= 3.12; deps FastAPI, uvicorn, pydantic, python-dotenv, google-genai, pyyaml, pypdf, python-docx; ruff + pytest settings; `integration` marker |
| `workers/ai-worker/uv.lock` | Locked dependency versions for `uv sync` |
| `ai_worker/__init__.py` | Empty package marker |
| `ai_worker/__main__.py` | `python -m ai_worker` -> uvicorn on `0.0.0.0:8000` with reload |
| `ai_worker/main.py` | The FastAPI app: all routes, error handlers, prompt/provider dependencies |
| `ai_worker/config.py` | Loads root `.env`; `Settings` with all env values, budgets and defaults; hides the key in `repr` |
| `ai_worker/schemas.py` | Pydantic mirror of `packages/contracts` (JobForm, CompanyProfile, JdContent, resume screen request/result). Must change in the same PR as contracts |
| `ai_worker/llm_gateway/__init__.py` | `build_provider()` - picks the provider from `LLM_PROVIDER` (only `gemini`) |
| `ai_worker/llm_gateway/provider.py` | `LLMProvider` protocol, `LLMResult`, `Attachment` (file sent with the prompt) |
| `ai_worker/llm_gateway/gemini.py` | Gemini provider: JSON output, deadline-bounded timeouts, SDK retries on 5xx only, model fallback on 5xx/429, error mapping, metadata-only logs |
| `ai_worker/llm_gateway/prompts.py` | Loads prompt YAML into `PromptSpec` (`$name` templates), `escape_data`, `parse_json_object` |
| `ai_worker/llm_gateway/errors.py` | Safe error classes with HTTP status + code |
| `ai_worker/jd/__init__.py` | Exports `generate_jd`, `PROMPT_FILE` |
| `ai_worker/jd/generator.py` | M02 JD generation: builds data blocks, calls the model, one retry, validates `JdContent` |
| `ai_worker/parsing/__init__.py` | Exports text extraction + OCR fallback |
| `ai_worker/parsing/extract.py` | PDF/DOCX bytes -> clean text; raises `ResumeUnreadableError` / `ResumeHasNoTextError` |
| `ai_worker/parsing/ocr.py` | Scanned PDF -> Gemini reads the page images (`resume_transcribe.v1`) |
| `ai_worker/scoring/__init__.py` | Exports `screen_resume`, `WEIGHTS` |
| `ai_worker/scoring/ats.py` | M11: model reading of the resume + the fixed 100-point formula with reasons |
| `ai_worker/screening/__init__.py` | Exports the M12 screening functions and models |
| `ai_worker/screening/interview.py` | M12 turn (transcribe, then decide), audio MIME checks, final extraction, output checks |
| `ai_worker/screening/models.py` | M12 request/response shapes (mirror `packages/contracts/src/schemas/screening.ts`) |
| `ai_worker/interview/__init__.py` | Exports the interview functions and models |
| `ai_worker/interview/engine.py` | Interview plan, turn (transcribe + decide), repeat-question check, report; drops unverifiable output |
| `ai_worker/interview/models.py` | Interview request/response shapes (report mirrors `ai-interview.ts`) |
| `ai_worker/interview/normalize.py` | Rule-based spelling of technical terms in the transcript |
| `ai_worker/rounds/__init__.py` | Exports M05 question generation |
| `ai_worker/rounds/generator.py` | M05 question suggestions + clean-up |
| `ai_worker/rounds/models.py` | M05 shapes (round types: APTITUDE, TECHNICAL_MCQ, CODING, SQL, ...) |
| `ai_worker/rounds/coding.py` | DSA problems: `generate` / `extract` from a document (max 40,000 chars), reference solutions, de-duplication |
| `ai_worker/speech/__init__.py` | `build_speech_provider()` from settings |
| `ai_worker/speech/provider.py` | `SpeechProvider` protocol + `SpeechResult` |
| `ai_worker/speech/gemini_tts.py` | Gemini TTS with model fallback; converts PCM to WAV |
| `ai_worker/evaluation/__init__.py` | Empty placeholder (not used yet) |
| `ai_worker/feedback/__init__.py` | Empty placeholder (not used yet) |
| `tests/conftest.py` | Blocks real LLM / TTS in every unit test; `fake_llm` fixture |
| `tests/fakes.py` | `FakeProvider` that returns prepared JSON |
| `tests/test_health.py` | `/health` |
| `tests/test_config.py` | Repo-root detection, prompts dir, `.env` loading, key hidden in repr |
| `tests/test_gemini_provider.py` | Gemini error mapping and request shape with a fake SDK client |
| `tests/test_jd.py` | Basic JD generate + validate (empty responsibilities rejected) |
| `tests/test_jd_generate.py` | `/v1/jd/generate` behaviour with a fake provider |
| `tests/test_jd_company.py` | Company profile is passed into the JD prompt |
| `tests/test_jd_budget.py` | The JD time budget covers retries + fallbacks |
| `tests/test_jd_integration.py` | Real Gemini call; runs only with `RUN_LLM_INTEGRATION=1` |
| `tests/test_resume_screen.py` | Text extraction, score formula, `/v1/resume/screen` |
| `tests/test_screening.py` | `/v1/screening/turn`, `/speak`, `/extract` |
| `tests/test_interview.py` | `/v1/interview/plan`, `/turn`, `/speak`, `/report` |
| `tests/test_round_questions.py` | M05 question shape, clean-up, safety |
| `tests/test_coding_problems.py` | DSA problem generation and extraction |

---

### `workers/gd-agent`

**What it does.** The Python side of the **M18 AI group discussion**. It has two processes:
1. **GD worker API** on `127.0.0.1:8010`: Whisper transcription, rubric evaluation with Gemini, and one mixed audio
   file of the whole session.
2. **AI moderator bot**: a LiveKit Agents worker named `gd-moderator`. It joins the group's video room and speaks
   with Piper (free offline TTS).

Everything is free and local except Gemini (evaluation) and LiveKit Cloud (the room).

**How it connects.**
- The API calls the worker through `apps/api/src/modules/group-discussion/gd-worker.client.ts` (`GD_WORKER_URL`,
  default `http://localhost:8010`, timeout `GD_WORKER_TIMEOUT_MS` = 15 min). If `GD_WORKER_SECRET` is set, the API
  sends it in the `x-worker-secret` header.
- The worker reads audio files **directly from disk** (`GD_STORAGE_DIR`, default `~/.hireflow/gd-recordings`), so it
  must run on the same machine as the API. It only accepts `*.chunk` files inside that folder.
- The API dispatches the moderator (`MODERATOR_AGENT = 'gd-moderator'` in `group-discussion/livekit.ts`) when a room
  locks. The API owns the timeline and writes it into the **room metadata** as JSON. The bot only reacts to it. If the
  bot drops, the GD still runs; only the spoken announcements are missing.
- Reads the same repo-root `.env` (`LIVEKIT_*`, `GEMINI_API_KEY`, `GD_*`).

**Endpoints** (`gd_agent/api.py`)

| Method | Path | Called by | What |
|---|---|---|---|
| GET | `/health` | manual checks | Status, Whisper model name, whether a Gemini key is set |
| POST | `/gd/transcribe` | `gd-results.service.ts` via `gd-worker.client.ts` | Join one recording's chunks -> faster-whisper (CPU, int8, VAD on) -> `{language, segments[start,end,text]}`. A broken recording returns empty segments, not an error |
| POST | `/gd/evaluate` | same | One student: rubric prompt -> Gemini JSON -> Pydantic check (one retry) -> `{result, model}` |
| POST | `/gd/mix` | same | All tracks placed on one timeline, private talks with HR muted -> `session.webm` (Opus, 16 kHz). Only `<group folder>/session.webm` under the storage folder may be written |

Transcribe and mix share one lock: only one heavy job at a time, because Whisper uses the whole laptop CPU.

**Main flow of the moderator** (`moderator.py` + `script.py`)
1. Pre-warm: load the Piper voice (downloaded once, about 60 MB).
2. Join the room, subscribe to **nobody**, name itself "AI Moderator", publish one audio track.
3. On every room metadata change: parse `LiveState` (`v: 1`, phase, topic, speaker, plan...). `lines_for(prev, new)`
   returns what to say: rules in INTRO, topic in THINKING, start in DISCUSSION, each name in SUMMARY, hand-over in
   HR_QUESTIONS, goodbye in CLOSE. Old lines are dropped when the phase changes.
4. Every second, `silence_prompt` checks for silence in DISCUSSION (never while paused or HR speaks) and gently asks
   the group, or names the quietest student.
5. At `ENDED` it finishes speaking and leaves.
- `GD_MODERATOR_LOAD_THRESHOLD` (default: no limit) stops LiveKit from refusing groups on a busy laptop.

**Evaluation rubric** (`evaluate.py`): content, communication (accent and small grammar never penalised),
initiative, listening, balance, summary, and `hr_response` (must be null if HR asked nothing). Each 1-10, plus
`overall`, `recommendation` (`selected` / `not_selected`), 3-8 evidence quotes and feedback. The prompt is written
**inside the code** (not in `prompts/`). Model: `GD_LLM_MODEL` or `LLM_MODEL` (default `gemini-2.5-flash`), with
`LLM_FALLBACK_MODELS`.

**Commands:** `uv run python -m gd_agent api` | `... moderator dev` (or `start`) | `... download` (Whisper + Piper once).

**Files**

| File | What it does |
|---|---|
| `workers/gd-agent/README.md` | Processes, commands, file list (it does not mention `mix.py`) |
| `workers/gd-agent/pyproject.toml` | Python 3.11-3.12; FastAPI, faster-whisper, PyAV pinned `<14.3`, livekit-agents, piper-tts, google-genai; wheels only for av/ctranslate2/onnxruntime |
| `workers/gd-agent/uv.lock` | Locked dependency versions |
| `workers/gd-agent/.python-version` | `3.12` (uv downloads it) |
| `gd_agent/__init__.py` | Package docstring only |
| `gd_agent/__main__.py` | CLI: `api` (uvicorn 127.0.0.1:8010), `moderator ...`, `download` |
| `gd_agent/api.py` | FastAPI routes, secret check (`hmac.compare_digest`), path safety, Whisper lock |
| `gd_agent/config.py` | `Settings` from `.env`: Gemini, Whisper model/language, storage dir, secret, Piper voice/dir |
| `gd_agent/transcribe.py` | `safe_chunk_paths`, lazy Whisper model, join chunks to a temp file and transcribe |
| `gd_agent/evaluate.py` | Rubric prompt, Pydantic models, parse + one retry, Gemini call with fallbacks |
| `gd_agent/mix.py` | Decode each track (PyAV) to 16 kHz, place at its offset, mute ranges, mix in 10 s blocks, encode WebM/Opus; max 4 h |
| `gd_agent/moderator.py` | LiveKit agent: voice track, metadata follower, silence watcher |
| `gd_agent/script.py` | Pure functions: `LiveState.parse`, `lines_for`, `silence_prompt` (the moderator's words) |
| `tests/test_gd_agent.py` | Script lines, silence prompt, prompt rules, retry, path safety, HTTP routes, mix |

---

### `workers/workflows`

**What it does.** Placeholder only (a README). Planned Temporal workflows `JobWorkflow` and `CandidateWorkflow`
(M29). Today the API uses its own background loops instead. Listed in `pnpm-workspace.yaml`, but it has no
`package.json`, so pnpm finds no package there.

| File | What it does |
|---|---|
| `workers/workflows/README.md` | Plan, owner (Intern 1), folder rules |

### `workers/activities-ts`

**What it does.** Placeholder only. Planned Temporal activities in TypeScript (email, publish, calendar; M29/M24).
Same as above: in the workspace list, but no package yet.

| File | What it does |
|---|---|
| `workers/activities-ts/README.md` | Plan, owner (Intern 1), folder rules |

---

### `prompts`

**What it does.** Versioned prompt YAML files for the AI worker. Each file has `task`, `version`, `temperature`,
`system` and `user` (a `$name` template). The `version` travels back to the API (`X-Prompt-Version` /
`promptVersion`), so every AI output can be traced to its prompt. Rule: change the wording = bump the version.

**How it connects.** Loaded by `ai_worker/llm_gateway/prompts.py` from `<repo>/prompts` (or `PROMPTS_DIR`). The GD
worker does **not** use this folder.

| File | What it does |
|---|---|
| `prompts/README.md` | Owner (Intern 2) and folder rules |
| `prompts/jd_generate.v3.yaml` | **Active** JD prompt (temp 0.4). Two inputs: `<company_profile>` and `<job_requirements>`. Used by `ai_worker/jd/generator.py` |
| `prompts/jd_generate.v2.yaml` | Older JD prompt (every bullet grounded in a requirement keyword). Not used by code; only `tests/test_config.py` checks the file exists |
| `prompts/jd_generate.v1.yaml` | First JD prompt (temp 0.6). Not used by any code |
| `prompts/resume_screen.v1.yaml` | Resume reading against one job: skills, experience, projects with evidence; no score (temp 0.1). `scoring/ats.py` |
| `prompts/resume_transcribe.v1.yaml` | Read text from scanned resume page images (temp 0). `parsing/ocr.py` |
| `prompts/screening_transcribe.v1.yaml` | Transcribe one short audio answer, audio only (temp 0). `screening/interview.py` |
| `prompts/screening_turn.v1.yaml` | Decide a follow-up / answer the candidate's question from verified job info (temp 0.2). `screening/interview.py` |
| `prompts/screening_extract.v1.yaml` | Finished screening transcript -> structured HR notes (temp 0). `screening/interview.py` |
| `prompts/interview_plan.v1.yaml` | Plan the topics of a spoken technical interview (temp 0.4). `interview/engine.py` |
| `prompts/interview_transcribe.v1.yaml` | Transcribe one interview answer with a list of tech terms (temp 0). `interview/engine.py` |
| `prompts/interview_turn.v1.yaml` | Choose the next question / follow-up (temp 0.3). `interview/engine.py` |
| `prompts/interview_report.v1.yaml` | Evidence-based evaluation of the finished interview (temp 0.1). `interview/engine.py` |
| `prompts/round_questions.v1.yaml` | Question suggestions for one assessment round (temp 0.7). `rounds/generator.py` |
| `prompts/coding_problems.v2.yaml` | DSA problems with stdin/stdout, reference solution, test inputs (temp 0.4). `rounds/coding.py` |

---

### `scripts`

**What it does.** Windows PowerShell scripts that run the whole stack on one laptop with public links, plus a daily
sync script and two (switched off) demos.

#### `scripts/start-public.ps1` step by step

Run with `pnpm start:public`, by the VS Code folder-open task, or every 5 minutes by Task Scheduler. It is safe to
run many times. Logs go to `%LOCALAPPDATA%\hireflow\logs`.

0. **One run at a time.** A named mutex `Local\HireFlowStartPublic`. If another run holds it, exit 0. (Two runs could
   start two tunnels and email different links.)
1. **Docker Desktop.** If `docker info` fails, start Docker Desktop and wait up to 300 s.
2. **Postgres, Redis, Mailpit.** `docker compose -f infra/docker/docker-compose.yml up -d` (output to `docker.log`).
   Wait up to 120 s for `pg_isready` in container `hireflow-postgres-1`.
3. **Cloudflare quick tunnels** (no account needed). Function `Tunnel(port)`:
   - If a `cloudflared` process for that port exists, read its URL from the log and test it. Keep it if it still
     answers.
   - Else kill the old process, delete the log, start `cloudflared tunnel --url http://localhost:<port>`, and wait up
     to 60 s for a `https://*.trycloudflare.com` URL in the log.
   - **Careers site :3300** -> URL written to `CAREERS_SITE_URL` in `.env`. No tunnel = exit 1.
   - **Candidate app :3001** (DSA/SQL coding page) -> only when `PISTON_URL` is set. URL written to
     `CANDIDATE_APP_URL`; else it is set back to `http://localhost:3001` (and coding invitations wait).
   - **HR dashboard :3000** -> only when `HR_PUBLIC_TUNNEL=on`. URL written to `HR_DASHBOARD_URL`; else it is set
     back to `http://localhost:3000`.
   - `.env` is rewritten as UTF-8 without BOM.
4. **Restart on URL change.** If any URL changed and the API is up, stop ports 4000, 3000, 3300, 3001 (it walks up
   the parent processes and kills the pnpm/turbo/nest/next tree).
5. **API + web apps.** If the API or careers site is not listening:
   - run `prisma generate` in `apps/api` (with `npx -y pnpm@9.15.0`, because pnpm is not on PATH on these laptops);
   - start `serve-public.ps1` hidden (production builds of the web apps);
   - wait up to 900 s for :4000 and :3300, then :3001 (if DSA) and :3000.
6. **AI worker.** If :8000 is free and `uv` exists: `uv run python -m ai_worker` in `workers/ai-worker`
   (`ai-worker.log`).
7. **GD worker + moderator.** If `workers/gd-agent` exists: start `gd_agent api` on :8010 (`gd-worker.log`) and, if
   no python process runs `gd_agent moderator`, start `gd_agent moderator start` (`gd-moderator.log`).
8. **Final checks.** Wait until the public careers URL returns 200, then the DSA URL and the HR URL (`/login`).
   Print the links. If Google login is configured, print the redirect URI to add in Google Cloud Console.

Note: a quick tunnel gets a **new random URL** each time `cloudflared` restarts. The API then edits old posts with the
new apply link (`PUBLISHER_AUTO_REFRESH_LINKS`). Links already emailed for assessments or DSA stop working.

**Files**

| File | What it does |
|---|---|
| `scripts/start-public.ps1` | Starts the whole public stack (see steps above) |
| `scripts/serve-public.ps1` | Called by start-public. Builds `careers-site`, `candidate-app`, `hr-dashboard` with `turbo run build`, then runs the API in watch mode (`pnpm --filter api dev`) and each web app with `next start` on 3300 / 3001 / 3000. If a build fails, falls back to `pnpm dev`. Sets `PUBLISHER_AUTO_REFRESH_LINKS=true`. Logs `build.log`, `dev.log`, `web-<app>.log` |
| `scripts/register-autostart.ps1` | Registers Task Scheduler task "HireFlow public apply link": at login + every 5 minutes, on battery too, never two at once, 30 min limit |
| `scripts/sync.sh` | `pnpm sync`: `git pull --rebase --autostash`, `pnpm install`, `uv sync` in ai-worker (not gd-agent), `prisma migrate deploy` + `generate` |
| `scripts/demo-gd/demo-gd.mjs` | Demo that created one ready GD group with student links. **Switched off** (exits at once since 2026-11-04: links go only to students who applied) |
| `scripts/demo-m06-m09/demo.mjs` | End-to-end demo of publish -> apply -> intake against the real API. **Switched off** for the same reason |
| `scripts/demo-m06-m09/demo-job.json` | Job form + ready-made JD used by `demo.mjs` when the AI worker is down |
| `scripts/demo-m06-m09/README.md` | What the M06-M09 demo did and how to run it |
| `scripts/demo-m06-m09/create-parser-login.sql` | Creates read-only DB login `hireflow_parser` (role `hireflow_resume_reader`, view `parser_resume_inbox`) |

---

### `infra`

**What it does.** Local services in Docker, and the Piston code runner. `k8s` and `terraform` are placeholders.

**Docker services** (`infra/docker/docker-compose.yml`, project name `hireflow`)

| Service | Image | Ports | When | Used for |
|---|---|---|---|---|
| `postgres` | `pgvector/pgvector:pg17` | `${POSTGRES_PORT:-5432}:5432` | always | Main DB (user/pass/db `hire`/`hire`/`hireflow`), volume `pgdata` |
| `redis` | `redis:7.4-alpine` | 6379 | always | Cache/queues (`REDIS_URL`) |
| `mailpit` | `axllent/mailpit` | 1025 SMTP, 8025 web inbox | always | Fake inbox for local email (`SMTP_URL`) |
| `minio` | `minio/minio` | 9000, 9001 | profile `storage` | S3-like storage. Off by default |
| `clamav` | `clamav/clamav:stable` | 3310 | profile `security` | Resume virus scan (`VIRUS_SCANNER=clamav`) |
| `temporal` | `temporalio/temporal:latest` | 7233, 8233 | profile `workflow` | Future workflows. Not used yet |

**Piston** (`infra/piston/`): the sandbox that runs students' DSA code. `start-piston.ps1` starts container
`hireflow-piston` from an image **pinned by digest**, on `127.0.0.1:2000` only (Piston has no login), on Docker's
default network (not `hireflow_default`, so it cannot see the DB), packages in volume `hireflow_piston_data`. It
installs Python 3.12.0, Java 15.0.2, gcc 10.2.0, checks the safety rules, and runs a real program in each language.
`-Recreate` rebuilds the container with the current `piston.env`. The API reaches it through `PISTON_URL`.

| File | What it does |
|---|---|
| `infra/docker/docker-compose.yml` | Local services above |
| `infra/piston/piston.env` | Piston ceilings: no network, 64 KB output, 4 parallel jobs, 8 s run / 15 s compile, 256 MB run / 512 MB compile memory, bigger limits for Java |
| `infra/piston/start-piston.ps1` | Start / repair / smoke-test Piston (see above) |
| `infra/k8s/README.md` | Placeholder: "Kubernetes manifests. Not needed yet." |
| `infra/terraform/README.md` | Placeholder: "Cloud infra. Not needed yet." |

---

### `seed`

| File | What it does |
|---|---|
| `seed/README.md` | Placeholder: planned demo tenant/users/jobs and sample resumes |
| `seed/resumes/.gitkeep` | Keeps the empty `seed/resumes/` folder in git |

---

### `.github`

| File | What it does |
|---|---|
| `.github/workflows/ci.yml` | CI on pull requests and on push to `main`. Job `typescript`: Node 22, `pnpm install --frozen-lockfile`, `pnpm check`. Job `python`: in `workers/ai-worker` only: `uv sync`, `ruff check`, `pytest`. (gd-agent is not tested in CI. The default branch is `master`, so the push trigger on `main` does not fire there.) |
| `.github/CODEOWNERS` | Auto-reviewers per folder: Intern 1 `@naina-chauhan` (HR/core), Intern 2 `@Dhruv-Andhariya` (ai-worker, prompts, ats, screening, evaluator, feedback), Intern 3 `@GrisaPujara` (candidate side, publisher, round-engine). `packages/contracts`, `apps/api/prisma`, `infra/docker` need all three |
| `.github/pull_request_template.md` | PR checklist: modules touched, shared changes (contracts + `schemas.py` mirror, Prisma, compose/.env.example), `pnpm check` + `uv run pytest`, no secrets |

### `.vscode`

| File | What it does |
|---|---|
| `.vscode/tasks.json` | Task "HireFlow: start public stack" runs `scripts/start-public.ps1` **automatically when the folder opens** (`runOn: folderOpen`). Also `infra: up`, `infra: down`, `check: everything` |
| `.vscode/settings.json` | Format on save (Prettier; Ruff for Python), LF line ends, workspace TypeScript, Python interpreter `workers/ai-worker/.venv/bin/python` (a Linux/Mac path) |
| `.vscode/extensions.json` | Recommended extensions: Prettier, ESLint, Tailwind, Prisma, Python/Pylance, Ruff, Docker, GitLens, Error Lens, EditorConfig, Vitest, Mermaid |

---

### Root files

| File | What it does |
|---|---|
| `package.json` | Root of the pnpm monorepo. `pnpm@9.15.0`, Node >= 22. Scripts: `dev`/`build`/`lint`/`typecheck`/`test` (all via turbo), `check` (all four), `start:public`, `sync`, `infra:up`/`infra:down`, `demo:m06-m09`, `format` (Prettier). Dev deps: prettier, turbo, typescript |
| `pnpm-lock.yaml` | Locked JS dependency versions |
| `pnpm-workspace.yaml` | Workspace packages: `apps/*`, `packages/*`, `packages/rounds/*`, `workers/workflows`, `workers/activities-ts` (the Python workers are not pnpm packages) |
| `turbo.json` | Tasks `build` (outputs `dist/**`, `.next/**`), `dev` (no cache, persistent, passes `PUBLISHER_AUTO_REFRESH_LINKS`), `lint`, `typecheck`, `test`. All depend on `^build` (build dependencies first) |
| `.env.example` | Template for `.env` (see the next section) |
| `.editorconfig` | UTF-8, LF, 2 spaces (4 for Python), final newline, trim spaces |
| `.prettierrc` | Single quotes, semicolons, trailing commas, width 100 |
| `.gitattributes` | Force LF for all text; png/jpg/pdf/zip are binary |
| `.gitignore` | node_modules, build outputs, logs, `.env` and `.env.*` (except `.env.example`), Python caches/venv, `infra/docker/data/`, `/env` |
| `AGENTS.md` | Block written by Turborepo for AI agents: read the docs of the **installed** turbo version before changing turbo config |
| `README.md` | Project overview, what works, day-by-day timeline, each module, apps and ports, quick start, Neon DB, public link. Some parts are older than the code (for example it still says resume parsing/ATS are mocks) |
| `hello.py` | One line, `print("hello")`. A leftover test file; nothing uses it |

---

### Environment variables (`.env.example`)

The API, the AI worker and the GD worker all read the **same repo-root `.env`**. Rule for timeouts: the worker
budget (seconds) must be **smaller** than the API timeout (ms) for the same call.

| Variable(s) | Area | What it controls |
|---|---|---|
| `DATABASE_URL`, `REDIS_URL` | Database / infra | Postgres and Redis. The example uses port **5433** (for laptops that already have Postgres on 5432) while compose defaults to 5432 |
| `RESUME_STORAGE`, `S3_ENDPOINT`, `S3_REGION`, `S3_BUCKET`, `S3_ACCESS_KEY`, `S3_SECRET_KEY` | Resume files (M09) | `s3` = files in a bucket; empty = bytes in the DB (dev only) |
| `SMTP_URL`, `GMAIL_USER`, `GMAIL_APP_PASSWORD`, `MAIL_FROM` | Email | Gmail if both Gmail values set, else SMTP (Mailpit) |
| `TEMPORAL_ADDRESS` | Workflows | Future Temporal server |
| `API_PORT`, `AI_WORKER_URL`, `NEXT_PUBLIC_API_URL`, `HR_DASHBOARD_URL`, `HR_PUBLIC_TUNNEL` | Service URLs | Where services live; `HR_PUBLIC_TUNNEL=on` makes `start-public.ps1` open a tunnel for the HR dashboard |
| `CORS_ORIGINS`, `COOKIE_SECURE`, `LOGIN_2FA`, `SIGNUP_VERIFY_EMAIL`, `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_REDIRECT_URI`, `LOGIN_CODE_TTL_MINUTES`, `LOGIN_CODE_MAX_ATTEMPTS`, `SESSION_TTL_HOURS`, `INVITATION_TTL_HOURS`, `LOGIN_MAX_ATTEMPTS`, `SIGNUP_MAX_PER_IP`, `PLATFORM_ADMIN_*` | Auth (M28) | Cookie sessions, email 2FA, sign-up verification, Google login, limits, first platform admin |
| `LLM_PROVIDER`, `LLM_MODEL`, `LLM_FALLBACK_MODELS`, `GEMINI_API_KEY`, `LLM_TIMEOUT_SECONDS`, `LLM_TRANSIENT_RETRIES` | AI gateway | Provider (only `gemini` works), model, backup models, per-request timeout, 5xx retries |
| `GROQ_API_KEY`, `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `EMBEDDING_MODEL` | AI (unused) | Present in the template, not read by the workers today |
| `AI_WORKER_TIMEOUT_MS`, `JD_GENERATION_BUDGET_SECONDS` | JD generation | API wait (90 s) vs worker budget (80 s) |
| `RESUME_SCREEN_BUDGET_SECONDS`, `RESUME_OCR_FALLBACK`, `ATS_WORKER_TIMEOUT_MS`, `ATS_PROCESSING`, `ATS_POLL_MS`, `ATS_CONCURRENCY` | ATS (M10/M11) | Worker budget, scanned-PDF fallback, API background loop and parallel resumes |
| `AI_SCREENING_URL`, `AUTO_REJECTION_EMAILS`, `AUTO_SHORTLIST_EMAILS` | After ATS | Screening link and automatic shortlist / rejection emails |
| `SCREENING_TTS_MODEL`, `SCREENING_TTS_FALLBACK_MODELS`, `SCREENING_TTS_VOICE`, `SCREENING_TURN_BUDGET_SECONDS`, `SCREENING_SPEECH_BUDGET_SECONDS`, `SCREENING_EXTRACT_BUDGET_SECONDS`, `SCREENING_THINKING_BUDGET`, `SCREENING_WORKER_TIMEOUT_MS`, `SCREENING_SPEECH_TIMEOUT_MS`, `SCREENING_EXTRACT_TIMEOUT_MS`, `SCREENING_FOLLOW_UPS_PER_QUESTION`, `SCREENING_MAX_CANDIDATE_QUESTIONS`, `SCREENING_MAX_TURNS`, `SCREENING_MAX_MINUTES` | M12 voice screening | TTS model/voice (also used by the interview), worker budgets, API timeouts, conversation limits |
| `ROUND_QUESTIONS_BUDGET_SECONDS`, `ROUND_QUESTIONS_WORKER_TIMEOUT_MS` | M05 rounds | AI question suggestions: worker budget vs API wait |
| `ASSESSMENT_URL`, `ASSESSMENT_LINK_DAYS`, `ASSESSMENT_REQUIRE_CAMERA`, `AUTO_ROUND_INVITES`, `ROUND_ENGINE_PROCESSING`, `ROUND_ENGINE_POLL_MS`, `ROUND_ALLOW_LOCAL_LINKS` | M13 online tests | Test link, link life, camera, automatic invites, background loop, allow localhost links (Mailpit testing) |
| `DSA_ROUNDS`, `CANDIDATE_APP_URL`, `DSA_ASSESSMENT_URL`, `PISTON_URL`, `DSA_PROCTOR_MAX_STRIKES`, `DSA_PROCTOR_CAMERA`, `DSA_PROCTOR_CAMERA_STRIKES`, `DSA_PROCTOR_CAMERA_COOLDOWN_MS`, `DSA_ADMIN_API_KEY`, `DSA_SHOW_SCORE`, `DSA_DEMO_MODE` | DSA coding round | Coding page URL (tunnel), code runner, proctoring strikes and camera, admin routes, demo mode |
| `AI_INTERVIEW_URL`, `AI_INTERVIEW_LINK_DAYS`, `AI_INTERVIEW_PROCESSING`, `AI_INTERVIEW_POLL_MS`, `AUTO_AI_INTERVIEW_INVITES`, `AUTO_AI_INTERVIEW_DECISIONS`, `INTERVIEW_PLAN_TIMEOUT_MS`, `INTERVIEW_TURN_TIMEOUT_MS`, `INTERVIEW_REPORT_TIMEOUT_MS`, `INTERVIEW_PLAN_BUDGET_SECONDS`, `INTERVIEW_TURN_BUDGET_SECONDS`, `INTERVIEW_REPORT_BUDGET_SECONDS`, `INTERVIEW_THINKING_BUDGET` | AI technical interview | Link, loop, auto invites/decisions, API timeouts vs worker budgets (turn must stay under the careers-site proxy's 30 s) |
| `SQL_RUNNER_CONCURRENCY`, `SQL_RUNNER_HEAP_MB` | SQL round | Parallel SQLite query processes and their memory |
| `GD_PROCESSING`, `GD_POLL_MS`, `GD_LIVE_POLL_MS`, `GD_URL`, `GD_ALLOW_LOCAL_LINKS`, `GD_STORAGE_DIR`, `GD_RETENTION_DAYS`, `AUTO_GD_CONFIRM`, `AUTO_GD_SCHEDULE`, `AUTO_GD_MIN_STUDENTS`, `AUTO_GD_LEAD_MINUTES` | M18 GD (API side) | Pools, live timeline loop, student link, audio folder and retention, auto schedule / confirm |
| `GD_WORKER_URL`, `GD_WORKER_TIMEOUT_MS`, `GD_WORKER_SECRET`, `GD_WHISPER_MODEL`, `GD_WHISPER_LANGUAGE`, `GD_LLM_MODEL`, `GD_PIPER_VOICE` | M18 GD worker | Worker address, secret header, Whisper model, evaluation model, moderator voice (`GD_PIPER_DIR`, `GD_LLM_TIMEOUT_SECONDS`, `GD_MODERATOR_LOAD_THRESHOLD` are read by code but not in the template) |
| `LIVEKIT_URL`, `LIVEKIT_API_KEY`, `LIVEKIT_API_SECRET` | M18 video room | LiveKit Cloud project used by the API and the moderator bot |
| `KEYCLOAK_URL`, `WHATSAPP_TOKEN`, `GOOGLE_OAUTH_CLIENT_ID`, `GOOGLE_OAUTH_CLIENT_SECRET`, `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY` | Later versions | Empty for now |
| `ENCRYPTION_KEY` | Security | 32 random bytes (base64); encrypts channel OAuth tokens |
| `CAREERS_SITE_URL`, `PUBLISHER_AUTO_REFRESH_LINKS` | Publisher (M06) | Base of every Apply link (the tunnel writes it); auto-edit old posts when it changes |
| `LINKEDIN_CLIENT_ID`, `LINKEDIN_CLIENT_SECRET`, `LINKEDIN_REDIRECT_URI`, `LINKEDIN_POST_AS`, `LINKEDIN_SCOPES`, `LINKEDIN_ACCESS_TOKEN`, `LINKEDIN_FALLBACK_TENANT_ID`, `LINKEDIN_ALLOWED_MEMBER_ID`, `LINKEDIN_API_VERSION` | LinkedIn channel (M07) | OAuth app, post as page or member, fallback token for one company, allowed account, API version header |
| `OTP_PROVIDER`, `OTP_SIGNING_SECRET` | Apply page (M08) | Phone OTP (`dev` shows the code on screen) |
| `VIRUS_SCANNER`, `CLAMAV_HOST`, `CLAMAV_PORT` | Resume upload (M09) | `clamav` or `dev-eicar-only` |
| `PUBLIC_RATE_LIMITS`, `TRUST_PROXY`, `TRUSTED_CLIENT_IP_HEADER`, `TURNSTILE_SECRET_KEY`, `NEXT_PUBLIC_TURNSTILE_SITE_KEY` | Public endpoint protection | Per-IP limits, real client IP behind the tunnel (`cf-connecting-ip`), Cloudflare Turnstile CAPTCHA |
| `FEATURE_PUBLISHER`, `FEATURE_APPLICATIONS`, `FEATURE_SCREENING`, `FEATURE_ROUNDS`, `FEATURE_ROUND_ENGINE`, `FEATURE_AI_INTERVIEW`, `FEATURE_AI_INTERVIEW_PROTECTION`, `FEATURE_DSA`, `FEATURE_SQL`, `FEATURE_GROUP_DISCUSSION` | Feature flags | `off` turns a module's routes off without a code change |
| `ASSESSMENT_PROTECTION`, `ASSESSMENT_MAX_WARNINGS`, `ASSESSMENT_TAB_MONITORING`, `ASSESSMENT_FULLSCREEN_REQUIRED`, `ASSESSMENT_FACE_POLICY` | Online test proctoring | Warning counter, tab / full-screen / face rules |

Also read by the AI worker but not in the template: `PROMPTS_DIR`, `HIREFLOW_ENV_FILE`, `CODING_PROBLEMS_BUDGET_SECONDS`.

---

### `docs`

| File | What it covers |
|---|---|
| `docs/README.md` | Index of the docs. It links `PROJECT_MAP.md`, `JD_WORKFLOW.md` and `MULTI_COMPANY_AUTH.md`, which are **not in the repo** |
| `docs/HireFlow_AI_Documentation.md` | The full product and architecture spec (about 1,950 lines): problem, flow, architecture, tech stack, all modules M01-M33, wiring, data model, rules. Module numbers used everywhere come from here |
| `docs/TEAM_GUIDE.md` | How 3 interns work in parallel: who builds what, setup on every laptop, git workflow, daily routine, troubleshooting |
| `docs/FOLDER_STRUCTURE.md` | Tour of every folder: purpose, owner, status (working / partial / placeholder), example of JD generation across folders |
| `docs/M06_M09_PUBLISH_AND_APPLY.md` | Publisher, channel adapters, apply page, application intake: flow, guarantees, run, tests, LinkedIn needs |
| `docs/ASSESSMENT_ROUNDS.md` | M05 HR side: choosing rounds, questions, pass marks on the Publish page; DB tables; API routes |
| `docs/SCREENING_ROUND.md` | M12 AI voice screening: flow, voice architecture, privacy, run, tests, limits |
| `docs/APTITUDE_ASSESSMENT_ROUND.md` | M13 online test rounds: invitation, timed webcam test, server grading, emails, env vars, deployment |
| `docs/DSA_ASSESSMENT_ROUND.md` | DSA coding round: HR setup, invitation, coding page, scoring, HR review, running it |
| `docs/dsa-integration-notes.md` | DSA module notes: files, API endpoints, questions, token, Piston, test results (Judge0 was dropped) |
| `docs/dsa-piston-hosting.md` | Hosting Piston on one PC: requirements, startup order, settings, remote test, troubleshooting |
| `docs/dsa-proctoring-implementation.md` | Browser proctoring for DSA: full screen, tab focus, clipboard, violation model, privacy, limits |
| `docs/SQL_ASSESSMENT_ROUND.md` | SQL round: question bank, candidate page, SQLite sandbox security, setup, tests |
| `docs/SQL_CAMERA_PROCTORING.md` | AI camera monitoring (MediaPipe in the browser) on SQL/DSA pages; `DSA_PROCTOR_CAMERA` modes |
| `docs/AI_TECHNICAL_INTERVIEW.md` | AI technical interview: conversation, voice architecture, camera/mic, report, HR decision, security, tests |
| `docs/GROUP_DISCUSSION_ROUND.md` | M18 GD: pools, groups, LiveKit room, AI moderator, evaluation, HR live controls, privacy, env vars, limits |
| `docs/HIRING_PIPELINE.md` | Rules between rounds: stages, Stage 2 criterion, rejection reasons and emails, GD rubric, personal interview, freezing |

---

---

## Where do I look when…

| I want to… | Start here |
|---|---|
| change what the AI writes in a JD | `prompts/jd_generate.v3.yaml`, `workers/ai-worker/ai_worker/jd/` |
| change how the ATS score is calculated | `workers/ai-worker/ai_worker/scoring/ats.py` (fixed formula; Gemini only reads the resume) |
| change the shortlist / rejection email text | `apps/api/src/modules/ats/emails/decision-emails.ts` |
| change what happens after a round (next invite / cleared / rejected) | `apps/api/src/modules/round-engine/online-test.ts` `nextStep()` and `RoundEngineService.nextFor()` / `applyStep()` |
| change round invitation or outcome email text | `apps/api/src/modules/round-engine/emails.ts` |
| add a new round type | `packages/contracts/src/schemas/rounds.ts` (types, labels, `ROUND_MODULE_OF`), `apps/api/prisma/schema/rounds.prisma` (enum), the rounds editor `apps/hr-dashboard/src/components/rounds/` |
| change the Stage 2 rule or the pipeline status | `packages/contracts/src/schemas/pipeline.ts`, `apps/api/src/modules/pipeline` |
| change the AI technical interview | `apps/api/src/modules/ai-interview` (+ `workers/ai-worker/ai_worker/interview`, careers-site `/interview/[token]`) |
| add a table or column | a new `.prisma` file / model in `apps/api/prisma/schema/` + a new migration (additive only — the DB is shared) |
| add an API endpoint | a controller in the module, the body schema in `packages/contracts`, a client in `apps/hr-dashboard/src/lib/*-api.ts` |
| find why a background job does not run on my laptop | the loops table in part 4 — check the `*_PROCESSING` / `AUTO_*` / `FEATURE_*` values in your `.env` |
| find why an email with a link was not sent | the link must be public and open: `CAREERS_SITE_URL` / `CANDIDATE_APP_URL` from `pnpm start:public` (or `ROUND_ALLOW_LOCAL_LINKS=on` for Mailpit) |
| see who did what | the `audit_logs` table (`apps/api/src/modules/audit`) |

## Things found while writing this guide

These came up while reading the code. They are not fixed here; they are listed so nobody is surprised.

**Behaviour to know**
- The AI technical interview runs only when HR adds an "AI Technical Interview" round. Without it, clearing the last
  round sends `CLEARED_ALL` (no interview link).
- Voice screening never rejects anyone. When it is completed, the first round's invitation goes out automatically.
- GD selection and the AI interview decision are automatic by default (`AUTO_GD_CONFIRM`,
  `AUTO_AI_INTERVIEW_DECISIONS`); interviews ended by too many warnings always wait for HR.
- DSA / SQL camera proctoring ends the attempt when violations go **above** `DSA_PROCTOR_MAX_STRIKES` (default 3 →
  the 4th ends it). Some docs say "the third ends it".
- Every write request (also public student ones like apply / OTP) needs the header `X-Requested-With: hireflow`.
- Login 2FA and signup email codes are on only when `NODE_ENV=production`. The phone OTP sender is `dev` only (no
  SMS; code returned to the browser) — production has no real OTP sender yet.
- Rate limits, the speech cache, the account email queue and connected round modules are **per API process** (in
  memory), even though the database is shared.
- `.vscode/tasks.json` runs `scripts/start-public.ps1` when the folder is opened in VS Code; with
  `register-autostart.ps1` it also runs at login and every 5 minutes. A restarted tunnel means a new public URL.

**Possible bugs / loose ends**
- `RoundEngineService.continueAfterCoding()` sorts rounds by position without skipping HR-skipped rounds: when the next
  round is skipped, the coding page shows "done" although the later round's invitation is emailed.
- `.github/workflows/ci.yml` runs on pushes to `main`, but the main branch is `master`. Nothing tests `workers/gd-agent`.
- `apps/candidate-app/src/app/dsa/[token]/webcam.tsx` and `public/models/blaze_face_short_range.tflite` (candidate-app)
  are unused; `tsconfig.tsbuildinfo` is committed by mistake.
- `.env.example` uses port 5433 in `DATABASE_URL`, docker-compose defaults to 5432.

**Out-of-date documents**
- Root `README.md` still says resume parsing / ATS scoring are mocks — they are real now.
- `docs/README.md`, jobs and tenants READMEs link to `PROJECT_MAP.md`, `JD_WORKFLOW.md`, `MULTI_COMPANY_AUTH.md`,
  which do not exist.
- auth README: invitations ARE emailed; `/auth/password/forgot` and `/reset` are missing from its table. jobs README:
  `DELETE /jobs/:id` missing. applications README: "my data" actions go to `audit_logs` (the old table was dropped).
- `ats/decision.controller.ts` comment says decision emails are "never sent automatically" — they are, by default.
- Empty placeholders: `packages/rounds/*`, `round-sdk`, `ui`, `config`, `email-templates`, `proctor-sdk`,
  `apps/admin-console`, `workers/workflows`, `workers/activities-ts`, and the API modules `analytics`, `evaluator`,
  `feedback`, `notifications`, `proctoring`, `scheduler`. `hello.py` is unused.

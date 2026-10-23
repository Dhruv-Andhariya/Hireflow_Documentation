# HireFlow – Project Map (simple version)

One file that explains **what every folder does** and **how the pieces talk to each other**.
Read it top to bottom once; after that, use the "Where do I change…?" table at the end.

---

## 1. The big picture in 30 seconds

HireFlow helps a company go from "we have an opening" to "students have applied".

There are **4 running programs** and **1 database**:

```
   HR person (browser)                     Student (browser)
          │                                       │
          ▼                                       ▼
 ┌─────────────────┐                   ┌─────────────────┐
 │  HR Dashboard   │                   │  Careers Site   │
 │  Next.js :3000  │                   │  Next.js :3300  │
 └────────┬────────┘                   └────────┬────────┘
          │  calls /api/v1/...                  │  calls /api/v1/public/... (through its own proxy)
          ▼                                       ▼
        ┌───────────────────────────────────────────┐
        │                API (NestJS)  :4000          │──────► LinkedIn (posts jobs)
        │   the "brain": rules, logins, saving data   │──────► Gmail (sends emails)
        └───────┬─────────────────────────┬──────────┘──────► ClamAV (virus check, optional)
                │ Prisma                  │ HTTP
                ▼                         ▼
     ┌──────────────────────┐   ┌──────────────────────┐
     │ PostgreSQL database  │   │ AI Worker (Python)    │──────► Google Gemini (writes the JD)
     │ (shared, on Neon)    │   │ FastAPI  :8000        │
     └──────────────────────┘   └──────────────────────┘
```

- **Websites never touch the database directly.** They always ask the API.
- **The API is the only one that reads/writes the database.**
- **The AI Worker only writes text** (job descriptions). It does not save anything.

---

## 2. Top-level folders

| Folder | What it is | Real code? |
|---|---|---|
| `apps/` | The programs you run (API + websites) | Yes – `api`, `hr-dashboard`, `careers-site` |
| `packages/` | Shared code used by several apps | Only `contracts` has code; the rest are plans |
| `workers/` | Background helpers | Only `ai-worker` (Python) has code |
| `prompts/` | The text instructions we give the AI | Yes (`jd_generate.v3.yaml` is the one used) |
| `infra/` | Docker setup for local Postgres / Redis / Mailpit | Yes (`infra/docker/docker-compose.yml`) |
| `scripts/` | Helper scripts (daily sync, public link, demo) | Yes |
| `docs/` | Documentation (you are here) | Yes |
| `seed/` | Demo data (planned) | Empty for now |
| `.github/` | CI checks that run on every PR | Yes |

Root files:
- `package.json` – the commands: `pnpm dev`, `pnpm check`, `pnpm infra:up`, `pnpm sync` …
- `pnpm-workspace.yaml` – tells pnpm which folders are "projects" in this monorepo.
- `turbo.json` – tells Turborepo to **build `packages/contracts` first**, then start the apps.
- `.env` – your private settings (passwords, URLs). **Never commit it.** `.env.example` is the template.

---

## 3. `apps/api` – the brain (NestJS, port 4000)

Every URL starts with `http://localhost:4000/api/v1`.

### How it starts
1. `src/main.ts` loads the root `.env` (via `src/common/env.ts`), turns on CORS, sets the `/api/v1` prefix, listens on port 4000.
2. `src/app.module.ts` plugs in all the modules below.
3. **A guard checks every request** (`modules/auth/auth.guard.ts`):
   - Is this route public, or must you be logged in? Which role is needed?
   - For anything that changes data (POST/PUT/PATCH/DELETE) the browser must send the header `x-requested-with: hireflow` (protection against fake requests from other sites).
   - Login is remembered with a **cookie** (httpOnly – JavaScript can't read it).

### `src/common/` – small shared helpers
| File | In simple words |
|---|---|
| `prisma.service.ts` | The single connection to the database |
| `api-error.ts` | Makes every error look the same: `{ statusCode, code, message }` |
| `secret-box.ts` | Locks/unlocks secrets (LinkedIn tokens) with `ENCRYPTION_KEY` |
| `env.ts` | Finds and loads the root `.env` |
| `feature-flags.ts` | Lets you switch a module off with `FEATURE_PUBLISHER=off` etc. |
| `uuidv7.ts` | Makes unique ids that sort by time |

### `src/modules/` – one folder per feature
Each module usually has: **controller** (the URLs) → **service** (the rules) → **store/repository** (database queries).

| Module | What it does | Main URLs |
|---|---|---|
| `auth` | Sign up, log in (+ email code), log out, forgot password, Google login, **team & invitations** | `/auth/...`, `/company/members`, `/company/invitations` |
| `tenants` | Companies and the company profile. "Tenant" = one company | `/companies`, `/company-profile`, `/admin/companies` |
| `jobs` | Job form → asks AI Worker for a JD → versions → approve | `/jobs`, `/jobs/:id/jd/generate`, `/jobs/:id/approve` |
| `publisher` | Publishes an approved job to the careers site + LinkedIn; builds the post text | `/publisher/...` |
| `applications` | Public careers API, phone OTP, receiving applications + resumes, HR's application list, "my data" delete | `/public/...`, `/applications/...` |
| `audit` | Writes a log row for every important action ("who did what, when") | (no URLs) |
| others (`ats`, `screening`, `rounds-config`…) | Plans only – README, no code yet | – |

How modules lean on each other:
```
auth  ◄── tenants ◄── jobs
                 ◄── publisher ◄── applications
audit is used by everybody
```

Important files inside modules (good to know):
- `jobs/ai-worker.client.ts` – the API's "phone line" to the AI Worker.
- `publisher/post-formatter.ts` – builds the LinkedIn/careers post text (ends with **Apply here** + **Company website** links).
- `publisher/channels/linkedin.adapter.ts` – actually sends the post to LinkedIn.
- `publisher/channel-credentials.ts` – stores each company's LinkedIn token (encrypted).
- `applications/email/mailer.ts` – sends emails (Gmail if `GMAIL_USER` is set, else Mailpit).
- `auth/team.service.ts` + `auth/invitation-email.ts` – create an invitation and email the link.

### `src/cli/` – one-off commands
- `pnpm --filter api admin:create` – create the first Platform Admin (reads `PLATFORM_ADMIN_*` from `.env`).
- `pnpm --filter api company:assign-admin` – make a user the admin of a company.
- `pnpm --filter api resumes:to-storage` – move resumes from the database to S3.

### `prisma/` – the database shape
- `prisma/schema/*.prisma` – the tables, split by topic:
  `tenants` (companies, profiles) · `auth` (users, sessions, invitations…) · `jobs` (jobs, JD versions, approvals) ·
  `publisher` (listings, posts, LinkedIn settings) · `applications` (candidates, applications, resumes) · `audit`.
- `prisma/schema/migrations/` – 15 step-by-step changes to the database. **Never edit old ones.**
- Apply them with `pnpm --filter api db:deploy`.

### `tests/`
- Normal tests use fake in-memory data – run with `pnpm --filter api test`.
- Files ending in `.db.test.ts` need a real **test** database (`TEST_DATABASE_URL`, name must contain "test"). Never point them at Neon.

---

## 4. `apps/hr-dashboard` – the HR website (Next.js, port 3000)

Who uses it: company HR people, Company Admins, and the Platform Admin.

Pages (folder `src/app/`):
| Page | What happens there |
|---|---|
| `/login`, `/signup`, `/forgot-password`, `/reset-password/[token]` | Account pages |
| `/register-company` | After signup: create your company |
| `/company` | Company profile (name, about, **website**, careers link…) – the AI uses this for every JD |
| `/jobs`, `/jobs/new`, `/jobs/[id]` | Create a job, get the AI JD, edit, approve |
| `/publish` | Publish approved jobs to careers site + LinkedIn |
| `/applications` | See who applied, download resumes |
| `/team` | Invite people (link is emailed), change roles |
| `/invite/[token]` | The page the invited person opens to join |
| `/admin` | Platform Admin: all companies |

Wiring:
- `src/lib/api.ts` and `src/lib/publisher-api.ts` – **every call to the API lives here**. The browser calls `NEXT_PUBLIC_API_URL` (`http://localhost:4000/api/v1`) directly, with the login cookie.
- `src/components/` – reusable pieces (session handling, buttons, JD panel, publish dialog, team table…).

---

## 5. `apps/careers-site` – the public student website (Next.js, port 3300)

Who uses it: students. No login.

Pages:
- `/[company]` – all live jobs of one company (e.g. `/the-tfpl`).
- `/[company]/jobs/[job]` – the full job + **Apply form** (phone OTP, resume upload).
- `/[company]/my-data` – a student can see/delete their data.

Wiring (important for the public link):
- The page is built on the server, which calls the API at `localhost:4000`.
- The student's browser calls **its own address** `/api/v1/public/...`; `next.config.ts` forwards that to the API.
  → That's why only port 3300 needs to be public (the Cloudflare tunnel), not the API.
- `src/middleware.ts` cleans the visitor's IP header so nobody can fake their IP to dodge rate limits.
- `src/content.ts` – all the visible text in one place.

## 6. `apps/admin-console`, `apps/candidate-app`
Empty placeholders (planned for later).

---

## 7. `packages/contracts` – the shared "agreement" (most important package)

**Problem it solves:** the API and the websites must agree on the exact shape of data
(e.g. "a job has a title, a salary range, skills…").

**Solution:** the shapes are written **once** here (with the Zod library) and used by both sides:
- the API uses them to **reject bad requests**,
- the websites use them to **check forms and responses**.

Files in `src/schemas/`: `auth.ts`, `company.ts`, `job.ts`, `publisher.ts`, `application.ts`, `resume.ts`.
`src/events/` – names and shapes of events like `job.published`, `application.received`.

It is compiled to `dist/` – Turborepo builds it automatically before `pnpm dev`.
If you change a shape here, **both** the API and the websites may need updating.

The other packages (`ui`, `config`, `email-templates`, `round-sdk`, `rounds/*`, `proctor-sdk`) are plans – README only.

---

## 8. `workers/ai-worker` – the AI writer (Python FastAPI, port 8000)

- Start: `cd workers/ai-worker && uv run python -m ai_worker` – docs at `http://localhost:8000/docs`.
- Reads the same root `.env`.
- Code is in `workers/ai-worker/ai_worker/`:
  - `main.py` – the URLs.
  - `jd/generator.py` – builds the job description.
  - `llm_gateway/` – talks to **Google Gemini** (`LLM_PROVIDER=gemini`, `LLM_MODEL`, backup models in `LLM_FALLBACK_MODELS`).
- URLs:
  - `POST /v1/jd/generate` – **real**: job form + company profile → AI-written JD.
  - `POST /v1/jd/validate` – simple checks before publishing.
  - `POST /v1/resume/parse`, `/v1/resume/score` – fake answers for now.
- The instructions given to the AI come from `prompts/jd_generate.v3.yaml` (change wording → make a new version file).

`workers/workflows` and `workers/activities-ts` – plans only (Temporal, later).

---

## 9. Where the data lives

| Thing | Where |
|---|---|
| All app data (companies, users, jobs, applications, resumes) | **PostgreSQL on Neon (Singapore)** – shared by the whole team. Address = `DATABASE_URL` in `.env` |
| Your old local database | Docker Postgres on port 5433 (still there, line commented in `.env`) |
| Test database | Local Docker, `TEST_DATABASE_URL` |
| Emails in local testing | Mailpit inbox `http://localhost:8025` (only when Gmail is not configured) |
| Logs of the public-link script | `%LOCALAPPDATA%\hireflow\logs` |

Docker (`infra/docker/docker-compose.yml`): `postgres`, `redis` (started but not really used yet), `mailpit`.
Optional extras: `clamav` (virus scan, profile `security`), `minio`, `temporal`.

---

## 10. The main journeys, step by step

### A. HR logs in
1. Dashboard `/login` → API `POST /auth/login` (checks password).
2. If 2-step login is on, the API emails a 6-digit code → `POST /auth/login/verify`.
3. API sets the login cookie. From now on the dashboard sends it automatically.

### B. Create a job and get the AI job description
1. `/company` → `PUT /company-profile` (saved once).
2. `/jobs/new` → `POST /jobs` (saved as DRAFT).
3. "Generate" → `POST /jobs/:id/jd/generate` → API calls **AI Worker** `/v1/jd/generate` → Gemini writes it → saved as **version 1**.
4. HR edits (new versions) → `POST /jobs/:id/approve` → job is **APPROVED** and locked.

### C. Publish to careers site + LinkedIn
1. (Once per company) "Connect LinkedIn" on `/publish` → LinkedIn login → token saved encrypted.
2. `/publish` → preview → `POST /publisher/jobs/:jobId/publish`.
3. The API builds the post (`post-formatter.ts`):
   - **Apply here:** `CAREERS_SITE_URL/<company>/jobs/<job>?src=linkedin`
   - **Company website:** from the company profile.
4. Careers site listing goes live; LinkedIn post is created. Results saved as `job_posts` rows.

> ⚠️ `CAREERS_SITE_URL` must be a **public https address** (from `scripts/start-public.ps1`),
> otherwise the Apply link in LinkedIn is `localhost` and is not clickable for anyone.

### D. A student applies
1. Student opens the Apply link → careers site page.
2. Phone OTP: `POST /public/otp/send` → `POST /public/otp/verify` (in dev mode the code is shown on screen).
3. `POST /public/applications` with the resume:
   checks job is live → checks file → virus scan → duplicate check → saves candidate + application → sends confirmation email.
4. HR sees it in `/applications`.

### E. Invite a team member
1. `/team` → enter email + role → `POST /company/invitations`.
2. API saves the invitation (only a scrambled copy of the secret) and **emails the link** (also shown once to copy).
3. The person opens `HR_DASHBOARD_URL/invite/<secret>` → `/invite/[token]` page → accepts → joins the company.
   (With `HR_DASHBOARD_URL=http://localhost:3000` this works only for people running HireFlow on their own laptop with the same Neon database.)

---

## 11. How to run everything

| Goal | Command |
|---|---|
| Daily update after pulling | `pnpm sync` |
| Start Docker services | `pnpm infra:up` |
| Start API + both websites | `pnpm dev` |
| Start AI Worker | `cd workers/ai-worker && uv run python -m ai_worker` |
| Everything + public Apply link | `powershell -ExecutionPolicy Bypass -File "C:\Users\DHRUV\Downloads\hireflow\scripts\start-public.ps1"` |
| Apply database changes | `pnpm --filter api db:deploy` |
| All checks before a PR | `pnpm check` (+ `uv run pytest` in `workers/ai-worker` if Python changed) |
| Full demo of publish + apply | `pnpm demo:m06-m09` |

Ports: API **4000** · HR Dashboard **3000** · Careers Site **3300** · AI Worker **8000** · Mailpit **8025** · local Postgres **5433**.

---

## 12. `.env` in groups (what each part controls)

| Group | Examples | Controls |
|---|---|---|
| Database | `DATABASE_URL`, `REDIS_URL` | Where data is stored (Neon) |
| App addresses | `API_PORT`, `NEXT_PUBLIC_API_URL`, `HR_DASHBOARD_URL`, `CAREERS_SITE_URL`, `AI_WORKER_URL` | Who calls whom; links in emails/posts |
| Login & security | `LOGIN_2FA`, `SESSION_TTL_HOURS`, `INVITATION_TTL_HOURS`, `CORS_ORIGINS`, `GOOGLE_*` | Logins, sessions, invite lifetime |
| AI | `LLM_PROVIDER`, `LLM_MODEL`, `GEMINI_API_KEY`, timeouts | Which AI writes the JD |
| Email | `GMAIL_USER`, `GMAIL_APP_PASSWORD`, `SMTP_URL` | Real emails (Gmail) or test inbox (Mailpit) |
| LinkedIn | `LINKEDIN_CLIENT_ID/SECRET`, `LINKEDIN_POST_AS`, `LINKEDIN_API_VERSION` | Posting jobs on LinkedIn |
| Secrets | `ENCRYPTION_KEY` | Locks stored LinkedIn tokens – **must be the same for the whole team** |
| Apply flow | `OTP_PROVIDER`, `VIRUS_SCANNER`, `RESUME_STORAGE`, `TURNSTILE_*` | Phone codes, virus scan, resume storage, captcha |
| Feature switches | `FEATURE_PUBLISHER`, `FEATURE_APPLICATIONS` | Turn whole modules on/off |

---

## 13. Where do I change…?

| I want to change… | Go to |
|---|---|
| Text of the LinkedIn / careers post | `apps/api/src/modules/publisher/post-formatter.ts` |
| What the AI writes in the JD | `prompts/jd_generate.v3.yaml` (copy to v4) + `workers/ai-worker/ai_worker/jd/generator.py` |
| The job form fields | `packages/contracts/src/schemas/job.ts` → API `modules/jobs` → dashboard `src/app/jobs/new` |
| Company profile fields | `packages/contracts/src/schemas/company.ts` → API `modules/tenants` → dashboard `src/app/company` |
| A database table | `apps/api/prisma/schema/<topic>.prisma`, then `pnpm --filter api db:migrate` **on local Docker only** |
| Invitation email text | `apps/api/src/modules/auth/invitation-email.ts` |
| Application confirmation email | `apps/api/src/modules/applications/email/confirmation-email.ts` |
| Apply form / job page look | `apps/careers-site/src/app/[company]/jobs/[job]/` and `src/content.ts` |
| Any HR page | `apps/hr-dashboard/src/app/<page>/` |
| Who can do what (roles) | `apps/api/src/modules/auth/permissions.ts` |
| Docker services | `infra/docker/docker-compose.yml` |

---

## 14. Team rules (short)

1. Everyone uses the **same Neon `DATABASE_URL`** and the **same `ENCRYPTION_KEY`**. Share them privately, never in git.
2. Database changes: make the migration on **local Docker**, commit it, then **one person** runs `db:deploy` on Neon. Never run `db:migrate` against Neon.
3. Tests use the local test database, never Neon.
4. Run `pnpm check` before every PR.
5. The public Apply link changes every time the tunnel restarts → re-publish jobs after a restart.

More detail: `docs/TEAM_GUIDE.md` (setup & workflow), `docs/FOLDER_STRUCTURE.md` (owners & status),
`docs/M06_M09_PUBLISH_AND_APPLY.md` (publish/apply deep dive), `docs/HireFlow_AI_Documentation.md` (full spec).

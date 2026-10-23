# JD Workflow — Company Profile → Job → AI JD → HR Approval

| | |
|---|---|
| **Doc modules** | M33 (tenants / company profile), M01 (job form), M02 (JD generator), M04 (approval) |
| **Owner** | Intern 1 (API + dashboard) · Intern 2 (AI worker + prompt) |
| **Status** | Working end to end, tested with real PostgreSQL and real Gemini |

This document explains **what** the JD workflow does, **how it was built step by step**, and **how to run,
test and extend it**. Read section 1 for the idea, section 3 to run it, and section 4 for the build story.

---

## 1. What it does (the 4 steps)

| Step | Who | What happens | Where |
|---|---|---|---|
| **1. Company profile** | HR | Enters company name, industry, description, products/services, website, and other details. Saved **once** in PostgreSQL. | `/company` |
| **2. Create a job** | HR | Enters title, skills, experience, salary, location, etc. HireFlow **loads the saved company profile automatically**. | `/jobs/new` |
| **3. AI writes the JD** | Gemini | Receives **two inputs** — company info + job requirements — and writes a JD that describes both the role and the company. | `/jobs/[id]` |
| **4. Review and approve** | HR | Edits the JD, **verifies the company details**, and approves. The job is then locked. | `/jobs/[id]` |

```text
  Step 1                 Step 2                  Step 3                          Step 4
┌──────────────┐     ┌──────────────┐     ┌───────────────────────┐     ┌──────────────────────────┐
│ Company      │     │ Job form     │     │ Gemini                │     │ HR review                │
│ profile      │────▶│ (+ profile   │────▶│ <company_profile>     │────▶│ edit → new version       │
│ saved once   │     │  auto-loaded)│     │ <job_requirements>    │     │ verify company ✔         │
└──────────────┘     └──────────────┘     │ → JD JSON             │     │ approve → job locked     │
 company_profiles     jobs (DRAFT)        └───────────────────────┘     └──────────────────────────┘
                                           jd_versions (AI)              jd_versions (HR_EDIT)
                                                                         job_approvals
```

**Job status:** `DRAFT` (saved, no JD yet) → `JD_GENERATED` (has a JD, waiting for HR) → `APPROVED` (locked).

---

## 2. Big picture — who talks to whom

```text
Browser (HR dashboard, Next.js :3000)
   │  only ever calls the HireFlow API — no AI keys in the browser
   ▼
HireFlow API (NestJS :4000)  ──────────▶  PostgreSQL (:5432 / :5433)
   │  modules/tenants  → company profile         tenants, company_profiles
   │  modules/jobs     → jobs, versions, approval jobs, jd_versions, job_approvals
   ▼
AI worker (FastAPI :8000)  ──────────▶  Gemini
      jd/generator.py + prompts/jd_generate.v3.yaml
```

The shapes of every request and response live in **one place**: `packages/contracts` (Zod). The browser and
the API validate with the same schemas, and the Python worker mirrors them in `ai_worker/schemas.py`.

---

## 3. How to run it

### 3.1 One-time setup
```bash
pnpm install                                  # also runs "prisma generate" for the API
cd workers/ai-worker && uv sync && cd ../..
cp .env.example .env                          # then fill in LLM_MODEL and GEMINI_API_KEY
```

### 3.2 Start the database and create the tables
```bash
docker compose -f infra/docker/docker-compose.yml up -d postgres
pnpm --filter api db:deploy                   # applies apps/api/prisma/schema/migrations
```

> **Already have PostgreSQL installed on Windows/Mac?** It also uses port 5432, so the API connects to
> *your* PostgreSQL and fails with `P1000: Authentication failed`. Fix: start Docker on another port and
> point `.env` at it:
> ```bash
> POSTGRES_PORT=5433 docker compose -f infra/docker/docker-compose.yml up -d postgres   # PowerShell: $env:POSTGRES_PORT='5433'; docker compose ...
> # .env →  DATABASE_URL=postgresql://hire:hire@localhost:5433/hireflow
> ```

### 3.3 Start the three services (three terminals)
```bash
cd workers/ai-worker && uv run python -m ai_worker      # AI worker   http://localhost:8000/docs
pnpm --filter api build && pnpm --filter api start      # API         http://localhost:4000/api/v1/health
pnpm --filter hr-dashboard dev                          # Dashboard   http://localhost:3000
```

### 3.4 Try the flow
1. Open `http://localhost:3000` → you land on **Jobs**, which asks you to create the company profile.
2. Fill in the company profile → **Save company profile**. You are taken to **New job**.
3. The company profile is shown on the right (loaded automatically). Fill in the job → **Save job & generate JD**.
4. The review page writes the JD (10–30 s). Check the **About the Company** section.
5. Click **Edit**, change something → **Save as new version**. Look at **Version history**.
6. In **Verify and approve**: check the company details, tick the box, type your name → **Approve**.
7. The job shows **Approved** and all edit buttons are gone (locked).

### 3.5 Run the tests
```bash
pnpm check                                    # build + lint + typecheck + tests for all TS packages
cd workers/ai-worker && uv run ruff check . && uv run pytest
```
No test needs a database or an AI key. API tests use in-memory repositories; worker tests use a fake LLM.

---

## 4. How it was built — step by step

### Build step 0 — The starting point
Already done before this work: Step 3 in a **stateless** form. The dashboard sent the job form to
`POST /api/v1/jobs/jd/generate`, the API forwarded it to the AI worker, and Gemini wrote a JD from the job
form only. **Nothing was saved**, there was **no company information**, and **no approval**.

Missing: Step 1 (company profile), Step 2 (saved jobs), Step 4 (edit + approve), and the wiring that gives
Gemini the company info.

### Build step 1 — Agree on the data shapes first (`packages/contracts`)
Before any code, define what every piece of data looks like, so the frontend, API and worker agree.

| File | New schema | Used for |
|---|---|---|
| `schemas/company.ts` | `companyProfileSchema` | Step 1 form + `PUT /company-profile` (name, industry, description ≥ 30 chars required; website must be http(s); size from a fixed list; up to 15 benefits) |
| `schemas/job.ts` | `aboutCompany` on `jdContentSchema` | New JD section written from the company profile (defaults to `""`, so old JDs stay valid) |
| `schemas/job.ts` | `jobStatusSchema`, `jdVersionSchema`, `jobApprovalSchema`, `jobSummarySchema`, `jobDetailSchema` | What the jobs API returns |
| `schemas/job.ts` | `editedJdSchema`, `editJdRequestSchema` | Step 4 edit — title/summary can't be blank, no empty bullets |
| `schemas/job.ts` | `approveJobRequestSchema` | Step 4 approve — `companyDetailsVerified` must be literally `true` |

The Python mirror (`CompanyProfile`, `JdGenerateRequest`, `JdContent.aboutCompany`) was added to
`workers/ai-worker/ai_worker/schemas.py` in the same change, as the repo rule requires.

### Build step 2 — Create the database tables (Prisma)
Prisma 6 was added to the API (`@prisma/client`, `prisma`). Following the repo rule "one `.prisma` file per
area", two files were created in `apps/api/prisma/schema/`:

| File | Table | Key columns | Why |
|---|---|---|---|
| `tenants.prisma` | `tenants` | id, name, slug | One row per company (spec §11) |
| | `company_profiles` | tenant_id (unique), name, industry, description, products_services, website, headquarters, company_size, founded_year, culture, benefits[] | **Step 1** — one profile per company |
| `jobs.prisma` | `jobs` | tenant_id, title, form_data (JSONB), status, current_jd_version_id | **Step 2** — the saved job form |
| | `jd_versions` | job_id, version, source (`AI`/`HR_EDIT`), content (JSONB), **company_snapshot** (JSONB), prompt_version, model | **Steps 3–4** — every JD ever written |
| | `job_approvals` | job_id, jd_version_id, approved_by, company_details_verified, note | **Step 4** — who approved what |

All IDs are UUIDv7 (`@default(uuid(7))`), as the spec asks. The migration was generated with
`prisma migrate dev --name company_profile_and_jobs` and lives in `prisma/schema/migrations/`.

Supporting files:
- `apps/api/prisma.config.ts` — tells the Prisma CLI where the schema is and loads the repo-root `.env`.
- `apps/api/src/common/prisma.service.ts` — one shared database client for the whole API.
- `apps/api/src/common/env.ts` — the API now loads the repo-root `.env` at startup (like the worker does).

### Build step 3 — Step 1 backend: the company profile (`modules/tenants`)
| File | Job |
|---|---|
| `tenant-context.ts` | Answers "which company is this request for?". Auth doesn't exist yet, so `DefaultTenantContext` uses one default tenant (created automatically). When auth lands, only this file changes. |
| `company-profile.repository.ts` | Reads/writes `company_profiles`. An abstract class + a Prisma version, so tests can use an in-memory version. |
| `company-profile.service.ts` | Validates with `companyProfileSchema`. `require()` throws `409 COMPANY_PROFILE_REQUIRED` — used by jobs so Step 2 can't happen before Step 1. |
| `company-profile.controller.ts` | `GET /api/v1/company-profile` and `PUT /api/v1/company-profile` |

"Save once" = `PUT` creates the profile the first time and updates the same row afterwards.

### Build step 4 — Step 2 backend: saved jobs (`modules/jobs`)
| File | Job |
|---|---|
| `jobs.repository.ts` | Reads/writes `jobs`, `jd_versions`, `job_approvals`. Every write is a **compare-and-set** (see build step 6). |
| `jobs.service.ts` | The rules: `create()` validates the form with `jobFormSchema`, **requires the company profile**, and saves a `DRAFT` job. |
| `jobs.controller.ts` | `POST /jobs`, `GET /jobs`, `GET /jobs/:id`, `PATCH /jobs/:id` |

### Build step 5 — Step 3 wiring: give Gemini both inputs
This is the "wire it all" part. The data now flows like this:

1. `JobsService.generateJd()` loads the saved job form **and** the saved company profile.
2. `AiWorkerClient.generateJd(form, company)` sends `{ ...jobForm, company }` to the worker and reads back
   the `X-LLM-Model` / `X-Prompt-Version` headers.
3. The worker's `JdGenerateRequest` validates both parts (bad company data → `422`, LLM never called).
4. `jd/generator.py` turns the company into a `<company_profile>` block and the form into a
   `<job_requirements>` block. `<` and `>` in user text are escaped so nobody can close a tag and inject
   instructions.
5. **New prompt `prompts/jd_generate.v3.yaml`** (v2 kept for history). It keeps all v2 role rules and adds:
   - every company fact must come from `<company_profile>` — "do not invent facts that are not in the input";
   - write `aboutCompany` (2–4 sentences) only from the profile;
   - benefits may only come from the profile's benefits or the job input (salary, work mode).
6. Safety net in code: if no company was sent, `aboutCompany` is forced to `""` even if the model wrote one.
7. The API saves the result as a new `jd_versions` row with `source = AI`, the prompt version, the model,
   and a **snapshot of the company profile** exactly as Gemini received it.

### Build step 6 — Step 4 backend: edit, version history, approve, lock
| Rule | How it is enforced |
|---|---|
| **Never overwrite a JD** | Regenerate → new `AI` row. HR edit → new `HR_EDIT` row. Old rows are never updated, so HR can compare and roll back. |
| **No lost edits between two tabs** | An edit sends `baseVersionId`, an approval sends `jdVersionId`. The database update only succeeds if that is still the current version; otherwise `409 STALE_JD_VERSION`. |
| **HR must verify the company** | `companyDetailsVerified` must be `true` (schema + DB column). The UI shows the exact company snapshot used for that version. |
| **Approve locks the job** | After approval, generate / edit / change form / approve again all return `409 JOB_APPROVED`. |
| **A failed AI call saves nothing** | The version row is only written after the worker returns a valid JD. |

Endpoints: `POST /jobs/:id/jd/generate`, `PATCH /jobs/:id/jd`, `POST /jobs/:id/approve`.

### Build step 7 — The HR dashboard pages
| Page / file | What HR sees |
|---|---|
| `app/company/` + `components/company-profile-form.tsx` | **Step 1.** Company form with friendly errors (same schema as the API). Typing `acme.com` is saved as `https://acme.com`. |
| `app/jobs/` (`jobs-list.tsx`) | Job list with status badges. If there is no company profile yet, it shows "Set up your company profile first". |
| `app/jobs/new/` (`new-job.tsx`) | **Step 2.** The job form, with the saved company profile shown next to it (auto-loaded). "Save job & generate JD" saves the job and opens its review page. |
| `app/jobs/[id]/` (`job-review.tsx`) | **Steps 3–4.** Generates the JD, shows loading / errors / "Try again". Then: **Edit** (save as new version), **Regenerate**, **Copy**, **Version history** (view + restore old versions), **Edit requirements** (save + regenerate). |
| `components/approval-panel.tsx` | **Verify and approve**: shows the company snapshot, warns if the company profile changed since, needs the checkbox + name, then shows "Approved by …". |
| `lib/api.ts` | One small `request()` helper + one function per endpoint; every response is checked against the contract. |

### Build step 8 — Tests and real-world check
| Suite | Tests | What is covered |
|---|---|---|
| `packages/contracts` | 14 | Company schema rules, `aboutCompany` default, HR edit rules, "verified" must be true |
| `apps/api` (`jobs.test.ts`) | 24 | All 4 steps over real HTTP: 404 before profile, profile required for jobs, version history, company sent to worker, every AI error code, stale edits, approval rules, locking |
| `apps/hr-dashboard` | 24 | API client, company form validation, snapshot comparison, JD text/labels |
| `workers/ai-worker` | 65 (+1 live, opt-in) | Company block in prompt, injection escaping, `aboutCompany` dropped without a profile, invalid company → 422 |

**End-to-end check (real PostgreSQL + real Gemini `gemini-3.5-flash`):** saved a test company profile →
created a job → generated a JD (≈19 s) → edited it (version 2) → tried to approve version 1 (`409`) → tried
approving without verifying (`400`) → approved → tried to regenerate (`409 JOB_APPROVED`). The generated JD
used only the profile's facts (products, size, founding year, culture, benefits) and kept the role sections
grounded in the job form. All rows were confirmed in the database. All dashboard pages render without errors.

---

## 5. API reference (all under `/api/v1`)

| Method | Path | Body | Returns | Step |
|---|---|---|---|---|
| GET | `/company-profile` | — | `CompanyProfileRecord` or `404` | 1 |
| PUT | `/company-profile` | `CompanyProfile` | `CompanyProfileRecord` | 1 |
| POST | `/jobs` | `JobForm` | `JobDetail` (status `DRAFT`) | 2 |
| GET | `/jobs` | — | `JobSummary[]` | — |
| GET | `/jobs/:id` | — | `JobDetail` | — |
| PATCH | `/jobs/:id` | `JobForm` | `JobDetail` | 2 |
| POST | `/jobs/:id/jd/generate` | — | `JobDetail` (new `AI` version) | 3 |
| PATCH | `/jobs/:id/jd` | `{ baseVersionId, content }` | `JobDetail` (new `HR_EDIT` version) | 4 |
| POST | `/jobs/:id/approve` | `{ jdVersionId, approvedBy, companyDetailsVerified: true, note? }` | `JobDetail` (`APPROVED`) | 4 |

The old stateless `POST /jobs/jd/generate` was **removed**: it skipped the company profile and saved nothing.

### Error codes (body is always `{ statusCode, code, message }`)
| Status | Code | Meaning |
|---|---|---|
| 400 | `INVALID_COMPANY_PROFILE`, `INVALID_JOB_FORM`, `INVALID_JD`, `INVALID_APPROVAL` | Bad input; `issues[]` lists the fields |
| 404 | `COMPANY_PROFILE_NOT_FOUND`, `JOB_NOT_FOUND` | Not saved yet / wrong id |
| 409 | `COMPANY_PROFILE_REQUIRED` | Do step 1 first |
| 409 | `JD_NOT_GENERATED` | Approve/edit before any JD exists |
| 409 | `STALE_JD_VERSION` | Someone saved a newer version — reload |
| 409 | `JOB_APPROVED` | The job is locked |
| 422–504 | `LLM_*`, `AI_WORKER_*` | AI problems (quota, timeout, unavailable…) — unchanged from before, safe messages only |

---

## 6. Why it was built this way

- **Contracts first.** One schema → the form, the API and the worker can never disagree on what "valid" means.
- **Company snapshot on every version.** If HR edits the company profile later, old JDs still show the
  exact facts they were written from — that is what HR verifies in step 4.
- **Repositories as abstract classes.** Services don't know about Prisma, so all API tests run in memory in
  under a second, and CI needs no database.
- **Generation happens on the review page.** The new-job page only saves the job and redirects with
  `?generate=1`. Loading, errors and "Try again" therefore live in one place, and a failed AI call never
  creates a duplicate job.
- **Prompt versioning.** v3 is a new file; the version is stored on each JD row, so any JD can be traced
  back to the exact prompt that wrote it.

---

## 7. Known limits and next steps

| Not done yet | Why / who |
|---|---|
| Auth + roles; `approvedBy` is a typed name | M28 not implemented. Replace `DefaultTenantContext` and use the logged-in user. |
| Single tenant, no Row Level Security | Comes with M28/M33. Every table already has `tenant_id`. |
| M03 validation before approval (bias / salary checks) | `/v1/jd/validate` is rules-only today; call it before allowing approve. |
| Events `job.created`, `jd.generated`, `jd.approved` | M29/M30 orchestrator + event bus. A `TODO(M29)` marks the spot in `jobs.service.ts`. |
| Diff view between versions; multi-approver | Spec "nice to have" items for M04. |
| Editing after approval | Spec: editing a published job should create a new version and ask about live posts (M06). |

---

## 8. Troubleshooting

| Symptom | Fix |
|---|---|
| `P1000: Authentication failed … hire` | Another PostgreSQL is on 5432 — see the note in 3.2. |
| Dashboard says "Could not reach the HireFlow API" | Start the API; check `NEXT_PUBLIC_API_URL`. |
| `AI generation is not configured on the server` | Set `LLM_MODEL` and `GEMINI_API_KEY` in `.env`, restart the worker. |
| `The AI service is busy or its usage quota was reached` | Gemini free-tier limit — wait a minute. |
| `relation "jobs" does not exist` | Run `pnpm --filter api db:deploy`. |
| TypeScript can't find `@prisma/client` types | Run `pnpm --filter api exec prisma generate` (normally done by `pnpm install`). |

---

## 9. Files added or changed

```text
packages/contracts/src/schemas/company.ts            NEW  company profile schema
packages/contracts/src/schemas/job.ts                     aboutCompany, job/version/approval/edit schemas
packages/contracts/tests/company.test.ts             NEW
apps/api/prisma.config.ts                            NEW  Prisma CLI config (loads root .env)
apps/api/prisma/schema/tenants.prisma                NEW  tenants, company_profiles
apps/api/prisma/schema/jobs.prisma                   NEW  jobs, jd_versions, job_approvals
apps/api/prisma/schema/migrations/…                  NEW  first migration
apps/api/src/common/{api-error,env,prisma.service}.ts NEW
apps/api/src/modules/tenants/*                       NEW  step 1
apps/api/src/modules/jobs/{jobs.controller,jobs.service,jobs.repository}.ts  NEW  steps 2–4
apps/api/src/modules/jobs/ai-worker.client.ts             sends company, returns model + prompt version
apps/api/src/modules/jobs/jd-generation.{controller,service}.ts  REMOVED (stateless endpoint)
apps/api/tests/{jobs.test,in-memory}.ts              NEW  (replaces jd-generation.test.ts)
workers/ai-worker/ai_worker/{schemas,main}.py             CompanyProfile, JdGenerateRequest
workers/ai-worker/ai_worker/jd/generator.py               <company_profile> block, aboutCompany rule
workers/ai-worker/tests/test_jd_company.py           NEW
prompts/jd_generate.v3.yaml                          NEW  two-input prompt
apps/hr-dashboard/src/app/company/*                  NEW  step 1 page
apps/hr-dashboard/src/app/jobs/{page,jobs-list}.tsx  NEW  job list
apps/hr-dashboard/src/app/jobs/new/new-job.tsx       NEW  step 2 (replaces jd-generator.tsx)
apps/hr-dashboard/src/app/jobs/[id]/*                NEW  steps 3–4 review page
apps/hr-dashboard/src/components/{company-profile-form,company-card,approval-panel,version-history}.tsx  NEW
apps/hr-dashboard/src/components/{jd-panel,job-form-panel,ui}.tsx   saved edits, labels, badges
apps/hr-dashboard/src/lib/{api,company-form,jd,job-form}.ts
infra/docker/docker-compose.yml                           POSTGRES_PORT override
.env.example                                              DEFAULT_TENANT_SLUG, port note
```

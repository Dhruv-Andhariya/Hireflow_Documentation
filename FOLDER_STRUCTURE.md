# HireFlow — Folder Structure Guide

A short tour of every folder: what it is for, who owns it, and whether it has real code yet.

**Status key:** ✅ working code · 🟡 partial / mock · ⬜ placeholder (only a README describing the plan)

Owners: **I1** = Intern 1 (Core & HR), **I2** = Intern 2 (AI), **I3** = Intern 3 (Candidate & Comms).
Module numbers (M01, M02 …) refer to [HireFlow_AI_Documentation.md](HireFlow_AI_Documentation.md).

---

## 1. The big picture

HireFlow is a **monorepo**: many apps and libraries in one git repo.

- **TypeScript side** is managed by **pnpm workspaces + Turborepo** (`apps/*`, `packages/*`, two `workers/*`).
- **Python side** (`workers/ai-worker`) is managed separately by **uv**.
- They talk over **HTTP**, and share data shapes through `packages/contracts` (TS) mirrored in `ai_worker/schemas.py` (Python).

```text
Browser (Next.js apps)  ──►  apps/api (NestJS :4000)  ──►  workers/ai-worker (FastAPI :8000)  ──►  Gemini LLM
                                   │                                │
                                   ▼                                ▼
                        Postgres / Redis (infra/docker)      prompts/*.yaml
```

The browser never talks to the AI worker or the LLM directly — only to the API.

---

## 2. Root files

| File | What it does |
|---|---|
| `package.json` | Root scripts: `pnpm dev`, `pnpm check` (build + lint + typecheck + test), `pnpm sync`, `pnpm infra:up/down`, `pnpm format` |
| `pnpm-workspace.yaml` | Tells pnpm which folders are JS packages |
| `turbo.json` | Turborepo task pipeline (`build`, `dev`, `lint`, `typecheck`, `test`) — builds dependencies first (`^build`) |
| `pnpm-lock.yaml` | Exact JS dependency versions (never edit by hand) |
| `.env.example` → `.env` | Environment variables (DB URL, AI worker URL, Gemini key…). `.env` is git-ignored |
| `.editorconfig`, `.prettierrc` | Formatting rules shared by everyone |
| `.gitattributes`, `.gitignore` | Git line-ending and ignore rules |
| `README.md` | Quick start |
| `AGENTS.md` | Auto-generated note for AI coding agents about the Turborepo version |

---

## 3. `apps/` — things users open or call

### `apps/api/` — NestJS core API, port 4000 (✅ running)
The single backend every frontend calls. All routes are prefixed `/api/v1`.

| Path | Purpose |
|---|---|
| `src/main.ts` | Boots Nest, sets `/api/v1` prefix and CORS for the local Next.js apps |
| `src/app.module.ts` | Root module — each owner registers their module here with one import line |
| `src/health.controller.ts` | `GET /api/v1/health` |
| `src/common/` ⬜ | Shared Nest helpers: guards, filters, event bus/outbox (M30) — I1 |
| `prisma/schema/` 🟡 | Database schema, **one `.prisma` file per owner** to avoid conflicts. Only `base.prisma` exists so far |
| `tests/` ✅ | API tests (currently `jd-generation.test.ts`) |

**`src/modules/`** — one folder per business area. Other modules may import a module only through its `index.ts`.

| Module | Owner | Status | What it will do |
|---|---|---|---|
| `jobs/` | I1 | ✅ | M01–M04: job form, JD generation. **Implemented:** `POST /api/v1/jobs/jd/generate` → calls AI worker (`ai-worker.client.ts`). Not yet: saving jobs, approval |
| `auth/` | I1 | ⬜ | M28: login, roles (super_admin, company_admin, hr, interviewer, candidate) |
| `tenants/` | I1 | ⬜ | M33: companies, per-company settings, row-level security |
| `rounds-config/` | I1 | ⬜ | M05: which test rounds a job has, their order and cut-offs |
| `proctoring/` | I1 | ⬜ | M21: proctor events, heartbeat, cheating flags |
| `analytics/` | I1 | ⬜ | M27: hiring funnel and source stats |
| `audit/` | I1 | ⬜ | M32: log of every AI decision and human override |
| `ats/` | I2 | ⬜ | M10–M11: asks the AI worker to parse + score resumes, stores scores |
| `screening/` | I2 | ⬜ | M12: AI voice screening (LiveKit) |
| `evaluator/` | I2 | ⬜ | M22: scores round attempts with reasons |
| `feedback/` | I2 | ⬜ | M23: writes feedback / report-card content |
| `applications/` | I3 | ⬜ | M08–M09: apply link, intake, dedupe, consent, resume upload |
| `notifications/` | I3 | ⬜ | M24: email (Mailpit locally), WhatsApp later |
| `publisher/` | I3 | ⬜ | M06–M07: post jobs to careers page / job boards |
| `scheduler/` | I3 | ⬜ | M25: interview slot booking (Google Calendar) |
| `round-engine/` | I3 | ⬜ | M13: runs test rounds via the `round-sdk` plugin interface |

### `apps/hr-dashboard/` — Next.js, port 3000, I1 (✅ first page working)
The HR user's web app.

| Path | Purpose |
|---|---|
| `src/app/` | Pages (Next.js App Router). `jobs/new/` = job form + "Generate Job Description" |
| `src/components/` | UI pieces: `job-form-panel.tsx` (form), `jd-panel.tsx` (shows/edits the generated JD), `tag-input.tsx` (skills input), `ui.tsx` |
| `src/lib/` | Logic without UI: `api.ts` (calls the API), `job-form.ts` (draft → validated `JobForm`), `jd.ts` (JD helpers) |
| `tests/` | Vitest tests for the `lib/` files |

### Other apps (⬜ empty for now)
| App | Port | Owner | For |
|---|---|---|---|
| `candidate-app/` | 3100 | I3 | Candidates taking rounds / tracking status |
| `admin-console/` | 3200 | I1 | Super-admin / company admin |
| `careers-site/` | 3300 | I3 | Public job listings and apply page |

---

## 4. `workers/` — background services

### `workers/ai-worker/` — Python FastAPI, port 8000, I2 (✅ JD generation real, rest mock)
All AI work happens here. Run with `uv run python -m ai_worker`; Swagger at `http://localhost:8000/docs`.

| Path | Purpose |
|---|---|
| `ai_worker/main.py` | HTTP endpoints: `/v1/jd/generate` ✅, `/v1/jd/validate` 🟡 (rule checks only), `/v1/resume/parse` + `/v1/resume/score` 🟡 (mocks) |
| `ai_worker/schemas.py` | Pydantic models — **must mirror `packages/contracts`** |
| `ai_worker/config.py` | Reads settings from the repo-root `.env` (LLM provider, model, API key, timeouts) |
| `ai_worker/llm_gateway/` | "One door for all AI calls": `provider.py` (interface), `gemini.py` (Gemini implementation), `prompts.py` (loads `prompts/*.yaml`), `errors.py` |
| `ai_worker/jd/` | JD generator: turns a `JobForm` into the `<job_requirements>` block, calls the LLM, validates and cleans the output |
| `ai_worker/parsing/`, `scoring/`, `evaluation/`, `feedback/` | ⬜ Future: resume parsing, ATS scoring, round evaluation, feedback |
| `tests/` | Pytest suite. `fakes.py` provides a fake LLM so tests **never call the real API** |
| `pyproject.toml`, `uv.lock` | Python dependencies |

### `workers/workflows/` (⬜, I1)
M29 orchestrator — `JobWorkflow` and `CandidateWorkflow` (Temporal, or a simple state machine first). The glue that moves a job/candidate from step to step.

### `workers/activities-ts/` (⬜, I1)
Temporal "activities" in TypeScript: the actual side effects (send email, publish job, book calendar).

---

## 5. `packages/` — shared libraries (imported, never run alone)

| Package | Owner | Status | Purpose |
|---|---|---|---|
| `contracts/` | All review | ✅ | **The most important shared package.** Zod schemas (`schemas/job.ts`: `JobForm`, `JdContent`; `schemas/resume.ts`: `ParsedProfile`, `AtsScore`) and the event envelope (`events/`). Both frontend and API validate against these. Change = small PR, merged same day |
| `ui/` | I1 | ⬜ | Shared React components (shadcn/ui) |
| `config/` | I1 | ⬜ | Shared ESLint / TypeScript / Tailwind configs |
| `email-templates/` | I3 | ⬜ | React Email templates: rejection, invite, offer |
| `round-sdk/` | I3 | ⬜ | The `RoundPlugin` interface every test round implements |
| `proctor-sdk/` | I3 | ⬜ | Browser-side proctoring library |
| `rounds/*` | I3 | ⬜ | One plugin per round type: `aptitude`, `coding`, `sql`, `website`, `gaming`, `communication`, `group-discussion` |

---

## 6. Supporting folders

| Folder | Purpose |
|---|---|
| `prompts/` | Versioned LLM prompts in YAML (I2). Never hard-code prompts in code. Bump `version` on any wording change — e.g. `jd_generate.v1.yaml` → `jd_generate.v2.yaml` (current) |
| `docs/` | `HireFlow_AI_Documentation.md` (full spec), `TEAM_GUIDE.md` (setup + workflow), this file |
| `infra/docker/` | `docker-compose.yml` for local Postgres (with pgvector), Redis, Mailpit (fake inbox :8025). MinIO (`--profile storage`) and Temporal (`--profile workflow`) are opt-in |
| `infra/k8s/`, `infra/terraform/` | ⬜ Production deployment, later (v1.0) |
| `seed/` | Demo data and sample resumes (`seed/resumes/`) for testing |
| `scripts/sync.sh` | `pnpm sync` — pull, install JS + Python deps, apply DB migrations. Run daily |
| `.github/` | `CODEOWNERS` (who owns which folder), PR template, `workflows/ci.yml` (CI runs `pnpm check` + ruff + pytest on every PR) |
| `.vscode/` | Recommended extensions, editor settings, tasks |

---

## 7. Example: how one feature crosses the folders (JD generation)

```text
apps/hr-dashboard   src/app/jobs/new          HR fills the form
                    src/lib/job-form.ts       validated with packages/contracts → JobForm
                    src/lib/api.ts            POST /api/v1/jobs/jd/generate
        ▼
apps/api            modules/jobs/             validates again, forwards to the AI worker
                    ai-worker.client.ts       POST http://localhost:8000/v1/jd/generate
        ▼
workers/ai-worker   main.py → jd/generator.py builds <job_requirements> from the form
                    llm_gateway/prompts.py    fills prompts/jd_generate.v2.yaml
                    llm_gateway/gemini.py     calls Gemini, gets JSON
                    jd/generator.py           validates + cleans → JdContent
        ▲
        └── JdContent travels back the same way and is shown in jd-panel.tsx
```

Most future features will follow the same pattern: **app → api module → (worker) → contracts shape back**.

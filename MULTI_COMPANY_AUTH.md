# Multi-Company Login, Roles and Data Isolation

| | |
|---|---|
| **Doc modules** | M28 (auth and roles), M33 (multi-tenancy), M32 (audit log, minimal) |
| **Owner** | Intern 1 (Core & HR) |
| **Status** | Working and tested: 23 two-company tests pass against a real PostgreSQL database. Live Gemini generation was not re-checked (Google returned "high demand" errors on the test day). |
| **Builds on** | [JD_WORKFLOW.md](JD_WORKFLOW.md): the JD module was **not rebuilt**, only connected to login |

This document explains **everything that was added to turn HireFlow from "one HR, one company" into "many
companies, each with its own team"**, in plain language. Read sections 1–3 for the idea, section 4 to run it,
section 5 for how a request is checked, and sections 6–10 for the details of every change.

---

## Contents

1. [The problem we solved](#1-the-problem-we-solved)
2. [The three roles](#2-the-three-roles)
3. [Key ideas in simple words](#3-key-ideas-in-simple-words)
4. [How to run it (step by step)](#4-how-to-run-it-step-by-step)
5. [What happens on every request](#5-what-happens-on-every-request)
6. [Database changes](#6-database-changes)
7. [Backend (API) changes](#7-backend-api-changes)
8. [How the existing JD module was connected](#8-how-the-existing-jd-module-was-connected)
9. [Frontend (dashboard) changes](#9-frontend-dashboard-changes)
10. [Security features explained](#10-security-features-explained)
11. [API reference](#11-api-reference)
12. [Error codes](#12-error-codes)
13. [Testing](#13-testing)
14. [Known limits and next steps](#14-known-limits-and-next-steps)
15. [Troubleshooting](#15-troubleshooting)
16. [Every file added or changed](#16-every-file-added-or-changed)
17. [Glossary](#17-glossary)

---

## 1. The problem we solved

**Before:** HireFlow worked for exactly one company.

- There was **no login**. Anyone who could reach the API could read and change everything.
- Every request was put into one hard-coded company called `default` (`DefaultTenantContext` +
  `DEFAULT_TENANT_SLUG`). A second company had nowhere to go.
- When approving a JD, HR **typed their name** into a box. Anyone could type any name.
- The CORS rule (`/localhost:\d+$/`) would also have accepted look-alike sites such as `http://evil-localhost:80`.

**What already worked and was kept as it is:** company profile, job form, AI JD generation through the Python
worker and Gemini, HR editing, regeneration, version history, company snapshots, and approval with locking.

**After:**

- Staff log in with email and password.
- Each company has its own users, jobs, JDs, versions and approvals, and **cannot see or touch another
  company's data**, even by guessing IDs.
- Three roles: **Platform Admin**, **Company Admin**, **HR Member**.
- The person who approves a JD is recorded from their login, not typed in.
- Important actions are written to an **audit log**.

---

## 2. The three roles

| Role | Level | What they can do | What they can NOT do |
|---|---|---|---|
| **Platform Admin** | Whole platform | Create companies, assign each company's first Company Admin, suspend or re-activate companies | Read any company's jobs, JDs or profile. They manage *companies*, not *recruitment data* |
| **Company Admin** | One company | Everything an HR Member can do, plus edit the company profile and manage the team (invite, change role, disable) | Touch other companies; make anyone a Platform Admin |
| **HR Member** | One company | See the company profile, create and edit jobs, generate, edit and approve JDs, view version history | Edit the company profile, manage the team, use admin pages |

The exact list lives in **one file**: `apps/api/src/modules/auth/permissions.ts`.

```ts
'company.read':   ['COMPANY_ADMIN', 'HR_MEMBER'],
'company.update': ['COMPANY_ADMIN'],
'jobs.read':      ['COMPANY_ADMIN', 'HR_MEMBER'],
'jobs.write':     ['COMPANY_ADMIN', 'HR_MEMBER'],
'jd.generate':    ['COMPANY_ADMIN', 'HR_MEMBER'],
'jd.edit':        ['COMPANY_ADMIN', 'HR_MEMBER'],
'jd.approve':     ['COMPANY_ADMIN', 'HR_MEMBER'],   // the spec says HR can "create/approve jobs"
'team.manage':    ['COMPANY_ADMIN'],
```

To change who can do what, change this table. Every route reads from it.

**How does someone become a Platform Admin?** Only through a command run on the server (section 4.3).
There is no button, API or sign-up form that grants it, so nobody can give themselves platform powers.

---

## 3. Key ideas in simple words

| Idea | Simple explanation |
|---|---|
| **Tenant / company** | One customer company. In the database it is a row in `tenants`. "Tenant" and "company" mean the same thing here. |
| **User** | One person with an email and password (`users`). One person can belong to more than one company. |
| **Membership** | The link "this user belongs to this company with this role" (`memberships`). It can be `ACTIVE` or `DISABLED`. |
| **Session** | "You are logged in." Created at login, stored in the database (`sessions`), and ends at logout or after 12 hours. |
| **Session cookie** | A random secret the browser keeps and sends with every request so the API knows who you are. JavaScript cannot read it (`httpOnly`). |
| **Invitation** | A one-time link that lets a new person join a company. It expires after 7 days and works only once. |
| **Guard** | A gatekeeper that runs **before** every API endpoint and decides "allowed" or "not allowed". |
| **Suspended company** | The company and its data still exist, but nobody in it can use jobs, JDs or the profile until a Platform Admin re-activates it. |
| **Audit log** | A diary of important actions: who did what, and when. |

### How the pieces relate

```text
             ┌─────────────┐
             │   users     │  Priya (priya@acme.dev)
             └──────┬──────┘
                    │ has many
             ┌──────▼──────┐
             │ memberships │  Priya → Acme   (HR_MEMBER, ACTIVE)
             └──────┬──────┘  Priya → Beta   (COMPANY_ADMIN, ACTIVE)   ← one person, two companies
                    │ belongs to
             ┌──────▼──────┐
             │  tenants    │  Acme (ACTIVE)     Beta (SUSPENDED)
             └──────┬──────┘
                    │ owns
   ┌────────────────┼─────────────────┬──────────────────┐
company_profiles   jobs ──► jd_versions ──► job_approvals   (every row carries tenant_id)
```

A **session** remembers *which* of the user's companies they are working in right now. If a user belongs to
two companies, they switch with the company menu in the header.

---

## 4. How to run it (step by step)

### 4.1 Start the database and apply the new migration

```bash
pnpm infra:up                                  # or: POSTGRES_PORT=5433 docker compose -f infra/docker/docker-compose.yml up -d postgres
pnpm --filter api db:deploy                    # applies 20261013090000_multi_company_auth
```

The migration only **adds** things. Existing companies, profiles, jobs, versions and approvals are kept (see 6.3).

### 4.2 Update your `.env`

Remove `DEFAULT_TENANT_SLUG` (it is no longer used). New optional settings, already in `.env.example`:

| Variable | Default | Meaning |
|---|---|---|
| `HR_DASHBOARD_URL` | `http://localhost:3000` | Invitation links point here (`/invite/<token>`) |
| `CORS_ORIGINS` | empty (= any `http://localhost:<port>`) | In production, the dashboard address(es), comma-separated |
| `COOKIE_SECURE` | `false` | Set `true` behind HTTPS (automatic when `NODE_ENV=production`) |
| `SESSION_TTL_HOURS` | `12` | How long a login lasts |
| `INVITATION_TTL_HOURS` | `168` (7 days) | How long an invitation link works |
| `LOGIN_MAX_ATTEMPTS` | `10` | Failed logins allowed per email + IP in 15 minutes |
| `PLATFORM_ADMIN_*` | empty | Only used by the command in 4.3. Do not leave the password in the file. |

### 4.3 Create the first Platform Admin (one time)

```bash
PLATFORM_ADMIN_EMAIL=you@example.com PLATFORM_ADMIN_NAME="Your Name" PLATFORM_ADMIN_PASSWORD='at-least-10-chars' \
  pnpm --filter api admin:create
```

- If the email is new, the account is created as Platform Admin.
- If it exists, it is promoted to Platform Admin (and the password is reset only if you pass one).
- Passwords must be 10–128 characters.

### 4.4 Start everything

```bash
pnpm --filter api dev                 # API on :4000
cd workers/ai-worker && uv run python -m ai_worker   # AI worker on :8000
pnpm --filter hr-dashboard dev        # dashboard on :3000
```

### 4.5 Walk-through: onboard a company

1. Open `http://localhost:3000`. You are sent to **/login**. Log in as the Platform Admin.
2. You land on **Companies** (`/admin`). Fill in **company name**, **slug** (short unique name, e.g. `acme-labs`)
   and the **Company Admin email**. Click **Create company**.
3. A **one-time invitation link** appears. Copy it and send it to that person (there is no email sending yet).
   The link is shown only once, because the database keeps only a scrambled copy (hash).
4. The Company Admin opens the link, enters their name and a password, and is logged in to the company.

### 4.6 Walk-through: the existing "TechFlitter" company

The company that existed before login (slug `default`, name "TechFlitter") is now a normal company with all its
jobs, versions and approvals. It simply has no users yet:

1. As Platform Admin open **Companies → TechFlitter**.
2. Under **Assign a Company Admin**, enter the email of the person who should own it, then **Create invitation link**.
3. They open the link and set a password. They now see the old jobs exactly as before.

### 4.7 Walk-through: Company Admin builds the team and the JD flow runs

1. **Company profile** (`/company`): the Company Admin saves it once (only Company Admins can edit it).
2. **Team** (`/team`): invite an HR Member (email + role) and send them the link.
3. The HR Member accepts and logs in. They:
   - open **New job**: the saved company profile is **loaded automatically**; they only enter job details
     (title, skills, experience, salary, location…),
   - click **Save job & generate JD**: the AI writes version 1 using job details + *their own* company profile,
   - edit, regenerate, look at the version history, then tick "company details are correct" and **Approve**.
     The approval is recorded under the logged-in person's name. The job is now locked.
4. The Company Admin can later change roles or **Disable** a member on the Team page. A disabled member loses
   access on their very next click.

### 4.8 Walk-through: suspend a company

Platform Admin → **Companies → (company) → Suspend company**. Members of that company can still log in, but they
see "This company is suspended" and every job/profile/team request is refused. Nothing is deleted.
**Activate company** restores access.

---

## 5. What happens on every request

This is the most important part. Every call from the dashboard to the API goes through these checks, in order.
They run on the **server**, so changing the browser or calling the API directly does not get around them.

```text
Browser ──► API request (cookie: hireflow_session=<random secret>)
              │
              ▼
   ① CSRF check      Is it a write (POST/PUT/PATCH/DELETE)? Then the header
                      "X-Requested-With: hireflow" must be there.           ✗ → 403 CSRF_CHECK_FAILED
              │
   ② Route label     Does this endpoint say who may use it?
                      (@Public / @Authenticated / @PlatformAdminOnly / @RequirePermission)
                      No label → refused ("fail closed").                     ✗ → 403 FORBIDDEN
              │
   ③ Who are you?    Hash the cookie, find the session in the database.
                      Missing, expired, or user disabled?                     ✗ → 401 UNAUTHENTICATED
              │
   ④ Which company?  Load the user's membership in the session's company.
                      Must be ACTIVE.                                         ✗ → 403 NO_COMPANY
                      The company must be ACTIVE (not suspended).             ✗ → 403 COMPANY_SUSPENDED
              │
   ⑤ May you do it?  Look up the role in the permission table.                ✗ → 403 FORBIDDEN
              │
              ▼
   ⑥ The endpoint runs. It asks TenantContext "which company?" and gets the
      company FROM THE SESSION. Every database query has  WHERE tenant_id = <that company>.
      A job id from another company simply is not found                      → 404 JOB_NOT_FOUND
```

Why check the membership **on every request** instead of once at login? So that disabling a member, changing a
role or suspending a company takes effect **immediately**, without waiting for the person to log out.

Why answer **404 (not found)** for another company's job instead of 403? So an outsider cannot even learn that
the job exists.

---

## 6. Database changes

### 6.1 New tables

| Table | File | What it stores |
|---|---|---|
| `users` | `prisma/schema/auth.prisma` | email (stored in lower case, unique), name, password hash, `is_platform_admin`, `is_active`, last login |
| `sessions` | `auth.prisma` | **hash** of the session token, user, current company, expiry |
| `memberships` | `auth.prisma` | user + company + role (`COMPANY_ADMIN` / `HR_MEMBER`) + status (`ACTIVE` / `DISABLED`). One per user per company |
| `invitations` | `auth.prisma` | company, email, role, **hash** of the invite token, who invited, expiry, accepted/revoked time |
| `audit_logs` | `prisma/schema/audit.prisma` | company, who, action (e.g. `job.approved`), subject, extra details, time |

### 6.2 Changed tables (only new columns and constraints)

| Table | Change | Why |
|---|---|---|
| `tenants` | new `status` (`ACTIVE` / `SUSPENDED`, default `ACTIVE`) | so a Platform Admin can suspend a company |
| `jobs` | new `created_by_id` (may be empty) | who created the job |
| `jobs` | new unique pair `(id, tenant_id)` | needed for the safety rule below |
| `jd_versions` | link to job changed from `job_id` to the pair `(job_id, tenant_id)` | the **database itself** refuses a version whose company differs from its job's company |
| `job_approvals` | same pair link + new `approved_by_id` | same protection; record the approver's user id |
| `jd_versions.created_by` | existing column, now filled with the user id | who wrote each version |

### 6.3 How existing data was kept safe

The migration file is `apps/api/prisma/schema/migrations/20261013090000_multi_company_auth/migration.sql`.

1. **A backup was taken first** (`pg_dump`) before applying anything.
2. **Only additions.** No table, column or row is dropped. The two old links were swapped for the stronger
   `(job_id, tenant_id)` links, which describe the same relationship more strictly.
3. **Safety check at the top.** Before changing the links, the migration checks that every version and approval
   has the same company as its job. If not, it **stops with an error** instead of guessing.
4. **Ownership was clear.** All existing data belonged to the single `default` tenant, so nothing had to be
   moved. Existing companies become `ACTIVE`. Old jobs/approvals have an empty `created_by_id` /
   `approved_by_id` because no login existed when they were made; their typed `approved_by` names are kept.
5. **Checked after applying:** 1 tenant, 2 jobs, 3 versions, 2 approvals, 1 profile — the same as before.

> The migration SQL was generated with `prisma migrate diff` (because `prisma migrate dev` does not run in a
> non-interactive terminal) and reviewed by hand before it was applied with `prisma migrate deploy`.

---

## 7. Backend (API) changes

All under `apps/api/src/`.

### 7.1 New `modules/auth/` (login and permissions)

| File | What it does, simply |
|---|---|
| `password.ts` | Turns a password into a **scrypt hash** (slow on purpose, so stolen hashes are hard to crack) and checks a password against it. Uses Node's built-in crypto, no new package. |
| `tokens.ts` | Makes 256-bit random tokens and their SHA-256 hashes; reads the session/invitation lifetimes and dashboard URL from `.env`. |
| `cookies.ts` | Reads and writes the `hireflow_session` cookie (httpOnly, SameSite=Lax, Secure in production). |
| `rate-limit.ts` | Counts attempts per key and answers `429 TOO_MANY_ATTEMPTS` after too many in 15 minutes. |
| `permissions.ts` | The permission table from section 2. |
| `access.ts` | The route labels `@Public`, `@Authenticated`, `@PlatformAdminOnly`, `@RequirePermission(...)` and the shape of "who is logged in" (`AuthInfo`). |
| `sessions.service.ts` | Creates sessions at login, looks them up on every request, re-checks membership and company status, switches company, deletes sessions at logout. |
| `auth.service.ts` | Login (with rate limit and same-speed failure), logout, switch company, preview and accept invitations. |
| `team.service.ts` | Company team: list members and pending invitations, create/revoke invitations, change role or status. Refuses to remove the **last active Company Admin**. |
| `auth.guard.ts` | The gatekeeper from section 5. Registered **globally**, so it protects every route automatically. |
| `auth.controller.ts` | Endpoints under `/auth/*`. |
| `team.controller.ts` | Endpoints under `/company/*` (Company Admin only). |
| `auth.module.ts`, `index.ts` | Wiring; other modules import only through `index.ts`. |

**Login, step by step**

1. Check the email/password format.
2. Count the attempt (per IP, and per IP + email).
3. Find the user. **Always run the password check**, even if the email does not exist (with a dummy hash), so the
   response time does not reveal which emails are registered.
4. Wrong email, wrong password, or disabled user → the **same** `401 INVALID_CREDENTIALS` message, and an
   `auth.login_failed` audit entry.
5. Success → create a session in the user's first active company, set the cookie, record `auth.login`.

**Accepting an invitation, step by step**

1. Find the invitation by the **hash** of the token; it must be unused, not revoked and not expired.
2. If the email **already has an account**, the person must type **that account's password**. This stops anyone
   holding the link from taking over someone else's account.
3. If the email is new, a name and password (10+ characters) create the account.
4. In one database transaction: mark the invitation used (only the first click can win), create or re-activate the
   membership with the invited role.
5. Log the person in to that company.

### 7.2 New `modules/audit/`

`AuditService.record()` writes one row to `audit_logs`. If writing the audit row fails, the error is logged but the
user's action is not undone. Never stores passwords, tokens or JD text.

**Actions recorded:** `auth.login`, `auth.login_failed`, `auth.logout`, `auth.company_switched`,
`company.created`, `company.suspended`, `company.activated`, `invitation.created`, `invitation.revoked`,
`invitation.accepted`, `membership.updated`, `company_profile.saved`, `job.created`, `jd.generated`, `jd.edited`,
`job.approved`, and `platform_admin.granted` (from the CLI).

### 7.3 `modules/tenants/` (company)

| File | Change |
|---|---|
| `tenant-context.ts` | **`DefaultTenantContext` removed.** Replaced by `RequestTenantContext`, which reads the company and the acting user from the session the guard checked. Added `currentActor()` (who is acting). |
| `company-profile.controller.ts` | `GET` needs `company.read`; `PUT` needs `company.update` (Company Admin). |
| `company-profile.service.ts` | Records `company_profile.saved` in the audit log. Logic unchanged. |
| `companies.service.ts` / `companies.controller.ts` | **New.** Platform Admin endpoints under `/admin/companies`: list, create (+ first Company Admin invitation), view, suspend/activate, invite another Company Admin. |
| `tenants.module.ts`, `index.ts` | Wiring. |

### 7.4 Other API files

| File | Change |
|---|---|
| `main.ts` | CORS now allows **only** `http://localhost:<port>` (or `CORS_ORIGINS` in production) and allows cookies (`credentials: true`). |
| `app.module.ts` | Registers `AuditModule` and `AuthModule`. |
| `health.controller.ts` | Marked `@Public()` so health checks need no login. |
| `cli/create-platform-admin.ts` | **New.** The command from section 4.3. |
| `package.json` | New scripts: `admin:create`, `test:db`. **No new dependencies.** |

---

## 8. How the existing JD module was connected

The goal was **"connect, do not rebuild."** The JD module already asked an abstraction called `TenantContext`
"which company is this?". Before, that answer was always "default". Now it is "the company in your session".
So the JD logic itself barely changed.

| What | Before | After |
|---|---|---|
| Which company | always `default` | the logged-in user's active company |
| Routes | open to anyone | each route has a permission (`jobs.read`, `jobs.write`, `jd.generate`, `jd.edit`, `jd.approve`) |
| `jobs.created_by_id` | — | the user who created the job |
| `jd_versions.created_by` | always empty | the user who generated or edited that version |
| Approver | name typed in the form | the logged-in user's name + id. A name sent by the browser is **ignored** |
| Audit | none | `job.created`, `jd.generated`, `jd.edited`, `job.approved` |

**Unchanged on purpose:** job form validation, AI generation through the worker and Gemini, the company snapshot on
each version, regeneration, HR edits saved as new versions, "stale version" protection between tabs, approval only
of the current version, and locking after approval. The company profile is still loaded automatically, so HR never
re-types company details; job-specific inputs (role, skills, experience, salary, location) are entered per job.

**Shared contract change** (`packages/contracts/src/schemas/job.ts`): the approve request no longer has
`approvedBy`; the approval response has a new `approvedById` (empty for old approvals). The API, dashboard and
tests were updated together.

---

## 9. Frontend (dashboard) changes

All under `apps/hr-dashboard/src/`.

### 9.1 New pages

| Page | Who | What it does |
|---|---|---|
| `/login` | everyone | Email + password. No sign-up link: accounts come from invitations. |
| `/invite/[token]` | invited person | Shows company and role; new users set name + password, existing users confirm their password. |
| `/team` | Company Admin | Member list, change role, disable/enable, revoke pending invitations, create invitation links. |
| `/admin` | Platform Admin | All companies with status, member and job counts; form to onboard a company. |
| `/admin/companies/[id]` | Platform Admin | Members and invitations of one company, suspend/activate, assign a Company Admin. |

### 9.2 New building blocks

| File | What it does |
|---|---|
| `components/session.tsx` | Loads "who am I" (`/auth/me`) once. Sends logged-out users to `/login`, sends Platform Admins without a company to `/admin`, shows friendly notices for suspended companies / no company / wrong role, and draws the header (role-based menu, company switcher, user name, **Log out**). |
| `components/invite-link.tsx` | Shows a new invitation link once, with a **Copy link** button. |
| `components/team-table.tsx` | Table of members and pending invitations (used by Team and Admin pages). |
| `lib/roles.ts` | Display names: "Company Admin", "HR Member". |
| `lib/safe-next.ts` | Allows only same-site "continue to" links after login (no open redirects). Shared with `/company`. |

### 9.3 Changed files

| File | Change |
|---|---|
| `lib/api.ts` | Every call sends the cookie (`credentials: 'include'`) and the `X-Requested-With: hireflow` header. On `401` it tells the session provider to go to login. New functions for login, logout, session, invitations, team and admin. Friendly messages for new error codes. Handles empty `204` answers. |
| `app/layout.tsx` | Wraps every page in `SessionProvider` (the header moved into it). |
| `components/approval-panel.tsx` | The **"Your name" field was removed.** Shows "You approve as *name*". |
| `app/jobs/[id]/job-review.tsx` | Approves without sending a name; passes the logged-in name to the panel. |
| `app/company/company-editor.tsx` | HR Members see the profile **read-only**; only Company Admins see the form. |
| `app/jobs/jobs-list.tsx`, `app/jobs/new/new-job.tsx` | If no profile exists: Company Admins get "Create company profile", HR Members get "Ask your Company Admin". |

> The dashboard hides things a role cannot use, but that is only for a nicer screen. **The API is what actually
> enforces every rule.**

The Platform Admin pages live in the HR dashboard under `/admin` because `apps/admin-console` is still an empty
placeholder. They can be moved there later without API changes.

---

## 10. Security features explained

| Feature | What it protects against | How |
|---|---|---|
| **Password hashing (scrypt)** | Stolen database revealing passwords | Only a slow, salted hash is stored |
| **Same-speed failed login** | Finding out which emails have accounts | A dummy hash is checked when the email is unknown; one generic error message |
| **Login rate limit** | Password guessing | 10 tries per email + IP and 100 per IP every 15 minutes, then `429` |
| **Server-side sessions** | Stolen or forged tokens | Random 256-bit token; the database stores only its SHA-256 hash; logout deletes it |
| **httpOnly cookie** | Malicious scripts stealing the login | JavaScript cannot read the cookie |
| **SameSite=Lax + CSRF header** | Another website making your browser act for you | Cross-site POSTs carry no cookie, and writes need a header other sites cannot add without permission |
| **Strict CORS** | Look-alike sites calling the API with your cookie | Exact origin allowlist |
| **Global guard, fail closed** | A new endpoint accidentally left open | Routes without an access label are refused |
| **Company from the session only** | Changing a company id in the URL/body/header | The API ignores those; the tests try it and confirm it does nothing |
| **Tenant filter in every query** | Reading another company's rows | `WHERE tenant_id = <session company>` in every repository method |
| **Composite foreign keys** | A bug writing a version into the wrong company | The database rejects it |
| **One-time, hashed invitations** | Reused or leaked links | Single use, expires in 7 days, old pending invites for the same email are revoked |
| **Password needed for existing accounts** | Taking over an account through an invite | Accepting for an existing email requires that account's password |
| **No public sign-up / Platform Admin via CLI only** | Granting yourself power | No API creates a Platform Admin; invites offer only company roles |
| **Last-admin rule** | A company locked out of its own team | Cannot disable or demote the last active Company Admin (checked in a strict "serializable" transaction) |
| **Approver from login** | Faking who approved | Name and id come from the session |
| **Audit log** | "Who did this?" | Important actions are recorded |

---

## 11. API reference

All paths start with `/api/v1`. Writes need the header `X-Requested-With: hireflow`.

### Login and session

| Method | Path | Who | Body | Answer |
|---|---|---|---|---|
| POST | `/auth/login` | anyone | `{ email, password }` | `Session` + sets cookie |
| POST | `/auth/logout` | anyone | — | `204`, clears cookie |
| GET | `/auth/me` | logged in | — | `Session` |
| POST | `/auth/company` | logged in | `{ companyId }` | `Session` (switched company) |
| GET | `/auth/invitations/:token` | anyone | — | `{ email, companyName, role, expiresAt, existingUser }` |
| POST | `/auth/invitations/accept` | anyone | `{ token, name?, password }` | `Session` + sets cookie |

`Session` looks like:

```json
{
  "user": { "id": "…", "email": "priya@acme.dev", "name": "Priya", "isPlatformAdmin": false },
  "company": { "id": "…", "name": "Acme", "slug": "acme", "status": "ACTIVE", "role": "HR_MEMBER" },
  "companies": [ { "id": "…", "name": "Acme", "slug": "acme", "status": "ACTIVE", "role": "HR_MEMBER" } ]
}
```

### Team (Company Admin)

| Method | Path | Body | Answer |
|---|---|---|---|
| GET | `/company/members` | — | `{ members[], invitations[] }` |
| POST | `/company/invitations` | `{ email, role }` | invitation + `inviteUrl` (shown once) |
| DELETE | `/company/invitations/:id` | — | `204` |
| PATCH | `/company/members/:userId` | `{ role?, status? }` | updated team |

### Companies (Platform Admin)

| Method | Path | Body | Answer |
|---|---|---|---|
| GET | `/admin/companies` | — | list with member and job counts |
| POST | `/admin/companies` | `{ name, slug, adminEmail }` | `{ company, invitation }` |
| GET | `/admin/companies/:id` | — | company + members + invitations |
| PATCH | `/admin/companies/:id` | `{ status: "ACTIVE" \| "SUSPENDED" }` | updated company |
| POST | `/admin/companies/:id/admin-invitations` | `{ email }` | invitation + `inviteUrl` |

### Company profile and jobs (unchanged paths, now protected)

| Method | Path | Permission |
|---|---|---|
| GET | `/company-profile` | `company.read` (both roles) |
| PUT | `/company-profile` | `company.update` (Company Admin) |
| POST / GET | `/jobs` | `jobs.write` / `jobs.read` |
| GET / PATCH | `/jobs/:id` | `jobs.read` / `jobs.write` |
| POST | `/jobs/:id/jd/generate` | `jd.generate` |
| PATCH | `/jobs/:id/jd` | `jd.edit` |
| POST | `/jobs/:id/approve` | `jd.approve` — body `{ jdVersionId, companyDetailsVerified: true, note? }` |

---

## 12. Error codes

Every error has the same shape: `{ statusCode, code, message }`.

| Status | Code | Meaning |
|---|---|---|
| 400 | `INVALID_LOGIN`, `INVALID_INVITATION`, `INVALID_INVITATION_ACCEPT`, `INVALID_MEMBER_UPDATE`, `INVALID_COMPANY`, `NAME_REQUIRED` | Bad input (with `issues[]` where useful) |
| 401 | `UNAUTHENTICATED` | Not logged in or session ended |
| 401 | `INVALID_CREDENTIALS` | Wrong email or password (also for a wrong password when accepting an invite for an existing account) |
| 403 | `CSRF_CHECK_FAILED` | Write request without the `X-Requested-With` header |
| 403 | `FORBIDDEN` | Your role may not do this, or the route has no access label |
| 403 | `NO_COMPANY` | No active membership in a company (e.g. disabled, or a Platform Admin calling company routes) |
| 403 | `COMPANY_SUSPENDED` | Your company is suspended |
| 403 | `NOT_A_MEMBER` | Tried to switch to a company you do not belong to |
| 404 | `INVITATION_INVALID` | Invitation link unknown, expired, revoked or already used |
| 404 | `MEMBER_NOT_FOUND`, `INVITATION_NOT_FOUND`, `COMPANY_NOT_FOUND`, `JOB_NOT_FOUND` | Not found **in your company** |
| 409 | `ALREADY_MEMBER` | That email is already an active member |
| 409 | `LAST_COMPANY_ADMIN` | Would leave the company with no active Company Admin |
| 409 | `SLUG_TAKEN` | Another company uses this slug |
| 429 | `TOO_MANY_ATTEMPTS` | Too many logins or invite attempts; wait 15 minutes |

The JD error codes (`STALE_JD_VERSION`, `JOB_APPROVED`, `LLM_*`, …) are unchanged; see [JD_WORKFLOW.md](JD_WORKFLOW.md).

---

## 13. Testing

### 13.1 What was run and the results

| Suite | Command | Result |
|---|---|---|
| Shared contracts | `pnpm --filter @hireflow/contracts test` | 20 / 20 passed (was 14) |
| API JD regression (no database) | `pnpm --filter api test` | 24 / 24 passed |
| **Two-company test (real PostgreSQL)** | see 13.2 | **23 / 23 passed** |
| Dashboard | `pnpm --filter hr-dashboard test` | 29 / 29 passed (was 24) |
| AI worker (Python) | `cd workers/ai-worker && uv run pytest` | 65 passed, 1 skipped (live-LLM test is opt-in) |
| Prisma schema | `npx prisma validate` | valid |
| Type check, lint, build | `turbo run build`, `typecheck`, `lint` | all pass |

### 13.2 The two-company test

File: `apps/api/tests/multi-company.e2e.test.ts`. It starts the **real** API (real guard, real Prisma queries)
against a separate test database. Only the Python AI worker is replaced by a small local stub.

```bash
DATABASE_URL=postgresql://hire:hire@localhost:5433/hireflow_test pnpm --filter api db:deploy   # once
TEST_DATABASE_URL=postgresql://hire:hire@localhost:5433/hireflow_test pnpm --filter api test:db
```

It refuses to run unless the database name contains "test", and it never deletes rows (each run uses unique names).

**What it does:** the Platform Admin creates **Company A** and **Company B**. Each Company Admin accepts their
invite, saves a profile and invites an HR Member. Each HR Member creates a job, generates a JD, edits it,
regenerates (3 versions) and approves. Then it tries to break the rules:

| Checked | Result expected and confirmed |
|---|---|
| Anonymous access; wrong password vs unknown email give identical answers | `401`, same body |
| 7 bad logins in a row | 6th+ answer `429` |
| Cookie is httpOnly + SameSite; database holds only the hash | yes |
| Logout, then replay the old cookie | `401` |
| Write without CSRF header | `403 CSRF_CHECK_FAILED` |
| Public sign-up endpoint | does not exist (`404`) |
| Each company's AI request carried **its own** company profile | yes |
| Versions/approval record the real user; a fake `approvedBy` is ignored | yes |
| A lists jobs | only A's jobs |
| A reads / edits / regenerates / edits JD / approves B's job | all `404`, B unchanged |
| Company ids in headers, query, body | ignored; job saved in A |
| A switches session into B | `403 NOT_A_MEMBER` |
| A's admin disables B's member / revokes B's invitation | `404`, B unchanged |
| Database insert of a version with A's company and B's job | rejected by the database |
| HR Member edits profile / manages team / uses admin pages | `403` |
| Company Admin uses platform pages or invites a "PLATFORM_ADMIN" | `403` / `400` |
| Platform Admin reads jobs | `403 NO_COMPANY` |
| Demote the last Company Admin | `409 LAST_COMPANY_ADMIN` |
| Disable then re-enable an HR Member | access lost, then back, immediately |
| Suspend B | every B request `403 COMPANY_SUSPENDED`; A unaffected; re-activate works |
| Reuse an invitation | `404 INVITATION_INVALID` |
| Invite an existing user, accept with a wrong password | `401`; with the right password → joins second company |

**Proof the test is meaningful:** the tenant filter was temporarily removed from the job lookup; the cross-company
test **failed**; the filter was put back.

### 13.3 Not verified

- **Live Gemini generation after these changes.** The real stack was run (login, onboarding, invitations, profile,
  job creation all worked), but Google answered `503 – model is currently experiencing high demand` twice. The API
  passed the error on safely and saved nothing, as designed. Re-run the flow when Gemini is available.
- Under `turbo run test`, the two-company test is **skipped** because turbo does not pass `TEST_DATABASE_URL`
  through. Use `pnpm --filter api test:db`.

---

## 14. Known limits and next steps

| Not done yet | Notes |
|---|---|
| PostgreSQL Row Level Security | Isolation is enforced by the guard, tenant-filtered queries and composite foreign keys. RLS would be an extra safety net (M33). |
| Email delivery of invitations | The inviter copies the link. Needs the notifications module (M24). |
| Password reset, 2FA, Google SSO | Planned in the spec for M28. |
| Rate limits shared between servers | Kept in memory per API process; move to Redis when running several API copies. |
| Audit log viewer | Rows are written; no screen or API to read them yet. |
| Interviewer and candidate roles | Later modules. |
| `turbo run build typecheck` together can fail `hr-dashboard#typecheck` | Pre-existing race: `next build` regenerates `.next/types` while `tsc` reads it. Each task passes on its own. |

---

## 15. Troubleshooting

| Symptom | Likely cause and fix |
|---|---|
| Always sent back to `/login` | The API is not running on `NEXT_PUBLIC_API_URL`, or the dashboard origin is not allowed by CORS. In production set `CORS_ORIGINS`. |
| Login works but nothing loads on HTTPS | Set `COOKIE_SECURE=true` and serve both apps over HTTPS. |
| `403 CSRF_CHECK_FAILED` from your own script | Add the header `X-Requested-With: hireflow` to POST/PUT/PATCH/DELETE. |
| `403 NO_COMPANY` | The user's membership is disabled or they were never added, or a Platform Admin is calling company routes. |
| `403 COMPANY_SUSPENDED` | A Platform Admin must activate the company. |
| `429 TOO_MANY_ATTEMPTS` | Wait 15 minutes (or restart the API in development). |
| Invitation link says "invalid" | It was used, revoked, or more than 7 days old. Create a new one. |
| `admin:create` says the password is too short | Use 10–128 characters. |
| `prisma migrate dev` says "non-interactive" | Use `pnpm --filter api db:deploy` to apply migrations. |

---

## 16. Every file added or changed

```text
apps/api/prisma/schema/auth.prisma                          NEW  users, sessions, memberships, invitations
apps/api/prisma/schema/audit.prisma                         NEW  audit_logs
apps/api/prisma/schema/tenants.prisma                            tenant status (+ relations)
apps/api/prisma/schema/jobs.prisma                               created_by_id, approved_by_id, (job_id, tenant_id) links
apps/api/prisma/schema/migrations/20261013090000_multi_company_auth/migration.sql   NEW
apps/api/src/modules/auth/*                                 NEW  login, sessions, guard, permissions, team
apps/api/src/modules/audit/{audit.service,index}.ts         NEW  audit log
apps/api/src/modules/tenants/tenant-context.ts                   DefaultTenantContext → RequestTenantContext
apps/api/src/modules/tenants/{companies.controller,companies.service}.ts  NEW  Platform Admin
apps/api/src/modules/tenants/{company-profile.controller,company-profile.service,tenants.module,index}.ts
apps/api/src/modules/jobs/{jobs.controller,jobs.service,jobs.repository}.ts   permissions, actor, audit
apps/api/src/cli/create-platform-admin.ts                   NEW  CLI
apps/api/src/{main,app.module,health.controller}.ts              CORS, modules, public health
apps/api/package.json                                            admin:create, test:db scripts
apps/api/tests/multi-company.e2e.test.ts                    NEW  two-company test
apps/api/tests/{in-memory,jobs.test}.ts                          actor + audit fakes, approver from session
apps/api/src/modules/{auth,tenants,jobs,audit}/README.md         updated
packages/contracts/src/schemas/auth.ts                      NEW  roles, session, invitations, team, admin
packages/contracts/src/schemas/job.ts                            approve request without approvedBy; approvedById
packages/contracts/src/index.ts                                  exports auth
packages/contracts/tests/{auth.test,company.test}.ts             new + updated
apps/hr-dashboard/src/app/login/*                           NEW
apps/hr-dashboard/src/app/invite/[token]/*                  NEW
apps/hr-dashboard/src/app/team/*                            NEW
apps/hr-dashboard/src/app/admin/** (page, companies-admin, companies/[id]/*)   NEW
apps/hr-dashboard/src/components/{session,invite-link,team-table}.tsx          NEW
apps/hr-dashboard/src/lib/{roles,safe-next}.ts              NEW
apps/hr-dashboard/src/lib/api.ts                                 cookies, CSRF header, new endpoints
apps/hr-dashboard/src/app/layout.tsx                             SessionProvider
apps/hr-dashboard/src/components/approval-panel.tsx              no typed name
apps/hr-dashboard/src/app/jobs/[id]/job-review.tsx               approver from session
apps/hr-dashboard/src/app/company/{page,company-editor}.tsx      read-only for HR, shared safeNext
apps/hr-dashboard/src/app/jobs/{jobs-list,new/new-job}.tsx       role-aware "create profile" prompt
apps/hr-dashboard/tests/api.test.ts                              session client tests
apps/hr-dashboard/README.md                                      new pages
.env.example                                                     DEFAULT_TENANT_SLUG removed; auth settings added
docs/{FOLDER_STRUCTURE,JD_WORKFLOW}.md                           status + links updated
docs/MULTI_COMPANY_AUTH.md                                  NEW  this document
```

Not changed: the AI worker (`workers/ai-worker`), the prompt files, and the JD business rules.

---

## 17. Glossary

| Term | Meaning |
|---|---|
| **CORS** | Browser rule deciding which websites may call the API. |
| **CSRF** | An attack where another website tricks your browser into sending a request while you are logged in. |
| **Hash** | A one-way scramble. You can check a value against it, but cannot turn it back into the original. |
| **httpOnly cookie** | A cookie the browser sends automatically but page JavaScript cannot read. |
| **Guard** | NestJS code that runs before an endpoint and allows or blocks the request. |
| **Migration** | A SQL file that changes the database structure, applied once, in order. |
| **Composite foreign key** | A database rule that links two columns together (job + company) so mismatches are impossible. |
| **Rate limiting** | Allowing only a certain number of attempts in a time window. |
| **Serializable transaction** | The strictest database mode: concurrent changes behave as if they ran one after another. |
| **Tenant** | A customer company in a multi-company system. |

# ari-analytics — Handover Assessment

**Date**: 2026-05-02
**Assessor**: Yehia Gamal
**Status**: handover

## Origin

- **Where it came from**: Internal build — custom analytics dashboard for Arab Reform Initiative
- **Original owner**: Yehia Gamal
- **Repo location**: https://github.com/yehiagamalx/ari-analytics (local: `/Users/ye/Projects/analytics`)
- **First commit date**: 2026-02-20
- **Last commit date**: 2026-04-21

## Current State

### Tech stack

| Layer | Details |
|-------|---------|
| Language | JavaScript (JSX) + Python 3 |
| Frontend framework | React 19 / Vite 7 |
| Frontend state | Zustand (global), React Router v7 (routing) |
| Frontend charts | Recharts, D3-geo, TopoJSON |
| Admin API | FastAPI + Uvicorn |
| Auth | Google OAuth2 via oauth2-proxy container |
| Data pipeline | Python scripts: GA4, Google Search Console, Zoom, Mailjet, WordPress REST API |
| Storage | SQLite (pipeline) + `data.json` (flat JSON served to dashboard) |
| Reverse proxy | Nginx (SSL termination + auth gate) |
| Containerisation | Docker Compose (4 services: dashboard, backend, oauth2-proxy, nginx) |
| CI | None |

### Build status

| Step | Status |
|------|--------|
| `npm install` (dashboard) | not attempted |
| `npm run build` (dashboard) | not attempted |
| `npm run lint` (dashboard) | not attempted |
| `uvicorn main:app` (backend) | not attempted |
| Pipeline scripts | not attempted |

> README includes a pre-built `data.json` so the dashboard can be viewed with just `cd dashboard && npm install && npm run dev`.

### Test coverage

| Component | Tests | Coverage |
|-----------|-------|----------|
| React frontend | None — no Vitest/Jest in `package.json` | 0% |
| FastAPI backend | None | 0% |
| Python pipeline | `pipeline/scripts/test_ga4.py` (one script) | Minimal |

### Repo activity

- Commits in last 90 days: **48** (active)
- Open issues: 0
- Open PRs: 0
- Top contributor: Yehia Gamal (sole contributor)

## Quality Risks

### Security

- **CORS `allow_origins=["*"]`** in `backend/main.py` — the comment notes nginx handles CORS in production, but local dev exposes any origin. Should be tightened to explicit origins or an env-var allowlist.
- **Google service account JSON** (`pipeline/credentials/ga4-service-account.json`) exists locally. It is gitignored (confirmed via `.gitignore`), but production secret rotation and storage strategy are undocumented.
- **`dashboard/.env.local`** is present (gitignored via `.env.*` rule) — likely holds Vite dev env vars; no risk to the repo, but contents should be audited for any hardcoded credentials.
- **Admin panel** is protected only by email allowlist in `backend/config.json` — no rate limiting, no brute-force protection on the `/api/admin/*` endpoints beyond OAuth2 proxy.
- No dependency vulnerability scan has been run.

### Dependencies

- Frontend runs **React 19** and **Vite 7** — both are very recent releases; no stability issues expected, but worth monitoring.
- No `package-lock.json` committed for the dashboard (or `requirements.txt` lockfile for backend/pipeline) — builds are not reproducible.
- No automated dependency audit (no `npm audit` in CI, no `pip-audit`).

### Technical debt

- **No TypeScript** — frontend is plain JSX with no type checking. Large component tree (20+ pages, 15+ shared components) carries growing refactor risk.
- **No frontend tests** — zero test coverage on a 20+ page React app.
- **No backend unit tests** — FastAPI routes are untested.
- **Flat `data.json` architecture** — the dashboard reads a single large JSON blob built by the pipeline; no pagination, no incremental updates. May become a performance issue as data grows.
- **`design_handoff/` directory** present in repo — design artefacts stored alongside code rather than in a design system (Figma, etc.).

### Operational

- **No CI pipeline** — no `.github/workflows/`; PRs receive no automated checks (lint, build, tests).
- **No error monitoring** — no Sentry, Datadog, or equivalent in docker-compose or code.
- **No health endpoint** — FastAPI backend has no `/health` or `/ready` route (not confirmed, but not visible in the surface read).
- **Pipeline is manual / cron-triggered** — `refresh.sh` runs pipeline scripts on the host; no scheduling mechanism is visible in docker-compose.

## Integration Plan

### Roles that apply

| Role | Why |
|------|-----|
| `tech-lead` | Technical design, PR approval gate |
| `frontend-engineer` | React/Vite UI (20+ pages, 15+ shared components) |
| `backend-engineer` | FastAPI admin API + Python data pipeline |
| `sre` | Docker Compose, Nginx, production deployment on analytics.arab-reform.net |
| `security-auditor` | Google OAuth2 auth, service account credentials, CORS, admin panel |

### Workflows that kick in

- [ ] PR workflow (`.claude/rules/pr-workflow.md`) — every change goes through a PR
- [ ] AgDR for technical decisions (especially any data architecture change)
- [ ] Code Reviewer agent (Rex) on every PR
- [ ] Security Reviewer agent on first pass and any PR touching auth/pipeline/credentials
- [ ] `/audit-deps` on adoption and monthly thereafter

### Hooks to enable

- [ ] `block-git-add-all`
- [ ] `block-main-push`
- [ ] `validate-branch-name` (ticket_prefix: GH)
- [ ] `validate-pr-create`
- [ ] `pre-push-gate`
- [ ] `check-secrets`

### CI templates to copy in

- [ ] `golden-paths/pipelines/ci.yml`
- [ ] `golden-paths/pipelines/security.yml`
- [ ] `golden-paths/pipelines/pr-title-check.yml`

### Registry entry

```yaml
- name: ari-analytics
  repo: yehiagamalx/ari-analytics
  workspace: workspace/ari-analytics
  docs: projects/ari-analytics
  status: handover
  tier: P1
  roles:
    - tech-lead
    - frontend-engineer
    - backend-engineer
    - sre
    - security-auditor
  tags:
    - ari
    - analytics
    - python
    - react
  ticket_prefix: GH
```

## Next Steps

1. **Copy in `golden-paths/pipelines/ci.yml`** — no CI exists; lint + build checks must run on every PR before the first feature lands.
2. **Add Vitest to the dashboard** and write smoke tests for at least the top 3 pages (`SiteOverview`, `Publications`, `Newsletter`) — these are the highest-traffic pages and currently have 0% test coverage.
3. **Tighten CORS in `backend/main.py`** — replace `allow_origins=["*"]` with an env-var allowlist (`ARI_CORS_ORIGINS`); the current setting is a risk in local dev even if nginx handles it in prod.
4. **Run `/audit-deps ari-analytics`** — no vulnerability scan has been performed; dashboard uses 10+ npm packages and backend/pipeline use Google SDK and FastAPI.
5. **Document the service account rotation process** — `pipeline/credentials/ga4-service-account.json` is gitignored but there's no documented procedure for rotating the key or storing it in production secrets management.
6. **Code-review the most recent PR on this repo as Rex** to calibrate review standards before the first new feature lands.

## Post-Handover Checklist

- [ ] Review this assessment with the previous owner
- [ ] Add CI pipeline (Step 1 above) — close before the first feature PR
- [ ] Add frontend test baseline (Step 2 above) — scheduled in first 2 weeks
- [ ] Fix CORS configuration (Step 3 above) — close before the first feature PR
- [ ] Run `/audit-deps ari-analytics` (Step 4 above) — close before the first feature PR
- [ ] Document service account rotation (Step 5 above) — scheduled in first 2 weeks
- [ ] Add `ari-analytics` to the weekly `/stakeholder-update` rollup
- [ ] Onboard the roles listed above into the team's on-call / review rotation
- [ ] Clone live working copy: `git clone https://github.com/yehiagamalx/ari-analytics workspace/ari-analytics`

## Open Questions

- What is the current production deployment process? (`refresh.sh` on the host — is this manual or scheduled?)
- Is the Google service account JSON stored as a server secret in production, or is the local file manually copied?
- Are there any SLOs / uptime targets defined for analytics.arab-reform.net?
- Is the `design_handoff/` directory actively used? Should it move to Figma?
- Are there any external consumers of the FastAPI backend besides the dashboard UI?

<!-- Source: ApexYard · templates/prd.md · github.com/me2resh/apexyard · MIT -->

# PRD: HRD Survey Dashboard — Dynamic Chart Engine

**Status**: Approved (source: operator-authored technical brief)
**Author**: Mariam (Product Manager) — on behalf of the site maintainer
**Created**: 2026-09-19
**Last Updated**: 2026-09-19
**Repo**: yehiagamalx/hrd-survey-dashboard

---

## Overview

### Problem Statement

A WordPress plugin renders aggregated statistics from LimeSurvey testimony
data documenting human-rights violations, replacing a Metabase dashboard.
Today the plugin is 10 separate PHP files, one per chart, each with a
hardcoded query — adding or changing a chart means editing PHP and
redeploying via a slow, manual `docker cp` to the server. The project holds
sensitive testimony data, so every change carries real risk: a hardcoded
mistake (e.g. exposing the wrong question) is a code change away, not a
config change with guardrails around it.

### Target User

**Primary**: the site maintainer/administrator, who adds and edits chart
definitions from wp-admin without touching PHP or redeploying.
**Secondary**: site visitors, who view the aggregated, anonymised statistics
publicly (this is an intentional transparency feature, not an oversight —
see Security Considerations in the technical design).

### Goals

1. Replace 10 hardcoded chart files with **one generic engine** driven by
   stored chart definitions — adding or editing a chart requires zero code
   changes and zero redeploys.
2. Give the maintainer a **capability-gated admin UI** to add, edit, delete,
   and reorder chart definitions.
3. Serve chart data through **one dynamic, intentionally-public REST
   endpoint** that accepts only a pre-stored definition ID — never a qid,
   column name, or query shape from the request.
4. **Zero regression** against the original Metabase-verified numbers for
   the four charts with published reference counts (see Success Metrics).
5. **Absolute, code-enforced exclusion** of qid=745 ("do you want
   protection?") and every free-text field from all charts and endpoints —
   enforced in the validation layer itself, not left to reviewer discipline.
6. Replace manual `docker cp` deploys with a Git-based workflow (bind mount
   + `git pull` + a deploy script) — scoped to producing the *artifacts* for
   this; actually wiring them into the live server is explicitly the
   maintainer's own step, not part of this build (see Non-Goals).

### Non-Goals (Out of Scope)

- Exporting raw survey responses or any per-respondent drill-down — every
  endpoint returns aggregated counts only.
- Editing LimeSurvey's own survey structure (questions, answer options).
- Replicating the *old* per-file REST routes (`v1/chart-1-info-source`,
  etc.) — those live on the server and aren't touched; this PRD covers only
  the new generic `v2` engine.
- Connecting to, or making any changes on, the real production server —
  explicitly out of scope for this build by the maintainer's direct
  instruction. Verifying against the *real* LimeSurvey database, and
  applying the bind-mount / deploy-script changes to the live
  `docker-compose.yml`, remain the maintainer's own follow-up steps.
- A distributed-plugin auto-update mechanism (PUC/WordPress.org-style) —
  this is a single private install; deploy is git-pull based, not
  release-based.

### Success Metrics

| Metric | Target | How Measured |
|--------|--------|--------------|
| Chart-add turnaround | New chart live with zero code edits | Manual test: add a definition via admin UI, confirm it renders |
| Regression accuracy | Chart 4/5/6/8 counts match the original Metabase report exactly | PHPUnit + REST assertions against the local fixture DB (see Testing Strategy in the technical design) |
| Forbidden-data exposure | Zero — qid=745 and free-text fields never reachable via any admin path or endpoint | Validation-layer unit tests attempting to define a chart on qid=745 / a free-text qid, expecting rejection |

---

## User Stories

### US-1: Add a chart without touching code
> As the site maintainer, I want to add a new chart by filling in a form in
> wp-admin, so that I never have to write PHP or redeploy for a new chart.

**Acceptance Criteria**:

- [ ] Admin page lets me pick a `chart_type`, a `source_type` (from the
      fixed enum), qid/parent_qid, and AR/EN titles.
- [ ] Saving the form makes the chart appear on the public shortcode
      immediately (subject to `enabled: true`), with no deploy step.
- [ ] The public REST endpoint immediately serves data for the new
      definition ID.

---

### US-2: Reorder and toggle existing charts
> As the site maintainer, I want to reorder and enable/disable charts, so
> that I control what visitors see without deleting definitions.

**Acceptance Criteria**:

- [ ] Definitions have an `order` field editable from the admin UI.
- [ ] The shortcode renders only `enabled: true` definitions, sorted by
      `order`.
- [ ] Disabling a chart removes it from the shortcode but the definition
      (and its REST route) is preserved for later re-enabling.

---

### US-3: Visitor sees accurate, matching charts
> As a site visitor, I want the published charts to show the same numbers
> the organisation has already verified, so that the dashboard is
> trustworthy.

**Acceptance Criteria**:

- [ ] Chart 4 (who committed the violation): government=6, militia=5,
      civil=3, official=1.
- [ ] Chart 5 (main violation types): physical=7, sexual=5, threats=4,
      legal=6, financial=0, defamation=8.
- [ ] Chart 6 (affected rights sectors): 12, 10, 3, 3, 2 in descending
      order.
- [ ] Chart 8 (direct physical violence detail): direct=4, attempt=2,
      killing=1, abduction=1 (total=8).
- [ ] All of the above verified against a local, schema-faithful fixture
      (no server access in this build — see Technical Constraints).

---

### US-4: Forbidden data can never be exposed, even by mistake
> As the organisation responsible for this data, I want it to be
> structurally impossible to define a chart on the protection question or
> any free-text field, so that a config mistake can't leak sensitive
> testimony.

**Acceptance Criteria**:

- [ ] Attempting to save a chart definition referencing qid=745 is
      rejected by the validation layer, not merely discouraged in docs.
- [ ] Attempting to save a chart definition referencing a known free-text
      qid is rejected the same way.
- [ ] The "Import JSON" admin feature runs the same validation — pasting a
      JSON blob that includes a forbidden qid is rejected, never silently
      imported.
- [ ] The public REST endpoint's only client-controlled input is a
      pre-stored definition ID and `lang`; no request parameter can ever
      resolve to an arbitrary qid or column.

---

### Edge Cases

| Scenario | Expected Behavior |
|----------|-------------------|
| A multi-checkbox question gains new subquestions in LimeSurvey after the chart was defined | Subquestions are discovered dynamically from `lime_questions.parent_qid` at request time — no hardcoded subcode list, so new subquestions appear automatically |
| Admin pastes malformed or partially-invalid JSON into "Import JSON" | Every field validated (qid as integer, chart/source type against the enum) before anything is saved; invalid JSON is rejected with a clear error, nothing partial is persisted |
| `lang` query param is missing, empty, or an unexpected value | Falls back to a fixed default (`ar`) rather than erroring or reflecting the raw input back |
| A chart definition references a qid that no longer exists in LimeSurvey | Endpoint returns an empty/zeroed dataset for that chart rather than a fatal error; logged for the admin, not surfaced to the public visitor |
| `qid=717`/`718` country/nationality values include `OT` ("other") | Excluded from chart 717 (incident country) per spec; included as-is for chart 718 (nationality) per spec |

---

## Requirements

### Functional Requirements

| ID | Requirement | Priority | Notes |
|----|-------------|----------|-------|
| FR-1 | Generic chart-definition schema (`id`, `chart_type`, `source_type`, `categories`/`qid`/`parent_qid`, AR/EN titles, `enabled`, `order`) | Must | Fixed `source_type` enum: `single_choice`, `multi_checkbox`, `standalone_yn_set`, `or_aggregate_category_set` |
| FR-2 | Generic execution engine dispatching each definition to the matching breakdown function | Must | Reuses the four `hrd_dash_*_breakdown()` functions named in the brief |
| FR-3 | Capability-gated admin UI: add/edit/delete/reorder + JSON import (validated) | Must | Viewing the shortcode stays public; editing definitions requires an admin capability |
| FR-4 | One dynamic public REST endpoint, `GET /hrd-dashboard/v2/chart/{id}?lang=` | Must | Definition-ID-only input — see US-4 |
| FR-5 | Shortcode renders all `enabled` definitions sorted by `order`, each backed by the v2 endpoint | Must | |
| FR-6 | Regression validation against the four charts with published reference numbers | Must | Local fixture only — see Technical Constraints |
| FR-7 | Deploy-workflow artifacts (bind-mount compose snippet, `deploy.sh`) | Should | Documentation/scripts only; not applied to the live server in this build |

### Non-Functional Requirements

| Category | Requirement | Target |
|----------|-------------|--------|
| Security | LimeSurvey DB access via a dedicated `SELECT`-only user, on a fixed table allow-list | Read-only user, 5 named tables, mysqli + prepared statements |
| Security | Dynamic column names validated before use in any SQL string | Strict regex + `information_schema` cross-check |
| Security | LimeSurvey DB credentials | `wp-config.php` via `putenv()`/`getenv()`, never hardcoded, never in `wp_options` |
| Security | qid=745 + all free-text fields | Never reachable via any admin path or public endpoint, enforced at the validation layer |
| Resource footprint | Runs alongside other Docker Compose projects on a 3.8GB-RAM shared host | No new heavyweight services; `wp_options`-backed storage, no extra DB server for definitions |
| Compatibility | Matches the WordPress/PHP versions already running (`wordpress:latest`, PHP 8.2.27) and the Bricks Builder + WPML setup | Shortcode enqueues its own assets from inside the render callback (not via `has_shortcode()` on `post_content`, which Bricks doesn't support) |

---

## Design

### User Flow — Admin adding a chart

```
[wp-admin → HRD Dashboard menu]
    |
    v
[Click "Add Chart"]
    |
    v
[Fill form: chart_type, source_type, qid(s), AR/EN titles, order]
    |
    v
[Submit] --> [Validation: enum check, integer bounds, forbidden-qid check]
    |                                   |
    v                                   v
[Saved to wp_options]           [Rejected with a specific error, nothing persisted]
    |
    v
[Chart appears on the public shortcode + its own REST route, immediately]
```

### Wireframes / Mockups

None commissioned for this MVP — the admin UI is a standard WordPress
settings-table CRUD form (see the technical design's Admin UI section for
the concrete layout, modelled on this portfolio's existing
`wp-taxonomy-manager` and `wp-backup-sync` admin patterns).

---

## Technical Notes

### Dependencies

| Dependency | Type | Status | Owner |
|------------|------|--------|-------|
| `dashboard_readonly` MySQL user on the LimeSurvey DB (5-table allow-list) | External (server-side) | Assumed already provisioned per brief §3.1 — not created by this build | Maintainer |
| LimeSurvey DB credentials in `wp-config.php` (`HRD_LIME_DB_*`) | External (server-side) | Not set by this build — the plugin reads them via `getenv()` but never provisions them | Maintainer |
| Chart.js (frontend rendering) | External library | To be vendored/enqueued in the plugin's `assets/` | This build |

### Technical Constraints

- The `wordpress:latest` Docker image lacks the `pdo_mysql` extension —
  the external LimeSurvey DB connection must use `mysqli`, not PDO or
  `$wpdb` (which targets WordPress's own database).
- **This build has no access to the real server or the real LimeSurvey
  database.** All development and regression validation happens against a
  local Docker MySQL fixture engineered to match the documented schema and
  reference numbers. A final regression pass against the real production
  data, and the actual server-side deploy, are the maintainer's own
  follow-up steps once they choose to connect the two.

---

## Launch Plan

### Rollout Strategy

- [ ] Not applicable to this build directly — the plugin is delivered as a
      reviewed, merged, tagged private GitHub repo. The maintainer applies
      the bind-mount + `git pull` deploy (brief §5) to the real server on
      their own schedule, after independently re-verifying the regression
      numbers against production data.

---

## Open Questions

| Question | Owner | Status | Resolution |
|----------|-------|--------|------------|
| Default `lang` when the REST query param is absent | Tech Lead | Resolved | Default to `ar`, matching the org's primary-language convention seen elsewhere in this portfolio (WPML `ar`/`en` sites) — confirmable/overridable by the maintainer at any time, not a hard architectural commitment |
| Exact wp-admin capability name for chart-definition editing (`manage_options` vs. a custom `hrd_manage_dashboard` capability) | Tech Lead | Resolved | Custom `hrd_manage_dashboard` capability, mapped to `administrator` by default — matches brief §4.3's phrasing ("gated by the `hrd_view_dashboard` capability, or a higher admin capability specifically for editing") more precisely than reusing the generic `manage_options` |

---

## Timeline

| Milestone | Target Date | Status |
|-----------|-------------|--------|
| PRD Approved | 2026-09-19 | Approved |
| Technical Design + AgDRs | 2026-09-19 | In progress |
| Dev Complete (all 7 tickets) | TBD | Not started |
| Regression validated (local fixture) | TBD | Not started |
| Launch (maintainer connects to real server) | TBD — maintainer's own step | Not started |

---

## Approvals

| Role | Name | Date | Status |
|------|------|------|--------|
| Product Manager | Mariam (on behalf of the maintainer) | 2026-09-19 | Approved |
| Tech Lead | Hisham | 2026-09-19 | Pending (technical design next) |

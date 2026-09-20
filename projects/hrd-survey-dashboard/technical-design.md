<!-- Source: ApexYard · templates/technical-design.md · github.com/me2resh/apexyard · MIT -->

# Technical Design: HRD Survey Dashboard — Dynamic Chart Engine

**Status**: In Review
**Author**: Hisham (Tech Lead)
**Date**: 2026-09-19
**PRD**: [prd.md](prd.md)

---

## Overview

### Summary

Replace 10 hardcoded per-chart PHP files with a data-driven engine: chart
definitions (chart type, source type, qid(s), titles, order) are stored in
`wp_options` and executed through four existing, reusable breakdown
functions. A capability-gated admin UI manages definitions; one public,
definition-ID-only REST endpoint serves the data; a shortcode renders every
enabled definition. Built and regression-tested entirely locally — **no
connection to the real server or the real LimeSurvey database** is made in
this build (explicit operator instruction).

### Goals

- Zero-code-change chart authoring via an admin UI.
- One generic engine on top of four already-specified breakdown functions.
- A public REST surface whose only client-controlled input is a pre-stored
  definition ID + `lang` — never a qid or column.
- Code-enforced (not convention-enforced) exclusion of qid=745 and all
  free-text fields.

### Non-Goals

- Replicating the old `v1` per-file routes.
- Any change to LimeSurvey's own schema or survey structure.
- Applying the deploy artifacts to the real server, or a final regression
  pass against real production data — both remain the maintainer's own
  step (see PRD § Non-Goals).

---

## Domain Model

### Entities

```
ChartDefinition
├── id: string               (e.g. "chart-5", stable, used as the REST path + storage key)
├── chart_type: enum         (bar | pie | line | doughnut | grouped_bar)
├── source_type: enum        (single_choice | multi_checkbox | standalone_yn_set | or_aggregate_category_set)
├── qid: int|null            (single_choice; also used as parent_qid source for multi_checkbox)
├── parent_qid: int|null     (multi_checkbox — alias of qid, kept distinct in the schema for clarity)
├── standalone_qids: int[]|null   (standalone_yn_set — 1+ independent Y/N qids)
├── categories: Category[]|null   (or_aggregate_category_set only)
├── title_ar: string
├── title_en: string
├── enabled: bool
├── order: int
└── Methods:
    ├── validate(QuestionTypeResolver): ValidationResult   (enum membership, integer bounds, forbidden-qid + type-allow-list rejection)
    └── to_engine_request(): EngineRequest
```

`validate()` takes the `QuestionTypeResolver` port as a parameter rather than
reaching for the DB connection itself — this keeps the domain method pure
and unit-testable with a fake resolver (per
`handbooks/architecture/clean-architecture-layers.md`), even though the
*behaviour* it implements does require a metadata read (see Data Flow,
below — this is a correction from an earlier draft that claimed the admin
write path made no LimeSurvey DB access at all, which stopped being true
the moment type-derived validation was introduced).

### Value Objects

| Value Object | Fields | Purpose |
|--------------|--------|---------|
| `Category` | `parent_qid: int`, `title_ar: string`, `title_en: string` | One OR-aggregated category within an `or_aggregate_category_set` definition (brief §4 example: physical/sexual/threats/... under chart 5) |
| `ColumnRef` | `sid: int`, `gid: int`, `qid: int`, `subcode: string\|null` | A resolved, regex-validated LimeSurvey column name — never constructed from raw request input |
| `ForbiddenQidList` | `static_qids: int[]` (contains 745, checked first, never removable) | The narrow, hard-coded deny-list for questions excluded by policy rather than by type — checked *before* the type-allow-list, so it can never be bypassed by a type reclassification |
| `QuestionTypeResolver` (port) | `resolve(qid): QuestionType\|null` | Abstraction over the "what type is this qid" lookup — implemented against `lime_questions.type` in production, fakeable in unit tests. Returning `null` (lookup failure) is treated as reject, not allow — see Security Considerations |
| `EngineRequest` | `source_type`, resolved `ColumnRef`s, `lang` | What gets handed to the breakdown-function dispatcher — the engine never sees raw `$_GET` |

### Domain Events

None — this plugin has no async workflows or notifications. Admin writes
are synchronous (`update_option()` inside a single request); the public
endpoint is a pure read.

---

## Architecture

### Component Diagram

Small, single-plugin scope — the ASCII fallback is proportionate here (no
new service, no new deployable unit, so a full C4 diagram would be
over-engineering for what is one WordPress plugin's internals).

```
                         ┌─────────────────────────────┐
                         │   wp-admin (capability-gated)│
                         │  admin/class-admin.php        │
                         │  admin/views/{list,edit}.php  │
                         └───────────────┬───────────────┘
                                         │ validate() — via QuestionTypeResolver,
                                         │ reads lime_questions.type ONLY (metadata,
                                         │ never respondent data) through the same
                                         │ read-only mysqli connection below
                                         v
                         ┌─────────────────────────────┐
                         │  QuestionTypeResolver          │
                         │  (validates qid/subquestions    │
                         │  against the type allow-list    │
                         │  before anything is saved)      │
                         └───────────────┬───────────────┘
                                         │ update_option() / get_option()
                                         v
┌────────────────────┐    ┌──────────────────────────────┐
│ Public shortcode    │    │  Chart-Definition Repository   │
│ [hrd_dashboard]     │───>│  includes/class-definition-    │
│ (enqueues assets    │    │  repository.php (wp_options)   │
│  from inside the    │    └───────────────┬───────────────┘
│  render callback —  │                    │
│  Bricks-compatible) │                    v
└─────────┬───────────┘    ┌──────────────────────────────┐
          │ fetch()        │   Execution Engine              │
          v                │  includes/class-chart-engine.php│
┌─────────────────────┐   │  dispatches definition→breakdown│
│ api/class-rest-      │──>│  fn based on source_type        │
│ controller.php        │   └───────────────┬───────────────┘
│ GET /hrd-dashboard/   │                    │
│ v2/chart/{id}?lang=   │                    v
│ (public, ID-only)     │   ┌──────────────────────────────┐
└──────────────────────┘   │  Breakdown functions            │
                            │  hrd_dash_single_choice_        │
                            │  breakdown() / multi_checkbox_  │
                            │  breakdown() / standalone_yn_   │
                            │  breakdown() / or_aggregate_    │
                            │  count()                        │
                            └───────────────┬───────────────┘
                                            │ resolves columns via
                                            v
                            ┌──────────────────────────────┐
                            │  Security core (unchanged      │
                            │  regardless of engine version) │
                            │  class-db-connection.php        │
                            │   (mysqli, env-var creds)       │
                            │  class-column-resolver.php      │
                            │   (regex + information_schema)  │
                            │  class-lang.php / class-i18n.php│
                            └───────────────┬───────────────┘
                                            │ SELECT-only, 5-table
                                            │ allow-list, prepared stmts
                                            v
                            ┌──────────────────────────────┐
                            │  External LimeSurvey DB         │
                            │  (production: real server,      │
                            │   NOT reachable from this build)│
                            │  Local dev: Docker MySQL fixture│
                            └──────────────────────────────┘
```

The `QuestionTypeResolver` box and the Security core box both ultimately
read the same external LimeSurvey DB, through the same read-only mysqli
connection and the same 5-table allow-list — they are two *call sites*,
not two *connections*. The diagram draws them separately because they run
at different times (validate-on-save vs. execute-on-read), not because
they're different infrastructure.

### Data Flow

Admin write path: `admin/class-admin.php` → `validate()` → **reads
`lime_questions.type` for the target qid (and, for `multi_checkbox`, for
every auto-discovered subquestion) via the `QuestionTypeResolver`** → on
pass, the repository writes to `wp_options`. This is a **metadata-only**
read — the target qid's declared type, never a respondent's row — but it
is a real LimeSurvey DB access, correcting an earlier draft of this design
that claimed the admin write path touched no external DB at all. That
claim stopped being true the moment type-derived free-text exclusion (see
Security Considerations) replaced a static list.

Public read path: REST request → the repository loads the definition by ID
(request supplies **only the ID**) → the engine resolves it to a
`ColumnRef`/set of `ColumnRef`s via the (unchanged) column-resolver → the
matching breakdown function runs a parameterised query against the
read-only LimeSurvey connection → aggregated JSON `{labels, datasets}`
returned. At no point does a value that originated in the HTTP request
reach the SQL layer except the definition ID (used only as an
array/`wp_options` lookup key, never interpolated into SQL) and `lang`
(whitelisted to `ar`/`en` before any use).

---

## API Design

### Endpoints

| Method | Path | Purpose | Auth |
|--------|------|---------|------|
| GET | `/wp-json/hrd-dashboard/v2/chart/{id}?lang=ar\|en` | Return aggregated `{labels, datasets}` for one chart definition | **Intentionally public** — no `permission_callback` capability gate. Security boundary is "ID-only input", not authentication (brief §3.5, and PRD US-4) |
| Admin AJAX | `wp_ajax_hrd_save_definition`, `wp_ajax_hrd_delete_definition`, `wp_ajax_hrd_reorder_definitions`, `wp_ajax_hrd_import_definitions` | CRUD + reorder + validated JSON import | `check_ajax_referer('hrd_admin')` + `current_user_can('hrd_manage_dashboard')` — matches this portfolio's established AJAX pattern (`wp-taxonomy-manager`'s `ajax_rename_term`, `wp-backup-sync`'s `wbs_*` actions) rather than a second internal REST namespace for admin writes |

### Request/Response Examples

**GET /wp-json/hrd-dashboard/v2/chart/chart-5?lang=en**

Response:

```json
{
  "labels": ["Physical violence", "Sexual violence", "Threats & extortion", "Legal actions", "Financial targeting", "Defamation"],
  "datasets": [
    { "label": "Count", "data": [7, 5, 4, 6, 0, 8] }
  ]
}
```

**wp_ajax_hrd_save_definition** (admin, POST, nonce+capability gated)

Request (`$_POST`, sanitized field-by-field before use):

```
action=hrd_save_definition
nonce=<wp_create_nonce('hrd_admin')>
id=chart-11
chart_type=bar
source_type=single_choice
qid=716
title_ar=...
title_en=Region
enabled=1
order=11
```

Response on the forbidden-qid case (qid=745, or a qid/subquestion whose
LimeSurvey type resolves outside the allow-list, including an
unresolvable type):

```json
{ "success": false, "data": { "message": "This question cannot be used in a chart definition.", "code": "forbidden_qid" } }
```

### Error Responses

| Status | Code | When |
|--------|------|------|
| 404 | `rest_no_route` (WP core default) | Definition ID not found in `wp_options` **or found but `enabled: false`** — the two cases are indistinguishable to the caller by design (see Security Considerations: disabling a chart must withdraw the data, not just hide it from the shortcode) |
| 200 w/ empty dataset | `HRD_STALE_DEFINITION` (custom, in the JSON body, not an HTTP error) | Definition references a qid that no longer exists in LimeSurvey — logged for the admin, never surfaced as a fatal error to a public visitor (PRD Edge Cases) |
| 403 (`wp_send_json_error`) | `forbidden_qid` | Admin AJAX save/import references qid=745, a type-derived free-text qid, or a qid whose type couldn't be resolved (fail-closed) |
| 403 (`wp_send_json_error`) | (WP core nonce/capability failure) | AJAX nonce invalid or capability missing |
| 429 (REST, before any DB query) | `rate_limited` | Per-IP request rate over the endpoint's threshold — see **Performance & Availability**, below |

---

## Data Model

### Database Schema

**WordPress side — one `wp_options` row** (chosen over a custom table — the
trade-off is recorded in this doc's Risks section below and will be
formalised as AgDR-0002 in the plugin's own `docs/agdr/` as part of ticket
T1, once that repo has content to hold it — it does not exist yet at design
time, and this doc should not have implied otherwise):

| Field (within the JSON array) | Type | Purpose |
|-------|------|-----|
| `id` | string | Stable definition key, also the REST path segment |
| `chart_type` | enum string | Chart.js chart type |
| `source_type` | enum string | Which breakdown function handles this definition |
| `qid` / `parent_qid` / `standalone_qids` / `categories` | int / int / int[] / object[] | Shape depends on `source_type` — validated per-shape, not just per-field |
| `title_ar` / `title_en` | string | Rendered chart title |
| `enabled` | bool | Shortcode/REST visibility |
| `order` | int | Shortcode render order |

**LimeSurvey side — read-only, unchanged, exactly the 5 tables named in the
brief's allow-list**: `lime_survey_812986`, `lime_questions`,
`lime_question_l10ns`, `lime_answers`, `lime_answer_l10ns`. This plugin
never writes to, nor extends, this schema.

### Access Patterns

| Access Pattern | Query |
|----------------|-------|
| Load one definition by ID (public REST) | `get_option()` result, PHP array lookup by key — no SQL involved |
| List all definitions ordered (admin UI, shortcode) | Same `get_option()` result, sorted in PHP by `order` |
| Resolve a dynamic column name | `information_schema.columns` prepared-statement lookup, then a strict regex check (brief §2, unchanged from the existing spec) |
| Discover subquestions for a `parent_qid` | `SELECT DISTINCT qid, title FROM lime_questions WHERE parent_qid = ?` (brief §2) |
| Resolve a qid's question type (admin write path, via `QuestionTypeResolver`) | `SELECT type FROM lime_questions WHERE qid = ?` — metadata only, run on every save/import, never on the public read path (the public path only re-executes a definition that already passed this check at save time) |

---

## Implementation Plan

### Tasks

Maps directly to the 7 filed tickets (see the epic in
`yehiagamalx/hrd-survey-dashboard`):

| # | Task | Dependencies |
|---|------|--------------|
| 1 | Local dev environment + plugin foundation + security core | - |
| 2 | Breakdown engine functions + fixture-based regression tests (charts 4/5/6/8) | 1 |
| 3 | Chart-definition storage + generic execution engine + validation (incl. forbidden-qid enforcement) | 2 |
| 4 | Admin UI (CRUD + reorder + JSON import) | 3 |
| 5 | Dynamic REST endpoint + frontend shortcode | 3, 4 |
| 6 | Full regression validation (all charts, PHPUnit + live REST) | 2, 3, 5 |
| 7 | Deploy workflow artifacts (docs/scripts only) | 1 |

**Total estimate**: not time-boxed — this is a solo-maintainer build where
correctness and security review depth matter more than a calendar estimate;
each ticket gets a full review cycle (Rex + Security Auditor + explicit CEO
approval) regardless of size.

---

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Regression numbers validated only against a synthetic fixture, not real production data | High (structural — no server access in this build) | Medium (numbers could still diverge on real deploy) | Stated explicitly in the PRD, this doc, every PR description, and the repo README; the maintainer re-verifies against real data before/at deploy time |
| Admin UI or JSON-import path becomes a second, less-scrutinised way to reach a forbidden qid | Medium | Critical (sensitive-data exposure) | Single shared validation function used by *both* the form-save path and the JSON-import path — no separate, less-strict code path for either (T3/T4) |
| Dynamic column-name resolution mis-validates and enables SQL injection via a crafted qid/subcode | Low (regex + `information_schema` cross-check per brief §2, already a strong pattern) | Critical | Carried over from the brief's own already-specified defense; PHPUnit tests exercise the resolver against deliberately malformed subcodes |
| Local fixture doesn't faithfully represent real LimeSurvey column-naming edge cases (e.g. `A4`, `A5`, `other` subcodes seen live but not yet in the fixture) | Medium | Medium (a real-world subcode format slips past testing) | Fixture seed script explicitly includes the full subcode set documented in brief §2 (`A`–`D`, `A1`–`A5`, `other`), not just the minimal set |
| A new free-text question gets added to LimeSurvey after this engine ships, and a static "known free-text qids" list doesn't know about it | Medium (LimeSurvey surveys evolve) | Critical | **Not mitigated by a maintained list.** Free-text exclusion is derived at validation time from `lime_questions.type` (LimeSurvey's own question-type code — the long/short/huge free-text types), not from a static array — see Security Considerations. A lookup failure (qid not found, type unrecognised) rejects the definition rather than allowing it (fail-closed) |
| Unauthenticated public endpoint queries the external DB on every request, on a 3.8GB shared host, with no cache or rate limit | Medium (low traffic expected, but unbounded as designed) | Medium (resource contention with sibling Docker projects; a trivial DoS vector) | A short-TTL (~5 min) WordPress transient cache per `(definition ID, lang)` pair, populated on first request and served from cache thereafter; a per-IP rate limit ahead of any DB query (T5) |
| No documented rollback if the new engine misbehaves after the maintainer deploys it | Low (the old `v1` files are left untouched on the server) | Medium (unclear point-of-no-return without a written plan) | See **Rollback Plan** below — the mitigation already exists structurally (v1 untouched), this fixes it not being written down |

### wp_options vs. a custom DB table (chart-definition storage)

| Option | Pros | Cons |
|--------|------|------|
| Single `wp_options` row (chosen) | No schema/`dbDelta` to write or version; `get_option()`/`update_option()` is all that's needed for ~10-20 definitions; atomic single-row write | Whole-array read/write on every edit (fine at this scale — an admin editing one chart at a time, sequentially, not high-concurrency); no native indexing (irrelevant — the shortcode/REST paths already load and iterate the full small array in PHP) |
| Custom table (`wp_hrd_chart_definitions`) | Per-row locking semantics; a more "conventional" schema for a CRUD admin screen | Requires a `dbDelta` schema + version-upgrade routine for a table that will hold ~10-20 rows for the life of the plugin — pure overhead for this scale, and this build has no server to test a real upgrade path against anyway |

Decision: `wp_options`, single JSON-encoded row. To be recorded as
AgDR-0002 in the plugin's own `docs/agdr/` once T1 creates that directory.

### Rollback Plan

The old, hardcoded per-file `v1` system is **not removed or modified** by
this build — it continues to exist on the server exactly as it is today.
The new `v2` engine is purely additive (a new plugin, a new REST namespace,
a new shortcode tag). This gives a genuine, already-structural rollback
path once the maintainer deploys `v2`:

1. **Point of no return**: whenever the maintainer edits the live page(s)
   to swap the old shortcode/embed for the new `[hrd_dashboard]` one (or
   removes the old plugin). Until that edit, both systems can run side by
   side and the maintainer can compare outputs directly.
2. **Rollback action**: revert the page edit (swap the shortcode back) and
   optionally deactivate the new plugin. No data migration to undo — chart
   *definitions* live only in the new plugin's `wp_options`; nothing in
   `v1`'s files or the LimeSurvey DB is touched by `v2` at any point.
3. **Owned by**: the maintainer, at deploy time — this build does not
   perform the deploy, so it cannot execute the rollback itself, but the
   plan needs to exist in writing before that point, not be improvised then.

---

## Security Considerations

- [x] LimeSurvey DB access via a dedicated `SELECT`-only user on a
      5-table allow-list (assumed already provisioned server-side per
      brief §3.1 — this build's `class-db-connection.php` enforces the
      allow-list in code as a second layer, not just trusting the DB grant)
- [x] mysqli only (`MYSQLI_REPORT_ERROR | MYSQLI_REPORT_STRICT`), prepared
      statements for every variable value — no PDO (unavailable in the
      `wordpress:latest` image) and no ad-hoc string interpolation
- [x] Dynamic column/identifier names validated by strict regex +
      `information_schema` cross-check before ever appearing in a SQL
      string (identifiers can't be bound parameters)
- [x] LimeSurvey credentials via `putenv()`/`getenv()` only
      (`HRD_LIME_DB_{HOST,PORT,NAME,USER,PASS}`) — never hardcoded, never
      stored in `wp_options`
- [x] Public REST endpoint takes **only** a pre-stored definition ID +
      whitelisted `lang` — no request parameter ever resolves to a qid,
      column, or query shape
- [x] Admin write paths (form save + JSON import) share **one** validation
      function that hard-rejects qid=745 (static deny-list, first check),
      **then** checks every qid the definition would touch — the top-level
      `qid`/`parent_qid`/`standalone_qids`, **and every subquestion
      auto-discovered for a `multi_checkbox` definition** — against a
      fixed **allow-list** of LimeSurvey question types:
      `HRD_ALLOWED_QUESTION_TYPES = ['L', 'M', 'Y']` (list/radio,
      multiple-choice-with-subquestions, yes/no — exactly the types the
      brief's own 10 charts use). This is an allow-list, not a deny-list,
      specifically because a deny-list of "known free-text types" fails
      open on anything new or unanticipated; an allow-list fails closed by
      construction. Any qid/subquestion whose type is absent from the
      allow-list is rejected — this includes both genuine free-text types
      (long/short/huge text) **and** the `O`/`P` comment-enabled question
      types, whose per-item "comment" sub-columns are free text even
      though the parent question itself is a legitimate categorical type.
      The brief's own `...other` checkbox-selected flag (chart 4) is a
      `Y`/`N` indicator of whether "other" was chosen, not the free text
      itself, and is counted the same as any other subquestion — it is
      **not** a carve-out from the allow-list, it simply already satisfies
      it. A lookup failure (qid not found, or a type code the resolver
      doesn't recognise) **rejects** the definition — fail-closed, never
      fail-open. Expanding the allow-list to a new type is a deliberate
      code change with its own review, not a config toggle.
- [x] `enabled: false` withdraws a chart from the **public REST endpoint
      as well as** the shortcode — the endpoint returns the same 404 for
      "ID not found" and "ID found but disabled". The definition-level
      cache (see Performance & Availability, below) is invalidated
      **synchronously, as part of the same write**, for every save,
      delete, and enable/disable toggle — so a disable takes effect
      immediately, not after the cache's TTL expires. Without that
      invalidation rule, the cache and the disable control would
      contradict each other (a gap flagged in Solution Architect review);
      with it, both "visible immediately after enabling" (PRD US-1) and
      "gone immediately after disabling" (PRD US-2/US-4) hold at once.
- [x] `MYSQLI_REPORT_STRICT` exceptions are caught at the plugin boundary
      and never allowed to propagate a raw DB error (which could include
      schema/query detail) into the public REST response — logged
      internally, a generic error returned externally
- [x] No PII in logs — the "stale definition" logging path (Edge Cases)
      logs only the qid and definition ID, never response content

---

## Performance & Availability

The public endpoint is unauthenticated by design (PRD US-3/brief §3.5) and
runs on a 3.8GB host shared with other Docker Compose projects — it needs a
concrete ceiling, not just a stated intention to have one. These are
starting defaults for a low-traffic public transparency dashboard, not
numbers derived from load testing (none is possible without the real
server); the maintainer can retune them post-deploy.

| Control | Default | Rationale |
|---------|---------|-----------|
| Per-IP rate limit | 30 requests/minute per IP, per chart ID | Generous enough for a real visitor loading all ~10 charts on one page load more than once; tight enough that a scripted loop can't drive unbounded query volume |
| Cache TTL | 5 minutes, WordPress transient keyed `hrd_chart_{id}_{lang}` | Short enough that a legitimate edit (US-1) is never stale for long even if invalidation somehow missed a path; long enough to absorb repeat requests without re-querying the external DB every time |
| Cache invalidation | Synchronous, on every save/delete/enable-toggle/import — not just TTL expiry | The load-bearing rule that keeps the cache from contradicting the `enabled` disable control (see Security Considerations) |
| Rate-limit enforcement point | Before the cache lookup and before any DB connection is opened | A rate-limited request costs nothing beyond reading a WordPress transient/option for the counter itself |

Rate-limit and cache state both live in WordPress transients (no new
service, no new dependency — consistent with the `wp_options`-only storage
decision above).

---

## Testing Strategy

| Type | Coverage | Notes |
|------|----------|-------|
| Unit | Validation layer, column-resolver regex, forbidden-qid rejection (including type-derived free-text rejection and fail-closed-on-lookup-failure), disabled-definition 404 behavior | Plain PHPUnit + hand-rolled WP-function stubs (this portfolio's `wp-taxonomy-manager` pattern — no Brain Monkey/WP_Mock dependency) |
| Integration | The four breakdown functions against a local Docker MySQL fixture | Real mysqli against a real MySQL instance, not a mock — the fixture is seeded to reproduce the brief's exact published numbers for charts 4/5/6/8 |
| Browser (manual, via `claude-in-chrome`) | Admin CRUD UI (T4), shortcode rendering + REST wiring (T5) | Local WordPress dev site via `docker-compose.dev.yml`; this project's standing rule that UI changes get browser-verified before being called done |
| Regression (T6) | All 10 charts, not just the 4 with published numbers | PHPUnit + live REST calls against the local dev site; explicitly flagged as fixture-only, not production-verified |

---

## Open Questions

| Question | Owner | Status |
|----------|-------|--------|
| Should the admin UI's `hrd_manage_dashboard` capability be assignable to a role below Administrator (e.g. Editor)? | Tech Lead | Deferred — default to Administrator-only for the MVP; revisit if the maintainer wants to delegate |
| Small-cell disclosure risk: several charts can produce n=1 cells when intersected across the ten published charts, which is a pre-existing property carried over from the Metabase-era report, not something this rebuild introduces | Maintainer | Accepted risk, carried forward unchanged — flagged here rather than silently passed over, per Solution Architect review. Not blocking this build; revisit if the maintainer wants a suppression threshold (e.g. redact counts below N) in a future ticket |

---

## Approvals

| Role | Name | Date | Status |
|------|------|------|--------|
| Tech Lead | Hisham | 2026-09-19 | Author |
| Solution Architect | Tariq | | Pending (`/design-review`) |
| Security (if needed) | Hakim | | Pending (per-ticket, not gated here) |

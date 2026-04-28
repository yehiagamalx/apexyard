# ari-publication-importer — Handover Assessment

**Date**: 2026-04-28
**Assessor**: Yehia Gamal
**Status**: handover

---

## Origin

- **Where it came from**: Built in-house for Arab Reform Initiative
- **Original owner**: Yehia Gamal
- **Repo location**: `/Users/ye/Projects/ari/ari-publication-importer` (local only — not on GitHub)
- **First commit date**: unknown (no git repository initialised)
- **Last commit date**: unknown (no git repository initialised)

---

## Current State

### Tech Stack

- **Language**: PHP 7.4+
- **Runtime**: WordPress Plugin API
- **Framework**: WordPress admin + ACF Pro + WPML
- **Database**: MySQL (via WordPress — no direct queries)
- **Frontend**: jQuery (admin UI only)
- **External tools**: Pandoc (docx/rtf → HTML conversion, called via `exec()`)
- **External APIs**: Anthropic Claude API (Mode 2 smart detection)
- **Test framework**: none
- **CI**: none

### Build Status

- `git init`: **not run** — no version history at all
- `npm install`: n/a (no Node dependencies)
- Tests: **none exist**
- Lint: **none configured**

### Test Coverage

- **0%** — no test framework, no test files found

### Repo Activity

- Commits in last 90 days: **unknown** (no git)
- Open issues: **none** (not on GitHub)
- Open PRs: **none** (not on GitHub)
- Top contributors: Yehia Gamal (sole author based on CHANGELOG)

---

## Quality Risks

### Critical

- **No git repository** — the entire codebase has no version history. Any change overwrites the previous state with no rollback. This is the highest-risk gap and must be fixed before any further development.
- **No GitHub remote** — plugin exists only on one local machine. One disk failure = total loss.

### Security

- **File upload handling** — `class-admin-page.php` handles `.docx`/`.rtf`/`.aripack` uploads. File type is validated server-side, but the exec() call to Pandoc uses `escapeshellarg()` (verify this is applied to the full path). Worth a security review pass.
- **API key storage** — Anthropic API key stored in `wp_options` (standard WP pattern, reasonable). Not hardcoded.
- **`exec()` usage** — Pandoc is invoked via shell exec. The file path must be fully escaped. This is the highest code-level security surface.

### Dependencies

- **Pandoc** — external binary, not bundled. Must be installed on the server. Version not pinned. If the server is updated or Pandoc breaks, the plugin silently fails.
- **ACF Pro** — premium plugin dependency. Not declared anywhere. If it's deactivated, all ACF field saves will fail silently.
- **WPML** — same: undeclared dependency, silent failure if absent.
- **Anthropic SDK** — called via raw `wp_remote_post()` (no SDK). This is fine but the API version (`anthropic-version` header) is hardcoded and will need updating when Anthropic deprecates it.

### Technical Debt

- **No tests** — 2383 lines of business logic, zero test coverage. Author matching, tag matching, content cleaning, WPML integration — all untested.
- **No README** — no onboarding doc (stub created in this assessment).
- **`aripack.py`** is a standalone Python script in the plugin root — not wired into the plugin, not tested, no requirements file.
- **`egypt.aripack`** sample file in the root — should be in a `samples/` or `tests/` folder, not root.

### Operational

- **No CI/CD** — no automated lint, test, or build on any trigger.
- **No deploy automation** — deployment is manual rsync or file copy.
- **No error monitoring** — failures surface only as admin error messages; nothing is logged or alerted.
- **Pandoc version not recorded** — the CHANGELOG references Pandoc behaviour but no version requirement is documented.

---

## Integration Plan

### Roles That Apply

- `tech-lead` — owns architecture decisions and code review gate
- `backend-engineer` — PHP plugin logic, AJAX handlers, ACF/WPML integration
- `frontend-engineer` — jQuery admin UI (upload form, review screen)
- `security-auditor` — file upload handling, exec() shell calls, API key storage

### Workflows That Kick In

- [ ] PR workflow — every change through a PR (once git + GitHub are set up)
- [ ] AgDR for technical decisions (Pandoc version pinning, Claude model choice)
- [ ] Code Reviewer agent (Rex) on every PR
- [ ] Security Reviewer on any PR touching file upload or exec() code
- [ ] `/audit-deps` — monthly (Pandoc, WPML, ACF versions)

### Hooks to Enable (once on GitHub)

- [ ] `block-git-add-all`
- [ ] `block-main-push`
- [ ] `validate-branch-name`
- [ ] `validate-pr-create`
- [ ] `pre-push-gate`
- [ ] `check-secrets`

### CI Templates to Copy In (once on GitHub)

- [ ] `golden-paths/pipelines/ci.yml`
- [ ] `golden-paths/pipelines/security.yml`
- [ ] `golden-paths/pipelines/pr-title-check.yml`

---

## Next Steps

1. **`git init` + GitHub repo** — run `git init`, make a first commit, create `yehiagamalx/ari-publication-importer` on GitHub, push. Until this is done, no other process improvements are safe (you can't branch, review, or roll back).
2. **Security review on `exec()` and file upload** — run `/security-review` on `class-admin-page.php` and `class-docx-parser.php` before the next feature. The `exec()` + file upload combination is the highest code-level risk.
3. **Set up a minimum test baseline** — even 3-5 unit tests on `class-metadata-extractor.php` (the parser logic) and `class-content-cleaner.php` establish a regression floor before further changes.
4. **Pin Pandoc version** — document the minimum required Pandoc version in README and add a server-side check in the settings page "Test Pandoc" button.
5. **Stakeholder sync** — walk through the CHANGELOG with the ARI editorial team to confirm which recent fixes (Arabic RTF, WPML linking, category picker) have been validated in production.

---

## Post-Handover Checklist

- [ ] `git init` + push to GitHub (blocker for everything else)
- [ ] Security review on file upload + exec() surface — close before next feature PR
- [ ] Set up test coverage baseline (even minimal PHPUnit stubs)
- [ ] Add `ari-publication-importer` to weekly `/stakeholder-update` rollup
- [ ] Move `egypt.aripack` sample to `samples/` folder
- [ ] Document Pandoc version requirement in README
- [ ] Run `/audit-deps` monthly for the next 3 months once on GitHub

---

## Open Questions

- What GitHub org/account should this live under? (`yehiagamalx/` like profile-photo-processor, or `arab-reform-initiative/`?)
- Is `aripack.py` production-used or just a dev utility? If production, it needs its own requirements.txt and documentation.
- Which Pandoc version is installed on the production server?
- Is ACF Pro and WPML version locked on the production site?

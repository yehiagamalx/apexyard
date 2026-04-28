# WordPress Plugin Standards

Applies to every WordPress plugin managed in this portfolio.

## On Every Fix or Update (MANDATORY)

Before committing any code change to a WordPress plugin, always complete these three steps **in order**:

### 1. Bump the Version

Version follows semantic versioning (`MAJOR.MINOR.PATCH`):

| Change type | Bump |
|-------------|------|
| Bug fix, no new behaviour | PATCH (1.1.0 → 1.1.1) |
| New feature, backwards-compatible | MINOR (1.1.1 → 1.2.0) |
| Breaking change, removes existing behaviour | MAJOR (1.2.0 → 2.0.0) |

Update in **two places** in the main plugin `.php` file:
```php
 * Version:     X.Y.Z          ← plugin header comment (~line 5)
define( 'PLUGIN_VERSION', 'X.Y.Z' );  ← version constant (~line 14)
```

Each plugin has its own constant name — check the plugin header.

### 2. Add a CHANGELOG Entry

Prepend a new block at the top of `CHANGELOG.md` (below the `# Changelog` header):

```markdown
## [X.Y.Z] — YYYY-MM-DD

### Fixed / Added / Changed / Removed
- One-line description of each change.
```

### 3. Rebuild the ZIP

The ZIP is the distributable. Rebuild it after every commit. The command lives in each plugin's `CLAUDE.md`. General pattern:

```bash
cd /Users/ye/Projects/apexyard/workspace
zip -r <plugin-name>.zip <plugin-name> \
  --exclude "<plugin-name>/.git/*" \
  --exclude "<plugin-name>/.claude/*"
```

The zip is stored at `workspace/<plugin-name>.zip` alongside the plugin folder.

---

## About Page (MANDATORY for every plugin)

Every plugin **must** have an "About" tab on its WordPress settings page. The tab shows:

- Plugin name, version, author
- What the plugin does (2–3 sentences)
- Usage overview
- Changelog (rendered from `CHANGELOG.md`)
- GitHub repo link

Implementation: add a `?tab=about` branch in the settings page `render_page()` method using WordPress's standard `nav-tab-wrapper` UI. Read and display `CHANGELOG.md` using `file_get_contents( plugin_dir_path( __FILE__ ) . '../CHANGELOG.md' )`.

---

## readme.txt (MANDATORY for every plugin)

Every plugin must have a `readme.txt` at the root. Use WordPress.org format even for private plugins — it serves as documentation:

```
=== Plugin Name ===
Contributors: yehia
Tags: relevant, tags
Requires at least: 6.0
Tested up to: 6.5
Stable tag: X.Y.Z
License: Proprietary

One-line description.

== Description ==
Full description. What the plugin does and why.

== Installation ==
1. Upload the plugin folder to /wp-content/plugins/
2. Activate in Plugins → Installed Plugins
3. Configure under Settings → Plugin Name

== Changelog ==
= X.Y.Z =
* Latest changes (mirror the top CHANGELOG.md entry)
```

Update the `Stable tag` and `== Changelog ==` section on every version bump.

---

## Sync to Local WordPress Install

Each plugin's `CLAUDE.md` contains the exact rsync command for that plugin's local WP path. Run it after rebuilding the zip when testing locally:

```bash
rsync -a --delete \
  /Users/ye/Projects/apexyard/workspace/<plugin-name>/ \
  "/Users/ye/Local Sites/<site>/app/public/wp-content/plugins/<plugin-name>/" \
  --exclude=".git" --exclude=".claude/"
```

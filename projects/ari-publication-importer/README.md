# ARI Publication Importer

WordPress plugin for Arab Reform Initiative. Imports `.docx` / `.rtf` / `.aripack` files into WordPress as publication draft posts.

## Repo

TBD — not yet on GitHub. See handover assessment for next steps.

## Local path

`/Users/ye/Projects/ari/ari-publication-importer`

## Stack

- PHP + WordPress Plugin API
- ACF Pro (Advanced Custom Fields)
- WPML (EN / AR / FR)
- Pandoc (external binary — docx/rtf → HTML)
- jQuery (admin UI)
- Anthropic Claude API (Mode 2 smart detection)

## Key Features

- Mode 1: metadata lines (`Author:` / `Tags:`) parsed from the document
- Mode 2: Claude API detects authors and suggests tags from document content
- Review screen before any post is created — always creates a `draft`, never auto-publishes
- WPML translation linking on the review screen
- `.aripack` format support (ZIP of `content.html` + images)
- Arabic RTF encoding fix (CP1256 → UTF-8 via iconv)

## Local Setup

1. Copy plugin folder to `wp-content/plugins/`
2. Activate in WordPress admin
3. Install Pandoc on the server: `apt install pandoc` or set custom path in Settings → ARI Importer
4. Set Anthropic API key in Settings → ARI Importer (required for Mode 2 only)

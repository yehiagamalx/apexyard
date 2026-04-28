# Profile Photo Processor

WordPress plugin that removes the background from a profile photo, places the subject on a `#D9D9D9` canvas (600×750px), and provides an admin approval workflow before saving.

## Repo
https://github.com/yehiagamalx/profile-photo-processor

## Stack
- PHP + WordPress Plugin API
- Remove.bg API (background removal)
- PHP GD / Imagick (image compositing)

## Key Features
- Manual-trigger only — "Process Photo" button in CPT meta box
- AJAX processing with loading spinner
- Before/after preview with approve/discard
- Settings page for Remove.bg API key
- Post meta flag `_photo_processed` set only on approval

## Local Setup
1. Copy plugin folder to `wp-content/plugins/`
2. Activate in WordPress admin
3. Set API key under Settings → Profile Photo Processor

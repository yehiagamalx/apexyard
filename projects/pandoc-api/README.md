# pandoc-api

Pandoc DOCX-to-HTML conversion API server. Called by the ARI Publication Importer WordPress plugin.

**Repo**: [yehiagamalx/pandoc-api](https://github.com/yehiagamalx/pandoc-api)  
**Image**: `ghcr.io/yehiagamalx/pandoc-api:latest`  
**Status**: active

## Overview

Self-contained Flask API running inside Docker. Accepts a `.docx` file via `POST /convert` and returns the converted HTML plus base64-encoded embedded images. Sits behind Nginx Proxy Manager on a VPS.

## Stack

- Python 3.12 + Flask + Gunicorn
- Pandoc (Debian apt)
- Docker + GHCR
- Watchtower for auto-deploy

## Key decisions

- [AgDR-0001: Flask/Docker/GHCR architecture](../../workspace/pandoc-api/docs/agdr/AgDR-0001-flask-docker-ghcr-deployment.md)

## Contacts

- Owner: Yehia Gamal

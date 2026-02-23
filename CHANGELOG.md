# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

---

## [Unreleased]

---

## [1.0.0] - 2026-02-23

### Added
- Initial `docker-compose.yml` for Portainer stack deployment
- Base image `ghcr.io/astral-sh/uv:python3.12-bookworm-slim` (Debian Bookworm Slim + Python 3.12 + uv)
- Automatic git clone on first container start, pull on subsequent starts
- Support for public GitHub repositories
- Support for private GitHub repositories via personal access token (`REPO_AUTH_TYPE=token`)
- Support for private GitHub repositories via SSH deploy key (`REPO_AUTH_TYPE=ssh`)
- Background repo watcher — polls `git fetch` on a configurable interval and applies updates without container restart
- Graceful Gunicorn reload (`SIGHUP`) when new commits are detected, preserving in-flight requests
- Automatic dependency installation from `requirements.txt` or `pyproject.toml` on start and after each update
- Fallback Flask install when no dependency file is present in the cloned repo
- Configurable `DYNAMIC_STORAGE` and `STATIC_STORAGE` volume mount paths
- Named Docker volumes (`app_dynamic`, `app_static`) automatically scoped per stack by Portainer
- Configurable Gunicorn workers, threads, port, and app module via environment variables
- Clean shutdown handler — `SIGTERM`/`SIGINT` stops Gunicorn gracefully before the container exits
- `README.md` with full deployment, authentication, storage, and troubleshooting documentation

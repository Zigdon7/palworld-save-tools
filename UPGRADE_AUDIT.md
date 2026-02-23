# Project Upgrade Audit

**Date:** 2026-02-23
**Last meaningful activity:** January 2025 (core updates), February 2024 (API/config changes)

---

## Executive Summary

This project has multiple areas requiring upgrades across Python version support, dependencies, Docker infrastructure, CI/CD pipelines, and tooling. Several components are at or past end-of-life. The most urgent items are the Docker base image (Debian Buster EOL), Python 3.9 in the CI matrix (EOL), and GitHub Actions running on Node 20 (deprecating March 2026).

---

## 1. Python Version Support

| Area | Current | Recommended | Priority |
|------|---------|-------------|----------|
| `pyproject.toml` constraint | `~3.11.0` (3.11.x only) | `>=3.11,<3.15` | High |
| CI test matrix | 3.9, 3.10, 3.11, 3.12 | 3.11, 3.12, 3.13, 3.14 | High |
| Dockerfile base | `python:3.11-slim-buster` | `python:3.13-slim-bookworm` | Critical |
| `.pylintrc` py-version | 3.11 | 3.13 | Low |

**Details:**
- **Python 3.9** reached EOL in October 2025. Remove from CI matrix.
- **Python 3.10** is nearing EOL (~2026). Consider dropping.
- **Python 3.13** and **3.14** are the current actively-supported releases. Add to CI matrix.
- The `~3.11.0` tilde constraint in `pyproject.toml` locks to 3.11.x only. This should be widened to support 3.12+ at minimum.

---

## 2. Docker Infrastructure

### 2a. Base Image (CRITICAL)

| Item | Current | Issue | Recommended |
|------|---------|-------|-------------|
| Base image | `python:3.11-slim-buster` | Debian Buster reached EOL June 2024. No security patches. | `python:3.13-slim-bookworm` or `python:3.13-slim-trixie` |

### 2b. Poetry in Docker

| Item | Current | Latest | Notes |
|------|---------|--------|-------|
| Poetry version | 1.6.1 | 2.3.2 | Major version bump. Poetry 2.x changed dependency resolution and `pyproject.toml` handling. `--no-dev` flag replaced with `--without dev`. |

### 2c. Dockerfile Syntax

- The `as` keyword in `FROM ... as ...` should use uppercase `AS` per modern convention.
- `poetry install --no-dev` is deprecated in Poetry 2.x; use `poetry install --without dev`.

### 2d. docker-compose.yml

| Item | Current | Issue | Recommended |
|------|---------|-------|-------------|
| `version: '3.8'` | Present | Deprecated and ignored by modern Compose | Remove the line entirely |
| File name | `docker-compose.yml` | Legacy convention | Rename to `compose.yaml` |
| PostgreSQL image | `postgres:latest` | Unpinned, will break unexpectedly | Pin to `postgres:17` or similar |

---

## 3. Python Dependencies

### 3a. Direct Dependencies (pyproject.toml)

| Package | Current | Latest | Change | Risk | Notes |
|---------|---------|--------|--------|------|-------|
| `fastapi` | `^0.103.2` | `0.131.0` | Minor (0.x) | Medium | Caret on 0.x constrains to `<0.104`. Many API changes across 28 releases. |
| `uvicorn` | `^0.23.2` | `0.41.0` | Minor (0.x) | Medium | Caret on 0.x constrains to `<0.24`. 18 minor versions behind. |
| `pyyaml` | `^6.0.1` | `6.0.3` | Patch | Low | Already allowed by caret constraint. |
| `pydantic-settings` | `^2.1.0` | `2.13.1` | Minor | Low | Already allowed by caret constraint. |
| `python-multipart` | `=0.0.6` | `0.0.22` | Patch | Medium | Exact pin blocks all upgrades. Now requires Python >=3.10. |
| `aiofiles` | `^23.2.1` | `25.1.0` | Major (CalVer) | Medium | Caret constrains to `<24.0`. Two calendar years behind. |
| `palworld-save-tools` | `0.17.1` | `0.24.0` | Minor | Medium | 7 minor versions behind. Review changelog for breaking changes. |

### 3b. Problematic Dependency: `fastapi-router-controller`

| Item | Detail |
|------|--------|
| Source | `git = "https://github.com/ResTrekAI/fastapi-router-controller.git"` |
| Status | **Unmaintained.** Original repo (KiraPC) last released v0.5.0 in Feb 2022. Fork (ResTrekAI) has minimal activity. |
| Risk | **High.** Git-based dependency with no version pin. Could break at any time. No PyPI release from the fork. |
| Recommendation | Either fork and vendor this dependency, switch to the PyPI package `fastapi-router-controller==0.5.0` (from original author), or remove the dependency and use FastAPI's built-in `APIRouter` directly. |

### 3c. Stale Configuration Reference

`config.yaml` references `agent.openai_model: "gpt-3.5-turbo-0613"`. The `gpt-3.5-turbo-0613` model was deprecated by OpenAI in June 2024. If this agent functionality is still used, it needs updating. If not, this config should be removed.

---

## 4. CI/CD — GitHub Actions

### 4a. Action Versions (URGENT — Node 20 deprecation)

Starting **March 2026**, GitHub will begin deprecating Node 20 actions. The v4 versions of first-party actions all run on Node 20 and will stop working in summer 2026.

| Action | Current | Latest | Priority |
|--------|---------|--------|----------|
| `actions/checkout` | `v4` | `v6` | **High** — Node 20 EOL |
| `actions/setup-python` | `v4` | `v6` | **High** — Node 20 EOL |
| `actions/upload-artifact` | `v4` | `v6` | **High** — Node 20 EOL |
| `pre-commit/action` | `v3.0.0` | `v3.0.1` | Low — patch update |
| `softprops/action-gh-release` | `v1` | `v2` | Medium — major version |
| `proudust/gh-describe` | `v1` | `v2.1.1` | Medium — major version |

### 4b. CI Matrix Updates

**File:** `.github/workflows/premerge.yaml`

- Remove Python 3.9 (EOL)
- Consider removing Python 3.10 (near EOL)
- Add Python 3.13 and 3.14
- Linter Python version (currently 3.12) should be updated to 3.13 or 3.14

### 4c. Workflow Improvements

- `premerge.yaml` triggers on all pushes (`on: [push]`). Consider restricting to `push` + `pull_request` on specific branches.
- `package.yaml` also triggers on all pushes. The packaging step runs on every push even though releases only happen on tags.
- The `softprops/action-gh-release@v2` uses `github.ref_type == 'tag'` instead of `startsWith(github.ref, 'refs/tags/')`.

---

## 5. Pre-commit & Linting Tools

| Tool | Current | Latest | Change Type |
|------|---------|--------|-------------|
| `pre-commit/pre-commit-hooks` | `v4.5.0` | `v6.0.0` | Major (2 versions) |
| `black` | `24.1.1` | `26.1.0` | Major (CalVer, 2 years) |
| `isort` | `5.13.2` | `6.0.0+` | Major |
| `pylint` (in .pylintrc) | Config targets 3.11 | Should target 3.13 | Config update |

**Notes:**
- `black` 26.x may reformat existing code. Run `black --check .` after upgrading to preview changes.
- `isort` had breaking config changes in major versions. Review `.pre-commit-config.yaml` after upgrade.
- Consider adding `ruff` as a faster alternative to both `black`, `isort`, and `pylint`.

---

## 6. Build & Packaging System

| Item | Current | Issue | Recommendation |
|------|---------|-------|----------------|
| Poetry version | 1.6.1 (Docker) | Poetry 2.x is current | Upgrade to Poetry 2.x. Update `--no-dev` to `--without dev`. |
| Package name | `palword-sav-api` (typo) | Missing 'l' in "palworld" | Fix to `palworld-sav-api` |
| Build backend | `poetry.core.masonry.api` | Still valid | No change needed |

---

## 7. Security Considerations

| Item | Concern | Recommendation |
|------|---------|----------------|
| Docker base image | Debian Buster EOL — no security patches since June 2024 | Upgrade to Bookworm or Trixie immediately |
| `python-multipart` pinned at 0.0.6 | Older versions may have unpatched vulnerabilities | Upgrade to latest |
| `postgres:latest` in docker-compose | Unpinned database image | Pin to a specific major version |
| Git dependency (fastapi-router-controller) | Unversioned, from a fork | Pin or vendor |
| CORS allows localhost origins | Fine for dev, should be configurable for production | Ensure production config restricts origins |
| DB credentials in docker-compose.yml | Hardcoded `user`/`password` | Use `.env` file or secrets management |

---

## 8. Miscellaneous Issues

| Item | Issue | Recommendation |
|------|-------|----------------|
| `.pylintrc` references Python 2/3 compat | Comment mentions "support both Python 2 and 3" | Clean up — Python 2 is irrelevant |
| `tests/requirements.txt` | Referenced in CI but file not found in repo | Create the file or fix CI step |
| Dual core implementations | `api/core/` and `api/core3/` exist side by side | Determine if `core3/` should replace `core/`, remove the unused one |
| Package name typo | `palword-sav-api` in pyproject.toml | Fix to `palworld-sav-api` |

---

## Recommended Upgrade Order

### Phase 1 — Critical / Security (do first)
1. Upgrade Docker base image from Buster to Bookworm/Trixie
2. Upgrade GitHub Actions to v6 (Node 20 deprecation imminent)
3. Remove Python 3.9 from CI matrix

### Phase 2 — Dependencies
4. Upgrade `fastapi` to latest 0.13x
5. Upgrade `uvicorn` to latest 0.4x
6. Upgrade `python-multipart` to latest
7. Upgrade `aiofiles` to 25.x
8. Upgrade `palworld-save-tools` to 0.24.0
9. Resolve `fastapi-router-controller` dependency (vendor, pin, or remove)

### Phase 3 — Tooling
10. Upgrade Poetry to 2.x (in Dockerfile and locally)
11. Upgrade pre-commit hooks (`black`, `isort`, `pre-commit-hooks`)
12. Consider migrating to `ruff` for linting/formatting
13. Update `.pylintrc` target Python version

### Phase 4 — Cleanup
14. Fix package name typo (`palword` -> `palworld`)
15. Remove deprecated `config.yaml` OpenAI reference
16. Clean up docker-compose.yml (remove `version`, pin postgres, rename file)
17. Resolve dual `core/` vs `core3/` implementations
18. Add or fix `tests/requirements.txt`
19. Widen Python version constraint in `pyproject.toml`
20. Add Python 3.13 and 3.14 to CI matrix

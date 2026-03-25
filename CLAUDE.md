# maisen-ci

Shared reusable GitHub Actions workflows for Maisengasse GmbH projects.

## Structure

All workflows are in `.github/workflows/`:
- `django-lint-test.yml` — Python lint (Ruff) + pytest with PostgreSQL & Redis services
- `vue-build.yml` — Node.js lint + Vite build
- `docker-build-push.yml` — Docker Buildx build + push to ghcr.io with layer caching
- `deploy.yml` — SSH-based deploy: SCP files to server, write .env from secret, docker compose pull/up, migrations, collectstatic, health check, git tag

## Key Design Decisions

- All workflows use `workflow_call` (reusable workflows)
- Deploy does NOT require a git repo on the server — files are copied via SCP
- `.env` is written from a GitHub Secret (`ENV_FILE`) to avoid storing secrets on the server filesystem permanently
- Deploy creates a git tag (`deploy/YYYY-MM-DD-<sha>`) after successful health check
- `compose_files` input accepts full docker compose flags (e.g. `--env-file .env -f base.yml -f prod.yml`)
- Ruff is installed separately in the lint workflow (not required in project's requirements.txt)
- JWT env vars (`JWT_ALGORITHM`, `JWT_SECRET_KEY`) are set in the test environment for ninja_jwt compatibility

## Testing Changes

This repo has no tests. Changes are validated by running the consumer project's CI pipeline (e.g. intersport-budget).

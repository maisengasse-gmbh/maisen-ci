# maisen-ci

Shared GitHub Actions CI/CD workflows for Maisengasse projects.

## Available Workflows

| Workflow | Purpose |
|----------|---------|
| `django-lint-test.yml` | Ruff lint + pytest with PostgreSQL & Redis |
| `vue-build.yml` | ESLint + Vite build |
| `docker-build-push.yml` | Docker build + push to ghcr.io |
| `deploy.yml` | SSH-based deploy with Docker Compose |

## Usage

In your project's `.github/workflows/ci.yml`:

```yaml
name: CI/CD
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:

jobs:
  lint-test:
    uses: maisengasse-gmbh/maisen-ci/.github/workflows/django-lint-test.yml@main
    with:
      working_directory: path/to/django
      pytest_args: "--ds myapp.settings.test -x"
      project_name: myproject
      project_verbose_name: "My Project"

  vue-build:
    uses: maisengasse-gmbh/maisen-ci/.github/workflows/vue-build.yml@main
    with:
      working_directory: path/to/frontend

  build-backend:
    if: (github.event_name == 'push' || github.event_name == 'workflow_dispatch') && github.ref == 'refs/heads/main'
    needs: [lint-test, vue-build]
    uses: maisengasse-gmbh/maisen-ci/.github/workflows/docker-build-push.yml@main
    permissions:
      contents: read
      packages: write
    with:
      image_name: my-project/backend
      dockerfile: path/to/Dockerfile
      context: path/to/context
      build_args: |
        PROJECT_NAME=myproject

  deploy:
    if: (github.event_name == 'push' || github.event_name == 'workflow_dispatch') && github.ref == 'refs/heads/main'
    needs: [build-backend]
    uses: maisengasse-gmbh/maisen-ci/.github/workflows/deploy.yml@main
    permissions:
      contents: write
    with:
      deploy_path: /home/deploy/my-project
      compose_source_files: "docker/docker-compose.base.yml docker/docker-compose.prod.yml"
      compose_files: "-f docker/docker-compose.base.yml -f docker/docker-compose.prod.yml"
      env_file_path: ".env"
      health_check_url: https://my-project.example.com
      ssh_user: deploy
      ssh_host: server.example.com
    secrets:
      SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
      SSH_KNOWN_HOSTS: ${{ secrets.SSH_KNOWN_HOSTS }}
      ENV_FILE: ${{ secrets.ENV_FILE }}
```

## Workflow Inputs

### django-lint-test.yml

| Input | Default | Description |
|-------|---------|-------------|
| `python_version` | `3.12` | Python version |
| `working_directory` | `.` | Path to Django project |
| `ruff_args` | `check .` | Ruff linter arguments |
| `pytest_args` | `--ds settings.test -x` | Pytest arguments |
| `postgres_version` | `16` | PostgreSQL version |
| `database_url` | `postgres://postgres:postgres@localhost:5432/test_db` | Database URL |
| `cache_url` | `redis://localhost:6379/0` | Cache URL |
| `project_name` | `""` | Django PROJECT_NAME env var |
| `project_verbose_name` | `""` | Django PROJECT_VERBOSE_NAME env var |

### vue-build.yml

| Input | Default | Description |
|-------|---------|-------------|
| `node_version` | `22` | Node.js version |
| `working_directory` | `.` | Path to Vue project |
| `lint_command` | `npm run lint` | Lint command |
| `build_command` | `npm run build` | Build command |

### docker-build-push.yml

| Input | Default | Description |
|-------|---------|-------------|
| `image_name` | (required) | Image name under `ghcr.io/<org>/` |
| `dockerfile` | `Dockerfile` | Path to Dockerfile |
| `context` | `.` | Docker build context |
| `build_args` | `""` | Newline-separated build args |

### deploy.yml

| Input | Default | Description |
|-------|---------|-------------|
| `deploy_path` | (required) | Absolute path on server |
| `compose_files` | `-f docker-compose.base.yml -f docker-compose.prod.yml` | Docker Compose flags for `docker compose` commands |
| `compose_source_files` | `""` | Space-separated files to SCP to server before deploy |
| `env_file_path` | `.env` | Path (relative to deploy_path) where ENV_FILE secret is written |
| `health_check_url` | `""` | URL for health check after deploy |
| `run_migrations` | `true` | Run Django migrations |
| `migration_command` | `python manage.py migrate --noinput` | Migration command |
| `ssh_user` | (required) | SSH username |
| `ssh_host` | (required) | SSH hostname |
| `create_tag` | `true` | Create a git tag after successful deploy |
| `tag_prefix` | `deploy` | Tag prefix (e.g. `deploy/2026-03-24-a1b2c3d`) |

**Secrets:**

| Secret | Required | Description |
|--------|----------|-------------|
| `SSH_PRIVATE_KEY` | Yes | Ed25519 SSH private key for deployment server |
| `SSH_KNOWN_HOSTS` | Yes | SSH known_hosts entry (`ssh-keyscan <host>`) |
| `ENV_FILE` | No | Contents of the .env file to write on the server |

## Deploy Process

1. Checkout repo (for compose files and tagging)
2. SCP compose/env template files to server (if `compose_source_files` set)
3. Write `.env` from `ENV_FILE` secret to server (if `env_file_path` set)
4. SSH to server: `docker compose pull` + `up -d`
5. Run migrations + collectstatic
6. Health check (if URL provided)
7. Create git tag (if enabled)

No git repo needed on the server. All files are copied via SCP.

## Required Server Setup

1. Docker + Docker Compose v2 installed
2. Traefik running with external `web` network
3. One-time `docker login ghcr.io` with a classic PAT (`read:packages` scope)
4. SSH access for the deploy user (Ed25519 key in `authorized_keys`)

## Required GitHub Secrets (per consumer repo)

| Secret | Purpose |
|--------|---------|
| `SSH_PRIVATE_KEY` | Ed25519 SSH key for deployment server |
| `SSH_KNOWN_HOSTS` | SSH known_hosts entry for the server |
| `ENV_FILE` | Production .env file contents (optional) |

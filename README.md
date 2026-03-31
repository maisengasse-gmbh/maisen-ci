# maisen-ci

Shared GitHub Actions CI/CD workflows for Maisengasse projects.

## Available Workflows

| Workflow | Purpose |
|----------|---------|
| `django-lint-test.yml` | Ruff lint + pytest (PostgreSQL + Valkey) |
| `laravel-lint-test.yml` | Laravel Pint + PHPUnit (MySQL/PostgreSQL + Valkey) |
| `node-lint-test.yml` | npm lint + test (optional DB + cache) |
| `frontend-build.yml` | npm lint + build (any framework: Vue, React, Svelte, etc.) |
| `flutter-build.yml` | Flutter analyze + test + optional build |
| `docker-build-push.yml` | Docker build + push to ghcr.io |
| `deploy.yml` | SSH deploy with configurable migrations, post-deploy commands, backup |
| `vue-build.yml` | **Deprecated** — alias for frontend-build.yml |

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
    uses: maisengasse-gmbh/maisen-ci/.github/workflows/django-lint-test.yml@v2
    with:
      working_directory: path/to/django
      pytest_args: "--ds myapp.settings.test -x"
      project_name: myproject
      project_verbose_name: "My Project"

  frontend-build:
    uses: maisengasse-gmbh/maisen-ci/.github/workflows/frontend-build.yml@v2
    with:
      working_directory: path/to/frontend

  build-backend:
    if: (github.event_name == 'push' || github.event_name == 'workflow_dispatch') && github.ref == 'refs/heads/main'
    needs: [lint-test, frontend-build]
    uses: maisengasse-gmbh/maisen-ci/.github/workflows/docker-build-push.yml@v2
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
    uses: maisengasse-gmbh/maisen-ci/.github/workflows/deploy.yml@v2
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

### Laravel project

```yaml
  lint-test:
    uses: maisengasse-gmbh/maisen-ci/.github/workflows/laravel-lint-test.yml@v2
    with:
      working_directory: path/to/laravel
      php_version: "8.3"
      database: mysql
```

### Node.js project

```yaml
  lint-test:
    uses: maisengasse-gmbh/maisen-ci/.github/workflows/node-lint-test.yml@v2
    with:
      working_directory: path/to/node
      node_version: "22"
      test_command: "npm test"
```

### Frontend build (Vue / React / Svelte / etc.)

```yaml
  frontend-build:
    uses: maisengasse-gmbh/maisen-ci/.github/workflows/frontend-build.yml@v2
    with:
      working_directory: path/to/frontend
      build_command: "npm run build"
```

### Flutter project

```yaml
  flutter-build:
    uses: maisengasse-gmbh/maisen-ci/.github/workflows/flutter-build.yml@v2
    with:
      working_directory: path/to/flutter
      flutter_version: "3.24.0"
      build_target: apk
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
| `cache_url` | `redis://localhost:6379/0` | Cache/Valkey URL |
| `project_name` | `""` | Django PROJECT_NAME env var |
| `project_verbose_name` | `""` | Django PROJECT_VERBOSE_NAME env var |

### laravel-lint-test.yml

| Input | Default | Description |
|-------|---------|-------------|
| `php_version` | `8.3` | PHP version |
| `working_directory` | `.` | Path to Laravel project |
| `database` | `mysql` | Database engine (`mysql` or `pgsql`) |
| `mysql_version` | `8.0` | MySQL version (used when database is mysql) |
| `postgres_version` | `16` | PostgreSQL version (used when database is pgsql) |
| `cache_url` | `redis://localhost:6379/0` | Cache/Valkey URL |

### node-lint-test.yml

| Input | Default | Description |
|-------|---------|-------------|
| `node_version` | `22` | Node.js version |
| `working_directory` | `.` | Path to Node project |
| `lint_command` | `npm run lint` | Lint command |
| `test_command` | `npm test` | Test command |
| `with_database` | `false` | Spin up a PostgreSQL service |
| `with_cache` | `false` | Spin up a Valkey/Redis service |

### frontend-build.yml

| Input | Default | Description |
|-------|---------|-------------|
| `node_version` | `22` | Node.js version |
| `working_directory` | `.` | Path to frontend project |
| `lint_command` | `npm run lint` | Lint command |
| `build_command` | `npm run build` | Build command |

### vue-build.yml

Deprecated alias for `frontend-build.yml`. Accepts the same inputs. Use `frontend-build.yml` for new projects.

### flutter-build.yml

| Input | Default | Description |
|-------|---------|-------------|
| `flutter_version` | `stable` | Flutter SDK version or channel |
| `working_directory` | `.` | Path to Flutter project |
| `build_target` | `""` | Build target (`apk`, `ios`, `web`, etc.) — skipped if empty |

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
| `backend_container` | `backend` | Container name used for migration and post-deploy commands |
| `migration_command` | `""` | Command to run inside backend container after deploy (e.g. `python manage.py migrate --noinput`) — skipped if empty |
| `post_deploy_commands` | `""` | Additional commands to run inside backend container after migrations — skipped if empty |
| `backup_command` | `""` | Command to run on server before pulling new images (e.g. a DB dump script) — skipped if empty |
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
2. Run backup command on server (if `backup_command` set)
3. SCP compose/env template files to server (if `compose_source_files` set)
4. Write `.env` from `ENV_FILE` secret to server (if `env_file_path` set)
5. SSH to server: `docker compose pull` + `up -d`
6. Run migration command inside backend container (if `migration_command` set)
7. Run post-deploy commands inside backend container (if `post_deploy_commands` set)
8. Health check (if URL provided)
9. Create git tag (if enabled)

No git repo needed on the server. All files are copied via SCP.

## Version Pinning

Use major version tags to pin workflows:

| Tag | Description |
|-----|-------------|
| `@v1` | Last stable release before multi-stack refactoring |
| `@v2` | Generic deploy, new stack workflows, Valkey defaults |

Example:

```yaml
uses: maisengasse-gmbh/maisen-ci/.github/workflows/deploy.yml@v2
```

Breaking changes only happen on major version bumps. Consumer projects can revert to `@v1` with a one-line change if `@v2` causes issues.

## 1Password Environments

### Overview

`deploy.yml` supports optional 1Password Environments for secrets management. When `op_environment_id` is provided, the workflow uses the 1Password CLI to inject secrets into the `.env` file on the server — replacing the static `ENV_FILE` secret approach.

### Setup

1. **Install the `op` CLI** (for local development):
   ```bash
   brew install 1password-cli
   ```

2. **Create Environments in the 1Password Desktop App** — one per deployment stage per project (e.g. `my-project-dev`, `my-project-prod`).

3. **Create a Service Account** in 1Password with read access to the production Environment(s).

4. **Configure GitHub** for the consumer repo:
   - Repository variable: `OP_ENVIRONMENT_ID` — the ID of the 1Password Environment for production
   - Repository secret: `OP_SERVICE_ACCOUNT_TOKEN` — the service account token

### Local Development

Add a `.env.tpl` file to the repo with `op://` references:

```dotenv
DATABASE_URL=op://my-project-prod/database/url
SECRET_KEY=op://my-project-prod/django/secret-key
```

Run `make up` — the Makefile should auto-generate `.env` via:

```bash
op inject -i .env.tpl -o .env
```

### CI/CD

Pass `op_environment_id` and `OP_SERVICE_ACCOUNT_TOKEN` to the deploy workflow:

```yaml
deploy:
  uses: maisengasse-gmbh/maisen-ci/.github/workflows/deploy.yml@v2
  with:
    op_environment_id: ${{ vars.OP_ENVIRONMENT_ID }}
  secrets:
    SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
    SSH_KNOWN_HOSTS: ${{ secrets.SSH_KNOWN_HOSTS }}
    OP_SERVICE_ACCOUNT_TOKEN: ${{ secrets.OP_SERVICE_ACCOUNT_TOKEN }}
```

The workflow will run `op inject` on the server to produce the `.env` file from a committed `.env.tpl`.

### Migration from ENV_FILE

1. Add `.env.tpl` to the repo with `op://` references for all secrets.
2. Remove the `ENV_FILE` secret from GitHub.
3. Add `OP_SERVICE_ACCOUNT_TOKEN` secret and `OP_ENVIRONMENT_ID` variable to GitHub.
4. Update the workflow call as shown above (remove `ENV_FILE:`, add `OP_SERVICE_ACCOUNT_TOKEN:`).

### Rollback

To revert to the `ENV_FILE` approach:

1. Remove `op_environment_id` from the workflow `with:` block.
2. Re-add the `ENV_FILE` secret to GitHub.
3. Pass it in `secrets:` as `ENV_FILE: ${{ secrets.ENV_FILE }}`.

## Required Server Setup

1. Docker + Docker Compose v2 installed
2. Traefik running with external `web` network
3. One-time `docker login ghcr.io` with a classic PAT (`read:packages` scope)
4. SSH access for the deploy user (Ed25519 key in `authorized_keys`)

## SSH Deploy Key einrichten

Die CI-Pipeline verbindet sich per SSH zum Produktionsserver. Dafür braucht es ein Ed25519 Keypair.

### 1. Key erstellen (einmalig pro Projekt)

```bash
ssh-keygen -t ed25519 -f ~/.ssh/projektname-deploy -C "projektname-deploy" -N ""
```

### 2. Public Key auf dem Server hinterlegen

```bash
# Public Key anzeigen
cat ~/.ssh/projektname-deploy.pub

# Auf dem Server eintragen
ssh op@mgh3.mynet.at "echo 'INHALT_VON_PUB_KEY' >> ~/.ssh/authorized_keys"
```

Oder manuell: Per SSH auf den Server → `~/.ssh/authorized_keys` bearbeiten → Public Key am Ende einfügen.

### 3. GitHub Secrets setzen

Am einfachsten per CLI (vermeidet Copy-Paste-Fehler):

```bash
# Private Key direkt aus Datei setzen
gh secret set SSH_PRIVATE_KEY < ~/.ssh/projektname-deploy

# Known Hosts direkt vom Server holen und setzen
ssh-keyscan mgh3.mynet.at 2>/dev/null | gh secret set SSH_KNOWN_HOSTS
```

Alternativ im Browser: GitHub Repo → Settings → Secrets and variables → Actions → New repository secret.

### 4. Testen

```bash
gh workflow run ci.yml
```

Bei SSH-Fehlern im Deploy-Step:
- `error in libcrypto` → Key ist beschädigt, nochmal mit `gh secret set` aus Datei setzen
- `Permission denied (publickey)` → Public Key nicht in `authorized_keys` auf dem Server
- `Host key verification failed` → `SSH_KNOWN_HOSTS` Secret fehlt oder ist veraltet, nochmal mit `ssh-keyscan` setzen

## GitHub Secrets & Variables (pro Consumer-Repo)

### Secrets

| Secret | Required | Beschreibung |
|--------|----------|-------------|
| `SSH_PRIVATE_KEY` | Ja | Ed25519 Private Key (`gh secret set SSH_PRIVATE_KEY < ~/.ssh/key`) |
| `SSH_KNOWN_HOSTS` | Ja | Host Keys (`ssh-keyscan host \| gh secret set SSH_KNOWN_HOSTS`) |
| `OP_SERVICE_ACCOUNT_TOKEN` | Ja* | 1Password Service Account Token |
| `ENV_FILE` | Nein | Fallback: komplette .env als Plaintext (deprecated, nutze 1Password) |

*Wenn 1Password Environments genutzt werden.

### Variables

| Variable | Required | Beschreibung |
|----------|----------|-------------|
| `OP_ENVIRONMENT_ID` | Ja* | 1Password Prod-Environment ID (nicht geheim, nur ein Identifier) |

*Wenn 1Password Environments genutzt werden.

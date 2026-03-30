---
name: maisengasse-conventions
description: Use when working in any Maisengasse GmbH project (Django, Laravel, Vue) - covers shared CI/CD, Docker, coding conventions, and deployment patterns across zakb-cloud, intersport-budget, my-wto and other projects
---

# Maisengasse Project Conventions

## Overview

Shared conventions for all Maisengasse GmbH projects. These override defaults when working in any Maisengasse repo.

## CI/CD (maisen-ci)

All projects use reusable GitHub Actions workflows from `maisengasse-gmbh/maisen-ci`:

| Workflow | Purpose |
|----------|---------|
| `django-lint-test.yml` | Ruff lint + pytest (PostgreSQL 16 + Redis) |
| `vue-build.yml` | ESLint + Vite build |
| `docker-build-push.yml` | Build + push to ghcr.io |
| `deploy.yml` | SSH deploy, backup, migrations, health check, git tag |

### CI Config Pattern

```yaml
name: CI/CD
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  lint-test:
    uses: maisengasse-gmbh/maisen-ci/.github/workflows/django-lint-test.yml@main
  build:
    needs: [lint-test]
    uses: maisengasse-gmbh/maisen-ci/.github/workflows/docker-build-push.yml@main
  deploy:
    needs: [build]
    uses: maisengasse-gmbh/maisen-ci/.github/workflows/deploy.yml@main
```

## Docker Structure

```
docker/
  docker-compose.base.yml   # Shared service definitions
  docker-compose.dev.yml    # Dev overrides (local builds, ports)
  docker-compose.prod.yml   # Prod overrides (ghcr.io images)
Dockerfile                  # At project root
```

### Compose Pattern (base + override)

- **base.yml**: Services, networks, volumes, Traefik labels, env_file
- **dev.yml**: Local build context, exposed ports, debug volumes
- **prod.yml**: Remote images from ghcr.io, no exposed ports

### Standard Services

| Service | Image | Notes |
|---------|-------|-------|
| `backend` | Project Dockerfile | App server on port 8082 |
| `db` | postgres:16 | Internal only |
| `redis` | redis:5.0-alpine | Broker + cache |
| `backend-worker` | From Dockerfile | Celery (Django) / Queue (Laravel) |
| `nginx` | nginx:stable | Static/media serving |

### Networking

- External `web` network for Traefik (bridge, shared across projects)
- Internal `backend-network` per project
- Traefik handles HTTPS (Let's Encrypt) + redirect HTTP->HTTPS + www->non-www

### Environment Variables

- `.env` file at project root (git-ignored)
- Required: `PROJECT_NAME`, `PROJECT_DOMAIN`, `PROJECT_DEV_DOMAIN`, `DATABASE_URL`
- Production: Written from GitHub Secret `ENV_FILE` during deploy
- Development: From 1Password

## Deployment

- **Server**: mgh3.mynet.at, SSH user `op`
- **Deploy path**: `/home/op/projects/{project-name}`
- **Backups**: `/data/home/op/projects/{project-name}/backups`
- **Reverse proxy**: Traefik (external `web` network)
- **Registry**: ghcr.io (GitHub Container Registry)
- **Deploy tags**: `deploy/YYYY-MM-DD-<sha>` created on success

## Code Style

### Python (Django Projects)

- **Formatter**: Black (line-length=88) or Ruff format
- **Linter**: Ruff + Flake8
- **Quotes**: Double quotes
- **Config**: `pyproject.toml`
- **Pre-commit**: From `maisen-ci/shared/.pre-commit-config.yaml`

### Git Commits

```
feat: Kurzbeschreibung
fix: Kurzbeschreibung
refactor: Kurzbeschreibung
docs: Kurzbeschreibung
test: Kurzbeschreibung
chore: Kurzbeschreibung
```

- Kein Scope in Klammern (kein `feat(auth):`)
- Deutsch oder Englisch (konsistent pro Projekt)
- Enforced via `commit-check` pre-commit hook

## Django Conventions

### Views: Function-Based Only

```python
@permission_required("app.can_view_thing")
def my_view(request):
    items = Model.objects.managed_by(request.user)
    return render(request, "app/template.html", locals())
```

- Keine Class-Based Views
- `locals()` als Context
- Decorator-Reihenfolge: `@permission_required` -> `@ownership_required` -> `@cache_page`

### URLs: re_path mit Regex

```python
from django.urls import re_path as url, include

urlpatterns = [
    url(r"^item/$", views.item_list, name="item_list"),
    url(r"^item/([\d]+)/$", views.item_detail, name="item_detail"),
]
```

### Models

- `verbose_name` / `verbose_name_plural` auf Deutsch (`gettext_lazy as _`)
- `on_delete=models.PROTECT` als Standard
- Custom QuerySets: `managed_by(user)`, `owned_by(user)`
- Mixins: `OwnerAware`, `DublinCore`, `AddressAware`

### Forms: Factory-Pattern

```python
def myform_factory(user):
    choices = Model.objects.managed_by(user)
    class MyForm(forms.ModelForm):
        prefix = "mf"
        class Meta:
            model = MyModel
            fields = ("field1", "field2")
    return MyForm
```

### Admin: django-unfold

- Immer `UnfoldAdmin` statt `ModelAdmin`
- Inlines: `UnfoldTabularInline` / `UnfoldStackedInline`

### Tests

- Verzeichnis: `app/tests/test_models.py`, `test_views.py`, `test_api.py`
- `setUpTestData` (classmethod) + `setUp`
- Utilities in `core/testing.py`

## Makefile Targets

Jedes Projekt hat ein Makefile mit mindestens:

```makefile
make help       # Verfuegbare Befehle
make build      # Docker images bauen
make up         # Container starten
make down       # Container stoppen
make migrate    # Migrationen ausfuehren
make test       # Tests laufen lassen
make logs       # Container logs
make shell      # Shell im Backend
```

## Project-Specific Notes

| Projekt | Stack | Besonderheiten |
|---------|-------|----------------|
| zakb-cloud | Django + Bootstrap4 | Celery, 2FA, Unfold Admin |
| intersport-budget | Django + Vue 3 | django-ninja API, JWT Auth, dual Docker images |
| my-wto | Laravel + Tailwind | GitLab CI (nicht GitHub), OCR, PDF |

# shared-workflows

Reusable GitHub Actions workflows pour les monorepos Nx. Centralise la CI et le pipeline Docker/GitOps pour eviter la duplication entre projets.

Chaque projet consommateur n'a besoin que d'**un seul fichier** (`.github/workflows/main.yml`) qui reference ce repo. Toute modification ici est automatiquement repercutee sur tous les projets.

## Structure du repo

```
.github/workflows/
  nx-ci.yml       Reusable workflow : CI Nx (lint, test, build...)
  docker.yml      Reusable workflow : Docker build + deploy GitOps

templates/
  main.yml        Le seul fichier a copier dans chaque projet

examples/
  solar-monitor/main.yml
  portfolio/main.yml
```

## Mise en place sur un nouveau projet

### 1. Copier le template

Copier `templates/main.yml` dans votre projet :

```
mon-projet/
  .github/
    workflows/
      main.yml    <-- le seul fichier necessaire
```

### 2. Remplacer `<owner>`

Remplacer `<owner>` par votre username ou organisation GitHub :

```yaml
# Avant
uses: <owner>/shared-workflows/.github/workflows/nx-ci.yml@main

# Apres
uses: mon-org/shared-workflows/.github/workflows/nx-ci.yml@main
```

### 3. Ajuster les inputs

Toute la configuration est inline dans le YAML. Editez les valeurs `with:` pour correspondre a votre projet :

```yaml
jobs:
  ci:
    uses: mon-org/shared-workflows/.github/workflows/nx-ci.yml@main
    with:
      node_version: "24"
      nx_targets: "lint test typecheck build"
      prisma: true
      env_file: "DATABASE_URL=postgresql://user:pass@localhost:5432/dummy"
    secrets: inherit

  docker:
    if: github.ref == 'refs/heads/main' || github.event_name == 'release' || github.event_name == 'workflow_dispatch'
    needs: ci
    uses: mon-org/shared-workflows/.github/workflows/docker.yml@main
    with:
      images: '[{"name": "api", "dockerfile": "apps/api/Dockerfile"}]'
    secrets: inherit
```

Si votre projet n'a pas besoin de Docker, supprimez simplement le job `docker`.

### 4. Configurer les secrets et variables GitHub

Sur le repo GitHub de votre projet, ajouter :

**Secrets** (Settings > Secrets and variables > Actions > Secrets) :

| Secret | Description | Requis |
|--------|-------------|--------|
| `NX_CLOUD_ACCESS_TOKEN` | Token Nx Cloud pour la distribution de taches | Non |
| `GITOPS_PAT` | Personal Access Token avec acces au repo GitOps | Oui (deploy) |
| `SLACK_WEBHOOK_URL` | URL du webhook Slack pour les notifications | Non |

**Variables** (Settings > Secrets and variables > Actions > Variables) :

| Variable | Description | Requis |
|----------|-------------|--------|
| `DOCKER_MANUAL_ONLY` | Si `true`, les builds Docker ne se lancent qu'en workflow_dispatch | Non |

## Reference des inputs

### `nx-ci.yml`

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `node_version` | string | `"24"` | Version de Node.js (LTS active) |
| `nx_targets` | string | `"lint test build"` | Targets Nx a executer (separes par des espaces) |
| `prisma` | boolean | `false` | Lancer `prisma generate` avant les targets |
| `env_file` | string | `""` | Contenu a ecrire dans `.env` pour les tests CI |
| `runner` | string | `"ubuntu-latest"` | Label du runner |
| `affected` | boolean | `false` | Sur les pull requests, seulement les projets affectes (`nx affected`) ; tout le monorepo sinon |
| `knip` | boolean | `false` | Lancer `knip` (fichiers, dependances et exports inutilises) apres les targets |
| `gitleaks` | boolean | `false` | Chercher des secrets dans l'historique git avec gitleaks (binaire epingle et verifie) |

### `docker.yml`

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `images` | string | *requis* | JSON array des images Docker a construire |
| `gitops_repo` | string | `"simoncamilotti/infra"` | Repo GitOps mis a jour par les jobs de deploiement |
| `push` | boolean | `true` | Pousser les images sur GHCR ; `false` verifie seulement qu'elles se construisent |
| `deploy` | boolean | `true` | Mettre a jour les overlays GitOps (dev sur main, prod sur une release). Necessite `push` |

Le job appelant doit accorder `packages: write` quand `push` vaut `true`.

### `nx-e2e.yml`

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `node_version` | string | `"24"` | Version de Node.js |
| `runner` | string | `"ubuntu-latest"` | Label du runner |
| `playwright_image` | string | `"mcr.microsoft.com/playwright:v1.63.0-noble"` | Image du job : dependances systeme et navigateurs. A garder sur la version de `@playwright/test` |
| `browsers_command` | string | `""` | Resynchronise les navigateurs avec la version Playwright du code (ex: `pnpm --filter @repo/web-e2e exec playwright install chromium`) |
| `affected` | boolean | `false` | Sur les pull requests, seulement les projets affectes |
| `services_host` | string | `""` | Hote des services : vide sur les runners GitHub (nom du service), `localhost` sur ARC en `containerMode: kubernetes` |

Format de `images` :

```json
[
  { "name": "api", "dockerfile": "apps/api/Dockerfile" },
  { "name": "web", "dockerfile": "apps/web/Dockerfile" }
]
```

| Champ | Description |
|-------|-------------|
| `name` | Nom de l'image (suffixe dans `ghcr.io/<owner>/<repo>/<name>`) |
| `dockerfile` | Chemin vers le Dockerfile |

## Workflows en detail

### `nx-ci.yml` — CI

**Etapes** :

1. Checkout du code (avec historique complet pour Nx)
2. Setup Node.js (version depuis les inputs)
3. Detection du package manager par le lockfile : `pnpm-lock.yaml` -> pnpm, `package-lock.json` -> npm
4. Setup pnpm si besoin (version lue dans le champ `packageManager` du `package.json`)
5. `npm ci` ou `pnpm install --frozen-lockfile`
6. Creation du `.env` (si `env_file` fourni)
7. `prisma generate` (si `prisma: true`)
8. `nx run-many -t <nx_targets>`
9. `nx fix-ci` (self-healing, tourne meme si les targets echouent)

Les commandes passent par `npx` ou `pnpm exec` selon le package manager detecte.

### `nx-e2e.yml` — E2E full-stack

Lance les targets Nx `e2e` dans l'image Playwright, avec deux services :

| Service | Variable fournie aux tests |
|---------|---------------------------|
| PostgreSQL 18 | `E2E_DATABASE_URL` |
| Mailpit | `E2E_SMTP_URL`, `E2E_MAILPIT_URL` |

Les rapports Playwright (`**/test-output/results`, `**/playwright-report`) sont publies en artefact en cas d'echec.

### `docker.yml` — Build & Deploy

**Jobs** :

```
build         Build & push de chaque image en parallele (matrice)
  |
  +---> deploy-dev    Push sur main : met a jour l'overlay dev du repo GitOps
  |                   Tag: <branch>-<run_id>
  |
  +---> deploy-prod   Release : met a jour l'overlay prod du repo GitOps
                       Tag: <version> (sans prefixe v)
```

**Tags Docker generes** :

| Evenement | Tags |
|-----------|------|
| Push sur main | `latest`, `main`, `main-<run_id>`, `sha-<sha>` |
| Release `v1.2.3` | `1.2.3`, `1.2`, `sha-<sha>` |

**Structure GitOps attendue** :

```
gitops-repo/
  apps/
    <nom-du-repo>/
      overlays/
        dev/
          kustomization.yaml
        prod/
          kustomization.yaml
```

## Exemples

### solar-monitor

Node 24, Prisma, typecheck, 2 images (api + web) — voir [`examples/solar-monitor/main.yml`](examples/solar-monitor/main.yml)

### portfolio

Node 20, Prisma, 3 images (api + studio + client) — voir [`examples/portfolio/main.yml`](examples/portfolio/main.yml)

## Personnalisation des triggers

Les declencheurs (`on:`) sont definis dans le `main.yml` de chaque projet. Modifiez-les directement :

```yaml
on:
  push:
    branches:
      - main
      - develop        # ajouter une branche
  pull_request:
  release:
    types: [published]
  workflow_dispatch:
```

## Prerequis

- Ce repo doit etre **public** ou dans la meme organisation que les repos appelants (contrainte GitHub pour les reusable workflows).
- Les projets appelants utilisent **npm** ou **pnpm**, avec leur lockfile a la racine. Avec pnpm, le `package.json` racine declare `packageManager` (ex: `"pnpm@10.x.y"`).
- Les Dockerfiles doivent exister aux chemins specifies dans le input `images`.

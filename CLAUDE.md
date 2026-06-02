# CLAUDE.md

Guidance for Claude Code and other AI agents working in this repository.

## Project overview

This repository provides generic Docker base images for Laravel applications, published as `axazaracoredev/laravel-docker` on Docker Hub and `ghcr.io/axazara/laravel-docker` on GitHub Container Registry. Each top-level directory (`8.0`–`8.4`) contains a self-contained `Dockerfile` built on the official `php:<version>-alpine` base. The images are consumed by Laravel projects running their CI/CD pipelines (especially GitLab CI) or containerized deployments.

## Tech stack

- Base OS: Alpine Linux (via `php:<version>-alpine` official images)
- PHP versions: 8.0, 8.1, 8.2, 8.3, 8.4 (one Dockerfile per version directory)
- Composer 2 (installed via `docker-php-extension-installer`)
- Node.js + npm + Yarn (Alpine packages)
- PHP extensions: redis, imagick, xdebug, bcmath, calendar, exif, gd, intl, pdo_mysql, pdo_pgsql, pcntl, soap, zip, sockets, gettext, shmop, sysvmsg, swoole (8.2+)
- PHP_CodeSniffer (globally installed via Composer in each image)
- Chromium + ChromeDriver (for browser testing)
- Database clients: mysql-client, postgresql-libs, sqlite

## Getting started

```bash
# Build a specific PHP version image locally
docker build -t laravel-docker:8.4 -f 8.4/Dockerfile .

# Build all versions
for v in 8.0 8.1 8.2 8.3 8.4; do
  docker build -t laravel-docker:$v -f $v/Dockerfile .
done
```

## Common commands

| Task | Command |
|---|---|
| Build image (single version) | `docker build -t laravel-docker:8.4 -f 8.4/Dockerfile .` |
| Run a container interactively | `docker run --rm -it laravel-docker:8.4 bash` |
| Run PHPUnit in CI (via image) | `phpunit --coverage-text --colors=never` |
| Check code style in CI | `phpcs --standard=PSR2 --extensions=php app` |

## Architecture

The repository is structured as one directory per supported PHP version, each containing an independent `Dockerfile`:

- `8.0/Dockerfile` — PHP 8.0 Alpine image
- `8.1/Dockerfile` — PHP 8.1 Alpine image
- `8.2/Dockerfile` — PHP 8.2 Alpine image
- `8.3/Dockerfile` — PHP 8.3 Alpine image (pinned to `8.3.9-alpine`)
- `8.4/Dockerfile` — PHP 8.4 Alpine image (latest stable, tagged `stable` and `latest`)
- `.github/workflows/ci.yml` — builds all five versions in a matrix on every push/PR, pushes to GHCR on `main`
- `.github/workflows/packages.yml` — legacy workflow that also pushes to Docker Hub
- `gitlab/.gitlab-ci.tests.yml` — reusable GitLab CI snippet for running PHPUnit + PSR2 codestyle in a Laravel project
- `gitlab/.gitlab-ci.deployments.yml` — reusable GitLab CI snippet covering build, test, and SSH-based deployment stages for Laravel projects

CI runs on GitHub Actions with a matrix strategy (`fail-fast: false`) so a broken single-version build does not block others.

## Conventions

- Each PHP version directory is fully self-contained; changes to extensions or system packages must be applied to each `Dockerfile` individually.
- PHP_CodeSniffer is always globally installed at image build time so `phpcs` is available on `$PATH` without a project-level install.
- The `WORKDIR` is always `/var/www` — mount Laravel project source there when running containers.
- `./vendor/bin`, `/composer/vendor/bin`, and `/root/.composer/vendor/bin` are all on `$PATH` by default.
- The `8.0/Dockerfile` is named `8.0` but currently based on `php:8.5.6-alpine` (appears to be a branch mislabel); verify before relying on the exact base version.
- Images are pushed only on commits to `main`; PRs trigger build-only (no push).

## Git Conventions

### 1. Branch names

Enforced regex (`branch_name_pattern`):
```
^(feature|fix|hotfix|chore|docs|refactor|test|ci|perf|build|style)/[a-z0-9._-]+$
```

- Lowercase only, kebab-case after the prefix, **max 50 characters** total.
- Use the full word `feature/` — **never** `feat/` (the short `feat` form is only for commit message types).
- Include the ticket id when relevant: `feature/AXA-123-add-stripe` (the ticket id is lowercased to satisfy the pattern — e.g. `feature/axa-123-add-stripe`).
- **Never** use a `claude/` prefix or any prefix outside the allowed set.
- `main`, `release`, `staging` are permanent protected branches — never push to them directly.
- If a branch is misnamed, rename it before pushing: `git branch -m <old> <new>`.

### 2. Commit messages
Enforced regex (`commit_message_pattern`), applied to **every** commit:
```
^(feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert)(\([^)]+\))?!?: .+
```
- Lowercase type, optional scope in parens, optional `!` for breaking changes, subject after `: `.
- Subject starts with a lowercase letter and has no trailing period.
- Examples: `feat(checkout): add Apple Pay support`, `fix(api): handle expired tokens`, `chore(deps): bump axios from 1.7.2 to 1.15.2`, `refactor!: drop Node 18 support`.
- Do not rewrite Dependabot commits — `chore(deps): bump X from a to b` is already enforced via `.github/dependabot.yml`.

### 3. Files that are always rejected
Never stage or commit:
- `.env`, `.env.*` (only `.env.example` and `.env.sample` are allowed), `**/.env`, `**/.env.*`
- Private keys: `**/id_rsa{,.pub}`, `**/id_dsa`, `**/id_ecdsa`, `**/id_ed25519`, `**/.ssh/id_*`
- Credentials: `**/.aws/credentials`, `**/credentials.json`, `**/service-account.json`, `**/firebase-adminsdk-*.json`, `**/secrets.{yml,yaml}`
- Extensions: `*.pem`, `*.key`, `*.p12`, `*.pfx`, `*.jks`, `*.keystore`, `*.ppk`, `*.asc`, `*.gpg`
- Any file larger than 100 MB (use git LFS)
If a secret is needed, use `.env.example` for env vars and an external secret manager for credentials.

### Pull requests targeting `main`, `release`, `staging`
All three are protected — a PR is required (direct push blocked):
- 1 approval, all conversations resolved, **squash or rebase merge only** (linear history enforced — no merge commits).
- Commits must be GPG- or SSH-signed. Signing is required for `main` (`required-signatures-main` ruleset).
- The PR **title** becomes the squash commit message and must match the commit-message regex above (enforced on all three branches).

**Required workflows run on PRs whose base is `main` only** (not `release`/`staging`): `Branch naming convention`, `PR title — Conventional Commits`, and `PR size labeler`.
If a check shows `Waiting for workflow to run` for over a minute, the third-party action is likely missing from the enterprise allowlist.

When the branch-naming or PR-title check fails, the baseline bot auto-posts rename/title suggestions, following the enforced regex patterns.
If the bot's suggestions are incorrect, edit the PR title or branch name to match the required format.

### Pre-push checklist
Before running `git push`:
1. Branch name matches the regex.
2. Every commit in `origin/main..HEAD` matches the commit pattern (`git log --format=%s origin/main..HEAD`).
3. No staged file is in the blocked paths/extensions list.
4. Commits are signed if the target is `main`.

If any check fails, fix it locally rather than letting the server reject the push.

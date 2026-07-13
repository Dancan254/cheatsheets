# GitHub Actions

Reach for this when writing or debugging `.github/workflows/*.yml`.

## Anatomy

```yaml
name: CI
on:                          # triggers
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:         # manual "Run workflow" button

jobs:                        # jobs run in parallel by default
  test:                      # job id
    runs-on: ubuntu-latest   # runner
    steps:
      - uses: actions/checkout@v4    # pinned action version, never @main
      - name: Run tests
        run: ./mvnw test
```

`workflow → jobs → steps`. Steps run sequentially in a job; jobs run in parallel unless chained with `needs`.

## Java CI with Maven cache (your default)

```yaml
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '25'
          cache: maven                 # caches ~/.m2 between runs automatically
      - run: ./mvnw --batch-mode verify

  build:
    needs: test                        # only after test passes
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: temurin, java-version: '25', cache: maven }
      - run: ./mvnw --batch-mode package -DskipTests
      - uses: actions/upload-artifact@v4
        with: { name: jar, path: target/*.jar }
```

Separate jobs: `test → build → deploy`, never one monolithic job. Chain with `needs`.

## Job dependencies & conditions

```yaml
jobs:
  deploy:
    needs: [test, build]                          # wait for both
    if: github.ref == 'refs/heads/main'           # only on main
    runs-on: ubuntu-latest
    steps: [...]
```

Common `if` expressions:
```yaml
if: github.event_name == 'push'
if: startsWith(github.ref, 'refs/tags/')
if: success()          # default; also failure(), always(), cancelled()
```

## Matrix builds

```yaml
strategy:
  fail-fast: false                 # don't cancel siblings when one fails
  matrix:
    java: ['21', '25']
    os: [ubuntu-latest, windows-latest]
runs-on: ${{ matrix.os }}
steps:
  - uses: actions/setup-java@v4
    with: { distribution: temurin, java-version: ${{ matrix.java }} }
```

## Secrets & env

```yaml
env:                                    # workflow/job/step scoped
  LOG_LEVEL: INFO
steps:
  - run: ./deploy.sh
    env:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}   # from GitHub Secrets
      API_TOKEN: ${{ secrets.API_TOKEN }}
```

Secrets: repo/org/environment scoped. Never echo them; GitHub masks known secret values in logs but not derived ones.

## Manual caching (non-Maven)

```yaml
- uses: actions/cache@v4
  with:
    path: ~/.gradle/caches
    key: gradle-${{ hashFiles('**/*.gradle*') }}
    restore-keys: gradle-
```

## Docker build & push (GHCR)

```yaml
- uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}
- uses: docker/build-push-action@v6
  with:
    push: true
    tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
```

## Reusable workflows

```yaml
# .github/workflows/reusable-test.yml
on:
  workflow_call:
    inputs:
      java-version: { required: true, type: string }
# caller:
jobs:
  test:
    uses: ./.github/workflows/reusable-test.yml
    with: { java-version: '25' }
```

## Useful contexts

| Context | Example |
|---|---|
| `github.sha` | commit SHA (tag images with it) |
| `github.ref` | `refs/heads/main`, `refs/tags/v1.0` |
| `github.actor` | who triggered it |
| `github.event_name` | `push`, `pull_request`, ... |
| `runner.os` | `Linux` / `Windows` / `macOS` |

## Gotchas / things I always forget

- **Pin action versions** (`@v4`, or a SHA). `@main` = someone else's changes silently break your CI (and it's a supply-chain risk).
- Jobs run on **separate runners** - no shared filesystem. Pass data between jobs via `upload-artifact`/`download-artifact`, not disk.
- Each `run:` step is a fresh shell. `cd` or env exports don't carry to the next step. Use `$GITHUB_ENV`/`$GITHUB_PATH` to persist.
- `needs:` creates ordering; without it jobs run in parallel and your "deploy" can start before "test".
- `secrets` are **not** available to workflows triggered by `pull_request` from forks (by design) - plan CI accordingly.
- `fail-fast: true` (matrix default) cancels all siblings on first failure. Set `false` when you want the full matrix result.
- `GITHUB_TOKEN` is auto-provided and scoped to the repo - use it for GHCR/PR comments before minting a PAT.
- `cache: maven` in `setup-java` replaces a manual `actions/cache` for `~/.m2` - don't do both.

## Quick reference

| Task | Snippet |
|---|---|
| Checkout | `uses: actions/checkout@v4` |
| Java + m2 cache | `setup-java@v4` with `cache: maven` |
| Order jobs | `needs: [test]` |
| Only on main | `if: github.ref == 'refs/heads/main'` |
| Pass files between jobs | `upload-artifact` / `download-artifact` |
| Persist env across steps | `echo "K=V" >> $GITHUB_ENV` |
| Manual trigger | `on: workflow_dispatch` |

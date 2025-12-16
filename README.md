# Elm CI/CD Guide: Caching ELM_HOME

> **Using elm-tooling?** Check out the [elm-tooling CI setup guide](https://elm-tooling.github.io/elm-tooling-cli/ci/) instead.

This guide shows how to properly cache Elm dependencies in CI/CD pipelines. By caching `ELM_HOME`, you can:

- **Avoid network requests** to package.elm-lang.org and github.com on most builds
- **Skip downloading and building** packages you've already fetched
- **Eliminate transient network issues** that can cause flaky builds

## How It Works

Elm stores all downloaded packages and their compiled artifacts in a directory called `ELM_HOME` (defaults to `~/.elm`). This cache is:

- **Shared across all projects** on the same machine
- **Safe to cache** between CI runs

The downloaded artifacts are immutable — each package version has a specific hash that Elm verifies. `ELM_HOME` also contains compiled artifacts (`.elmi` and `.elmo` files) for packages, and these are deterministic as well.

By saving and restoring `ELM_HOME` between builds, you only need network access when you add a new dependency — not on every build. This is simpler, faster, and more reliable than setting up a mirror of package.elm-lang.org.

Since package contents never change for a given version, **the cache is perfectly safe to keep indefinitely.**

---

## Cache Key Strategy

**Use a hash of `elm.json` as your cache key**, with a fallback key where supported.

**Why not a simple shared key like `elm`?** Most CI systems only save the cache when the key doesn't already exist:

1. First build saves the cache ✓
2. You add a dependency → cache is **not updated** (key already exists)
3. Every subsequent build re-downloads that dependency

With hash-based keys + fallback:

1. When `elm.json` changes, a new cache key is generated
2. The previous cache is restored via fallback/prefix matching
3. Only the new dependencies are downloaded
4. The updated cache is saved under the new key

**How fallback keys work:** Most CI systems let you specify a prefix (like `elm-`). When no exact match exists, the system restores the most recent cache whose key starts with that prefix — giving you a partial cache to build on.

See the CI-specific examples below for exact syntax.

---

## CI Providers

- [GitHub Actions](#github-actions)
- [GitLab CI](#gitlab-ci)
- [CircleCI](#circleci)
- [Azure Pipelines](#azure-pipelines)
- [Bitbucket Pipelines](#bitbucket-pipelines)
- [Travis CI](#travis-ci)
- [AWS CodeBuild](#aws-codebuild)

---

### GitHub Actions

```yaml
name: Elm CI

on: [push, pull_request]

env:
  ELM_HOME: ${{ github.workspace }}/.elm

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Install Elm
        run: npm install -g elm

      - name: Cache ELM_HOME
        uses: actions/cache@v4
        with:
          path: ${{ env.ELM_HOME }}
          key: elm-${{ hashFiles('elm.json') }}
          restore-keys: |
            elm-

      - name: Build
        run: elm make src/Main.elm --optimize
```

With elm-review (or other tools with their own `elm.json`):

```yaml
- name: Cache ELM_HOME
  uses: actions/cache@v4
  with:
    path: ${{ env.ELM_HOME }}
    key: elm-${{ hashFiles('elm.json', 'review/elm.json') }}
    restore-keys: |
      elm-
```

> **Tip:** Add all `elm.json` files to `hashFiles()` — this ensures the cache updates when any of them change.

---

### GitLab CI

```yaml
stages:
  - build

variables:
  ELM_HOME: $CI_PROJECT_DIR/.elm

cache:
  key:
    files:
      - elm.json
  paths:
    - .elm

build:
  stage: build
  image: node:20
  before_script:
    - npm install -g elm
  script:
    - elm make src/Main.elm --optimize
```

With elm-review (or other tools with their own `elm.json`):

```yaml
cache:
  key:
    files:
      - elm.json
      - review/elm.json
  paths:
    - .elm
```

> **Note:** The cache path `.elm` is relative to the project directory and matches `ELM_HOME`.

#### Advanced: Fallback Keys with Branch Isolation (GitLab 16.0+)

The basic setup above causes a full re-download whenever `elm.json` changes. For larger projects, you can use `fallback_keys` combined with a write-only cache job to get incremental updates (and use the partial cache from earlier builds):

```yaml
stages:
  - build

variables:
  ELM_HOME: $CI_PROJECT_DIR/.elm

build:
  stage: build
  image: node:20
  before_script:
    - npm install -g elm
  script:
    - elm make src/Main.elm --optimize
  cache:
    - key:
        files:
          - elm.json
      fallback_keys:
        - elm-$CI_COMMIT_REF_SLUG
        - elm-main
      paths:
        - .elm

elm-cache-push:
  stage: build
  image: node:20
  script:
    - echo "Updating elm cache for branch $CI_COMMIT_REF_SLUG"
  cache:
    - key: elm-$CI_COMMIT_REF_SLUG
      paths:
        - .elm
      policy: push
  rules:
    - changes:
        - elm.json
  needs:
    - build
```

**How it works:**

1. **Primary key** — Hash of `elm.json`. When dependencies haven't changed, you get an exact cache hit.

2. **Fallback keys** — When `elm.json` changes (no exact match), GitLab tries `elm-$CI_COMMIT_REF_SLUG` (branch-specific), then `elm-main` (shared baseline). This restores the previous cache so only new packages are downloaded.

3. **Write-only cache job** — The `elm-cache-push` job uses `policy: push` to write (not read) the cache under the branch-specific key. It only runs when `elm.json` changes, keeping the fallback cache fresh.

**Benefits:**

- **Incremental updates** — Adding one dependency doesn't re-download everything
- **Branch isolation** — Experimental dependency changes on feature branches don't affect other branches
- **Shared baseline** — New branches start with the `main` branch cache
- **Minimal overhead** — The cache-push job only runs when `elm.json` actually changes

---

### CircleCI

```yaml
version: 2.1

jobs:
  build:
    docker:
      - image: cimg/node:20.0
    working_directory: ~/project
    environment:
      ELM_HOME: ~/project/.elm
    steps:
      - checkout
      - restore_cache:
          keys:
            - elm-{{ checksum "elm.json" }}
            - elm-
      - run: npm install -g elm
      - run: elm make src/Main.elm --optimize
      - save_cache:
          key: elm-{{ checksum "elm.json" }}
          paths:
            - ~/project/.elm
```

With elm-review (or other tools with their own `elm.json`):

```yaml
- restore_cache:
    keys:
      - elm-{{ checksum "elm.json" }}-{{ checksum "review/elm.json" }}
      - elm-
- save_cache:
    key: elm-{{ checksum "elm.json" }}-{{ checksum "review/elm.json" }}
    paths:
      - ~/project/.elm
```

> **Note:** The second key (`elm-`) acts as a fallback, restoring the most recent cache with that prefix.

---

### Azure Pipelines

```yaml
trigger:
  - main

variables:
  ELM_HOME: $(Pipeline.Workspace)/.elm

pool:
  vmImage: ubuntu-latest

steps:
  - task: Cache@2
    inputs:
      key: '"elm" | elm.json'
      restoreKeys: |
        "elm"
      path: $(ELM_HOME)
    displayName: Cache ELM_HOME

  - script: npm install -g elm
    displayName: Install Elm

  - script: elm make src/Main.elm --optimize
    displayName: Build
```

With elm-review (or other tools with their own `elm.json`):

```yaml
- task: Cache@2
  inputs:
    key: '"elm" | elm.json | review/elm.json'
    restoreKeys: |
      "elm"
    path: $(ELM_HOME)
  displayName: Cache ELM_HOME
```

> **Note:** Wrap static strings in quotes to prevent them being interpreted as file paths.

---

### Bitbucket Pipelines

```yaml
definitions:
  caches:
    elm:
      key:
        files:
          - elm.json
      path: .elm

pipelines:
  default:
    - step:
        name: Build
        caches:
          - elm
        script:
          - export ELM_HOME=$BITBUCKET_CLONE_DIR/.elm
          - npm install -g elm
          - elm make src/Main.elm --optimize
```

With elm-review (or other tools with their own `elm.json`):

```yaml
definitions:
  caches:
    elm:
      key:
        files:
          - elm.json
          - review/elm.json
      path: .elm
```

> **Note:** Bitbucket cache definitions don't support variables, so ensure the cache path (`.elm`) matches where `ELM_HOME` points. Bitbucket Pipelines does not support fallback keys, so a full re-download occurs when dependencies change.

---

### Travis CI

```yaml
language: node_js
node_js:
  - "20"

env:
  global:
    - ELM_HOME=$HOME/.elm

cache:
  directories:
    - $HOME/.elm

install:
  - npm install -g elm

script:
  - elm make src/Main.elm --optimize
```

With elm-review (or other tools with their own `elm.json`):

```yaml
cache:
  directories:
    - $HOME/.elm
```

> **Note:** Travis CI uses path-based caching without hash keys. There is one cache per branch — if a branch doesn't have its own cache, Travis CI fetches the default branch's cache. The cache updates automatically when the directory contents change.

---

### AWS CodeBuild

```yaml
version: 0.2

env:
  variables:
    ELM_HOME: /root/.elm

phases:
  install:
    runtime-versions:
      nodejs: 20
    commands:
      - npm install -g elm

  build:
    commands:
      - elm make src/Main.elm --optimize

cache:
  key: elm-$(codebuild-hash-files elm.json)
  fallback-keys:
    - elm-
  paths:
    - /root/.elm/**/*
```

With elm-review (or other tools with their own `elm.json`):

```yaml
cache:
  key: elm-$(codebuild-hash-files elm.json review/elm.json)
  fallback-keys:
    - elm-
  paths:
    - /root/.elm/**/*
```

> **Note:** CodeBuild requires an S3 bucket configured in your project settings for caching. The `codebuild-hash-files` command generates a SHA-256 hash of the specified files, and `fallback-keys` uses prefix matching when no exact key match exists.

---

## Troubleshooting

**Cache not being restored?**

- Ensure the cache path matches the `ELM_HOME` directory you configured
- Check your CI provider's cache logs for errors

**Still seeing network requests?**

- A new dependency was added (expected — new packages must be downloaded once)
- Check that the `ELM_HOME` environment variable is set correctly before `elm make` runs

---

## Contributing

Have experience with other CI systems (Jenkins, Buildkite, etc.)? Contributions are welcome!

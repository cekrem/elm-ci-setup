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

The downloaded artifacts are immutable. They all have a specific hash that is checked by Elm. `ELM_HOME` also contains the elmi and elmo for the packages, so those are deterministic as well.

By saving and restoring `ELM_HOME` between builds, you only need network access when you _add_ a new dependency – not on every build. This is simpler, faster, and more reliable than, say, setting up a mirror of package.elm-lang.org.

Since package contents never change for a given version, **it is perfectly safe to cache indefinitely.**

---

## Shared vs. Hash-based Cache Keys

Each CI example below shows two options for cache keys:

**Option A (Shared key):** All builds share the same cache.

- ✅ Simplest setup
- ✅ Cache always hits after the first build
- ✅ New packages are added incrementally
- ⚠️ Cache may accumulate unused packages over time (usually not a problem in practice)

**Option B (Hash-based key):** Cache key changes when `elm.json` changes.

- ✅ Clean cache when dependencies change
- ✅ With `restore-keys` (GitHub Actions, CircleCI, Azure Pipelines), still get partial cache hits
- ⚠️ Without `restore-keys` support (GitLab CI, Bitbucket Pipelines), a full re-download occurs whenever dependencies change

**Recommendation:** Option A (shared key) is often sufficient since `ELM_HOME` is immutable – packages are only ever added, never modified. Option B is useful if you want stricter cache hygiene or if your CI provider supports `restore-keys` for partial cache reuse.

---

## CI Providers

- [GitHub Actions](#github-actions)
- [GitLab CI](#gitlab-ci)
- [CircleCI](#circleci)
- [Azure Pipelines](#azure-pipelines)
- [Bitbucket Pipelines](#bitbucket-pipelines)

---

### GitHub Actions

#### Option A: Shared cache key

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
          key: elm

      - name: Build
        run: elm make src/Main.elm --optimize
```

#### Option B: Hash-based cache key

```yaml
- name: Cache ELM_HOME
  uses: actions/cache@v4
  with:
    path: ${{ env.ELM_HOME }}
    key: elm-${{ hashFiles('elm.json') }}
    restore-keys: elm-
```

With elm-review (or other tools with their own `elm.json`):

```yaml
- name: Cache ELM_HOME
  uses: actions/cache@v4
  with:
    path: ${{ env.ELM_HOME }}
    key: elm-${{ hashFiles('elm.json', 'review/elm.json') }}
    restore-keys: elm-
```

> **Note:** You can add as many `elm.json` files as needed to the `hashFiles()` function.

---

### GitLab CI

#### Option A: Shared cache key

```yaml
stages:
  - build

variables:
  ELM_HOME: $CI_PROJECT_DIR/.elm

cache:
  key: elm
  paths:
    - $ELM_HOME

build:
  stage: build
  image: node:20
  before_script:
    - npm install -g elm
  script:
    - elm make src/Main.elm --optimize
```

#### Option B: Hash-based cache key

```yaml
cache:
  key:
    files:
      - elm.json
  paths:
    - $ELM_HOME
```

With elm-review (or other tools with their own `elm.json`):

```yaml
cache:
  key:
    files:
      - elm.json
      - review/elm.json
  paths:
    - $ELM_HOME
```

---

### CircleCI

#### Option A: Shared cache key

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
            - elm
      - run: npm install -g elm
      - run: elm make src/Main.elm --optimize
      - save_cache:
          key: elm
          paths:
            - ~/project/.elm
```

#### Option B: Hash-based cache key

```yaml
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
```

> **Note:** The second key (`elm-`) acts as a fallback for partial cache hits. Cache paths must match `ELM_HOME`.

---

### Azure Pipelines

#### Option A: Shared cache key

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
      key: '"elm"'
      path: $(ELM_HOME)
    displayName: Cache ELM_HOME

  - script: npm install -g elm
    displayName: Install Elm

  - script: elm make src/Main.elm --optimize
    displayName: Build
```

#### Option B: Hash-based cache key

```yaml
- task: Cache@2
  inputs:
    key: '"elm" | elm.json'
    restoreKeys: |
      "elm"
    path: $(ELM_HOME)
  displayName: Cache ELM_HOME
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

#### Option A: Shared cache key

```yaml
definitions:
  caches:
    elm: .elm

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

#### Option B: Hash-based cache key

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

> **Note:** Bitbucket cache definitions don't support variables, so ensure the cache path (`.elm`) matches where `ELM_HOME` points. Bitbucket Pipelines does not support fallback keys for file-based caches.

---

## Troubleshooting

**Cache not being restored?**

- Ensure the cache path matches where Elm actually stores data (i.e., the `ELM_HOME` you configured)

**Still seeing network requests?**

- A new dependency was added (expected behavior)
- The cache key changed, triggering a fresh download
- Check that `ELM_HOME` environment variable is set correctly

---

## Contributing

Have experience with other CI systems (Jenkins, Travis CI, etc.)? Contributions are welcome!

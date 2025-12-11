# Elm CI/CD Guide: Caching ELM_HOME

This guide shows how to properly cache Elm dependencies in CI/CD pipelines. By caching `ELM_HOME`, you can:

- **Avoid network requests** to package.elm-lang.org and github.com on most builds
- **Skip downloading and building** packages you've already fetched
- **Eliminate transient network issues** that can cause flaky builds

## How It Works

Elm stores all downloaded packages and their compiled artifacts in a directory called `ELM_HOME` (defaults to `~/.elm`). This cache is:

- **Shared across all projects** on the same machine
- **Essentially immutable** – once a package version is downloaded, it doesn't change
- **Safe to cache** between CI runs

By saving and restoring `ELM_HOME` between builds, you only need network access when you _add_ a new dependency – not on every build.

---

## GitHub Actions

### Option A: Shared cache key

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

### Option B: Hash-based cache key

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

## GitLab CI

### Option A: Shared cache key

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

### Option B: Hash-based cache key

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

## Shared vs. Hash-based Cache Keys

**Option A (Shared key):** All builds share the same cache.

- ✅ Simplest setup
- ✅ Cache always hits after the first build
- ✅ New packages are added incrementally
- ⚠️ Cache may accumulate unused packages over time (usually not a problem in practice)

**Option B (Hash-based key):** Cache key changes when `elm.json` changes.

- ✅ Clean cache when dependencies change
- ✅ With `restore-keys` (GitHub Actions), still get partial cache hits
- ⚠️ Without `restore-keys` support (GitLab CI), a full re-download occurs whenever dependencies change

**Recommendation:** Option A (shared key) is often sufficient since `ELM_HOME` is immutable – packages are only ever added, never modified. Option B is useful if you want stricter cache hygiene or if your CI provider supports `restore-keys` for partial cache reuse.

---

## Key Points

1. **`ELM_HOME` is shared** – It doesn't matter how many Elm projects or tools you have; they all benefit from the same cache.

2. **Use `restore-keys` for partial cache hits** – When dependencies change, you'll still get the previous cache and only need to download the _new_ packages. Use this whenever your CI provider supports it (e.g., GitHub Actions). Some providers like GitLab CI don't have an equivalent mechanism.

3. **The cache is immutable** – Package contents never change for a given version, making it perfectly safe to cache indefinitely.

4. **No need for parallel package infrastructure** – Caching `ELM_HOME` is simpler, faster, and more reliable than, say, setting up a mirror of package.elm-lang.org.

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

Have experience with other CI systems (CircleCI, Jenkins, Azure Pipelines, etc.)? Contributions are welcome!

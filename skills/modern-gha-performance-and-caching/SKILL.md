---
name: modern-gha-performance-and-caching
description: >-
  Use this skill when optimizing GitHub Actions runtime performance, runner economics,
  and billing costs: implementing high-speed dependency caching (setup-* native cache vs actions/cache),
  crafting deterministic cache keys with lockfile hashing, configuring path filtering and selective execution
  (dorny/paths-filter), tuning job-level timeouts, optimizing artifact exchange
  (actions/upload-artifact@v4 and download-artifact@v4), and leveraging ephemeral self-hosted runner sets (ARC).
---

# Modern GitHub Actions Performance, Caching & Runner Economics

This skill guides you through slashing build times, optimizing runner minutes, and engineering deterministic caching strategies in GitHub Actions.

---

## 1. Native Setup Action Caching vs `actions/cache`

Whenever possible, leverage first-party `setup-*` actions with native caching. They handle cache path resolution, OS quirks, and lockfile hashing automatically without boilerplate.

### Standard Setup Action Caching Examples

#### Node.js (`npm`, `pnpm`, `yarn`)
```yaml
- name: Setup Node.js with Native Caching
  uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5af # v4.1.0
  with:
    node-version: 22
    cache: npm # Automatically tracks and hashes package-lock.json
    cache-dependency-path: '**/package-lock.json' # Useful for monorepos
```

#### Python (`pip`, `poetry`, `uv`)
```yaml
# For modern lightning-fast Python installations, prefer Astral's setup-uv
- name: Setup uv with Cache
  uses: astral-sh/setup-uv@887a942a15af3a7626099df99e897a18d9e5ab3a # v5.1.0
  with:
    enable-cache: true
    cache-dependency-glob: "uv.lock"

# Standard Python setup
- name: Setup Python with Native Pip Cache
  uses: actions/setup-python@42375524e23c412d93fb67b49958b491fce71c38 # v5.4.0
  with:
    python-version: "3.12"
    cache: "pip"
```

#### Java (Gradle & Maven)
```yaml
- name: Setup Java Toolchain with Gradle Cache
  uses: actions/setup-java@3a4f6e1af504cf6a31855fa899c6aa5355ba6c12 # v4.7.0
  with:
    distribution: "temurin"
    java-version: "21"
    cache: "gradle"
```

#### Go
```yaml
- name: Setup Go with Built-in Cache
  uses: actions/setup-go@f111f3795f8e5bec54b74d47d2b802b11488a3f9 # v5.3.0
  with:
    go-version: "1.23"
    cache: true
    cache-dependency-path: "go.sum"
```

---

## 2. Granular `actions/cache` Engineering

When caching custom artifacts, build outputs, Docker layers, or tools not supported by `setup-*`, use `actions/cache`.

### Deterministic Cache Key Formula
Construct keys using three dimensions:
1. Operating system identifier (`${{ runner.os }}`)
2. Component or dependency name (`npm`, `cargo`, `gradle`)
3. Cryptographic hash of lockfiles (`${{ hashFiles('**/lockfile') }}`)

```yaml
- name: Cache Rust Cargo Build Dependencies
  uses: actions/cache@d4323d4df104b026a6aa633fdb11d772115cf71e # v4.2.2
  with:
    path: |
      ~/.cargo/bin/
      ~/.cargo/registry/index/
      ~/.cargo/registry/cache/
      ~/.cargo/git/db/
      target/
    key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}
    restore-keys: |
      ${{ runner.os }}-cargo-
```

### Decoupled Cache Restoration and Saving
To avoid saving broken or failed build states to the cache, decouple restore and save steps:
```yaml
- name: Restore Build Cache
  id: build-cache
  uses: actions/cache/restore@d4323d4df104b026a6aa633fdb11d772115cf71e # v4.2.2
  with:
    path: .next/cache
    key: ${{ runner.os }}-nextjs-${{ hashFiles('**/package-lock.json') }}-${{ hashFiles('**.[jt]s*') }}
    restore-keys: |
      ${{ runner.os }}-nextjs-${{ hashFiles('**/package-lock.json') }}-

- name: Build Application
  run: npm run build

- name: Save Cache on Successful Build Only
  uses: actions/cache/save@d4323d4df104b026a6aa633fdb11d772115cf71e # v4.2.2
  if: success() && steps.build-cache.outputs.cache-hit != 'true'
  with:
    path: .next/cache
    key: ${{ steps.build-cache.outputs.cache-primary-key }}
```

---

## 3. Selective Execution & Path Filtering

Executing full CI suites on changes to markdown files or documentation wastes runner hours.

### Static Path Filtering via Workflow Triggers
```yaml
on:
  push:
    branches: [main]
    paths-ignore:
      - '**.md'
      - 'docs/**'
      - '.gitignore'
      - 'LICENSE'
```

### Dynamic Monorepo Filtering with `dorny/paths-filter`
When multiple projects reside in one repository, execute jobs selectively based on changed files:
```yaml
jobs:
  filter:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    outputs:
      backend: ${{ steps.changes.outputs.backend }}
      frontend: ${{ steps.changes.outputs.frontend }}
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - uses: dorny/paths-filter@de90cc6fb38fc0963ad72b210f1f284cd68cea36 # v3.0.2
        id: changes
        with:
          filters: |
            backend:
              - 'services/backend/**'
              - 'go.mod'
              - 'go.sum'
            frontend:
              - 'apps/web/**'
              - 'package.json'
              - 'package-lock.json'

  test-backend:
    needs: [filter]
    if: needs.filter.outputs.backend == 'true'
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - run: go test ./...

  test-frontend:
    needs: [filter]
    if: needs.filter.outputs.frontend == 'true'
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - run: npm test
```

---

## 4. Optimized Artifact Exchange (`upload-artifact@v4`)

`actions/upload-artifact@v4` provides massively improved upload throughput via parallelized multi-part streaming, but requires conscious retention management.

### High-Throughput Artifact Upload Pattern
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - run: npm run build

      - name: Upload Build Artifacts
        uses: actions/upload-artifact@4cec3d8aa04e39d1a68397de0c4cd6fb9dce8ec1 # v4.6.1
        with:
          name: production-bundle
          path: dist/
          # 1. Reduce retention for ephemeral pipeline artifacts (default is 90 days!)
          retention-days: 3
          # 2. Skip compression if files are already compressed binaries or images
          compression-level: 6
          # 3. Fail job if expected output files were not generated
          if-no-files-found: error

  deploy:
    needs: [build]
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - name: Download Build Bundle
        uses: actions/download-artifact@cc203385981b70ca67e1cc392babf9cc229d5806 # v4.1.9
        with:
          name: production-bundle
          path: dist/
```

---

## 5. Job Timeout Discipline

Prevent hung processes, test deadlocks, or unresponsive network connections from burning through team runner minutes.

```yaml
jobs:
  fast-lint:
    runs-on: ubuntu-latest
    # Fail fast if linter stalls
    timeout-minutes: 5

  unit-tests:
    runs-on: ubuntu-latest
    timeout-minutes: 15

  heavy-e2e-suite:
    runs-on: ubuntu-latest
    timeout-minutes: 45
```

---

## 6. Runner Performance Rules of Thumb

1. **Avoid Monolithic Steps**: Group related commands but keep distinct stages (install, build, test, package) separated so step timings identify bottlenecks.
2. **Utilize RAM Disk for I/O Heavy Tests**: On `ubuntu-latest`, `/dev/shm` is an in-memory tmpfs filesystem. Directing SQLite or ephemeral database files to `/dev/shm` dramatically speeds up test execution.
3. **Turn Off Unnecessary Services**: Don't spin up service containers if lightweight in-memory mocks or Testcontainers can be used within the test runner.
4. **Prune Unused Docker Layers**: If building Docker images in CI, use BuildKit cache mounts (`--mount=type=cache`) and `cache-from: type=gha`.

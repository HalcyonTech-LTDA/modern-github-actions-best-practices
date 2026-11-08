---
name: modern-gha-cicd-and-release-pipelines
description: >-
  Use this skill when building, auditing, or orchestrating end-to-end CI/CD and release automation
  in GitHub Actions: enforcing shift-left workflow linting (actionlint, shellcheck),
  managing GitHub Environments with deployment protection rules and manual approvals,
  choreographing automated Semantic Versioning and Conventional Changelog releases,
  executing blue-green or canary deployment verification, generating rich step summaries
  ($GITHUB_STEP_SUMMARY), and posting automated PR feedback.
---

# Modern GitHub Actions CI/CD & Release Pipelines

This skill guides you through constructing production-grade deployment pipelines, automated releases, environment gates, and shift-left workflow validation.

---

## 1. Shift-Left Workflow Validation (`actionlint` & `shellcheck`)

Before merging workflow changes, validate YAML schemas, expression types, and inline shell scripts using `actionlint` and `shellcheck`.

### Automated Workflow Linter CI Job (`.github/workflows/lint-workflows.yml`)
```yaml
name: Lint GitHub Actions Workflows

on:
  pull_request:
    paths:
      - '.github/workflows/**'
      - '.github/actions/**'

permissions:
  contents: read

jobs:
  actionlint:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - name: Checkout Code
        uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - name: Run actionlint
        uses: reviewdog/action-actionlint@db58217885f9a6570da9c71be4e40ec33fe44a1f # v1.65.0
        with:
          actionlint_flags: "-shellcheck="
```

---

## 2. GitHub Environments & Deployment Gates

GitHub Environments provide formal protection rules, audit logging, and secret segregation for deployment targets (`staging`, `production`).

### Environment Deployment Configuration
```yaml
jobs:
  deploy-production:
    name: Deploy to Production
    needs: [build-and-test]
    runs-on: ubuntu-latest
    timeout-minutes: 30
    # Bind job to protected GitHub Environment
    environment:
      name: production
      url: https://api.production.mycompany.com/health
    permissions:
      contents: read
      id-token: write # For OIDC cloud auth

    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - name: Execute Deployment
        run: |
          echo "Deploying release to production infrastructure..."
```

### Environment Governance Best Practices
1. **Required Reviewers**: Enforce at least 1-2 authorized engineering leads or DevOps team approvals before jobs targeting `production` execute.
2. **Wait Timer**: Configure a 5-minute wait timer on staging to allow automated smoke tests or canary metrics to settle before manual promotion.
3. **Deployment Branches**: Restrict production environment deployments exclusively to the `main` branch or release tags (`v*`).

---

## 3. Automated Semantic Releases & Changelogs

Eliminate manual tagging and changelog generation by automating Semantic Versioning releases from Conventional Commits (`feat:`, `fix:`, `feat!:`, `BREAKING CHANGE:`).

### Release Automation Job
```yaml
name: Release

on:
  push:
    branches: [main]

permissions: {}

jobs:
  release:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    permissions:
      contents: write # Required to create tags and publish GitHub Releases
      pull-requests: read
      id-token: write # Required if publishing to npm/PyPI with provenance
    steps:
      - name: Checkout Code
        uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
        with:
          fetch-depth: 0 # Full history required for changelog generation

      - name: Setup Node Runtime
        uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5af # v4.1.0
        with:
          node-version: 22

      - name: Run Semantic Release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: npx semantic-release
```

---

## 4. Rich Markdown Step Summaries (`$GITHUB_STEP_SUMMARY`)

Make workflow run results easily readable by emitting structured markdown tables, metrics, and alerts to `$GITHUB_STEP_SUMMARY`.

```yaml
      - name: Generate Pipeline Run Summary
        run: |
          echo "## 🚀 Deployment Verification Summary" >> "$GITHUB_STEP_SUMMARY"
          echo "" >> "$GITHUB_STEP_SUMMARY"
          echo "| Check | Status | Duration |" >> "$GITHUB_STEP_SUMMARY"
          echo "| :--- | :---: | :---: |" >> "$GITHUB_STEP_SUMMARY"
          echo "| Unit Tests | ✅ Passed | 45s |" >> "$GITHUB_STEP_SUMMARY"
          echo "| Security Scan | ✅ Zero High/Crit | 1m 12s |" >> "$GITHUB_STEP_SUMMARY"
          echo "| Smoke Tests | ✅ 100% OK | 30s |" >> "$GITHUB_STEP_SUMMARY"
          echo "" >> "$GITHUB_STEP_SUMMARY"
          echo "> **Release Tag**: \`v1.4.0\` deployed by **${{ github.actor }}**" >> "$GITHUB_STEP_SUMMARY"
```

---

## 5. Workflow Command Annotations

Emit visual notices, warnings, and errors directly in the GitHub commit / PR diff UI using workflow commands:

```bash
# Workflow command syntax:
# ::notice file={name},line={line},endLine={endLine},title={title}::{message}
# ::warning file={name},line={line},endLine={endLine},title={title}::{message}
# ::error file={name},line={line},endLine={endLine},title={title}::{message}

echo "::warning file=package.json,line=10,title=Outdated Dependency::Lodash is 2 major versions behind"
echo "::notice title=Deployment Finished::Successfully updated production cluster"
```

---

## 6. End-to-End Enterprise CI/CD Pipeline Blueprint

```yaml
name: Enterprise CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.ref != 'refs/heads/main' }}

permissions: {}

jobs:
  quality-gate:
    name: Code Quality & Security
    runs-on: ubuntu-latest
    timeout-minutes: 15
    permissions:
      contents: read
      security-events: write
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5af # v4.1.0
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npm run lint
      - run: npm test -- --ci

  build:
    name: Build & Package
    needs: [quality-gate]
    runs-on: ubuntu-latest
    timeout-minutes: 15
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5af # v4.1.0
        with:
          node-version: 22
          cache: npm
      - run: npm ci && npm run build
      - uses: actions/upload-artifact@4cec3d8aa04e39d1a68397de0c4cd6fb9dce8ec1 # v4.6.1
        with:
          name: app-dist
          path: dist/
          retention-days: 3

  deploy-staging:
    name: Deploy to Staging
    needs: [build]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    timeout-minutes: 20
    environment:
      name: staging
      url: https://staging.mycompany.com
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/download-artifact@cc203385981b70ca67e1cc392babf9cc229d5806 # v4.1.9
        with:
          name: app-dist
          path: dist/
      - name: Deploy to Cloud
        run: echo "Deploying to Staging..."

  deploy-production:
    name: Deploy to Production
    needs: [deploy-staging]
    runs-on: ubuntu-latest
    timeout-minutes: 20
    environment:
      name: production
      url: https://mycompany.com
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/download-artifact@cc203385981b70ca67e1cc392babf9cc229d5806 # v4.1.9
        with:
          name: app-dist
          path: dist/
      - name: Deploy to Cloud
        run: echo "Deploying to Production..."
```

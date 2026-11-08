<div align="center">

<a href="https://www.halcyontech.com.br/" target="_blank" rel="noopener noreferrer">
  <img src="assets/halcyon-horizontal-cor.svg" alt="Halcyon Tech" height="60">
</a>

# Modern GitHub Actions Best Practices Plugin
### Zero-Trust Supply Chain Security, Modular Workflows & High-Speed CI/CD Engineering

[![Antigravity Plugin](https://img.shields.io/badge/Antigravity-Plugin%20v1.0.0-4F46E5.svg)](https://antigravity.google)
[![License: MIT](https://img.shields.io/badge/License-MIT-10B981.svg)](./LICENSE)
[![Website](https://img.shields.io/badge/Website-halcyontech.com.br-008080.svg)](https://www.halcyontech.com.br/)
[![Organization](https://img.shields.io/badge/Organization-HalcyonTech--LTDA-black.svg)](https://github.com/HalcyonTech-LTDA)

<p align="center">
  <b>Enterprise-grade modern GitHub Actions best practices covering zero-trust security & supply chain hardening (SHA-256 action pinning, least-privilege permissions, OIDC cloud federation, injection defense), modular workflow architecture (reusable workflows, composite actions, concurrency control), high-performance caching & runner economics, and shift-left CI/CD release quality gates.</b>
</p>

</div>

---

## Installation & Verification

### Antigravity CLI (`agy`)

Install directly into your Antigravity environment via the CLI:

```bash
agy plugin install https://github.com/HalcyonTech-LTDA/modern-github-actions-best-practices
```

Verify that the plugin is active and conforms to the strict official schema:

```bash
agy plugin list
agy plugin validate --strict ./modern-github-actions-best-practices
```

---

## Overview

Modern cloud engineering relies on GitHub Actions as the central nervous system for continuous integration, testing, packaging, and production deployments. However, without strict architectural discipline and security guardrails, enterprise pipelines rapidly accumulate severe vulnerabilities and performance anti-patterns:

- **Supply Chain Vulnerabilities**: Referencing third-party actions by mutable Git tags (`@v4`, `@main`), exposing production environments to upstream repository compromises or tag rewrites.
- **Overprivileged Ambient Tokens**: Leaving default repository permissions (`permissions: write-all`) active, allowing compromised build steps or pull requests to alter production assets.
- **Remote Code Execution via Script Injection**: Evaluating untrusted user expressions (`${{ github.event.issue.title }}`) directly inside inline shell commands.
- **Static Long-Lived Cloud Keys**: Storing long-lived `AWS_ACCESS_KEY_ID` or GCP service account credentials in repository secrets instead of short-lived OpenID Connect (OIDC) federation.
- **Monolithic Spaghetti Workflows**: Duplicating identical multi-step build, lint, and test scripts across dozens of workflow files without reusable components.
- **Runaway Runner Minute Waste**: Missing job timeouts (`timeout-minutes`) and unmanaged concurrency, allowing deadlocked tests to burn 6 hours of billing quota.
- **Blinded CI Runs**: Leaving pipeline results buried in raw terminal text instead of publishing structured markdown step summaries and PR annotations.

This plugin establishes production-grade engineering standards for GitHub Actions across **four foundational pillars**:

```text
                                  ┌───────────────────────────────────────────────┐
                                  │   Modern GitHub Actions Engineering System   │
                                  └───────────────────────┬───────────────────────┘
                                                          │
         ┌────────────────────────────────┬───────────────┴───────────────┬────────────────────────────────┐
         │                                │                               │                                │
         ▼                                ▼                               ▼                                ▼
┌──────────────────┐            ┌──────────────────┐            ┌──────────────────┐            ┌──────────────────┐
│  Supply Chain &  │            │  Modular Layout  │            │  Performance &   │            │ CI/CD Deployment │
│     Security     │            │  & Architecture  │            │ Runner Economics │            │  Quality Gates   │
├──────────────────┤            ├──────────────────┤            ├──────────────────┤            ├──────────────────┤
│• SHA-256 Pinning │            │• workflow_call   │            │• setup-* Caching │            │• actionlint Gate │
│• Least Privilege │            │• Composite Act.  │            │• actions/cache   │            │• Environments    │
│• OIDC Cloud Auth │            │• Concurrency     │            │• paths-filter    │            │• Semantic Ver.   │
│• Injection Guard │            │• Matrix Builds   │            │• timeout-minutes │            │• Step Summaries  │
│• SLSA Provenance │            │• Containers      │            │• v4 Artifacts    │            │• Annotations     │
└──────────────────┘            └──────────────────┘            └──────────────────┘            └──────────────────┘
```

---

## Specialized Skills Index

This plugin provides four specialized skills that can be activated on-demand by the agent or developer:

| Skill | Description | Primary Use Cases |
| :--- | :--- | :--- |
| [`modern-gha-security-and-hardening`](skills/modern-gha-security-and-hardening/SKILL.md) | Zero-trust supply chain security and hardening. | SHA-256 pinning, Dependabot config, least-privilege token permissions, OIDC cloud logins (AWS, GCP, Azure), script injection defense, and SLSA provenance. |
| [`modern-gha-architecture-and-modularity`](skills/modern-gha-architecture-and-modularity/SKILL.md) | Modular workflow decomposition and design patterns. | Typed reusable workflows (`workflow_call`), custom composite actions (`using: composite`), concurrency cancellation, multi-dimensional matrices, and service containers. |
| [`modern-gha-performance-and-caching`](skills/modern-gha-performance-and-caching/SKILL.md) | High-speed runner performance and billing optimization. | First-party setup action caching (`setup-node`, `setup-python`, `setup-java`), `actions/cache` lockfile hashing, `dorny/paths-filter`, timeout bounds, and v4 artifact uploads. |
| [`modern-gha-cicd-and-release-pipelines`](skills/modern-gha-cicd-and-release-pipelines/SKILL.md) | Production delivery, release automation, and governance. | Shift-left `actionlint` / `shellcheck` gates, protected GitHub Environments with approvals, semantic releases, `$GITHUB_STEP_SUMMARY` markdown, and diff annotations. |

---

## Non-Negotiable Guardrails (Absolute Invariants)

1. **Mandatory Full 40-Character SHA-256 Action Pinning**:
   All non-local actions must be referenced by an immutable 40-character commit hash followed by a semantic version comment (`uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2`). Mutable tags (`@v4`, `@main`) are strictly prohibited in production.
2. **Least-Privilege Token Permissions by Default**:
   Every workflow must declare a root-level `permissions: {}` block. Specific granular permissions (`contents: read`, `id-token: write`, `pull-requests: write`) must be requested only at the individual job level.
3. **Zero Shell Injection: Untrusted Expressions Must Use Environment Variables**:
   Direct interpolation of GitHub event expressions (e.g. `${{ github.event.issue.title }}`) inside `run:` inline shell scripts is strictly prohibited. Untrusted inputs must be mapped to intermediate environment variables.
4. **Zero Static Long-Lived Cloud Credentials (Mandatory OIDC)**:
   Storing long-lived static cloud provider keys in GitHub Secrets is forbidden. Production pipelines must use OpenID Connect (OIDC) federation with AWS IAM, Google Cloud Workload Identity, or Azure Federated Credentials.
5. **Mandatory Job-Level Timeouts (`timeout-minutes`)**:
   Every job must define an explicit `timeout-minutes` value (typically 10-15 minutes for unit tests, max 30-45 minutes for integration) to prevent 6-hour runaway executions.
6. **Concurrency Control with Cancel-in-Progress for Ephemeral Branches**:
   Workflows triggered on pull requests must specify a concurrency group with `cancel-in-progress: true` to eliminate redundant runs and free runner capacity.
7. **Shift-Left Static Analysis**:
   All workflow files must pass `actionlint` and `shellcheck` static validation before merging into the main branch.

---

## Quick Start Example: Zero-Trust Hardened Workflow

```yaml
name: Enterprise CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.ref != 'refs/heads/main' }}

# Invariant 2: Root-level least privilege
permissions: {}

jobs:
  test:
    name: Run Test Suite
    runs-on: ubuntu-latest
    # Invariant 5: Mandatory job timeout
    timeout-minutes: 15
    permissions:
      contents: read
    steps:
      # Invariant 1: Full SHA-256 action pinning
      - name: Checkout Code
        uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - name: Setup Node Runtime with Built-in Cache
        uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5af # v4.1.0
        with:
          node-version: 22
          cache: npm

      - name: Install Dependencies
        run: npm ci

      - name: Execute Tests
        run: npm test

      # Invariant 3: Untrusted input mapped to environment variable
      - name: Log Pull Request Context
        if: github.event_name == 'pull_request'
        env:
          PR_TITLE: ${{ github.event.pull_request.title }}
        run: |
          echo "Verified Pull Request: ${PR_TITLE}"
```

---

## What's Included (Component Catalog)

### Contextual Rules (`rules/`)
- [`rules/modern-github-actions-engineering.md`](rules/modern-github-actions-engineering.md): Comprehensive architectural standards, quality router, and the 7 non-negotiable guardrails enforced across all GitHub Actions files.

### Workflow Skills (`skills/`)
- [`skills/modern-gha-security-and-hardening/SKILL.md`](skills/modern-gha-security-and-hardening/SKILL.md): Supply chain hardening, OIDC federation, least privilege, and script injection defense.
- [`skills/modern-gha-architecture-and-modularity/SKILL.md`](skills/modern-gha-architecture-and-modularity/SKILL.md): Reusable workflows (`workflow_call`), composite actions, concurrency control, and matrix strategies.
- [`skills/modern-gha-performance-and-caching/SKILL.md`](skills/modern-gha-performance-and-caching/SKILL.md): Setup action caching, custom `actions/cache`, path filtering, and artifact exchange.
- [`skills/modern-gha-cicd-and-release-pipelines/SKILL.md`](skills/modern-gha-cicd-and-release-pipelines/SKILL.md): Linter integration (`actionlint`), GitHub Environments, semantic releases, and step summaries.

---

## Directory Hierarchy

```text
modern-github-actions-best-practices/
├── plugin.json                                         # Official manifest (schema v1)
├── LICENSE                                             # MIT License (Copyright 2026 HalcyonTech LTDA)
├── .gitignore                                          # Ignores system junk, logs, and caches
├── .gitattributes                                      # Deterministic line endings (* text=auto eol=lf)
├── README.md                                           # Comprehensive documentation & runbook
├── assets/
│   ├── logo.svg                                        # Official Halcyon Tech vector icon
│   └── halcyon-horizontal-cor.svg                      # Official Halcyon Tech horizontal brand card
├── rules/
│   └── modern-github-actions-engineering.md            # Normative guardrails and skill router
└── skills/
    ├── modern-gha-security-and-hardening/
    │   └── SKILL.md                                    # Supply chain hardening & OIDC runbook
    ├── modern-gha-architecture-and-modularity/
    │   └── SKILL.md                                    # Reusable workflows & composite actions
    ├── modern-gha-performance-and-caching/
    │   └── SKILL.md                                    # High-speed caching & runner economics
    └── modern-gha-cicd-and-release-pipelines/
        └── SKILL.md                                    # CI/CD pipelines & semantic releases
```

---

## 💡 Prompt Recipes & Use Cases

Copy and paste these prompts directly into your Antigravity chat:

### 1. Zero-Trust Security Audit
> *"Audit all workflows under `.github/workflows/` against modern security best practices. Check for unpinned actions, overly broad token permissions, potential script injection vulnerabilities, and missing job timeouts."*

### 2. OIDC Migration from Static Cloud Secrets
> *"Refactor our AWS deployment workflow to eliminate static `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`. Implement passwordless OIDC authentication using `aws-actions/configure-aws-credentials` and minimal token permissions."*

### 3. Pipeline Performance & Caching Optimization
> *"Analyze our CI pipeline for build bottlenecks. Implement native setup action dependency caching, configure smart path filtering with `dorny/paths-filter`, and add concurrency control to cancel outdated PR runs."*

### 4. Modular Reusable Workflow Refactoring
> *"Extract our repetitive Docker build and test steps into a reusable workflow (`workflow_call`) with typed inputs for image tags and build arguments, and a composite action for toolchain installation."*

---

## 📄 License & Corporate Governance

Distributed under the **MIT License**. See [LICENSE](LICENSE) for the full text.

Engineered with precision and excellence by **[Halcyon Tech](https://www.halcyontech.com.br/)** &mdash; [GitHub Organization](https://github.com/HalcyonTech-LTDA).

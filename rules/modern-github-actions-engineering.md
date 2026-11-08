---
description: >-
  Automatically enforces modern GitHub Actions engineering standards: zero-trust supply chain security (SHA-256 action pinning, least-privilege permissions, OIDC cloud federation, injection defense), modular workflow architecture (reusable workflows, composite actions, concurrency control), high-performance caching & runner economics, and shift-left CI/CD release quality gates.
trigger: model_decision
---

# Modern GitHub Actions Engineering Guardrails & Skill Router

When authoring, reviewing, refactoring, securing, or optimizing GitHub Actions workflows, strictly adhere to Modern GitHub Actions Engineering Standards and the corresponding specialized skills:

| When working on... | Consult & Activate Skill | Key Focus Areas |
| :--- | :--- | :--- |
| **Security, Hardening & Supply Chain** | `modern-gha-security-and-hardening` | Immutable SHA-256 action pinning (`uses: action@<commit-sha> # vX.Y.Z`), least-privilege token permissions (`permissions: {}` top-level), passwordless OIDC federation (AWS IAM, GCP Workload Identity, Azure Federated Credentials, Vault), bash script injection prevention via intermediate environment variables, `pull_request_target` vs `pull_request` boundary segregation, OpenSSF Scorecard, and artifact attestation (`actions/attest-build-provenance`). |
| **Workflow Architecture & Modularity** | `modern-gha-architecture-and-modularity` | Reusable workflows (`workflow_call`) with typed inputs/secrets/outputs, custom composite actions (`action.yaml` with `using: 'composite'`), concurrency control groups (`cancel-in-progress: true`), multidimensional & dynamic matrix strategies (`matrix:` with `fail-fast`), service containers (PostgreSQL, Redis, Kafka) for integration testing, and clean repository layout (`.github/workflows/`, `.github/actions/`). |
| **Performance, Caching & Runner Economics** | `modern-gha-performance-and-caching` | First-party setup action caching (`setup-node`, `setup-python`, `setup-java`, `setup-go`), granular `actions/cache` restoration with lockfile hashing, path-based selective execution (`dorny/paths-filter`, `paths:` / `paths-ignore:`), mandatory job `timeout-minutes` (preventing hung 6-hour runs), fast artifact exchange via `actions/upload-artifact@v4` and `download-artifact@v4`, and ephemeral self-hosted runner sets (ARC). |
| **CI/CD Quality Gates & Release Automation** | `modern-gha-cicd-and-release-pipelines` | Static workflow linting (`actionlint`, `shellcheck`), GitHub Environments with deployment protection rules and manual review gates, automated semantic versioning and conventional changelog releases (`softprops/action-gh-release`), zero-downtime deployment choreography (canary, blue-green), markdown step summaries (`$GITHUB_STEP_SUMMARY`), and automated status checks. |

---

## Non-Negotiable Guardrails (Absolute Invariants)

1. **Mandatory Full 40-Character SHA-256 Action Pinning**:
   - Referencing third-party actions by mutable Git tags (e.g. `@v4`, `@main`) in production workflows is strictly prohibited. Every non-local action must be pinned to a full 40-character SHA-256 commit hash, followed by an inline comment denoting the semantic version:
     ```yaml
     # ✅ Correct: Immutable commit SHA with human-readable tag comment
     uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
     uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5af # v4.1.0

     # ❌ Prohibited: Vulnerable to upstream tag hijacking / supply chain tampering
     uses: actions/checkout@v4
     uses: actions/checkout@main
     ```
   - Automated maintenance: Dependabot must be configured in `.github/dependabot.yml` with `package-ecosystem: "github-actions"` and `pin-actions-tags: false` to automatically propose SHA updates alongside version bumps.

2. **Least-Privilege Token Permissions by Default**:
   - Never rely on default repository token permissions (`permissions: write-all`).
   - Every workflow file must explicitly declare a top-level `permissions` block set to empty or read-only:
     ```yaml
     # Top of workflow file
     permissions: {}
     # OR
     permissions:
       contents: read
     ```
   - Individual jobs must surgically request only the specific permissions they need (e.g. `id-token: write` for OIDC, `pull-requests: write` for PR comments, `packages: write` for GHCR).

3. **Zero Shell Injection: Untrusted Context Must Never Be Interpolated Inline**:
   - Direct interpolation of GitHub event expressions inside `run:` inline shell scripts is strictly prohibited. Untrusted attacker-controlled inputs (issue titles, body, branch names, commit messages, PR titles) evaluated within bash commands lead directly to arbitrary command execution (RCE).
   - Untrusted values must always be mapped to intermediate environment variables before consumption:
     ```yaml
     # ❌ Prohibited: Remote Code Execution vulnerability
     - run: echo "Processing PR: ${{ github.event.pull_request.title }}"

     # ✅ Correct: Sanitized via intermediate environment variable
     - name: Process PR Title
       env:
         PR_TITLE: ${{ github.event.pull_request.title }}
       run: |
         echo "Processing PR: ${PR_TITLE}"
     ```

4. **Zero Static Long-Lived Cloud Credentials (Mandatory OIDC)**:
   - Storing long-lived cloud credentials (such as `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`, GCP service account JSON keys, or Azure Service Principal client secrets) in GitHub Secrets is forbidden for modern cloud deployments.
   - Use passwordless OpenID Connect (OIDC) federation. Configure cloud IAM trust policies with GitHub's OIDC Provider (`token.actions.githubusercontent.com`) and request temporary credentials in workflow jobs via `permissions: id-token: write`.

5. **Mandatory Job-Level Timeouts (`timeout-minutes`)**:
   - Every job must explicitly define `timeout-minutes`. The default GitHub Actions timeout is 360 minutes (6 hours), which risks consuming entire organization billing budgets and hanging self-hosted runners during hung network calls, deadlocks, or unresponsive tests.
   - Normal build/test jobs must specify `timeout-minutes: 15` or `timeout-minutes: 30`. Long-running integration jobs must set realistic explicit bounds (e.g. `timeout-minutes: 60`).

6. **Concurrency Control with Cancel-in-Progress for Ephemeral Branches**:
   - Pull request and feature branch workflows must define a concurrency group with `cancel-in-progress: true`:
     ```yaml
     concurrency:
       group: ${{ github.workflow }}-${{ github.ref }}
       cancel-in-progress: true
     ```
   - This eliminates redundant runs on rapid consecutive pushes, frees runner queues, and saves compute costs.
   - For protected production deployments (`refs/heads/main`), set `cancel-in-progress: false` to ensure atomic, ordered deployments without aborting active production rollouts.

7. **Shift-Left Static Analysis (`actionlint` & `shellcheck`)**:
   - All workflow YAML files and inline shell scripts must pass `actionlint` and `shellcheck` static analysis before merging. Workflows must be validated for schema conformance, shell syntax errors, and untrusted variable usage.

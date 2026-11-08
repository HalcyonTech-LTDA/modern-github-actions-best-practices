---
name: modern-gha-architecture-and-modularity
description: >-
  Use this skill when architecting, structuring, or refactoring GitHub Actions workflows:
  authoring typed Reusable Workflows (workflow_call) with inputs, secrets, and outputs;
  designing custom Composite Actions (action.yaml with using: 'composite');
  configuring smart concurrency groups (cancel-in-progress); implementing multi-dimensional
  dynamic matrix strategies; orchestrating service containers (PostgreSQL, Redis, Kafka)
  for integration testing; and establishing clean repository workflow layouts.
---

# Modern GitHub Actions Architecture & Modularity

This skill guides you through decomposing complex CI/CD logic into reusable, testable, and maintainable modular workflows and composite actions.

---

## 1. Reusable Workflows (`workflow_call`)

Reusable workflows enable centralized engineering standards across repositories or within large multi-service codebases. They run as independent jobs and can declare typed inputs, secrets, and outputs.

### Defining a Reusable Workflow (`.github/workflows/reusable-test.yml`)
```yaml
name: Reusable Test Suite

on:
  workflow_call:
    inputs:
      node-version:
        description: "Node.js runtime version to execute against"
        required: false
        type: string
        default: "22"
      run-e2e:
        description: "Whether to execute end-to-end integration tests"
        required: false
        type: boolean
        default: false
    secrets:
      NPM_AUTH_TOKEN:
        description: "Authentication token for private NPM registry"
        required: false
    outputs:
      test-coverage:
        description: "Overall percentage of test coverage"
        value: ${{ jobs.execute-tests.outputs.coverage }}

permissions:
  contents: read

jobs:
  execute-tests:
    runs-on: ubuntu-latest
    timeout-minutes: 20
    outputs:
      coverage: ${{ steps.coverage.outputs.percentage }}
    steps:
      - name: Checkout Source Code
        uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - name: Setup Node.js Runtime
        uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5af # v4.1.0
        with:
          node-version: ${{ inputs.node-version }}
          cache: npm

      - name: Run Test Suite
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_AUTH_TOKEN }}
        run: npm ci && npm test -- --coverage

      - name: Extract Coverage Metric
        id: coverage
        run: echo "percentage=94.5" >> "$GITHUB_OUTPUT"
```

### Invoking the Reusable Workflow from a Caller Workflow
```yaml
name: Pull Request CI

on:
  pull_request:
    branches: [main]

permissions: {}

jobs:
  test:
    name: Execute Modular Tests
    # Local reusable workflow invocation
    uses: ./.github/workflows/reusable-test.yml
    with:
      node-version: "22"
      run-e2e: true
    secrets:
      NPM_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}

  report:
    name: Report Test Results
    needs: [test]
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - name: Display Coverage Result
        run: echo "Total Coverage is: ${{ needs.test.outputs.test-coverage }}%"
```

---

## 2. Custom Composite Actions (`action.yaml`)

Composite actions encapsulate a sequence of reusable steps (such as setup, tool installation, or linting) into a single action file. Unlike reusable workflows, composite actions run inside the caller job's runner environment and share its filesystem and environment.

### Choosing Between Reusable Workflows vs Composite Actions
| Feature | Reusable Workflow (`workflow_call`) | Composite Action (`using: 'composite'`) |
| :--- | :--- | :--- |
| **Execution Boundary** | Runs as a separate job (spins up new runner) | Runs as steps within the caller job's runner |
| **Filesystem Isolation** | Completely separate filesystem | Shares current job workspace and files |
| **Matrix Support** | Can call a matrix across reusable jobs | Can be invoked inside a matrix step |
| **Multiple Jobs** | Supports multiple jobs and DAG dependencies | Single sequence of steps only |
| **Best For** | Enterprise standard pipelines, release flows | Reusable setup steps, tool installation, cache hooks |

### Canonical Composite Action Layout (`.github/actions/setup-toolchain/action.yaml`)
```yaml
name: Setup Project Toolchain & Cache
description: Checks out code, configures Node.js, and populates dependency caches.

inputs:
  node-version:
    description: "Node.js version to configure"
    required: false
    default: "22"

outputs:
  cache-hit:
    description: "Whether node_modules cache was resolved"
    value: ${{ steps.cache.outputs.cache-hit }}

runs:
  using: "composite"
  steps:
    - name: Setup Node Runtime
      uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5af # v4.1.0
      with:
        node-version: ${{ inputs.node-version }}

    - name: Cache Node Modules
      id: cache
      uses: actions/cache@d4323d4df104b026a6aa633fdb11d772115cf71e # v4.2.2
      with:
        path: ~/.npm
        key: ${{ runner.os }}-node-${{ inputs.node-version }}-${{ hashFiles('**/package-lock.json') }}
        restore-keys: |
          ${{ runner.os }}-node-${{ inputs.node-version }}-

    - name: Install Dependencies
      shell: bash # Note: shell declaration is mandatory in composite action run steps
      run: npm ci --prefer-offline
```

---

## 3. Concurrency Control Strategies

Concurrency groups prevent redundant parallel runs and ensure atomic serial deployments.

### Strategy A: Pull Request Fast Cancellation (Save Runner Hours)
Cancel in-progress runs when a developer pushes subsequent commits to the same PR:
```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref || github.run_id }}
  cancel-in-progress: true
```

### Strategy B: Production Deployment Queueing (Zero-Drop Serialization)
Never cancel active production deployments; queue them serially:
```yaml
concurrency:
  group: production-release-pipeline
  cancel-in-progress: false # Subsequent triggers will wait safely in queue
```

---

## 4. Multi-Dimensional & Dynamic Matrix Builds

Matrix builds enable parallel execution across multiple platforms, runtimes, or configurations.

### Multi-Dimensional Matrix with Exclusions
```yaml
jobs:
  matrix-test:
    runs-on: ${{ matrix.os }}
    timeout-minutes: 20
    strategy:
      fail-fast: false # Prevent one failure from abruptly cancelling all other matrix runs
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
        node-version: [20, 22]
        exclude:
          # Windows on Node 20 not supported for this project
          - os: windows-latest
            node-version: 20
        include:
          # Canary experimental run
          - os: ubuntu-latest
            node-version: 23
            experimental: true
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5af # v4.1.0
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm ci && npm test
        continue-on-error: ${{ matrix.experimental || false }}
```

### Dynamic Matrix Generation via Script Output
When testing affected microservices in a monorepo, compute matrix items dynamically:
```yaml
jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.set-matrix.outputs.matrix }}
    steps:
      - id: set-matrix
        run: |
          # Compute JSON array of changed service directories
          echo 'matrix=["services/billing", "services/auth"]' >> "$GITHUB_OUTPUT"

  build-services:
    needs: [detect-changes]
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: ${{ fromJson(needs.detect-changes.outputs.matrix) }}
    steps:
      - name: Build Service
        run: echo "Building microservice: ${{ matrix.service }}"
```

---

## 5. Integration Testing with Service Containers

GitHub Actions runners support running background Docker containers linked via host networking:

```yaml
jobs:
  integration-test:
    runs-on: ubuntu-latest
    timeout-minutes: 20
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: test_user
          POSTGRES_PASSWORD: test_password
          POSTGRES_DB: integration_test_db
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - name: Run Integration Tests
        env:
          DATABASE_URL: postgres://test_user:test_password@localhost:5432/integration_test_db
          REDIS_URL: redis://localhost:6379
        run: npm run test:integration
```

---

## 6. Directory Hierarchy Standards

Standardize project automation files under `.github/`:

```text
.github/
├── actions/
│   ├── setup-toolchain/
│   │   └── action.yaml          # Local composite action
│   └── docker-build-push/
│       └── action.yaml
├── workflows/
│   ├── ci.yml                   # Main PR and push integration pipeline
│   ├── release.yml              # Production semantic release pipeline
│   ├── security-scan.yml        # Scheduled CodeQL / Trivy security scanning
│   └── reusable-quality-gate.yml# Enterprise reusable workflow (workflow_call)
└── dependabot.yml               # Automated dependency and action updates
```

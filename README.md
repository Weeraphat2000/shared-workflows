# Shared Workflows

This repository contains **reusable GitHub Actions workflows** that can be called from other repositories. Reusable workflows follow the DRY (Don't Repeat Yourself) principle and ensure consistency across multiple projects.

## Available Workflows

### Node.js CI Pipeline (`nodejs-ci.yml`)

A comprehensive reusable workflow for Node.js projects that includes:

- 🔒 **Security Audit** - Check for vulnerabilities with npm audit
- 🎨 **Code Quality** - ESLint for linting and Prettier for formatting
- 🏗️ **Build** - Run build step if available
- 🧪 **Tests** - Single or matrix testing across multiple Node versions
- 📋 **Summary** - Pipeline status overview

#### Usage

In your repository, create a workflow file (e.g., `.github/workflows/ci.yml`):

```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  ci:
    uses: YOUR_USERNAME/shared-workflows/.github/workflows/nodejs-ci.yml@main
    with:
      node-version: "18"
      working-directory: "."
      matrix-test: false
    secrets: inherit
```

#### Inputs

| Input                | Description                                      | Required | Default | Type    |
| -------------------- | ------------------------------------------------ | -------- | ------- | ------- |
| `node-version`       | Node.js version to use                           | No       | `18`    | string  |
| `working-directory`  | Directory containing package.json                | No       | `.`     | string  |
| `run-lint`           | Whether to run ESLint                            | No       | `true`  | boolean |
| `run-tests`          | Whether to run tests                             | No       | `true`  | boolean |
| `run-security-audit` | Whether to run npm audit                         | No       | `true`  | boolean |
| `run-format-check`   | Whether to check code formatting with Prettier   | No       | `true`  | boolean |
| `run-build`          | Whether to run build step                        | No       | `true`  | boolean |
| `matrix-test`        | Run tests on multiple Node versions (18, 20, 22) | No       | `false` | boolean |
| `text`               | Additional text for pipeline summary             | No       | -       | string  |

---

## GitHub Actions Core Concepts

### 1. Conditional Execution (`if`)

Control when jobs or steps run based on conditions:

```yaml
# Run only if input is true
security-audit:
  if: ${{ inputs.run-security-audit }}

# Run specific step only for Node 20
- name: Upload coverage
  if: ${{ matrix.node-version == '20' }}

# Always run, even if previous jobs failed
summary:
  if: always()
```

**Common conditions:**
| Condition | Description |
|-----------|-------------|
| `${{ inputs.run-tests }}` | Check input value |
| `${{ matrix.node-version == '18' }}` | Compare matrix variable |
| `always()` | Always run (even if previous jobs failed) |
| `failure()` | Run only if previous job failed |
| `success()` | Run only if all previous jobs succeeded |

---

### 2. Passing Variables (Inputs)

Reusable workflows accept typed inputs that callers can customize:

```yaml
# Define inputs in reusable workflow
inputs:
  node-version:
    type: string
    default: "18"
    description: "Node.js version to use"
  matrix-test:
    type: boolean
    default: false
    description: "Run tests on multiple versions"
```

Access inputs in workflow steps:

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: ${{ inputs.node-version }}
```

Pass inputs from calling workflow:

```yaml
jobs:
  ci:
    uses: shared-workflows/.../nodejs-ci.yml@main
    with:
      node-version: "20"
      matrix-test: true
```

---

### 3. Default Values (`defaults`)

Set default working directory for all steps in a job (avoid repetition):

```yaml
defaults:
  run:
    working-directory: ${{ inputs.working-directory }}

steps:
  - name: Install dependencies
    run: npm ci # Automatically runs in working-directory

  - name: Run tests
    run: npm test # Also runs in working-directory
```

Without defaults, you'd need to specify `working-directory` in every step.

---

### 4. Job Dependencies (`needs`)

Control job execution order and data flow:

```yaml
jobs:
  code-quality:
    runs-on: ubuntu-latest
    # This job runs first

  build:
    needs: [code-quality] # Wait for code-quality to complete
    # Fails if code-quality fails

  test:
    needs: [code-quality] # Can run parallel with build

  summary:
    needs: [security-audit, code-quality, build, test]
    if: always() # Run even if some jobs failed
```

**Benefits:**

- Ensures tests run before deployment
- Saves resources by running in sequence when needed
- Can access previous job outputs with `${{ needs.jobname.outputs.name }}`

---

### 5. Matrix Strategy (`strategy`)

Run a job across multiple configurations without duplicating code:

```yaml
test-matrix:
  strategy:
    fail-fast: false # Continue even if one version fails
    matrix:
      node-version: ["18", "20", "22"] # Create 3 parallel jobs

  steps:
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ matrix.node-version }} # Access current value

    - name: Upload coverage for Node 20 only
      if: ${{ matrix.node-version == '20' }}
      run: npm run test:coverage
```

**What it does:**

- Creates 3 separate jobs (one for each Node version)
- All run in parallel
- Each job has access to `matrix.node-version`

**Configuration options:**
| Option | Description |
|--------|-------------|
| `fail-fast: true` (default) | Stop all jobs if one fails |
| `fail-fast: false` | Continue running even if one fails |

---

### 6. Secrets

Pass sensitive data securely without exposing in logs:

```yaml
# In calling workflow - inherit all secrets
jobs:
  ci:
    uses: shared-workflows/.../nodejs-ci.yml@main
    secrets: inherit
```

Access secrets in reusable workflow:

```yaml
- name: Send notification
  env:
    SLACK_TOKEN: ${{ secrets.SLACK_TOKEN }}
  run: echo "Token available in env"
```

**Important:**

- Secrets are never logged in output
- Must be defined in repository settings (Settings > Secrets)
- Use `secrets: inherit` to pass all repo secrets to reusable workflow

---

### 7. Artifacts

Store and share files between jobs and workflows:

```yaml
# Upload artifacts
- name: Upload coverage report
  uses: actions/upload-artifact@v4
  with:
    name: coverage-report
    path: coverage/
    retention-days: 7 # Auto-delete after 7 days

# Download in another job
- name: Download artifacts
  uses: actions/download-artifact@v4
  with:
    name: coverage-report
    path: ./reports/
```

**Use cases:**

- Store test coverage reports
- Save build outputs
- Archive logs for debugging

---

### 8. Continue on Error

Prevent job failure if a step fails:

```yaml
- name: Run npm audit
  run: npm audit
  continue-on-error: true # Job continues even if vulnerabilities found

- name: Format check
  run: npx prettier --check .
  continue-on-error: true # Non-critical, don't fail the job
```

**Use when:**

- Step is informational only
- You want to generate a report even if it fails
- Multiple checks and you want all results

---

### 9. GitHub Context Variables

Access workflow and environment information:

```yaml
- name: Write to job summary
  run: echo "✅ Build passed" >> $GITHUB_STEP_SUMMARY

- name: Save output for next step
  id: my-step
  run: echo "status=success" >> $GITHUB_OUTPUT

- name: Use output from previous step
  run: echo "Previous status: ${{ steps.my-step.outputs.status }}"

- name: Access GitHub context
  run: |
    echo "Repository: ${{ github.repository }}"
    echo "Branch: ${{ github.ref_name }}"
    echo "Commit: ${{ github.sha }}"
    echo "Actor: ${{ github.actor }}"
```

**Common variables:**
| Variable | Description |
|----------|-------------|
| `${{ github.repository }}` | Owner/repo name |
| `${{ github.ref_name }}` | Branch or tag name |
| `${{ github.sha }}` | Commit SHA |
| `${{ github.actor }}` | User who triggered workflow |
| `${{ github.run_id }}` | Unique run identifier |

---

## How Reusable Workflows Work

```
┌─────────────────────────────────────────┐
│  Calling Repository (backend-app)       │
│  .github/workflows/ci.yml               │
│  ├─ Defines: on: push, pull_request    │
│  └─ Calls: uses: shared-workflows/...  │
│             with: { inputs }            │
└─────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────┐
│  Shared Repository (shared-workflows)   │
│  .github/workflows/nodejs-ci.yml        │
│  ├─ Defines: on: workflow_call          │
│  ├─ Jobs: security, lint, test, build   │
│  └─ Returns: status & artifacts         │
└─────────────────────────────────────────┘
```

**Flow:**

1. Trigger (push/PR) activates calling workflow
2. Calling workflow references reusable workflow with `uses:`
3. Inputs and secrets are passed
4. Reusable workflow executes all its jobs
5. Results appear in the calling repository's Actions tab

---

## Best Practices

| Practice                         | Description                           |
| -------------------------------- | ------------------------------------- |
| ✅ Use `defaults:`               | Avoid repeating configuration         |
| ✅ Use `if:`                     | Skip expensive jobs conditionally     |
| ✅ Use `matrix:`                 | Test multiple versions/configurations |
| ✅ Set `fail-fast: false`        | Get comprehensive testing reports     |
| ✅ Upload artifacts              | Enable post-run analysis              |
| ✅ Use `retention-days`          | Manage storage costs                  |
| ✅ Use `continue-on-error: true` | Only for non-critical steps           |
| ✅ Use `secrets: inherit`        | Pass secrets to reusable workflows    |
| ✅ Add step descriptions         | Improve readability                   |
| ✅ Use `$GITHUB_STEP_SUMMARY`    | Create readable reports               |

---

## Troubleshooting

| Problem              | Solution                                                          |
| -------------------- | ----------------------------------------------------------------- |
| Jobs not running?    | Check `if:` conditions and verify input values                    |
| Workflow timeout?    | Consider parallel execution (remove `needs:`), reduce matrix size |
| Secrets not working? | Verify `secrets: inherit` is set, check repo settings             |
| Matrix job skipped?  | Confirm `matrix-test: true` is passed in inputs                   |
| Artifacts missing?   | Check artifact paths match actual file locations                  |

---

## Setting Up

To use these workflows in your repositories:

1. Make this repository public (or configure `actions: read` permission)
2. Reference the workflow using the full path: `owner/repo/.github/workflows/file.yml@ref`
3. Pass required inputs in the `with:` section
4. Use `secrets: inherit` to pass repository secrets

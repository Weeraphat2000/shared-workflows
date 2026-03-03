# Shared Workflows

This repository contains **reusable GitHub Actions workflows** that can be called from other repositories.

## Available Workflows

### Node.js CI Pipeline

A reusable workflow for Node.js projects that includes:

- Install dependencies
- Run linting
- Run unit tests
- Generate coverage report

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
    secrets: inherit
```

#### Inputs

| Input               | Description                       | Required | Default |
| ------------------- | --------------------------------- | -------- | ------- |
| `node-version`      | Node.js version to use            | No       | `18`    |
| `working-directory` | Directory containing package.json | No       | `.`     |
| `run-lint`          | Whether to run linting            | No       | `true`  |
| `run-tests`         | Whether to run tests              | No       | `true`  |

## How Reusable Workflows Work

Reusable workflows allow you to define a workflow once and call it from multiple repositories. This helps:

1. **DRY Principle** - Don't repeat yourself across repos
2. **Consistency** - Same CI/CD process for all projects
3. **Maintainability** - Update workflow in one place

### Key Requirements

- The reusable workflow must be defined with `workflow_call` trigger
- The calling workflow uses `uses:` to reference the reusable workflow
- Format: `{owner}/{repo}/.github/workflows/{filename}@{ref}`

## Setting Up

To use these workflows in your repositories:

1. Make this repository public (or use `actions: read` permission)
2. Reference the workflow using the full path
3. Pass any required inputs and secrets
# shared-workflows

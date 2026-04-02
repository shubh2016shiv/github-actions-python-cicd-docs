# Python Continuous Integration Workflow Guide

This guide provides a comprehensive approach to implementing a robust Continuous Integration (CI) workflow for Python projects. The focus is on ensuring code quality and testing without deployment, making it suitable for personal projects or team collaborations where quality gates are important before merging code.

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Overview](#overview)
3. [CI Workflow Architecture](#ci-workflow-architecture)
4. [Quality Gates](#quality-gates)
5. [Workflow Implementation](#workflow-implementation)
6. [Unified Configuration with pyproject.toml](#unified-configuration-with-pyprojecttoml)
7. [Local Development Integration](#local-development-integration)
8. [Customization Guide](#customization-guide)
9. [Cross-Platform Support](#cross-platform-support)
10. [Handling Special Project Types](#handling-special-project-types)
11. [Troubleshooting](#troubleshooting)
12. [Alternative CI Systems](#alternative-ci-systems)

---

## Prerequisites

Before implementing this CI workflow, ensure you have:

- **Python 3.9+** installed locally (for testing configurations)
- **Git** installed and a GitHub repository set up
- **Basic familiarity** with Python packaging (pip, requirements.txt, or pyproject.toml)
- **A GitHub account** with repository admin access (for enabling Actions)

> **Note:** This guide covers CI only (code quality + testing). For Continuous Deployment (CD), see the companion guide on deployment pipelines.

**Estimated implementation time:** 30-60 minutes for a standard project.

---

## Overview

Continuous Integration for Python projects involves automatically checking code quality, running tests, and validating that new changes meet project standards before they are merged. This workflow focuses on:

- Code style and quality enforcement
- Static type checking
- Security vulnerability scanning
- Comprehensive test execution
- Dependency management
- Documentation quality

### Who This Guide Is For

- Individual developers wanting to professionalize their Python projects
- Teams establishing consistent code quality standards
- Organizations migrating from manual code review to automated quality gates

### Benefits

- Catch issues early in the development cycle (shift-left testing)
- Maintain consistent code quality across the project
- Prevent security vulnerabilities from being introduced
- Ensure test coverage for new features and changes
- Automate repetitive code quality tasks
- Reduce code review burden by catching style/formatting issues automatically

---

## CI Workflow Architecture

The CI workflow follows this process, designed with a **fail-fast** philosophy:

```mermaid
graph TD
    A[Code Push/PR] --> B[Checkout Code]
    B --> C[Setup Python Environment]
    C --> D[Install Dependencies]
    D --> E[Code Quality Checks]
    E --> F[Security Scanning]
    F --> G[Run Tests]
    G --> H[Generate Reports]
    H --> I{Quality Gates Pass?}
    I -->|Yes| J[✅ Merge/Commit Allowed]
    I -->|No| K[❌ Block Merge/Commit]
    
    subgraph "Code Quality Checks"
        E1[Linting - flake8]
        E2[Type Checking - mypy]
        E3[Formatting - black]
        E4[Import Sorting - isort]
        E5[Docstring - pydocstyle]
    end
    
    subgraph "Security & Dependencies"
        F1[Security Scan - bandit]
        F2[Dependency Scan - pip-audit]
    end
    
    subgraph "Testing"
        G1[Unit Tests - pytest]
        G2[Coverage Analysis]
        G3[Test Reports]
    end

    classDef critical fill:#f96,stroke:#333,stroke-width:2px;
    classDef success fill:#9f6,stroke:#333,stroke-width:1px;
    classDef failure fill:#f66,stroke:#333,stroke-width:1px;
    
    class I critical;
    class J success;
    class K failure;
```

### Design Rationale

The pipeline order is intentional:

1. **Fast-fail first:** Formatting and linting checks (black, isort, flake8) execute before tests because they are fast and catch issues that would cause tests to fail anyway. This saves CI minutes.
2. **Security in parallel:** Security scanning (bandit, pip-audit) can run in parallel with quality checks since they are independent.
3. **Tests last:** Tests are the slowest step and depend on code passing quality gates, so they run after.
4. **Coverage as final gate:** Coverage threshold enforcement runs after tests complete, using the generated report.

### Parallelization Strategy

For larger projects, consider splitting the workflow into separate jobs:

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps: [black, isort, flake8, mypy, pydocstyle]
    
  security:
    runs-on: ubuntu-latest
    steps: [bandit, pip-audit]
    
  test:
    needs: [lint, security]
    runs-on: ubuntu-latest
    steps: [pytest, coverage]
```

This allows lint and security checks to run concurrently, reducing total CI time.

---

## Quality Gates

Quality gates are predefined thresholds that code must pass before being accepted. These gates ensure that only high-quality code makes it into your repository.

| Gate | Tool | Default Threshold | Description |
|------|------|-------------------|-------------|
| Code Style | flake8 | 0 errors | Enforces PEP 8 style guide |
| Type Checking | mypy | 0 errors | Ensures proper type annotations |
| Code Formatting | black | 0 formatting errors | Maintains consistent code style |
| Import Sorting | isort | 0 sorting errors | Organizes imports consistently |
| Security | bandit | 0 high/critical issues | Identifies security vulnerabilities in code |
| Dependency Security | pip-audit | 0 known vulnerabilities | Checks dependencies against vulnerability databases |
| Test Coverage | pytest-cov | 80% minimum | Ensures adequate test coverage |
| Documentation | pydocstyle | 0 errors | Validates docstring quality |

### Choosing Thresholds

- **New projects:** Start with 80% coverage and tighten over time
- **Legacy projects:** Begin at 50-60% and increase by 5% per sprint
- **Critical systems:** Consider 90%+ coverage with branch coverage enabled
- **Security:** Always enforce 0 high/critical issues; low/medium can be warnings initially

### Understanding Bandit Severity Levels

- **HIGH/CRITICAL:** Immediate security risks (e.g., SQL injection, hardcoded passwords). Must fail the build.
- **MEDIUM:** Potential issues requiring review (e.g., use of `eval()`). Consider failing.
- **LOW:** Informational (e.g., debug statements). Can be warnings.

---

## Workflow Implementation

Below is the complete GitHub Actions workflow file that implements the CI pipeline. Save this as `.github/workflows/python-ci.yml`.

```yaml
name: Python CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]
  workflow_dispatch:

env:
  PYTHON_VERSION: "3.11"
  SRC_DIR: "src"
  TEST_DIR: "tests"

jobs:
  quality:
    name: Code Quality & Tests
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        python-version: ["3.9", "3.10", "3.11", "3.12"]
        os: [ubuntu-latest, macos-latest, windows-latest]
        exclude:
          # Skip macOS and Windows for non-primary Python versions to reduce CI time
          - python-version: "3.9"
            os: macos-latest
          - python-version: "3.9"
            os: windows-latest
          - python-version: "3.10"
            os: macos-latest
          - python-version: "3.10"
            os: windows-latest
      fail-fast: false

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
          cache: 'pip'

      - name: Install system dependencies (Ubuntu)
        if: runner.os == 'Linux'
        run: |
          sudo apt-get update
          sudo apt-get install -y build-essential libffi-dev libssl-dev

      - name: Install system dependencies (macOS)
        if: runner.os == 'macOS'
        run: |
          brew install openssl readline

      - name: Detect project structure
        id: detect
        shell: bash
        run: |
          if [ -d "src" ]; then
            echo "src_dir=src" >> $GITHUB_OUTPUT
          elif [ -d "app" ]; then
            echo "src_dir=app" >> $GITHUB_OUTPUT
          else
            echo "src_dir=." >> $GITHUB_OUTPUT
          fi
          
          if [ -d "tests" ]; then
            echo "test_dir=tests" >> $GITHUB_OUTPUT
          elif [ -d "test" ]; then
            echo "test_dir=test" >> $GITHUB_OUTPUT
          else
            echo "test_dir=." >> $GITHUB_OUTPUT
          fi

      - name: Install dependencies
        shell: bash
        run: |
          python -m pip install --upgrade pip
          
          if [ -f "pyproject.toml" ]; then
            if grep -q "\[tool.poetry" pyproject.toml; then
              pip install poetry
              poetry install --with dev
            elif grep -q "\[project\]" pyproject.toml; then
              pip install -e ".[dev,test]"
            else
              pip install -e .
            fi
          elif [ -f "requirements.txt" ]; then
            pip install -r requirements.txt
            if [ -f "requirements-dev.txt" ]; then
              pip install -r requirements-dev.txt
            fi
            if [ -f "requirements-test.txt" ]; then
              pip install -r requirements-test.txt
            fi
          elif [ -f "setup.py" ]; then
            pip install -e ".[dev,test]"
          fi
          
          pip install pytest pytest-cov black isort flake8 mypy pydocstyle bandit pip-audit

      - name: Check code formatting
        shell: bash
        run: |
          black --check .
          isort --check .

      - name: Lint with flake8
        shell: bash
        run: |
          SRC="${{ steps.detect.outputs.src_dir }}"
          TST="${{ steps.detect.outputs.test_dir }}"
          
          flake8 $SRC $TST --count --select=E9,F63,F7,F82 --show-source --statistics
          flake8 $SRC $TST --count --exit-zero --max-complexity=10 --max-line-length=88 --statistics

      - name: Type check with mypy
        shell: bash
        run: |
          SRC="${{ steps.detect.outputs.src_dir }}"
          TST="${{ steps.detect.outputs.test_dir }}"
          mypy $SRC $TST

      - name: Check docstrings with pydocstyle
        shell: bash
        run: |
          SRC="${{ steps.detect.outputs.src_dir }}"
          pydocstyle $SRC

      - name: Security scan with bandit
        shell: bash
        run: |
          SRC="${{ steps.detect.outputs.src_dir }}"
          if [ -f "pyproject.toml" ]; then
            bandit -r $SRC -c pyproject.toml -ll
          else
            bandit -r $SRC -ll
          fi

      - name: Check dependencies for vulnerabilities
        shell: bash
        run: |
          pip-audit --exit-code 1

      - name: Run tests with pytest
        shell: bash
        run: |
          SRC="${{ steps.detect.outputs.src_dir }}"
          mkdir -p test-results
          pytest --cov=$SRC \
            --cov-report=xml:coverage.xml \
            --cov-report=term \
            --junitxml=test-results/test-results.xml \
            --basetemp=test-results/tmp

      - name: Upload coverage report
        uses: codecov/codecov-action@v4
        with:
          file: ./coverage.xml
          fail_ci_if_error: false
          token: ${{ secrets.CODECOV_TOKEN }}

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results-py${{ matrix.python-version }}-${{ matrix.os }}
          path: test-results/
          retention-days: 14

      - name: Check coverage threshold
        shell: bash
        run: |
          COVERAGE=$(python -c "
          import xml.etree.ElementTree as ET
          tree = ET.parse('coverage.xml')
          root = tree.getroot()
          print(float(root.attrib['line-rate']) * 100)
          ")
          echo "Coverage: ${COVERAGE}%"
          python -c "
          import sys
          coverage = float('${COVERAGE}')
          threshold = 80.0
          if coverage < threshold:
              print(f'Coverage {coverage}% is below threshold of {threshold}%')
              sys.exit(1)
          print(f'Coverage {coverage}% meets threshold of {threshold}%')
          "
    done
```

### Projects with C Extensions

```yaml
- name: Install build dependencies
  if: runner.os == 'Linux'
  run: sudo apt-get install -y build-essential python3-dev

- name: Install build dependencies (macOS)
  if: runner.os == 'macOS'
  run: brew install python

- name: Build and test C extensions
  run: |
    pip install -e .
    pytest tests/
```

---

## Troubleshooting

### Reading GitHub Actions Logs

1. Navigate to **Actions** tab in your repository
2. Click on a workflow run
3. Click on a job to see step-by-step logs
4. Expand individual steps to see detailed output
5. Look for lines marked with `##[error]` for failures

### Common Issues and Solutions

#### 1. Workflow Not Triggering

**Issue:** Pushing code doesn't start the workflow.
**Solution:**
- Verify the workflow file is in `.github/workflows/` directory
- Check the `on:` triggers match your branch names
- Ensure the file has a `.yml` or `.yaml` extension
- Check Actions are enabled in repository Settings

#### 2. Missing Dependencies

**Issue:** Tests fail with `ModuleNotFoundError`.
**Solution:**
- Verify all dependencies are in `requirements.txt` or `pyproject.toml`
- Check that optional dependency groups are installed: `pip install -e ".[dev,test]"`
- For Poetry: ensure `poetry install --with dev,test` includes all groups

#### 3. Platform-Specific Failures

**Issue:** Tests pass on Ubuntu but fail on Windows.
**Solution:**
- Check for hardcoded path separators (use `pathlib`)
- Verify file encoding is explicitly set to UTF-8
- Check for case-sensitive file imports
- Use `shell: bash` for consistent shell behavior across platforms

#### 4. Coverage Threshold Failures

**Issue:** Coverage check fails despite adequate testing.
**Solution:**
- Verify the `--cov` path matches your source directory
- Check that `coverage.xml` is being generated correctly
- Temporarily lower the threshold to identify gaps
- Use `--cov-report=term-missing` to see uncovered lines

#### 5. Mypy Type Checking Errors

**Issue:** Mypy reports errors in third-party libraries.
**Solution:**
```toml
[tool.mypy]
ignore_missing_imports = true  # Skip type checking for untyped third-party libs

# Or be specific:
[[tool.mypy.overrides]]
module = ["numpy.*", "pandas.*"]
ignore_missing_imports = true
```

#### 6. Workflow Timeout

**Issue:** CI runs take too long and time out.
**Solution:**
- Use the matrix `exclude` strategy to reduce job count
- Enable dependency caching (`cache: 'pip'`)
- Split into separate jobs (lint, security, test) for parallelism
- Set `fail-fast: true` if you want to stop on first failure

#### 7. Action Version Conflicts

**Issue:** Workflow fails with action compatibility errors.
**Solution:**
- Always use stable version tags (`@v4`, not `@main`)
- Check the action's marketplace page for latest version
- Run `actions/checkout` before any other action that needs code

#### 8. Permission Errors

**Issue:** Workflow fails with "Resource not accessible by integration".
**Solution:**
- Add permissions to the workflow:
```yaml
permissions:
  contents: read
  pull-requests: write
```
- For Codecov, add the `CODECOV_TOKEN` secret to repository settings

#### 9. Caching Not Working

**Issue:** Dependencies reinstall on every run.
**Solution:**
- Verify the cache key includes the lock file hash:
```yaml
- uses: actions/setup-python@v5
  with:
    python-version: "3.11"
    cache: 'pip'
    cache-dependency-path: '**/requirements*.txt'
```

#### 10. Secret Not Available

**Issue:** `${{ secrets.MY_SECRET }}` is empty.
**Solution:**
- Add the secret in Settings > Secrets and variables > Actions
- Secret names are case-sensitive
- Secrets are not available in workflows triggered by forks (use `pull_request_target` with caution)

---

## Alternative CI Systems

### GitLab CI

```yaml
# .gitlab-ci.yml
image: python:3.11

stages:
  - lint
  - security
  - test

variables:
  PIP_CACHE_DIR: "$CI_PROJECT_DIR/.cache/pip"

cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - .cache/pip
    - .venv/

before_script:
  - python -V
  - pip install --upgrade pip
  - pip install -r requirements.txt
  - pip install -r requirements-dev.txt

lint:
  stage: lint
  script:
    - black --check .
    - isort --check .
    - flake8 src tests
    - mypy src
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'

security:
  stage: security
  script:
    - bandit -r src -ll
    - pip-audit
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

test:
  stage: test
  script:
    - pytest --cov=src --cov-report=xml --junitxml=report.xml
  coverage: '/TOTAL.*\s+(\d+%)/'
  artifacts:
    when: always
    reports:
      junit: report.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
```

### CircleCI

```yaml
# .circleci/config.yml
version: 2.1

orbs:
  python: circleci/python@2.1

jobs:
  lint-and-test:
    executor: python/default
    steps:
      - checkout
      - python/install-packages:
          pkg-manager: pip
      - run:
          name: Run linters
          command: |
            black --check .
            flake8 src tests
            mypy src
      - run:
          name: Run tests
          command: |
            pytest --cov=src --cov-report=xml --junitxml=test-results/junit.xml
      - store_test_results:
          path: test-results
      - store_artifacts:
          path: coverage.xml

workflows:
  main:
    jobs:
      - lint-and-test:
          filters:
            branches:
              only: [main, develop]
```

### Azure DevOps Pipelines

```yaml
# azure-pipelines.yml
trigger:
  - main
  - develop

pool:
  vmImage: 'ubuntu-latest'

steps:
- task: UsePythonVersion@0
  inputs:
    versionSpec: '3.11'
    addToPath: true

- script: |
    python -m pip install --upgrade pip
    pip install -r requirements.txt
    pip install -r requirements-dev.txt
  displayName: 'Install dependencies'

- script: |
    black --check .
    flake8 src tests
    mypy src
  displayName: 'Run linters'

- script: |
    pytest --cov=src --cov-report=xml --junitxml=test-results.xml
  displayName: 'Run tests'

- task: PublishTestResults@2
  inputs:
    testResultsFiles: 'test-results.xml'
    testRunTitle: 'Python Tests'
  condition: succeededOrFailed()

- task: PublishCodeCoverageResults@1
  inputs:
    codeCoverageTool: Cobertura
    summaryFileLocation: 'coverage.xml'
```

### Migration Checklist: GitHub Actions to Other Platforms

| GitHub Actions Concept | GitLab CI | CircleCI | Azure DevOps |
|------------------------|-----------|----------|--------------|
| `.github/workflows/*.yml` | `.gitlab-ci.yml` | `.circleci/config.yml` | `azure-pipelines.yml` |
| `on: push/pull_request` | `rules` / `only` | `filters` | `trigger` |
| `jobs` | `stages` + jobs | `jobs` | `steps` |
| `runs-on` | `image` / `tags` | `executor` | `pool.vmImage` |
| `matrix` | `parallel` | `matrix` | Not directly supported |
| `uses:` | N/A (use images) | `orbs` | `task:` |
| `secrets` | CI/CD Variables | Context | Variable groups |
| `artifacts` | `artifacts` | `store_artifacts` | `PublishBuildArtifacts` |
| `cache` | `cache` | `save_cache` | `Cache@2` |

---

## Conclusion

This CI workflow provides a robust foundation for ensuring code quality in Python projects. By implementing these quality gates, you can maintain high standards in your codebase while still allowing for flexibility and customization based on project needs.

### Key Features

- Cross-platform testing (Windows, macOS, Linux) with optimized matrix strategy
- Auto-detection of dependency management systems (setuptools, pip, Poetry)
- Intelligent source directory detection for various project structures
- Unified `pyproject.toml` configuration support
- Comprehensive security scanning (bandit + pip-audit)
- Pre-commit hooks for local development parity
- Guidance for Django, Flask, FastAPI, data science, and C extension projects

### Best Practices Summary

1. **Fail fast:** Put quick checks (linting) before slow checks (tests)
2. **Cache aggressively:** Cache dependencies to speed up CI
3. **Local parity:** Use pre-commit to catch issues before CI
4. **Start lenient:** Begin with relaxed thresholds and tighten over time
5. **Monitor trends:** Use coverage tracking to ensure quality improves
6. **Keep actions updated:** Pin to major versions and update regularly
7. **Document exceptions:** If a check must be skipped, document why

Remember that CI is most effective when it's part of a larger development culture that values quality and testing. Use these tools to support good development practices, not replace them.

# CI/CD Pipeline for Python GenAI Projects: A Beginner's Guide

This guide walks you through creating a complete CI/CD (Continuous Integration/Continuous Deployment) pipeline for a Python-based Generative AI project using GitHub Actions. We start with fundamentals and build toward a fully automated workflow that tests, builds, and deploys your application securely.

## What is CI/CD?

- **Continuous Integration (CI)** is the practice of frequently merging code changes into a central repository, with automated builds and tests running on each change. The goal is to catch integration bugs early and maintain code quality.
- **Continuous Delivery** ensures that code that passes CI is automatically prepared for release to production. A human approves the deployment.
- **Continuous Deployment (CD)** goes further: every change that passes all stages is automatically deployed to production without manual intervention.

### Why CI/CD Matters for GenAI Projects

Generative AI projects have unique challenges that CI/CD addresses:
- **Complex dependencies**: ML frameworks (PyTorch, TensorFlow) have large, version-sensitive requirements
- **API key management**: GenAI apps require secure handling of LLM provider credentials
- **Model reproducibility**: Automated pipelines ensure consistent environments for inference
- **Rapid iteration**: Fast feedback loops accelerate prompt engineering and model tuning
- **Security**: Automated scanning catches vulnerabilities in dependencies and containers

## Prerequisites

Before starting, ensure you have:

1. A Python project hosted in a GitHub repository
2. A `requirements.txt` or `pyproject.toml` file listing dependencies
3. Basic familiarity with Git and GitHub
4. An Azure account (or other cloud provider) for deployment
5. Azure CLI installed locally (for credential setup)
6. A container registry (Azure Container Registry, Docker Hub, or GitHub Container Registry)
7. (Optional) Docker installed locally for testing

## Table of Contents

1. [What is CI/CD?](#what-is-cicd)
2. [Prerequisites](#prerequisites)
3. [The Building Blocks: GitHub Actions](#the-building-blocks-github-actions)
4. [Step 1: Create Your First Workflow File](#step-1-create-your-first-workflow-file)
5. [Step 2: Continuous Integration (CI) - Testing Your Code](#step-2-continuous-integration-ci---testing-your-code)
6. [Step 3: Containerize Your Application with Docker](#step-3-containerize-your-application-with-docker)
7. [Step 4: Continuous Deployment (CD) - Build and Deploy](#step-4-continuous-deployment-cd---build-and-deploy)
8. [Step 5: Enhancing Your Pipeline with Best Practices](#step-5-enhancing-your-pipeline-with-best-practices)
9. [Step 6: The Complete Workflow](#step-6-the-complete-workflow)
10. [Troubleshooting Common Issues](#troubleshooting-common-issues)
11. [Next Steps](#next-steps)

---

## The Building Blocks: GitHub Actions

We use GitHub Actions to build our pipeline. Key concepts:

- **Workflow**: An automated process defined in a YAML file. Represents your entire CI/CD pipeline.
- **Event**: An activity that triggers a workflow (e.g., `push`, `pull_request`, `workflow_dispatch`).
- **Job**: A set of steps executing on the same runner. Workflows can contain multiple jobs that run in parallel or sequentially.
- **Step**: An individual task that runs commands or uses an action.
- **Action**: Reusable code that performs common tasks (e.g., `actions/checkout` to get your code).
- **Runner**: A server that executes workflows. GitHub provides hosted runners (`ubuntu-latest`, `windows-latest`, `macos-latest`).
- **Artifacts**: Files preserved from workflow runs for later jobs or downloads.
- **Secrets**: Encrypted environment variables for sensitive data (API keys, credentials).
- **Environments**: Named deployment targets (dev, staging, prod) with protection rules and environment-specific secrets.

All workflows live in the `.github/workflows/` directory.

---

## Step 1: Create Your First Workflow File

Create the workflow directory structure:

**Linux/macOS:**
```bash
mkdir -p .github/workflows
touch .github/workflows/python-cicd.yml
```

**Windows (PowerShell):**
```powershell
New-Item -ItemType Directory -Force -Path .github/workflows
New-Item -ItemType File -Path .github/workflows/python-cicd.yml
```

---

## Step 2: Continuous Integration (CI) - Testing Your Code

The CI pipeline ensures every code change is automatically tested. Add this to `python-cicd.yml`:

```yaml
name: Python CI/CD Pipeline

on:
  push:
    branches: ["main"]
  pull_request:
    branches: ["main"]

jobs:
  test:
    name: Run Tests and Linting
    runs-on: ubuntu-latest
    permissions:
      contents: read

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"
          cache: "pip"

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install flake8 pytest pytest-cov

      - name: Lint with flake8
        run: |
          flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
          flake8 . --count --exit-zero --max-complexity=10 --max-line-length=127 --statistics

      - name: Test with pytest
        run: |
          pytest --cov=src --cov-report=xml --cov-report=term
```

### What does this do?

1. **Triggers**: Runs on pushes and pull requests to `main`
2. **Job `test`**: Executes on a fresh Ubuntu runner
3. **Checkout**: Downloads your repository code
4. **Set up Python**: Installs Python 3.11 with automatic pip caching
5. **Install Dependencies**: Installs project requirements plus testing tools
6. **Lint**: Runs flake8 to catch syntax errors and style issues
7. **Test**: Runs pytest with coverage reporting

> **GenAI Tip**: If your tests call external LLM APIs, mock them using `pytest-mock` or `responses` to avoid API costs and ensure deterministic tests.

---

## Step 3: Containerize Your Application with Docker

GenAI projects often have complex dependencies. Docker ensures your application runs identically everywhere.

### 1. Create a `Dockerfile`

```dockerfile
# Use a slim Python base image for smaller attack surface
FROM python:3.11-slim

# Set working directory
WORKDIR /app

# Install system dependencies needed by common ML packages
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Copy and install Python dependencies first (Docker layer caching)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Create non-root user for security
RUN useradd --create-home appuser && chown -R appuser:appuser /app
USER appuser

# Expose port for web applications
EXPOSE 8000

# Run the application
CMD ["uvicorn", "src.app:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 2. Create a `.dockerignore` File

```text
__pycache__
*.pyc
.env
.env.*
.git
.gitignore
.venv
venv
*.md
tests/
.pytest_cache/
coverage.xml
```

> **Security Note**: Never bake API keys or `.env` files into Docker images. Use environment variables or secret management at runtime.

---

## Step 4: Continuous Deployment (CD) - Build and Deploy

### Configure Secure Authentication with OIDC

Instead of storing long-lived credentials, use OpenID Connect (OIDC) for passwordless authentication:

1. In Azure Portal, create an App Registration
2. Add a federated credential with:
   - **Issuer**: `https://token.actions.githubusercontent.com`
   - **Subject**: `repo:your-username/your-repo:ref:refs/heads/main`
3. Store the Azure Client ID, Tenant ID, and Subscription ID as GitHub secrets:
   - `AZURE_CLIENT_ID`
   - `AZURE_TENANT_ID`
   - `AZURE_SUBSCRIPTION_ID`

### Add the Deployment Job

```yaml
jobs:
  test:
    # ... (test job from Step 2)

  build_and_deploy:
    name: Build and Deploy to Azure
    runs-on: ubuntu-latest
    needs: test
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    permissions:
      contents: read
      id-token: write  # Required for OIDC
      packages: write  # Required for container registry

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Log in to Azure (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Log in to Azure Container Registry
        uses: azure/cli@v2
        with:
          azcliversion: latest
          inlineScript: |
            az acr login --name your-registry-name

      - name: Build and Push Docker Image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: your-registry-name.azurecr.io/your-app-name:${{ github.sha }}

      - name: Scan Docker image for vulnerabilities
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: your-registry-name.azurecr.io/your-app-name:${{ github.sha }}
          format: "table"
          exit-code: "1"
          ignore-unfixed: true
          severity: "CRITICAL,HIGH"

      - name: Deploy to Azure Web App
        uses: azure/webapps-deploy@v3
        with:
          app-name: "your-azure-web-app-name"
          images: "your-registry-name.azurecr.io/your-app-name:${{ github.sha }}"
```

### What does this do?

1. **`needs: test`**: Deployment only runs if tests pass
2. **`if` condition**: Only deploys on pushes to `main`, not pull requests
3. **OIDC Login**: Passwordless Azure authentication (more secure than static credentials)
4. **Build and Push**: Creates Docker image tagged with commit SHA for traceability
5. **Security Scan**: Trivy scans for vulnerabilities before deployment proceeds
6. **Deploy**: Updates Azure Web App to use the new container image

---

## Step 5: Enhancing Your Pipeline with Best Practices

### 5.1 Add Dependency Caching

GitHub Actions supports built-in caching. The `setup-python` action already caches pip when you set `cache: "pip"`. For additional caching:

```yaml
- name: Cache pytest cache
  uses: actions/cache@v4
  with:
    path: .pytest_cache
    key: ${{ runner.os }}-pytest-cache-${{ hashFiles('**/requirements.txt') }}
    restore-keys: |
      ${{ runner.os }}-pytest-cache-
```

### 5.2 Add Test Coverage Reporting

```yaml
- name: Test with pytest and coverage
  run: |
    pytest --cov=src --cov-report=xml --cov-report=term --junitxml=test-results.xml

- name: Upload coverage to Codecov
  uses: codecov/codecov-action@v4
  with:
    file: ./coverage.xml
    fail_ci_if_error: false
    token: ${{ secrets.CODECOV_TOKEN }}
```

### 5.3 Add Security Scanning

**Dependency scanning:**
```yaml
- name: Scan dependencies for vulnerabilities
  run: |
    pip install safety
    safety check --full-report || echo "Safety check completed with warnings"
```

**Container scanning (before push):**
```yaml
- name: Build Docker image for scanning
  uses: docker/build-push-action@v6
  with:
    context: .
    push: false
    load: true
    tags: local-scan:${{ github.sha }}

- name: Scan image before push
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: local-scan:${{ github.sha }}
    format: "table"
    exit-code: "1"
    ignore-unfixed: true
    severity: "CRITICAL,HIGH"
```

### 5.4 Add Cross-Platform Testing

```yaml
test:
  name: Run Tests on ${{ matrix.os }} / Python ${{ matrix.python-version }}
  runs-on: ${{ matrix.os }}
  strategy:
    matrix:
      os: [ubuntu-latest, windows-latest]
      python-version: ["3.10", "3.11"]
    fail-fast: false
  permissions:
    contents: read

  steps:
    - name: Checkout repository
      uses: actions/checkout@v4

    - name: Set up Python
      uses: actions/setup-python@v5
      with:
        python-version: ${{ matrix.python-version }}
        cache: "pip"

    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
        pip install flake8 pytest pytest-cov

    - name: Lint with flake8
      run: flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics

    - name: Test with pytest
      run: pytest --cov=src --cov-report=xml
```

### 5.5 Add Environment Protection Rules

Use GitHub Environments to control deployments:

```yaml
deploy-staging:
  name: Deploy to Staging
  runs-on: ubuntu-latest
  needs: test
  environment:
    name: staging
    url: https://your-staging-app.azurewebsites.net
  if: github.event_name == 'push' && github.ref == 'refs/heads/main'
  # ... deployment steps

deploy-production:
  name: Deploy to Production
  runs-on: ubuntu-latest
  needs: deploy-staging
  environment:
    name: production
    url: https://your-prod-app.azurewebsites.net
  if: github.event_name == 'push' && github.ref == 'refs/heads/main'
  # ... deployment steps
```

Configure environment protection rules in **Settings > Environments** to require manual approval for production deployments.

### 5.6 Add Workflow Status Notifications

```yaml
- name: Notify on failure
  if: failure() && github.event_name == 'pull_request'
  uses: actions/github-script@v7
  with:
    script: |
      github.rest.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: `Workflow failed: ${context.job}`
      })
```

> **Note**: The `context.issue.number` is only available for pull request events. For push events, consider using Slack or email notifications instead.

### 5.7 Add Concurrency Control

Prevent multiple workflows from running simultaneously on the same branch:

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

---

## Step 6: The Complete Workflow

Here is the complete, production-ready `python-cicd.yml`:

```yaml
name: Python CI/CD Pipeline

on:
  push:
    branches: ["main"]
  pull_request:
    branches: ["main"]
  workflow_dispatch:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  test:
    name: Run Tests on ${{ matrix.os }} / Python ${{ matrix.python-version }}
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest]
        python-version: ["3.10", "3.11"]
      fail-fast: false
    permissions:
      contents: read

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
          cache: "pip"

      - name: Install system dependencies
        if: runner.os == 'Linux'
        run: |
          sudo apt-get update
          sudo apt-get install -y build-essential libffi-dev

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install flake8 pytest pytest-cov

      - name: Lint with flake8
        run: |
          flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
          flake8 . --count --exit-zero --max-complexity=10 --max-line-length=127 --statistics

      - name: Scan dependencies for vulnerabilities
        run: |
          pip install safety
          safety check --full-report || echo "Safety check completed with warnings"

      - name: Test with pytest
        run: |
          pytest --cov=src --cov-report=xml --cov-report=term --junitxml=test-results.xml

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          file: ./coverage.xml
          fail_ci_if_error: false

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results-${{ matrix.os }}-${{ matrix.python-version }}
          path: test-results.xml
          retention-days: 7

  build_and_deploy:
    name: Build and Deploy to Azure
    runs-on: ubuntu-latest
    needs: test
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    permissions:
      contents: read
      id-token: write
      packages: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Log in to Azure (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Log in to Azure Container Registry
        uses: azure/cli@v2
        with:
          azcliversion: latest
          inlineScript: |
            az acr login --name your-registry-name

      - name: Build Docker image for scanning
        uses: docker/build-push-action@v6
        with:
          context: .
          push: false
          load: true
          tags: local-scan:${{ github.sha }}

      - name: Scan image before push
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: local-scan:${{ github.sha }}
          format: "table"
          exit-code: "1"
          ignore-unfixed: true
          severity: "CRITICAL,HIGH"

      - name: Build and Push Docker Image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: your-registry-name.azurecr.io/your-app-name:${{ github.sha }}

      - name: Deploy to Azure Web App
        uses: azure/webapps-deploy@v3
        with:
          app-name: "your-azure-web-app-name"
          images: "your-registry-name.azurecr.io/your-app-name:${{ github.sha }}"
```

> **Remember**: Replace `your-registry-name`, `your-app-name`, and `your-azure-web-app-name` with your actual resource names.

---

## Troubleshooting Common Issues

### 1. Dependency Installation Failures

**Symptom**: Errors during `pip install -r requirements.txt`

**Solutions**:
- Verify `requirements.txt` formatting (one package per line)
- Pin exact versions to avoid conflicts: `package==1.2.3`
- Install system packages for native dependencies:
  ```yaml
  - name: Install system dependencies
    if: runner.os == 'Linux'
    run: |
      sudo apt-get update
      sudo apt-get install -y build-essential libffi-dev libssl-dev
  ```

### 2. Docker Build Failures

**Symptom**: Errors during `docker/build-push-action`

**Solutions**:
- Validate Dockerfile syntax with `docker build .` locally
- Ensure all referenced files exist
- For GenAI projects with large models:
  - Use `.dockerignore` to exclude model files
  - Download models at runtime from external storage
  - Consider multi-stage builds to reduce image size

### 3. Azure Deployment Failures

**Symptom**: Errors during `azure/webapps-deploy`

**Solutions**:
- Verify Azure Web App name matches exactly
- Ensure service principal has Contributor role on the Web App
- Confirm ACR is accessible (check network restrictions)
- Verify Web App is configured for container deployment
- Check OIDC federated credentials match your repository

### 4. Tests Pass Locally but Fail in CI

**Symptom**: Inconsistent test results

**Solutions**:
- Lock dependency versions: `pip freeze > requirements.txt`
- Match Python versions between local and CI
- Set environment variables explicitly:
  ```yaml
  env:
    PYTHONPATH: src
    ENVIRONMENT: test
  ```
- Mock external API calls in tests

### 5. OIDC Authentication Failures

**Symptom**: Azure login fails with credential errors

**Solutions**:
- Verify federated credential subject matches: `repo:owner/repo:ref:refs/heads/main`
- Ensure `id-token: write` permission is set in the job
- Check that secrets are set at the repository level
- Verify Azure App Registration has appropriate role assignments

### 6. Secret Access Issues

**Symptom**: Secrets appear empty in workflow

**Solutions**:
- Confirm secret names match exactly (case-sensitive)
- Check secrets are set in **Settings > Secrets and variables > Actions**
- Verify fork pull requests cannot access secrets (GitHub security feature)
- For organization secrets, ensure repository access is granted

### 7. Workflow Doesn't Trigger

**Symptom**: No workflow runs on push/PR

**Solutions**:
- Confirm file is in `.github/workflows/` with `.yml` or `.yaml` extension
- Validate YAML syntax (use a YAML linter)
- Check branch names match trigger conditions
- Verify no branch protection rules block workflows
- Check workflow permissions in repository settings

---

## Next Steps

After mastering this foundation, explore these advanced topics:

- **Reusable Workflows**: Share CI/CD logic across repositories using `workflow_call`
- **Environments**: Configure dev, staging, and prod with approval gates and environment-specific secrets
- **Advanced Triggers**: Use `workflow_run`, `schedule`, or repository dispatch for complex automation
- **Performance Optimization**: Optimize with job matrices, advanced caching, and artifact management
- **LLMOps-Specific Patterns**:
  - Model versioning with MLflow or Weights & Biases
  - Git LFS for model artifacts
  - GPU-enabled runners for training workloads
  - Prompt testing and evaluation in CI
  - Data validation and drift detection
- **Security Enhancements**:
  - CodeQL static analysis
  - Dependency review with GitHub's dependency graph
  - SBOM generation with Syft
  - Secret scanning with gitleaks
- **Observability**:
  - Workflow metrics dashboards
  - Slack/Teams notifications
  - Deployment tracking and rollback automation

By mastering these concepts, you will create reliable, secure CI/CD pipelines that accelerate your Python GenAI applications while maintaining quality and safety.

Happy coding!

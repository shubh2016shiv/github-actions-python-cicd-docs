# CI/CD Pipeline for Python GenAI Projects: A Beginner's Guide

This guide will walk you through creating a complete CI/CD (Continuous Integration/Continuous Deployment) pipeline for a Python-based Generative AI project using GitHub Actions. We'll start with the basics and build up to a fully automated workflow that tests, builds, and deploys your application.

## What is CI/CD?

*   **Continuous Integration (CI)** is the practice of frequently merging code changes from all developers into a central repository. After each merge, an automated build and automated tests are run to ensure the new code doesn't break anything. The goal is to catch bugs early.
*   **Continuous Deployment (CD)** is the next step after CI. It's the practice of automatically deploying every change that passes the CI stage to a production (or staging) environment. This makes your release process faster and more reliable.

## Prerequisites

Before we start, you should have:

1.  A Python project hosted in a GitHub repository.
2.  Your project should have a `requirements.txt` file listing its dependencies.
3.  Basic familiarity with Git and GitHub.
4.  An account with a cloud provider to deploy your application (we'll use Microsoft Azure as an example, since your project uses it).
5.  (Optional) Docker installed locally for testing (not required for GitHub Actions).

## Table of Contents

1.  [What is CI/CD?](#what-is-cicd)
2.  [Prerequisites](#prerequisites)
3.  [The Building Blocks: GitHub Actions](#the-building-blocks-github-actions)
4.  [Step 1: Create Your First Workflow File](#step-1-create-your-first-workflow-file)
5.  [Step 2: Continuous Integration (CI) - Testing Your Code](#step-2-continuous-integration-ci---testing-your-code)
6.  [Step 3: Containerize Your Application with Docker](#step-3-containerize-your-application-with-docker)
7.  [Step 4: Continuous Deployment (CD) - Build and Deploy](#step-4-continuous-deployment-cd---build-and-deploy)
8.  [Step 5: Enhancing Your Pipeline with Best Practices](#step-5-enhancing-your-pipeline-with-best-practices)
9.  [Step 6: The Complete Workflow](#step-6-the-complete-workflow)
10. [Troubleshooting Common Issues](#troubleshooting-common-issues)
11. [Next Steps](#next-steps)

---

## The Building Blocks: GitHub Actions

We will use GitHub Actions to build our pipeline. The key concepts are:

*   **Workflow**: An automated process defined by a YAML file. It's the entire CI/CD process.
*   **Event**: A specific activity that triggers a workflow (e.g., a `push` to a branch or a `pull_request`).
*   **Job**: A set of steps that execute on the same "runner" (a virtual machine). A workflow can have one or more jobs.
*   **Step**: An individual task that can run commands or an "action."
*   **Action**: A reusable piece of code that performs a complex but common task (e.g., `actions/checkout` to get your code, or `azure/login` to log into Azure).
*   **Runner**: A server that runs your workflows when they're triggered. GitHub provides hosted runners (ubuntu-latest, windows-latest, macos-latest) or you can use self-hosted runners.
*   **Artifacts**: Files created during a workflow run that you can preserve and use in later jobs or download.

All workflows are stored in the `.github/workflows/` directory in your repository.

---

## Step 1: Create Your First Workflow File

In your repository, create a directory named `.github` and inside it, another directory named `workflows`.

Inside `.github/workflows/`, create a new file named `python-cicd.yml`.

```bash
mkdir -p .github/workflows
touch .github/workflows/python-cicd.yml
```

Now, let's start building our workflow.

---

## Step 2: Continuous Integration (CI) - Testing Your Code

The first part of our pipeline will focus on CI. We want to ensure that every time new code is pushed or a pull request is created, it is automatically tested.

Add the following code to your `python-cicd.yml` file:

```yaml
name: Python CI/CD Pipeline

# 1. Define the triggers for this workflow
on:
  push:
    branches: [ "main" ] # Runs on pushes to the main branch
  pull_request:
    branches: [ "main" ] # Runs on pull requests targeting the main branch

jobs:
  # 2. Define the 'test' job
  test:
    name: Run Tests and Linting
    runs-on: ubuntu-latest # Use a fresh Ubuntu virtual machine

    steps:
      # 3. Step 1: Check out your code from the repository
      - name: Checkout repository
        uses: actions/checkout@v3

      # 4. Step 2: Set up a specific Python version
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      # 5. Step 3: Install project dependencies
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install flake8 pytest # Install testing tools

      # 6. Step 4: Run the linter (checks for code style issues)
      - name: Lint with flake8
        run: |
          # stop the build if there are Python syntax errors or undefined names
          flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
          # exit-zero treats all errors as warnings. The GitHub editor is 127 chars wide
          flake8 . --count --exit-zero --max-complexity=10 --max-line-length=127 --statistics

      # 7. Step 5: Run unit tests with pytest
      - name: Test with pytest
        run: |
          pytest
```

### What does this do?

1.  **Triggers**: The workflow runs on pushes and pull requests to the `main` branch.
2.  **Job `test`**: This job runs all our checks.
3.  **Checkout**: It downloads your repository's code onto the runner.
4.  **Set up Python**: It installs Python 3.11.
5.  **Install Dependencies**: It reads your `requirements.txt` to install all necessary libraries for your project, plus `flake8` for linting and `pytest` for testing.
6.  **Lint**: It runs `flake8` to check for style errors in your code. This helps maintain code quality.
7.  **Test**: It runs your unit tests using `pytest`. Make sure you have your tests in a `tests/` directory.

At this point, you have a solid CI pipeline!

---

## Step 3: Containerize Your Application with Docker

For GenAI projects with complex dependencies (like PyTorch, TensorFlow, etc.), it's best to package the application into a **Docker container**. A container ensures your application runs the same way everywhere.

### 1. Create a `Dockerfile`

Create a `Dockerfile` in the root of your project. This file contains instructions to build your Docker image.

```dockerfile
# Use an official Python runtime as a parent image
FROM python:3.11-slim

# Set the working directory in the container
WORKDIR /app

# Copy the requirements file into the container
COPY requirements.txt .

# Install any needed packages specified in requirements.txt
RUN pip install --no-cache-dir -r requirements.txt

# Copy the rest of your application's code
COPY . .

# Command to run your application (e.g., if you are using FastAPI)
# Replace this with the command to start your app
CMD ["uvicorn", "src.app:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 2. Create a `.dockerignore` File

To optimize your Docker build, create a `.dockerignore` file to exclude unnecessary files:

```dockerfile
# .dockerignore
__pycache__
*.pyc
.env
.git
.gitignore
README.md
```

---

## Step 4: Continuous Deployment (CD) - Build and Deploy

Now, let's extend our workflow to build the Docker image and deploy it to Azure. This will only happen on a successful push to the `main` branch, not on pull requests.

First, you need to store your Azure credentials securely as **GitHub Secrets**:

1.  Go to your GitHub repository -> `Settings` -> `Secrets and variables` -> `Actions`.
2.  Create the secrets needed for Azure login and your container registry. For example: `AZURE_CREDENTIALS`, `REGISTRY_USERNAME`, `REGISTRY_PASSWORD`. The names can be anything, but they must match what you use in the workflow.

Now, let's add the `build_and_deploy` job to `python-cicd.yml`:

```yaml
# ... (keep the 'on' and 'test' job from before)

jobs:
  test:
    # ... (the test job from Step 2 remains here)

  build_and_deploy:
    name: Build and Deploy to Azure
    runs-on: ubuntu-latest
    needs: test # This job will only run if the 'test' job succeeds
    if: github.event_name == 'push' # Only run this job on a push, not a pull request

    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      # Log in to your container registry (e.g., Azure Container Registry)
      - name: Log in to Azure Container Registry
        uses: docker/login-action@v2
        with:
          registry: your-registry-name.azurecr.io
          username: ${{ secrets.REGISTRY_USERNAME }}
          password: ${{ secrets.REGISTRY_PASSWORD }}

      # Build and push the Docker image
      - name: Build and Push Docker Image
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: your-registry-name.azurecr.io/your-app-name:${{ github.sha }}

      # Log in to Azure
      - name: Log in to Azure
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      # Deploy to Azure Web App
      - name: Deploy to Azure Web App
        uses: azure/webapps-deploy@v2
        with:
          app-name: 'your-azure-web-app-name'
          images: 'your-registry-name.azurecr.io/your-app-name:${{ github.sha }}'
```

### What does this new job do?

*   `needs: test`: This ensures that we only try to deploy if all the tests passed.
*   `if: github.event_name == 'push'`: This ensures deployment only happens when code is merged/pushed to `main`, not for every commit in a pull request.
*   **Login to Registry**: It securely logs into your Azure Container Registry using the secrets you stored.
*   **Build and Push**: It builds your Docker image using the `Dockerfile` and pushes it to your registry. The image is tagged with the unique commit SHA (`${{ github.sha }}`) for versioning.
*   **Login to Azure**: It logs into your Azure account.
*   **Deploy**: It tells your Azure Web App to pull and run the new Docker image you just pushed.

---

## Step 5: Enhancing Your Pipeline with Best Practices

Let's improve our pipeline with several DevOps best practices that will make it more robust, efficient, and secure.

### 5.1 Add Dependency Caching

To speed up your workflows, add dependency caching between runs:

```yaml
# Add this step after "Set up Python" and before "Install dependencies"
- name: Cache pip
  uses: actions/cache@v3
  with:
    path: ~/.cache/pip
    key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}
    restore-keys: |
      ${{ runner.os }}-pip-
```

### 5.2 Add Test Coverage Reporting

To ensure adequate test coverage, integrate coverage reporting:

```yaml
# Update the "Install dependencies" step to include coverage tools
- name: Install dependencies
  run: |
    python -m pip install --upgrade pip
    pip install -r requirements.txt
    pip install flake8 pytest pytest-cov # Add pytest-cov

# Update the "Test with pytest" step to generate coverage
- name: Test with pytest
  run: |
    pytest --cov=src --cov-report=xml --cov-report=term

# Add a step to upload coverage to Codecov (optional but recommended)
- name: Upload coverage to Codecov
  uses: codecov/codecov-action@v3
  with:
    file: ./coverage.xml
    fail_ci_if_error: false

# Add a step to enforce minimum coverage threshold
- name: Check coverage threshold
  run: |
    python -c "import xml.etree.ElementTree as ET; tree = ET.parse('coverage.xml'); root = tree.getroot(); coverage = float(root.attrib['line-rate']) * 100; print(f'Coverage: {coverage:.2f}%'); exit(1 if coverage < 80 else 0)"
```

### 5.3 Add Security Scanning

Integrate security scanning for both dependencies and container images:

```yaml
# Add this step after "Install dependencies" in the test job
- name: Scan dependencies for vulnerabilities
  run: |
    pip install safety
    safety check --full-report

# Add this step after "Build and Push Docker Image" in the build_and_deploy job
- name: Scan Docker image for vulnerabilities
  uses: aquasecurity/trivy-action@0.23.0
  with:
    image-ref: your-registry-name.azurecr.io/your-app-name:${{ github.sha }}
    format: 'table'
    exit-code: '1'
    ignore-unfixed: true
    severity: 'CRITICAL,HIGH'
```

### 5.4 Add Cross-Platform Testing

To ensure your application works across different operating systems, use a matrix strategy:

```yaml
# Replace the existing test job with this matrix version
test:
  name: Run Tests and Linting
  runs-on: ${{ matrix.os }}
  strategy:
    matrix:
      os: [ubuntu-latest, windows-latest, macos-latest]
      python-version: ['3.9', '3.10', '3.11']
    # Don't fail the entire matrix if one combination fails
    fail-fast: false

  steps:
    # ... (keep the same steps as before, but use matrix values)
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: ${{ matrix.python-version }}

    # ... rest of steps remain the same
```

### 5.5 Add Notification on Failure

To get notified when your workflow fails, add this step to both jobs:

```yaml
# Add this as the last step in each job
- name: Notify on failure
  if: failure()
  uses: actions/github-script@v6
  with:
    script: |
      github.rest.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: `:x: Workflow failed: ${context.job.name} on ${{ matrix.os }} with Python ${{ matrix.python-version }}`
      })
```

### 5.6 Add Workflow Badges

Add a status badge to your README.md to show the current state of your workflow:

```markdown
![Python CI/CD Pipeline](https://github.com/your-username/your-repo/actions/workflows/python-cicd.yml/badge.svg)
```

Replace `your-username` and `your-repo` with your actual GitHub username and repository name.

---

## Step 6: The Complete Workflow

Here is the complete enhanced `python-cicd.yml` file incorporating all the best practices we've discussed:

```yaml
name: Python CI/CD Pipeline

# 1. Define the triggers for this workflow
on:
  push:
    branches: [ "main" ] # Runs on pushes to the main branch
  pull_request:
    branches: [ "main" ] # Runs on pull requests targeting the main branch
  workflow_dispatch:  # Allows manual triggering from the Actions tab

jobs:
  # 2. Define the 'test' job with matrix strategy for cross-platform testing
  test:
    name: Run Tests and Linting
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        python-version: ['3.9', '3.10', '3.11']
      fail-fast: false  # Continue with other versions if one fails

    steps:
      # 3. Step 1: Check out your code from the repository
      - name: Checkout repository
        uses: actions/checkout@v3
        with:
          fetch-depth: 0  # Fetch all history for proper coverage analysis

      # 4. Step 2: Set up a specific Python version
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: ${{ matrix.python-version }}
          cache: 'pip'  # Cache pip dependencies

      # 5. Step 3: Install project dependencies with caching
      - name: Cache pip
        uses: actions/cache@v3
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}
          restore-keys: |
            ${{ runner.os }}-pip-

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install flake8 pytest pytest-cov  # Install testing and coverage tools

      # 6. Step 4: Run the linter (checks for code style issues)
      - name: Lint with flake8
        run: |
          # stop the build if there are Python syntax errors or undefined names
          flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
          # exit-zero treats all errors as warnings. The GitHub editor is 127 chars wide
          flake8 . --count --exit-zero --max-complexity=10 --max-line-length=127 --statistics

      # 7. Step 5: Scan dependencies for vulnerabilities
      - name: Scan dependencies for vulnerabilities
        run: |
          pip install safety
          safety check --full-report || echo "Safety check completed with warnings"

      # 8. Step 6: Run unit tests with pytest and coverage
      - name: Test with pytest
        run: |
          pytest --cov=src --cov-report=xml --cov-report=term --junitxml=test-results.xml

      # 9. Step 7: Upload coverage to Codecov
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage.xml
          fail_ci_if_error: false

      # 10. Step 8: Check coverage threshold
      - name: Check coverage threshold
        run: |
          python -c "import xml.etree.ElementTree as ET; tree = ET.parse('coverage.xml'); root = tree.getroot(); coverage = float(root.attrib['line-rate']) * 100; print(f'Coverage: {coverage:.2f}%'); exit(1 if coverage < 80 else 0)"

      # 11. Step 9: Upload test results
      - name: Upload test results
        uses: actions/upload-artifact@v3
        with:
          name: test-results-${{ matrix.python-version }}-${{ matrix.os }}
          path: test-results.xml

      # 12. Step 10: Notify on failure
      - name: Notify on failure
        if: failure()
        uses: actions/github-script@v6
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `:x: Workflow failed: ${{ job.name }} on ${{ matrix.os }} with Python ${{ matrix.python-version }}`
            })

  # 13. Define the 'build_and_deploy' job
  build_and_deploy:
    name: Build and Deploy to Azure
    runs-on: ubuntu-latest
    needs: test  # This job will only run if the 'test' job succeeds
    if: github.event_name == 'push'  # Only run this job on a push, not a pull request

    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      # Log in to your container registry (e.g., Azure Container Registry)
      - name: Log in to Azure Container Registry
        uses: docker/login-action@v2
        with:
          registry: your-registry-name.azurecr.io  # UPDATE THIS
          username: ${{ secrets.REGISTRY_USERNAME }}
          password: ${{ secrets.REGISTRY_PASSWORD }}

      # Build and push the Docker image
      - name: Build and Push Docker Image
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: your-registry-name.azurecr.io/your-app-name:${{ github.sha }}  # UPDATE THIS

      # Scan Docker image for vulnerabilities
      - name: Scan Docker image for vulnerabilities
        uses: aquasecurity/trivy-action@0.23.0
        with:
          image-ref: your-registry-name.azurecr.io/your-app-name:${{ github.sha }}
          format: 'table'
          exit-code: '1'
          ignore-unfixed: true
          severity: 'CRITICAL,HIGH'

      # Log in to Azure
      - name: Log in to Azure
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      # Deploy to Azure Web App
      - name: Deploy to Azure Web App
        uses: azure/webapps-deploy@v2
        with:
          app-name: 'your-azure-web-app-name'  # UPDATE THIS
          images: 'your-registry-name.azurecr.io/your-app-name:${{ github.sha }}'  # UPDATE THIS

      # Notify on failure
      - name: Notify on failure
        if: failure()
        uses: actions/github-script@v6
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `:x: Build and Deploy workflow failed`
            })
```

Remember to replace the placeholders like `your-registry-name.azurecr.io` with your actual resource names.

---

## Troubleshooting Common Issues

Here are solutions to common problems you might encounter when setting up your CI/CD pipeline:

### 1. Dependency Installation Failures

**Symptom**: Errors during `pip install -r requirements.txt`

**Solutions**:
- Check that your `requirements.txt` is correctly formatted
- Ensure you're not trying to install incompatible package versions
- For native dependencies, you may need to install system packages first:
  ```yaml
  - name: Install system dependencies (Ubuntu)
    if: contains(matrix.os, 'ubuntu')
    run: |
      sudo apt-get update
      sudo apt-get install -y build-essential libffi-dev libssl-dev
  ```

### 2. Docker Build Failures

**Symptom**: Errors during `docker/build-push-action`

**Solutions**:
- Verify your `Dockerfile` syntax is correct
- Check that all files referenced in the Dockerfile exist
- For GenAI projects with large models, consider:
  - Using `.dockerignore` to exclude large files from build context
  - Loading models at runtime instead of baking them into the image
  - Using external storage solutions for model files

### 3. Azure Deployment Failures

**Symptom**: Errors during `azure/webapps-deploy`

**Solutions**:
- Verify your Azure Web App name is correct
- Ensure your Azure service principal has sufficient permissions
- Check that your container registry is accessible from Azure
- Make sure your Azure Web App is configured to use Docker containers

### 4. Test Failures Due to Environment Differences

**Symptom**: Tests pass locally but fail in CI

**Solutions**:
- Use dependency locking (e.g., `pip freeze > requirements.txt`)
- Specify exact versions in your `requirements.txt`
- Use the same Python version in CI as you use locally
- Set environment variables explicitly in your workflow:
  ```yaml
  env:
    PYTHONPATH: src
    ENVIRONMENT: test
  ```

### 5. Coverage Threshold Failures

**Symptom**: Workflow fails despite adequate testing

**Solutions**:
- Check your actual coverage percentage in the workflow logs
- Add tests for uncovered code
- Temporarily lower the threshold to investigate:
  ```yaml
  # Change exit(1 if coverage < 80 else 0) to exit(1 if coverage < 70 else 0)
  ```

### 6. Secret Access Issues

**Symptom**: Errors when trying to access secrets like `${{ secrets.AZURE_CREDENTIALS }}`

**Solutions**:
- Verify the secrets are correctly set in Repository Settings > Secrets and variables > Actions
- Check that the secret names match exactly (case-sensitive)
- Ensure the workflow has permission to access secrets (they're available by default)
- For organization-level secrets, ensure they're granted access to this repository

### 7. Workflow Doesn't Trigger

**Symptom**: Workflow doesn't run when expected

**Solutions**:
- Check that your workflow file is in `.github/workflows/`
- Verify the spelling and indentation of your YAML
- Ensure your trigger conditions match your actual events (push vs pull_request)
- Check if branch protection rules are interfering
- Make sure you're pushing to the correct branch

---

## Next Steps

This guide covers the fundamentals of CI/CD for Python GenAI projects. From here, you can explore more advanced topics:

*   **Reusable Workflows**: To avoid duplicating code across different pipelines.
*   **Environments**: To create separate deployment environments (like dev, qa, prod) with their own protection rules and secrets.
*   **Advanced Triggers**: Using workflow_run, schedule, or webhook events for more complex automation.
*   **Performance Optimization**: Using job concurrency, caching strategies, and artifact management to speed up workflows.
*   **ML-Specific Considerations**: 
    - Model versioning and tracking with MLflow or Weights & Biases
    - Handling large model files with Git LFS or external storage
    - GPU-enabled workflows for training and inference
    - Data validation and drift detection in CI/CD
*   **Security Enhancements**:
    - Implementing OpenID Connect (OIDC) for Azure authentication instead of secrets
    - Using code scanning tools like CodeQL
    - Implementing dependency review with GitHub's dependency graph
*   **Observability**:
    - Adding detailed logging and monitoring to your workflows
    - Creating custom dashboards for CI/CD metrics
    - Setting up alerts for workflow failures or performance degradation

By mastering these concepts, you'll be able to create sophisticated, reliable CI/CD pipelines that support the rapid, safe evolution of your Python GenAI applications.

Happy coding!
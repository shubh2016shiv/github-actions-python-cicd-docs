# CI/CD Documentation (Python, FastAPI & GitHub Actions)

This repository contains production-grade Markdown guides on continuous integration and deployment for Python projects, with examples oriented toward Generative AI-style applications, FastAPI microservices, and multi-cloud deployments (Azure and AWS). The material is instructional reference documentation—not a runnable application or official product guidance.

## Quick Start

| Your Goal | Start Here |
|-----------|-----------|
| I'm new to CI/CD | [CICD_for_beginners.md](CICD_for_beginners.md) |
| I want CI-only (no deployment) | [Python_CI_Workflow.md](Python_CI_Workflow.md) |
| I'm deploying FastAPI to Azure | [FastAPI_Azure_CICD_GitHub_Action.md](FastAPI_Azure_CICD_GitHub_Action.md) |
| I'm deploying FastAPI to AWS ECS | [FastAPI_AWS_CICD_GitHub_Action.md](FastAPI_AWS_CICD_GitHub_Action.md) |
| I want enterprise multi-env pipelines | [Advanced_CICD_Pipeline.md](Advanced_CICD_Pipeline.md) |

## Contents

| Document | Description |
|----------|-------------|
| [CICD_for_beginners.md](CICD_for_beginners.md) | Introduces CI/CD concepts and walks through a GitHub Actions workflow: testing and linting, Docker, and deployment patterns using Azure as an example. Includes troubleshooting and follow-on topics. |
| [Python_CI_Workflow.md](Python_CI_Workflow.md) | Focuses on **continuous integration only** (no deployment): quality gates, tools such as flake8, mypy, black, security and dependency checks, matrix builds across Python versions and OSes, customization notes, and brief mention of other CI systems. |
| [Advanced_CICD_Pipeline.md](Advanced_CICD_Pipeline.md) | Covers more involved pipeline topics: reusable workflows, branching and environment layouts, multi-environment Azure resources (e.g. separate registries and web apps), branch protection, and related operational considerations. |
| [FastAPI_Azure_CICD_GitHub_Action.md](FastAPI_Azure_CICD_GitHub_Action.md) | Describes deploying a **FastAPI** app to **Azure App Service** with GitHub Actions: infrastructure setup examples, container image workflow, deployment slots, health checks, and Dockerfile-oriented notes. |
| [FastAPI_AWS_CICD_GitHub_Action.md](FastAPI_AWS_CICD_GitHub_Action.md) | Describes deploying a **FastAPI** app to **AWS ECS/Fargate** with GitHub Actions: OIDC authentication, ECR, VPC/ALB setup, rolling deployments, circuit-breaker rollback, and Docker Buildx caching. |

Read the guides in any order that matches your goal; the beginner guide and the CI-only guide overlap somewhat on testing and linting but differ in scope (full pipeline vs. CI-only).

## Architecture Overview

```
Developer Push → GitHub Actions (CI) → Build & Test → Container Registry → Cloud Deploy
                      │                      │               │               │
                  Lint/Scan            Docker Build      ACR / ECR     Azure / AWS
                  Security             Quality Gates     Image Store   App Service / ECS
```

## What you need

- Familiarity with Git and GitHub.
- A Python project layout (e.g. `requirements.txt`) where examples assume one.
- For Azure-related sections: an Azure subscription and willingness to substitute your own resource names, regions, and security settings.
- For AWS-related sections: an AWS account with permissions to create ECR, ECS, VPC, ALB, and IAM resources.

Tool versions, Action pins (`@v3`, `@v4`, etc.), and cloud CLI snippets in the docs reflect the time they were written. Verify current upstream documentation before applying them to production systems.

## Scope and limitations

- Examples are **illustrative**. You are responsible for secrets management, least-privilege access, cost controls, and compliance in your own environment.
- "GenAI" in the beginner guide mainly sets context for the type of Python project; the automation patterns are general Python/GitHub Actions material unless a section states otherwise.
- Cloud resource names (e.g. `your-registry-name`, `your-app-dev`) are placeholders. Replace them with your actual resource identifiers before running any scripts.

## Contributing

This documentation is meant to evolve with community best practices. If you find outdated action versions, broken links, or missing edge cases, open an issue or submit a pull request with your suggested changes.

## License

Add a `LICENSE` file to this repository if you intend to publish it publicly and want terms to apply to the text and examples.

# CI/CD documentation (Python & GitHub Actions)

This repository contains Markdown guides on continuous integration and deployment for Python projects, with examples oriented toward Generative AI-style applications and Microsoft Azure. The material is instructional reference documentation—not a runnable application or official product guidance.

## Contents

| Document | Description |
|----------|-------------|
| [CICD_for_beginners.md](CICD_for_beginners.md) | Introduces CI/CD concepts and walks through a GitHub Actions workflow: testing and linting, Docker, and deployment patterns using Azure as an example. Includes troubleshooting and follow-on topics. |
| [Python_CI_Workflow.md](Python_CI_Workflow.md) | Focuses on **continuous integration only** (no deployment): quality gates, tools such as flake8, mypy, black, security and dependency checks, matrix builds across Python versions and OSes, customization notes, and brief mention of other CI systems. |
| [Advanced_CICD_Pipeline.md](Advanced_CICD_Pipeline.md) | Covers more involved pipeline topics: reusable workflows, branching and environment layouts, multi-environment Azure resources (e.g. separate registries and web apps), branch protection, and related operational considerations. |
| [FastAPI_Azure_CICD_GitHub_Action.md](FastAPI_Azure_CICD_GitHub_Action.md) | Describes deploying a **FastAPI** app to **Azure App Service** with GitHub Actions: infrastructure setup examples, container image workflow, deployment slots, health checks, and Dockerfile-oriented notes. |

Read the guides in any order that matches your goal; the beginner guide and the CI-only guide overlap somewhat on testing and linting but differ in scope (full pipeline vs. CI-only).

## What you need

- Familiarity with Git and GitHub.
- A Python project layout (e.g. `requirements.txt`) where examples assume one.
- For Azure-related sections: an Azure subscription and willingness to substitute your own resource names, regions, and security settings.

Tool versions, Action pins (`@v3`, `@v4`, etc.), and Azure CLI snippets in the docs reflect the time they were written. Verify current upstream documentation before applying them to production systems.

## Scope and limitations

- Examples are **illustrative**. You are responsible for secrets management, least-privilege access, cost controls, and compliance in your own environment.
- “GenAI” in the beginner guide mainly sets context for the type of Python project; the automation patterns are general Python/GitHub Actions material unless a section states otherwise.

## License

Add a `LICENSE` file to this repository if you intend to publish it publicly and want terms to apply to the text and examples.

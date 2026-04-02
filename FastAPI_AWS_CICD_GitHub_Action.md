# Production-Ready FastAPI Deployment to AWS ECS/Fargate

This guide provides a robust, production-ready implementation for deploying FastAPI applications to AWS ECS/Fargate using GitHub Actions. It covers OIDC authentication, container hardening, health checks, load balancing, build caching, rolling deployments, and automated rollback strategies.

---

## Prerequisites

Before following this guide, ensure you have the following installed and configured:

| Tool | Minimum Version | Purpose |
|------|----------------|---------|
| AWS CLI | 2.x | AWS resource management |
| Python | 3.11+ | FastAPI development |
| Docker | 24.x+ | Container building and local testing |
| Git | 2.x | Version control |
| jq | 1.6+ | JSON processing in CI/CD |
| GitHub account | — | Repository hosting and Actions |
| AWS account | — | Cloud infrastructure |

### Required Knowledge

- FastAPI framework fundamentals
- Docker containerization concepts
- GitHub Actions workflow syntax
- AWS ECS, ECR, IAM, VPC, and ALB fundamentals

### Required AWS Permissions

The IAM user or role running `infrastructure/setup.sh` must have:

```
ecr:CreateRepository
ecs:CreateCluster
ecs:CreateService
ecs:CreateTaskDefinition
ec2:CreateVpc
ec2:CreateSubnet
ec2:CreateInternetGateway
ec2:CreateSecurityGroup
ec2:CreateRouteTable
ec2:AuthorizeSecurityGroupIngress
elasticloadbalancing:*
iam:CreateRole
iam:CreateOpenIDConnectProvider
iam:AttachRolePolicy
iam:PutRolePolicy
logs:CreateLogGroup
```

---

## Why This Architecture?

Understanding the reasoning behind architectural decisions helps you adapt this pattern to your specific needs:

### Container-Based Deployment on ECS/Fargate

We use AWS ECS with Fargate rather than EC2 instances or Elastic Beanstalk because:

- **Serverless compute** eliminates server management overhead
- **Automatic scaling** based on demand with configurable policies
- **Pay-per-use pricing** based on vCPU and memory allocated
- **Task-level isolation** through dedicated IAM roles per task
- **Native AWS integration** with ECR, CloudWatch, ALB, and Secrets Manager

### OIDC Authentication for GitHub Actions

We use OpenID Connect (OIDC) federation instead of long-lived AWS access keys because:

- **No persistent credentials** that could be leaked or rotated
- **Just-in-time access** with automatic token expiration (1 hour max)
- **Repository-scoped trust** limits access to specific repos and branches
- **Audit trail** through AWS CloudTrail showing which workflow assumed the role
- **AWS security best practice** for CI/CD pipelines

### Rolling Updates with Health Checks

We use ECS service rolling updates (not blue/green) because:

- **Zero-downtime deployments** when configured with proper min/max healthy percentages
- **Built-in health checks** via Application Load Balancer target groups
- **Automatic task replacement** when health checks fail
- **Simpler operational model** compared to CodeDeploy blue/green
- **Native ECS capability** requiring no additional services

> **Note:** If you need true blue/green with traffic shifting, integrate AWS CodeDeploy. This guide focuses on rolling updates for simplicity and cost-effectiveness.

### Docker Build Caching

We implement ECR-backed Docker Buildx caching because:

- **Persistent cache** survives workflow runs and GitHub runner recycling
- **Shared across branches** and team members via centralized ECR cache repository
- **Faster builds** by reusing unchanged layers (dependencies, base image)
- **Cost savings** through reduced compute time in CI/CD

### Structured Logging and Observability

We configure CloudWatch Logs integration because:

- **Centralized log aggregation** across all ECS tasks
- **Log streams per task** for easy debugging
- **Integration with CloudWatch Alarms** for automated alerting
- **No additional infrastructure** required

---

## Implementation Architecture

### Project Structure

```plaintext
your-fastapi-project/
├── .github/
│   └── workflows/
│       ├── deploy.yml                # Main trigger workflow
│       └── reusable-workflow.yml     # Reusable build-and-deploy logic
├── app/
│   ├── __init__.py
│   ├── main.py                       # FastAPI application entry point
│   └── config.py                     # Application configuration
├── infrastructure/
│   └── setup.sh                      # AWS resource provisioning script
├── taskdef/
│   └── taskdef.json                  # ECS task definition template
├── tests/
│   ├── __init__.py
│   └── test_main.py                  # Application tests
├── .dockerignore                     # Docker build exclusions
├── Dockerfile                        # Multi-stage hardened container
├── requirements.txt                  # Python dependencies
├── pytest.ini                        # Test configuration
└── pyproject.toml                    # Python project metadata (optional)
```

### Deployment Flow

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Developer  │────▶│  GitHub     │────▶│  Build &    │────▶│  Push Image │
│   Pushes     │     │  Actions    │     │  Test       │     │  to ECR     │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
                                                                    │
                                                                    ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Verified   │◀────│  Health     │◀────│  Update     │◀────│  Register   │
│  Deploy     │     │  Check      │     │  ECS Service│     │  Task Def   │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
```

---

## 1. AWS Infrastructure Setup

> **Purpose:** This script provisions all AWS resources required for container deployment: ECR repository, VPC networking, ECS cluster, ECS service, Application Load Balancer, IAM roles, and CloudWatch log groups.

### Script: `infrastructure/setup.sh`

```bash
#!/bin/bash
set -euo pipefail

# ============================================================
# Configuration - customize these for your environment
# ============================================================
AWS_REGION="${AWS_REGION:-us-east-1}"
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
ECR_REPOSITORY="${ECR_REPOSITORY:-fastapi-app}"
ECS_CLUSTER_NAME="${ECS_CLUSTER_NAME:-fastapi-cluster}"
ECS_SERVICE_NAME="${ECS_SERVICE_NAME:-fastapi-service}"
ECS_TASK_FAMILY="${ECS_TASK_FAMILY:-fastapi-task}"
VPC_CIDR="${VPC_CIDR:-10.0.0.0/16}"
CONTAINER_PORT="${CONTAINER_PORT:-80}"
ALB_PORT="${ALB_PORT:-80}"

echo "=== AWS Account: $AWS_ACCOUNT_ID | Region: $AWS_REGION ==="

# ============================================================
# 1. ECR Repository
# ============================================================
echo "[1/8] Creating ECR repository: $ECR_REPOSITORY"
aws ecr create-repository \
    --repository-name "$ECR_REPOSITORY" \
    --region "$AWS_REGION" \
    --image-scanning-configuration scanOnPush=true \
    --image-tag-mutability MUTABLE \
    --query 'repository.repositoryUri' --output text || true

ECR_URI=$(aws ecr describe-repositories \
    --repository-names "$ECR_REPOSITORY" \
    --query 'repositories[0].repositoryUri' \
    --output text --region "$AWS_REGION")
echo "  ECR URI: $ECR_URI"

# Create dedicated cache repository for Docker Buildx
CACHE_REPO="${ECR_REPOSITORY}-buildcache"
aws ecr create-repository \
    --repository-name "$CACHE_REPO" \
    --region "$AWS_REGION" \
    --image-tag-mutability MUTABLE \
    --query 'repository.repositoryUri' --output text || true
echo "  Cache URI: $(aws ecr describe-repositories --repository-names "$CACHE_REPO" --query 'repositories[0].repositoryUri' --output text --region "$AWS_REGION")"

# ============================================================
# 2. VPC and Networking
# ============================================================
echo "[2/8] Setting up VPC and networking..."

VPC_ID=$(aws ec2 create-vpc \
    --cidr-block "$VPC_CIDR" \
    --query 'Vpc.VpcId' --output text --region "$AWS_REGION")
aws ec2 modify-vpc-attribute --vpc-id "$VPC_ID" --enable-dns-hostnames --region "$AWS_REGION"
aws ec2 modify-vpc-attribute --vpc-id "$VPC_ID" --enable-dns-support --region "$AWS_REGION"
echo "  VPC: $VPC_ID"

# Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway \
    --query 'InternetGateway.InternetGatewayId' --output text --region "$AWS_REGION")
aws ec2 attach-internet-gateway --vpc-id "$VPC_ID" --internet-gateway-id "$IGW_ID" --region "$AWS_REGION"

# Public Subnets (2 for high availability)
SUBNET_1=$(aws ec2 create-subnet \
    --vpc-id "$VPC_ID" --cidr-block 10.0.1.0/24 \
    --availability-zone "${AWS_REGION}a" \
    --query 'Subnet.SubnetId' --output text --region "$AWS_REGION")
SUBNET_2=$(aws ec2 create-subnet \
    --vpc-id "$VPC_ID" --cidr-block 10.0.2.0/24 \
    --availability-zone "${AWS_REGION}b" \
    --query 'Subnet.SubnetId' --output text --region "$AWS_REGION")
SUBNET_IDS="${SUBNET_1},${SUBNET_2}"
echo "  Subnets: $SUBNET_IDS"

# Route Table
RTB_ID=$(aws ec2 create-route-table --vpc-id "$VPC_ID" \
    --query 'RouteTable.RouteTableId' --output text --region "$AWS_REGION")
aws ec2 create-route --route-table-id "$RTB_ID" \
    --destination-cidr-block 0.0.0.0/0 --gateway-id "$IGW_ID" --region "$AWS_REGION"
aws ec2 associate-route-table --subnet-id "$SUBNET_1" --route-table-id "$RTB_ID" --region "$AWS_REGION"
aws ec2 associate-route-table --subnet-id "$SUBNET_2" --route-table-id "$RTB_ID" --region "$AWS_REGION"

# ============================================================
# 3. Security Groups
# ============================================================
echo "[3/8] Creating security groups..."

# ALB Security Group (allows inbound HTTP/HTTPS)
ALB_SG=$(aws ec2 create-security-group \
    --group-name "fastapi-alb-sg" \
    --description "Security group for FastAPI ALB" \
    --vpc-id "$VPC_ID" --query 'GroupId' --output text --region "$AWS_REGION")
aws ec2 authorize-security-group-ingress \
    --group-id "$ALB_SG" --protocol tcp --port 80 --cidr 0.0.0.0/0 --region "$AWS_REGION"

# ECS Task Security Group (allows inbound from ALB only)
ECS_SG=$(aws ec2 create-security-group \
    --group-name "fastapi-ecs-sg" \
    --description "Security group for FastAPI ECS tasks" \
    --vpc-id "$VPC_ID" --query 'GroupId' --output text --region "$AWS_REGION")
aws ec2 authorize-security-group-ingress \
    --group-id "$ECS_SG" --protocol tcp --port "$CONTAINER_PORT" \
    --source-group "$ALB_SG" --region "$AWS_REGION"

echo "  ALB SG: $ALB_SG"
echo "  ECS SG: $ECS_SG"

# ============================================================
# 4. Application Load Balancer
# ============================================================
echo "[4/8] Creating Application Load Balancer..."

ALB_ARN=$(aws elbv2 create-load-balancer \
    --name "fastapi-alb" \
    --subnets "$SUBNET_1" "$SUBNET_2" \
    --security-groups "$ALB_SG" \
    --scheme internet-facing \
    --type application \
    --query 'LoadBalancers[0].LoadBalancerArn' --output text --region "$AWS_REGION")

ALB_DNS=$(aws elbv2 describe-load-balancers \
    --load-balancer-arns "$ALB_ARN" \
    --query 'LoadBalancers[0].DNSName' --output text --region "$AWS_REGION")

# Target Group
TARGET_GROUP_ARN=$(aws elbv2 create-target-group \
    --name "fastapi-tg" \
    --protocol HTTP --port "$CONTAINER_PORT" \
    --vpc-id "$VPC_ID" \
    --target-type ip \
    --health-check-protocol HTTP \
    --health-check-path /health \
    --health-check-interval-seconds 30 \
    --health-check-timeout-seconds 5 \
    --healthy-threshold-count 3 \
    --unhealthy-threshold-count 3 \
    --query 'TargetGroups[0].TargetGroupArn' --output text --region "$AWS_REGION")

# Listener
aws elbv2 create-listener \
    --load-balancer-arn "$ALB_ARN" \
    --protocol HTTP --port "$ALB_PORT" \
    --default-actions Type=forward,TargetGroupArn="$TARGET_GROUP_ARN" \
    --region "$AWS_REGION"

echo "  ALB DNS: $ALB_DNS"
echo "  Target Group: $TARGET_GROUP_ARN"

# ============================================================
# 5. CloudWatch Log Group
# ============================================================
echo "[5/8] Creating CloudWatch log group..."
aws logs create-log-group --log-group-name "/ecs/fastapi" --region "$AWS_REGION" || true

# ============================================================
# 6. ECS Cluster
# ============================================================
echo "[6/8] Creating ECS cluster: $ECS_CLUSTER_NAME"
aws ecs create-cluster \
    --cluster-name "$ECS_CLUSTER_NAME" \
    --region "$AWS_REGION" \
    --capacity-providers FARGATE FARGATE_SPOT \
    --default-capacity-provider-strategy \
        capacityProvider=FARGATE,weight=1,base=1 || true

# ============================================================
# 7. IAM Roles
# ============================================================
echo "[7/8] Creating IAM roles..."

# ECS Task Execution Role
TASK_EXECUTION_ROLE="ecsTaskExecutionRole"
if ! aws iam get-role --role-name "$TASK_EXECUTION_ROLE" --region "$AWS_REGION" &>/dev/null; then
    aws iam create-role \
        --role-name "$TASK_EXECUTION_ROLE" \
        --assume-role-policy-document '{
            "Version": "2012-10-17",
            "Statement": [{
                "Effect": "Allow",
                "Principal": {"Service": "ecs-tasks.amazonaws.com"},
                "Action": "sts:AssumeRole"
            }]
        }' --region "$AWS_REGION"

    aws iam attach-role-policy \
        --role-name "$TASK_EXECUTION_ROLE" \
        --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy \
        --region "$AWS_REGION"
fi

# GitHub Actions OIDC Role (least-privilege)
GITHUB_ROLE_NAME="github-actions-ecs-deploy"
OIDC_PROVIDER_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:oidc-provider/token.actions.githubusercontent.com"

# Create OIDC provider if it doesn't exist
THUMBPRINT=$(echo | openssl s_client -servername token.actions.githubusercontent.com \
    -showcerts -connect token.actions.githubusercontent.com:443 2>/dev/null | \
    openssl x509 -fingerprint -noout | sed 's/.*=//;s/://g' | tr 'A-F' 'a-f')

aws iam create-open-id-connect-provider \
    --url https://token.actions.githubusercontent.com \
    --client-id-list sts.amazonaws.com \
    --thumbprint-list "$THUMBPRINT" \
    --region "$AWS_REGION" 2>/dev/null || echo "  OIDC provider already exists"

# Create role if it doesn't exist
if ! aws iam get-role --role-name "$GITHUB_ROLE_NAME" --region "$AWS_REGION" &>/dev/null; then
    aws iam create-role \
        --role-name "$GITHUB_ROLE_NAME" \
        --assume-role-policy-document "{
            \"Version\": \"2012-10-17\",
            \"Statement\": [{
                \"Effect\": \"Allow\",
                \"Principal\": {
                    \"Federated\": \"$OIDC_PROVIDER_ARN\"
                },
                \"Action\": \"sts:AssumeRoleWithWebIdentity\",
                \"Condition\": {
                    \"StringLike\": {
                        \"token.actions.githubusercontent.com:sub\": \"repo:YOUR_ORG/YOUR_REPO:*\"
                    },
                    \"StringEquals\": {
                        \"token.actions.githubusercontent.com:aud\": \"sts.amazonaws.com\"
                    }
                }
            }]
        }" --region "$AWS_REGION"
fi

# Attach least-privilege policies (replace with custom policies in production)
aws iam attach-role-policy \
    --role-name "$GITHUB_ROLE_NAME" \
    --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPowerUser \
    --region "$AWS_REGION"

# Create custom inline policy for ECS operations
aws iam put-role-policy \
    --role-name "$GITHUB_ROLE_NAME" \
    --policy-name "ECSDeployPolicy" \
    --policy-document "{
        \"Version\": \"2012-10-17\",
        \"Statement\": [
            {
                \"Effect\": \"Allow\",
                \"Action\": [
                    \"ecs:DescribeTaskDefinition\",
                    \"ecs:RegisterTaskDefinition\",
                    \"ecs:UpdateService\",
                    \"ecs:DescribeServices\",
                    \"ecs:ListTasks\"
                ],
                \"Resource\": [
                    \"arn:aws:ecs:${AWS_REGION}:${AWS_ACCOUNT_ID}:service/${ECS_CLUSTER_NAME}/${ECS_SERVICE_NAME}\",
                    \"arn:aws:ecs:${AWS_REGION}:${AWS_ACCOUNT_ID}:task-definition/${ECS_TASK_FAMILY}:*\"
                ]
            },
            {
                \"Effect\": \"Allow\",
                \"Action\": [
                    \"ecs:DescribeClusters\"
                ],
                \"Resource\": \"arn:aws:ecs:${AWS_REGION}:${AWS_ACCOUNT_ID}:cluster/${ECS_CLUSTER_NAME}\"
            },
            {
                \"Effect\": \"Allow\",
                \"Action\": [
                    \"logs:GetLogEvents\",
                    \"logs:FilterLogEvents\"
                ],
                \"Resource\": \"arn:aws:logs:${AWS_REGION}:${AWS_ACCOUNT_ID}:log-group:/ecs/fastapi:*\"
            }
        ]
    }" --region "$AWS_REGION"

echo "  GitHub Actions Role: $GITHUB_ROLE_NAME"

# ============================================================
# 8. ECS Service
# ============================================================
echo "[8/8] Creating ECS service: $ECS_SERVICE_NAME"

# Create a placeholder task definition first (will be updated by CI/CD)
PLACEHOLDER_IMAGE="${ECR_URI}:placeholder"
aws ecs register-task-definition \
    --region "$AWS_REGION" \
    --cli-input-json "{
        \"family\": \"${ECS_TASK_FAMILY}\",
        \"networkMode\": \"awsvpc\",
        \"requiresCompatibilities\": [\"FARGATE\"],
        \"cpu\": \"256\",
        \"memory\": \"512\",
        \"executionRoleArn\": \"arn:aws:iam::${AWS_ACCOUNT_ID}:role/${TASK_EXECUTION_ROLE}\",
        \"containerDefinitions\": [{
            \"name\": \"fastapi-container\",
            \"image\": \"${PLACEHOLDER_IMAGE}\",
            \"portMappings\": [{\"containerPort\": ${CONTAINER_PORT}, \"protocol\": \"tcp\"}],
            \"essential\": true,
            \"logConfiguration\": {
                \"logDriver\": \"awslogs\",
                \"options\": {
                    \"awslogs-group\": \"/ecs/fastapi\",
                    \"awslogs-region\": \"${AWS_REGION}\",
                    \"awslogs-stream-prefix\": \"ecs\"
                }
            }
        }]
    }"

aws ecs create-service \
    --cluster "$ECS_CLUSTER_NAME" \
    --service-name "$ECS_SERVICE_NAME" \
    --task-definition "$ECS_TASK_FAMILY" \
    --desired-count 1 \
    --launch-type FARGATE \
    --network-configuration "awsvpcConfiguration={subnets=[${SUBNET_1//,/\",\"}],securityGroups=[${ECS_SG}],assignPublicIp=ENABLED}" \
    --load-balancers "targetGroupArn=${TARGET_GROUP_ARN},containerName=fastapi-container,containerPort=${CONTAINER_PORT}" \
    --deployment-configuration "maximumPercent=200,minimumHealthyPercent=100,deploymentCircuitBreaker={enable=true,rollback=true}" \
    --region "$AWS_REGION"

echo ""
echo "============================================"
echo "  Infrastructure Setup Complete!"
echo "============================================"
echo "  ECR URI:      $ECR_URI"
echo "  ALB DNS:      $ALB_DNS"
echo "  ECS Cluster:  $ECS_CLUSTER_NAME"
echo "  ECS Service:  $ECS_SERVICE_NAME"
echo "  Target Group: $TARGET_GROUP_ARN"
echo "============================================"
```

> **Important:** Replace `YOUR_ORG/YOUR_REPO` in the OIDC trust policy with your actual GitHub organization and repository name.

---

## 2. Hardened Multi-Stage Dockerfile

### Dockerfile

```dockerfile
# ============================================================
# Stage 1: Build dependencies
# ============================================================
FROM python:3.11-slim AS builder

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1

WORKDIR /build

# Install build dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --prefix=/install -r requirements.txt

# ============================================================
# Stage 2: Production runtime
# ============================================================
FROM python:3.11-slim AS production

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PORT=80 \
    WORKERS=4 \
    LOG_LEVEL=info \
    PYTHONPATH=/app

WORKDIR /app

# Copy installed dependencies from builder
COPY --from=builder /install /usr/local

# Install curl for health checks
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Create non-root user first
RUN useradd --create-home --shell /bin/bash appuser

# Copy application code
COPY --chown=appuser:appuser ./app ./app

# Switch to non-root user
USER appuser

EXPOSE ${PORT}

# Docker-level health check
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:${PORT}/health || exit 1

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "80", "--workers", "4"]
```

### `.dockerignore`

```
__pycache__/
*.pyc
*.pyo
.env
.venv/
venv/
.git/
.github/
*.md
tests/
infrastructure/
taskdef/
.pytest_cache/
.coverage
htmlcov/
```

---

## 3. FastAPI Application with Production Configuration

### `app/config.py`

```python
from pydantic_settings import BaseSettings
from functools import lru_cache


class Settings(BaseSettings):
    app_name: str = "FastAPI Production App"
    debug: bool = False
    allowed_origins: str = "https://yourdomain.com"
    log_level: str = "info"

    class Config:
        env_file = ".env"


@lru_cache()
def get_settings() -> Settings:
    return Settings()
```

### `app/main.py`

```python
import logging
from contextlib import asynccontextmanager

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import JSONResponse

from app.config import get_settings

# Configure structured logging
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s | %(levelname)s | %(name)s | %(message)s",
)
logger = logging.getLogger(__name__)


@asynccontextmanager
async def lifespan(app: FastAPI):
    """Application lifespan handler for startup/shutdown events."""
    logger.info("Application starting up...")
    yield
    logger.info("Application shutting down...")


settings = get_settings()

app = FastAPI(
    title=settings.app_name,
    lifespan=lifespan,
    docs_url="/docs" if settings.debug else None,
    redoc_url="/redoc" if settings.debug else None,
)

# Configure CORS with specific origins (not wildcard in production)
allowed_origins = [origin.strip() for origin in settings.allowed_origins.split(",")]
app.add_middleware(
    CORSMiddleware,
    allow_origins=allowed_origins,
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
)


@app.get("/health")
async def health_check():
    """Health check endpoint for load balancer and orchestrator probes."""
    return JSONResponse(
        status_code=200,
        content={"status": "healthy", "version": "1.0.0"},
    )


@app.get("/ready")
async def readiness_check():
    """Readiness check endpoint for Kubernetes/ECS readiness probes."""
    return JSONResponse(
        status_code=200,
        content={"status": "ready"},
    )


@app.get("/")
async def root():
    return {"message": f"{settings.app_name} is running"}
```

---

## 4. ECS Task Definition Template

### `taskdef/taskdef.json`

```json
{
  "family": "fastapi-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "runtimePlatform": {
    "cpuArchitecture": "X86_64",
    "operatingSystemFamily": "LINUX"
  },
  "executionRoleArn": "arn:aws:iam::<AWS_ACCOUNT_ID>:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "fastapi-container",
      "image": "<ECR_REPOSITORY_URI>:<IMAGE_TAG>",
      "portMappings": [
        {
          "containerPort": 80,
          "protocol": "tcp"
        }
      ],
      "essential": true,
      "environment": [
        {
          "name": "PORT",
          "value": "80"
        },
        {
          "name": "WORKERS",
          "value": "4"
        },
        {
          "name": "LOG_LEVEL",
          "value": "info"
        }
      ],
      "secrets": [],
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:80/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 10
      },
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/fastapi",
          "awslogs-region": "<AWS_REGION>",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "memoryReservation": 256
    }
  ]
}
```

> **Note:** Replace `<AWS_ACCOUNT_ID>`, `<ECR_REPOSITORY_URI>`, `<IMAGE_TAG>`, and `<AWS_REGION>` with actual values. The CI/CD workflow handles this substitution automatically.

---

## 5. Production CI/CD Workflow

### Understanding the Deployment Pipeline

The connection between GitHub Actions and AWS ECS follows this sequence:

1. **Authentication**: GitHub Actions assumes an AWS IAM role via OIDC federation—no long-lived credentials
2. **Build**: Docker builds the container image with ECR-backed layer caching for speed
3. **Push**: Image is pushed to Amazon ECR with a unique git SHA tag
4. **Register**: A new ECS task definition revision is created pointing to the new image
5. **Deploy**: The ECS service is updated, triggering a rolling deployment
6. **Verify**: The workflow waits for service stabilization and verifies task health
7. **Rollback**: If deployment fails, ECS deployment circuit breaker automatically rolls back

### Main Trigger Workflow: `.github/workflows/deploy.yml`

```yaml
name: Deploy FastAPI to AWS ECS

on:
  push:
    branches: [main, develop]
    paths-ignore:
      - '**.md'
      - 'docs/**'
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production

jobs:
  deploy:
    uses: ./.github/workflows/reusable-workflow.yml
    with:
      app-name: fastapi-app
      environment: ${{ github.event.inputs.environment || (github.ref == 'refs/heads/main' && 'production' || 'staging') }}
      python-version: '3.11'
      aws-region: 'us-east-1'
      ecr-repository: fastapi-app
      ecs-cluster: fastapi-cluster
      ecs-service: fastapi-service
      ecs-task-family: fastapi-task
    secrets: inherit
```

### Reusable Workflow: `.github/workflows/reusable-workflow.yml`

```yaml
name: Reusable FastAPI Deploy to AWS

on:
  workflow_call:
    inputs:
      app-name:
        required: true
        type: string
      environment:
        required: true
        type: string
      python-version:
        required: true
        type: string
      aws-region:
        required: true
        type: string
      ecr-repository:
        required: true
        type: string
      ecs-cluster:
        required: true
        type: string
      ecs-service:
        required: true
        type: string
      ecs-task-family:
        required: true
        type: string
    secrets:
      AWS_ACCOUNT_ID:
        required: true

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}

    permissions:
      id-token: write
      contents: read

    env:
      AWS_REGION: ${{ inputs.aws-region }}
      ECR_REPOSITORY: ${{ inputs.ecr-repository }}
      ECS_CLUSTER: ${{ inputs.ecs-cluster }}
      ECS_SERVICE: ${{ inputs.ecs-service }}
      ECS_TASK_FAMILY: ${{ inputs.ecs-task-family }}
      IMAGE_TAG: ${{ github.sha }}
      ACCOUNT_ID: ${{ secrets.AWS_ACCOUNT_ID }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ inputs.python-version }}
          cache: 'pip'
          cache-dependency-path: 'requirements.txt'

      - name: Install dependencies and run tests
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install pytest pytest-cov
          pytest tests/ --cov=app --cov-report=xml --cov-report=term-missing --cov-fail-under=80

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ env.ACCOUNT_ID }}:role/github-actions-ecs-deploy
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push image to ECR
        id: build-image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          CACHE_URI: ${{ steps.login-ecr.outputs.registry }}/${{ inputs.ecr-repository }}-buildcache
        run: |
          docker buildx build \
            --platform linux/amd64 \
            --cache-from type=registry,ref=${{ env.CACHE_URI }}:latest \
            --cache-to type=registry,ref=${{ env.CACHE_URI }}:latest,mode=max \
            --tag ${{ env.ECR_REGISTRY }}/${{ env.ECR_REPOSITORY }}:${{ env.IMAGE_TAG }} \
            --tag ${{ env.ECR_REGISTRY }}/${{ env.ECR_REPOSITORY }}:latest \
            --push \
            .
          echo "image=${{ env.ECR_REGISTRY }}/${{ env.ECR_REPOSITORY }}:${{ env.IMAGE_TAG }}" >> $GITHUB_OUTPUT

      - name: Register new task definition
        id: register-task
        run: |
          # Get current task definition
          TASK_DEF=$(aws ecs describe-task-definition \
            --task-definition ${{ env.ECS_TASK_FAMILY }} \
            --query "taskDefinition" --output json --region ${{ env.AWS_REGION }})

          # Update image and register new revision
          NEW_TASK_DEF=$(echo "$TASK_DEF" | jq \
            --arg image "${{ env.ECR_REGISTRY }}/${{ env.ECR_REPOSITORY }}:${{ env.IMAGE_TAG }}" \
            'del(.taskDefinitionArn) | del(.revision) | del(.status) | del(.requiresAttributes) | del(.compatibilities) | del(.registeredAt) | del(.registeredBy) | .containerDefinitions[0].image = $image')

          REGISTERED=$(aws ecs register-task-definition \
            --cli-input-json "$NEW_TASK_DEF" \
            --region ${{ env.AWS_REGION }})

          REVISION=$(echo "$REGISTERED" | jq -r '.taskDefinition.revision')
          echo "revision=$REVISION" >> $GITHUB_OUTPUT
          echo "Registered task definition revision: $REVISION"

      - name: Update ECS service
        run: |
          aws ecs update-service \
            --cluster ${{ env.ECS_CLUSTER }} \
            --service ${{ env.ECS_SERVICE }} \
            --task-definition ${{ env.ECS_TASK_FAMILY }}:${{ steps.register-task.outputs.revision }} \
            --force-new-deployment \
            --region ${{ env.AWS_REGION }}

          echo "Waiting for service to stabilize..."
          aws ecs wait services-stable \
            --cluster ${{ env.ECS_CLUSTER }} \
            --services ${{ env.ECS_SERVICE }} \
            --region ${{ env.AWS_REGION }}

      - name: Verify deployment
        run: |
          SERVICE=$(aws ecs describe-services \
            --cluster ${{ env.ECS_CLUSTER }} \
            --services ${{ env.ECS_SERVICE }} \
            --region ${{ env.AWS_REGION }})

          RUNNING=$(echo "$SERVICE" | jq -r '.services[0].runningCount')
          DESIRED=$(echo "$SERVICE" | jq -r '.services[0].desiredCount')
          PENDING=$(echo "$SERVICE" | jq -r '.services[0].pendingCount')

          echo "Desired: $DESIRED | Running: $RUNNING | Pending: $PENDING"

          if [ "$RUNNING" -lt "$DESIRED" ] || [ "$PENDING" -gt 0 ]; then
            echo "Deployment verification failed: tasks not healthy"
            exit 1
          fi

          echo "Deployment verified: all tasks running and healthy"
```

---

## 6. GitHub Repository Setup

### Step 1: Configure OIDC Trust Policy

After running `infrastructure/setup.sh`, update the OIDC trust policy with your actual repository:

1. Go to AWS IAM Console > Roles > `github-actions-ecs-deploy`
2. Click **Trust relationships** > **Edit trust policy**
3. Replace `YOUR_ORG/YOUR_REPO` with your actual GitHub organization and repository:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:your-org/your-repo:*"
        },
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

### Step 2: Configure GitHub Secrets and Environments

1. Go to your GitHub repository > **Settings** > **Secrets and variables** > **Actions**

2. Add the following **repository secret**:
   | Secret Name | Value | Description |
   |------------|-------|-------------|
   | `AWS_ACCOUNT_ID` | Your 12-digit AWS account ID | Found via `aws sts get-caller-identity --query Account --output text` |

3. Create **Environments** (Settings > Environments):

   **Staging Environment:**
   - Name: `staging`
   - No protection rules needed (auto-deploy on push to `develop`)

   **Production Environment:**
   - Name: `production`
   - Enable **Required reviewers** (add at least 1 reviewer)
   - Enable **Wait timer** (optional, e.g., 5 minutes)
   - Enable **Deployment branches** (restrict to `main` only)

### Step 3: Configure Branch Protection

1. Go to **Settings** > **Branches** > **Add rule**
2. Branch pattern: `main`
3. Enable:
   - Require a pull request before merging
   - Require status checks to pass before merging
   - Require branches to be up to date before merging
   - Include administrators (recommended)

### Step 4: Verify Setup

Push a commit to your repository and monitor:

1. **GitHub Actions tab**: Check workflow runs for successful execution
2. **AWS ECR Console**: Verify images appear with correct SHA tags
3. **AWS ECS Console**: Check service deployment status and task health
4. **ALB DNS**: Access the load balancer URL to verify the application is reachable

---

## 7. Requirements and Dependencies

### `requirements.txt`

```
fastapi>=0.104.0
uvicorn[standard]>=0.24.0
pydantic-settings>=2.0.0
python-multipart>=0.0.6
pytest>=7.4.3
pytest-cov>=4.1.0
httpx>=0.25.1
```

### `pytest.ini`

```ini
[pytest]
testpaths = tests
python_files = test_*.py
python_functions = test_*
addopts = -v --tb=short
```

---

## 8. Troubleshooting Common Issues

### Authentication Failures

**Symptom:** `configure-aws-credentials` fails with `AccessDenied` or `InvalidToken`

**Resolution:**
1. Verify OIDC provider exists: `aws iam list-open-id-connect-providers`
2. Check trust policy has correct repository reference (`repo:org/repo:*`)
3. Confirm audience is `sts.amazonaws.com` (not `sts.amazonaws.com:443`)
4. Verify `AWS_ACCOUNT_ID` secret is set correctly in GitHub
5. Check GitHub Actions `id-token: write` permission is set in workflow

### OIDC Thumbprint Rotation

**Symptom:** OIDC authentication fails after AWS thumbprint update

**Resolution:**
```bash
# Get current thumbprint
echo | openssl s_client -servername token.actions.githubusercontent.com \
    -showcerts -connect token.actions.githubusercontent.com:443 2>/dev/null | \
    openssl x509 -fingerprint -noout

# Update in AWS IAM Console > Identity providers > token.actions.githubusercontent.com
```

### ECS Tasks Fail to Start

**Symptom:** Tasks enter STOPPED state immediately

**Resolution:**
1. Check CloudWatch Logs: `/ecs/fastapi` log group
2. Verify image exists: `aws ecr describe-images --repository-name fastapi-app --image-ids imageTag=<SHA>`
3. Test locally:
   ```bash
   aws ecr get-login-password --region us-east-1 | \
     docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com
   docker run -p 8080:80 <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/fastapi-app:<SHA>
   ```
4. Check task execution role has `AmazonECSTaskExecutionRolePolicy` attached
5. Verify security group allows inbound traffic from ALB on container port

### ECS Service Not Stabilizing

**Symptom:** `aws ecs wait services-stable` times out

**Resolution:**
1. Check deployment events:
   ```bash
   aws ecs describe-services --cluster <CLUSTER> --services <SERVICE> \
     --query 'services[0].events' --region <REGION>
   ```
2. Check stopped tasks:
   ```bash
   aws ecs list-tasks --cluster <CLUSTER> --service-name <SERVICE> --desired-status STOPPED
   aws ecs describe-tasks --cluster <CLUSTER> --tasks <TASK_ARN>
   ```
3. Verify target group health checks are passing in EC2 Console > Target Groups
4. Check `maximumPercent` and `minimumHealthyPercent` settings allow deployment
5. Ensure subnets have internet access (for image pull) via NAT Gateway or public IP

### VPC Networking Issues

**Symptom:** Tasks cannot pull images or reach external services

**Resolution:**
1. Verify route table has route to Internet Gateway (public subnets) or NAT Gateway (private subnets)
2. Check security group allows outbound traffic (default allows all outbound)
3. For private subnets, ensure NAT Gateway is configured
4. Verify `assignPublicIp=ENABLED` in service network configuration for public subnets

### Build Cache Not Working

**Symptom:** Every build takes the same amount of time, no cache hits

**Resolution:**
1. Verify cache repository exists: `aws ecr describe-repositories --repository-name fastapi-app-buildcache`
2. Check Buildx output in workflow logs for ` importing cache manifest`
3. Ensure `--cache-from` and `--cache-to` flags are present in build command
4. First build will never use cache (nothing to cache from)
5. Cache is invalidated when Dockerfile or source files change (expected behavior)

---

## 9. Deployment Strategy and Rollback

### Rolling Update Strategy

Our implementation uses ECS rolling updates with these characteristics:

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `maximumPercent` | 200% | Allows up to 2x desired tasks during deployment |
| `minimumHealthyPercent` | 100% | Ensures no downtime during rollout |
| `deploymentCircuitBreaker` | enabled | Auto-rollback if tasks fail to start |
| `rollback` | true | Automatic rollback on circuit breaker trigger |

### Deployment Flow

```
1. Build image with SHA tag          → ECR
2. Register new task definition       → ECS
3. Update service with new revision   → ECS Service
4. ECS starts new tasks (up to 200%)  → Fargate
5. ALB health checks validate tasks   → Target Group
6. Old tasks drain and stop           → ECS
7. Service stabilizes                 → Complete
```

### Manual Rollback

If automatic rollback doesn't trigger and you need to roll back manually:

```bash
# Find the previous revision
aws ecs list-task-definitions --family-prefix fastapi-task --sort DESC --max-items 2

# Roll back to previous revision
aws ecs update-service \
  --cluster fastapi-cluster \
  --service fastapi-service \
  --task-definition fastapi-task:<PREVIOUS_REVISION> \
  --region us-east-1
```

### Blue/Green with CodeDeploy (Optional)

For true blue/green deployments with traffic shifting:

1. Create an AWS CodeDeploy application and deployment group
2. Configure traffic routing (Canary, Linear, or All-at-once)
3. Update the workflow to use `aws deploy create-deployment`
4. CodeDeploy handles traffic shifting and automatic rollback

This adds complexity but provides finer-grained traffic control during deployments.

---

## 10. Security Best Practices Checklist

- [x] OIDC authentication (no long-lived AWS credentials)
- [x] Least-privilege IAM policies for GitHub Actions role
- [x] Non-root user in Docker container
- [x] ECR image scanning on push enabled
- [x] Security group restricts ECS task access to ALB only
- [x] CORS configured with specific origins (not wildcard)
- [x] Debug endpoints disabled in production
- [x] Environment variables for configuration (not hardcoded)
- [x] GitHub environment protection rules for production
- [x] Branch protection rules for main branch

---

## 11. Cost Optimization Tips

| Area | Tip | Estimated Savings |
|------|-----|-------------------|
| Fargate | Use Fargate Spot for non-production | Up to 70% |
| Task Size | Right-size CPU/memory (start with 256 CPU, 512 MB) | 30-50% |
| ECR | Lifecycle policies to expire old images | Storage costs |
| CloudWatch | Set log retention (e.g., 14 days) | Log storage costs |
| ALB | Use Application Load Balancer (not NLB) for HTTP | Lower hourly cost |
| Scaling | Configure auto-scaling based on CPU/memory | Pay only for what you use |

### ECR Lifecycle Policy

```bash
aws ecr put-lifecycle-policy \
  --repository-name fastapi-app \
  --lifecycle-policy-text '{
    "rules": [{
      "rulePriority": 1,
      "description": "Keep last 10 images, expire older untagged",
      "selection": {
        "tagStatus": "untagged",
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": {"type": "expire"}
    }, {
      "rulePriority": 2,
      "description": "Keep last 5 tagged images",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["sha"],
        "countType": "imageCountMoreThan",
        "countNumber": 5
      },
      "action": {"type": "expire"}
    }]
  }' --region us-east-1
```
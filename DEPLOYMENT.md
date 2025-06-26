# Langflow Deployment Guide

This document describes the deployment setup for Langflow in the BetterUp infrastructure.

## Overview

Langflow is deployed using a modern GitOps approach with:
- **Container Images**: Built and stored in AWS ECR
- **Kubernetes**: Deployed via Helm charts
- **Database**: PostgreSQL RDS managed by Terraform
- **CI/CD**: GitHub Actions workflows for automated deployment

## Architecture

```
┌─────────────────┐    ┌──────────────┐    ┌─────────────────┐
│   GitHub Actions│    │     ECR      │    │   EKS Cluster   │
│   (Build & Push)│───►│  (Images)    │───►│   (Langflow)    │
└─────────────────┘    └──────────────┘    └─────────────────┘
                                                     │
                                                     ▼
                                            ┌─────────────────┐
                                            │   RDS PostgreSQL│
                                            │   (Database)    │
                                            └─────────────────┘
```

## Repository Structure

```
langflow/
├── .github/workflows/          # CI/CD workflows
│   ├── staging_cd.yml         # Main deployment pipeline
│   ├── build_and_push_*.yml   # Container build workflows
│   └── deploy_*.yml           # Helm deployment workflows
├── release/kubernetes/langflow/ # Helm chart
│   ├── Chart.yaml             # Chart metadata
│   ├── values.yaml            # Default values
│   ├── templates/             # Kubernetes manifests
│   └── values/                # Environment-specific values
├── Dockerfile.production      # Production container build
└── DEPLOYMENT.md             # This file
```

## Deployment Workflow

### 1. Infrastructure Setup
First, the infrastructure must be deployed via the `betterup-infrastructure` repository:
- PostgreSQL RDS instance
- IAM service accounts
- Security groups
- ECR repository

### 2. Application Deployment
The application deployment follows this flow:

1. **Code Push** → GitHub Actions triggered
2. **Build Image** → Docker image built and pushed to ECR
3. **Package Chart** → Helm chart packaged and pushed
4. **Deploy** → Helm upgrade/install to Kubernetes

### 3. Environment Promotion
- **Dev**: Automatic deployment from `main` branch
- **Staging**: Automatic deployment from `staging` branch  
- **Production**: Manual deployment via workflow_dispatch

## Environment Configuration

### Development (us-east-1-dev)
- Single replica
- Reduced resource limits
- Debug logging enabled
- Auto-deployment from main branch

### Staging (us-east-1-staging)
- Multi-replica setup
- Production-like resources
- Info-level logging
- Auto-deployment from staging branch

### Production (Future)
- High availability setup
- Full resource allocation
- Error-level logging
- Manual deployment only

## Database Configuration

Langflow connects to PostgreSQL using:
- **Connection String**: Retrieved from AWS Secrets Manager
- **SSL**: Required for all connections
- **User**: Dedicated database user with limited privileges
- **Migrations**: Handled automatically by Langflow on startup

## Monitoring

### Application Metrics
- **Datadog APM**: Automatic instrumentation
- **Health Checks**: Kubernetes liveness/readiness probes
- **Logs**: Structured logging to CloudWatch

### Infrastructure Metrics
- **RDS Monitoring**: CloudWatch metrics and Performance Insights
- **EKS Monitoring**: Cluster and pod metrics
- **Resource Usage**: CPU, memory, and storage tracking

## Security

### Container Security
- Non-root user execution
- Minimal base image (Python slim)
- No secrets in image layers
- Regular vulnerability scanning

### Network Security
- Database access limited to EKS security groups
- No public database access
- TLS encryption for all connections

### Access Control
- IAM roles for service accounts
- Least privilege permissions
- AWS Secrets Manager for credentials

## Troubleshooting

### Common Issues

1. **Database Connection Failures**
   ```bash
   kubectl logs deployment/langflow
   # Check for PostgreSQL connection errors
   ```

2. **Image Pull Errors**
   ```bash
   kubectl describe pod -l app.kubernetes.io/name=langflow
   # Verify ECR permissions and image tags
   ```

3. **Resource Constraints**
   ```bash
   kubectl top pods -l app.kubernetes.io/name=langflow
   # Check CPU and memory usage
   ```

### Useful Commands

```bash
# View application logs
kubectl logs -f deployment/langflow

# Check pod status
kubectl get pods -l app.kubernetes.io/name=langflow

# Access application shell
kubectl exec -it deployment/langflow -- bash

# Port forward for local testing
kubectl port-forward service/langflow 7860:7860
```

## Development

### Local Development
For local development with the production database:
```bash
# Port forward to database (if allowed)
kubectl port-forward service/langflow-db 5432:5432

# Run Langflow locally
export LANGFLOW_DATABASE_URL="postgresql://user:pass@localhost:5432/langflow"
python -m langflow run
```

### Testing Changes
1. Create feature branch
2. Push changes to trigger CI
3. Review deployment logs
4. Test functionality
5. Create pull request

## Rollback Procedure

In case of deployment issues:
```bash
# Get previous release
helm history langflow -n default

# Rollback to previous version
helm rollback langflow [REVISION] -n default
```
# Coworking Analytics Application - Deployment Guide

## Overview

This guide documents the CI/CD pipeline and deployment architecture for the Coworking Analytics microservice on AWS EKS.

## Architecture

The application follows a microservices architecture deployed on Amazon EKS with the following components:

- **Application**: Python Flask API containerized with Docker
- **Database**: PostgreSQL running as a Kubernetes deployment
- **CI/CD**: AWS CodeBuild for automated Docker image builds
- **Container Registry**: Amazon ECR for Docker image storage
- **Monitoring**: CloudWatch Container Insights for logs and metrics
- **Orchestration**: Kubernetes (EKS) for container management

## Prerequisites

- AWS CLI configured with appropriate IAM permissions
- kubectl v1.32+ installed and configured
- eksctl for EKS cluster management
- Docker (for local testing)
- GitHub repository access

## Deployment Process

### 1. Infrastructure Setup

The EKS cluster is provisioned with:

```bash
eksctl create cluster --name my-cluster --region us-east-1 --nodegroup-name my-nodes --node-type t3.small --nodes 1 --nodes-min 1 --nodes-max 2
```

This creates a production-ready cluster with auto-scaling capabilities (1-2 nodes) using cost-effective t3.small instances.

### 2. Database Deployment

PostgreSQL is deployed using Kubernetes manifests without persistent volumes for development/testing. For production, implement persistent volumes with EBS storage class:

```bash
kubectl apply -f postgresql-ephemeral.yaml
```

The database is accessible within the cluster via the `postgresql-service` ClusterIP service on port 5432.

### 3. CI/CD Pipeline

**Automated Build Process:**

1. Code pushed to GitHub triggers AWS CodeBuild via webhook
2. CodeBuild executes `buildspec.yaml`:
   - Authenticates with ECR
   - Builds Docker image from `analytics/Dockerfile`
   - Tags image with build number, commit hash, and 'latest'
   - Pushes images to ECR repository
3. Build artifacts are stored in S3 (imagedefinitions.json)

**Environment Variables (CodeBuild):**

- `AWS_ACCOUNT_ID`: 183302320810
- `AWS_DEFAULT_REGION`: us-east-1
- `IMAGE_REPO_NAME`: coworking

### 4. Application Deployment

Kubernetes resources are applied in this order:

```bash
kubectl apply -f deployment/configmap.yaml
kubectl apply -f deployment/secret.yaml
kubectl apply -f deployment/coworking.yaml
```

**ConfigMap** (`configmap.yaml`): Stores non-sensitive environment variables (DB host, port, username, database name)
**Secret** (`secret.yaml`): Stores base64-encoded sensitive data (DB password)
**Deployment** (`coworking.yaml`): Defines the application deployment with:

- LoadBalancer service exposing port 5153
- Health check probes (liveness and readiness)
- Environment variable injection from ConfigMap and Secret
- Image pull policy: Always (ensures latest image is used)

### 5. Monitoring Setup

CloudWatch Container Insights is enabled for comprehensive monitoring:

```bash
kubectl apply -f https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/cloudwatch-namespace.yaml
# Additional manifests for CloudWatch agent and Fluent Bit
```

## Releasing New Builds

### Automatic Deployment

1. Make code changes in the `analytics/` directory
2. Commit and push to GitHub main branch
3. CodeBuild automatically triggers and builds new Docker image
4. Image is tagged with new build number and pushed to ECR
5. Update Kubernetes deployment to use new image:

```bash
kubectl set image deployment/coworking coworking=183302320810.dkr.ecr.us-east-1.amazonaws.com/coworking:<NEW_BUILD_NUMBER>
```

Or force a rolling update:

```bash
kubectl rollout restart deployment/coworking
```

### Manual Deployment

For testing or emergency deployments:

```bash
# Build locally
cd analytics
docker build -t coworking:local .

# Tag for ECR
docker tag coworking:local 183302320810.dkr.ecr.us-east-1.amazonaws.com/coworking:manual-v1.0.0

# Push to ECR
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 183302320810.dkr.ecr.us-east-1.amazonaws.com
docker push 183302320810.dkr.ecr.us-east-1.amazonaws.com/coworking:manual-v1.0.0

# Update deployment
kubectl set image deployment/coworking coworking=183302320810.dkr.ecr.us-east-1.amazonaws.com/coworking:manual-v1.0.0
```

## Configuration Updates

### Environment Variables

To update application configuration:

```bash
kubectl edit configmap coworking-configmap
kubectl rollout restart deployment/coworking
```

### Secrets

To update sensitive data:

```bash
kubectl edit secret coworking-secret
kubectl rollout restart deployment/coworking
```

## Scaling

### Horizontal Pod Autoscaling

```bash
kubectl autoscale deployment coworking --cpu-percent=70 --min=1 --max=5
```

### Manual Scaling

```bash
kubectl scale deployment coworking --replicas=3
```

### Cluster Node Scaling

Managed node groups auto-scale between 1-2 nodes based on resource demands. To modify:

```bash
eksctl scale nodegroup --cluster=my-cluster --name=my-nodes --nodes=3 --nodes-min=2 --nodes-max=4
```

## Monitoring and Troubleshooting

### View Application Logs

```bash
# Real-time logs
kubectl logs -f deployment/coworking

# CloudWatch logs
aws logs tail /aws/containerinsights/my-cluster/application --follow
```

### Check Service Health

```bash
kubectl get pods
kubectl describe deployment coworking
kubectl describe svc coworking
```

### Access Application

The application is exposed via AWS LoadBalancer:

```bash
kubectl get svc coworking
# Use EXTERNAL-IP:5153 to access the application
```

## Cost Optimization Recommendations

1. **Right-size Instances**: Use t3.small nodes ($0.0208/hour) which provide adequate resources for the analytics workload. The application requires minimal compute resources, making burstable instances ideal.

2. **Spot Instances**: For non-production environments, implement Spot instances for up to 90% cost savings. Use managed node groups with mixed instance policies.

3. **Horizontal Pod Autoscaling**: Automatically scale pods based on CPU/memory usage to avoid over-provisioning resources during low traffic periods.

4. **Cluster Autoscaler**: Enable cluster autoscaler to scale nodes down to minimum (1 node) during off-peak hours, saving $15-20/day.

5. **Reserved Instances**: For production, purchase 1-year Reserved Instances for 40% savings on EC2 costs.

6. **ECR Lifecycle Policies**: Implement lifecycle policies to delete old/unused Docker images, reducing storage costs.

7. **CloudWatch Log Retention**: Set log retention to 7-30 days instead of indefinite retention, saving on storage costs.

## Instance Type Recommendation

**Recommended: t3.small**

- 2 vCPUs, 2 GiB RAM
- Cost: ~$15/month per node
- Rationale: The analytics application is I/O bound with periodic batch processing (daily report generation every 30 seconds). The Flask application with PostgreSQL queries requires minimal CPU but benefits from burstable performance during report generation. T3 instances provide baseline CPU with burst capability, making them cost-effective for this workload pattern.

**Alternative for Production: t3.medium**

- 2 vCPUs, 4 GiB RAM
- Cost: ~$30/month per node
- Use when scaling to 5+ pods or implementing heavy caching layers

## Security Best Practices

1. **IAM Roles**: Use IRSA (IAM Roles for Service Accounts) instead of storing AWS credentials
2. **Network Policies**: Implement Kubernetes Network Policies to restrict pod-to-pod communication
3. **Secrets Management**: Consider AWS Secrets Manager integration for enhanced secret rotation
4. **Image Scanning**: Enable ECR image scanning for vulnerability detection
5. **Pod Security**: Implement Pod Security Standards (restricted profile)

## Rollback Procedure

If a deployment fails:

```bash
# View deployment history
kubectl rollout history deployment/coworking

# Rollback to previous version
kubectl rollout undo deployment/coworking

# Rollback to specific revision
kubectl rollout undo deployment/coworking --to-revision=2
```

## Technology Stack Summary

- **Application**: Python 3.10, Flask, SQLAlchemy, psycopg2
- **Container**: Docker with multi-stage builds
- **Orchestration**: Kubernetes 1.32 on AWS EKS
- **CI/CD**: AWS CodeBuild, GitHub webhooks
- **Registry**: Amazon ECR
- **Database**: PostgreSQL 15
- **Monitoring**: CloudWatch Container Insights, Fluent Bit
- **Infrastructure**: AWS (EKS, EC2, VPC, ELB, CloudWatch)

# Coworking Analytics Microservice

A cloud-native analytics microservice for coworking space management, deployed on AWS EKS with automated CI/CD pipelines.

## Table of Contents

- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [Technology Stack](#technology-stack)
- [Cluster Configuration](#cluster-configuration)
- [Deployment Process](#deployment-process)
- [CI/CD Pipeline](#cicd-pipeline)
- [Releasing New Builds](#releasing-new-builds)
- [Monitoring and Logging](#monitoring-and-logging)
- [Configuration Management](#configuration-management)
- [Scaling Strategy](#scaling-strategy)
- [Cost Optimization](#cost-optimization)
- [Security Best Practices](#security-best-practices)
- [Troubleshooting](#troubleshooting)

## Project Overview

This microservice provides business analytics APIs for coworking space administrators to track user activity, generate daily usage reports, and monitor user visits. The application is built with Python Flask and PostgreSQL, containerized with Docker, and deployed on AWS EKS with full automation via AWS CodeBuild.

### Key Features

- **Daily Usage Reports**: Aggregated check-ins grouped by date
- **User Visit Analytics**: User activity tracking and reporting
- **Automated CI/CD**: GitHub webhook integration with AWS CodeBuild
- **High Availability**: Kubernetes-based deployment with health checks
- **Comprehensive Monitoring**: CloudWatch Container Insights integration
- **Scalable Architecture**: Horizontal Pod Autoscaling support

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                         GitHub                               │
│                    (Source Control)                          │
└───────────────────────┬─────────────────────────────────────┘
                        │ Webhook Trigger
                        ▼
┌─────────────────────────────────────────────────────────────┐
│                    AWS CodeBuild                             │
│            (Build & Push Docker Image)                       │
└───────────────────────┬─────────────────────────────────────┘
                        │ Push Image
                        ▼
┌─────────────────────────────────────────────────────────────┐
│                       AWS ECR                                │
│               (Container Registry)                           │
└───────────────────────┬─────────────────────────────────────┘
                        │ Pull Image
                        ▼
┌─────────────────────────────────────────────────────────────┐
│                      AWS EKS Cluster                         │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  ┌──────────────┐       ┌──────────────┐            │   │
│  │  │  Application │◄──────┤   ConfigMap  │            │   │
│  │  │     Pod      │       │   & Secret   │            │   │
│  │  │  (Flask API) │       └──────────────┘            │   │
│  │  └──────┬───────┘                                    │   │
│  │         │                                            │   │
│  │         ▼                                            │   │
│  │  ┌──────────────┐                                   │   │
│  │  │  PostgreSQL  │                                   │   │
│  │  │     Pod      │                                   │   │
│  │  └──────────────┘                                   │   │
│  └──────────────────────────────────────────────────────┘   │
└───────────────────────┬─────────────────────────────────────┘
                        │ Logs & Metrics
                        ▼
┌─────────────────────────────────────────────────────────────┐
│                   AWS CloudWatch                             │
│              (Monitoring & Logging)                          │
└─────────────────────────────────────────────────────────────┘
```

## Technology Stack

| Component                 | Technology                    | Version |
| ------------------------- | ----------------------------- | ------- |
| **Application Framework** | Python Flask                  | 3.10+   |
| **Database**              | PostgreSQL                    | 15      |
| **Container Runtime**     | Docker                        | Latest  |
| **Orchestration**         | Kubernetes (EKS)              | 1.32+   |
| **CI/CD**                 | AWS CodeBuild                 | -       |
| **Container Registry**    | Amazon ECR                    | -       |
| **Monitoring**            | CloudWatch Container Insights | -       |
| **Log Aggregation**       | Fluent Bit                    | Latest  |
| **Infrastructure**        | AWS (EKS, EC2, VPC, ELB)      | -       |

### Application Dependencies

- **Flask**: Web application framework
- **SQLAlchemy**: ORM for database interactions
- **psycopg2-binary**: PostgreSQL adapter
- **boto3**: AWS SDK for Python (CloudWatch integration)

## Cluster Configuration

### EKS Cluster Specifications

The application runs on Amazon EKS with the following configuration:

- **Cluster Name**: `my-cluster`
- **Region**: `us-east-1`
- **Kubernetes Version**: 1.32+
- **Node Group**: `my-nodes`
- **Instance Type**: `t3.small` (2 vCPU, 2 GiB RAM)
- **Node Count**: 1-2 nodes (auto-scaling enabled)
- **Networking**: VPC with public and private subnets
- **Load Balancer**: AWS Classic Load Balancer (automatically provisioned)

### Cluster Creation Command

```bash
eksctl create cluster \
  --name my-cluster \
  --region us-east-1 \
  --nodegroup-name my-nodes \
  --node-type t3.small \
  --nodes 1 \
  --nodes-min 1 \
  --nodes-max 2
```

**Rationale for Configuration:**

- **t3.small instances**: Burstable performance ideal for I/O-bound applications with periodic batch processing. Provides baseline CPU with burst capability during report generation.
- **Auto-scaling (1-2 nodes)**: Enables cost optimization during off-peak hours while maintaining availability during high-demand periods.
- **Single region deployment**: Simplifies networking and reduces cross-region data transfer costs for MVP/development phase.

### Kubernetes Resources

The application deployment consists of:

1. **PostgreSQL Deployment** (`postgresql-deployment.yaml`):

   - 1 replica
   - ClusterIP service on port 5432
   - Ephemeral storage (for development; use PVC for production)
   - Environment variables for database initialization

2. **Application Deployment** (`deployment/coworking.yaml`):

   - 1 replica (configurable)
   - LoadBalancer service exposing port 5153
   - Liveness and readiness probes
   - Resource limits: 500Mi memory, 250m CPU

3. **ConfigMap** (`deployment/configmap.yaml`):

   - Database connection parameters
   - Non-sensitive environment variables

4. **Secret** (`deployment/secret.yaml`):
   - Base64-encoded database password
   - Sensitive configuration data

## Deployment Process

### Prerequisites

Before deploying, ensure you have:

1. **AWS CLI** configured with appropriate IAM permissions

   ```bash
   aws configure
   ```

2. **kubectl** installed and configured

   ```bash
   aws eks update-kubeconfig --region us-east-1 --name my-cluster
   ```

3. **eksctl** for EKS cluster management

   ```bash
   eksctl version
   ```

4. **Docker** (for local testing)
   ```bash
   docker --version
   ```

### Step-by-Step Deployment

#### 1. Create EKS Cluster

```bash
eksctl create cluster \
  --name my-cluster \
  --region us-east-1 \
  --nodegroup-name my-nodes \
  --node-type t3.small \
  --nodes 1 \
  --nodes-min 1 \
  --nodes-max 2
```

**Expected Time**: 15-20 minutes

Verify cluster creation:

```bash
kubectl get nodes
```

#### 2. Deploy PostgreSQL Database

```bash
# Apply PostgreSQL deployment and service
kubectl apply -f postgresql-deployment.yaml
kubectl apply -f postgresql-service.yaml

# Verify database pod is running
kubectl get pods -l app=postgresql

# Expected output:
# NAME                          READY   STATUS    RESTARTS   AGE
# postgresql-xxxxxxxxxx-xxxxx   1/1     Running   0          2m
```

#### 3. Initialize Database Schema

```bash
# Port forward to PostgreSQL service
kubectl port-forward service/postgresql-service 5432:5432 &

# Run seed files to create tables and populate data
PGPASSWORD=mypassword psql -h 127.0.0.1 -U myuser -d mydatabase -p 5432 -f db/1_create_tables.sql
PGPASSWORD=mypassword psql -h 127.0.0.1 -U myuser -d mydatabase -p 5432 -f db/2_seed_users.sql
PGPASSWORD=mypassword psql -h 127.0.0.1 -U myuser -d mydatabase -p 5432 -f db/3_seed_tokens.sql
```

#### 4. Deploy Application Configuration

```bash
# Create ConfigMap with database connection parameters
kubectl apply -f deployment/configmap.yaml

# Create Secret with database password (base64 encoded)
kubectl apply -f deployment/secret.yaml

# Verify configuration
kubectl get configmap coworking-configmap
kubectl get secret coworking-secret
```

#### 5. Deploy Application

```bash
# Deploy the Coworking Analytics application
kubectl apply -f deployment/coworking.yaml

# Verify deployment
kubectl get deployments
kubectl get pods
kubectl get services

# Expected output:
# NAME                         READY   STATUS    RESTARTS   AGE
# coworking-xxxxxxxxxx-xxxxx   1/1     Running   0          1m
# postgresql-xxxxxxxxxx-xxxxx  1/1     Running   0          5m
```

#### 6. Enable CloudWatch Container Insights

```bash
# Install CloudWatch namespace
kubectl apply -f https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/cloudwatch-namespace.yaml

# Deploy CloudWatch agent
kubectl apply -f cwagent-configmap.yaml

# Deploy Fluent Bit for log aggregation
kubectl apply -f fluent-bit-configmap.yaml
```

#### 7. Verify Deployment

```bash
# Get LoadBalancer external IP
kubectl get svc coworking

# Test API endpoints
curl http://<EXTERNAL-IP>:5153/api/reports/daily_usage
curl http://<EXTERNAL-IP>:5153/api/reports/user_visits
```

## CI/CD Pipeline

### Overview

The CI/CD pipeline uses **AWS CodeBuild** with **GitHub webhooks** for automated Docker image builds and deployments to ECR.

### Pipeline Architecture

```
Developer Push → GitHub → Webhook → CodeBuild → Build Image → Push to ECR → Update EKS
```

### CodeBuild Configuration

**Project Name**: `coworking-analytics-build`

**Environment Variables**:

- `AWS_ACCOUNT_ID`: 183302320810
- `AWS_DEFAULT_REGION`: us-east-1
- `IMAGE_REPO_NAME`: coworking

**Build Specification** (`buildspec.yaml`):

```yaml
version: 0.2
phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
  build:
    commands:
      - echo Build started on `date`
      - cd analytics
      - docker build -t $IMAGE_REPO_NAME:$CODEBUILD_BUILD_NUMBER .
      - docker tag $IMAGE_REPO_NAME:$CODEBUILD_BUILD_NUMBER $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$CODEBUILD_BUILD_NUMBER
      - docker tag $IMAGE_REPO_NAME:$CODEBUILD_BUILD_NUMBER $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:latest
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing Docker image to ECR...
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$CODEBUILD_BUILD_NUMBER
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:latest
      - printf '[{"name":"coworking","imageUri":"%s"}]' $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$CODEBUILD_BUILD_NUMBER > imagedefinitions.json
artifacts:
  files:
    - imagedefinitions.json
```

### GitHub Integration

1. **Webhook Setup**: CodeBuild project configured with GitHub source provider
2. **Trigger**: Automatic builds on push to `main` branch
3. **Branch Filter**: `main` (configurable to other branches)
4. **Authentication**: GitHub personal access token or OAuth

### Build Process Flow

1. Developer commits code changes to `analytics/` directory
2. Developer pushes to GitHub `main` branch
3. GitHub webhook triggers CodeBuild project automatically
4. CodeBuild:
   - Authenticates with ECR
   - Builds Docker image from `analytics/Dockerfile`
   - Tags image with:
     - Build number: `$CODEBUILD_BUILD_NUMBER`
     - Commit SHA: `$CODEBUILD_RESOLVED_SOURCE_VERSION`
     - Latest: `latest`
   - Pushes all tags to ECR
   - Generates `imagedefinitions.json` artifact
5. Build artifacts stored in S3 (automatic by CodeBuild)

## Releasing New Builds

### Automated Release (Recommended)

**For releasing new features or bug fixes:**

1. **Make Code Changes**:

   ```bash
   cd analytics/
   # Edit app.py, config.py, or other files
   ```

2. **Commit and Push**:

   ```bash
   git add .
   git commit -m "feat: add new analytics endpoint"
   git push origin main
   ```

3. **Monitor Build**:

   ```bash
   # Check CodeBuild console or use AWS CLI
   aws codebuild list-builds-for-project --project-name coworking-analytics-build
   ```

4. **Update Kubernetes Deployment**:

   ```bash
   # Option 1: Set specific image version
   kubectl set image deployment/coworking \
     coworking=183302320810.dkr.ecr.us-east-1.amazonaws.com/coworking:<BUILD_NUMBER>

   # Option 2: Force rolling update to pull latest
   kubectl rollout restart deployment/coworking

   # Monitor rollout status
   kubectl rollout status deployment/coworking
   ```

### Manual Release (For Testing)

**For local development and testing:**

```bash
# 1. Build Docker image locally
cd analytics
docker build -t coworking:test .

# 2. Test locally (optional)
docker run -p 5153:5153 \
  -e DB_HOST=host.docker.internal \
  -e DB_PORT=5432 \
  -e DB_USERNAME=myuser \
  -e DB_PASSWORD=mypassword \
  -e DB_NAME=mydatabase \
  coworking:test

# 3. Tag for ECR
docker tag coworking:test \
  183302320810.dkr.ecr.us-east-1.amazonaws.com/coworking:manual-$(date +%Y%m%d-%H%M%S)

# 4. Authenticate with ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  183302320810.dkr.ecr.us-east-1.amazonaws.com

# 5. Push to ECR
docker push 183302320810.dkr.ecr.us-east-1.amazonaws.com/coworking:manual-$(date +%Y%m%d-%H%M%S)

# 6. Update Kubernetes deployment
kubectl set image deployment/coworking \
  coworking=183302320810.dkr.ecr.us-east-1.amazonaws.com/coworking:manual-<TIMESTAMP>
```

### Semantic Versioning Strategy

For production releases, implement semantic versioning:

```bash
# Tag with semantic version
git tag v1.2.3
git push origin v1.2.3

# Update buildspec.yaml to use git tags
# Then CodeBuild will tag images as:
# - 183302320810.dkr.ecr.us-east-1.amazonaws.com/coworking:v1.2.3
# - 183302320810.dkr.ecr.us-east-1.amazonaws.com/coworking:v1.2
# - 183302320810.dkr.ecr.us-east-1.amazonaws.com/coworking:v1
# - 183302320810.dkr.ecr.us-east-1.amazonaws.com/coworking:latest
```

**Version Format**: `MAJOR.MINOR.PATCH`

- **MAJOR**: Breaking changes
- **MINOR**: New features (backward compatible)
- **PATCH**: Bug fixes (backward compatible)

### Rollback Procedure

If a deployment introduces issues:

```bash
# View deployment history
kubectl rollout history deployment/coworking

# Rollback to previous version
kubectl rollout undo deployment/coworking

# Rollback to specific revision
kubectl rollout undo deployment/coworking --to-revision=3

# Verify rollback
kubectl rollout status deployment/coworking
```

## Monitoring and Logging

### CloudWatch Container Insights

**Enabled Features**:

- Pod-level metrics (CPU, memory, network)
- Node-level metrics
- Service-level metrics
- Automatic dashboard creation

**Accessing Metrics**:

1. Navigate to AWS CloudWatch Console
2. Select "Container Insights" from left menu
3. Choose cluster: `my-cluster`
4. View metrics by: Resources, Performance, or Alarms

### Application Logs

**View Real-Time Logs**:

```bash
# Stream logs from coworking pod
kubectl logs -f deployment/coworking

# View logs from specific pod
kubectl logs <POD_NAME> --tail=100

# View logs from all pods with label
kubectl logs -l app=coworking --all-containers=true
```

**CloudWatch Logs**:

```bash
# Tail CloudWatch logs
aws logs tail /aws/containerinsights/my-cluster/application --follow

# Filter logs for coworking namespace
aws logs filter-log-events \
  --log-group-name /aws/containerinsights/my-cluster/application \
  --filter-pattern "coworking"
```

**Expected Log Output** (CloudWatch):

```
[INFO] Database connection established
[INFO] Fetching daily usage report...
[DEBUG] Query: SELECT DATE(created_at), COUNT(*) FROM tokens GROUP BY DATE(created_at)
[INFO] Daily usage report generated: 15 records
[INFO] 200 GET /api/reports/daily_usage
```

The application **periodically queries the database** and logs the results, confirming successful database connectivity and operation.

### Health Checks

The application includes built-in health check endpoints:

```yaml
# Configured in deployment/coworking.yaml
livenessProbe:
  httpGet:
    path: /health
    port: 5153
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: 5153
  initialDelaySeconds: 5
  periodSeconds: 5
```

## Configuration Management

### Environment Variables (ConfigMap)

Non-sensitive configuration stored in ConfigMap:

```yaml
# deployment/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coworking-configmap
data:
  DB_HOST: "postgresql-service"
  DB_PORT: "5432"
  DB_USERNAME: "myuser"
  DB_NAME: "mydatabase"
```

**Updating ConfigMap**:

```bash
# Edit ConfigMap
kubectl edit configmap coworking-configmap

# Or apply updated file
kubectl apply -f deployment/configmap.yaml

# Restart deployment to pick up changes
kubectl rollout restart deployment/coworking
```

### Secrets Management

Sensitive data stored in Kubernetes Secrets:

```yaml
# deployment/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: coworking-secret
type: Opaque
data:
  DB_PASSWORD: <base64-encoded-password>
```

**Creating/Updating Secret**:

```bash
# Create secret from literal value
kubectl create secret generic coworking-secret \
  --from-literal=DB_PASSWORD=mypassword

# Or encode and apply YAML
echo -n 'mypassword' | base64
# Output: bXlwYXNzd29yZA==

kubectl apply -f deployment/secret.yaml

# Restart to apply changes
kubectl rollout restart deployment/coworking
```

**Best Practice**: For production, integrate with **AWS Secrets Manager**:

```bash
# Install External Secrets Operator
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets

# Create ExternalSecret resource pointing to AWS Secrets Manager
```

## Scaling Strategy

### Horizontal Pod Autoscaling (HPA)

**Enable HPA based on CPU utilization**:

```bash
# Create HPA for coworking deployment
kubectl autoscale deployment coworking \
  --cpu-percent=70 \
  --min=1 \
  --max=5

# Verify HPA
kubectl get hpa
```

**Custom Metrics HPA** (based on request rate):

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: coworking-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: coworking
  minReplicas: 1
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

### Manual Scaling

```bash
# Scale to 3 replicas
kubectl scale deployment coworking --replicas=3

# Verify scaling
kubectl get pods -l app=coworking
```

### Cluster Node Autoscaling

The EKS cluster auto-scales between 1-2 nodes based on resource demands.

**Modify Autoscaling Limits**:

```bash
eksctl scale nodegroup \
  --cluster=my-cluster \
  --name=my-nodes \
  --nodes=2 \
  --nodes-min=1 \
  --nodes-max=4
```

## Cost Optimization

### Current Monthly Cost Estimate

| Resource                  | Configuration          | Monthly Cost       |
| ------------------------- | ---------------------- | ------------------ |
| EKS Cluster               | 1 cluster              | $72                |
| EC2 Instances (t3.small)  | 1-2 nodes @ $0.0208/hr | $15-30             |
| Application Load Balancer | 1 ALB                  | $16                |
| ECR Storage               | ~5GB images            | $0.50              |
| CloudWatch Logs           | 5GB/month              | $2.50              |
| Data Transfer             | 10GB/month             | $0.90              |
| **Total**                 |                        | **$107-122/month** |

### Cost Saving Strategies

#### 1. Right-Size Instance Types

**Current**: `t3.small` (2 vCPU, 2 GiB RAM)

- Perfectly sized for I/O-bound workload with periodic batch processing
- Burstable performance handles traffic spikes without over-provisioning
- **Savings**: Already optimized (vs. t3.medium saves $15/month per node)

#### 2. Implement Spot Instances

**For non-production environments:**

```bash
eksctl create nodegroup \
  --cluster=my-cluster \
  --name=spot-nodes \
  --node-type=t3.small \
  --nodes=1 \
  --nodes-min=1 \
  --nodes-max=3 \
  --spot
```

**Savings**: Up to 90% on EC2 costs ($15-30/month → $2-3/month)

#### 3. Enable Cluster Autoscaler

Auto-scale nodes to 0 during off-peak hours:

```bash
# Install Cluster Autoscaler
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
```

**Savings**: $10-15/day during off-hours (nights/weekends)

#### 4. ECR Lifecycle Policies

Automatically delete old Docker images:

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep last 10 images",
      "selection": {
        "tagStatus": "any",
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": {
        "type": "expire"
      }
    }
  ]
}
```

**Savings**: $0.10-0.50/month on storage

#### 5. CloudWatch Log Retention

Set log retention to 7-30 days:

```bash
aws logs put-retention-policy \
  --log-group-name /aws/containerinsights/my-cluster/application \
  --retention-in-days 7
```

**Savings**: $1-2/month on storage

#### 6. Use AWS Savings Plans

Purchase 1-year compute savings plan for 40% discount on EC2 costs.

**Savings**: $6-12/month on compute

#### 7. Optimize Load Balancer

Use Network Load Balancer instead of Classic Load Balancer for lower cost:

**Savings**: $5/month

### Total Potential Savings

| Strategy                  | Savings            |
| ------------------------- | ------------------ |
| Spot Instances (non-prod) | $12-27/month       |
| Cluster Autoscaler        | $150-225/month     |
| ECR Lifecycle             | $0.50/month        |
| CloudWatch Retention      | $1-2/month         |
| Savings Plans             | $6-12/month        |
| NLB vs CLB                | $5/month           |
| **Total**                 | **$175-272/month** |

**Net Cost After Optimization**: $50-70/month for production workload

## Security Best Practices

### 1. IAM Roles for Service Accounts (IRSA)

Instead of storing AWS credentials in pods:

```bash
# Create IAM OIDC provider
eksctl utils associate-iam-oidc-provider \
  --cluster=my-cluster \
  --approve

# Create IAM role for service account
eksctl create iamserviceaccount \
  --name coworking-sa \
  --namespace default \
  --cluster my-cluster \
  --attach-policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy \
  --approve
```

### 2. Network Policies

Restrict pod-to-pod communication:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: coworking-network-policy
spec:
  podSelector:
    matchLabels:
      app: coworking
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: coworking
      ports:
        - protocol: TCP
          port: 5153
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: postgresql
      ports:
        - protocol: TCP
          port: 5432
```

### 3. Secrets Management with AWS Secrets Manager

```bash
# Create secret in Secrets Manager
aws secretsmanager create-secret \
  --name coworking/db-password \
  --secret-string "mypassword"

# Use External Secrets Operator to sync
```

### 4. Enable ECR Image Scanning

```bash
aws ecr put-image-scanning-configuration \
  --repository-name coworking \
  --image-scanning-configuration scanOnPush=true
```

### 5. Pod Security Standards

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: coworking
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
  containers:
    - name: coworking
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL
```

### 6. Regular Security Audits

```bash
# Audit Kubernetes configuration
kubectl auth can-i --list

# Scan for vulnerabilities
trivy image 183302320810.dkr.ecr.us-east-1.amazonaws.com/coworking:latest
```

## Troubleshooting

### Common Issues and Solutions

#### Issue: Pod Not Starting

```bash
# Check pod status
kubectl describe pod <POD_NAME>

# Common causes:
# - ImagePullBackOff: Check ECR permissions
# - CrashLoopBackOff: Check application logs
# - Pending: Check node resources

# View events
kubectl get events --sort-by='.lastTimestamp'
```

#### Issue: Database Connection Failed

```bash
# Verify PostgreSQL service
kubectl get svc postgresql-service

# Test connectivity from app pod
kubectl exec -it <APP_POD> -- curl postgresql-service:5432

# Check database logs
kubectl logs <POSTGRESQL_POD>

# Verify secret/configmap
kubectl get configmap coworking-configmap -o yaml
kubectl get secret coworking-secret -o yaml
```

#### Issue: LoadBalancer Not Accessible

```bash
# Check service external IP
kubectl get svc coworking

# If <pending>, check AWS Load Balancer Controller
kubectl get deployment -n kube-system aws-load-balancer-controller

# Verify security groups allow traffic on port 5153
aws ec2 describe-security-groups --filters "Name=tag:kubernetes.io/cluster/my-cluster,Values=owned"
```

#### Issue: High Memory Usage

```bash
# Check pod resource usage
kubectl top pods

# Increase memory limits in deployment
kubectl edit deployment coworking

# Update resources:
#   limits:
#     memory: "1Gi"
#   requests:
#     memory: "512Mi"
```

#### Issue: CodeBuild Fails

```bash
# View build logs
aws codebuild batch-get-builds --ids <BUILD_ID>

# Common causes:
# - ECR authentication failure: Check IAM permissions
# - Dockerfile errors: Test locally with `docker build`
# - Insufficient build resources: Upgrade CodeBuild instance type
```

### Debug Commands Cheatsheet

```bash
# Get all resources
kubectl get all

# Describe pod with details
kubectl describe pod <POD_NAME>

# Execute command in pod
kubectl exec -it <POD_NAME> -- /bin/bash

# Port forward for local testing
kubectl port-forward svc/coworking 5153:5153

# View resource usage
kubectl top nodes
kubectl top pods

# Check cluster info
kubectl cluster-info

# View deployment rollout status
kubectl rollout status deployment/coworking

# Scale deployment
kubectl scale deployment coworking --replicas=2

# Restart deployment
kubectl rollout restart deployment/coworking
```

## Contributing

For bug reports, feature requests, or contributions, please contact the DevOps team.

## License

This project is proprietary and confidential.

---

**Last Updated**: October 2024  
**Maintained By**: DevOps Team  
**Contact**: devops@example.com

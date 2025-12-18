# Helm Chart Deployment Guide

Complete documentation for deploying the GKE application with Cloud SQL Proxy using Helm.

## PostgreSQL Server Details

```
Instance Name:       Mypostgresql
Connection Name:     ornate-producer-477604-s3:us-central1:mypostgresql
Public IP Address:   34.123.230.140
Port:                5432
Username:            Mypostgresql
Password:            Mypostgre@123
Database Name:       Mypostgresql
Region:              us-central1
Type:                PostgreSQL 17
High Availability:   Enable
```

## Prerequisites

### Local Development
- Helm 3.0+
- kubectl 1.20+
- gcloud CLI
- Docker (optional, for building custom images)

### GKE Cluster Requirements
- GKE cluster named `gke-cluster` in `us-central1` region
- Kubernetes 1.20+
- Workload Identity enabled (for GitHub Actions)
- Cloud SQL Admin API enabled

## Project Structure

```
.
├── helm-charts/
│   └── gke-app/
│       ├── Chart.yaml                 # Helm chart metadata
│       ├── values.yaml                # Default values
│       ├── values-dev.yaml           # Development overrides
│       ├── values-prod.yaml          # Production overrides
│       └── templates/
│           ├── namespace.yaml         # Kubernetes namespace
│           ├── secret.yaml            # Cloud SQL credentials
│           ├── serviceaccount.yaml   # Service account
│           ├── deployment.yaml        # Main deployment with 2 containers
│           ├── service.yaml           # Kubernetes service
│           ├── ingress.yaml          # Ingress configuration
│           └── _helpers.tpl          # Helm template helpers
├── .github/
│   └── workflows/
│       ├── deploy-gke.yaml           # Main deployment pipeline
│       └── postgres-health-check.yaml # Scheduled health checks
└── README.md
```

## Deployment Architecture

### Pods
The deployment creates pods with 2 containers:

1. **Application Container**
   - Image: `nginx:latest` (replace with your app image)
   - Port: 8080
   - Environment variables for database connection

2. **Cloud SQL Proxy Container**
   - Image: `gcr.io/cloudsql-docker/cloud-sql-proxy:2.4.0`
   - Port: 5432
   - Secure connection to Cloud SQL instance

### Data Flow
```
Application Pod
├── app container (localhost:8080)
└── cloud-sql-proxy container (localhost:5432)
    └── Cloud SQL Instance (34.123.230.140:5432)
```

## Local Helm Commands

### 1. Lint the Chart
```bash
helm lint helm-charts/gke-app/
```

### 2. Validate Templates
```bash
helm template gke-app helm-charts/gke-app/ \
  --values helm-charts/gke-app/values.yaml
```

### 3. Dry Run
```bash
helm install gke-app helm-charts/gke-app/ \
  --namespace default \
  --values helm-charts/gke-app/values.yaml \
  --set cloudsql.password='Mypostgre@123' \
  --dry-run \
  --debug
```

### 4. Install Release
```bash
helm install gke-app helm-charts/gke-app/ \
  --namespace default \
  --values helm-charts/gke-app/values.yaml \
  --set cloudsql.password='Mypostgre@123'
```

### 5. Upgrade Release
```bash
helm upgrade gke-app helm-charts/gke-app/ \
  --namespace default \
  --values helm-charts/gke-app/values.yaml \
  --set cloudsql.password='Mypostgre@123'
```

### 6. Uninstall Release
```bash
helm uninstall gke-app --namespace default
```

## GitHub Actions Setup

### Prerequisites

1. **GCP Service Account Setup**
   ```bash
   # Create service account
   gcloud iam service-accounts create github-actions
   
   # Grant necessary permissions
   gcloud projects add-iam-policy-binding ornate-producer-477604-s3 \
     --member=serviceAccount:github-actions@ornate-producer-477604-s3.iam.gserviceaccount.com \
     --role=roles/container.developer
   
   gcloud projects add-iam-policy-binding ornate-producer-477604 \
     --member=serviceAccount:github-actions@ornate-producer-477604.iam.gserviceaccount.com \
     --role=roles/cloudsql.admin
   ```

2. **Configure Workload Identity**
   ```bash
   # Create identity pool and provider
   gcloud iam workload-identity-pools create "github" \
     --project="ornate-producer-477604" \
     --location="global" \
     --display-name="GitHub Actions"
   
   # Create service account attribute mapping
   gcloud iam workload-identity-pools providers create-oidc "github" \
     --project="ornate-producer-477604" \
     --location="global" \
     --workload-identity-pool="github" \
     --display-name="GitHub" \
     --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor,attribute.aud=assertion.aud,attribute.repository=assertion.repository" \
     --issuer-uri="https://token.actions.githubusercontent.com" \
     --attribute-condition="assertion.repository == 'Amolghadge/GKE-PostgreSql-CloudSql-proxy-deployment'"
   ```

3. **Grant permissions to GitHub**
   ```bash
   gcloud iam service-accounts add-iam-policy-binding \
     github-actions@ornate-producer-477604-s3.iam.gserviceaccount.com \
     --project="ornate-producer-477604-s3" \
     --role="roles/iam.workloadIdentityUser" \
     --member="principalSet://iam.googleapis.com/projects/123456789/locations/global/workloadIdentityPools/github/attribute.repository/Amolghadge/GKE-PostgreSql-CloudSql-proxy-deployment"
   ```

### GitHub Secrets Configuration

Add the following secrets to your GitHub repository:

1. **WIF_PROVIDER** - Workload Identity Provider
   ```
   projects/123456789/locations/global/workloadIdentityPools/github/providers/github
   ```

2. **WIF_SERVICE_ACCOUNT** - Service Account Email
   ```
   github-actions@ornate-producer-477604-s3.iam.gserviceaccount.com
   ```

3. **CLOUDSQL_PASSWORD** - PostgreSQL Password
   ```
   Mypostgre@123
   ```

4. **SLACK_WEBHOOK** (Optional) - Slack notifications
   ```
   https://hooks.slack.com/services/YOUR/WEBHOOK/URL
   ```

## Deployment Workflows

### Manual Deployment
```bash
# Via GitHub UI: Actions > Deploy to GKE > Run workflow
# Select environment: dev or prod
```

### Automatic Deployment
- **Push to main branch** → Deploy to production
- **Push to develop branch** → Deploy to development
- **Pull request** → Lint only (no deployment)

## Kubernetes Deployment Verification

### Check Deployment Status
```bash
kubectl get deployment -n default
kubectl get pods -n default
kubectl describe deployment gke-app -n default
```

### View Logs
```bash
# Application logs
kubectl logs -n default -l app=app -c app

# Cloud SQL Proxy logs
kubectl logs -n default -l app=app -c cloud-sql-proxy
```

### Port Forwarding (Local Testing)
```bash
# Forward service to localhost
kubectl port-forward svc/app-service 8080:80 -n default

# Access application
curl http://localhost:8080
```

### Test Database Connection
```bash
# Get pod name
POD=$(kubectl get pod -n default -l app=app -o jsonpath='{.items[0].metadata.name}')

# Test connection to Cloud SQL Proxy
kubectl exec -n default $POD -c app -- \
  sh -c "nc -zv localhost 5432"
```

## Configuration Customization

### Application Configuration
Edit `helm-charts/gke-app/values.yaml`:
```yaml
application:
  replicaCount: 1
  image:
    repository: your-image  # Your application image
    tag: "latest"
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
```

### Database Connection
Edit `helm-charts/gke-app/values.yaml`:
```yaml
cloudsql:
  host: "34.123.230.140"
  port: 5432
  database: "Mypostgresql"
  username: "Mypostgresql"
```

### Environment-Specific Overrides
- **Development**: `values-dev.yaml`
- **Production**: `values-prod.yaml`

## Troubleshooting

### Pod fails to start
```bash
# Check events
kubectl describe pod <pod-name> -n default

# Check logs
kubectl logs <pod-name> -n default --all-containers
```

### Cloud SQL Proxy connection issues
```bash
# Check proxy status
kubectl logs -n default -l app=app -c cloud-sql-proxy

# Verify Cloud SQL instance is accessible
gcloud sql instances describe mypostgresql --project=ornate-producer-477604-s3
```

### Service not accessible
```bash
# Check service
kubectl get svc -n default

# Check endpoints
kubectl get endpoints -n default

# Port forward and test
kubectl port-forward svc/app-service 8080:80 -n default
```

## Security Best Practices

1. **Secrets Management**
   - Store passwords in GitHub Secrets, not in code
   - Rotate Cloud SQL passwords regularly

2. **Network Policies**
   - Implement network policies to restrict pod communication
   - Use Cloud SQL Private IP when possible

3. **RBAC**
   - Service account has minimal required permissions
   - Review and update roles as needed

4. **Image Security**
   - Use specific image tags (not `latest`)
   - Scan images for vulnerabilities
   - Use private container registries

## Cleanup

### Remove Helm Release
```bash
helm uninstall gke-app -n default
```

### Delete Namespace
```bash
kubectl delete namespace default
```

### Remove GitHub Actions
- Delete `.github/workflows/` files or disable in repository settings

## Support and Documentation

- [Helm Documentation](https://helm.sh/docs/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [GKE Documentation](https://cloud.google.com/kubernetes-engine/docs)
- [Cloud SQL Proxy Documentation](https://cloud.google.com/sql/docs/postgres/cloud-sql-proxy)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)

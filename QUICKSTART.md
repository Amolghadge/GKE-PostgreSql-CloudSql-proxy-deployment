# Quick Start Guide

Deploy the Helm chart to GKE in 5 minutes.

## Prerequisites

```bash
# Install required tools
gcloud components install gke-gcloud-auth-plugin
brew install helm kubectl  # macOS
# or use: choco install kubernetes-cli helm  # Windows

# Login to GCP
gcloud auth login
gcloud config set project ornate-producer-477604-s3
```

## PostgreSQL Connection Details

```yaml
Instance Name:       Mypostgresql
Connection String:   ornate-producer-477604-s3:us-central1:mypostgresql
Public IP:           34.123.230.140
Port:                5432
Username:            Mypostgresql
Password:            Mypostgre@123
Database:            Mypostgresql
Region:              us-central1
Type:                PostgreSQL 17
```

## Step 1: Get GKE Credentials

```bash
gcloud container clusters get-credentials gke-cluster \
  --region us-central1 \
  --project ornate-producer-477604-s3
```

## Step 2: Create Namespace and Secret

```bash
# Create namespace
kubectl create namespace default --dry-run=client -o yaml | kubectl apply -f -

# Create Cloud SQL credentials secret
kubectl create secret generic cloudsql-credentials \
  --from-literal=username='Mypostgresql' \
  --from-literal=password='Mypostgre@123' \
  --namespace=default \
  --dry-run=client -o yaml | kubectl apply -f -
```

## Step 3: Deploy with Helm

```bash
# Install the release
helm install gke-app helm-charts/gke-app/ \
  --namespace default \
  --values helm-charts/gke-app/values.yaml \
  --set cloudsql.password='Mypostgre@123'

# Wait for deployment
kubectl rollout status deployment/app -n default --timeout=5m
```

## Step 4: Verify Deployment

```bash
# Check pods
kubectl get pods -n default

# Check services
kubectl get svc -n default

# Check logs
kubectl logs -n default -l app=app -c cloud-sql-proxy --tail=20
```

## Step 5: Test Connectivity

```bash
# Get pod name
POD=$(kubectl get pod -n default -l app=app -o jsonpath='{.items[0].metadata.name}')

# Test Cloud SQL Proxy
kubectl exec -n default $POD -c cloud-sql-proxy -- \
  sh -c "echo 'Cloud SQL Proxy is running'"

# Port forward for local testing
kubectl port-forward svc/app-service 8080:80 -n default
```

## Upgrade the Release

```bash
helm upgrade gke-app helm-charts/gke-app/ \
  --namespace default \
  --values helm-charts/gke-app/values.yaml \
  --set cloudsql.password='Mypostgre@123'
```

## Rollback a Release

```bash
# Rollback to previous version
helm rollback gke-app -n default

# Or rollback to specific revision
helm rollback gke-app 1 -n default
```

## Uninstall the Release

```bash
helm uninstall gke-app -n default
```

## Check Revision History

```bash
helm history gke-app -n default
```

## View Helm Release Status

```bash
helm status gke-app -n default
```

## Get Release Values

```bash
helm get values gke-app -n default
```

## Troubleshooting

### Pods not starting?
```bash
kubectl describe pod <pod-name> -n default
kubectl logs <pod-name> -n default --all-containers
```

### Cloud SQL Proxy not connecting?
```bash
kubectl logs -n default -l app=app -c cloud-sql-proxy
```

### Service not accessible?
```bash
kubectl get svc -n default
kubectl describe svc app-service -n default
```

## Next Steps

1. Replace `nginx:latest` with your actual application image
2. Configure health check endpoints (`/health`, `/ready`)
3. Set up monitoring and logging
4. Configure GitHub Actions secrets (see GITHUB_SECRETS_SETUP.md)
5. Push to main branch to trigger automatic deployment

For detailed documentation, see: `HELM_DEPLOYMENT_GUIDE.md`

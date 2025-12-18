# GitHub Actions Secrets Setup Guide

This guide explains how to set up the required GitHub Actions secrets for deploying to GKE.

## Required Secrets

### 1. WIF_PROVIDER
Workload Identity Federation Provider ID for Google Cloud authentication.

**Format:**
```
projects/{PROJECT_NUMBER}/locations/global/workloadIdentityPools/{POOL_NAME}/providers/{PROVIDER_NAME}
```

**Example:**
```
projects/123456789/locations/global/workloadIdentityPools/github/providers/github
```

**How to find:**
```bash
gcloud iam workload-identity-pools describe github \
  --project="ornate-producer-477604" \
  --location="global" \
  --format="value(name)"
```

### 2. WIF_SERVICE_ACCOUNT
Google Cloud Service Account email for Workload Identity.

**Format:**
```
{SERVICE_ACCOUNT_NAME}@{PROJECT_ID}.iam.gserviceaccount.com
```

**Example:**
```
github-actions@ornate-producer-477604.iam.gserviceaccount.com
```

### 3. CLOUDSQL_PASSWORD
PostgreSQL password for authentication.

**Value:**
```
Mypostgre@123
```

### 4. SLACK_WEBHOOK (Optional)
Slack webhook URL for notifications.

**Format:**
```
https://hooks.slack.com/services/{YOUR}/{WEBHOOK}/{URL}
```

## Setup Instructions

### Step 1: Configure GCP Service Account

```bash
# Set variables
PROJECT_ID="ornate-producer-477604-s3"
SERVICE_ACCOUNT_NAME="github-actions"
POOL_NAME="github"
PROVIDER_NAME="github"

# Create service account (if not exists)
gcloud iam service-accounts create ${SERVICE_ACCOUNT_NAME} \
  --project=${PROJECT_ID}

# Grant GKE developer role
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member=serviceAccount:${SERVICE_ACCOUNT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com \
  --role=roles/container.developer

# Grant Cloud SQL client role
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member=serviceAccount:${SERVICE_ACCOUNT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com \
  --role=roles/cloudsql.client
```

### Step 2: Create Workload Identity Pool

```bash
gcloud iam workload-identity-pools create ${POOL_NAME} \
  --project=${PROJECT_ID} \
  --location=global \
  --display-name="GitHub Actions"
```

### Step 3: Create OIDC Provider

```bash
gcloud iam workload-identity-pools providers create-oidc ${PROVIDER_NAME} \
  --project=${PROJECT_ID} \
  --location=global \
  --workload-identity-pool=${POOL_NAME} \
  --display-name="GitHub" \
  --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor,attribute.aud=assertion.aud,attribute.repository=assertion.repository" \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --attribute-condition="assertion.repository == 'Amolghadge/GKE-PostgreSql-CloudSql-proxy-deployment'"
```

### Step 4: Configure Workload Identity Binding

```bash
# Get project number
PROJECT_NUMBER=$(gcloud projects describe ${PROJECT_ID} --format='value(projectNumber)')

# Add IAM binding
gcloud iam service-accounts add-iam-policy-binding \
  ${SERVICE_ACCOUNT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com \
  --project=${PROJECT_ID} \
  --role=roles/iam.workloadIdentityUser \
  --member="principalSet://iam.googleapis.com/projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/${POOL_NAME}/attribute.repository/Amolghadge/GKE-PostgreSql-CloudSql-proxy-deployment"
```

### Step 5: Get Provider ID

```bash
gcloud iam workload-identity-pools providers describe ${PROVIDER_NAME} \
  --project=${PROJECT_ID} \
  --location=global \
  --workload-identity-pool=${POOL_NAME} \
  --format="value(name)"
```

This will output the WIF_PROVIDER value.

### Step 6: Add Secrets to GitHub

1. Go to your GitHub repository
2. Settings → Secrets and variables → Actions
3. Create new repository secrets:
   - **WIF_PROVIDER**: `projects/{PROJECT_NUMBER}/locations/global/workloadIdentityPools/github/providers/github`
   - **WIF_SERVICE_ACCOUNT**: `github-actions@ornate-producer-477604-s3.iam.gserviceaccount.com`
   - **CLOUDSQL_PASSWORD**: `Mypostgre@123`
   - **SLACK_WEBHOOK**: (Optional) Your Slack webhook URL

## Verification

### Test Workload Identity
```bash
# Create a test pod
kubectl run test-pod --image=google/cloud-sdk:slim --restart=Never -it

# Inside the pod, try to authenticate
gcloud auth application-default print-access-token
```

### Test GitHub Actions
1. Push a change to trigger the workflow
2. Go to Actions tab in GitHub
3. Check if the deployment succeeds
4. Review the logs for any errors

## Troubleshooting

### "Permission denied" errors
- Verify service account has correct roles
- Check Workload Identity binding is correct
- Ensure OIDC provider issuer-uri is correct

### "Invalid token" errors
- Verify WIF_PROVIDER value is correct
- Check GitHub repository name matches the condition
- Ensure OIDC provider is properly configured

### "Cloud SQL connection failed"
- Verify CLOUDSQL_PASSWORD is correct
- Check Cloud SQL instance is accessible from GKE cluster
- Verify Cloud SQL Proxy connection name is correct

## Additional Resources

- [Workload Identity Documentation](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity)
- [GitHub Actions with Google Cloud](https://github.com/google-github-actions)
- [Cloud SQL Proxy Documentation](https://cloud.google.com/sql/docs/postgres/cloud-sql-proxy)

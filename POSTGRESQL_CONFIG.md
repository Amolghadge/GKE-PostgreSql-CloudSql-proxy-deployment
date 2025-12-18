# PostgreSQL Server Configuration Reference

Complete details of the PostgreSQL server you'll be connecting to.

## Connection Details

| Property | Value |
|----------|-------|
| **Instance Name** | Mypostgresql |
| **Instance ID** | mypostgresql |
| **Connection Name** | ornate-producer-477604-s3:us-central1:mypostgresql |
| **Public IP Address** | 34.123.230.140 |
| **Private IP Address** | (To be configured) |
| **TCP Port** | 5432 |
| **Region** | us-central1 |
| **Zone** | us-central1-c |
| **Database Engine** | PostgreSQL 17 |
| **Machine Type** | db-f1-micro (or similar) |

## Authentication Details

| Property | Value |
|----------|-------|
| **Default User** | Mypostgresql |
| **Password** | Mypostgre@123 |
| **Database Name** | Mypostgresql |

## Network Configuration

| Property | Value |
|----------|-------|
| **Public IP Enabled** | Yes - 34.123.230.140 |
| **High Availability** | Enabled |
| **Backup Configuration** | Daily automated backups |
| **SSL/TLS** | Required for external connections |

## Cloud SQL Proxy Connection String

For use in Kubernetes:
```
ornate-producer-477604-s3:us-central1:mypostgresql
```

## Connection Methods

### 1. Cloud SQL Proxy (Recommended for GKE)
```bash
cloud_sql_proxy -instances=ornate-producer-477604-s3:us-central1:mypostgresql=tcp:5432
```

### 2. Public IP Connection
```bash
psql -h 34.123.230.140 -U Mypostgresql -d Mypostgresql -p 5432
```

### 3. Private IP Connection (When configured)
```bash
psql -h <private-ip> -U Mypostgresql -d Mypostgresql -p 5432
```

## Environment Variables

For application configuration:
```bash
export DB_HOST=localhost          # When using Cloud SQL Proxy
export DB_PORT=5432
export DB_NAME=Mypostgresql
export DB_USER=Mypostgresql
export DB_PASSWORD=Mypostgre@123
```

## Connection Test Commands

### Using Cloud SQL Proxy
```bash
gcloud sql connect mypostgresql \
  --user=Mypostgresql \
  --project=ornate-producer-477604
```

### Using psql directly
```bash
psql "postgresql://Mypostgresql:Mypostgre@123@34.123.230.140:5432/Mypostgresql"
```

### Using Python
```python
import psycopg2

conn = psycopg2.connect(
    host="34.123.230.140",
    port=5432,
    database="Mypostgresql",
    user="Mypostgresql",
    password="Mypostgre@123"
)
```

### Using Node.js
```javascript
const { Client } = require('pg');

const client = new Client({
  host: '34.123.230.140',
  port: 5432,
  database: 'Mypostgresql',
  user: 'Mypostgresql',
  password: 'Mypostgre@123'
});
```

## GKE Application Connection

When deploying to GKE with Cloud SQL Proxy:

```yaml
env:
  - name: DB_HOST
    value: "localhost"  # Cloud SQL Proxy runs on same pod
  - name: DB_PORT
    value: "5432"
  - name: DB_NAME
    value: "Mypostgresql"
  - name: DB_USER
    valueFrom:
      secretKeyRef:
        name: cloudsql-credentials
        key: username
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: cloudsql-credentials
        key: password
```

## Kubernetes Secret Creation

```bash
kubectl create secret generic cloudsql-credentials \
  --from-literal=username='Mypostgresql' \
  --from-literal=password='Mypostgre@123' \
  --namespace=default
```

## GCP Project Details

| Property | Value |
|----------|-------|
| **Project ID** | ornate-producer-477604-s3 |
| **Project Name** | My First Project |

## Useful Commands

### Check instance status
```bash
gcloud sql instances describe mypostgresql \
  --project=ornate-producer-477604
```

### Connect via gcloud
```bash
gcloud sql connect mypostgresql \
  --user=Mypostgresql \
  --project=ornate-producer-477604
```

### View instance IP addresses
```bash
gcloud sql instances describe mypostgresql \
  --project=ornate-producer-477604 \
  --format="value(ipAddresses[0])"
```

### View instance users
```bash
gcloud sql users list --instance=mypostgresql \
  --project=ornate-producer-477604-s3
```

### Reset user password
```bash
gcloud sql users set-password Mypostgresql \
  --instance=mypostgresql \
  --password=<NEW_PASSWORD> \
  --project=ornate-producer-477604-s3
```

## Security Notes

1. **SSL/TLS Required**: All connections require SSL/TLS
2. **IP Whitelisting**: Configure authorized networks in Cloud SQL
3. **Cloud SQL Proxy**: Preferred for Kubernetes deployments
4. **Secrets Management**: Use Kubernetes secrets for credentials
5. **Audit Logging**: Enable Cloud SQL audit logging

## Network Access Configuration

### Allow GKE Cluster Access
```bash
gcloud sql instances patch mypostgresql \
  --allowed-networks=<GKE_CLUSTER_NETWORK_IP> \
  --project=ornate-producer-477604-s3
```

### Allow Your Local Machine
```bash
YOUR_IP=$(curl -s https://api.ipify.org)

gcloud sql instances patch mypostgresql \
  --allowed-networks=$YOUR_IP \
  --project=ornate-producer-477604
```

## Backup and Recovery

### View backups
```bash
gcloud sql backups list --instance=mypostgresql \
  --project=ornate-producer-477604-s3
```

### Create backup
```bash
gcloud sql backups create --instance=mypostgresql \
  --project=ornate-producer-477604-s3
```

### Restore from backup
```bash
gcloud sql backups restore <BACKUP_ID> \
  --backup-instance=mypostgresql \
  --project=ornate-producer-477604-s3
```

## Resource Limits

| Resource | Limit |
|----------|-------|
| **Maximum Connections** | Based on instance class |
| **Storage** | As configured (1GB - 65.536GB) |
| **Backup Retention** | 7 days (default) |
| **Backup Frequency** | Daily (automatic) |

## Additional Documentation

- [Cloud SQL Documentation](https://cloud.google.com/sql/docs)
- [Cloud SQL Proxy Documentation](https://cloud.google.com/sql/docs/postgres/cloud-sql-proxy)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)

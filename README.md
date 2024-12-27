# N8N Kubernetes Deployment and Upgrade

A comprehensive guide for deploying and upgrading N8N in Kubernetes environments.

## Project Structure
```
n8n/
├── README.md                 # This guide
├── n8n-deployment.yaml      # Kubernetes deployment configuration
├── cloudbuild.yaml          # Cloud Build pipeline configuration
├── Dockerfile               # N8N container image configuration
└── scripts/
    └── upgrade-onprem.sh    # On-premise upgrade script
```

## Deployment
Deploy N8N to your Kubernetes cluster:

```bash
kubectl apply -f n8n-deployment.yaml
```

The deployment includes:
- N8N application
- PostgreSQL database
- Persistent volumes
- Service, ingress, secret, configmap

## Upgrade Methods

### 1. N8N Upgrade in GKE (Using Cloud Build)
For N8N deployments in Google Kubernetes Engine environments, you have two options:

#### Option A: Manual Upgrade (Local)
First, clone this repository:
```bash
git clone https://github.com/[YOUR_ORGANIZATION]/n8n.git
cd n8n
```

Then run the upgrade from your terminal:
```bash
gcloud builds submit --config=cloudbuild.yaml \
  --substitutions=_TARGET_N8N_VERSION=1.69.1
```

Configuration variables:
| Variable | Description | Example |
|----------|-------------|---------|
| `_CLUSTER_LOCATION` | GKE cluster location (must specify region or zone based on cluster type) | Region: `europe-west1`<br>Zone: `europe-west1-b` |
| `_CLUSTER_NAME` | Name of the GKE cluster | `n8n-cluster` |
| `_DEPLOYMENT_NAME` | Name of the N8N deployment | `n8n` |
| `_NAMESPACE` | Kubernetes namespace | `n8n` |
| `_PG_SECRET_NAME` | PostgreSQL secret name | `postgres-secret` |
| `_BACKUP_BUCKET` | GCS bucket for backups | `backup-gcs` |
| `_REPO_LOCATION` | Artifact Registry location | `europe-west1` |
| `_TARGET_N8N_VERSION` | Specific N8N version (optional) | `1.69.1` |
| `_AUTO_UPGRADE` | Set to `true` for automatic upgrades; leave empty for manual upgrades | `true` |

#### Option B: Automated Upgrades
For automated upgrades, follow these steps in Google Cloud Console:

1. **Repository Setup**
   - Fork this repository to your GitHub organization
   - Go to [Cloud Build Triggers](https://console.cloud.google.com/cloud-build/triggers)
   - Click "Connect Repository"
   - Select your forked repository
   - Follow the authentication steps

2. **Service Account Setup**
   - Go to [IAM & Admin > Service Accounts](https://console.cloud.google.com/iam-admin/serviceaccounts)
   - Click "Create Service Account"
   - Name: `n8n-automation`
   - Description: "Service account for N8N automated upgrades"
   - Click "Create and Continue"
   - Add the following roles:
     - Cloud Build Editor (`roles/cloudbuild.builds.editor`)
     - Cloud Scheduler Admin (`roles/cloudscheduler.admin`)
     - GKE Backup Admin (`roles/gkebackup.admin`)
     - GKE Developer (`roles/container.developer`)
     - Artifact Registry Writer (`roles/artifactregistry.writer`)
     - Logs Writer (`roles/logging.logWriter`)
     - Logs Viewer (`roles/logging.viewer`)
     - Storage Object Creator (`roles/storage.objectCreator`)
   - Click "Done"

3. **Cloud Build Trigger Setup**
   - Go to [Cloud Build Triggers](https://console.cloud.google.com/cloud-build/triggers)
   - Click "Create Trigger"
   - Name: `n8n-auto-upgrade`
   - Event: "Manual invocation"
   - Source:
     - Repository: Select your connected repository
     - Branch: `^master$`
   - Configuration:
     - Type: "Cloud Build configuration file (yaml or json)"
     - Location: "Repository"
     - Cloud Build configuration file location: `/cloudbuild.yaml`
   - Variables (click "Add variable" for each):
     - Required variables:
       - `_CLUSTER_LOCATION`: Your GKE cluster location (e.g., `europe-west1-b`)
       - `_CLUSTER_NAME`: Your GKE cluster name
       - `_DEPLOYMENT_NAME`: Your N8N deployment name
       - `_NAMESPACE`: Kubernetes namespace where N8N is deployed
       - `_PG_SECRET_NAME`: PostgreSQL secret name
       - `_BACKUP_BUCKET`: GCS bucket for backups
       - `_REPO_LOCATION`: Artifact Registry location (e.g., `europe-west1`)
     - Auto upgrade setting:
       - `_AUTO_UPGRADE`: Set to `true`
   - Service account: Select `n8n-automation@[PROJECT_ID].iam.gserviceaccount.com`
   - Click "Create"

4. **Cloud Scheduler Setup**
   - Go to [Cloud Scheduler](https://console.cloud.google.com/cloudscheduler)
   - Click "Create Job"
   - Name: `n8n-version-check`
   - Region: Select your region (e.g., `europe-west1`)
   - Frequency: `0 0 1 * *` (runs monthly, see note below)
   - Timezone: Select your timezone
   - Target:
     - Target type: "HTTP"
     - URL: `https://cloudbuild.googleapis.com/v1/projects/[PROJECT_ID]/triggers/[TRIGGER_ID]:run`
       (Get TRIGGER_ID from the trigger's details page)
     - HTTP method: "POST"
     - Body: `{"branchName":"main"}`
     - Auth header: "Add OAuth token"
     - Service account: Select `n8n-automation@[PROJECT_ID].iam.gserviceaccount.com`
   - Click "Create"

> Note: You can adjust the schedule using cron expressions:
> - Monthly: `0 0 1 * *` (First day of each month at 00:00)
> - Daily: `0 0 * * *` (Every day at 00:00)

### Version Upgrade Behavior

The upgrade process follows these priorities:
1. If `_TARGET_N8N_VERSION` is specified, that exact version will be used regardless of `_AUTO_UPGRADE` setting
2. If no target version is specified but `_AUTO_UPGRADE=true`, the latest stable version will be used
3. If neither is specified, no upgrade will be performed

Example:
```bash
# Upgrade to specific version (highest priority)
gcloud builds submit --config=cloudbuild.yaml --substitutions=_TARGET_N8N_VERSION=1.69.1

# Auto upgrade to latest stable version (if no target version specified)
gcloud builds submit --config=cloudbuild.yaml --substitutions=_AUTO_UPGRADE=true

# No upgrade will be performed
gcloud builds submit --config=cloudbuild.yaml
```

### 2. On-Premise Upgrade

> **Note:** The upgrade script is theoretically implemented and not tested yet. Use with caution in production environments.

For self-managed Kubernetes clusters:

```bash
cd /path/to/n8n/scripts

# Show available options
./upgrade-onprem.sh -h

# Upgrade to latest version
./upgrade-onprem.sh

# Upgrade to specific version
./upgrade-onprem.sh -v 1.69.1

# Upgrade in different namespace
./upgrade-onprem.sh -n custom-namespace -v 1.69.1
```

Available flags:
- `-n, --namespace`: Kubernetes namespace (default: n8n)
- `-v, --version`: Target N8N version
- `-d, --deployment`: N8N deployment name (default: n8n)
- `-b, --backup-dir`: Backup directory (default: ./n8n-backups)

## Upgrade Features
- Automatic version detection and compatibility check
- Database and persistent volume backups before upgrade
- Safe deployment with rollback capability
- Node.js version compatibility verification
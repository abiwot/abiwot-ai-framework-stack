# OpenWebUI with Ollama on Kubernetes

This repository contains Kubernetes manifests and Helm chart values for deploying OpenWebUI with Ollama on Kubernetes v1.32, including automated PostgreSQL backup configuration.

## Architecture Overview

The deployment consists of the following components:

1. **OpenWebUI** - Web interface for interacting with LLMs
2. **Ollama** - Local LLM server with GPU support
3. **PostgreSQL** - Database for OpenWebUI
4. **Backup System** - Automated backup of PostgreSQL data to S3 or local storage

## Deployment Instructions

### Prerequisites

- Kubernetes v1.32 cluster with GPU support
- Helm v3.x
- Storage classes: `local-path-retain`
- NVIDIA GPU with drivers installed on worker nodes

### Deployment Steps

1. **Deploy OpenWebUI with Ollama**

   ```bash
   # Add Helm repo
   helm repo add open-webui https://helm.openwebui.com/
   helm repo update

   # Deploy using Helm
   helm upgrade --install open-webui open-webui/open-webui /
   -n myai01 --create-namespace -f /
   ./namespace/myai01/helm/openwebui/helm-openwebui-override-values-tls
   ```

2. **Verify Deployment**

   ```bash
   # Wait for OpenWebUI pods to deploy
   kubectl wait -n myai01 --for=condition=Available deployment /
   --selector=app.kubernetes.io/instance=open-webui --timeout=300s
   
   # Check all pods are running
   kubectl get pods -n myai01

   # Check OpenWebUI is accessible
   kubectl get ingress -n myai01
   ```

## Backup System

The backup system is designed to automatically back up the PostgreSQL database to an S3-compatible storage when available, with fallback to local storage when S3 is not reachable.

### Backup Components

- **Scheduled Backups**: CronJob runs every 4 hours
- **Manual Backups**: On-demand backup job
- **Restore Utility**: Deployment for restoring from backups
- **Persistent Storage**: PVC for storing local backups
- **Container Image**: Uses bitnami/postgresql with MinIO client installed

### Backup Configuration

The backup system uses the following configuration files:

- `10-pvc-postgres-backup.yaml`: Persistent Volume Claim for backup storage
- `20-secret-postgres-backup-secrets.yaml`: Contains sensitive credentials
- `30-configmap-postgres-backup-scripts.yaml`: Contains the backup script
- `30-configmap-postgres-backup-scripts-variables.yaml`: Contains environment variables
- `40-cronjob-postgres-backup.yaml`: Scheduled backup job
- `./adhoc/job-postgres-backup-manual.yaml`: Manual backup job
- `./restore/deployment-postgres-restore.yaml`: Restore utility

### Deployment Steps

**Note**: Ensure the values in `20-secret-postgres-backup-secrets.yaml` are correctly set with your S3 credentials and PostgreSQL database connection details.

1. **Logical Order of Operations**

   - Create a Persistent Volume Claim (PVC) for backup storage
   - Apply the backup configuration files
   - Verify the backup CronJob is running

2. **Deploy Postgres Backup Cron**

   Before running any backup jobs, you need to create the PVC:

   ```bash
   kubectl apply -f ./namespace/myai01/manifests/openwebui/postgresql/
   ```

### Technical Implementation

1. **Container Image**

   The backup and restore jobs use the `bitnami/postgresql:latest` image, which provides PostgreSQL client tools. The MinIO client is installed at runtime to enable S3 operations:

   ```bash
   # Install MinIO client
   curl -O https://dl.min.io/client/mc/release/linux-amd64/mc
   chmod +x mc
   mv mc /usr/local/bin/
   ```

   This approach ensures both PostgreSQL tools (`pg_dump`, `psql`) and MinIO client (`mc`) are available in the same container.

2. **Script Enhancements**

   - All scripts use `$(which mc)` to ensure the correct path to the MinIO client is used
   - Detailed logging with timestamps and log levels
   - Proper error handling and fallback mechanisms
   - Structured with functions for better maintainability

### Backup Process

1. **Automatic Backups**

   The system automatically backs up the PostgreSQL database every 4 hours.
   
   - First attempts to connect to S3 storage
   - If S3 is available, uploads backup and cleans up old backups (older than 7 days)
   - If S3 is not available, stores backup locally

2. **Manual Backups**

   To trigger a manual backup:

   ```bash
   kubectl apply ./namespace/myai01/manifest/openwebui/postgres/backukp adhoc/job-postgres-backup-manual.yaml
   ```

3. **Monitoring Backups**

   To check backup job status:

   ```bash
   # Check scheduled backup jobs
   kubectl get cronjob postgres-backup-cronjob -n myai01
   kubectl get jobs -n myai01 | grep postgres-backup

   # Check backup logs
   kubectl logs job/postgres-backup-manual -n myai01
   ```

### Restore Process

1. **Deploy Restore Utility**

   The restore utility is deployed as a separate pod that can be accessed to restore backups from S3 or local storage.

   ```bash
   kubectl apply -f ./namespace/myai01/manifest/openwebui/postgres/restore/deployment-postgres-restore.yaml
   ```

2. **Access the Restore Utility**

   ```bash
   # Get the pod name
   kubectl get pods -n myai01 | grep open-webui-restore

   # Access the pod
   kubectl exec -it <restore-pod-name> -n myai01 -- /bin/bash
   ```

3. **List Available Backups**

   ```bash
   # Inside the pod
   /scripts/list-backups.sh
   ```

4. **Download Backup from S3 (if needed)**

   ```bash
   # Inside the pod
   /scripts/download-from-s3.sh <backup-filename>
   ```

5. **Restore from Backup**

   ```bash
   # Inside the pod
   /scripts/restore.sh <backup-filename>
   ```

## Troubleshooting

### Common Issues

1. **Backup Job Fails**

   Check the logs for the failed job:

   ```bash
   kubectl logs job/<job-name> -n myai01
   ```

   Common issues:
   - S3 connectivity problems
   - PostgreSQL connection issues
   - Insufficient storage

2. **Restore Fails**

   Check the logs for the restore pod:

   ```bash
   kubectl logs <restore-pod-name> -n myai01
   ```

   Common issues:
   - Corrupted backup file
   - PostgreSQL connection issues
   - Insufficient permissions

### Logs

All components have been configured with verbose logging to aid in troubleshooting:

- **Backup Jobs**: Use `set -x` for verbose bash logging
- **PostgreSQL**: Configured with detailed logging
- **OpenWebUI**: Set to debug log level
- **Ollama**: Debug mode enabled

## Security Considerations

- Database credentials are stored in Kubernetes secrets
- S3 credentials are stored in Kubernetes secrets
- TLS is enabled for the ingress
- Network policies restrict cross-namespace communication

## Maintenance

### Updating Models

To update the models pulled by Ollama, modify the `models.pull` section in the Helm values file:

```yaml
models:
  pull:
    - qwen3:14b
    - llama3.1:8b
    - gemma3:12b
    - deepseek-r1:14b
```

### Scaling Resources

Resource requests and limits can be adjusted in the Helm values file based on workload requirements.

## API Access

If you need to access your OpenWebUI API, you need to configure the following components:

- **DNS**: Ensure your DNS can resolve the URLs within the Helm values file.
  - **ollamaUrls**: Set to point to your external LB
  - **openwebuiUrls**: Set to point to your external LB
- **External LB**: Ensure your external LB has entries to accept traffic for the OpenWebUI and Ollama URLs.
- **Ingress**: Ensure your ingress controller is configured to route traffic to the OpenWebUI and Ollama services.

### RooCode Connection - Example

To connect RooCode to your OpenWebUI API, you can use the follow these basic steps:

1. **Install RooCode**: Follow the RooCode installation instructions.
2. **Configure RooCode**:
   - Open the RooCode settings
   - `API Provider`: "Ollama"
   - `Base URL`: Set to the OpenWebUI API URL
   - `Model ID`: Set to the model you want to use (e.g., `qwen3:14b`)
   - Click "Save"
3. **Test Connection**: Use the RooCode interface to test the connection to the OpenWebUI API.

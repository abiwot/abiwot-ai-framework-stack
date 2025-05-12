# Manual steps to configure AI framework stack

## Framework Components

- nVidia GPU Operator
- OpenWebUI
  - Ollama installed/controlled via OpenWebUI

## Install nVidia GPU Operator via Helm

Notes:

- nVidia drivers have already been installed on the host system

Add the Helm repository

```shell
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update
```

Install the nVidia operator

```shell
helm upgrade -i nvidia-gpu nvidia/gpu-operator -n nvidia-gpu-operator --create-namespace --set driver.enabled=false
```

Verify pod deployment

```shell
kubectl wait -n nvidia-gpu-operator --for=condition=ready pod --selector=app.kubernetes.io/managed-by=gpu-operator --timeout=120s
kubectl wait -n nvidia-gpu-operator --for=condition=ready pod --selector=app.kubernetes.io/component=gpu-operator --timeout=120s
kubectl wait -n nvidia-gpu-operator --for=condition=ready pod --selector=app.kubernetes.io/instance=nvidia-gpu --timeout=120s
```

Verify GPU noticed

```shell
kubectl get nodes -o=custom-columns=NAME:.metadata.name,GPU:.status.allocatable."nvidia\.com/gpu"

kubectl -n nvidia-gpu-operator exec -it $(kubectl -n nvidia-gpu-operator get pods -l app=nvidia-device-plugin-daemonset -o=name | head -n 1) -- nvidia-smi
```

## Install OpenWebUI and Ollama

Add openwebui Helm repo

```shell
helm repo add open-webui https://helm.openwebui.com/
helm repo update
```

Install OpenWebUI and Ollama via helm - For gpu-enabled pod

```shell
helm upgrade --install open-webui open-webui/open-webui -n myai01 --create-namespace -f ./namespace/myai01/helm/openwebui/helm-openwebui-override-values-tls.yaml

kubectl wait -n myai01 --for=condition=Available deployment --selector=app.kubernetes.io/instance=open-webui --timeout=300s
```

Verify access to OpenWebUI

```shell
curl -s -o /dev/null -w "%{http_code}" <host address in ./ai-framework-stack/namespace/myai01/helm/openwebui/helm-openwebui-override-values.yaml - ingress.host>
```

## Configure OpenWebUI Postgres Backups to S3

We will configure regular backups of the OpenWebUI postgres to an S3 external target.  If the local system cannot currently reach the S3 (due to portability), then the backup falls-back to the local system storage.

### Prerequisites

- S3 target location
  - bucket: openwebui-backups
  - access key
  - secret key
- OpenWebUI_postgres details
  - postgres hostname
  - openwebui postgres database name
  - openwebui postgres username
  - openwebui postgres password

### Deploy Postgres Backup

```shell
kubectl apply -f ./namespace/myai01/manifests/openwebui/postgres/backup/
```


<!-- ## FUTURE REFERENCE ONLY -->
<!-- ## Install n8n Environment

### Pre-deploy - qdrant vector DB

```shell
helm repo add qdrant https://qdrant.github.io/qdrant-helm
helm repo update
```

Install qdrant

```shell
helm upgrade --install qdrant-n8n qdrant/qdrant -n myai01 -f ./namespace/myai01/helm/qdrant/helm-qdrant-override-values.yaml

kubectl wait -n myai01 --for=condition=Ready pod --selector=app.kubernetes.io/instance=qdrant-n8n --timeout=120s
```

### Install n8n

Add n8n helm repo

```shell
helm repo add community-charts https://community-charts.github.io/helm-charts
helm repo update
```

Install n8n

```shell
helm upgrade --install n8n community-charts/n8n -n myai01 -f ./namespace/myai01/helm/n8n/helm-n8n-override-values-tls.yaml

kubectl wait -n myai01 --for=condition=Ready pod --selector=app.kubernetes.io/instance=n8n --timeout=300s
```

## Configure n8n Postgres Backups to S3

We will configure regular backups of the n8n postgres to an S3 external target.  If the local system cannot currently reach the S3 (due to portability), then the backup falls-back to the local system storage.

### Prerequisites

- S3 target location
  - bucket: n8n-backups
  - access key
  - secret key
- n8n_postgres details
  - postgres hostname
  - n8n postgres database name
  - n8n postgres username
  - n8n postgres password

### Deploy Postgres Backup

```shell
kubectl apply -f ./namespace/myai01/manifests/n8n/postgres/backup/
``` -->
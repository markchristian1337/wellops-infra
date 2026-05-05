# WellOps Console — Infrastructure

Kubernetes manifests and Azure resource notes for the WellOps Console deployment. Pairs with [wellops-backend](https://github.com/markchristian1337/wellops-backend) and [wellops-frontend](https://github.com/markchristian1337/wellops-frontend).

> **Note:** The cluster and managed services are currently scaled down to avoid charges. The manifests and resource definitions in this repo are the full source of truth — `kubectl apply -f` against a fresh AKS cluster recreates the deployment. Cost when running is roughly $200-300/month (1-node AKS + Postgres Flexible Server + ACR + Key Vault + LoadBalancer IP).

## Structure

```
wellops-infra/
  backend/
    deployment.yaml          — FastAPI Deployment, single replica, references the ACR image
    service.yaml             — LoadBalancer Service exposing the FastAPI pod on port 80
    secretproviderclass.yaml — Tells the CSI driver which Key Vault secrets to mount
  rabbitmq/
    deployment.yaml          — RabbitMQ Deployment, image pulled from wellopsacr
    service.yaml             — ClusterIP Service exposing RabbitMQ inside the cluster only
    secretproviderclass.yaml — Tells the CSI driver which Key Vault secrets to mount
  workers/
    temperature/
     deployment.yaml          — Temperature worker Deployment, consumes from RabbitMQ, writes to Postgres
     secretproviderclass.yaml — Tells the CSI driver which Key Vault secrets to mount
  simulator/
    deployment.yaml          — Simulator Deployment, publishes fake sensor readings to RabbitMQ
    secretproviderclass.yaml — Tells the CSI driver which Key Vault secrets to mount
```

## Architecture

```
Internet
   |
   v
Azure LoadBalancer (public IP)
   |
   v
AKS cluster (wellops-aks, 1 node, standard_dc2ads_v5)
   |
   +-- FastAPI pod (image from wellopsacr)
   |         |
   |         v
   |   Azure Database for PostgreSQL Flexible Server
   |   (wellops-postgres, Central US, sslmode=require)
   |
   +-- RabbitMQ pod (image from wellopsacr)
   |         |
   |         +-- receives messages from simulator
   |         +-- delivers messages to worker pod
   |
   +-- Temperature worker pod (image from wellopsacr)
   |         |
   |         +-- consumes from RabbitMQ
   |         +-- calculates rolling average
   |         +-- writes to Postgres
   |
   +-- Simulator pod (image from wellopsacr)
             |
             +-- publishes fake sensor readings to RabbitMQ
   +-- CSI driver mounts secrets from Azure Key Vault
       (wellopskeyvault, RBAC mode)
       using kubelet managed identity
```

## Azure resources

- **Resource Group:** `wellops-rg` (East US)
- **AKS:** `wellops-aks`, 1 node, `standard_dc2ads_v5`
- **ACR:** `wellopsacr` — backend `:v3`, rabbitmq `:3.13.1-management`, worker-temperature `:v1`, simulator `:v1`
- **Postgres:** `wellops-postgres` (Flexible Server, Central US, `wellops` DB)
- **Key Vault:** `wellopskeyvault` (RBAC mode), holds DB and RabbitMQ credentials
- **Identity:** Kubelet managed identity granted "Key Vault Secrets User" role on the vault

## Deploy

Assumes you have the AKS cluster created, ACR attached, and Key Vault populated with the expected secrets.

```bash
# point kubectl at the cluster
az aks get-credentials --resource-group wellops-rg --name wellops-aks

# apply backend manifests
kubectl apply -f backend/secretproviderclass.yaml
kubectl apply -f backend/deployment.yaml
kubectl apply -f backend/service.yaml

# apply rabbitmq manifests
kubectl apply -f rabbitmq/secretproviderclass.yaml
kubectl apply -f rabbitmq/deployment.yaml
kubectl apply -f rabbitmq/service.yaml

# apply worker manifests
kubectl apply -f workers/temperature/secretproviderclass.yaml
kubectl apply -f workers/temperature/deployment.yaml

# apply simulator manifests
kubectl apply -f simulator/secretproviderclass.yaml
kubectl apply -f simulator/deployment.yaml
```

## Secret rotation

The K8s Secret is a cached copy of what the CSI driver pulls from Key Vault. To force a refresh:

```bash
# backend
kubectl delete secret wellops-secret
kubectl rollout restart deployment wellops-backend

# rabbitmq
kubectl delete secret rabbitmq-secret
kubectl rollout restart deployment rabbitmq

# worker
kubectl delete secret temperature-worker-secret
kubectl rollout restart deployment temperature-worker

# simulator
kubectl delete secret simulator-secret
kubectl rollout restart deployment simulator
```

The CSI driver re-mounts the secret on pod startup, the cached Secret is repopulated.

## Production deviations from this setup

- **1-node AKS cluster.** Production minimum is 2+ nodes across availability zones for HA. The 1-node config is a cost compromise for a personal project.
- **No Ingress controller.** Currently uses a LoadBalancer Service directly. Production would put nginx Ingress in front for multi-service routing once a worker pod gets its own Service.
- **Manual deployment.** Currently `kubectl apply` from my machine. GitHub Actions pipeline (build + push + deploy on merge to main) is the planned next step.
- **No autoscaling.** No HPA configured because the workload doesn't justify it yet.
- **RabbitMQ not clustered.** Single RabbitMQ pod with no mirroring. Production would run a 3-node RabbitMQ cluster for HA.

## Related repos

- [wellops-backend](https://github.com/markchristian1337/wellops-backend) — FastAPI service
- [wellops-frontend](https://github.com/markchristian1337/wellops-frontend) — React + TypeScript SPA

## License

MIT

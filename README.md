# WellOps Console — Infrastructure

Kubernetes manifests and Azure resource notes for the WellOps Console deployment. Pairs with [wellops-backend](https://github.com/markchristian1337/wellops-backend) and [wellops-frontend](https://github.com/markchristian1337/wellops-frontend).

> **Note:** The cluster and managed services are currently scaled down to avoid charges. The manifests and resource definitions in this repo are the full source of truth — `kubectl apply -f` against a fresh AKS cluster recreates the deployment. Cost when running is roughly $200-300/month (1-node AKS + Postgres Flexible Server + ACR + Key Vault + LoadBalancer IP).

## Manifests

- `deployment.yaml` — FastAPI Deployment, single replica, references the ACR image
- `service.yaml` — LoadBalancer Service exposing the FastAPI pod on port 80
- `secret.yaml` — Cached K8s Secret populated by the CSI driver from Key Vault
- `secretproviderclass.yaml` — Tells the CSI driver which Key Vault secrets to mount

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
   +-- CSI driver mounts secrets from Azure Key Vault
       (wellopskeyvault, RBAC mode)
       using kubelet managed identity
```

## Azure resources

- **Resource Group:** `wellops-rg` (East US)
- **AKS:** `wellops-aks`, 1 node, `standard_dc2ads_v5`
- **ACR:** `wellopsacr` — current image tag `:v3`
- **Postgres:** `wellops-postgres` (Flexible Server, Central US, `wellops` DB)
- **Key Vault:** `wellopskeyvault` (RBAC mode), holds DB credentials and config
- **Identity:** Kubelet managed identity granted "Key Vault Secrets User" role on the vault

## Deploy

Assumes you have the AKS cluster created, ACR attached, and Key Vault populated with the five expected secrets.

```bash
# point kubectl at the cluster
az aks get-credentials --resource-group wellops-rg --name wellops-aks

# apply manifests in order
kubectl apply -f secretproviderclass.yaml
kubectl apply -f secret.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

# get the public IP once the LoadBalancer provisions
kubectl get service
```

## Secret rotation

The K8s Secret is a cached copy of what the CSI driver pulls from Key Vault. To force a refresh:

```bash
kubectl delete secret wellops-secret
kubectl rollout restart deployment wellops-fastapi
```

The CSI driver re-mounts the secret on pod startup, the cached Secret is repopulated.

## Production deviations from this setup

Honest list of where this deployment diverges from what I'd build for production:

- **1-node AKS cluster.** Production minimum is 2+ nodes across availability zones for HA. The 1-node config is a cost compromise for a personal project.
- **No Ingress controller.** Currently uses a LoadBalancer Service directly. Production would put nginx Ingress in front for multi-service routing once a worker pod gets its own Service.
- **Manual deployment.** Currently `kubectl apply` from my machine. GitHub Actions pipeline (build + push + deploy on merge to main) is the planned next step.
- **No autoscaling.** No HPA configured because the workload doesn't justify it yet.

## Related repos

- [wellops-backend](https://github.com/markchristian1337/wellops-backend) — FastAPI service
- [wellops-frontend](https://github.com/markchristian1337/wellops-frontend) — React + TypeScript SPA

## License

MIT

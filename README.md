<p align="center">
  <img alt="Kubernetes" src="https://img.shields.io/badge/Kubernetes-1.34-326CE5?logo=kubernetes&logoColor=white">
  <img alt="Azure AKS" src="https://img.shields.io/badge/Azure-AKS-0078D4?logo=microsoftazure&logoColor=white">
  <img alt="Istio" src="https://img.shields.io/badge/Istio-1.29-466BB0?logo=istio&logoColor=white">
  <img alt="Helm" src="https://img.shields.io/badge/Helm-3.x-0F1689?logo=helm&logoColor=white">
  <img alt="License" src="https://img.shields.io/badge/License-GPLv3-blue.svg">
</p>

# soqoni-k8s

Kubernetes manifests for the Soqoni microservices platform. Supports two environments via branches — local Minikube development and production AKS on Azure.

<!-- TODO: full system architecture diagram showing:
     Azure Load Balancer → ingress-nginx → pods (frontend, catalogue, carts, auth)
     → managed databases (MySQL Flexible Server, Cosmos DB)
     with Key Vault CSI arrow showing secret injection into each pod
     Suggested file: docs/soqoni-aks-architecture.png
     (Excalidraw source already at docs/soqoni-aks-architecture.excalidraw) -->
<p align="center">
  <img src="docs/soqoni-aks-architecture.png" alt="Soqoni AKS Architecture" width="900">
</p>

## Branches

| Branch | Target | Database | Secrets |
|--------|--------|----------|---------|
| `main` | Minikube (local) | MySQL + MongoDB pods with PVCs | `kubectl create secret` |
| `azure-managed-db` | AKS (`soqoni-aks`, eastasia) | Azure MySQL Flexible Server + Cosmos DB | Azure Key Vault CSI Driver |

The live deployment is on the `azure-managed-db` branch at `http://20.24.112.153`.

## Architecture

```
Internet
  └── Azure Load Balancer (ingress-nginx)
        └── soqoni-ingress (cookie affinity)
              ├── /catalogue  → catalogue:80   (Go)
              └── /           → frontend:8079  (Node.js)
                    ├── /catalogue → catalogue:80
                    ├── /cart      → carts:80       (Spring Boot)
                    └── /api/user  → auth:8082       (Go)

Databases (AKS branch):
  catalogue → Azure MySQL Flexible Server (soqoni-mysql.mysql.database.azure.com)
  carts     → Azure Cosmos DB serverless  (MongoDB API)
  auth      → Azure MySQL Flexible Server (shared socksdb)

Secrets (AKS branch):
  Azure Key Vault (soqoni-kv) → CSI Driver → mounted as K8s Secrets in each pod
```

## Manifests

| File | Description |
|------|-------------|
| `frontend.yaml` | Deployment + ClusterIP Service, `AUTH_URL` env var |
| `catalogue.yaml` | Deployment + ClusterIP Service, DSN from `catalogue-secret` |
| `carts.yaml` | Deployment + ClusterIP Service (port 80 → targetPort 8081) |
| `auth.yaml` | Deployment + ClusterIP Service (port 8082), CSI volume mount |
| `ingress.yaml` | nginx Ingress with cookie session affinity |
| `hpa.yaml` | HPA for frontend (min 2, max 5, CPU 70%) |
| `secret-provider-classes.yaml` | `SecretProviderClass` for all 4 services via Azure Key Vault |
| `mysql.yaml.minikube-only` | catalogue-db pod — Minikube only, excluded from AKS |
| `mongodb.yaml.minikube-only` | carts-db pod — Minikube only, excluded from AKS |

## Deploy to AKS

```bash
# Switch to AKS branch
git checkout azure-managed-db

# Get credentials
az aks get-credentials --resource-group soqoni-rg --name soqoni-aks --overwrite-existing

# Install Key Vault CSI secrets driver (if not enabled on cluster)
az aks enable-addons --addons azure-keyvault-secrets-provider \
  --resource-group soqoni-rg --name soqoni-aks

# Apply all manifests (secrets come from Key Vault — no kubectl create secret needed)
kubectl apply -f secret-provider-classes.yaml
kubectl apply -f catalogue.yaml
kubectl apply -f carts.yaml
kubectl apply -f auth.yaml
kubectl apply -f frontend.yaml
kubectl apply -f ingress.yaml
kubectl apply -f hpa.yaml

# Check rollout
kubectl get pods
kubectl get ingress
```

## Deploy to Minikube

```bash
git checkout main

minikube start --cpus=4 --memory=6144
minikube addons enable ingress

# Create secrets manually (minikube branch uses kubectl create secret)
kubectl create secret generic catalogue-secret \
  --from-literal=db-dsn='catalogue_user:default_password@tcp(catalogue-db:3306)/socksdb'
kubectl create secret generic auth-secret \
  --from-literal=db-user=catalogue_user \
  --from-literal=db-password=default_password \
  --from-literal=jwt-secret=$(openssl rand -hex 32) \
  --from-literal=admin-promote-secret=$(openssl rand -hex 16)

# Load custom catalogue-db image (mysql:5.7 + dump.sql baked in)
minikube image load abdvswmdr/soqonicatalogue-db:latest

kubectl apply -f mysql.yaml.minikube-only
kubectl apply -f mongodb.yaml.minikube-only
kubectl apply -f catalogue.yaml
kubectl apply -f carts.yaml
kubectl apply -f auth.yaml
kubectl apply -f frontend.yaml
kubectl apply -f ingress.yaml
kubectl apply -f hpa.yaml

open http://$(minikube ip)
```

## Key Vault CSI — How It Works

Secrets in the `azure-managed-db` branch are never stored in etcd. Instead:

1. `secret-provider-classes.yaml` defines a `SecretProviderClass` per service — each references named secrets in `soqoni-kv` Key Vault
2. Each pod spec mounts a CSI volume (`driver: secrets-store.csi.k8s.io`) pointing to its `SecretProviderClass`
3. On pod start, the CSI driver authenticates to Key Vault using the AKS Managed Identity and injects the secrets as both mounted files and K8s `Secret` objects (via `secretObjects`)
4. Environment variables in each container reference the auto-created K8s Secrets via `secretKeyRef`

The `userAssignedIdentityID` in each `SecretProviderClass` is the kubelet identity (`5737b2d8-f064-4ae7-8bbb-dfb7b99940da`) — granted `Key Vault Secrets User` role on `soqoni-kv`.

## Known Gotchas

- **Carts service port**: ClusterIP must expose port 80. The frontend calls `http://carts/carts` (port 80 by default). `targetPort` stays 8081.
- **Ingress `/cart` path**: Do NOT add `/cart` to the ingress. Browser calls `GET /cart` → must hit frontend Node.js (adds `customerId` to session before proxying to carts). Direct routing bypasses this.
- **HPA + sessions**: Frontend uses in-memory sessions. With HPA min=2, requests can hit different pods. Session affinity annotation (`nginx.ingress.kubernetes.io/affinity: cookie`) is set in `ingress.yaml` to fix this.
- **catalogue-db PVC**: MySQL init scripts only run on an empty data dir. Schema changes require deleting the PVC first.

## License

[GNU General Public License v3.0](LICENSE)

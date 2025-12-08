# Infrastructure

## Overview

RealmGrid runs on a hybrid infrastructure:

| Component | Provider | Purpose |
|-----------|----------|---------|
| Game Servers | Hetzner (K3s) | Low-cost, high-performance game hosting |
| Azure Functions | Azure | Serverless API backend |
| Cosmos DB | Azure | Global document database |
| Admin/Web | Azure Static Web Apps | Frontend hosting |

## Kubernetes Cluster (K3s on Hetzner)

### Why Hetzner + K3s?

- **Cost**: Hetzner dedicated servers are 50-70% cheaper than cloud VMs
- **Performance**: Bare metal = no hypervisor overhead
- **K3s**: Lightweight Kubernetes, perfect for edge/gaming workloads

### Cluster Setup

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        K3S CLUSTER (Hetzner)                                 │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  Control Plane   │     │  Control Plane   │     │  Control Plane   │
│  (CPX31 VM)      │     │  (CPX31 VM)      │     │  (CPX31 VM)      │
│  4 vCPU, 8GB     │     │  4 vCPU, 8GB     │     │  4 vCPU, 8GB     │
└────────┬─────────┘     └────────┬─────────┘     └────────┬─────────┘
         │                        │                        │
         └────────────────────────┼────────────────────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    │                           │
         ┌──────────┴──────────┐     ┌──────────┴──────────┐
         │   Worker Node 1     │     │   Worker Node 2     │
         │   (AX41 Dedicated)  │     │   (AX41 Dedicated)  │
         │   6 cores, 64GB RAM │     │   6 cores, 64GB RAM │
         │   2x 512GB NVMe     │     │   2x 512GB NVMe     │
         └─────────────────────┘     └─────────────────────┘
                    │                           │
         ┌──────────┴──────────────────────────┴──────────┐
         │                GAME SERVERS                     │
         │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐  │
         │  │ MC #1  │ │ MC #2  │ │Valheim │ │Palworld│  │
         │  └────────┘ └────────┘ └────────┘ └────────┘  │
         └─────────────────────────────────────────────────┘
```

### Node Capacity

Each AX41 worker can run approximately:

| Tier | Max per Node | Based on |
|------|--------------|----------|
| Small | 30 servers | 64GB / 2GB = 32 (with overhead) |
| Medium | 15 servers | 64GB / 4GB = 16 (with overhead) |
| Heavy | 7 servers | 64GB / 8GB = 8 (with overhead) |

### Namespace: game-servers

All game server pods run in the `game-servers` namespace:

```bash
kubectl get pods -n game-servers
NAME                           READY   STATUS    RESTARTS   AGE
gs-srv_a1b2c3d4-xxxxx         1/1     Running   0          2d
gs-srv_e5f6g7h8-xxxxx         1/1     Running   0          5h
gs-srv_i9j0k1l2-xxxxx         0/1     Pending   0          1m
```

## Azure Functions

### Function Apps

| Function | Trigger | Purpose |
|----------|---------|---------|
| `checkout_create` | HTTP | Create Mollie checkout |
| `webhook_mollie` | HTTP | Handle Mollie webhooks |
| `server_provision` | HTTP | Create K8s resources |
| `server_start` | HTTP | Scale to 1 replica |
| `server_stop` | HTTP | Scale to 0 replicas |
| `server_update` | HTTP | Update config/tier |
| `server_delete` | HTTP | Remove K8s resources |
| `server_list` | HTTP | List with K8s status |
| `server_status` | HTTP | Get single server status |
| `server_command` | HTTP | Execute RCON command |
| `server_backup` | HTTP | Create world backup |
| `server_restore` | HTTP | Restore from backup |
| `subscription_list` | HTTP | List subscriptions |
| `subscription_cancel` | HTTP | Cancel subscription |
| `subscription_expire` | Timer (hourly) | Stop servers when cancel period ends |
| `subscription_cleanup` | Timer (daily 3AM) | Delete data after 30-day retention |

### Environment Variables

```bash
# Azure
COSMOS_ENDPOINT=https://xxx.documents.azure.com:443/
COSMOS_DATABASE=realm-prod

# Kubernetes (base64 encoded kubeconfig)
K8S_KUBECONFIG=<base64 kubeconfig>

# Mollie
MOLLIE_API_KEY=live_xxx

# Environment
ENVIRONMENT=prod
```

## Cosmos DB

### Databases

| Database | Environment |
|----------|-------------|
| `realm-dev` | Development |
| `realm-test` | Testing |
| `realm-acc` | Acceptance |
| `realm-prod` | Production |

### Containers (Collections)

| Container | Partition Key | Purpose |
|-----------|--------------|---------|
| `users` | `/id` | User accounts |
| `subscriptions` | `/userId` | Subscription records |
| `servers` | `/userId` | Server configurations |
| `payments` | `/userId` | Payment history |
| `invoices` | `/userId` | Invoice records |

### Indexing Policy

```json
{
    "indexingMode": "consistent",
    "automatic": true,
    "includedPaths": [
        {"path": "/userId/?"},
        {"path": "/status/?"},
        {"path": "/gameType/?"},
        {"path": "/tier/?"},
        {"path": "/createdAt/?"}
    ],
    "excludedPaths": [
        {"path": "/settings/*"},
        {"path": "/rconPassword/?"}
    ]
}
```

## Networking

### External Access

```
Internet
    │
    ▼
┌─────────────────┐
│ Hetzner         │
│ Load Balancer   │
└────────┬────────┘
         │
    ┌────┴────┐
    │ K3s     │
    │ Ingress │
    └────┬────┘
         │
    ┌────┴─────────────────────────────────┐
    │           game-servers namespace      │
    │  ┌───────────────────────────────┐   │
    │  │ Service (LoadBalancer)        │   │
    │  │ server-srv_xxx                │   │
    │  │ External IP: 203.0.113.50     │   │
    │  └───────────────┬───────────────┘   │
    │                  │                    │
    │  ┌───────────────┴───────────────┐   │
    │  │ Pod                           │   │
    │  │ gs-srv_xxx-xxxxx              │   │
    │  │ Port 25565 (Minecraft)        │   │
    │  └───────────────────────────────┘   │
    └──────────────────────────────────────┘
```

### Firewall Rules

| Port | Protocol | Purpose |
|------|----------|---------|
| 22 | TCP | SSH (admin only) |
| 80 | TCP | HTTP redirect |
| 443 | TCP | HTTPS API |
| 6443 | TCP | K8s API (internal) |
| 25565 | TCP | Minecraft |
| 2456-2458 | UDP | Valheim |
| 8211 | UDP | Palworld |
| 28015-28016 | UDP | Rust |
| 7777 | TCP | Terraria |

## Storage

### PersistentVolumes

Each game server gets a PVC for world data:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-srv_a1b2c3d4
  namespace: game-servers
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 25Gi  # Tier-based
  storageClassName: local-path  # K3s default
```

### Backup Storage

World backups are stored in Azure Blob Storage:

```
realmgrid-backups/
├── srv_a1b2c3d4/
│   ├── 2024-12-01_backup.tar.gz
│   ├── 2024-12-08_backup.tar.gz
│   └── latest.tar.gz
├── srv_e5f6g7h8/
│   └── ...
```

## Monitoring

### Metrics

- **Prometheus**: Cluster metrics
- **Grafana**: Dashboards
- **Azure Monitor**: Function metrics

### Alerts

| Alert | Condition | Action |
|-------|-----------|--------|
| Node Down | Node not ready for 5m | Page on-call |
| Pod Crash Loop | >3 restarts in 10m | Notify admin |
| High Memory | >90% node memory | Scale alert |
| Payment Failed | 3+ failures | Email user + admin |

## Deployment

### CI/CD Pipeline

```
Push to main
     │
     ▼
GitHub Actions
├── Run tests
├── Build functions
├── Deploy to Azure Functions
└── Update K8s manifests
     │
     ▼
Azure Functions updated
K8s resources applied
```

### Terraform

Infrastructure is managed via Terraform in `realm-infra/terraform/`:

```hcl
# Main resources
resource "azurerm_cosmosdb_account" "main" { ... }
resource "azurerm_function_app" "api" { ... }
resource "azurerm_static_site" "admin" { ... }
```

# Architecture Overview

## System Components

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              FRONTEND                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│  realm-web (Customer)          │  realm-admin (Internal)                    │
│  - Pricing/Checkout            │  - Server Management                       │
│  - Dashboard                   │  - Subscription Admin                      │
│  - Server Control Panel        │  - Billing Overview                        │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           AZURE FUNCTIONS                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│  checkout_create    │  Mollie checkout session creation                     │
│  webhook_mollie     │  Payment webhooks & subscription events               │
│  server_provision   │  Create K8s deployment for new server                 │
│  server_start       │  Scale deployment to 1 replica                        │
│  server_stop        │  Scale deployment to 0 replicas                       │
│  server_update      │  Update tier/config, patch deployment                 │
│  server_delete      │  Remove K8s resources                                 │
│  server_list        │  List servers with K8s status                         │
│  subscription_list  │  List user subscriptions                              │
│  subscription_cancel│  Handle cancellation                                  │
└─────────────────────────────────────────────────────────────────────────────┘
                          │                    │
                          ▼                    ▼
┌────────────────────────────────┐  ┌─────────────────────────────────────────┐
│         COSMOS DB              │  │           KUBERNETES (K3s)               │
├────────────────────────────────┤  ├─────────────────────────────────────────┤
│  users                         │  │  Namespace: game-servers                 │
│  subscriptions                 │  │  ├── Deployments (gs-{serverId})        │
│  servers                       │  │  ├── Services (LoadBalancer)            │
│  payments                      │  │  └── PersistentVolumeClaims             │
│  invoices                      │  │                                          │
└────────────────────────────────┘  └─────────────────────────────────────────┘
                                                      │
                                                      ▼
                                    ┌─────────────────────────────────────────┐
                                    │           GAME CONTAINERS                │
                                    ├─────────────────────────────────────────┤
                                    │  minecraft (itzg/minecraft-server)      │
                                    │  valheim (lloesche/valheim-server)      │
                                    │  palworld (thijsvanloef/palworld)       │
                                    │  rust, enshrouded, terraria             │
                                    └─────────────────────────────────────────┘
```

## Data Flow

### New Subscription Flow

```
Customer selects tier + game
         │
         ▼
checkout_create → Mollie Checkout
         │
         ▼
Customer pays via iDEAL/Card/etc
         │
         ▼
Mollie sends webhook → webhook_mollie
         │
         ├── Create subscription doc (Cosmos)
         ├── Create server doc (Cosmos)
         └── Call server_provision
                    │
                    ▼
         Create K8s resources:
         ├── PersistentVolumeClaim
         ├── Deployment (replicas: 1)
         └── LoadBalancer Service
                    │
                    ▼
         Pod scheduled, container starts
                    │
                    ▼
         External IP assigned
                    │
                    ▼
         Server doc updated with IP/port
                    │
                    ▼
         Customer can connect!
```

## Key Identifiers

| ID | Format | Example | Purpose |
|----|--------|---------|---------||
| `serverId` | `srv_{uuid}` | `srv_a1b2c3d4` | Unique server identifier |
| `subscriptionId` | `sub_{paymentId}` | `sub_tr_xyz123` | Links to Mollie payment |
| `userId` | Auth provider ID | `aad|{guid}` | User identity (Azure AD) |

## Kubernetes Resources per Server

Each game server creates:

1. **PersistentVolumeClaim**: `pvc-{serverId}`
   - Stores world data, configs, backups
   - Size based on tier (10Gi / 25Gi / 50Gi)

2. **Deployment**: `gs-{serverId}`
   - Single replica (0 when stopped, 1 when running)
   - Resource limits based on tier
   - Recreate strategy (not rolling)

3. **Service**: `server-{serverId}`
   - Type: LoadBalancer
   - Exposes game port + query port
   - Gets external IP from cloud provider

## Labels on K8s Resources

```yaml
labels:
  app: game-server
  server-id: srv_a1b2c3d4
  user-id: aad|12345678-1234-1234-1234-123456789012
  game-type: minecraft
  tier: medium
  managed-by: realm-functions
```

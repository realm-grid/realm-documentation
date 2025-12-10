# Server Management

## Server Tiers

Users select a tier (not raw CPU/RAM). This simplifies the UX while we manage the technical details.

### Tier Specifications

| Tier | CPU Request | CPU Limit | Memory | Storage | Price |
|------|-------------|-----------|--------|---------|-------|
| **Small** | 500m | 1 core | 1-2 Gi | 10 Gi | €4.99/mo |
| **Medium** | 1 core | 2 cores | 2-4 Gi | 25 Gi | €9.99/mo |
| **Heavy** | 2 cores | 4 cores | 4-8 Gi | 50 Gi | €24.99/mo |

### Player Limits by Game

Different games use resources differently. A Minecraft server with mods uses more RAM per player than Terraria.

| Game | Small | Medium | Heavy |
|------|-------|--------|-------|
| **Minecraft** | 10 players | 25 players | 60 players |
| **Valheim** | 6 players | 12 players | 20 players |
| **Palworld** | 8 players | 16 players | 32 players |
| **Rust** | 20 players | 50 players | 150 players |
| **Enshrouded** | 4 players | 8 players | 16 players |
| **Terraria** | 16 players | 32 players | 64 players |

## Kubernetes Scaling

### How Servers Map to K8s

Each subscription creates ONE game server with these K8s resources:

```yaml
# PersistentVolumeClaim - Stores world data
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-srv_a1b2c3d4
  namespace: game-servers
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 25Gi  # Based on tier

---
# Deployment - The actual game container
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gs-srv_a1b2c3d4
  namespace: game-servers
  labels:
    server-id: srv_a1b2c3d4
    user-id: aad|12345678-1234-1234-1234-123456789012
    game-type: minecraft
    tier: medium
spec:
  replicas: 1  # 0 when stopped, 1 when running
  strategy:
    type: Recreate  # Important for game servers
  template:
    spec:
      containers:
      - name: game-server
        image: itzg/minecraft-server:latest
        resources:
          requests:
            cpu: "1000m"
            memory: "2Gi"
          limits:
            cpu: "2000m"
            memory: "4Gi"
        env:
        - name: MAX_PLAYERS
          value: "25"
        - name: MEMORY
          value: "3G"
        volumeMounts:
        - name: server-data
          mountPath: /data

---
# Service - External access
apiVersion: v1
kind: Service
metadata:
  name: server-srv_a1b2c3d4
  namespace: game-servers
spec:
  type: LoadBalancer
  ports:
  - port: 25565
    targetPort: 25565
    protocol: TCP
    name: game
  selector:
    server-id: srv_a1b2c3d4
```

### Scaling Operations

| Operation | K8s API Call | Effect |
|-----------|--------------|--------|
| **Start** | `patch_deployment_scale(replicas=1)` | Schedules pod, container starts |
| **Stop** | `patch_deployment_scale(replicas=0)` | Graceful shutdown, data saved |
| **Restart** | Delete pod | Deployment recreates pod |
| **Upgrade tier** | `patch_deployment` | New resources, pod restart |
| **Delete** | Delete Deployment, Service, PVC | All resources removed |

### Start Server

```python
# server_start/__init__.py
k8s["apps"].patch_namespaced_deployment_scale(
    name=f"gs-{server_id}",
    namespace="game-servers",
    body={"spec": {"replicas": 1}}
)
```

### Stop Server

```python
# server_stop/__init__.py
k8s["apps"].patch_namespaced_deployment_scale(
    name=f"gs-{server_id}",
    namespace="game-servers",
    body={"spec": {"replicas": 0}}
)
```

### Update Tier

```python
# server_update/__init__.py
tier_config = get_tier_config(new_tier, game_type)

patch = {
    "spec": {
        "template": {
            "spec": {
                "containers": [{
                    "name": "game-server",
                    "resources": {
                        "requests": {
                            "cpu": tier_config["cpu_request"],
                            "memory": tier_config["memory_request"],
                        },
                        "limits": {
                            "cpu": tier_config["cpu_limit"],
                            "memory": tier_config["memory_limit"],
                        }
                    }
                }]
            }
        }
    }
}

k8s["apps"].patch_namespaced_deployment(
    name=f"gs-{server_id}",
    namespace="game-servers",
    body=patch
)
```

## Server Lifecycle

```
                    ┌──────────────────────────────────────────────┐
                    │           SERVER LIFECYCLE                    │
                    └──────────────────────────────────────────────┘

Subscription Created                                    Subscription Canceled
        │                                                       │
        ▼                                                       ▼
┌──────────────┐      Start      ┌───────────────┐      ┌──────────────┐
│ PROVISIONING │ ──────────────► │    RUNNING    │ ───► │   STOPPED    │
└──────────────┘                 └───────────────┘      └──────────────┘
        │                              │    ▲                  │
        │                              │    │                  │
        │                           Stop│   │Start             │
        │                              │    │                  │
        │                              ▼    │                  │
        │                        ┌───────────────┐             │
        │                        │    STOPPED    │             │
        │                        └───────────────┘             │
        │                              │                       │
        │                              │ Delete                │
        │                              ▼                       │
        │                        ┌───────────────┐             │
        └───────────────────────►│   DELETED     │◄────────────┘
                                 └───────────────┘
                                        │
                                        ▼
                                 Data Purged (30 days)
```

## Server Document (Cosmos DB)

```json
{
    "id": "srv_a1b2c3d4",
    "userId": "aad|12345678-1234-1234-1234-123456789012",
    "subscriptionId": "sub_tr_xyz123",
    
    "name": "My Minecraft Server",
    "gameType": "minecraft",
    "tier": "medium",
    "maxPlayers": 25,
    "version": "1.20.4",
    
    "status": "running",
    "ip": "203.0.113.50",
    "port": 25565,
    "queryPort": 25565,
    
    "settings": {
        "difficulty": "normal",
        "gamemode": "survival",
        "pvp": true,
        "motd": "Welcome to my server!"
    },
    
    "rconPassword": "xxx",  // For admin access
    
    "createdAt": "2024-12-08T12:34:56Z",
    "updatedAt": "2024-12-08T12:34:56Z",
    "lastStartedAt": "2024-12-08T12:34:56Z",
    "lastStoppedAt": null
}
```

## Supported Games

| Game | Image | Port | Protocol |
|------|-------|------|----------|
| Minecraft | `itzg/minecraft-server:latest` | 25565 | TCP |
| Valheim | `lloesche/valheim-server:latest` | 2456 | UDP |
| Palworld | `thijsvanloef/palworld-server-docker:latest` | 8211 | UDP |
| Rust | `didstopia/rust-server:latest` | 28015 | UDP |
| Enshrouded | `mornedhels/enshrouded-server:latest` | 15636 | UDP |
| Terraria | `ryshe/terraria:latest` | 7777 | TCP |

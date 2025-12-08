# RealmGrid Infrastructure Alignment

## Overview

This document describes the complete infrastructure alignment between Azure resources, Kubernetes clusters, game servers, and customers.

## Infrastructure Summary

### Azure Resources

| Resource Type | Name | Location | Purpose |
|--------------|------|----------|---------|
| **Cosmos DB** | `realm-dta-app-cosmos` | West Europe | Main database account |
| **App Service Plan** | `xevolve-dta-asp` | North Europe | Shared hosting plan |
| **Container Registry** | `realmgridacr` | West Europe | Docker images |
| **DNS Zone** | `realmgrid.com` | Global | Domain management |

### Resource Groups

| Resource Group | Purpose | Resources |
|---------------|---------|-----------|
| `realm-dta-website-rg` | Development/Test/Accept | Web Apps, Functions, Cosmos DB |
| `realm-prod-website-rg` | Production | Production Web Apps |
| `xevolve-dta-rg` | Shared | App Service Plan |

## Kubernetes Infrastructure

### Hetzner K3s Cluster

**Server**: `157.180.57.169` (Hetzner EX44 Bare Metal)

| Component | Details |
|-----------|---------|
| **Kubernetes** | K3s (lightweight Kubernetes) |
| **Namespaces** | `gameservers`, `minecraft` |
| **CRDs** | `GameServer` (realm.grid/v1) |
| **Operator** | Game Server Operator |
| **Port Range** | 30000-32767 (NodePort services) |

## Cosmos DB Structure

### realm-dta-app-cosmos Account

#### Databases

| Database | Purpose | Environments |
|----------|---------|-------------|
| `realm-dev` | Web app dev | Development |
| `realm-test` | Web app test | Testing |
| `realm-acc` | Web app acc | Acceptance |
| `realm-prod` | Web app prod | Production |
| `prod-admin-db` | Admin prod | Production only |
| `realm-admin-dev` | Admin dev | Development |
| `realm-admin-test` | Admin test | Testing |
| `realm-admin-acc` | Admin acc | Acceptance |

#### Container Schema

**realm-admin-* databases** contain:
- `vms` - Virtual machine inventory
- `environments` - Environment configurations
- `k8s-clusters` - Kubernetes cluster info
- `stats` - Dashboard statistics
- `crm_contacts` - Customer contact records
- `crm_companies` - Company records
- `crm_deals` - Sales pipeline
- `servers` - Game server mappings

**realm-* databases** contain:
- `users` - User accounts
- `servers` - Game server instances
- `subscriptions` - Billing subscriptions
- `backups` - Server backups
- `invoices` - Billing invoices
- `payments` - Payment records

## Game Server Alignment

### Container-to-Customer Mapping

#### Patrick Rosen (patrickr@realmgrid.com)
- **Company**: RealmGrid
- **Role**: Founder & CEO
- **CRM Contact ID**: `contact-patrick-001`

| Server ID | Game Type | Port | Address | K8s Namespace |
|-----------|-----------|------|---------|---------------|
| `patrick-minecraft-001` | Minecraft | 30001 | 157.180.57.169:30001 | gameservers |
| `patrick-valheim-001` | Valheim | 30334 | 157.180.57.169:30334 | gameservers |
| `patrick-palworld-001` | Palworld | 30667 | 157.180.57.169:30667 | gameservers |

#### Yair Ben-David (yair@cloudevolvers.com)
- **Company**: Cloud Evolvers
- **Role**: CTO
- **CRM Contact ID**: `contact-yair-001`

| Server ID | Game Type | Port | Address | K8s Namespace |
|-----------|-----------|------|---------|---------------|
| `yair-minecraft-001` | Minecraft | 31001 | 157.180.57.169:31001 | gameservers |
| `yair-rust-001` | Rust | 31334 | 157.180.57.169:31334 | gameservers |
| `yair-ark-001` | ARK | 31667 | 157.180.57.169:31667 | gameservers |

### Port Allocation Strategy

| Game Type | Port Range | Increment |
|-----------|-----------|-----------|
| Minecraft | 30000-30999 | +333 |
| Valheim | 30000-30999 | +333 |
| Palworld | 30000-30999 | +333 |
| Rust | 31000-31999 | +333 |
| ARK | 31000-31999 | +333 |

## Application Deployments

### realm-admin (Admin Dashboard)

| Environment | URL | Database | Resource Group |
|------------|-----|----------|----------------|
| **dev** | https://admin-dev.realmgrid.com | realm-admin-dev | realm-dta-website-rg |
| **test** | https://admin-test.realmgrid.com | realm-admin-test | realm-dta-website-rg |
| **acc** | https://admin-acc.realmgrid.com | realm-admin-acc | realm-dta-website-rg |
| **prod** | https://admin.realmgrid.com | prod-admin-db | realm-prod-website-rg |

### realm-web (Customer Portal)

| Environment | Website | API | Database |
|------------|---------|-----|----------|
| **dev** | https://dev.realmgrid.com | https://api-dev.realmgrid.com | realm-dev |
| **test** | https://test.realmgrid.com | https://api-test.realmgrid.com | realm-test |
| **acc** | https://acc.realmgrid.com | https://api-acc.realmgrid.com | realm-acc |
| **prod** | https://realmgrid.com | https://api.realmgrid.com | realm-prod |

### realm-functions (Azure Functions)

| Environment | URL | Database |
|------------|-----|----------|
| **dev** | realm-dev-api-fa.azurewebsites.net | realm-dev |
| **test** | realm-test-api-fa.azurewebsites.net | realm-test |
| **acc** | realm-acc-api-fa.azurewebsites.net | realm-acc |
| **prod** | realm-prod-api-fa.azurewebsites.net | realm-prod |

## Network Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                         Azure Cloud                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────────┐      ┌──────────────────┐           │
│  │   realm-web      │      │   realm-admin    │           │
│  │  (Web Apps)      │      │   (Web Apps)     │           │
│  └────────┬─────────┘      └────────┬─────────┘           │
│           │                         │                      │
│           │                         │                      │
│  ┌────────▼─────────────────────────▼─────────┐           │
│  │       realm-dta-app-cosmos                 │           │
│  │     (NoSQL Database - Serverless)          │           │
│  └────────────────────────────────────────────┘           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                              │
                              │ API Calls (HTTPS)
                              │
┌─────────────────────────────▼─────────────────────────────┐
│                   Hetzner Infrastructure                   │
├───────────────────────────────────────────────────────────┤
│                                                            │
│  ┌──────────────────────────────────────────────────┐    │
│  │         K3s Cluster @ 157.180.57.169             │    │
│  │                                                   │    │
│  │  Namespace: gameservers                          │    │
│  │  ┌──────────────────────────────────────────┐   │    │
│  │  │  GameServer CRDs                         │   │    │
│  │  │  - patrick-minecraft-001  :30001        │   │    │
│  │  │  - patrick-valheim-001    :30334        │   │    │
│  │  │  - patrick-palworld-001   :30667        │   │    │
│  │  │  - yair-minecraft-001     :31001        │   │    │
│  │  │  - yair-rust-001          :31334        │   │    │
│  │  │  - yair-ark-001           :31667        │   │    │
│  │  └──────────────────────────────────────────┘   │    │
│  │                                                   │    │
│  │  Operator: gameserver-operator                   │    │
│  └──────────────────────────────────────────────────┘    │
│                                                            │
└────────────────────────────────────────────────────────────┘
                              │
                              │ Game Protocols (TCP/UDP)
                              │
                    ┌─────────▼──────────┐
                    │   Game Clients     │
                    │  (Players)         │
                    └────────────────────┘
```

## Authentication & Access

### Managed Identities (RBAC)

All Azure Web Apps use System-Assigned Managed Identities for:
- **Cosmos DB Data Contributor** - Read/write database access
- **ACR Pull** - Pull Docker images

### Kubernetes Authentication

- **Service Account Token** - Stored in Azure Key Vault
- **Kubeconfig** - Base64-encoded in Function App settings
- **API Server**: https://157.180.57.169:6443

## Data Flow

### Server Provisioning

1. Customer creates server via `realm-web` portal
2. Request sent to `realm-functions` (server_provision)
3. Function creates GameServer CRD in K3s cluster
4. Game Server Operator provisions pod and service
5. NodePort allocated (30000-32767 range)
6. Server details saved to Cosmos DB `servers` container
7. Status updated in CRM `servers` container

### Server Monitoring

1. `realm-admin` dashboard queries K3s API
2. Lists GameServer CRDs in `gameservers` namespace
3. Queries game server status (RCON/Query protocols)
4. Updates dashboard with live player counts
5. Stores stats in Cosmos DB `stats` container

## Terraform Management

### Modules

| Module | Path | Purpose |
|--------|------|---------|
| **realm-web** | `terraform/realm-web/` | Customer portal (4 envs) |
| **realm-functions** | `terraform/realm-functions/` | Azure Functions (4 envs) |
| **realm-admin** | `terraform/realm-admin/` | Admin dashboard (3 envs) |

### State Management

- **Backend**: Azure Storage Account
- **Container**: `terraform-state`
- **State Files**: One per module

### Deployment Commands

```bash
# Initialize
cd terraform/realm-admin
terraform init

# Plan
terraform plan

# Apply
terraform apply

# Destroy (WARNING)
terraform destroy
```

## Security Considerations

### Network Security

- All Azure traffic is HTTPS-only
- K3s API server has IP whitelisting
- Game servers use NodePort (no LoadBalancer)
- Cosmos DB has firewall rules

### Authentication

- Managed Identities (no connection strings)
- Azure AD/Entra ID for SSO
- Service Account Tokens for K8s
- API keys in Azure Key Vault

### Data Protection

- Cosmos DB automatic backups
- Point-in-time restore enabled
- Game server persistent volumes
- Backup container in Cosmos DB

## Monitoring & Observability

### Application Insights

- All Web Apps instrumented
- Custom metrics for game servers
- Performance tracking
- Error logging

### Log Analytics

- Container App logs
- K8s operator logs
- Function App execution logs
- Query logs with KQL

### Alerts

- App Service availability
- Cosmos DB throttling
- K8s pod failures
- High player count warnings

## Disaster Recovery

### Azure Resources

- **RPO**: 1 hour (Cosmos DB backups)
- **RTO**: 15 minutes (redeploy Web Apps)
- **Backup**: Automatic Cosmos DB snapshots

### Kubernetes Cluster

- **RPO**: 24 hours (persistent volume snapshots)
- **RTO**: 2 hours (rebuild cluster from IaC)
- **Backup**: GameServer CRD manifests in Git

## Cost Optimization

### Current Setup

- **Cosmos DB**: Serverless (pay per RU)
- **App Service Plan**: B1 (shared across apps)
- **Container Registry**: Basic tier
- **Function Apps**: Consumption plan
- **DNS**: Standard tier

### Monthly Estimate

- Cosmos DB: ~$50-100 (serverless)
- App Service: ~$15 (B1 tier)
- ACR: ~$5 (Basic)
- Functions: ~$10 (consumption)
- **Total**: ~$80-130/month

## Support & Maintenance

### Runbooks

- [Server Provisioning](../realm-functions/README.md)
- [Admin Deployment](../realm-admin/DEPLOYMENT.md)
- [K8s Operations](../realm-infra/hetzner/README.md)

### Contacts

- **Infrastructure**: DevOps team
- **Support**: support@realmgrid.com
- **Emergency**: On-call rotation

## Future Enhancements

### Planned

- [ ] Multi-region Cosmos DB replication
- [ ] Additional K8s clusters (US, Asia)
- [ ] Auto-scaling for game servers
- [ ] Advanced monitoring dashboards
- [ ] Automated backup testing

### Under Consideration

- [ ] Azure Kubernetes Service (AKS) migration
- [ ] Azure Container Apps for functions
- [ ] Azure Front Door for CDN
- [ ] Azure Monitor Workbooks

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2024-12-08 | Added realm-admin Terraform module | System |
| 2024-12-08 | Created 3 admin databases in DTA Cosmos | System |
| 2024-12-08 | Aligned K8s server IP (157.180.57.169) | System |
| 2024-12-08 | Mapped 6 containers to 2 customers | System |

---

**Document Version**: 1.0  
**Last Updated**: December 8, 2024  
**Maintained By**: RealmGrid DevOps Team

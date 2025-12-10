# Cosmos DB Schema Design - User Container

## Partition Key Strategy

**Chosen Strategy:** `id` (user ID) as partition key

### Rationale (Based on Azure Best Practices):

1. **High Cardinality** ✅
   - Every user has a unique ID
   - Natural distribution across partitions
   - Prevents hot partitions

2. **Query Patterns** ✅
   - Most queries are by user ID (point reads)
   - Profile updates, server listings, etc. are user-scoped
   - Efficient 1 RU point reads when querying by ID

3. **Data Size** ✅
   - Each user partition stays well under 20 GB limit
   - User data includes: profile, preferences, billing (~1-10 KB per user)
   - Related data (servers, backups) stored in separate containers

4. **Write Distribution** ✅
   - New user signups distributed evenly
   - Login updates spread across all users
   - No temporal hot spots

### Alternative Considered:

**Email as partition key** ❌
- Email queries require cross-partition lookups
- We use secondary index on email instead
- ID-based partitioning is more efficient for our access patterns

## Container Structure

```
users (container)
├── Partition Key: /id
├── Secondary Index: email (for login lookups)
└── TTL: Not enabled (users persist indefinitely unless deleted)
```

## Schema Design Decisions

### 1. Embedded vs. Referenced Data

**Billing Address: EMBEDDED** ✅
- Always accessed with user profile
- Single atomic update
- No separate lookups needed
- Follows Cosmos DB best practice: "embed data that changes together"

**Preferences: EMBEDDED** ✅
- Part of user profile
- Updated together
- Small data size

**Servers: SEPARATE CONTAINER** ✅
- Can grow beyond 20 GB per user (heavy users)
- Different access patterns (list, filter by game type)
- Separate partition key: userId + serverId (hierarchical)
- Independent scaling

### 2. Denormalization

**User data is denormalized:**
- `serverCount`, `maxServers`, `storageUsed` duplicated from aggregates
- Avoids expensive cross-partition queries
- Trade-off: slightly stale data, but updated on each operation
- **Rationale:** Performance > strict consistency for metrics

### 3. Document Type Field

**Added `type: 'user'`:**
- Future-proofs for mixed containers (users + teams in one container)
- Enables container consolidation if needed
- Minimal overhead (6 bytes)

### 4. Soft Deletes

**Soft delete pattern implemented:**
- `deleted: true` flag instead of hard delete
- `deletedAt` timestamp
- **Rationale:** 
  - GDPR compliance (audit trail)
  - Data recovery
  - Cascading cleanup can be async

### 5. Timestamps

**Multiple timestamp fields:**
- `createdAt`: Account creation (immutable)
- `updatedAt`: Last profile update
- `lastLoginAt`: Last authentication
- `lastSeenAt`: Last activity (any API call)
- **Rationale:** Different use cases (analytics, security, UX)

## Indexing Strategy

### Automatic Indexing (Cosmos DB default):
- All fields automatically indexed (except excluded paths)

### Recommended Custom Indexing:

```json
{
  "indexingPolicy": {
    "automatic": true,
    "indexingMode": "consistent",
    "includedPaths": [
      { "path": "/email/?" },
      { "path": "/customerId/?" },
      { "path": "/subscriptionStatus/?" },
      { "path": "/status/?" },
      { "path": "/createdAt/?" },
      { "path": "/lastLoginAt/?" }
    ],
    "excludedPaths": [
      { "path": "/metadata/*" },
      { "path": "/preferences/*" },
      { "path": "/billingAddress/*" },
      { "path": "/_etag/?" }
    ]
  }
}
```

**Rationale:**
- Index fields used in WHERE clauses
- Exclude nested objects rarely queried
- Reduces RU costs for writes
- Email needs index for login lookups

## Query Patterns

### Efficient (In-Partition):
```sql
-- By ID (point read - 1 RU)
SELECT * FROM c WHERE c.id = 'user-123'

-- Update user (in-partition - ~5-10 RU)
UPDATE user SET lastLoginAt = '2025-12-09T...'
```

### Cross-Partition (Indexed):
```sql
-- By email (cross-partition but indexed - ~3-5 RU)
SELECT * FROM c WHERE c.email = 'user@example.com'

-- By subscription status (cross-partition - ~10-20 RU)
SELECT * FROM c WHERE c.subscriptionStatus = 'active'
```

## Scalability Considerations

### Current Design Scales to:
- **100M+ users** (each < 20 GB partition)
- **Unlimited throughput** (distribute across partitions)
- **Global distribution** (multi-region replication)

### When to Consider Changes:

**If** you start storing large blobs per user (>1 MB):
- Move to hierarchical partition key: `userId/dataType`
- Or separate containers for large data

**If** query patterns change significantly:
- Add Global Secondary Index (GSI) with different partition key
- Example: GSI partitioned by `customerId` for billing queries

## Compliance & Security

### GDPR:
- ✅ Soft deletes with `deletedAt` timestamp
- ✅ `metadata` field for consent tracking
- ✅ Export: Single partition query by user ID
- ✅ Hard delete: `DELETE` with `?hard=true`

### Data Retention:
- Active users: Indefinite
- Soft deleted: 30 days (cleanup job)
- Hard deleted: Immediate (after backup)

### PII Fields:
- Encrypted at rest (Cosmos DB default)
- Access via Managed Identity only
- No connection strings in code

## Cost Optimization

### RU Consumption Estimates:
- User creation: ~10 RU
- Login (update lastLoginAt): ~5 RU
- Profile read: ~1 RU (point read)
- Profile update: ~10 RU
- Email lookup: ~5 RU (indexed cross-partition)

### Storage Costs:
- ~1-5 KB per user (minimal)
- 100K users = ~500 MB storage
- 1M users = ~5 GB storage

### Recommendations:
- Use serverless tier for < 10K users
- Use provisioned (autoscale) for > 10K users
- Start with 400 RU/s autoscale (scales 40-400 RU/s)

## Migration Path

### If Schema Changes:
1. Add new fields with `setdefault()` in code
2. Existing users auto-upgraded on next login
3. Run migration script for bulk updates if needed
4. No downtime required

### If Partition Key Changes:
1. Create new container with new partition key
2. Run container copy job (Azure Portal)
3. Update application config
4. Cutover with zero downtime

## Future-Ready Optional Fields

The schema includes optional fields (initialized as `null`/empty) for future features:

### Business & Compliance
- `vatNumber`: EU VAT validation
- `taxId`: Non-EU tax identification
- `companyRegistration`: Company registration number

### Growth & Marketing
- `referralCode`: Referral program
- `referredBy`: Attribution tracking
- `affiliateId`: Affiliate partnerships
- `tags`: User segmentation
- `nps`: Net Promoter Score surveys

### Enterprise Features
- `teamId`, `organizationRole`: Multi-tenant organizations
- `apiKeys`: Programmatic access
- `webhooks`: Event notifications
- `integrations`: Third-party services (Discord, Slack, GitHub)

### Advanced Security & MFA
- `mfa.enabled`: Multi-factor authentication active
- `mfa.method`: TOTP, SMS, email, or authenticator
- `mfa.totpSecret`: Encrypted TOTP secret (base32)
- `mfa.totpBackupCodes`: Hashed recovery codes
- `mfa.trustedDevices`: "Remember this device" functionality
- `mfa.mfaFailedAttempts`: Brute-force protection counter
- `mfa.mfaLockedUntil`: Temporary lockout after failed attempts
- `securitySettings.ipWhitelist`: IP restrictions
- `securitySettings.requireMfa`: Enhanced security for sensitive actions
- `securitySettings.sessionTimeout`: Custom session timeouts

### Backup & Recovery
- `backupPreferences.autoBackup`: Automated backups
- `backupPreferences.backupFrequency`: Schedule control
- `backupPreferences.retentionDays`: Retention policies

### Business Intelligence
- `credits`: Account balance system
- `lifetimeValue`: Revenue analytics
- `churnRisk`: ML-predicted churn probability
- `billingHistory`: Invoice references

**Design Philosophy:** 
- Fields are added now but unused (zero cost)
- No schema migrations needed when features launch
- Backward compatible (old users auto-upgraded)
- Cosmos DB charges only for stored data, not schema size

## References

- [Azure Cosmos DB Partitioning Best Practices](https://learn.microsoft.com/en-us/azure/cosmos-db/partitioning-overview)
- [Modeling Data in Cosmos DB](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/modeling-data)
- [Cosmos DB Indexing Policies](https://learn.microsoft.com/en-us/azure/cosmos-db/index-policy)

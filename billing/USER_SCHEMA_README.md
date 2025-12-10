# User Schema Documentation

This directory contains the complete user schema and database design documentation for Realm Grid.

## Files

- **[USER_FUNCTIONS.md](./USER_FUNCTIONS.md)** - Azure Functions API for user CRUD operations
- **[COSMOS_SCHEMA_DESIGN.md](./COSMOS_SCHEMA_DESIGN.md)** - Cosmos DB schema design rationale and best practices

## Quick Reference

### User Schema Overview

```typescript
interface User {
  // Core Identity
  id: string;
  email: string;
  name: string;
  
  // Subscription & Billing
  subscriptionStatus: string;
  billingAddress: BillingAddress;
  
  // Optional Future Fields
  vatNumber?: string;
  referralCode?: string;
  teamId?: string;
  apiKeys?: string[];
  webhooks?: string[];
  // ... and more
}
```

### API Endpoints

- `POST /api/user` - Create/upsert user (SSO login)
- `GET /api/user/{id}` - Get user by ID
- `GET /api/user?email={email}` - Get user by email
- `PATCH /api/user/{id}` - Update user profile
- `DELETE /api/user/{id}` - Soft delete user
- `DELETE /api/user/{id}?hard=true` - Hard delete user

### Database

- **Container:** `users` in `realm-dev` Cosmos DB
- **Partition Key:** `/id` (user ID)
- **Indexes:** Email (for login lookups)
- **Authentication:** Managed Identity (RBAC)

## Key Design Decisions

1. **Partition Key:** User ID for efficient point reads
2. **Billing Address:** Embedded (accessed together)
3. **Optional Fields:** Pre-defined for zero-migration launches
4. **Soft Deletes:** GDPR compliance and audit trail
5. **Denormalized Metrics:** Performance over strict consistency

## Related Documentation

- [Subscription Management](../subscriptions/README.md)
- [Mollie Integration](../billing/README.md)
- [Server Management](../servers/README.md)

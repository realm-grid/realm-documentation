# User Management Functions

These Azure Functions provide CRUD operations for user management after SSO authentication.

## Functions

### 1. `user_create` - Create/Upsert User
**Route:** `POST /api/user`

Creates a new user or updates an existing one (upsert by email). Automatically called after successful SSO login.

**Request Body:**
```json
{
  "email": "user@example.com",
  "name": "John Doe",
  "firstName": "John",
  "lastName": "Doe",
  "provider": "aad",
  "providerId": "abc-123-def",
  "avatar": "https://...",
  "metadata": {}
}
```

**Response:**
```json
{
  "success": true,
  "message": "User created successfully",
  "user": {
    "id": "uuid",
    "email": "user@example.com",
    "name": "John Doe",
    "role": "user",
    "subscriptionStatus": "none",
    "createdAt": "2025-12-09T...",
    "updatedAt": "2025-12-09T...",
    "lastLoginAt": "2025-12-09T..."
  },
  "isNew": true
}
```

### 2. `user_get` - Get User
**Route:** `GET /api/user/{id}` or `GET /api/user?email={email}`

Retrieves a user by ID or email.

**Response:**
```json
{
  "success": true,
  "user": {
    "id": "uuid",
    "email": "user@example.com",
    ...
  }
}
```

### 3. `user_update` - Update User
**Route:** `PATCH /api/user/{id}` or `PUT /api/user/{id}`

Updates user profile information.

**Request Body:**
```json
{
  "name": "Jane Doe",
  "firstName": "Jane",
  "avatar": "https://...",
  "metadata": {
    "customField": "value"
  }
}
```

**Response:**
```json
{
  "success": true,
  "message": "User updated successfully",
  "user": { ... }
}
```

### 4. `user_delete` - Delete User
**Route:** `DELETE /api/user/{id}` or `DELETE /api/user/{id}?hard=true`

Soft-deletes a user by default (marks as deleted). Use `?hard=true` for permanent deletion.

**Response:**
```json
{
  "success": true,
  "message": "User deleted successfully"
}
```

## User Schema

```typescript
interface User {
  // Identity
  id: string;                    // UUID (partition key)
  email: string;                 // User email (unique, indexed)
  emailVerified: boolean;        // Email verification status
  name: string;                  // Full name
  displayName: string;           // Display name (can differ from legal name)
  firstName: string;             // First name
  lastName: string;              // Last name
  provider: 'aad' | 'google' | 'discord';  // SSO provider
  providerId: string;            // Provider's user ID
  role: 'user' | 'admin';        // User role
  avatar: string;                // Avatar URL
  
  // Subscription & Billing (Mollie Integration)
  subscriptionStatus: 'none' | 'active' | 'cancelled' | 'expired' | 'past_due' | 'trialing';
  subscriptionId: string | null; // Mollie subscription ID
  customerId: string | null;     // Mollie customer ID
  subscriptionPlan: string | null; // Plan name/tier (e.g., 'starter', 'pro', 'enterprise')
  subscriptionStartDate: string | null; // ISO 8601
  subscriptionEndDate: string | null;   // ISO 8601
  
  // Billing Address (required for invoices/VAT compliance)
  billingAddress: {
    firstName: string;
    lastName: string;
    company: string | null;      // Optional for business accounts
    street: string | null;
    streetNumber: string | null;
    postalCode: string | null;
    city: string | null;
    region: string | null;       // State/Province
    country: string | null;      // ISO 3166-1 alpha-2 (NL, DE, US, etc.)
  };
  
  // Contact Information
  phone: string | null;          // Phone number with country code
  phoneVerified: boolean;        // Phone verification status
  language: string;              // Preferred language (ISO 639-1: en, nl, de, etc.)
  timezone: string;              // IANA timezone (Europe/Amsterdam, America/New_York, etc.)
  
  // Account Status
  status: 'active' | 'suspended' | 'banned' | 'deleted';
  accountType: 'personal' | 'business';
  
  // User Preferences
  preferences: {
    emailNotifications: boolean;
    marketingEmails: boolean;
    newsletter: boolean;
    twoFactorEnabled: boolean;
  };
  
  // Usage & Limits (based on subscription plan)
  serverCount: number;           // Current number of active servers
  maxServers: number;            // Maximum allowed servers
  storageUsed: number;           // Storage used in GB
  storageLimit: number;          // Storage limit in GB
  
  // Timestamps
  createdAt: string;             // ISO 8601 timestamp
  updatedAt: string;             // ISO 8601 timestamp
  lastLoginAt: string;           // ISO 8601 timestamp
  lastSeenAt: string;            // Last activity timestamp
  deleted?: boolean;             // Soft delete flag
  deletedAt?: string;            // Soft delete timestamp
  
  // Additional Fields
  type: 'user';                  // Document type (for mixed containers)
  metadata: Record<string, any>; // Custom metadata (flexible field)
}
```

### Field Guidelines

**Billing Address:**
- Required for paid subscriptions (VAT/tax compliance)
- Country code must be ISO 3166-1 alpha-2 (2 letters)
- For EU customers, VAT validation may be required
- Business accounts should provide company name

**Language & Timezone:**
- Language uses ISO 639-1 (2-letter codes): en, nl, de, fr, etc.
- Timezone uses IANA format: Europe/Amsterdam, America/New_York, etc.
- Used for localization and scheduling

**Subscription Status:**
- `none`: No active subscription (free tier or no plan)
- `active`: Currently paying and active
- `trialing`: In trial period
- `past_due`: Payment failed, grace period
- `cancelled`: Cancelled but still active until period end
- `expired`: Subscription ended

**Account Type:**
- `personal`: Individual consumer account
- `business`: Company/organization account (may need VAT number)

## Optional Future Fields

These fields are initialized as `null`/empty but can be populated as features are built:

### Tax & Compliance
```typescript
vatNumber: string | null;              // EU VAT number (e.g., "NL123456789B01")
taxId: string | null;                  // Tax ID for non-EU (e.g., US EIN)
companyRegistration: string | null;    // Company registration number
```

### Referral & Affiliates
```typescript
referralCode: string | null;           // User's unique referral code
referredBy: string | null;             // User ID of referrer
affiliateId: string | null;            // Affiliate partner ID
```

### Teams & Organizations
```typescript
teamId: string | null;                 // Primary team/organization ID
organizationRole: string | null;       // 'owner' | 'admin' | 'member'
```

### API & Integrations
```typescript
apiKeys: string[];                     // List of API key IDs
webhooks: string[];                    // Webhook URLs for events
integrations: {                        // Third-party integrations
  discord?: { userId: string, serverId: string },
  slack?: { workspaceId: string, channelId: string },
  github?: { username: string },
  // ... extensible
}
```

### Backup Preferences
```typescript
backupPreferences: {
  autoBackup: boolean;                 // Enable automatic backups
  backupFrequency: 'daily' | 'weekly' | 'manual';
  retentionDays: number;               // How long to keep backups
}
```

### Multi-Factor Authentication (MFA)
```typescript
mfa: {
  enabled: boolean;                    // MFA is active
  method: 'totp' | 'sms' | 'email' | 'authenticator' | null;
  totpSecret: string | null;           // Encrypted TOTP secret (base32)
  totpBackupCodes: string[];           // Hashed one-time recovery codes
  phoneNumberMfa: string | null;       // Phone for SMS (can differ from main phone)
  trustedDevices: Array<{
    deviceId: string;                  // Unique device identifier
    name: string;                      // User-friendly name ("iPhone 15", "Chrome on MacBook")
    addedAt: string;                   // ISO 8601 timestamp
    lastUsedAt: string;                // Last login from this device
    ipAddress?: string;                // Last known IP
    userAgent?: string;                // Browser/device info
  }>;
  lastMfaPrompt: string | null;        // Last time user was prompted
  mfaFailedAttempts: number;           // Failed attempts counter
  mfaLockedUntil: string | null;       // Lockout until this time
}
```

**MFA Implementation Notes:**
- `totpSecret`: Store encrypted with Azure Key Vault reference
- `totpBackupCodes`: Hash with bcrypt/argon2 (never store plain)
- `trustedDevices`: Allow "Remember this device for 30 days"
- Lockout: 5 failed attempts = 15 minute lockout
- Methods:
  - `totp`: Time-based OTP (Google Authenticator, Authy)
  - `sms`: SMS verification codes
  - `email`: Email verification codes
  - `authenticator`: Platform authenticators (WebAuthn, Face ID, Touch ID)

### Security Settings
```typescript
securitySettings: {
  ipWhitelist: string[];               // Allowed IP addresses
  sessionTimeout: number;              // Session timeout in seconds
  requireMfa: boolean;                 // Require MFA for sensitive actions
}
```

### Analytics & Business Intelligence
```typescript
credits: number;                       // Account credits/balance
lifetimeValue: number;                 // Total revenue from user (analytics)
churnRisk: number | null;              // Predicted churn probability (0-1)
nps: number | null;                    // Net Promoter Score (-100 to 100)
tags: string[];                        // Custom tags for segmentation
billingHistory: string[];              // Invoice IDs (or use separate container)
```

### Usage
These fields can be updated via the `user_update` function or populated during specific workflows:
- VAT/Tax: During checkout or billing setup
- Referrals: When user signs up via referral link
- Teams: When creating or joining organizations
- API Keys: When user generates API keys
- Analytics: Background jobs or ML models

## Database

- **Database:** `realm-dev` (or `COSMOS_DATABASE` env var)
- **Container:** `users`
- **Partition Key:** `id`

## Authentication

All functions use:
- **Auth Level:** `function` (requires function key)
- **Cosmos DB:** Managed Identity (DefaultAzureCredential)

## Integration

The local auth flow automatically calls `user_create` after successful OAuth:

1. User logs in via Microsoft/Google/Discord
2. OAuth completes and returns ID token
3. Backend decodes ID token
4. Backend calls `POST /api/user` to create/update user
5. User is redirected to dashboard

## Environment Variables

Required in Azure Functions:
```bash
COSMOS_ENDPOINT=https://realm-dta-app-cosmos.documents.azure.com:443/
COSMOS_DATABASE=realm-dev
```

## RBAC Permissions

The Function App's Managed Identity needs:
- **Cosmos DB Built-in Data Contributor** role on the Cosmos DB account

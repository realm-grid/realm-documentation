# SSO Authentication Architecture

## Overview

Realm Grid uses Azure Entra ID (formerly Azure AD) for Single Sign-On (SSO) authentication. The authentication flow is implemented through Azure Functions that handle OAuth 2.0 authorization code flow with PKCE.

## Authentication Flow

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Frontend      │     │  Azure Functions │     │   Azure AD      │     │   Key Vault     │
│  (realm-admin/  │     │  (realm-dev-api) │     │  (Entra ID)     │     │ (realm-shared)  │
│   realm-web)    │     │                  │     │                 │     │                 │
└────────┬────────┘     └────────┬────────┘     └────────┬────────┘     └────────┬────────┘
         │                       │                       │                       │
         │ 1. Login button       │                       │                       │
         │─────────────────────>│                       │                       │
         │                       │                       │                       │
         │ 2. Redirect to /api/auth/login/aad           │                       │
         │─────────────────────>│                       │                       │
         │                       │                       │                       │
         │                       │ 3. Get client secret │                       │
         │                       │───────────────────────────────────────────────>
         │                       │<──────────────────────────────────────────────│
         │                       │                       │                       │
         │ 4. 302 Redirect to Azure AD authorize        │                       │
         │<──────────────────────│                       │                       │
         │                       │                       │                       │
         │ 5. User authenticates │                       │                       │
         │──────────────────────────────────────────────>│                       │
         │                       │                       │                       │
         │ 6. Redirect to /api/auth/callback/aad        │                       │
         │<──────────────────────────────────────────────│                       │
         │─────────────────────>│                       │                       │
         │                       │                       │                       │
         │                       │ 7. Exchange code      │                       │
         │                       │─────────────────────>│                       │
         │                       │<────────────────────│                       │
         │                       │   (tokens)           │                       │
         │                       │                       │                       │
         │                       │ 8. Create JWT session │                       │
         │                       │───────────────────────────────────────────────>
         │                       │<──────────────────────────────────────────────│
         │                       │   (jwt-signing-secret)                        │
         │                       │                       │                       │
         │ 9. Redirect with JWT  │                       │                       │
         │<──────────────────────│                       │                       │
         │                       │                       │                       │
         │ 10. Store JWT locally │                       │                       │
         │                       │                       │                       │
```

## Azure Resources

### SSO Application

| Property | Value |
|----------|-------|
| **Name** | realm-admin-sso-app |
| **Client ID** | 524d8f43-6a4a-4117-b4fe-008906598d0b |
| **Tenant ID** | 34dd9821-1508-4858-974c-e5fd1493a58f |
| **Type** | Web Application |

### Redirect URIs

The SSO app has the following redirect URIs configured:

- `https://realm-dev-api-fa.azurewebsites.net/api/auth/callback/aad`
- `https://realm-sta-api-fa.azurewebsites.net/api/auth/callback/aad`
- `https://realm-prd-api-fa.azurewebsites.net/api/auth/callback/aad`
- `http://localhost:7071/api/auth/callback/aad`

### Key Vault Secrets

| Secret Name | Purpose |
|-------------|---------|
| `sso-client-secret` | OAuth client secret for token exchange |
| `jwt-signing-secret` | Secret for signing JWT session tokens |

## Function App Configuration

### Environment Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `SSO_CLIENT_ID` | Azure AD Application (client) ID | `524d8f43-6a4a-4117-b4fe-008906598d0b` |
| `SSO_TENANT_ID` | Azure AD Tenant ID | `34dd9821-1508-4858-974c-e5fd1493a58f` |
| `SSO_CLIENT_SECRET` | Key Vault reference | `@Microsoft.KeyVault(SecretUri=...)` |
| `FUNCTIONS_BASE_URL` | Base URL for callbacks | `https://realm-dev-api-fa.azurewebsites.net/api` |
| `JWT_SIGNING_SECRET` | Key Vault reference | `@Microsoft.KeyVault(SecretUri=...)` |

## API Endpoints

### Login

**Endpoint:** `GET /api/auth/login/{provider}`

**Supported Providers:** `aad` (Azure AD)

**Query Parameters:**
| Parameter | Required | Description |
|-----------|----------|-------------|
| `redirect_uri` or `post_login_redirect_uri` | Yes | Where to redirect after successful auth |

**Response:** 302 redirect to Azure AD authorize endpoint

**Example:**
```
GET /api/auth/login/aad?post_login_redirect_uri=https://realm-admin.com/dashboard
```

### Callback

**Endpoint:** `GET /api/auth/callback/{provider}`

**Query Parameters:**
| Parameter | Description |
|-----------|-------------|
| `code` | Authorization code from Azure AD |
| `state` | Contains encrypted post_login_redirect_uri |

**Response:** 302 redirect to original `post_login_redirect_uri` with `auth_token` parameter

**Example Response:**
```
302 Location: https://realm-admin.com/dashboard?auth_token=eyJhbGciOiJIUzI1...
```

### Get Current User

**Endpoint:** `GET /api/auth/me`

**Headers:**
| Header | Value |
|--------|-------|
| `Authorization` | `Bearer {jwt_token}` |

**Response (200 OK):**
```json
{
  "userId": "aad|12345678-1234-1234-1234-123456789012",
  "email": "user@example.com",
  "name": "John Doe",
  "provider": "aad",
  "tenantId": "34dd9821-1508-4858-974c-e5fd1493a58f"
}
```

**Response (401 Unauthorized):**
```json
{
  "error": "unauthorized",
  "message": "Invalid or expired token"
}
```

### Logout

**Endpoint:** `GET /api/auth/logout`

**Query Parameters:**
| Parameter | Required | Description |
|-----------|----------|-------------|
| `redirect_uri` or `post_login_redirect_uri` | Yes | Where to redirect after logout |

**Response:** 302 redirect to the specified URL with `logged_out=true` parameter

## JWT Token Structure

The JWT token created by the callback function contains:

```json
{
  "sub": "aad|12345678-1234-1234-1234-123456789012",
  "email": "user@example.com",
  "name": "John Doe",
  "provider": "aad",
  "tenantId": "34dd9821-1508-4858-974c-e5fd1493a58f",
  "iat": 1699999999,
  "exp": 1700086399
}
```

**Token Expiry:** 24 hours

## Frontend Integration

### Token Storage

Tokens are stored in both:
- `localStorage` as `realm_auth_token`
- HTTP-only cookie `realm_auth_token` (for SSR)

### URL Mapping

Frontends map their origin to the appropriate Functions URL:

```typescript
const API_URL_MAP = {
  'localhost:3000': 'http://localhost:7071/api',
  'localhost:4321': 'http://localhost:7071/api',
  'realm-web-dev.azurewebsites.net': 'https://realm-dev-api-fa.azurewebsites.net/api',
  'realm-admin-dev.azurewebsites.net': 'https://realm-dev-api-fa.azurewebsites.net/api',
  // ... other environments
};
```

### Authentication Hook (realm-admin)

```typescript
// app/hooks/useAuth.ts
const login = (provider: string = 'aad') => {
  const currentUrl = window.location.href;
  window.location.href = `${apiUrl}/auth/login/${provider}?post_login_redirect_uri=${encodeURIComponent(currentUrl)}`;
};

const logout = () => {
  localStorage.removeItem('realm_auth_token');
  const loginUrl = `${window.location.origin}/login`;
  window.location.href = `${apiUrl}/auth/logout?post_login_redirect_uri=${encodeURIComponent(loginUrl)}`;
};
```

### Token Extraction

After redirect back from SSO:

```typescript
// Check URL for auth_token parameter
const urlParams = new URLSearchParams(window.location.search);
const tokenFromUrl = urlParams.get('auth_token');
if (tokenFromUrl) {
  localStorage.setItem('realm_auth_token', tokenFromUrl);
  // Clean up URL
  window.history.replaceState({}, '', window.location.pathname);
}
```

## Easy Auth Integration

Azure Easy Auth is configured on:
- **Function App:** realm-dev-api-fa
- **Web Apps:** realm-admin-dev, realm-web-dev

This provides an additional layer of authentication and allows the use of `/.auth/me` endpoint for getting user claims.

### Managed Identity App Roles

Web Apps have Managed Identity app role assignments allowing them to call the Function App API:

```bash
# Get Managed Identity principal ID
az webapp identity show --name realm-admin-dev --resource-group xevolve-dta-rg --query principalId

# Assign app role (done via Azure Portal or Graph API)
```

## Troubleshooting

### Common Issues

#### 1. 404 on Auth Endpoints

**Cause:** Cold start or HEAD requests not supported

**Solution:** 
- Added HEAD method to all auth function.json files
- Keep-alive function pings auth endpoints

#### 2. Callback URL Mismatch

**Cause:** Redirect URI in token request doesn't match authorize request

**Solution:** Both auth_login and auth_callback use the same pattern:
```python
callback_url = f'{FUNCTIONS_BASE_URL}/auth/callback/{provider}'
```

#### 3. Invalid Client Secret

**Cause:** Key Vault secret doesn't match Azure AD app secret

**Solution:** Create new secret in Azure AD app and update Key Vault

#### 4. Token Not Saved

**Cause:** Looking for wrong parameter name

**Solution:** Frontend must look for `auth_token` parameter (not `token`)

### Debug Commands

```bash
# Test login redirect
curl -I "https://realm-dev-api-fa.azurewebsites.net/api/auth/login/aad?post_login_redirect_uri=https://example.com"

# Test me endpoint with token
curl -H "Authorization: Bearer eyJ..." "https://realm-dev-api-fa.azurewebsites.net/api/auth/me"

# Check Function App settings
az functionapp config appsettings list --name realm-dev-api-fa --resource-group xevolve-dta-rg
```

### E2E Test

A Playwright test exists at `/realm-e2e-tests/test_sso_auth.py`:

```bash
cd realm-e2e-tests
python test_sso_auth.py
```

Test user: `test.sso.user@xevolve.io`

## Security Considerations

1. **Client Secret:** Stored in Azure Key Vault, referenced via Key Vault reference in Function App settings
2. **JWT Secret:** Stored in Key Vault, never exposed to frontend
3. **Token Expiry:** 24-hour expiry prevents long-term token compromise
4. **HTTPS Only:** All production endpoints are HTTPS-only
5. **State Parameter:** Encrypted state prevents CSRF attacks
6. **Redirect URI Validation:** Azure AD only allows pre-configured redirect URIs

## Future Identity Providers

The architecture supports adding additional identity providers:

1. Add provider-specific logic in `auth_login/__init__.py`
2. Add callback handling in `auth_callback/__init__.py`
3. Configure redirect URIs in the identity provider
4. Update frontend to offer new provider option

Potential providers:
- Microsoft Personal Accounts (MSA)
- Google
- GitHub
- Custom SAML/OIDC

## Change History

| Date | Change | Author |
|------|--------|--------|
| 2025-01-XX | Initial SSO implementation with Azure AD | - |
| 2025-01-XX | Renamed ENTRA_ variables to SSO_ for clarity | - |
| 2025-01-XX | Fixed callback URL construction | - |
| 2025-01-XX | Added HEAD method support | - |

# Multi-Factor Authentication (MFA) Implementation Guide

## Overview

Realm Grid supports multiple MFA methods for enhanced account security:
- **TOTP** (Time-based One-Time Password) - Google Authenticator, Authy, 1Password
- **SMS** - Text message verification codes
- **Email** - Email verification codes
- **WebAuthn** - Platform authenticators (Face ID, Touch ID, Windows Hello)

## Schema Fields

### MFA Object Structure

```typescript
mfa: {
  enabled: boolean;                    // Is MFA active?
  method: 'totp' | 'sms' | 'email' | 'authenticator' | null;
  
  // TOTP (Authenticator Apps)
  totpSecret: string | null;           // Base32-encoded secret (encrypted)
  totpBackupCodes: string[];           // Hashed 8-10 digit recovery codes
  
  // SMS MFA
  phoneNumberMfa: string | null;       // Can differ from billing phone
  
  // Trusted Devices (Remember Me)
  trustedDevices: Array<{
    deviceId: string;                  // UUID or fingerprint
    name: string;                      // "iPhone 15" or "Chrome on MacBook"
    addedAt: string;                   // When device was trusted
    lastUsedAt: string;                // Last successful login
    ipAddress?: string;                // Last known IP
    userAgent?: string;                // Browser/OS info
  }>;
  
  // Security & Rate Limiting
  lastMfaPrompt: string | null;        // Last prompt timestamp
  mfaFailedAttempts: number;           // Failed verification counter
  mfaLockedUntil: string | null;       // Lockout expiry timestamp
}
```

## Implementation Steps

### 1. Enable MFA (Setup Flow)

**User Journey:**
1. User navigates to Security Settings
2. Clicks "Enable Two-Factor Authentication"
3. Chooses method (TOTP, SMS, Email)
4. Completes setup wizard
5. Downloads backup codes
6. Verifies with first code

**API Call:**
```http
POST /api/user/{userId}/mfa/enable
{
  "method": "totp"
}

Response:
{
  "success": true,
  "secret": "JBSWY3DPEHPK3PXP",  // Base32 secret for QR code
  "qrCodeUrl": "otpauth://totp/RealmGrid:user@example.com?secret=...",
  "backupCodes": [
    "12345678",
    "87654321",
    // ... 8 more codes
  ]
}
```

**Backend (Azure Function):**
```python
import pyotp
import secrets
import bcrypt

def enable_mfa_totp(user_id: str):
    # Generate TOTP secret
    secret = pyotp.random_base32()
    
    # Generate 10 backup codes
    backup_codes = [secrets.token_hex(4).upper() for _ in range(10)]
    backup_codes_hashed = [bcrypt.hashpw(code.encode(), bcrypt.gensalt()) for code in backup_codes]
    
    # Store encrypted secret (use Azure Key Vault reference)
    encrypted_secret = encrypt_with_key_vault(secret)
    
    # Update user document
    user_doc = get_user(user_id)
    user_doc['mfa'] = {
        'enabled': False,  # Not enabled until verified
        'method': 'totp',
        'totpSecret': encrypted_secret,
        'totpBackupCodes': backup_codes_hashed,
        'trustedDevices': [],
        'lastMfaPrompt': None,
        'mfaFailedAttempts': 0,
        'mfaLockedUntil': None
    }
    update_user(user_doc)
    
    # Return plain codes to user (one-time only)
    return {
        'secret': secret,
        'qrCodeUrl': pyotp.totp.TOTP(secret).provisioning_uri(
            name=user_doc['email'],
            issuer_name='RealmGrid'
        ),
        'backupCodes': backup_codes
    }
```

### 2. Verify MFA Setup

**API Call:**
```http
POST /api/user/{userId}/mfa/verify-setup
{
  "code": "123456"
}

Response:
{
  "success": true,
  "message": "MFA enabled successfully"
}
```

**Backend:**
```python
def verify_mfa_setup(user_id: str, code: str):
    user_doc = get_user(user_id)
    secret = decrypt_from_key_vault(user_doc['mfa']['totpSecret'])
    
    totp = pyotp.TOTP(secret)
    if totp.verify(code, valid_window=1):  # Allow 30s clock drift
        user_doc['mfa']['enabled'] = True
        user_doc['updatedAt'] = datetime.utcnow().isoformat()
        update_user(user_doc)
        return {'success': True}
    else:
        return {'success': False, 'error': 'Invalid code'}
```

### 3. Login with MFA

**Flow:**
1. User enters email/password (SSO)
2. Backend checks `user.mfa.enabled`
3. If enabled, prompt for MFA code
4. Verify code or backup code
5. Check if device is trusted
6. If not trusted, offer "Remember this device"

**API Call:**
```http
POST /api/auth/mfa/verify
{
  "userId": "user-123",
  "code": "123456",
  "rememberDevice": true,
  "deviceInfo": {
    "name": "Chrome on MacBook Pro",
    "userAgent": "Mozilla/5.0...",
    "ipAddress": "203.0.113.42"
  }
}

Response:
{
  "success": true,
  "deviceId": "device-abc-123"
}
```

**Backend:**
```python
def verify_mfa_login(user_id: str, code: str, device_info: dict):
    user_doc = get_user(user_id)
    mfa = user_doc['mfa']
    
    # Check lockout
    if mfa['mfaLockedUntil']:
        locked_until = datetime.fromisoformat(mfa['mfaLockedUntil'])
        if datetime.utcnow() < locked_until:
            return {'success': False, 'error': 'Account locked', 'retryAfter': locked_until.isoformat()}
    
    # Decrypt secret
    secret = decrypt_from_key_vault(mfa['totpSecret'])
    totp = pyotp.TOTP(secret)
    
    # Verify TOTP code
    is_valid = totp.verify(code, valid_window=1)
    
    # Or check backup codes
    if not is_valid:
        for hashed_code in mfa['totpBackupCodes']:
            if bcrypt.checkpw(code.encode(), hashed_code):
                is_valid = True
                # Remove used backup code
                mfa['totpBackupCodes'].remove(hashed_code)
                break
    
    if is_valid:
        # Reset failed attempts
        mfa['mfaFailedAttempts'] = 0
        mfa['mfaLockedUntil'] = None
        mfa['lastMfaPrompt'] = datetime.utcnow().isoformat()
        
        # Add trusted device if requested
        if device_info.get('rememberDevice'):
            device_id = str(uuid.uuid4())
            mfa['trustedDevices'].append({
                'deviceId': device_id,
                'name': device_info.get('name', 'Unknown Device'),
                'addedAt': datetime.utcnow().isoformat(),
                'lastUsedAt': datetime.utcnow().isoformat(),
                'ipAddress': device_info.get('ipAddress'),
                'userAgent': device_info.get('userAgent')
            })
        
        update_user(user_doc)
        return {'success': True, 'deviceId': device_id if device_info.get('rememberDevice') else None}
    else:
        # Increment failed attempts
        mfa['mfaFailedAttempts'] += 1
        
        # Lock after 5 failed attempts
        if mfa['mfaFailedAttempts'] >= 5:
            mfa['mfaLockedUntil'] = (datetime.utcnow() + timedelta(minutes=15)).isoformat()
        
        update_user(user_doc)
        return {'success': False, 'error': 'Invalid code', 'attemptsRemaining': 5 - mfa['mfaFailedAttempts']}
```

### 4. Check Trusted Device

**Before prompting for MFA, check if device is trusted:**

```python
def is_device_trusted(user_id: str, device_id: str):
    user_doc = get_user(user_id)
    trusted_devices = user_doc['mfa']['trustedDevices']
    
    for device in trusted_devices:
        if device['deviceId'] == device_id:
            # Update last used
            device['lastUsedAt'] = datetime.utcnow().isoformat()
            update_user(user_doc)
            return True
    
    return False
```

### 5. Disable MFA

**Requires MFA verification to disable (prevent hijacking):**

```http
POST /api/user/{userId}/mfa/disable
{
  "code": "123456",  // Current MFA code required
  "password": "user-password"  // Or require password confirmation
}
```

## Security Best Practices

### 1. Secret Storage
```python
# NEVER store TOTP secrets in plain text
# Use Azure Key Vault reference:
user_doc['mfa']['totpSecret'] = "@Microsoft.KeyVault(VaultName=realm-shared-kv;SecretName=mfa-secret-{userId})"
```

### 2. Backup Codes
```python
# ALWAYS hash backup codes
import bcrypt
hashed = bcrypt.hashpw(code.encode(), bcrypt.gensalt())
```

### 3. Rate Limiting
- 5 failed attempts = 15 minute lockout
- Consider IP-based rate limiting
- Log all MFA failures for security monitoring

### 4. Trusted Devices
- Expire after 30 days of inactivity
- Allow users to revoke devices
- Show device list in settings

### 5. Recovery Flow
- Backup codes are one-time use
- Offer alternative: email/SMS recovery
- Consider admin-assisted recovery for enterprise

## Azure Functions to Create

1. **`mfa_enable`** - Start MFA setup
2. **`mfa_verify_setup`** - Complete MFA setup
3. **`mfa_verify_login`** - Verify MFA during login
4. **`mfa_disable`** - Disable MFA
5. **`mfa_regenerate_backup_codes`** - Generate new backup codes
6. **`mfa_trusted_devices_list`** - List trusted devices
7. **`mfa_trusted_devices_revoke`** - Remove trusted device

## Dependencies

Add to `requirements.txt`:
```
pyotp>=2.9.0          # TOTP generation/verification
bcrypt>=4.0.0         # Backup code hashing
```

## Testing

```python
# Test TOTP generation
import pyotp
secret = 'JBSWY3DPEHPK3PXP'
totp = pyotp.TOTP(secret)
print(totp.now())  # Current code

# Test backup code hashing
import bcrypt
code = '12345678'
hashed = bcrypt.hashpw(code.encode(), bcrypt.gensalt())
assert bcrypt.checkpw(code.encode(), hashed)
```

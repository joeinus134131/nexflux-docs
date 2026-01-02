# Two-Factor Authentication (2FA) Documentation

Dokumentasi lengkap untuk implementasi Two-Factor Authentication di NexFlux Frontend.

---

## 📋 Table of Contents

1. [Overview](#1-overview)
2. [Current Implementation](#2-current-implementation)
3. [API Endpoints Required](#3-api-endpoints-required)
4. [Frontend Components](#4-frontend-components)
5. [User Flows](#5-user-flows)
6. [Additional Features to Implement](#6-additional-features-to-implement)
7. [Security Considerations](#7-security-considerations)

---

## 1. Overview

Two-Factor Authentication (2FA) menambahkan lapisan keamanan ekstra ke akun pengguna dengan memerlukan:
1. **Something they know** - Password
2. **Something they have** - TOTP code dari authenticator app

### Supported Methods
- **TOTP (Time-based One-Time Password)** - Google Authenticator, Authy, 1Password, dll.
- **Backup Codes** - One-time use codes untuk recovery

---

## 2. Current Implementation

### 2.1 Enable 2FA Flow (Settings Page)
**File:** `src/pages/Settings.tsx`

```
User clicks "Enable 2FA" toggle
    ↓
POST /auth/2fa/enable → Returns QR code, secret, backup codes
    ↓
Modal shows QR code + manual secret
    ↓
User scans QR code with authenticator app
    ↓
User enters 6-digit TOTP code
    ↓
POST /auth/2fa/verify → Activates 2FA
    ↓
2FA Enabled ✓
```

### 2.2 Login with 2FA Flow
**File:** `src/components/auth/SignInForm.tsx`

```
User enters email + password
    ↓
POST /auth/login → Returns { requires_2fa: true, temp_token: "..." }
    ↓
Show 2FA verification form
    ↓
User enters 6-digit TOTP code
    ↓
POST /auth/login/2fa → Returns JWT token
    ↓
Login successful ✓
```

### 2.3 Disable 2FA Flow
```
User clicks "Disable 2FA" toggle
    ↓
Modal asks for password + TOTP code
    ↓
POST /auth/2fa/disable → Deactivates 2FA
    ↓
2FA Disabled ✓
```

---

## 3. API Endpoints Required

### 3.1 Enable 2FA
**Endpoint:** `POST /auth/2fa/enable`

**Request:** No body required (uses JWT token)

**Response:**
```json
{
  "success": true,
  "data": {
    "secret": "JBSWY3DPEHPK3PXP",
    "qr_code_url": "otpauth://totp/NexFlux:user@example.com?secret=JBSWY3DPEHPK3PXP&issuer=NexFlux",
    "backup_codes": [
      "ABC123DEF",
      "GHI456JKL",
      "MNO789PQR",
      "STU012VWX",
      "YZA345BCD"
    ]
  },
  "message": "Two-factor authentication setup initiated"
}
```

### 3.2 Verify 2FA Setup
**Endpoint:** `POST /auth/2fa/verify`

**Request:**
```json
{
  "code": "123456"
}
```

**Response:**
```json
{
  "success": true,
  "message": "Two-factor authentication verified and activated"
}
```

### 3.3 Get 2FA Status
**Endpoint:** `GET /auth/2fa/status`

**Response:**
```json
{
  "success": true,
  "data": {
    "enabled": true,
    "enabled_at": "2024-12-26T10:00:00Z",
    "backup_codes_remaining": 3,
    "last_used": "2024-12-27T08:00:00Z"
  }
}
```

### 3.4 Disable 2FA
**Endpoint:** `POST /auth/2fa/disable`

**Request:**
```json
{
  "code": "123456",
  "password": "userPassword123"
}
```

**Response:**
```json
{
  "success": true,
  "message": "Two-factor authentication disabled"
}
```

### 3.5 Login with 2FA
**Endpoint:** `POST /auth/login/2fa`

**Request:**
```json
{
  "email": "user@example.com",
  "password": "userPassword123",
  "code": "123456",
  "temp_token": "temporary-token-from-login"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "token": "jwt-token-here",
    "user": {
      "id": "user-id",
      "email": "user@example.com",
      "name": "User Name"
    }
  }
}
```

### 3.6 Use Backup Code
**Endpoint:** `POST /auth/2fa/backup`

**Request:**
```json
{
  "email": "user@example.com",
  "password": "userPassword123",
  "backup_code": "ABC123DEF"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "token": "jwt-token-here",
    "backup_codes_remaining": 2
  },
  "message": "Backup code used successfully"
}
```

### 3.7 Regenerate Backup Codes
**Endpoint:** `POST /auth/2fa/backup/regenerate`

**Request:**
```json
{
  "password": "userPassword123",
  "code": "123456"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "backup_codes": [
      "NEW123ABC",
      "DEF456GHI",
      "JKL789MNO",
      "PQR012STU",
      "VWX345YZA"
    ]
  },
  "message": "Backup codes regenerated. Previous codes are now invalid."
}
```

---

## 4. Frontend Components

### 4.1 Current Components

| Component | Location | Description |
|-----------|----------|-------------|
| SecuritySection | `Settings.tsx` | 2FA enable/disable toggle with modals |
| 2FA Setup Modal | `Settings.tsx` | QR code display, backup codes, verification |
| 2FA Disable Modal | `Settings.tsx` | Password + code confirmation |
| 2FA Login Form | `SignInForm.tsx` | 6-digit code input during login |

### 4.2 Component States

```typescript
// Settings.tsx - 2FA States
const [twoFactorEnabled, setTwoFactorEnabled] = useState(false);
const [twoFactorLoading, setTwoFactorLoading] = useState(false);
const [show2FASetupModal, setShow2FASetupModal] = useState(false);
const [show2FADisableModal, setShow2FADisableModal] = useState(false);
const [twoFactorSetup, setTwoFactorSetup] = useState<{
  secret: string;
  qr_code_url: string;
  backup_codes: string[];
} | null>(null);
const [verifyCode, setVerifyCode] = useState('');
const [disableForm, setDisableForm] = useState({ code: '', password: '' });
const [twoFactorError, setTwoFactorError] = useState<string | null>(null);
const [twoFactorSuccess, setTwoFactorSuccess] = useState<string | null>(null);
const [backupCodesRemaining, setBackupCodesRemaining] = useState<number>(0);

// SignInForm.tsx - Login 2FA States
const [requires2FA, setRequires2FA] = useState(false);
const [twoFactorCode, setTwoFactorCode] = useState("");
const [tempToken, setTempToken] = useState<string | null>(null);
```

---

## 5. User Flows

### 5.1 First-Time 2FA Setup
```
┌─────────────────────────────────────────────────────────────┐
│                     SETTINGS PAGE                            │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Two-Factor Authentication                             │  │
│  │ ○ Enable 2FA                          [Recommended]   │  │
│  │   Protect your account with 2FA           [Toggle]    │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼ [Click Toggle]
┌─────────────────────────────────────────────────────────────┐
│                   2FA SETUP MODAL                            │
│                                                              │
│  ┌───────────────────────────────────────────────────────┐ │
│  │ 1. Scan this QR code                                   │ │
│  │    ┌─────────────┐                                     │ │
│  │    │ [QR CODE]   │                                     │ │
│  │    └─────────────┘                                     │ │
│  │                                                        │ │
│  │ Or enter manually: JBSWY3DPEHPK3PXP                   │ │
│  └───────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌───────────────────────────────────────────────────────┐ │
│  │ ⚠️ Save your backup codes                              │ │
│  │   ABC123DEF  GHI456JKL  MNO789PQR                     │ │
│  │   STU012VWX  YZA345BCD                                │ │
│  └───────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌───────────────────────────────────────────────────────┐ │
│  │ 2. Enter verification code                             │ │
│  │    ┌──────────────────────┐                           │ │
│  │    │     [0][0][0][0][0][0]                           │ │
│  │    └──────────────────────┘                           │ │
│  └───────────────────────────────────────────────────────┘ │
│                                                              │
│           [Cancel]                    [Enable 2FA]           │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 Login with 2FA
```
┌─────────────────────────────────────────────────────────────┐
│                      SIGN IN PAGE                            │
│                                                              │
│  Email:    [user@example.com        ]                       │
│  Password: [••••••••••              ]                       │
│                                                              │
│           [Sign In]                                          │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼ [2FA Required]
┌─────────────────────────────────────────────────────────────┐
│               TWO-FACTOR AUTHENTICATION                      │
│                                                              │
│                       🔒                                     │
│                                                              │
│        Enter the 6-digit code from your                     │
│              authenticator app                               │
│                                                              │
│              ┌──────────────────────┐                       │
│              │   [0][0][0][0][0][0] │                       │
│              └──────────────────────┘                       │
│                                                              │
│           [Back to login]    [Verify & Sign In]              │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │ Lost access to your authenticator?                  │    │
│  │ Use one of your backup codes or contact support.    │    │
│  └────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. Additional Features to Implement

### 6.1 High Priority 🔴

#### A. Backup Code Login
**Description:** Allow users to login using backup codes when they don't have access to their authenticator app.

**UI Changes Needed:**
- Add "Use backup code instead" link on 2FA login form
- Create backup code input form (single field for 9-character code)

**New File:** `src/components/auth/BackupCodeForm.tsx`

```typescript
// Example structure
interface BackupCodeFormProps {
  email: string;
  password: string;
  onSuccess: () => void;
  onBack: () => void;
}
```

#### B. View/Download Backup Codes
**Description:** Let users view their remaining backup codes from settings.

**UI Changes Needed:**
- Add "View Backup Codes" button in Security Settings
- Modal to display remaining codes (require password verification)
- Option to copy or download codes

**Endpoint Needed:** `GET /auth/2fa/backup` (with password confirmation)

#### C. Regenerate Backup Codes
**Description:** Allow users to generate new backup codes (invalidating old ones).

**UI Changes Needed:**
- Add "Regenerate Backup Codes" button in Security Settings
- Confirmation modal warning about invalidating old codes
- Display new codes with copy/download option

**Endpoint Needed:** `POST /auth/2fa/backup/regenerate`

---

### 6.2 Medium Priority 🟡

#### D. Remember Device (Trust Device)
**Description:** Allow users to skip 2FA on trusted devices for 30 days.

**UI Changes Needed:**
- Add "Remember this device for 30 days" checkbox on 2FA login form
- Show list of trusted devices in Security Settings
- Allow revoking trusted devices

**Endpoints Needed:**
- `GET /auth/2fa/trusted-devices`
- `DELETE /auth/2fa/trusted-devices/:id`
- `DELETE /auth/2fa/trusted-devices` (revoke all)

**Local Storage:**
```typescript
// Store device trust token
localStorage.setItem('device_trust_token', 'token-from-backend');
```

#### E. 2FA Required by Admin
**Description:** Show indicator if 2FA is required by organization policy.

**UI Changes Needed:**
- Warning banner if 2FA is required but not enabled
- Block certain actions until 2FA is enabled
- Redirect to 2FA setup on login if required

**User Model Addition:**
```typescript
interface User {
  // ... existing fields
  two_factor_required: boolean;
  two_factor_deadline?: string; // Date by which 2FA must be enabled
}
```

#### F. 2FA Activity Log
**Description:** Show when 2FA was used (successful verifications and failed attempts).

**UI Changes Needed:**
- Add "2FA Activity" section in Security Settings
- Show list of recent 2FA verifications with timestamp, device, location

**Endpoint Needed:** `GET /auth/2fa/activity`

---

### 6.3 Low Priority 🟢

#### G. SMS/Email Backup (Not Recommended)
**Description:** Fallback to SMS or email code if TOTP unavailable.

**Note:** Not recommended due to security concerns (SIM swapping, email compromise).

#### H. Hardware Security Keys (WebAuthn)
**Description:** Support for FIDO2/WebAuthn hardware keys (YubiKey, etc.)

**Complexity:** High - requires WebAuthn API implementation

#### I. Multiple Authenticator Apps
**Description:** Allow users to register multiple TOTP devices.

**UI Changes Needed:**
- List registered authenticator devices
- Add/remove devices
- Name each device for identification

---

## 7. Security Considerations

### 7.1 Rate Limiting
```
- 2FA code verification: Max 5 attempts per 5 minutes
- Backup code usage: Max 3 attempts per 15 minutes
- After exceeded: Temporary account lock (15-30 minutes)
```

### 7.2 Code Expiration
```
- TOTP codes valid for current + 1 previous time window (~60 seconds total)
- Backup codes never expire but are one-time use
- Temp tokens for 2FA login expire after 5 minutes
```

### 7.3 Logging
```
Log all 2FA events:
- 2FA enabled/disabled
- Successful/failed verifications
- Backup code usage
- Trusted device additions/revocations
```

### 7.4 Recovery Process
```
If user loses access to authenticator AND backup codes:
1. Identity verification required (support ticket)
2. Admin manually disables 2FA after verification
3. User must immediately set up new 2FA
4. Notify all user's email addresses about 2FA change
```

---

## 8. Implementation Checklist

### Phase 1: Core 2FA (✅ Completed)
- [x] Enable 2FA with QR code
- [x] Verify TOTP code
- [x] Disable 2FA with password confirmation
- [x] 2FA status display
- [x] 2FA login flow
- [x] Backup codes display on setup

### Phase 2: Backup Codes Enhancement
- [ ] Backup code login flow
- [ ] View remaining backup codes
- [ ] Regenerate backup codes
- [ ] Download backup codes as file

### Phase 3: Device Trust
- [ ] Remember device checkbox
- [ ] Trusted devices list
- [ ] Revoke trusted devices

### Phase 4: Admin Features
- [ ] Require 2FA policy
- [ ] 2FA activity log
- [ ] Bulk 2FA status management

---

## 9. Testing Checklist

### Enable 2FA
- [ ] QR code displays correctly
- [ ] Manual secret code works
- [ ] Backup codes are shown
- [ ] Invalid TOTP code shows error
- [ ] Valid TOTP code enables 2FA
- [ ] Status updates after enabling

### Login with 2FA
- [ ] 2FA form appears after password login
- [ ] Invalid code shows error
- [ ] Valid code completes login
- [ ] Back button returns to login form
- [ ] Rate limiting works after failed attempts

### Disable 2FA
- [ ] Modal requires password + code
- [ ] Invalid credentials show error
- [ ] Valid credentials disable 2FA
- [ ] Status updates after disabling

### Edge Cases
- [ ] Session timeout during 2FA setup
- [ ] Network error during verification
- [ ] Concurrent login attempts
- [ ] Browser refresh during 2FA flow

---

*Last Updated: 2026-01-02*
*Frontend Version: NexFlux v1.0*

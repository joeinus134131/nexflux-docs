# ACL (Access Control List) Backend Requirements

Dokumen ini menjelaskan kebutuhan backend untuk sistem Access Control List (ACL) pada platform NexFlux. ACL mengatur akses fitur berdasarkan **Role User**, **Level Gamifikasi**, dan **Status Sesi Hardware**.

---

## 📋 Table of Contents

1. [Overview](#overview)
2. [Database Schema](#database-schema)
3. [API Endpoints](#api-endpoints)
4. [Access Rules](#access-rules)
5. [Middleware Implementation](#middleware-implementation)
6. [Environment Variables](#environment-variables)
7. [Testing Scenarios](#testing-scenarios)

---

## Overview

### Tujuan ACL
Sistem ACL bertujuan untuk:
1. **Membatasi akses fitur** berdasarkan subscription tier (Free vs Pro)
2. **Gating konten** berdasarkan level gamifikasi user
3. **Mengontrol hardware access** berdasarkan status sesi booking
4. **Logging audit trail** untuk security compliance

### Komponen Utama
- **User Roles**: `user_free`, `user_pro`, `admin`
- **User Levels**: Integer 1-100 (gamification)
- **Session Status**: `ACTIVE`, `EXPIRED`, `PENDING`, `CANCELLED`

---

## Database Schema

### Table: `profiles` (Update existing)
```sql
-- Add/update columns for ACL support
ALTER TABLE profiles ADD COLUMN IF NOT EXISTS role VARCHAR(20) DEFAULT 'user_free';
ALTER TABLE profiles ADD COLUMN IF NOT EXISTS level INTEGER DEFAULT 1;
ALTER TABLE profiles ADD COLUMN IF NOT EXISTS xp INTEGER DEFAULT 0;
ALTER TABLE profiles ADD COLUMN IF NOT EXISTS xp_to_next_level INTEGER DEFAULT 100;
ALTER TABLE profiles ADD COLUMN IF NOT EXISTS streak INTEGER DEFAULT 0;

-- Add constraint for valid roles
ALTER TABLE profiles ADD CONSTRAINT valid_role 
  CHECK (role IN ('user_free', 'user_pro', 'admin'));

-- Index for quick role lookups
CREATE INDEX IF NOT EXISTS idx_profiles_role ON profiles(role);
CREATE INDEX IF NOT EXISTS idx_profiles_level ON profiles(level);
```

### Table: `sessions` (Hardware Booking Sessions)
```sql
CREATE TABLE IF NOT EXISTS hardware_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lab_id UUID REFERENCES labs(id) NOT NULL,
    user_id UUID REFERENCES profiles(id) NOT NULL,
    device_id VARCHAR(100) NOT NULL,
    status VARCHAR(20) DEFAULT 'PENDING',
    started_at TIMESTAMPTZ DEFAULT NOW(),
    expires_at TIMESTAMPTZ NOT NULL,
    duration_seconds INTEGER DEFAULT 1800, -- 30 minutes default
    
    -- Metadata
    ip_address INET,
    user_agent TEXT,
    livekit_room VARCHAR(100),
    
    -- Timestamps
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    
    -- Constraints
    CONSTRAINT valid_session_status 
      CHECK (status IN ('ACTIVE', 'EXPIRED', 'PENDING', 'CANCELLED'))
);

-- Indexes for quick lookups
CREATE INDEX IF NOT EXISTS idx_sessions_user ON hardware_sessions(user_id);
CREATE INDEX IF NOT EXISTS idx_sessions_lab ON hardware_sessions(lab_id);
CREATE INDEX IF NOT EXISTS idx_sessions_status ON hardware_sessions(status);
CREATE INDEX IF NOT EXISTS idx_sessions_expires ON hardware_sessions(expires_at);

-- Ensure only one active session per user
CREATE UNIQUE INDEX IF NOT EXISTS idx_unique_active_session 
  ON hardware_sessions(user_id) 
  WHERE status = 'ACTIVE';
```

### Table: `feature_access_logs` (Audit Trail)
```sql
CREATE TABLE IF NOT EXISTS feature_access_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES profiles(id) NOT NULL,
    feature_id VARCHAR(100) NOT NULL,
    action VARCHAR(50) NOT NULL, -- 'access_granted', 'access_denied', 'upgrade_prompted'
    reason VARCHAR(100), -- 'LEVEL_TOO_LOW', 'PRO_REQUIRED', etc.
    metadata JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_access_logs_user ON feature_access_logs(user_id);
CREATE INDEX IF NOT EXISTS idx_access_logs_feature ON feature_access_logs(feature_id);
CREATE INDEX IF NOT EXISTS idx_access_logs_created ON feature_access_logs(created_at);
```

### Table: `subscription_tiers` (Reference)
```sql
CREATE TABLE IF NOT EXISTS subscription_tiers (
    id VARCHAR(20) PRIMARY KEY, -- 'free', 'pro', 'enterprise'
    name VARCHAR(100) NOT NULL,
    price_monthly DECIMAL(10, 2) DEFAULT 0,
    price_yearly DECIMAL(10, 2) DEFAULT 0,
    features JSONB NOT NULL, -- Array of feature IDs
    max_session_duration INTEGER DEFAULT 1800, -- seconds
    priority_queue BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Seed default tiers
INSERT INTO subscription_tiers (id, name, price_monthly, features, max_session_duration, priority_queue) VALUES
  ('free', 'Free', 0, '["REMOTE_LAB_ACCESS", "CREATE_PROJECTS", "CONTROL_LED", "CONTROL_SERVO"]', 1800, false),
  ('pro', 'Pro', 9.99, '["REMOTE_LAB_ACCESS", "CREATE_PROJECTS", "CONTROL_LED", "CONTROL_SERVO", "CONTROL_MOTOR", "EXTENDED_SESSION", "PRIORITY_QUEUE", "CIRCUIT_STUDIO_PRO", "EMBED_PROJECTS"]', 3600, true)
ON CONFLICT (id) DO NOTHING;
```

---

## API Endpoints

### 1. Check User Access
```
GET /api/v1/access/check
```

**Query Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `feature_id` | string | Yes | Feature to check access for |
| `min_level` | integer | No | Minimum level check (alternative) |

**Response (Success):**
```json
{
    "success": true,
    "data": {
        "allowed": true,
        "user": {
            "id": "uuid",
            "role": "user_pro",
            "level": 5,
            "xp": 450
        },
        "session": {
            "id": "uuid",
            "status": "ACTIVE",
            "expires_at": "2024-01-15T11:00:00Z"
        }
    }
}
```

**Response (Denied):**
```json
{
    "success": false,
    "error": {
        "code": "ACCESS_DENIED",
        "reason": "LEVEL_TOO_LOW",
        "message": "ACCESS DENIED: LEVEL TOO LOW",
        "details": {
            "required_level": 10,
            "current_level": 5
        }
    }
}
```

### 2. Get User Profile with ACL Data
```
GET /api/v1/users/me
```

**Response:**
```json
{
    "success": true,
    "data": {
        "id": "uuid",
        "email": "user@example.com",
        "name": "John Doe",
        "avatar_url": "https://...",
        "role": "user_pro",
        "level": 5,
        "xp": 450,
        "xp_to_next_level": 550,
        "streak": 7,
        "subscription": {
            "tier": "pro",
            "started_at": "2024-01-01T00:00:00Z",
            "expires_at": "2025-01-01T00:00:00Z",
            "is_active": true
        },
        "features": {
            "can_control_hardware": true,
            "can_use_priority_queue": true,
            "max_session_duration": 3600
        }
    }
}
```

### 3. Get Current Session
```
GET /api/v1/sessions/current
```

**Response (Active Session):**
```json
{
    "success": true,
    "data": {
        "id": "uuid",
        "lab_id": "uuid",
        "device_id": "arduino-lab-1",
        "status": "ACTIVE",
        "started_at": "2024-01-15T10:00:00Z",
        "expires_at": "2024-01-15T10:30:00Z",
        "remaining_seconds": 1200
    }
}
```

**Response (No Session):**
```json
{
    "success": true,
    "data": null,
    "message": "No active session"
}
```

### 4. Start Hardware Session
```
POST /api/v1/labs/:lab_id/session/start
```

**Authorization Check:**
1. User must be authenticated
2. User role must be `user_pro` OR `admin` for hardware control
3. No existing active session for this user
4. Lab must be available (status = 'available')

**Response:**
```json
{
    "success": true,
    "data": {
        "session_id": "uuid",
        "lab_id": "uuid",
        "device_id": "arduino-lab-1",
        "status": "ACTIVE",
        "livekit_token": "eyJ...",
        "livekit_url": "wss://livekit.nexflux.io",
        "room_name": "lab-arduino-1-session-xyz",
        "expires_at": "2024-01-15T11:00:00Z",
        "max_duration": 3600
    }
}
```

### 5. End Hardware Session
```
POST /api/v1/labs/:lab_id/session/end
```

**Request Body:**
```json
{
    "session_id": "uuid",
    "feedback": "Great experience!",
    "rating": 5
}
```

### 6. Upgrade to Pro
```
POST /api/v1/subscription/upgrade
```

**Request Body:**
```json
{
    "tier": "pro",
    "payment_method": "stripe",
    "billing_period": "monthly"
}
```

### 7. Add XP to User
```
POST /api/v1/users/xp/add
```

**Request Body:**
```json
{
    "amount": 50,
    "source": "lab_session_complete",
    "metadata": {
        "lab_id": "uuid",
        "session_id": "uuid"
    }
}
```

---

## Access Rules

### Feature Gates Configuration
```json
{
    "CONTROL_LED": {
        "min_level": 1,
        "requires_pro": false,
        "requires_active_session": true
    },
    "CONTROL_SERVO": {
        "min_level": 3,
        "requires_pro": false,
        "requires_active_session": true
    },
    "CONTROL_MOTOR": {
        "min_level": 5,
        "requires_pro": true,
        "requires_active_session": true
    },
    "REMOTE_LAB_ACCESS": {
        "min_level": 1,
        "requires_pro": false,
        "requires_active_session": false
    },
    "EXTENDED_SESSION": {
        "min_level": 5,
        "requires_pro": true,
        "requires_active_session": false
    },
    "PRIORITY_QUEUE": {
        "min_level": 1,
        "requires_pro": true,
        "requires_active_session": false
    },
    "ADVANCED_TUTORIALS": {
        "min_level": 5,
        "requires_pro": false,
        "requires_active_session": false
    },
    "EXPERT_CHALLENGES": {
        "min_level": 10,
        "requires_pro": false,
        "requires_active_session": false
    },
    "CIRCUIT_STUDIO_PRO": {
        "min_level": 3,
        "requires_pro": true,
        "requires_active_session": false
    },
    "CREATE_PROJECTS": {
        "min_level": 2,
        "requires_pro": false,
        "requires_active_session": false
    },
    "EMBED_PROJECTS": {
        "min_level": 5,
        "requires_pro": true,
        "requires_active_session": false
    }
}
```

### Access Check Logic (Pseudocode)
```go
func CheckAccess(user User, featureId string, session *Session) AccessResult {
    gate := GetFeatureGate(featureId)
    if gate == nil {
        return AccessResult{Allowed: true} // Unknown feature = allow
    }
    
    // Check authentication
    if user.ID == "" {
        return AccessResult{
            Allowed: false,
            Reason:  "NOT_AUTHENTICATED",
        }
    }
    
    // Check level requirement
    if user.Level < gate.MinLevel {
        return AccessResult{
            Allowed:       false,
            Reason:        "LEVEL_TOO_LOW",
            RequiredLevel: gate.MinLevel,
            CurrentLevel:  user.Level,
        }
    }
    
    // Check Pro requirement
    if gate.RequiresPro && user.Role != "user_pro" && user.Role != "admin" {
        return AccessResult{
            Allowed: false,
            Reason:  "PRO_REQUIRED",
        }
    }
    
    // Check active session requirement
    if gate.RequiresActiveSession {
        if session == nil {
            return AccessResult{
                Allowed: false,
                Reason:  "SESSION_NOT_FOUND",
            }
        }
        if session.Status != "ACTIVE" || session.ExpiresAt.Before(time.Now()) {
            return AccessResult{
                Allowed: false,
                Reason:  "SESSION_EXPIRED",
            }
        }
    }
    
    return AccessResult{Allowed: true}
}
```

---

## Middleware Implementation

### Go/Fiber ACL Middleware
```go
package middleware

import (
    "github.com/gofiber/fiber/v2"
)

type ACLConfig struct {
    FeatureID            string
    MinLevel             int
    RequiresPro          bool
    RequiresActiveSession bool
}

func RequireAccess(config ACLConfig) fiber.Handler {
    return func(c *fiber.Ctx) error {
        // Get user from context (set by auth middleware)
        user := GetUserFromContext(c)
        if user == nil {
            return c.Status(401).JSON(fiber.Map{
                "success": false,
                "error": fiber.Map{
                    "code":    "UNAUTHORIZED",
                    "reason":  "NOT_AUTHENTICATED",
                    "message": "ACCESS DENIED: LOGIN REQUIRED",
                },
            })
        }
        
        // Check level
        if config.MinLevel > 0 && user.Level < config.MinLevel {
            LogAccessDenied(user.ID, config.FeatureID, "LEVEL_TOO_LOW")
            return c.Status(403).JSON(fiber.Map{
                "success": false,
                "error": fiber.Map{
                    "code":    "FORBIDDEN",
                    "reason":  "LEVEL_TOO_LOW",
                    "message": "ACCESS DENIED: LEVEL TOO LOW",
                    "details": fiber.Map{
                        "required_level": config.MinLevel,
                        "current_level":  user.Level,
                    },
                },
            })
        }
        
        // Check Pro requirement
        if config.RequiresPro && user.Role != "user_pro" && user.Role != "admin" {
            LogAccessDenied(user.ID, config.FeatureID, "PRO_REQUIRED")
            return c.Status(403).JSON(fiber.Map{
                "success": false,
                "error": fiber.Map{
                    "code":    "FORBIDDEN",
                    "reason":  "PRO_REQUIRED",
                    "message": "ACCESS DENIED: PRO REQUIRED",
                },
            })
        }
        
        // Check active session
        if config.RequiresActiveSession {
            session := GetActiveSession(user.ID)
            if session == nil {
                LogAccessDenied(user.ID, config.FeatureID, "SESSION_NOT_FOUND")
                return c.Status(403).JSON(fiber.Map{
                    "success": false,
                    "error": fiber.Map{
                        "code":    "FORBIDDEN",
                        "reason":  "SESSION_NOT_FOUND",
                        "message": "ACCESS DENIED: NO ACTIVE SESSION",
                    },
                })
            }
            if session.IsExpired() {
                LogAccessDenied(user.ID, config.FeatureID, "SESSION_EXPIRED")
                return c.Status(403).JSON(fiber.Map{
                    "success": false,
                    "error": fiber.Map{
                        "code":    "FORBIDDEN",
                        "reason":  "SESSION_EXPIRED",
                        "message": "ACCESS DENIED: SESSION EXPIRED",
                    },
                })
            }
            
            // Add session to context for handlers
            c.Locals("session", session)
        }
        
        // Log successful access
        LogAccessGranted(user.ID, config.FeatureID)
        
        return c.Next()
    }
}

// Usage example:
// app.Post("/labs/:id/actuators/control", 
//     RequireAccess(ACLConfig{
//         FeatureID:            "CONTROL_LED",
//         MinLevel:             1,
//         RequiresPro:          false,
//         RequiresActiveSession: true,
//     }),
//     handlers.ControlActuator,
// )
```

---

## Environment Variables

```env
# ACL Configuration
ACL_ENABLED=true
ACL_DEFAULT_ROLE=user_free
ACL_DEFAULT_LEVEL=1

# Session Configuration
SESSION_DEFAULT_DURATION=1800
SESSION_PRO_MAX_DURATION=3600
SESSION_ADMIN_MAX_DURATION=7200

# XP & Gamification
XP_PER_SESSION_COMPLETE=50
XP_PER_CODE_UPLOAD=20
XP_PER_CHALLENGE_COMPLETE=100
XP_LEVEL_MULTIPLIER=1.5

# Feature Gate Cache
FEATURE_GATE_CACHE_TTL=300
```

---

## Testing Scenarios

### Test Cases

| Scenario | User Role | Level | Session | Expected Result |
|----------|-----------|-------|---------|-----------------|
| LED Control - No Session | user_pro | 5 | null | DENIED: SESSION_NOT_FOUND |
| LED Control - Active Session | user_pro | 5 | ACTIVE | ALLOWED |
| LED Control - Expired Session | user_pro | 5 | EXPIRED | DENIED: SESSION_EXPIRED |
| Motor Control - Free User | user_free | 10 | ACTIVE | DENIED: PRO_REQUIRED |
| Motor Control - Pro, Low Level | user_pro | 3 | ACTIVE | DENIED: LEVEL_TOO_LOW |
| Motor Control - Pro, High Level | user_pro | 7 | ACTIVE | ALLOWED |
| Expert Challenges - Free L5 | user_free | 5 | - | DENIED: LEVEL_TOO_LOW |
| Expert Challenges - Free L10 | user_free | 10 | - | ALLOWED |
| Circuit Studio Pro - Free | user_free | 5 | - | DENIED: PRO_REQUIRED |
| Circuit Studio Pro - Pro L2 | user_pro | 2 | - | DENIED: LEVEL_TOO_LOW |
| Circuit Studio Pro - Pro L3 | user_pro | 3 | - | ALLOWED |

### cURL Test Commands
```bash
# Check access for a feature
curl -X GET "http://localhost:8005/api/v1/access/check?feature_id=CONTROL_LED" \
  -H "Authorization: Bearer <token>"

# Get user profile with ACL data
curl -X GET "http://localhost:8005/api/v1/users/me" \
  -H "Authorization: Bearer <token>"

# Get current session
curl -X GET "http://localhost:8005/api/v1/sessions/current" \
  -H "Authorization: Bearer <token>"

# Start lab session
curl -X POST "http://localhost:8005/api/v1/labs/<lab_id>/session/start" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json"

# Control actuator (requires active session)
curl -X POST "http://localhost:8005/api/v1/labs/<lab_id>/actuators/control" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"actuator": "LED", "action": "on"}'
```

---

## Security Considerations

1. **Token Validation**: Always validate JWT tokens before checking ACL
2. **Session Binding**: Sessions are bound to user + device + IP
3. **Rate Limiting**: Apply rate limits per role tier
4. **Audit Logging**: Log all access attempts (granted and denied)
5. **Time-based Expiry**: Sessions auto-expire, cron job cleans up stale sessions
6. **Privilege Escalation**: Admin actions require separate auth flow
7. **Input Validation**: Sanitize all feature IDs and parameters

---

## Migration Notes

### From Existing System
1. Add `role`, `level`, `xp` columns to existing `profiles` table
2. Create `hardware_sessions` table
3. Create `feature_access_logs` table
4. Seed `subscription_tiers` with default values
5. Update `/api/v1/users/me` endpoint to include ACL data
6. Add ACL middleware to protected routes
7. Update LiveKit token generation to check Pro status for extended duration

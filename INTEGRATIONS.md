# NexFlux Integrations Specification

This document outlines the integrations feature for NexFlux platform.

## Overview

The Integrations feature allows users to connect their NexFlux account with external services for enhanced functionality such as:
- IoT device communication (MQTT)
- Code repository sync (GitHub)
- Cloud IoT platforms (Arduino Cloud, Blynk, ThingsBoard)
- OAuth login providers (Google, GitHub)
- Export capabilities (Google Drive, OneDrive)

---

## 1. MQTT/IoT Device Integration

**Priority: CRITICAL**

This integration allows users to configure their own MQTT broker for Remote Lab functionality.

### Use Cases
- Connect to institutional MQTT brokers
- Configure personal IoT device communication
- Custom device topics

### Frontend Requirements
- Form to input MQTT broker URL (WebSocket)
- Username/Password authentication (optional)
- Test connection button
- Device ID configuration
- Topic prefix customization

### Backend API

#### GET `/api/v1/users/me/integrations/mqtt`
Returns user's MQTT configuration.

**Response:**
```json
{
  "success": true,
  "data": {
    "enabled": true,
    "broker_url": "ws://broker.example.com:9001",
    "username": "user123",
    "use_authentication": true,
    "device_id": "device-01",
    "topic_prefix": "nexflux/device",
    "last_connected_at": "2026-01-15T10:30:00Z",
    "status": "connected"
  }
}
```

#### PUT `/api/v1/users/me/integrations/mqtt`
Update MQTT configuration.

**Request:**
```json
{
  "enabled": true,
  "broker_url": "ws://broker.example.com:9001",
  "username": "user123",
  "password": "secret123",
  "use_authentication": true,
  "device_id": "device-01",
  "topic_prefix": "nexflux/device"
}
```

#### POST `/api/v1/users/me/integrations/mqtt/test`
Test MQTT connection.

**Response:**
```json
{
  "success": true,
  "data": {
    "connected": true,
    "latency_ms": 45,
    "broker_info": {
      "version": "mosquitto/2.0.15",
      "protocol": "mqtt/5.0"
    }
  }
}
```

#### DELETE `/api/v1/users/me/integrations/mqtt`
Disconnect and remove MQTT configuration.

---

## 2. GitHub Integration

**Priority: HIGH**

OAuth-based GitHub integration for syncing projects.

### Use Cases
- Backup circuits/projects to GitHub
- Version control for projects
- Share projects via GitHub
- Login with GitHub

### Frontend Requirements
- OAuth connect button (redirects to GitHub)
- Repository selection
- Auto-sync toggle
- Sync status display

### Backend API

#### GET `/api/v1/users/me/integrations/github`
Returns GitHub integration status.

**Response:**
```json
{
  "success": true,
  "data": {
    "connected": true,
    "username": "johndoe",
    "avatar_url": "https://github.com/johndoe.png",
    "email": "john@example.com",
    "connected_at": "2026-01-10T08:00:00Z",
    "repositories": [
      {
        "id": 12345,
        "name": "my-circuits",
        "full_name": "johndoe/my-circuits",
        "private": false,
        "synced": true,
        "last_sync": "2026-01-15T09:00:00Z"
      }
    ],
    "auto_sync": true,
    "scopes": ["repo", "read:user"]
  }
}
```

#### GET `/api/v1/auth/github/connect`
Initiates GitHub OAuth flow.

**Response:** Redirect to GitHub authorization page.

#### GET `/api/v1/auth/github/callback`
GitHub OAuth callback handler.

#### POST `/api/v1/users/me/integrations/github/sync`
Trigger manual sync to GitHub.

**Request:**
```json
{
  "repository_id": 12345,
  "project_ids": ["uuid-1", "uuid-2"]
}
```

#### DELETE `/api/v1/users/me/integrations/github`
Disconnect GitHub integration.

---

## 3. Arduino Cloud Integration

**Priority: HIGH**

Integration with Arduino Cloud for IoT devices.

### Use Cases
- Sync Arduino projects
- Deploy code to Arduino Cloud devices
- Monitor device status from NexFlux

### Frontend Requirements
- OAuth/API key connection
- Device list display
- Sync status

### Backend API

#### GET `/api/v1/users/me/integrations/arduino-cloud`
Returns Arduino Cloud integration status.

**Response:**
```json
{
  "success": true,
  "data": {
    "connected": true,
    "username": "johndoe",
    "organization": "Personal",
    "connected_at": "2026-01-12T14:00:00Z",
    "devices": [
      {
        "id": "abc123",
        "name": "ESP32 Weather Station",
        "type": "ESP32",
        "status": "online",
        "last_activity": "2026-01-15T10:00:00Z"
      }
    ],
    "things": [
      {
        "id": "thing-123",
        "name": "Weather Monitor",
        "device_id": "abc123"
      }
    ]
  }
}
```

#### PUT `/api/v1/users/me/integrations/arduino-cloud`
Configure Arduino Cloud integration.

**Request:**
```json
{
  "client_id": "xxxxx",
  "client_secret": "yyyyy"
}
```

---

## 4. Google OAuth Integration

**Priority: MEDIUM**

Login and Google Drive integration.

### Use Cases
- Sign in with Google
- Export projects to Google Drive
- Import from Google Drive

### Backend API

#### GET `/api/v1/auth/google/connect`
Initiates Google OAuth flow.

#### GET `/api/v1/auth/google/callback`
Google OAuth callback.

#### GET `/api/v1/users/me/integrations/google`
Returns Google integration status.

---

## 5. Blynk Integration

**Priority: MEDIUM**

Integration for Blynk IoT platform.

### Use Cases
- Connect Blynk devices
- Monitor device dashboards
- Send commands to Blynk devices

### Backend API

Similar pattern to Arduino Cloud.

---

## Database Schema

### Table: `user_integrations`

```sql
CREATE TABLE user_integrations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    provider VARCHAR(50) NOT NULL, -- 'mqtt', 'github', 'arduino_cloud', 'google', 'blynk'
    enabled BOOLEAN DEFAULT true,
    config JSONB NOT NULL DEFAULT '{}',
    access_token TEXT, -- encrypted
    refresh_token TEXT, -- encrypted
    token_expires_at TIMESTAMP,
    metadata JSONB DEFAULT '{}',
    connected_at TIMESTAMP DEFAULT NOW(),
    last_used_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(user_id, provider)
);

CREATE INDEX idx_user_integrations_user_id ON user_integrations(user_id);
CREATE INDEX idx_user_integrations_provider ON user_integrations(provider);
```

### Example Config JSON by Provider

**MQTT:**
```json
{
  "broker_url": "ws://broker.example.com:9001",
  "username": "user123",
  "password_encrypted": "...",
  "use_authentication": true,
  "device_id": "device-01",
  "topic_prefix": "nexflux/device"
}
```

**GitHub:**
```json
{
  "username": "johndoe",
  "avatar_url": "...",
  "email": "...",
  "scopes": ["repo", "read:user"],
  "auto_sync": true,
  "selected_repos": [12345, 67890]
}
```

---

## Environment Variables (Backend)

```env
# GitHub OAuth
GITHUB_CLIENT_ID=your_github_client_id
GITHUB_CLIENT_SECRET=your_github_client_secret
GITHUB_CALLBACK_URL=http://localhost:8005/api/v1/auth/github/callback

# Google OAuth
GOOGLE_CLIENT_ID=your_google_client_id
GOOGLE_CLIENT_SECRET=your_google_client_secret
GOOGLE_CALLBACK_URL=http://localhost:8005/api/v1/auth/google/callback

# Arduino Cloud
ARDUINO_CLIENT_ID=your_arduino_client_id
ARDUINO_CLIENT_SECRET=your_arduino_client_secret

# MQTT Default Broker (fallback for users without custom config)
DEFAULT_MQTT_BROKER_URL=ws://212.85.24.191:9001
```

---

## Implementation Priority

1. **Phase 1 (Week 1):** MQTT Integration - Full implementation
2. **Phase 2 (Week 2):** GitHub Integration - OAuth + basic sync
3. **Phase 3 (Week 3):** Arduino Cloud Integration
4. **Phase 4 (Week 4):** Google/Blynk Integration

---

## Security Considerations

1. **Token Encryption:** All OAuth tokens and passwords must be encrypted at rest
2. **Token Refresh:** Implement automatic token refresh before expiry
3. **Scope Limits:** Request minimum necessary OAuth scopes
4. **Revocation:** Users can revoke integrations anytime
5. **Audit Logging:** Log all integration connect/disconnect events

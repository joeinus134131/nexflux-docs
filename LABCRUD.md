# Lab Management - Backend CRUD Requirements

This document summarizes the backend API requirements for the Lab Management feature in the NexFlux admin panel.

---

## 📋 Overview

The Lab Management feature allows administrators to perform CRUD (Create, Read, Update, Delete) operations on remote lab configurations. This includes managing lab hardware specifications, device connections, sensors, actuators, and session configurations.

---

## 🗃️ Database Schema

### Labs Table

```sql
CREATE TABLE labs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) UNIQUE NOT NULL,
    description TEXT,
    platform VARCHAR(50) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'offline',
    device_id VARCHAR(100),
    thumbnail_url TEXT,
    max_session_duration INTEGER NOT NULL DEFAULT 30,
    total_sessions INTEGER DEFAULT 0,
    queue_count INTEGER DEFAULT 0,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Status ENUM: 'available', 'busy', 'maintenance', 'offline'
-- Platform ENUM: 'arduino_uno', 'arduino_mega', 'esp32', 'esp8266', 'raspberry_pi', 'stm32'
```

### Lab Hardware Specs Table

```sql
CREATE TABLE lab_hardware_specs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lab_id UUID NOT NULL REFERENCES labs(id) ON DELETE CASCADE,
    camera VARCHAR(255),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

### Lab Sensors Table

```sql
CREATE TABLE lab_sensors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lab_id UUID NOT NULL REFERENCES labs(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    type VARCHAR(50), -- e.g., 'temperature', 'humidity', 'motion', 'light'
    model VARCHAR(100), -- e.g., 'DHT22', 'LDR', 'PIR'
    description TEXT,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_lab_sensors_lab_id ON lab_sensors(lab_id);
```

### Lab Actuators Table

```sql
CREATE TABLE lab_actuators (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lab_id UUID NOT NULL REFERENCES labs(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    type VARCHAR(50), -- e.g., 'led', 'servo', 'relay', 'motor', 'display'
    model VARCHAR(100), -- e.g., 'SG90', 'OLED 0.96"'
    description TEXT,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_lab_actuators_lab_id ON lab_actuators(lab_id);
```

---

## 🔌 API Endpoints

### 1. List All Labs (Admin)

**Endpoint:** `GET /api/v1/admin/labs`

**Description:** Retrieve all labs for admin management.

**Authentication:** Required (Admin role)

**Query Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `search` | string | Optional. Search by name, platform, or device_id |
| `status` | string | Optional. Filter by status |
| `platform` | string | Optional. Filter by platform |
| `page` | int | Optional. Page number (default: 1) |
| `limit` | int | Optional. Items per page (default: 20) |

**Response:**
```json
{
    "success": true,
    "data": {
        "labs": [
            {
                "id": "uuid",
                "name": "Arduino Lab #1",
                "slug": "arduino-lab-1",
                "description": "Basic Arduino setup for learning",
                "platform": "arduino_uno",
                "status": "available",
                "device_id": "device-01",
                "thumbnail_url": "https://example.com/image.jpg",
                "max_session_duration": 30,
                "total_sessions": 1250,
                "queue_count": 0,
                "hardware_specs": {
                    "camera": "USB 1080p",
                    "sensors": ["DHT22", "LDR", "Ultrasonic"],
                    "actuators": ["RGB LED", "Servo", "Buzzer"]
                },
                "created_at": "2025-01-01T00:00:00Z",
                "updated_at": "2025-01-01T00:00:00Z"
            }
        ],
        "total": 10,
        "page": 1,
        "limit": 20
    }
}
```

---

### 2. Get Single Lab (Admin)

**Endpoint:** `GET /api/v1/admin/labs/:id`

**Description:** Retrieve detailed information about a specific lab.

**Authentication:** Required (Admin role)

**Response:**
```json
{
    "success": true,
    "data": {
        "id": "uuid",
        "name": "Arduino Lab #1",
        "slug": "arduino-lab-1",
        "description": "Basic Arduino setup for learning",
        "platform": "arduino_uno",
        "status": "available",
        "device_id": "device-01",
        "thumbnail_url": "https://example.com/image.jpg",
        "max_session_duration": 30,
        "total_sessions": 1250,
        "queue_count": 0,
        "hardware_specs": {
            "camera": "USB 1080p",
            "sensors": [
                {"id": "uuid", "name": "DHT22", "type": "temperature", "is_active": true},
                {"id": "uuid", "name": "LDR", "type": "light", "is_active": true}
            ],
            "actuators": [
                {"id": "uuid", "name": "RGB LED", "type": "led", "is_active": true},
                {"id": "uuid", "name": "Servo", "type": "servo", "is_active": true}
            ]
        },
        "created_at": "2025-01-01T00:00:00Z",
        "updated_at": "2025-01-01T00:00:00Z"
    }
}
```

---

### 3. Create Lab

**Endpoint:** `POST /api/v1/admin/labs`

**Description:** Create a new lab configuration.

**Authentication:** Required (Admin role)

**Request Body:**
```json
{
    "name": "Arduino Lab #1",
    "slug": "arduino-lab-1",
    "description": "Basic Arduino setup for learning",
    "platform": "arduino_uno",
    "status": "offline",
    "device_id": "device-01",
    "thumbnail_url": "https://example.com/image.jpg",
    "max_session_duration": 30,
    "hardware_specs": {
        "camera": "USB 1080p",
        "sensors": ["DHT22", "LDR", "Ultrasonic"],
        "actuators": ["RGB LED", "Servo", "Buzzer"]
    }
}
```

**Validation Rules:**
| Field | Rule |
|-------|------|
| `name` | Required, max 255 characters |
| `slug` | Optional (auto-generated if not provided), unique, lowercase with hyphens |
| `platform` | Required, must be one of: arduino_uno, arduino_mega, esp32, esp8266, raspberry_pi, stm32 |
| `status` | Required, must be one of: available, busy, maintenance, offline |
| `max_session_duration` | Required, integer between 5-120 minutes |

**Response:**
```json
{
    "success": true,
    "data": {
        "id": "uuid",
        "name": "Arduino Lab #1",
        ...
    },
    "message": "Lab created successfully"
}
```

---

### 4. Update Lab

**Endpoint:** `PUT /api/v1/admin/labs/:id`

**Description:** Update an existing lab configuration.

**Authentication:** Required (Admin role)

**Request Body:**
```json
{
    "name": "Arduino Lab #1 Updated",
    "slug": "arduino-lab-1",
    "description": "Updated description",
    "platform": "arduino_uno",
    "status": "available",
    "device_id": "device-01",
    "thumbnail_url": "https://example.com/new-image.jpg",
    "max_session_duration": 45,
    "hardware_specs": {
        "camera": "USB 1080p HD",
        "sensors": ["DHT22", "LDR", "Ultrasonic", "PIR Motion"],
        "actuators": ["RGB LED", "Servo", "Buzzer", "Relay"]
    }
}
```

**Response:**
```json
{
    "success": true,
    "data": {
        "id": "uuid",
        "name": "Arduino Lab #1 Updated",
        ...
    },
    "message": "Lab updated successfully"
}
```

---

### 5. Delete Lab

**Endpoint:** `DELETE /api/v1/admin/labs/:id`

**Description:** Delete a lab and all associated data (sensors, actuators, sessions).

**Authentication:** Required (Admin role)

**Response:**
```json
{
    "success": true,
    "message": "Lab deleted successfully"
}
```

**Notes:**
- This should cascade delete all related records (hardware_specs, sensors, actuators)
- Consider archiving instead of hard delete for audit purposes
- Should not allow deletion if lab has active sessions

---

### 6. Update Lab Status

**Endpoint:** `PATCH /api/v1/admin/labs/:id/status`

**Description:** Quick endpoint to update only the lab status.

**Authentication:** Required (Admin role)

**Request Body:**
```json
{
    "status": "maintenance"
}
```

**Response:**
```json
{
    "success": true,
    "data": {
        "id": "uuid",
        "status": "maintenance"
    },
    "message": "Lab status updated successfully"
}
```

---

## 🔧 Additional Endpoints for Hardware Management

### 7. Add Sensor to Lab

**Endpoint:** `POST /api/v1/admin/labs/:id/sensors`

**Request Body:**
```json
{
    "name": "DHT22",
    "type": "temperature",
    "model": "DHT22 Temperature & Humidity",
    "description": "Digital temperature and humidity sensor"
}
```

---

### 8. Remove Sensor from Lab

**Endpoint:** `DELETE /api/v1/admin/labs/:id/sensors/:sensorId`

---

### 9. Add Actuator to Lab

**Endpoint:** `POST /api/v1/admin/labs/:id/actuators`

**Request Body:**
```json
{
    "name": "Servo Motor",
    "type": "servo",
    "model": "SG90",
    "description": "180-degree micro servo"
}
```

---

### 10. Remove Actuator from Lab

**Endpoint:** `DELETE /api/v1/admin/labs/:id/actuators/:actuatorId`

---

## 📊 Statistics Endpoint

### Lab Statistics (Admin Dashboard)

**Endpoint:** `GET /api/v1/admin/labs/stats`

**Description:** Get aggregated statistics for all labs.

**Response:**
```json
{
    "success": true,
    "data": {
        "total_labs": 10,
        "by_status": {
            "available": 5,
            "busy": 2,
            "maintenance": 1,
            "offline": 2
        },
        "by_platform": {
            "arduino_uno": 4,
            "esp32": 3,
            "raspberry_pi": 2,
            "stm32": 1
        },
        "total_sessions_today": 45,
        "total_sessions_week": 312,
        "most_popular_lab": {
            "id": "uuid",
            "name": "ESP32 Lab #1",
            "sessions": 890
        }
    }
}
```

---

## 🔐 Authorization

All admin endpoints require:
1. Valid JWT token in Authorization header
2. User role must be `admin`

**Middleware Example (Go):**
```go
func AdminMiddleware() fiber.Handler {
    return func(c *fiber.Ctx) error {
        user := c.Locals("user").(*models.User)
        if user.Role != "admin" {
            return c.Status(403).JSON(fiber.Map{
                "success": false,
                "error": "Admin access required",
            })
        }
        return c.Next()
    }
}
```

---

## 📝 Implementation Checklist

### Backend Tasks

- [x] **Database Migrations**
  - [x] Create `labs` table (via GORM AutoMigrate)
  - [x] Hardware specs stored as JSONB in `labs.hardware_specs` column
  - [x] Sensors/Actuators embedded in hardware_specs JSONB
  - [x] Add indexes for performance (status, platform, slug, etc.)

- [x] **Repository Layer** (`api/repositories/lab.repository.go`)
  - [x] `Create(lab *Lab) error`
  - [x] `FindByID(id string) (*Lab, error)`
  - [x] `FindBySlug(slug string) (*Lab, error)`
  - [x] `FindAllAdmin(filter LabFilter, page, limit) ([]*Lab, int64, error)`
  - [x] `Update(lab *Lab) error`
  - [x] `Delete(id string) error`
  - [x] `UpdateStatus(id string, status LabStatus) error`
  - [x] `GetStats() (totalLabs, byStatus, byPlatform, error)`
  - [x] `GetMostPopularLab() (*Lab, error)`
  - [x] `CheckSlugExists(slug, excludeID) (bool, error)`

- [x] **Service Layer** (`api/services/lab.service.go`)
  - [x] Input validation
  - [x] Slug auto-generation (if not provided)
  - [x] Admin CRUD methods: CreateLab, UpdateLab, DeleteLab, UpdateLabStatus
  - [x] AdminListLabs, AdminGetLab
  - [x] GetLabStats for admin dashboard
  - [x] Business logic (can't delete lab with active sessions)
  - [x] Cache invalidation

- [x] **Handler Layer** (`api/handlers/admin.lab.handler.go`)
  - [x] GET /admin/labs - List all labs
  - [x] GET /admin/labs/:id - Get single lab
  - [x] POST /admin/labs - Create lab
  - [x] PUT /admin/labs/:id - Update lab
  - [x] DELETE /admin/labs/:id - Delete lab
  - [x] PATCH /admin/labs/:id/status - Update status only
  - [x] GET /admin/labs/stats - Get lab statistics

- [x] **Routes** (`api/routes/routes.go`)
  - [x] Register all routes under `/api/v1/admin/labs`
  - [x] Apply admin middleware (RequireAdmin)

- [ ] **MQTT Integration** (existing implementation)
  - [x] Sync lab status with device heartbeat (via `UpdateHeartbeat`)
  - [x] Auto-set offline if device disconnects (via `handleDeviceDisconnect`)


---

## 🔄 Data Sync with MQTT

When a device connects or disconnects, the lab status should be automatically updated:

```go
// On MQTT connection
func handleDeviceConnect(deviceID string) {
    lab, _ := labRepo.GetLabByDeviceID(deviceID)
    if lab != nil && lab.Status == "offline" {
        labRepo.UpdateLabStatus(lab.ID, "available")
    }
}

// On MQTT disconnection
func handleDeviceDisconnect(deviceID string) {
    lab, _ := labRepo.GetLabByDeviceID(deviceID)
    if lab != nil {
        labRepo.UpdateLabStatus(lab.ID, "offline")
    }
}
```

---

## 🚀 Frontend Integration

The frontend Lab Management page (`/admin/labs`) is already implemented at:
```
src/pages/Admin/Labs/index.tsx
```

It calls the following endpoints:
- `GET /admin/labs` - List labs
- `POST /admin/labs` - Create lab
- `PUT /admin/labs/:id` - Update lab
- `DELETE /admin/labs/:id` - Delete lab

---

## 📌 Notes

1. **Caching:** Consider caching lab list for public endpoints
2. **Rate Limiting:** Apply rate limiting to prevent abuse
3. **Audit Log:** Log all admin actions for accountability
4. **Soft Delete:** Consider implementing soft delete instead of hard delete
5. **Validation:** Always validate device_id format and uniqueness
6. **WebSocket Updates:** Push real-time updates when lab status changes

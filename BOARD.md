# Board Management - Backend CRUD Requirements

This document specifies the backend API requirements for the **Board Management** feature in the NexFlux admin panel. Boards are customizable hardware configurations that can be assigned to virtual labs.

> ⏳ **Status: Pending Implementation**
> - Backend: Not yet implemented
> - Frontend Admin: `/admin/boards` - Pending
> - Integration: Pending

---

## 📋 Overview

Board Management allows administrators to:
1. **Create board templates** - Define reusable hardware configurations
2. **Manage board inventory** - Track available boards and their capabilities
3. **Assign boards to labs** - Link boards to specific virtual lab instances
4. **Configure GPIO pins** - Define pin mappings for sensors and actuators
5. **Track board health** - Monitor connectivity and performance

---

## 🔧 Board Types (Platforms)

| Platform Code | Name | Description |
|--------------|------|-------------|
| `arduino_uno` | Arduino UNO | ATmega328P, 14 digital I/O, 6 analog inputs |
| `arduino_mega` | Arduino MEGA | ATmega2560, 54 digital I/O, 16 analog inputs |
| `arduino_nano` | Arduino Nano | ATmega328P, compact form factor |
| `esp32` | ESP32 | Dual-core, WiFi + Bluetooth, 34 GPIO |
| `esp8266` | ESP8266 | Single-core, WiFi, 17 GPIO |
| `raspberry_pi_pico` | Raspberry Pi Pico | RP2040, 26 GPIO |
| `raspberry_pi_4` | Raspberry Pi 4 | Full Linux SBC, 40 GPIO |
| `stm32` | STM32 | ARM Cortex-M, various models |
| `custom` | Custom Board | User-defined configuration |

---

## 🗃️ Database Schema

### 1. Boards Table (Main)

```sql
CREATE TABLE boards (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) UNIQUE NOT NULL,
    description TEXT,
    platform VARCHAR(50) NOT NULL,
    model VARCHAR(100), -- e.g., "UNO R3", "ESP32-WROOM-32"
    manufacturer VARCHAR(100), -- e.g., "Arduino", "Espressif"
    firmware_version VARCHAR(50),
    serial_number VARCHAR(100) UNIQUE,
    mac_address VARCHAR(17), -- For WiFi/BT boards
    ip_address INET, -- Current IP if connected
    status VARCHAR(20) NOT NULL DEFAULT 'inactive',
    health_status VARCHAR(20) DEFAULT 'unknown',
    thumbnail_url TEXT,
    specifications JSONB, -- Flexible specs storage
    last_heartbeat TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Status ENUM: 'active', 'inactive', 'maintenance', 'faulty', 'retired'
-- Health Status ENUM: 'healthy', 'warning', 'critical', 'unknown', 'offline'

CREATE INDEX idx_boards_platform ON boards(platform);
CREATE INDEX idx_boards_status ON boards(status);
CREATE INDEX idx_boards_serial ON boards(serial_number);
```

### 2. Board Pins Table

```sql
CREATE TABLE board_pins (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    board_id UUID NOT NULL REFERENCES boards(id) ON DELETE CASCADE,
    pin_number VARCHAR(10) NOT NULL, -- e.g., "D2", "A0", "GPIO23"
    pin_type VARCHAR(20) NOT NULL, -- 'digital', 'analog', 'pwm', 'i2c', 'spi', 'uart'
    pin_mode VARCHAR(20), -- 'input', 'output', 'input_pullup', 'input_pulldown'
    label VARCHAR(100), -- Custom label, e.g., "LED Red", "Temperature Sensor"
    assigned_component_id UUID, -- Reference to sensor or actuator
    assigned_component_type VARCHAR(20), -- 'sensor' or 'actuator'
    is_available BOOLEAN DEFAULT true,
    voltage DECIMAL(3,1), -- Operating voltage (3.3 or 5.0)
    max_current INTEGER, -- Max current in mA
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_board_pins_board ON board_pins(board_id);
CREATE INDEX idx_board_pins_available ON board_pins(is_available);
```

### 3. Board Components Table (Pre-defined Sensor/Actuator Library)

```sql
CREATE TABLE board_components (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    type VARCHAR(50) NOT NULL, -- 'sensor' or 'actuator'
    category VARCHAR(50) NOT NULL, -- e.g., 'temperature', 'led', 'motor', 'display'
    model VARCHAR(100), -- e.g., "DHT22", "SG90", "SSD1306"
    manufacturer VARCHAR(100),
    description TEXT,
    specifications JSONB, -- Technical specs
    datasheet_url TEXT,
    image_url TEXT,
    pin_requirements JSONB, -- Required pin types and count
    library_name VARCHAR(100), -- Arduino/Python library name
    sample_code TEXT, -- Example code snippet
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_board_components_type ON board_components(type);
CREATE INDEX idx_board_components_category ON board_components(category);
```

### 4. Board Component Assignments (Link Board to Components)

```sql
CREATE TABLE board_component_assignments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    board_id UUID NOT NULL REFERENCES boards(id) ON DELETE CASCADE,
    component_id UUID NOT NULL REFERENCES board_components(id) ON DELETE CASCADE,
    pin_mappings JSONB NOT NULL, -- e.g., {"VCC": "5V", "GND": "GND", "DATA": "D2"}
    label VARCHAR(100), -- Instance label
    is_active BOOLEAN DEFAULT true,
    configuration JSONB, -- Component-specific settings
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(board_id, component_id, label)
);

CREATE INDEX idx_board_assignments_board ON board_component_assignments(board_id);
```

### 5. Board Templates (Reusable Configurations)

```sql
CREATE TABLE board_templates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) UNIQUE NOT NULL,
    description TEXT,
    platform VARCHAR(50) NOT NULL,
    target_use_case VARCHAR(100), -- e.g., "IoT Weather Station", "Motor Control"
    components JSONB NOT NULL, -- Array of component IDs with configurations
    thumbnail_url TEXT,
    difficulty_level VARCHAR(20), -- 'beginner', 'intermediate', 'advanced'
    estimated_cost DECIMAL(10,2),
    is_public BOOLEAN DEFAULT false,
    author_id UUID REFERENCES users(id),
    usage_count INTEGER DEFAULT 0,
    rating DECIMAL(3,2) DEFAULT 0,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_board_templates_platform ON board_templates(platform);
CREATE INDEX idx_board_templates_public ON board_templates(is_public);
```

### 6. Board Lab Assignments (Link Board to Lab)

```sql
CREATE TABLE board_lab_assignments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    board_id UUID NOT NULL REFERENCES boards(id) ON DELETE CASCADE,
    lab_id UUID NOT NULL REFERENCES labs(id) ON DELETE CASCADE,
    is_primary BOOLEAN DEFAULT false, -- Primary board for the lab
    assigned_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    assigned_by UUID REFERENCES users(id),
    notes TEXT,
    UNIQUE(board_id, lab_id)
);

CREATE INDEX idx_board_lab_assignments_lab ON board_lab_assignments(lab_id);
CREATE INDEX idx_board_lab_assignments_board ON board_lab_assignments(board_id);
```

---

## 🔌 API Endpoints

### Base URL: `/api/v1/admin/boards`

---

### 1. List All Boards

**Endpoint:** `GET /api/v1/admin/boards`

**Description:** Retrieve all boards with optional filtering.

**Authentication:** Required (Admin role)

**Query Parameters:**
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `page` | int | 1 | Page number |
| `limit` | int | 20 | Items per page (max 50) |
| `search` | string | null | Search by name, serial, or model |
| `platform` | string | null | Filter by platform |
| `status` | string | null | Filter by status |
| `health` | string | null | Filter by health status |
| `assigned` | bool | null | Filter by lab assignment status |
| `sort` | string | "created_at" | Sort field |
| `order` | string | "desc" | Sort order (asc/desc) |

**Response:**
```json
{
    "success": true,
    "data": {
        "boards": [
            {
                "id": "uuid",
                "name": "ESP32 Board #1",
                "slug": "esp32-board-1",
                "description": "IoT Development Board",
                "platform": "esp32",
                "model": "ESP32-WROOM-32",
                "manufacturer": "Espressif",
                "firmware_version": "v2.0.1",
                "serial_number": "ESP32-001-ABC",
                "mac_address": "AA:BB:CC:DD:EE:FF",
                "ip_address": "192.168.1.100",
                "status": "active",
                "health_status": "healthy",
                "thumbnail_url": "https://example.com/esp32.jpg",
                "specifications": {
                    "cpu": "Xtensa LX6 240MHz",
                    "ram": "520KB SRAM",
                    "flash": "4MB",
                    "wifi": "802.11 b/g/n",
                    "bluetooth": "BT 4.2 + BLE"
                },
                "assigned_lab": {
                    "id": "uuid",
                    "name": "IoT Lab #1",
                    "slug": "iot-lab-1"
                },
                "components_count": 5,
                "last_heartbeat": "2026-01-14T01:00:00Z",
                "created_at": "2026-01-01T00:00:00Z",
                "updated_at": "2026-01-14T00:00:00Z"
            }
        ],
        "pagination": {
            "page": 1,
            "limit": 20,
            "total_items": 25,
            "total_pages": 2
        }
    }
}
```

---

### 2. Get Single Board

**Endpoint:** `GET /api/v1/admin/boards/:id`

**Description:** Get detailed information about a specific board.

**Response:**
```json
{
    "success": true,
    "data": {
        "id": "uuid",
        "name": "ESP32 Board #1",
        "slug": "esp32-board-1",
        "description": "IoT Development Board",
        "platform": "esp32",
        "model": "ESP32-WROOM-32",
        "manufacturer": "Espressif",
        "firmware_version": "v2.0.1",
        "serial_number": "ESP32-001-ABC",
        "mac_address": "AA:BB:CC:DD:EE:FF",
        "ip_address": "192.168.1.100",
        "status": "active",
        "health_status": "healthy",
        "thumbnail_url": "https://example.com/esp32.jpg",
        "specifications": {
            "cpu": "Xtensa LX6 240MHz",
            "ram": "520KB SRAM",
            "flash": "4MB",
            "gpio_count": 34,
            "adc_channels": 18,
            "dac_channels": 2,
            "wifi": "802.11 b/g/n",
            "bluetooth": "BT 4.2 + BLE"
        },
        "pins": [
            {
                "id": "uuid",
                "pin_number": "GPIO23",
                "pin_type": "digital",
                "pin_mode": "output",
                "label": "LED Red",
                "is_available": false,
                "assigned_component": {
                    "id": "uuid",
                    "name": "Red LED",
                    "type": "actuator"
                }
            }
        ],
        "components": [
            {
                "id": "uuid",
                "component": {
                    "id": "uuid",
                    "name": "DHT22",
                    "type": "sensor",
                    "category": "temperature"
                },
                "pin_mappings": {"VCC": "3V3", "GND": "GND", "DATA": "GPIO4"},
                "label": "Temperature Sensor",
                "is_active": true
            }
        ],
        "assigned_lab": {
            "id": "uuid",
            "name": "IoT Lab #1",
            "slug": "iot-lab-1"
        },
        "last_heartbeat": "2026-01-14T01:00:00Z",
        "created_at": "2026-01-01T00:00:00Z",
        "updated_at": "2026-01-14T00:00:00Z"
    }
}
```

---

### 3. Create Board

**Endpoint:** `POST /api/v1/admin/boards`

**Description:** Create a new board configuration.

**Request Body:**
```json
{
    "name": "ESP32 Board #2",
    "slug": "esp32-board-2",
    "description": "New IoT development board",
    "platform": "esp32",
    "model": "ESP32-WROOM-32",
    "manufacturer": "Espressif",
    "firmware_version": "v2.0.1",
    "serial_number": "ESP32-002-DEF",
    "mac_address": "AA:BB:CC:DD:EE:02",
    "status": "inactive",
    "thumbnail_url": "https://example.com/esp32.jpg",
    "specifications": {
        "cpu": "Xtensa LX6 240MHz",
        "ram": "520KB SRAM",
        "flash": "4MB"
    }
}
```

**Validation Rules:**
| Field | Rule |
|-------|------|
| `name` | Required, max 255 characters |
| `slug` | Optional (auto-generated), unique, lowercase with hyphens |
| `platform` | Required, must be valid platform code |
| `serial_number` | Optional, unique if provided |
| `mac_address` | Optional, valid MAC format |
| `status` | Required, valid status value |

**Response:**
```json
{
    "success": true,
    "data": {
        "id": "uuid",
        "name": "ESP32 Board #2",
        ...
    },
    "message": "Board created successfully"
}
```

---

### 4. Update Board

**Endpoint:** `PUT /api/v1/admin/boards/:id`

**Description:** Update an existing board.

**Request Body:** Same as Create (partial updates allowed)

**Response:**
```json
{
    "success": true,
    "data": {...},
    "message": "Board updated successfully"
}
```

---

### 5. Delete Board

**Endpoint:** `DELETE /api/v1/admin/boards/:id`

**Description:** Delete a board (soft delete recommended).

**Response:**
```json
{
    "success": true,
    "message": "Board deleted successfully"
}
```

**Notes:**
- Should fail if board is assigned to an active lab
- Consider soft delete for audit trail

---

### 6. Update Board Status

**Endpoint:** `PATCH /api/v1/admin/boards/:id/status`

**Request Body:**
```json
{
    "status": "maintenance",
    "reason": "Replacing faulty sensor"
}
```

---

## 📱 Board Components API

### 7. List Components Library

**Endpoint:** `GET /api/v1/admin/boards/components`

**Query Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `type` | string | Filter by 'sensor' or 'actuator' |
| `category` | string | Filter by category |
| `search` | string | Search by name or model |

**Response:**
```json
{
    "success": true,
    "data": {
        "components": [
            {
                "id": "uuid",
                "name": "DHT22",
                "type": "sensor",
                "category": "temperature",
                "model": "DHT22/AM2302",
                "manufacturer": "Aosong",
                "description": "Digital temperature and humidity sensor",
                "specifications": {
                    "temperature_range": "-40 to 80°C",
                    "humidity_range": "0-100%",
                    "accuracy": "±0.5°C, ±2%RH"
                },
                "pin_requirements": {
                    "vcc": {"type": "power", "voltage": [3.3, 5.0]},
                    "gnd": {"type": "ground"},
                    "data": {"type": "digital"}
                },
                "library_name": "DHT",
                "image_url": "https://example.com/dht22.jpg"
            }
        ]
    }
}
```

---

### 8. Add Component to Board

**Endpoint:** `POST /api/v1/admin/boards/:id/components`

**Request Body:**
```json
{
    "component_id": "uuid",
    "pin_mappings": {
        "VCC": "3V3",
        "GND": "GND",
        "DATA": "GPIO4"
    },
    "label": "Room Temperature Sensor",
    "configuration": {
        "read_interval": 2000
    }
}
```

---

### 9. Remove Component from Board

**Endpoint:** `DELETE /api/v1/admin/boards/:id/components/:assignmentId`

---

## 🔗 Board-Lab Assignment API

### 10. Assign Board to Lab

**Endpoint:** `POST /api/v1/admin/boards/:id/assign`

**Request Body:**
```json
{
    "lab_id": "uuid",
    "is_primary": true,
    "notes": "Primary board for IoT experiments"
}
```

---

### 11. Unassign Board from Lab

**Endpoint:** `DELETE /api/v1/admin/boards/:id/assign/:labId`

---

### 12. Get Boards by Lab

**Endpoint:** `GET /api/v1/admin/labs/:labId/boards`

---

## 📊 Board Templates API

### 13. List Board Templates

**Endpoint:** `GET /api/v1/admin/boards/templates`

**Query Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `platform` | string | Filter by platform |
| `use_case` | string | Filter by target use case |
| `difficulty` | string | Filter by difficulty level |
| `public` | bool | Filter public templates only |

---

### 14. Create Board Template

**Endpoint:** `POST /api/v1/admin/boards/templates`

**Request Body:**
```json
{
    "name": "IoT Weather Station",
    "slug": "iot-weather-station",
    "description": "Complete weather monitoring setup",
    "platform": "esp32",
    "target_use_case": "Weather Monitoring",
    "difficulty_level": "intermediate",
    "estimated_cost": 45.00,
    "components": [
        {
            "component_id": "uuid",
            "pin_mappings": {"VCC": "3V3", "GND": "GND", "DATA": "GPIO4"},
            "label": "DHT22 Temperature Sensor"
        },
        {
            "component_id": "uuid",
            "pin_mappings": {"VCC": "3V3", "GND": "GND", "SCL": "GPIO22", "SDA": "GPIO21"},
            "label": "BMP280 Pressure Sensor"
        }
    ],
    "is_public": true
}
```

---

### 15. Apply Template to Board

**Endpoint:** `POST /api/v1/admin/boards/:id/apply-template`

**Request Body:**
```json
{
    "template_id": "uuid"
}
```

---

## 📈 Statistics Endpoint

**Endpoint:** `GET /api/v1/admin/boards/stats`

**Response:**
```json
{
    "success": true,
    "data": {
        "total_boards": 25,
        "by_status": {
            "active": 15,
            "inactive": 5,
            "maintenance": 3,
            "faulty": 1,
            "retired": 1
        },
        "by_platform": {
            "esp32": 10,
            "arduino_uno": 8,
            "raspberry_pi_4": 4,
            "stm32": 3
        },
        "by_health": {
            "healthy": 18,
            "warning": 4,
            "critical": 1,
            "unknown": 2
        },
        "assigned_to_labs": 12,
        "unassigned": 13,
        "total_components_installed": 85,
        "most_used_components": [
            {"name": "DHT22", "count": 15},
            {"name": "LED", "count": 20},
            {"name": "Servo SG90", "count": 10}
        ]
    }
}
```

---

## 🔐 Authorization

All endpoints require:
1. Valid JWT token
2. User role: `admin`

---

## 🩺 Health Monitoring

### Board Heartbeat

Boards should send periodic heartbeat signals:

**MQTT Topic:** `nexflux/boards/{board_id}/heartbeat`

**Payload:**
```json
{
    "timestamp": "2026-01-14T01:00:00Z",
    "uptime": 86400,
    "free_memory": 150000,
    "wifi_rssi": -45,
    "temperature": 42.5,
    "status": "healthy"
}
```

### Auto Status Update

```go
// If no heartbeat for 60 seconds
if time.Since(board.LastHeartbeat) > 60 * time.Second {
    board.HealthStatus = "offline"
    board.Status = "inactive"
}
```

---

## 📝 Error Codes

| Code | Message |
|------|---------|
| `BOARD_001` | Board not found |
| `BOARD_002` | Board already exists (duplicate serial) |
| `BOARD_003` | Invalid platform type |
| `BOARD_004` | Board is assigned to active lab |
| `BOARD_005` | Component not compatible with platform |
| `BOARD_006` | Pin already assigned |
| `BOARD_007` | Invalid pin for this board |
| `BOARD_008` | Template not found |
| `BOARD_009` | Cannot delete active board |

---

## 📋 Implementation Checklist

### Backend Tasks

- [ ] **Database Migrations**
  - [ ] Create `boards` table
  - [ ] Create `board_pins` table
  - [ ] Create `board_components` table
  - [ ] Create `board_component_assignments` table
  - [ ] Create `board_templates` table
  - [ ] Create `board_lab_assignments` table
  - [ ] Add indexes

- [ ] **Seed Data**
  - [ ] Seed common components (DHT22, LED, Servo, etc.)
  - [ ] Seed platform specifications
  - [ ] Create sample templates

- [ ] **Repository Layer**
  - [ ] CRUD for boards
  - [ ] Component management
  - [ ] Template management
  - [ ] Lab assignment

- [ ] **Service Layer**
  - [ ] Validation logic
  - [ ] Slug generation
  - [ ] Health status updates
  - [ ] Template application

- [ ] **Handler Layer**
  - [ ] All CRUD endpoints
  - [ ] Statistics endpoint

- [ ] **MQTT Integration**
  - [ ] Heartbeat listener
  - [ ] Status sync

---

## 🚀 Frontend Integration

The frontend Board Management page will be at:
```
/admin/boards
```

Features:
- Grid/list view of all boards
- Add/Edit board modal
- Pin configuration interface
- Component drag-and-drop assignment
- Template management
- Lab assignment interface
- Health monitoring dashboard

---

## 📌 Component Categories

### Sensors
| Category | Examples |
|----------|----------|
| Temperature | DHT11, DHT22, DS18B20, LM35 |
| Humidity | DHT22, BME280, Si7021 |
| Pressure | BMP280, BMP180, BME280 |
| Light | LDR, BH1750, TSL2561 |
| Motion | PIR HC-SR501, MPU6050 |
| Distance | HC-SR04, VL53L0X |
| Gas | MQ-2, MQ-135, CCS811 |
| Sound | KY-038, MAX9814 |
| Touch | TTP223, Capacitive |

### Actuators
| Category | Examples |
|----------|----------|
| LED | Single LED, RGB LED, WS2812B |
| Motor | DC Motor, Stepper, Servo |
| Relay | Single, Multi-channel |
| Display | LCD 16x2, OLED, TFT |
| Buzzer | Active, Passive, Piezo |
| Fan | DC Fan, PWM Fan |

---

*Document Version: 1.0*
*Last Updated: 2026-01-14*
*Status: Pending Implementation*

# NexFlux AI Service Documentation

## Overview

NexFlux AI Service adalah layanan AI terpisah yang dibangun dengan Python (FastAPI) untuk menyediakan kemampuan Computer Vision dan AI Assistance untuk Virtual Lab. Service ini menangani:

- **Deteksi Komponen Elektronik** - Mengenali resistor, kapasitor, LED, dsb dari video stream
- **Validasi Langkah Lab** - Memverifikasi apakah setup circuit user sudah benar
- **Guidance Real-time** - Memberikan panduan visual overlay dan animasi
- **AI Chat** - Bantuan natural language untuk lab assistance

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                         NexFlux Frontend                              │
│                    (React + TypeScript + Vite)                        │
└─────────────────────────────┬────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────────┐
│                        NexFlux AI Service                             │
│                     (Python + FastAPI + YOLOv8)                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │
│  │  Vision API │  │  Chat API   │  │ Models API  │  │  WebSocket  │  │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  │
│         │                │                │                │          │
│  ┌──────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐  │
│  │  Component  │  │    Chat     │  │   Model     │  │   Stream    │  │
│  │  Detector   │  │   Service   │  │   Manager   │  │  Processor  │  │
│  └──────┬──────┘  └──────┬──────┘  └─────────────┘  └─────────────┘  │
│         │                │                                            │
│  ┌──────▼──────┐  ┌──────▼──────┐                                    │
│  │   YOLOv8    │  │  OpenAI /   │                                    │
│  │   Model     │  │  Anthropic  │                                    │
│  └─────────────┘  └─────────────┘                                    │
└──────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────────┐
│                       NexFlux Backend (Go)                            │
│                      (Session & User Data)                            │
└──────────────────────────────────────────────────────────────────────┘
```

---

## API Endpoints

### Health Check

```http
GET /health
```

**Response:**
```json
{
  "status": "healthy",
  "service": "nexflux-ai-service",
  "version": "1.0.0"
}
```

---

### Vision API

#### Detect Components

Mendeteksi komponen elektronik dalam gambar.

```http
POST /api/v1/vision/detect
Content-Type: application/json
```

**Request Body:**
```json
{
  "image": "data:image/jpeg;base64,/9j/4AAQ...",
  "confidence_threshold": 0.5,
  "max_detections": 20
}
```

**Response:**
```json
{
  "success": true,
  "components": [
    {
      "id": "comp_abc123",
      "type": "LED",
      "label": "Red LED",
      "confidence": 0.95,
      "bbox": {
        "x": 100,
        "y": 150,
        "width": 50,
        "height": 80
      },
      "properties": {
        "color": "red"
      }
    },
    {
      "id": "comp_def456",
      "type": "RESISTOR",
      "label": "220Ω Resistor",
      "confidence": 0.88,
      "bbox": {
        "x": 200,
        "y": 180,
        "width": 80,
        "height": 30
      },
      "properties": {
        "value": "220",
        "unit": "ohm"
      }
    }
  ],
  "annotated_image": "data:image/jpeg;base64,/9j/4AAQ...",
  "processing_time_ms": 45
}
```

---

#### Validate Lab Step

Memvalidasi apakah setup circuit sesuai dengan langkah lab yang diharapkan.

```http
POST /api/v1/vision/validate-step
Content-Type: application/json
```

**Request Body:**
```json
{
  "image": "data:image/jpeg;base64,/9j/4AAQ...",
  "step": {
    "step_id": 1,
    "title": "Connect LED to Breadboard",
    "description": "Place the LED on the breadboard with anode on row 5",
    "required_components": ["LED", "RESISTOR"],
    "expected_connections": [
      {
        "from_component": "LED",
        "from_pin": "anode",
        "to_component": "RESISTOR",
        "to_pin": "terminal1"
      }
    ]
  },
  "confidence_threshold": 0.5
}
```

**Response:**
```json
{
  "success": true,
  "step_id": 1,
  "status": "PARTIAL",
  "message": "LED detected. Missing: 220Ω Resistor",
  "detected_components": [...],
  "missing_components": ["RESISTOR"],
  "suggestions": [
    "Add a 220Ω resistor to protect the LED",
    "Connect the resistor between the LED anode and power rail"
  ],
  "annotated_image": "data:image/jpeg;base64,..."
}
```

**Step Status Values:**
| Status | Description |
|--------|-------------|
| `NOT_STARTED` | Belum ada komponen yang terdeteksi |
| `IN_PROGRESS` | Beberapa komponen terdeteksi, belum lengkap |
| `PARTIAL` | Komponen lengkap tapi koneksi belum benar |
| `CORRECT` | Semua komponen dan koneksi sudah benar |
| `INCORRECT` | Ada kesalahan dalam setup |

---

#### Real-time Video Stream (WebSocket)

WebSocket endpoint untuk analisis video real-time.

```
WS /api/v1/vision/stream?lab_id=led-blink
```

**Client → Server Messages:**

```json
// Send frame for analysis
{
  "type": "frame",
  "data": {
    "image": "data:image/jpeg;base64,...",
    "step": {
      "step_id": 1,
      "title": "Connect LED",
      "required_components": ["LED", "RESISTOR"]
    }
  }
}

// Ping
{
  "type": "ping"
}

// Close connection
{
  "type": "close"
}
```

**Server → Client Messages:**

```json
// Ready message (on connect)
{
  "type": "ready",
  "data": {
    "message": "Stream analysis ready",
    "lab_id": "led-blink"
  }
}

// Analysis result
{
  "type": "analysis",
  "data": {
    "frame_id": "frame_42",
    "components": [...],
    "step_status": "PARTIAL",
    "annotations": [
      {
        "component_id": "comp_abc",
        "type": "label",
        "position": {"x": 100, "y": 150},
        "label": "LED",
        "style": "correct",
        "animate": true
      },
      {
        "type": "hint",
        "message": "Add: Resistor",
        "style": "warning",
        "animate": true
      }
    ],
    "guidance": "Great! Now add a 220Ω resistor",
    "annotated_image": "data:image/jpeg;base64,..."
  }
}

// Heartbeat
{
  "type": "heartbeat"
}

// Error
{
  "type": "error",
  "data": {
    "message": "Invalid frame data"
  }
}
```

---

#### Get Component Types

Mendapatkan daftar tipe komponen yang didukung.

```http
GET /api/v1/vision/component-types
```

**Response:**
```json
{
  "success": true,
  "component_types": [
    {"value": "LED", "label": "Led"},
    {"value": "RESISTOR", "label": "Resistor"},
    {"value": "CAPACITOR", "label": "Capacitor"},
    {"value": "TRANSISTOR", "label": "Transistor"},
    {"value": "DIODE", "label": "Diode"},
    {"value": "INDUCTOR", "label": "Inductor"},
    {"value": "IC", "label": "Ic"},
    {"value": "MICROCONTROLLER", "label": "Microcontroller"},
    {"value": "BREADBOARD", "label": "Breadboard"},
    {"value": "WIRE", "label": "Wire"},
    {"value": "POWER_SUPPLY", "label": "Power Supply"},
    {"value": "BUTTON", "label": "Button"},
    {"value": "SWITCH", "label": "Switch"},
    {"value": "POTENTIOMETER", "label": "Potentiometer"},
    {"value": "MOTOR", "label": "Motor"},
    {"value": "SENSOR", "label": "Sensor"},
    {"value": "DISPLAY", "label": "Display"},
    {"value": "BUZZER", "label": "Buzzer"},
    {"value": "RELAY", "label": "Relay"},
    {"value": "UNKNOWN", "label": "Unknown"}
  ]
}
```

---

### Chat API

#### Send Message

Mengirim pesan ke AI assistant.

```http
POST /api/v1/chat/message
Content-Type: application/json
```

**Request Body:**
```json
{
  "message": "How do I connect an LED to Arduino?",
  "session_id": "session_abc123",
  "context": {
    "lab_id": "led-blink",
    "current_step": 2,
    "detected_components": ["LED", "RESISTOR"]
  },
  "model": "gpt-4o"
}
```

**Response:**
```json
{
  "success": true,
  "message": "To connect an LED to Arduino, you'll need:\n\n1. **Components:**\n   - 1x LED (any color)\n   - 1x 220Ω resistor\n   - Jumper wires\n\n2. **Connect the LED:**\n   - Connect the LED's **anode** (longer leg) to a digital pin (e.g., pin 13)\n   - Connect the LED's **cathode** (shorter leg) to one end of the resistor\n   - Connect the other end of the resistor to **GND**\n\n3. **Why use a resistor?**\n   The resistor limits current to protect the LED from burning out.\n\nWould you like me to explain the code to control this LED?",
  "session_id": "session_abc123",
  "model_used": "gpt-4o"
}
```

---

#### Stream Message (SSE)

Streaming response menggunakan Server-Sent Events.

```http
POST /api/v1/chat/message/stream
Content-Type: application/json
```

**Request Body:** (sama dengan `/message`)

**Response (SSE):**
```
data: {"content": "To connect"}

data: {"content": " an LED"}

data: {"content": " to Arduino..."}

data: {"done": true}
```

---

#### Chat WebSocket

WebSocket untuk chat real-time.

```
WS /api/v1/chat/ws?session_id=abc123
```

**Client → Server:**
```json
{
  "type": "message",
  "data": {
    "text": "What is a resistor?",
    "context": {"lab_id": "basic-electronics"}
  }
}
```

**Server → Client:**
```json
// Streaming chunks
{
  "type": "response",
  "data": {
    "text": "A resistor is",
    "complete": false
  }
}

// Final message
{
  "type": "response",
  "data": {
    "text": "",
    "complete": true,
    "full_text": "A resistor is an electronic component..."
  }
}
```

---

#### Get Chat History

```http
GET /api/v1/chat/history/{session_id}
```

**Response:**
```json
{
  "success": true,
  "session_id": "abc123",
  "messages": [
    {"role": "user", "content": "What is a resistor?"},
    {"role": "assistant", "content": "A resistor is..."}
  ]
}
```

---

#### Clear Chat History

```http
DELETE /api/v1/chat/history/{session_id}
```

---

### Models API

#### List Models

```http
GET /api/v1/models/
```

**Response:**
```json
[
  {
    "id": "component-detector",
    "name": "Component Detector",
    "type": "vision",
    "version": "1.0.0",
    "status": "LOADED",
    "memory_mb": 256
  },
  {
    "id": "step-validator",
    "name": "Step Validator",
    "type": "vision",
    "version": "1.0.0",
    "status": "LOADED",
    "memory_mb": 0
  }
]
```

---

#### Get Vision Capabilities

```http
GET /api/v1/models/capabilities/vision
```

**Response:**
```json
{
  "success": true,
  "capabilities": {
    "component_detection": true,
    "step_validation": true,
    "real_time_streaming": true,
    "supported_formats": ["jpeg", "png", "webp"],
    "max_image_size": "4096x4096",
    "demo_mode": false,
    "confidence_threshold_range": {"min": 0.1, "max": 1.0}
  }
}
```

---

#### Get Chat Capabilities

```http
GET /api/v1/models/capabilities/chat
```

**Response:**
```json
{
  "success": true,
  "capabilities": {
    "streaming": true,
    "conversation_memory": true,
    "context_aware": true,
    "providers": [
      {
        "id": "openai",
        "name": "OpenAI",
        "models": ["gpt-4o", "gpt-4o-mini", "gpt-4-turbo"]
      },
      {
        "id": "anthropic",
        "name": "Anthropic",
        "models": ["claude-3-5-sonnet", "claude-3-opus", "claude-3-haiku"]
      }
    ],
    "demo_mode": false
  }
}
```

---

## Data Models

### ComponentType (Enum)

```
LED, RESISTOR, CAPACITOR, TRANSISTOR, DIODE, INDUCTOR, IC,
MICROCONTROLLER, BREADBOARD, WIRE, POWER_SUPPLY, BUTTON,
SWITCH, POTENTIOMETER, MOTOR, SENSOR, DISPLAY, BUZZER, RELAY, UNKNOWN
```

### StepStatus (Enum)

```
NOT_STARTED, IN_PROGRESS, PARTIAL, CORRECT, INCORRECT
```

### ModelStatus (Enum)

```
AVAILABLE, LOADING, LOADED, ERROR, UNLOADING
```

### DetectedComponent

| Field | Type | Description |
|-------|------|-------------|
| id | string | Unique component identifier |
| type | ComponentType | Type of component |
| label | string | Human-readable label |
| confidence | float | Detection confidence (0-1) |
| bbox | BoundingBox | Position and size |
| properties | object | Additional properties |

### BoundingBox

| Field | Type | Description |
|-------|------|-------------|
| x | int | X coordinate (top-left) |
| y | int | Y coordinate (top-left) |
| width | int | Width in pixels |
| height | int | Height in pixels |

### LabStep

| Field | Type | Description |
|-------|------|-------------|
| step_id | int | Step number |
| title | string | Step title |
| description | string | Step description |
| required_components | ComponentType[] | Required components |
| expected_connections | Connection[] | Expected connections |

---

## Environment Variables

```env
# Server Configuration
HOST=0.0.0.0
PORT=8001
DEBUG=true
LOG_LEVEL=INFO

# CORS
ALLOWED_ORIGINS=http://localhost:5173,http://localhost:3000

# AI Provider API Keys
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
GOOGLE_AI_API_KEY=...

# NexFlux Backend Integration
NEXFLUX_BACKEND_URL=http://localhost:8000/api/v1
NEXFLUX_API_KEY=your-api-key

# Redis (Optional - for caching)
REDIS_URL=redis://localhost:6379
REDIS_PASSWORD=

# ML Model Paths
COMPONENT_DETECTION_MODEL=ml_models/component_detector.onnx
CIRCUIT_ANALYSIS_MODEL=ml_models/circuit_analyzer.onnx

# WebSocket Configuration
WS_HEARTBEAT_INTERVAL=30
MAX_CONNECTIONS_PER_USER=3

# Rate Limiting
RATE_LIMIT_PER_MINUTE=60
```

---

## Running the Service

### Development

```bash
cd nexflux-ai-service

# Create virtual environment
python3 -m venv venv
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Run with auto-reload
make run
# or: uvicorn main:app --reload --host 0.0.0.0 --port 8001
```

### Production

```bash
# Using Uvicorn with multiple workers
uvicorn main:app --host 0.0.0.0 --port 8001 --workers 4

# Using Docker
docker build -t nexflux-ai-service .
docker run -p 8001:8001 --env-file .env nexflux-ai-service
```

### Testing

```bash
# Run all tests
make test

# Run with coverage
make test-cov
```

---

## Frontend Integration

### React Hook Example

```typescript
// hooks/useAIVision.ts
import { useState, useCallback, useRef } from 'react';

const AI_SERVICE_URL = 'http://localhost:8001';

export function useAIVision() {
  const [components, setComponents] = useState<DetectedComponent[]>([]);
  const [stepStatus, setStepStatus] = useState<StepStatus | null>(null);
  const wsRef = useRef<WebSocket | null>(null);

  const connectStream = useCallback((labId: string) => {
    const ws = new WebSocket(`${AI_SERVICE_URL}/api/v1/vision/stream?lab_id=${labId}`);
    
    ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      
      if (data.type === 'analysis') {
        setComponents(data.data.components);
        setStepStatus(data.data.step_status);
      }
    };
    
    wsRef.current = ws;
  }, []);

  const sendFrame = useCallback((imageData: string, step?: LabStep) => {
    if (wsRef.current?.readyState === WebSocket.OPEN) {
      wsRef.current.send(JSON.stringify({
        type: 'frame',
        data: { image: imageData, step }
      }));
    }
  }, []);

  const disconnect = useCallback(() => {
    wsRef.current?.close();
  }, []);

  return { components, stepStatus, connectStream, sendFrame, disconnect };
}
```

---

## Error Codes

| Code | Description |
|------|-------------|
| 400 | Bad Request - Invalid input data |
| 401 | Unauthorized - Invalid API key |
| 404 | Not Found - Resource not found |
| 422 | Validation Error - Invalid request format |
| 429 | Too Many Requests - Rate limit exceeded |
| 500 | Internal Server Error |
| 503 | Service Unavailable - Model not loaded |

---

## Demo Mode

When ML models are not available, the service runs in **demo mode**:

- Component detection returns simulated results
- Chat uses fallback responses
- Useful for development and testing without GPU

To check if running in demo mode:
```http
GET /api/v1/models/capabilities/vision
# Response will include: "demo_mode": true
```

---

## Related Documentation

- [AI Studio Manager](./AI_STUDIO_MANAGER.md) - Admin panel for AI configuration
- [Lab Management](./LABCRUD.md) - Lab CRUD operations
- [Board Management](./BOARD.md) - Hardware board configuration

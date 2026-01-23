# NexFlux AI Vision API Documentation

Complete API reference for the AI Vision Service integration in NexFlux Virtual Lab.

## Overview

The AI Vision API provides computer vision capabilities for real-time circuit analysis, component detection, and step validation during lab sessions. The system consists of:

1. **Go Backend** (`/api/v1/ai-vision/*`) - Session management, data persistence, and AI Service proxy
2. **AI Service** (Python/FastAPI) - ML models for detection and validation

---

## Authentication

All endpoints require JWT authentication via the `Authorization` header:

```http
Authorization: Bearer <jwt_token>
```

---

## Base URLs

| Service | URL |
|---------|-----|
| Backend API | `http://localhost:8005/api/v1` |
| AI Service | `http://localhost:8001/api/v1` |

---

## Endpoints

### Vision Sessions

#### Create Session
Creates a new vision analysis session.

```http
POST /ai-vision/sessions
```

**Request Body:**
```json
{
  "lab_id": "uuid",
  "board_id": "uuid",
  "metadata": {
    "camera_type": "usb",
    "resolution": "1080p"
  }
}
```

**Response:**
```json
{
  "success": true,
  "message": "Session created",
  "data": {
    "id": "uuid",
    "user_id": "uuid",
    "lab_id": "uuid",
    "board_id": "uuid",
    "status": "active",
    "total_frames": 0,
    "analyzed_frames": 0,
    "detection_count": 0,
    "created_at": "2026-01-22T10:00:00Z"
  }
}
```

---

#### Get Session
Retrieves a specific session by ID.

```http
GET /ai-vision/sessions/:id
```

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "user_id": "uuid",
    "lab_id": "uuid",
    "status": "active",
    "current_step_id": 2,
    "total_frames": 150,
    "analyzed_frames": 145,
    "detection_count": 23,
    "last_frame_at": "2026-01-22T10:05:00Z",
    "created_at": "2026-01-22T10:00:00Z"
  }
}
```

---

#### List User Sessions
Retrieves all sessions for the authenticated user.

```http
GET /ai-vision/sessions?status=active&limit=10&offset=0
```

**Query Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `status` | string | Filter by status (active, paused, completed, error) |
| `limit` | int | Results per page (default: 10) |
| `offset` | int | Pagination offset |

**Response:**
```json
{
  "success": true,
  "data": {
    "sessions": [...],
    "total": 25,
    "limit": 10,
    "offset": 0
  }
}
```

---

#### Update Session Status

```http
PATCH /ai-vision/sessions/:id/status
```

**Request Body:**
```json
{
  "status": "completed"
}
```

**Valid Statuses:** `active`, `paused`, `completed`, `error`

---

#### Delete Session

```http
DELETE /ai-vision/sessions/:id
```

---

### Component Detection

#### Detect Components (Proxy to AI Service)
Sends an image to the AI Service for component detection.

```http
POST /ai-vision/proxy/detect
```

**Request Body:**
```json
{
  "image": "base64_encoded_image",
  "session_id": "uuid",
  "confidence_threshold": 0.5,
  "max_detections": 20
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "components": [
      {
        "id": "det_001",
        "type": "resistor",
        "label": "10kΩ Resistor",
        "confidence": 0.95,
        "bbox": {
          "x": 0.25,
          "y": 0.30,
          "width": 0.08,
          "height": 0.03
        }
      },
      {
        "id": "det_002",
        "type": "led",
        "label": "Red LED",
        "confidence": 0.92,
        "bbox": {
          "x": 0.45,
          "y": 0.35,
          "width": 0.05,
          "height": 0.06
        }
      }
    ],
    "annotated_image": "base64_encoded_annotated_image",
    "processing_time_ms": 125
  }
}
```

---

#### Get Session Detections
Retrieves all detections recorded for a session.

```http
GET /ai-vision/sessions/:id/detections?limit=50&offset=0
```

**Response:**
```json
{
  "success": true,
  "data": {
    "detections": [
      {
        "id": "uuid",
        "session_id": "uuid",
        "frame_number": 45,
        "component_type": "resistor",
        "label": "10kΩ Resistor",
        "confidence": 0.95,
        "bbox_x": 125,
        "bbox_y": 200,
        "bbox_width": 80,
        "bbox_height": 30,
        "created_at": "2026-01-22T10:02:30Z"
      }
    ],
    "total": 150,
    "limit": 50,
    "offset": 0
  }
}
```

---

#### Get Detection Statistics

```http
GET /ai-vision/sessions/:id/detections/stats
```

**Response:**
```json
{
  "success": true,
  "data": {
    "resistor": 15,
    "led": 8,
    "wire": 42,
    "arduino": 1
  }
}
```

---

### Step Validation

#### Validate Step (Proxy to AI Service)
Validates current circuit setup against expected lab step.

```http
POST /ai-vision/proxy/validate
```

**Request Body:**
```json
{
  "image": "base64_encoded_image",
  "session_id": "uuid",
  "lab_id": "uuid",
  "step": {
    "step_id": 3,
    "title": "Connect LED to Arduino",
    "description": "Connect the red LED anode to pin 13",
    "required_components": ["led", "resistor", "wire", "arduino"]
  },
  "confidence_threshold": 0.5
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "step_id": 3,
    "status": "partial",
    "message": "LED detected but connection to pin 13 not verified",
    "detected_components": [
      {"type": "led", "label": "Red LED", "confidence": 0.92},
      {"type": "resistor", "label": "220Ω Resistor", "confidence": 0.88},
      {"type": "arduino", "label": "Arduino Uno", "confidence": 0.97}
    ],
    "missing_components": ["wire"],
    "suggestions": [
      "Add a wire from the resistor to pin 13 on the Arduino",
      "Make sure the LED cathode is connected to GND"
    ],
    "annotated_image": "base64_annotated"
  }
}
```

**Validation Statuses:**
| Status | Description |
|--------|-------------|
| `not_started` | No components detected for this step |
| `in_progress` | Some components detected |
| `partial` | Most components correct, some issues |
| `correct` | Step completed successfully |
| `incorrect` | Wrong setup detected |

---

#### Get Session Validations

```http
GET /ai-vision/sessions/:id/validations
```

---

#### Get Lab Progress
Get user's overall progress on a specific lab.

```http
GET /ai-vision/labs/:id/progress
```

**Response:**
```json
{
  "success": true,
  "data": {
    "lab_id": "uuid",
    "steps_completed": 4,
    "completed_steps": {
      "1": true,
      "2": true,
      "3": true,
      "4": true
    },
    "validation_count": 12
  }
}
```

---

### Lab Chat (AI Assistance)

#### Create Chat Session

```http
POST /ai-vision/chat/sessions
```

**Request Body:**
```json
{
  "lab_id": "uuid",
  "title": "Help with LED Circuit",
  "context": {
    "current_step": 3,
    "difficulty": "beginner"
  }
}
```

---

#### Get Chat Sessions

```http
GET /ai-vision/chat/sessions?lab_id=uuid&limit=10
```

---

#### Add Chat Message

```http
POST /ai-vision/chat/sessions/:id/messages
```

**Request Body:**
```json
{
  "role": "user",
  "content": "Why isn't my LED lighting up?"
}
```

---

#### Get Chat Messages

```http
GET /ai-vision/chat/sessions/:id/messages?limit=50
```

---

#### Send Message to AI (Proxy)

```http
POST /ai-vision/proxy/chat
```

**Request Body:**
```json
{
  "session_id": "uuid",
  "message": "How do I calculate the resistor value for my LED?",
  "context": {
    "lab_name": "LED Blink Tutorial",
    "current_step": 2
  }
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "response": "To calculate the resistor value for an LED, use Ohm's Law: R = (V_supply - V_led) / I_led. For a red LED with a 5V supply: R = (5V - 2V) / 0.02A = 150Ω. Use the next standard value: 220Ω.",
    "suggestions": [
      "Would you like to learn more about LED specifications?",
      "Show me the color code for a 220Ω resistor"
    ]
  }
}
```

---

### Utility Endpoints

#### Get Component Types

```http
GET /ai-vision/component-types
```

**Response:**
```json
{
  "success": true,
  "data": {
    "component_types": [
      {"value": "resistor", "label": "Resistor"},
      {"value": "capacitor", "label": "Capacitor"},
      {"value": "led", "label": "LED"},
      {"value": "transistor", "label": "Transistor"},
      {"value": "arduino", "label": "Arduino"},
      {"value": "esp32", "label": "ESP32"}
    ]
  }
}
```

---

#### Check AI Service Health

```http
GET /ai-vision/health
```

**Response:**
```json
{
  "success": true,
  "data": {
    "ai_service": "healthy",
    "details": {
      "status": "healthy",
      "version": "1.0.0",
      "models_loaded": 2
    }
  }
}
```

---

#### Get Available Models

```http
GET /ai-vision/models
```

---

#### Get Usage Statistics

```http
GET /ai-vision/stats
```

**Response:**
```json
{
  "success": true,
  "data": {
    "total_sessions": 45,
    "total_frames": 12500,
    "total_detections": 3400,
    "avg_processing_time": 125.5
  }
}
```

---

## WebSocket Streaming (AI Service)

For real-time video analysis, connect directly to the AI Service WebSocket:

```javascript
const ws = new WebSocket('ws://localhost:8001/api/v1/vision/stream?lab_id=uuid');

// Send frame
ws.send(JSON.stringify({
  type: 'frame',
  data: {
    image: 'base64_encoded_frame',
    step: {
      step_id: 1,
      title: 'Connect LED',
      required_components: ['led', 'resistor']
    }
  }
}));

// Receive analysis
ws.onmessage = (event) => {
  const message = JSON.parse(event.data);
  if (message.type === 'analysis') {
    console.log('Detected:', message.data.components);
    console.log('Status:', message.data.step_status);
  }
};
```

---

## Error Responses

All endpoints return errors in this format:

```json
{
  "success": false,
  "message": "Error description",
  "error": "Detailed error message"
}
```

**Common HTTP Status Codes:**
| Code | Description |
|------|-------------|
| 400 | Bad Request - Invalid input |
| 401 | Unauthorized - Invalid or missing token |
| 404 | Not Found - Resource doesn't exist |
| 500 | Internal Server Error |
| 503 | Service Unavailable - AI Service offline |

---

## Rate Limits

| Endpoint Type | Limit |
|--------------|-------|
| Detection/Validation | 60 requests/minute |
| Chat | 30 requests/minute |
| Streaming | 30 fps (configurable) |

---

## Environment Variables

### Go Backend
```env
AI_SERVICE_URL=http://localhost:8001
AI_SERVICE_API_KEY=your-api-key
```

### AI Service
```env
HOST=0.0.0.0
PORT=8001
DEBUG=true
ALLOWED_ORIGINS=http://localhost:5173,http://localhost:3000
OPENAI_API_KEY=sk-xxx
NEXFLUX_BACKEND_URL=http://localhost:8005/api/v1
```

---

## Frontend Integration

### React Hook Example

```typescript
import { useAIVision, useLabChat, useVisionStream } from '@/hooks/useAIVision';

function LabSession({ labId }) {
  const { detectComponents, validateStep, loading } = useAIVision();
  const { sendMessage } = useLabChat();
  const { connect, sendFrame, isConnected } = useVisionStream({
    labId,
    onFrame: (result) => {
      console.log('Detected:', result.components);
    },
    onValidation: (result) => {
      if (result.status === 'correct') {
        showSuccessMessage('Step completed!');
      }
    },
  });

  // Connect to stream on mount
  useEffect(() => {
    connect();
    return () => disconnect();
  }, []);

  // Send camera frame
  const handleFrame = (base64Image) => {
    sendFrame(base64Image, currentStep);
  };

  return (
    <div>
      {isConnected ? 'Connected' : 'Connecting...'}
      <Camera onFrame={handleFrame} />
    </div>
  );
}
```

---

## Changelog

### v1.0.0 (2026-01-22)
- Initial release
- Component detection API
- Step validation API
- Lab chat integration
- Real-time WebSocket streaming
- Usage analytics

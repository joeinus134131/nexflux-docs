# AI Studio Backend API Requirements

Dokumen ini menjelaskan spesifikasi backend API untuk fitur **AI Studio** - AI Design Assistant untuk membantu pengguna merancang rangkaian elektronik, generate kode, debug masalah hardware, dan memberikan rekomendasi komponen.

## Fitur Utama

1. **Chat dengan AI Assistant** - Percakapan interaktif untuk bantuan desain
2. **Circuit Design** - Generasi skema rangkaian berdasarkan prompt
3. **Code Generation** - Generate kode Arduino/ESP32 dari deskripsi
4. **Component Recommendations** - Rekomendasi komponen berdasarkan kebutuhan
5. **Debug Assistance** - Troubleshoot masalah hardware
6. **Chat History** - Menyimpan riwayat percakapan
7. **Context Awareness** - Integrasi dengan project, simulation, dan component yang ada

---

## Database Models

### AI Chat Session

```sql
CREATE TABLE ai_chat_sessions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title VARCHAR(255) NOT NULL DEFAULT 'New Chat',
    context_type VARCHAR(50), -- 'project', 'circuit', 'lab', NULL (general)
    context_id UUID,          -- ID of the related context
    board_type VARCHAR(100),  -- 'arduino_uno', 'esp32', etc.
    language VARCHAR(50),     -- 'c_cpp', 'micropython', etc.
    framework VARCHAR(100),   -- 'arduino_ide', 'platformio', etc.
    is_active BOOLEAN DEFAULT TRUE,
    message_count INTEGER DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_ai_chat_sessions_user ON ai_chat_sessions(user_id);
CREATE INDEX idx_ai_chat_sessions_context ON ai_chat_sessions(context_type, context_id);
```

### AI Chat Message

```sql
CREATE TABLE ai_chat_messages (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    session_id UUID NOT NULL REFERENCES ai_chat_sessions(id) ON DELETE CASCADE,
    role VARCHAR(20) NOT NULL CHECK (role IN ('user', 'assistant', 'system')),
    content TEXT NOT NULL,
    message_type VARCHAR(50) DEFAULT 'text', -- 'text', 'code', 'circuit_schema', 'component_list'
    metadata JSONB,                          -- Code language, component IDs, etc.
    tokens_used INTEGER DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_ai_chat_messages_session ON ai_chat_messages(session_id);
CREATE INDEX idx_ai_chat_messages_created ON ai_chat_messages(created_at DESC);
```

### AI Generated Content

```sql
CREATE TABLE ai_generated_content (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    message_id UUID REFERENCES ai_chat_messages(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    content_type VARCHAR(50) NOT NULL, -- 'circuit_schema', 'code', 'bom', 'tutorial'
    title VARCHAR(255),
    content JSONB NOT NULL,            -- The generated content
    is_saved BOOLEAN DEFAULT FALSE,
    project_id UUID REFERENCES projects(id) ON DELETE SET NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_ai_generated_user ON ai_generated_content(user_id);
CREATE INDEX idx_ai_generated_type ON ai_generated_content(content_type);
```

### AI Usage Stats

```sql
CREATE TABLE ai_usage_stats (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    date DATE NOT NULL DEFAULT CURRENT_DATE,
    request_count INTEGER DEFAULT 0,
    tokens_used INTEGER DEFAULT 0,
    code_generations INTEGER DEFAULT 0,
    circuit_designs INTEGER DEFAULT 0,
    UNIQUE(user_id, date)
);

CREATE INDEX idx_ai_usage_user_date ON ai_usage_stats(user_id, date);
```

---

## API Endpoints

### Base URL: `/api/v1/ai-studio`

---

### 1. Chat Sessions

#### GET `/sessions` - List User Sessions

Mendapatkan daftar chat session user.

**Response:**

```json
{
  "success": true,
  "data": {
    "sessions": [
      {
        "id": "uuid",
        "title": "Smart Home LED Controller",
        "context_type": "project",
        "context_id": "uuid",
        "board_type": "arduino_uno",
        "language": "c_cpp",
        "message_count": 12,
        "is_active": true,
        "created_at": "2026-01-20T10:00:00Z",
        "updated_at": "2026-01-20T12:30:00Z",
        "last_message_preview": "Bagaimana cara menambahkan sensor DHT22?"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 45
    }
  }
}
```

#### POST `/sessions` - Create New Session

Membuat chat session baru.

**Request:**

```json
{
  "title": "New Project Design",
  "context_type": "project", // optional: 'project', 'circuit', 'lab'
  "context_id": "uuid", // optional
  "board_type": "esp32", // optional
  "language": "micropython" // optional
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "title": "New Project Design",
    "board_type": "esp32",
    "language": "micropython",
    "created_at": "2026-01-20T15:00:00Z"
  }
}
```

#### GET `/sessions/:id` - Get Session with Messages

**Response:**

```json
{
  "success": true,
  "data": {
    "session": {
      "id": "uuid",
      "title": "Smart Home LED Controller",
      "board_type": "arduino_uno",
      "context": {
        "type": "project",
        "id": "uuid",
        "name": "Smart Home Project"
      }
    },
    "messages": [
      {
        "id": "uuid",
        "role": "assistant",
        "content": "Halo! 👋 Saya AI Design Assistant NexFlux...",
        "message_type": "text",
        "created_at": "2026-01-20T10:00:00Z"
      },
      {
        "id": "uuid",
        "role": "user",
        "content": "Saya ingin membuat LED controller",
        "message_type": "text",
        "created_at": "2026-01-20T10:01:00Z"
      }
    ]
  }
}
```

#### PUT `/sessions/:id` - Update Session

**Request:**

```json
{
  "title": "LED Controller v2",
  "board_type": "esp32"
}
```

#### DELETE `/sessions/:id` - Delete Session

---

### 2. Chat Messages

#### POST `/sessions/:id/messages` - Send Message to AI

Mengirim pesan ke AI dan mendapatkan respons.

**Request:**

```json
{
  "content": "Saya ingin membuat rangkaian LED yang bisa dikontrol via WiFi",
  "quick_action": null // optional: 'design_circuit', 'generate_code', 'get_ideas', 'debug_help'
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "user_message": {
      "id": "uuid",
      "role": "user",
      "content": "Saya ingin membuat rangkaian LED yang bisa dikontrol via WiFi",
      "created_at": "2026-01-20T10:05:00Z"
    },
    "assistant_message": {
      "id": "uuid",
      "role": "assistant",
      "content": "Bagus! Untuk LED controller via WiFi, saya sarankan:\n\n**Komponen yang dibutuhkan:**\n• ESP32 DevKit\n• LED RGB WS2812B\n• Level Shifter 3.3V to 5V\n• Power Supply 5V 2A\n\n**Estimasi biaya:** Rp 180.000\n\nMau saya buatkan skema rangkaiannya? 🔧",
      "message_type": "text",
      "metadata": {
        "suggested_components": ["esp32", "ws2812b", "level_shifter"],
        "estimated_cost": 180000,
        "has_follow_up_actions": [
          "generate_schema",
          "generate_code",
          "view_components"
        ]
      },
      "tokens_used": 245,
      "created_at": "2026-01-20T10:05:01Z"
    }
  }
}
```

---

### 3. Quick Actions

#### POST `/actions/design-circuit` - Generate Circuit Design

**Request:**

```json
{
  "session_id": "uuid", // optional, creates new if not provided
  "prompt": "LED dimmer dengan potentiometer",
  "board_type": "arduino_uno"
}
```

**Response:**

```json
{
    "success": true,
    "data": {
        "message": {
            "id": "uuid",
            "role": "assistant",
            "content": "Berikut adalah skema rangkaian LED dimmer dengan potentiometer...",
            "message_type": "circuit_schema"
        },
        "circuit": {
            "id": "uuid",
            "name": "LED Dimmer with Potentiometer",
            "schema_data": {
                "components": [...],
                "wires": [...]
            },
            "components": [
                {"name": "Arduino Uno", "quantity": 1},
                {"name": "LED 5mm", "quantity": 1},
                {"name": "Resistor 220Ω", "quantity": 1},
                {"name": "Potentiometer 10KΩ", "quantity": 1}
            ],
            "preview_url": "/uploads/circuit_previews/uuid.png"
        },
        "actions": [
            {"type": "open_simulator", "label": "Buka di Simulator"},
            {"type": "save_to_project", "label": "Simpan ke Project"},
            {"type": "generate_code", "label": "Generate Kode"}
        ]
    }
}
```

#### POST `/actions/generate-code` - Generate Code

**Request:**

```json
{
  "session_id": "uuid",
  "prompt": "Blink LED setiap 500ms dengan interrupt",
  "board_type": "arduino_uno",
  "language": "c_cpp",
  "circuit_id": "uuid" // optional, for context
}
```

**Response:**

````json
{
  "success": true,
  "data": {
    "message": {
      "id": "uuid",
      "role": "assistant",
      "content": "Berikut kode untuk LED blink dengan interrupt:\n\n```cpp\n// LED Blink dengan Interrupt\nvolatile bool ledState = false;\nconst int LED_PIN = 13;\n\nvoid setup() {\n  pinMode(LED_PIN, OUTPUT);\n  // Setup Timer1 interrupt\n  TCCR1A = 0;\n  TCCR1B = (1 << WGM12) | (1 << CS12);\n  OCR1A = 31249; // 500ms at 16MHz/256\n  TIMSK1 = (1 << OCIE1A);\n  sei();\n}\n\nISR(TIMER1_COMPA_vect) {\n  ledState = !ledState;\n  digitalWrite(LED_PIN, ledState);\n}\n\nvoid loop() {\n  // Main loop kosong - LED dikontrol via interrupt\n}\n```",
      "message_type": "code",
      "metadata": {
        "language": "cpp",
        "board": "arduino_uno",
        "lines": 23
      }
    },
    "code": {
      "id": "uuid",
      "language": "cpp",
      "content": "...",
      "filename": "led_blink_interrupt.ino"
    },
    "actions": [
      { "type": "copy_code", "label": "Copy" },
      { "type": "download", "label": "Download" },
      { "type": "run_simulation", "label": "Test di Simulator" },
      { "type": "save_to_project", "label": "Simpan ke Project" }
    ]
  }
}
````

#### POST `/actions/get-ideas` - Get Project Ideas

**Request:**

```json
{
  "session_id": "uuid",
  "category": "smart_home", // optional
  "difficulty": "beginner", // optional
  "board_type": "esp32" // optional
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "message": {
      "id": "uuid",
      "content": "Berikut ide project menarik untuk kamu:\n\n1. **Smart Plant Monitor** 🌱\n   - Sensor kelembaban tanah + ESP32\n   - Notifikasi via Telegram\n   - Difficulty: Beginner\n\n2. **WiFi Doorbell** 🔔\n   - Push notification ke smartphone\n   - Streaming kamera\n   - Difficulty: Intermediate"
    },
    "ideas": [
      {
        "title": "Smart Plant Monitor",
        "description": "Monitor kelembaban tanah dengan notifikasi via Telegram",
        "difficulty": "Beginner",
        "estimated_time": "2 jam",
        "components": ["esp32", "soil_moisture_sensor"],
        "tags": ["iot", "smart-home", "garden"]
      }
    ]
  }
}
```

#### POST `/actions/debug-help` - Debug Assistance

**Request:**

```json
{
  "session_id": "uuid",
  "problem": "LED tidak menyala meskipun kode sudah benar",
  "code": "...", // optional
  "circuit_id": "uuid", // optional
  "error_messages": ["..."] // optional
}
```

---

### 4. Component Recommendations

#### GET `/recommendations/components` - Get Component Recommendations

**Query Parameters:**

- `session_id` - Chat session for context
- `use_case` - Use case description
- `board_type` - Board type filter

**Response:**

```json
{
  "success": true,
  "data": {
    "recommendations": [
      {
        "component": {
          "id": "uuid",
          "name": "DHT22",
          "category": "Sensors",
          "price": 35000,
          "in_stock": true,
          "specs": {
            "type": "Temperature & Humidity",
            "range": "-40°C to 80°C"
          }
        },
        "reason": "Sensor suhu dan kelembaban dengan akurasi tinggi, cocok untuk monitoring environment",
        "compatibility": "Compatible with Arduino Uno, ESP32",
        "tutorials_available": 3
      }
    ],
    "estimated_total_cost": 175000
  }
}
```

---

### 5. Generated Content Management

#### GET `/content` - List Generated Content

**Response:**

```json
{
  "success": true,
  "data": {
    "content": [
      {
        "id": "uuid",
        "content_type": "circuit_schema",
        "title": "LED Dimmer Circuit",
        "is_saved": true,
        "project_id": "uuid",
        "preview_url": "/uploads/...",
        "created_at": "2026-01-20T10:00:00Z"
      }
    ]
  }
}
```

#### POST `/content/:id/save` - Save to Project

**Request:**

```json
{
  "project_id": "uuid",
  "circuit_name": "Main LED Controller" // optional, for circuit type
}
```

#### DELETE `/content/:id` - Delete Generated Content

---

### 6. Usage Stats

#### GET `/usage/stats` - Get User AI Usage Stats

**Response:**

```json
{
  "success": true,
  "data": {
    "today": {
      "request_count": 15,
      "tokens_used": 3500,
      "code_generations": 3,
      "circuit_designs": 2
    },
    "this_month": {
      "request_count": 245,
      "tokens_used": 58000,
      "code_generations": 42,
      "circuit_designs": 18
    },
    "limits": {
      "daily_requests": 50, // -1 for unlimited (Pro)
      "remaining_today": 35
    }
  }
}
```

---

### 7. Context Integration

#### GET `/context/project/:id` - Get Project Context

Mendapatkan context project untuk AI assistant.

**Response:**

```json
{
    "success": true,
    "data": {
        "project": {
            "id": "uuid",
            "name": "Smart Home Controller",
            "description": "...",
            "hardware_platform": "esp32"
        },
        "circuits": [...],
        "components": [...],
        "recent_simulations": [...]
    }
}
```

#### GET `/context/lab/:id` - Get Lab Context

Mendapatkan context Remote Lab untuk AI assistant.

---

## Access Control (ACL)

| Endpoint            | Free     | Pro       | Enterprise    |
| ------------------- | -------- | --------- | ------------- |
| Chat sessions       | 5 active | Unlimited | Unlimited     |
| Messages/day        | 20       | 200       | Unlimited     |
| Code generation     | 5/day    | 50/day    | Unlimited     |
| Circuit design      | 3/day    | 30/day    | Unlimited     |
| Saved content       | 10       | 100       | Unlimited     |
| Context integration | Basic    | Full      | Full + Custom |

---

## Integration Points

### 1. Circuit Simulator

- AI can generate circuits directly usable in simulator
- Apply AI-generated code to simulation

### 2. Projects

- Save AI content to projects
- Load project context for AI assistance

### 3. Components

- AI recommendations link to component library
- BOM generation with pricing

### 4. Remote Lab

- AI assistance during lab sessions
- Debugging help with real hardware

---

## Implementation Notes

### AI Provider Integration

Perlu implementasi adapter untuk AI provider:

- OpenAI GPT-4
- Anthropic Claude
- Google Gemini
- Open-source alternatives (Llama, Mistral)

### Prompt Engineering

Sistem prompt harus disesuaikan untuk:

- Electronics/IoT domain knowledge
- Code generation best practices
- Safety untuk hardware recommendations

### Rate Limiting

Implementasi rate limiting per tier:

- Request count per day
- Token usage per month
- Concurrent requests

# Remote Lab Communication Engine Architecture

This document outlines the communication architecture of the NexFlux Remote Lab, covering how code is compiled and uploaded, how the AI intercept engine operates, and how bidirectional serial monitoring functions.

## 1. High-Level Architecture Flow

The Remote Lab relies on three main communication layers:
1. **WebSocket / LiveKit (Video):** High-throughput, low-latency UDP/TCP connections for real-time video streaming from the lab hardware to the user's browser.
2. **REST API (Control & Compilation):** Standard HTTP endpoints for starting/ending sessions, requesting lab details, and submitting code for compilation.
3. **MQTT over WebSocket (Hardware Communication):** A lightweight pub/sub model for sending commands to the hardware (via `agent.js` on the lab station) and receiving real-time sensor data and serial logs.

---

## 2. Code Compilation & Upload Engine

When a user writes code in the Monaco Editor and clicks "Upload Code", the following sequence occurs:

### Frontend Flow (`LabSession.tsx`)
1. User clicks **Upload**.
2. Frontend sends user `code`, `session_id`, and `language` (e.g., `arduino` or `micropython`) to `POST /labs/{id}/code/submit`.
3. The API returns a `compilation_id`.
4. Frontend begins polling `GET /labs/{id}/code/status/{compilation_id}` every second to get real-time compiler output.

### Backend Flow (`lab.handler.go`)
1. **Validation:** Backend verifies the active session and validates the code payload.
2. **Compilation Queue:** The job is queued for the compilation worker.
3. **Compilation:** 
   - For Arduino: Backend invokes `arduino-cli compile --fqbn <board_target>`.
   - For ESP32/MicroPython: Backend prepares the appropriate binary or script flash process.
4. **Hardware Upload:** 
   - **Method A (Direct USB):** If the backend or `agent.js` has direct USB access to the board, it invokes the upload tool (e.g., `esptool.py` or `arduino-cli upload`).
   - **Method B (MQTT OTA):** If the hardware supports OTA (Over-The-Air) updates, the backend publishes the compiled binary path or download link to the device's MQTT command topic. The device downloads and flashes itself.
5. **Status Updates:** Throughout this process, the compilation status is updated in the database (`status: compiling | uploading | success | failed`), which the frontend polls.

---

## 3. Serial Monitoring (Bidirectional Communication)

The Serial Monitor must show live logs from the microcontroller and allow the user to send commands back to it. This relies heavily on the **MQTT Broker** and the intermediate **Agent**.

### The Flow
\`\`\`mermaid
sequenceDiagram
    participant User as User (Browser)
    participant MQTT as MQTT Broker
    participant Agent as Agent.js (Lab PC)
    participant MCU as MCU (Arduino/ESP32)

    Note over MCU, Agent: Serial Connection (USB Rx/Tx)
    MCU->>Agent: Serial.println("Hello")
    Note over Agent, MQTT: Publish to topic: nexflux/device/{id}/log
    Agent->>MQTT: Publish {"message": "Hello", "type": "info"}
    Note over User, MQTT: Subscribed via WebSocket
    MQTT->>User: Message Received & Displayed

    Note over User, MQTT: Publish to topic: nexflux/device/{id}/command
    User->>MQTT: Send Command: "LED:13:ON"
    MQTT->>Agent: Forward Command
    Note over Agent, MCU: Serial Write
    Agent->>MCU: Send over Serial: "LED:13:ON"
    MCU->>MCU: Parse & Execute
\`\`\`

### MQTT Topic Structure
- **Logs (Rx):** `nexflux/device/{id}/log` — Application logs and serial prints from the MCU.
- **Commands (Tx):** `nexflux/device/{id}/command` — Commands sent *to* the MCU.
- **Sensors (Rx):** `nexflux/device/{id}/sensors` — Structured JSON data representing current sensor states.
- **Actuators (Tx):** `nexflux/device/{id}/actuators` — Commands specifically for toggling hardware states (relays, motors).

### The `useMqtt.ts` Hook
The frontend manages this via the `useMqtt` hook, which establishes a WebSocket connection to the MQTT broker (e.g., Mosquitto or EMQX). It listens to the `log` and `sensors` topics and updates the React state, rendering real-time graphs and console output.

---

## 4. Security & Isolation

- **Sandbox:** User code is compiled in an isolated Docker container or constrained environment to prevent malicious code from accessing the host file system.
- **Rate Limiting:** Serial commands sent via MQTT are rate-limited on the frontend and filtered by the Agent to prevent flooding the microcontroller.
- **Timeouts:** Sessions are strictly time-bound. When `expiresAt` is reached, the backend forcefully disconnects the LiveKit token and drops MQTT subscriptions.

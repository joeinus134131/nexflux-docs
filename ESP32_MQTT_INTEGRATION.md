# NexFlux Remote Lab - ESP32 MQTT Integration Guide

This guide explains how the ESP32 Lab #1 integrates with the NexFlux frontend via MQTT.

## Architecture Overview

```
┌─────────────┐     Serial      ┌──────────────┐     MQTT      ┌─────────────┐
│   ESP32     │ ◄──────────────►│   Agent.js   │◄─────────────►│  Frontend   │
│  Hardware   │    USB/UART     │   (Laptop)   │   WebSocket   │  (Browser)  │
└─────────────┘                 └──────────────┘               └─────────────┘
```

## MQTT Topics

| Topic                                | Direction                | Description                  |
| ------------------------------------ | ------------------------ | ---------------------------- |
| `nexflux/device/{device_id}/command` | Frontend → Agent → ESP32 | Commands sent to hardware    |
| `nexflux/device/{device_id}/log`     | ESP32 → Agent → Frontend | Logs/responses from hardware |
| `nexflux/device/{device_id}/sensor`  | ESP32 → Agent → Frontend | Real-time sensor data        |
| `nexflux/device/{device_id}/status`  | Agent → Frontend         | Device online/offline status |

## Device ID Configuration

- ESP32 Lab #1 uses `device_id: "device-01"`
- This matches the `DEVICE_ID` constant in `agent.js`

## Command Protocol

Commands are sent as plain text strings. The ESP32 should parse these and respond accordingly.

### Display Commands

| Command     | Description              |
| ----------- | ------------------------ |
| `SHOW:text` | Display text on OLED/LCD |
| `CLEAR`     | Clear display            |

### LED Commands

| Command              | Description                    |
| -------------------- | ------------------------------ |
| `LED:pin:ON`         | Turn LED on at specified pin   |
| `LED:pin:OFF`        | Turn LED off at specified pin  |
| `BLINK:pin:interval` | Blink LED at interval (ms)     |
| `RGB:r:g:b`          | Set RGB LED color (0-255 each) |

### Actuator Commands

| Command              | Description                              |
| -------------------- | ---------------------------------------- |
| `SERVO:angle`        | Move servo to angle (0-180)              |
| `BUZZER:ON`          | Turn buzzer on                           |
| `BUZZER:OFF`         | Turn buzzer off                          |
| `TONE:freq:duration` | Play tone at frequency for duration (ms) |
| `RELAY:channel:ON`   | Turn relay on                            |
| `RELAY:channel:OFF`  | Turn relay off                           |

### Sensor Commands

| Command         | Description                  |
| --------------- | ---------------------------- |
| `READ:TEMP`     | Request temperature reading  |
| `READ:HUMIDITY` | Request humidity reading     |
| `READ:LIGHT`    | Request light sensor reading |
| `READ:DISTANCE` | Request ultrasonic distance  |
| `READ:ALL`      | Request all sensor readings  |

### System Commands

| Command  | Description                      |
| -------- | -------------------------------- |
| `PING`   | Health check (respond with PONG) |
| `STATUS` | Request system status            |
| `RESET`  | Soft reset device                |

## Expected Response Format

ESP32 should respond with JSON format for sensor data:

```json
{"type":"temperature","value":25.5,"unit":"°C"}
{"type":"humidity","value":60,"unit":"%"}
{"type":"status","value":"ok","uptime":12345}
```

For simple acknowledgments:

```json
{ "status": "ok", "command": "LED:13:ON" }
```

## Sample ESP32 Sketch

```cpp
#include <ArduinoJson.h>

void setup() {
  Serial.begin(115200);
  pinMode(13, OUTPUT);
  // Initialize other pins...
}

void loop() {
  if (Serial.available()) {
    String command = Serial.readStringUntil('\n');
    command.trim();
    processCommand(command);
  }

  // Periodically send sensor data
  static unsigned long lastSensor = 0;
  if (millis() - lastSensor > 2000) {
    sendSensorData();
    lastSensor = millis();
  }
}

void processCommand(String cmd) {
  // Parse command
  int colonIndex = cmd.indexOf(':');
  String action = colonIndex > 0 ? cmd.substring(0, colonIndex) : cmd;
  String params = colonIndex > 0 ? cmd.substring(colonIndex + 1) : "";

  if (action == "SHOW") {
    // Display text on OLED
    displayText(params);
    sendResponse("ok", cmd);
  }
  else if (action == "LED") {
    // Parse pin:state
    int pin = params.substring(0, params.indexOf(':')).toInt();
    bool state = params.endsWith("ON");
    digitalWrite(pin, state ? HIGH : LOW);
    sendResponse("ok", cmd);
  }
  else if (action == "RGB") {
    // Parse r:g:b
    int r = params.substring(0, params.indexOf(':')).toInt();
    String rest = params.substring(params.indexOf(':') + 1);
    int g = rest.substring(0, rest.indexOf(':')).toInt();
    int b = rest.substring(rest.indexOf(':') + 1).toInt();
    setRGB(r, g, b);
    sendResponse("ok", cmd);
  }
  else if (action == "READ") {
    if (params == "ALL") {
      sendSensorData();
    } else if (params == "TEMP") {
      sendTemperature();
    }
    // Handle other sensor reads...
  }
  else if (action == "PING") {
    Serial.println("{\"status\":\"PONG\"}");
  }
}

void sendResponse(String status, String cmd) {
  StaticJsonDocument<128> doc;
  doc["status"] = status;
  doc["command"] = cmd;
  serializeJson(doc, Serial);
  Serial.println();
}

void sendSensorData() {
  // Example: Send temperature
  StaticJsonDocument<128> doc;
  doc["type"] = "temperature";
  doc["value"] = readTemperature(); // Your sensor function
  doc["unit"] = "°C";
  serializeJson(doc, Serial);
  Serial.println();
}
```

## MQTT Broker Configuration

Your MQTT broker needs to support WebSocket for browser connections:

### Mosquitto Example (`mosquitto.conf`)

```
# TCP listener (for agent.js)
listener 1883
protocol mqtt

# WebSocket listener (for browser)
listener 9001
protocol websockets
```

## Agent.js Reference

The agent.js running on your laptop bridges Serial ↔ MQTT:

```javascript
const MQTT_URL = "mqtt://212.85.24.191";
const DEVICE_ID = "device-01";
const SERIAL_PORT = "/dev/cu.usbserial-A5069RR4";

// Serial → MQTT (hardware logs)
parser.on("data", (data) => {
  client.publish(`nexflux/device/${DEVICE_ID}/log`, data);
});

// MQTT → Serial (commands)
client.on("message", (topic, message) => {
  port.write(message.toString() + "\n");
});
```

## Testing

1. Start your MQTT broker with WebSocket enabled
2. Run agent.js on your laptop (connected to ESP32)
3. Open NexFlux frontend, go to Remote Lab → ESP32 Lab #1
4. Start Session - this will connect to MQTT
5. Use the Hardware Console to send commands
6. View logs from ESP32 in real-time

## Troubleshooting

### MQTT Connection Failed

- Ensure broker has WebSocket listener on port 9001
- Check VITE_MQTT_WS_URL in .env
- Verify broker is accessible from browser (CORS)

### Commands Not Reaching ESP32

- Check agent.js is running and connected
- Verify Serial communication works
- Check broker topic subscriptions

### No Logs in Console

- Ensure ESP32 outputs valid JSON
- Check agent.js is forwarding to log topic
- Verify topic names match device_id

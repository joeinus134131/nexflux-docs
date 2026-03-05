# Remote Lab AI Assistant 

This document details the functionality and architecture of the embedded AI Assistant in the NexFlux Remote Lab (`AIAssistantPanel.tsx`). The assistant is designed to provide contextual help, debug code, explain live streams, and offer a gamified learning experience for users engaged in hardware lab sessions.

## 1. Core Architecture

The AI Assistant is integrated directly into the `LabSession` via an interactive slide-in panel. It maintains conversational memory while continually receiving real-time context from the lab's state (code, sensors, actuators, and session metadata).

### Dependencies
- **Frontend Hook:** `useLabChat` (derived from `useAIStudio` or similar) handles the API communication and message history.
- **Backend Service:** A Python-based AI service (usually accessible at `POST /ai/chat/send`) powered by LLMs (e.g., Gemini or Claude) capable of parsing code and structural prompts.
- **State Integration:** The React component (`AIAssistantPanel.tsx`) maps the parent state (`code`, `sensors`, `labDetail`) into a serialized context string dynamically injected into system prompts.

## 2. Dynamic Context Injection

A critical feature of the AI Assistant is its ability to "see" what the user is doing. Before any query is sent to the LLM, the frontend prepares a **Dynamic Context String**:

\`\`\`typescript
const environmentContext = `
=== CURRENT LAB CONTEXT ===
📝 Lab: ${lab?.title || 'Unknown Lab'}
💻 Current Code Editor (${language}):
\`\`\`${language}
${code || '(empty)'}
\`\`\`
🌡️ Active Sensors: ${JSON.stringify(sensors)}
⚙️ Actuator States: ${JSON.stringify(actuators)}
=========================
`;
\`\`\`

This string is prepended to the user's message (as an invisible developer payload) or maintained as a strict system prompt instruction, guaranteeing that the AI answers based on the user's exact, up-to-the-second code and hardware state.

## 3. Interaction Modes

The assistant operates in four distinct persona-driven modes, toggled by the user in the UI. Each mode changes the underlying **System Prompt** injected into the chat request.

### A. 🎓 AI Guide
- **Purpose:** Step-by-step guidance, pedagogical support, and conceptual explanations.
- **Prompt Strategy:** The AI acts as a patient teacher. It is instructed to explain concepts behind the code (e.g., Ohm's Law, PWM, I2C routing) rather than just giving the final answer. It breaks down complex hardware logic into digestible pieces.

### B. 🐛 Debugger
- **Purpose:** Analyzing compilation errors and runtime misbehavior in user code.
- **Prompt Strategy:** The AI acts as an expert embedded systems engineer. It aggressively scans the provided code context for syntax errors, logical flaws (e.g., missing `pinMode()`, uninitialized variables in Arduino, or incorrect indentation in MicroPython), and suggests concrete fixes with snippet replacements.

### C. 🔥 Roasting
- **Purpose:** A gamified, humorous, and highly critical review of the user's code.
- **Prompt Strategy:** The AI takes on a sarcastic, snarky persona (akin to Gordon Ramsay reviewing code). It highlights bad practices, unstructured logic, or spaghetti code while still ultimately providing the correct solution buried in the critique.

### D. 📹 Stream Analysis (Live Session)
- **Purpose:** Explaining what is physically happening on the lab stream based on code and sensor data.
- **Prompt Strategy:** Since the AI cannot directly view the LiveKit UDP video stream, it operates as a "Director of Telemetry". It infers the physical state of the board by analyzing the sensor readings and actuator commands. E.g., "Based on your code setting Pin 13 to HIGH and the photoresistor value dropping, the LED on the stream is currently emitting light."

## 4. Fallback Mechanisms

If the backend AI service is unavailable (HTTP 500 or timeout), the frontend includes a graceful degradation fallback inside `useLabChat`. It generates simulated, context-aware responses (e.g., "I see your C++ code. Here is a simulated response indicating a potential missing semicolon on line 42.") to ensure the user experience is not entirely broken during service outages.

# NexFlux LiveKit Setup Guide

Setup LiveKit server for real-time video streaming in NexFlux Virtual Lab.

## Quick Start

### 1. Start LiveKit Server

```bash
cd docker/livekit
./start.sh
```

Or manually:

```bash
cd docker/livekit
docker-compose up -d
```

### 2. Verify LiveKit is Running

```bash
curl http://localhost:7880
```

You should see a response.

### 3. Configure Environment

#### Backend (.env)
```env
LIVEKIT_URL=ws://localhost:7880
LIVEKIT_API_KEY=devkey
LIVEKIT_API_SECRET=secret
```

#### Frontend (.env)
```env
VITE_LIVEKIT_URL=ws://localhost:7880
VITE_SKIP_LIVEKIT=false
```

## Architecture

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Frontend      │────▶│  Go Backend     │────▶│  LiveKit Server │
│   (React)       │     │  (Token Gen)    │     │  (Docker)       │
└─────────────────┘     └─────────────────┘     └─────────────────┘
        │                                               │
        │                                               │
        └───────────────WebRTC──────────────────────────┘
```

## Flow

1. **User opens Lab Session**
   - Frontend calls `/api/v1/livekit/labs/:lab_id/token`
   - Backend generates JWT token with room permissions

2. **Connect to LiveKit**
   - Frontend uses `useLiveKit()` hook
   - Connects to LiveKit server with token

3. **Enable Camera**
   - User grants camera permission
   - Video stream is published to room

4. **AI Analysis** (optional)
   - `captureFrame()` gets current video frame
   - Frame is sent to AI Vision API for analysis

## API Endpoints

### Get Token for Lab Room
```http
GET /api/v1/livekit/labs/:lab_id/token
Authorization: Bearer <jwt>
```

Response:
```json
{
  "success": true,
  "data": {
    "token": "eyJ...",
    "room_name": "lab-uuid",
    "identity": "user-uuid",
    "livekit_url": "ws://localhost:7880"
  }
}
```

### Generate Custom Token
```http
POST /api/v1/livekit/token
Authorization: Bearer <jwt>
Content-Type: application/json

{
  "room_name": "custom-room",
  "can_publish": true,
  "can_subscribe": true
}
```

## Frontend Usage

### Using LiveKit Room
```tsx
import { useLiveKit } from '@/hooks/useLiveKit';

function LabSession({ labId }) {
  const { 
    roomState, 
    connect, 
    disconnect, 
    enableCamera,
    captureFrame 
  } = useLiveKit();
  
  const videoRef = useRef<HTMLVideoElement>(null);

  useEffect(() => {
    if (videoRef.current) {
      connect(labId, videoRef.current);
    }
    return () => disconnect();
  }, [labId]);

  return (
    <video 
      ref={videoRef} 
      autoPlay 
      playsInline 
      muted 
    />
  );
}
```

### Local Camera Only (No LiveKit Server)
```tsx
import { useLocalCamera } from '@/hooks/useLiveKit';

function CameraPreview() {
  const { enableCamera, disableCamera, captureFrame } = useLocalCamera();
  const videoRef = useRef<HTMLVideoElement>(null);

  useEffect(() => {
    if (videoRef.current) {
      enableCamera(videoRef.current);
    }
    return () => disableCamera();
  }, []);

  return <video ref={videoRef} autoPlay playsInline muted />;
}
```

## Docker Configuration

### docker-compose.yml
Located at: `docker/livekit/docker-compose.yml`

### livekit.yaml
Located at: `docker/livekit/livekit.yaml`

### Ports
| Port | Purpose |
|------|---------|
| 7880 | HTTP/WebSocket API |
| 7881 | RTC (WebRTC) |
| 7882 | TURN/UDP |

## Troubleshooting

### Camera Not Working

1. **Check Browser Permissions**
   - Ensure camera permission is granted
   - Try in incognito mode if blocked

2. **Check Console for Errors**
   ```
   Failed to access camera. Please check permissions.
   ```

3. **macOS Camera Issues**
   - Go to System Preferences > Security & Privacy > Camera
   - Ensure browser is allowed

### LiveKit Connection Failed

1. **Check if Docker is running**
   ```bash
   docker ps
   ```

2. **Check LiveKit logs**
   ```bash
   docker-compose logs livekit
   ```

3. **Verify URL**
   - Use `ws://` for development, `wss://` for production

### Token Generation Failed

1. **Check backend logs for errors**
2. **Verify LIVEKIT_API_KEY and LIVEKIT_API_SECRET match**

## Production Considerations

1. **Use SSL/TLS**
   - Change `ws://` to `wss://`
   - Configure proper certificates

2. **Update API Keys**
   - Generate strong random keys
   - Never use "devkey" in production

3. **Configure TURN Server**
   - For NAT traversal in production

4. **Scale with Redis**
   - Uncomment Redis in docker-compose
   - Configure livekit.yaml for Redis

## Related Files

- `docker/livekit/docker-compose.yml` - Docker configuration
- `docker/livekit/livekit.yaml` - LiveKit server config
- `nexflux-backend/api/handlers/livekit.handler.go` - Token generation
- `nexflux-frontend/src/hooks/useLiveKit.ts` - Frontend hooks

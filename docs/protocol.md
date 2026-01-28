# WebSocket Protocol

## Connection

### Handshake

Connect to the RALF Gesture Studio WebSocket endpoint:

```
wss://your-server.com/motion
```

Optional authentication via query parameter or header:
```
wss://your-server.com/motion?token=<auth_token>
```

### Connection Message

Upon connection, client sends device info:

```json
{
  "type": "connect",
  "client": {
    "app": "gesture-input-ios",
    "version": "1.0.0",
    "devices": ["iphone", "watch"],
    "capabilities": {
      "maxSampleRate": 100,
      "sensors": ["accelerometer", "gyroscope", "attitude"],
      "onDeviceGestures": false
    }
  }
}
```

### Server Acknowledgment

```json
{
  "type": "connected",
  "sessionId": "abc123",
  "config": {
    "requestedSampleRate": 50,
    "sensors": ["accelerometer", "attitude"]
  }
}
```

## Message Types

### Client → Server

#### Motion Data

Sent at configured sample rate (e.g., 50-100 Hz):

```json
{
  "type": "motion",
  "device": "iphone",
  "seq": 12345,
  "t": 1706472834.123,
  "acc": [0.02, -0.98, 0.15],
  "gyro": [0.01, -0.03, 0.12],
  "att": [0.05, -0.12, 1.34],
  "rot": [0.01, -0.02, 0.08],
  "uacc": [0.02, 0.01, -0.03]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | Always "motion" |
| `device` | string | "iphone" or "watch" |
| `seq` | int | Sequence number for ordering |
| `t` | float | Unix timestamp (seconds, ms precision) |
| `acc` | [x,y,z] | Accelerometer (g) |
| `gyro` | [x,y,z] | Gyroscope (rad/s) |
| `att` | [roll,pitch,yaw] | Attitude (radians) |
| `rot` | [x,y,z] | Rotation rate (rad/s) |
| `uacc` | [x,y,z] | User acceleration (g, gravity removed) |

**Compact Format Option:**

For high-rate streaming, use binary or abbreviated JSON:

```json
{"m":"i","s":12345,"t":1706472834.123,"a":[0.02,-0.98,0.15],"g":[0.01,-0.03,0.12]}
```

#### Gesture Event (On-Device Detection)

```json
{
  "type": "gesture",
  "device": "watch",
  "t": 1706472834.456,
  "gesture": "flick_right",
  "confidence": 0.92,
  "duration": 0.25
}
```

#### Ping

```json
{
  "type": "ping",
  "t": 1706472834.789
}
```

### Server → Client

#### Pong

```json
{
  "type": "pong",
  "t": 1706472834.789,
  "serverTime": 1706472834.801
}
```

#### Configuration Update

Server can request configuration changes:

```json
{
  "type": "config",
  "sampleRate": 50,
  "sensors": ["accelerometer", "attitude"],
  "enableGestures": true
}
```

#### Gesture Feedback

Server-detected gesture feedback (optional):

```json
{
  "type": "gesture_detected",
  "gesture": "circle_cw",
  "confidence": 0.87,
  "feedback": "haptic_light"
}
```

Client should trigger haptic feedback if specified.

#### Error

```json
{
  "type": "error",
  "code": "rate_limit",
  "message": "Sample rate too high, throttling to 50 Hz"
}
```

## Batching (Optional)

For efficiency, client may batch multiple samples:

```json
{
  "type": "motion_batch",
  "device": "iphone",
  "samples": [
    {"s":100,"t":1706472834.000,"a":[0.02,-0.98,0.15],"att":[0.05,-0.12,1.34]},
    {"s":101,"t":1706472834.010,"a":[0.03,-0.97,0.14],"att":[0.06,-0.11,1.35]},
    {"s":102,"t":1706472834.020,"a":[0.02,-0.99,0.16],"att":[0.05,-0.12,1.34]}
  ]
}
```

Recommended batch size: 10-20 samples (100-200ms at 100 Hz)

## Reconnection

1. On disconnect, client waits 1 second then reconnects
2. Exponential backoff: 1s, 2s, 4s, 8s, max 30s
3. On reconnect, send fresh `connect` message
4. Resume with new sequence numbers

## Heartbeat

- Client sends `ping` every 30 seconds if no other traffic
- Server responds with `pong`
- If no `pong` within 10 seconds, client reconnects

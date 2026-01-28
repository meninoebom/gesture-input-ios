# Architecture

## System Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              RALF Gesture Studio                             │
│                            (Hosted Backend Server)                           │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────────┐  │
│  │ WebSocket Server│  │ Motion Buffer   │  │ Gesture Recognition Engine  │  │
│  │  (receives data)│─▶│ (time-series)   │─▶│ (pattern matching / ML)     │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ▲
                                    │ WebSocket (wss://)
                                    │
┌───────────────────────────────────┼───────────────────────────────────────┐
│                              iPhone App                                    │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────┐    │
│  │  MotionManager  │  │  DataFormatter  │  │   WebSocketManager      │    │
│  │  (Core Motion)  │─▶│  (JSON encode)  │─▶│   (Starscream/URLSession│────┼──┘
│  └─────────────────┘  └─────────────────┘  └─────────────────────────┘    │
│          ▲                                                                 │
│          │ WatchConnectivity                                               │
│          │                                                                 │
└──────────┼─────────────────────────────────────────────────────────────────┘
           │
┌──────────┼─────────────────────────────────────────────────────────────────┐
│          │                    Apple Watch App                              │
│  ┌───────▼───────┐  ┌─────────────────┐  ┌─────────────────────────────┐  │
│  │ MotionManager │  │ WCSession       │  │ Optional: Direct WebSocket  │  │
│  │ (800Hz batch) │─▶│ (to iPhone)     │  │ (when iPhone unavailable)   │  │
│  └───────────────┘  └─────────────────┘  └─────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Core Components

### 1. MotionManager

Wraps `CMMotionManager` and `CMBatchedSensorManager` for sensor access.

```swift
class MotionManager: ObservableObject {
    private let motionManager = CMMotionManager()

    @Published var isRunning = false
    @Published var currentMotion: MotionData?

    var sampleRate: Double = 100.0  // Hz

    func startUpdates(handler: @escaping (MotionData) -> Void)
    func stopUpdates()
}
```

**Responsibilities:**
- Configure sensor sample rates
- Handle sensor availability checks
- Provide unified motion data model
- Support both real-time and batched modes

### 2. WebSocketManager

Handles connection to RALF backend.

```swift
class WebSocketManager: ObservableObject {
    @Published var connectionState: ConnectionState = .disconnected
    @Published var lastError: Error?

    func connect(to url: URL)
    func disconnect()
    func send(_ data: Encodable)
}
```

**Responsibilities:**
- Establish/maintain WebSocket connection
- Auto-reconnect on disconnect
- Queue messages during brief disconnects
- Report connection health

### 3. DataFormatter

Converts motion data to wire format.

```swift
struct DataFormatter {
    static func encode(_ motion: MotionData) -> Data
    static func encode(_ gesture: GestureEvent) -> Data
}
```

**Responsibilities:**
- JSON encoding
- Timestamp normalization
- Optional compression for high-rate data

### 4. GestureDetector (Phase 3)

On-device gesture recognition.

```swift
class GestureDetector {
    func process(_ motion: MotionData) -> GestureEvent?
    func loadModel(_ url: URL) throws
}
```

**Responsibilities:**
- Buffer recent samples
- Run inference on Create ML model
- Debounce repeated detections

## Data Flow

### Normal Operation (Phone)

```
1. CMMotionManager generates device motion at 100 Hz
2. MotionManager packages into MotionData struct
3. DataFormatter encodes to JSON
4. WebSocketManager sends to server
5. Server processes and responds (optional)
```

### Watch Relay Mode

```
1. Watch CMMotionManager captures at up to 800 Hz (batched)
2. Watch app sends via WCSession.transferUserInfo() or sendMessage()
3. iPhone receives and merges with local data
4. Combined stream sent to server with device identifiers
```

### Direct Watch Mode (WiFi)

```
1. Watch connects directly to server (requires WiFi)
2. Lower rate due to watch power constraints (50 Hz recommended)
3. Useful when phone is unavailable
```

## Threading Model

```
┌─────────────────────────────────────────────────────────────┐
│                        Main Thread                          │
│  - UI updates                                               │
│  - @Published property changes                              │
└─────────────────────────────────────────────────────────────┘
                              ▲
                              │
┌─────────────────────────────┼───────────────────────────────┐
│                    Motion Queue (serial)                    │
│  - CMMotionManager callbacks                                │
│  - Data formatting                                          │
│  - Pass to WebSocket queue                                  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   WebSocket Queue (serial)                  │
│  - Send operations                                          │
│  - Receive handling                                         │
│  - Reconnection logic                                       │
└─────────────────────────────────────────────────────────────┘
```

## Error Handling

| Scenario | Behavior |
|----------|----------|
| Server unreachable | Retry with exponential backoff (1s, 2s, 4s, max 30s) |
| Connection lost | Attempt reconnect, queue up to 100 samples |
| Queue overflow | Drop oldest samples, log warning |
| Sensor unavailable | Show UI error, disable streaming |
| Watch disconnect | Continue with phone-only mode |

## Configuration

```swift
struct AppConfiguration {
    // Server
    var serverURL: URL = URL(string: "ws://localhost:8080/motion")!
    var autoReconnect: Bool = true
    var reconnectDelay: TimeInterval = 1.0

    // Motion
    var sampleRate: Double = 100.0  // Hz
    var includeAccelerometer: Bool = true
    var includeGyroscope: Bool = true
    var includeAttitude: Bool = true
    var includeUserAcceleration: Bool = true

    // Watch
    var useWatchRelay: Bool = true
    var watchSampleRate: Double = 100.0

    // Gestures
    var enableOnDeviceDetection: Bool = false
    var gestureModelURL: URL? = nil
}
```

## Security Considerations

1. **Transport**: Use `wss://` (TLS) in production
2. **Authentication**: Support bearer token in WebSocket handshake
3. **Privacy**: No location or personal data collected, only motion vectors
4. **Permissions**: Request motion permission with clear usage description

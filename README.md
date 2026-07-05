# Gesture Input iOS

> **ARCHIVED (July 2026).** Planning-only repo; no code was built. SOMI-1 wearables (shipped, sensors live) now fill the wearable-IMU input role. Plans remain readable here if iPhone/Watch input is ever revived.

Stream IMU sensor data from iPhone and Apple Watch to RALF Gesture Studio via WebSocket for real-time gesture recognition.

## Overview

This app captures motion data from device sensors (accelerometer, gyroscope, device motion) and streams it in real-time to a hosted backend. It enables gesture-based input without cameras or specialized hardware.

## Features

### Phase 1: iPhone Streaming
- Stream accelerometer data (up to 100 Hz)
- Stream gyroscope data (up to 100 Hz)
- Stream fused device motion (attitude, rotation rate, user acceleration)
- WebSocket connection to configurable server
- Connection status and diagnostics UI

### Phase 2: Apple Watch Integration
- WatchOS companion app
- Stream watch IMU data via phone relay
- Independent watch-to-server streaming (WiFi)
- Multi-device data fusion

### Phase 3: On-Device Gesture Detection
- Basic gesture recognition (shake, flick, rotate)
- Create ML activity classification model
- Gesture event streaming (vs raw data)
- Configurable gesture vocabulary

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        iPhone App                                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │ Core Motion │  │  Gesture    │  │    WebSocket Manager    │  │
│  │   Manager   │─▶│  Detector   │─▶│  (configurable server)  │──┼──▶ RALF
│  └─────────────┘  └─────────────┘  └─────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
         ▲
         │ WatchConnectivity
         │
┌─────────────────────┐
│   Apple Watch App   │
│  ┌───────────────┐  │
│  │  Core Motion  │  │
│  │   (800 Hz)    │  │
│  └───────────────┘  │
└─────────────────────┘
```

## Data Format

### Raw Sensor Stream (JSON over WebSocket)

```json
{
  "type": "motion",
  "device": "iphone",
  "timestamp": 1706472834.123,
  "accelerometer": {
    "x": 0.02,
    "y": -0.98,
    "z": 0.15
  },
  "gyroscope": {
    "x": 0.01,
    "y": -0.03,
    "z": 0.12
  },
  "attitude": {
    "roll": 0.05,
    "pitch": -0.12,
    "yaw": 1.34
  },
  "rotationRate": {
    "x": 0.01,
    "y": -0.02,
    "z": 0.08
  },
  "userAcceleration": {
    "x": 0.02,
    "y": 0.01,
    "z": -0.03
  }
}
```

### Gesture Event (when on-device detection enabled)

```json
{
  "type": "gesture",
  "device": "watch",
  "timestamp": 1706472834.456,
  "gesture": "flick_right",
  "confidence": 0.92
}
```

## Technical Specifications

| Specification | Value |
|---------------|-------|
| Min iOS Version | 16.0 |
| Min watchOS Version | 9.0 |
| Sample Rate (real-time) | 50-100 Hz configurable |
| Sample Rate (batched) | Up to 800 Hz |
| WebSocket Protocol | RFC 6455 |
| Data Format | JSON |

## Project Structure

```
gesture-input-ios/
├── GestureInput/                 # iOS app
│   ├── App/
│   │   └── GestureInputApp.swift
│   ├── Core/
│   │   ├── MotionManager.swift      # Core Motion wrapper
│   │   ├── WebSocketManager.swift   # WebSocket client
│   │   └── GestureDetector.swift    # On-device detection
│   ├── Models/
│   │   ├── MotionData.swift
│   │   └── GestureEvent.swift
│   ├── Views/
│   │   ├── ContentView.swift
│   │   ├── ConnectionView.swift
│   │   └── DiagnosticsView.swift
│   └── Resources/
├── GestureInputWatch/            # watchOS app
│   ├── App/
│   ├── Core/
│   └── Views/
├── GestureInputTests/
└── docs/
    ├── architecture.md
    └── protocol.md
```

## Getting Started

### Prerequisites

- Xcode 15+
- iOS 16+ device (simulator doesn't have motion sensors)
- Apple Watch (optional, for Phase 2)

### Installation

1. Clone the repository
2. Open `GestureInput.xcodeproj` in Xcode
3. Configure signing with your Apple Developer account
4. Build and run on a physical device

### Configuration

Set the WebSocket server URL in Settings or via environment:

```swift
// Default development server
let serverURL = "ws://localhost:8080/motion"
```

## Development Roadmap

See [GitHub Issues](../../issues) for the detailed roadmap.

### Milestones

1. **v0.1 - Basic Streaming** - iPhone IMU to WebSocket
2. **v0.2 - Watch Support** - Apple Watch data relay
3. **v0.3 - Gesture Detection** - On-device ML classification
4. **v1.0 - Production Ready** - Polished UI, error handling, documentation

## Related Projects

- [RALF Gesture Studio](../ralf) - Backend gesture processing
- [sensor-input-approaches.md](../docs/sensor-input-approaches.md) - Research on input methods

## References

- [Core Motion Documentation](https://developer.apple.com/documentation/coremotion/)
- [WWDC23: What's New in Core Motion](https://developer.apple.com/videos/play/wwdc2023/10179/)
- [Create ML Activity Classification](https://developer.apple.com/videos/play/wwdc2019/426/)

## License

MIT

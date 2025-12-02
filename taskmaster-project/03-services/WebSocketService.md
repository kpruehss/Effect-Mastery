# WebSocketService Implementation

**Real-time updates with WebSocket**

Complete implementation from the original project file - see lines 649-713 of `effectts-svelte5-complete-project.md`.

## Key Features

- WebSocket connection with automatic cleanup
- Stream-based message handling
- Send messages to server
- Queue for message buffering

## Scoped Layer Pattern

```typescript
const WebSocketServiceLive = Layer.scoped(
  WebSocketService,
  makeWebSocketService
)
```

Uses `Layer.scoped` to ensure WebSocket is properly closed when no longer needed.

[View complete implementation →](../effectts-svelte5-complete-project.md#websocket-service)

---

[← ProjectService](./ProjectService.md) | [Back to Services](./README.md)

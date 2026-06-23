# WebSocket — Full-Duplex Persistent Connections

**Source:** Technical notes (pasted content)
**Saved:** 2026-06-17
**Tags:** javascript, infrastructure, fundamentals, technology, tools

> Read alongside: `javascript/infrastructure/sse-server-sent-events.md` — these two notes form a pair covering the two main options for persistent server-client communication.

---

## TL;DR
WebSocket opens a persistent, bidirectional TCP connection between browser and server via an HTTP upgrade handshake. Once open, either side sends messages at any time with no per-message HTTP overhead. Best fit for true real-time bidirectional use cases (chat, live updates, collaborative editing, bidding systems). Tradeoffs: no auto-reconnect, requires stateful server infrastructure (can't use stateless serverless functions), occasionally blocked by corporate proxies.

## Key Concepts & Terms
- **`Upgrade: websocket` header**: The HTTP header that initiates the protocol switch. The browser sends a regular HTTP request with this header; if the server supports WebSocket, it responds with `101 Switching Protocols` and the HTTP connection is promoted to a WebSocket connection.
- **101 Switching Protocols**: The HTTP status code that confirms the upgrade. After this, the connection is no longer HTTP — it's a WebSocket TCP channel.
- **Framing**: WebSocket messages are sent as "frames" on the open TCP connection — no HTTP headers, no handshake per message. This is the source of the low latency.
- **Stateful connection**: The server must hold the open connection in memory for each connected client. This means a single persistent process (Node.js server, dedicated WebSocket server) rather than a stateless function that spins up per request.
- **No auto-reconnect**: Unlike `EventSource` (SSE), WebSocket does not automatically reconnect on disconnect. The application is responsible for detecting closure and re-establishing the connection.
- **Binary + text**: WebSocket supports both text frames (JSON strings) and binary frames (ArrayBuffer, Blob) — unlike SSE which is text-only.

## How It Works

```
Browser sends:
  GET /chat HTTP/1.1
  Upgrade: websocket
  Connection: Upgrade
  Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==

Server responds:
  HTTP/1.1 101 Switching Protocols
  Upgrade: websocket
  Connection: Upgrade
  Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=

[HTTP connection is now a WebSocket — both sides send freely]
```

## JavaScript Implementation

```javascript
const ws = new WebSocket('wss://api.example.com/chat?token=abc');

ws.onopen = () => {
  // Connection established — safe to send
  ws.send(JSON.stringify({ type: 'join', room: 'general' }));
};

ws.onmessage = (event) => {
  const message = JSON.parse(event.data);
  console.log(message);
};

ws.onclose = (event) => {
  console.log('Closed:', event.code, event.reason);
  // Auto-reconnect logic goes here — not built in
};

ws.onerror = () => {
  // Note: onerror gives minimal info; the real detail is in onclose
};

ws.close(); // Graceful close
```

**Sending binary:**
```javascript
const buffer = new ArrayBuffer(8);
ws.send(buffer); // Binary frame
```

**Reconnection pattern (manual):**
```javascript
function connect() {
  const ws = new WebSocket('wss://api.example.com/stream');

  ws.onclose = () => {
    // Reconnect after 1 second (add backoff for production)
    setTimeout(connect, 1000);
  };

  return ws;
}
```

## Server-Side Pattern (Node.js with `ws` library)

```javascript
import { WebSocketServer } from 'ws';

const wss = new WebSocketServer({ port: 8080 });

wss.on('connection', (ws, req) => {
  // ws = one connected client
  // req = the original HTTP upgrade request (for auth, headers)

  ws.on('message', (data) => {
    const message = JSON.parse(data.toString());
    // Broadcast to all connected clients
    wss.clients.forEach((client) => {
      if (client.readyState === WebSocket.OPEN) {
        client.send(JSON.stringify({ from: 'server', ...message }));
      }
    });
  });

  ws.on('close', () => {
    // Clean up client state
  });

  // Send immediately on connect
  ws.send(JSON.stringify({ type: 'welcome' }));
});
```

## The Three Metaphors (Comparison)

> **fetch** = walkie-talkie: one side talks, then the other. Each exchange is independent.
> **SSE (EventSource)** = radio broadcast: server transmits continuously, client listens. One direction.
> **WebSocket** = phone call: both sides talk freely at any time on one open line.

## WebSocket vs SSE — Decision Guide

| Question | If YES → | If NO → |
|----------|---------|---------|
| Does the client need to push data at high frequency? | WebSocket | SSE probably fine |
| Is the client behind corporate firewall/proxy? | SSE safer | Either works |
| Do you need binary data? | WebSocket | Either works |
| Is your server stateless (Lambda/serverless)? | Can't use WebSocket | Either works |
| Do you need built-in reconnection? | SSE | WebSocket (manual reconnect) |
| Is it a streaming-output-only use case (LLM tokens, notifications)? | SSE simpler | WebSocket if bidirectional needed |

## Infrastructure Requirements

WebSocket requires a **stateful server** — a long-running process that holds open connections in memory. This rules out:
- AWS Lambda (function terminates after response)
- Vercel serverless functions
- Most "serverless" compute that scales to zero

What works:
- Node.js long-running server (Express + `ws` library, Socket.io)
- Dedicated WebSocket server (e.g., Ably, Pusher, AWS API Gateway WebSocket)
- Containerized backend (ECS, EC2, Fly.io, Railway)

The stateful requirement also means horizontal scaling needs coordination — if clients connect to different server instances, messages sent to one instance need to be broadcast across all instances. Common solution: pub/sub layer (Redis Pub/Sub, Ably channels) between server instances.

## Production Edge Cases

**Corporate proxy blocking**: Some proxies don't understand the `101 Switching Protocols` response and drop the connection. SSE (plain HTTP) is the fallback for restricted networks. Socket.io handles this with automatic fallback from WebSocket to long-polling.

**Heartbeat / ping-pong**: WebSocket connections can silently die without either side knowing (network interruption without a clean close). Implement periodic ping frames from the server to detect dead connections:
```javascript
// Server-side keep-alive
setInterval(() => {
  wss.clients.forEach((ws) => {
    if (ws.isAlive === false) return ws.terminate();
    ws.isAlive = false;
    ws.ping();
  });
}, 30000);
ws.on('pong', () => { ws.isAlive = true; });
```

**Auth on the URL**: Like SSE, custom headers aren't supported in the browser `WebSocket` constructor — tokens must go in the URL query string or cookies. Short-lived tokens are recommended.

**`readyState`**: Always check before sending:
```javascript
if (ws.readyState === WebSocket.OPEN) {
  ws.send(data);
}
// States: CONNECTING=0, OPEN=1, CLOSING=2, CLOSED=3
```

## Questions & Gaps
- For horizontal scaling with multiple server instances: what's the latency overhead of a Redis Pub/Sub relay vs a dedicated WebSocket service like Ably? At what connection count does managed infrastructure become cost-effective?
- Socket.io adds automatic fallback (WebSocket → long-polling), rooms, namespaces, and reconnection. At what complexity threshold does adding Socket.io's overhead make sense vs raw `ws`?
- How does the WebSocket protocol handle message ordering? Are messages guaranteed to arrive in send order, or can frames be reordered?

## Related Notes
- [SSE (Server-Sent Events)](https://github.com/LutherCalvinRiggs/research/blob/main/javascript/infrastructure/sse-server-sent-events.md) — the direct counterpart. Read both together: SSE for server-push streaming use cases, WebSocket for bidirectional real-time. The "three metaphors" section spans both notes.
- [JWT Token Refresh Architecture](https://github.com/LutherCalvinRiggs/research/blob/main/javascript/infrastructure/jwt-token-refresh-architecture.md) — WebSocket auth tokens in the URL have the same expiry problem as SSE tokens. The 80% timer refresh pattern applies: the token in `wss://...?token=abc` should be short-lived and refreshed before the WebSocket is opened with it.
- [Node.js 8 Production Patterns](https://github.com/LutherCalvinRiggs/research/blob/main/javascript/infrastructure/nodejs-8-production-patterns.md) — the graceful shutdown pattern is critical for WebSocket servers: on `SIGTERM`, drain open connections before closing rather than terminating them immediately.

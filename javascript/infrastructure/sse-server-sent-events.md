# SSE (Server-Sent Events) — Architecture and Implementation

**Source:** Technical notes (pasted content)
**Saved:** 2026-06-17
**Tags:** javascript, infrastructure, fundamentals, technology, tools

---

## TL;DR
SSE is a one-way persistent HTTP stream from server to client. The browser opens an `EventSource` connection; the server keeps it alive and pushes newline-delimited text whenever it has data. Built-in auto-reconnect, works through firewalls that block WebSockets, text-only. Best fit for streaming responses (e.g., LLM output word-by-word) where the client doesn't need to send data back over the same connection — client sends messages via a separate `fetch` POST instead.

## Key Concepts & Terms
- **EventSource**: A browser-native API (alongside `fetch` and `WebSocket`) that opens a persistent HTTP GET connection and fires `onmessage` events as the server pushes data. The browser handles reconnection automatically if the connection drops.
- **`text/event-stream`**: The Content-Type header that signals to the browser "this HTTP response will never finish — keep reading it." The server responds with this header and then writes data incrementally.
- **Wire format**: Plain text lines prefixed with `data: `. A blank line signals end of one message. Simple enough to write by hand.
- **Auto-reconnect**: Built into EventSource — unlike WebSocket, you don't need to implement reconnection logic yourself. The browser retries automatically. The `Last-Event-ID` header can be used to resume from where it left off.
- **SSE Transport**: The pattern of using SSE for server→client data plus a separate `fetch POST` for client→server data. Commonly used in LLM chat interfaces (stream tokens from server; send user messages via POST).

## The Wire Format

```
data: Hello world

data: Here's another message

data: [DONE]
```

Each `data:` line becomes one `onmessage` event. The blank line between them signals "end of this message." Named events can also be sent:

```
event: status
data: {"state": "thinking"}

data: First token of the response
```

## JavaScript Implementation

```javascript
const es = new EventSource('https://api.example.com/stream?token=abc');

es.onmessage = (event) => {
  // event.data = "Hello world", "Here's another message", etc.
  console.log(event.data);
  
  if (event.data === '[DONE]') {
    es.close(); // Clean up when the stream is finished
  }
};

es.addEventListener('status', (event) => {
  // Named events are handled separately
  const status = JSON.parse(event.data);
});

es.onerror = () => {
  // Connection dropped — browser will auto-reconnect unless es.close() was called
};

// When done (e.g., component unmounts):
es.close();
```

**Sending data back (separate POST):**
```javascript
// SSE is server→client only — client messages go over a regular fetch
async function sendMessage(message) {
  await fetch('/api/chat/message', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ text: message })
  });
  // Server will push the response tokens back over the open EventSource connection
}
```

## Server-Side Pattern (Node.js)

```javascript
app.get('/stream', (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');

  // Push data as it's available
  const send = (data) => res.write(`data: ${JSON.stringify(data)}\n\n`);

  // Example: stream LLM tokens
  llmStream.on('token', (token) => send(token));
  llmStream.on('done', () => {
    res.write('data: [DONE]\n\n');
    res.end();
  });

  // Clean up if client disconnects
  req.on('close', () => llmStream.destroy());
});
```

Think of it as: the server is writing to a file, and the browser is `tail -f`ing it in real time.

## SSE vs WebSocket

| Dimension | SSE (EventSource) | WebSocket |
|-----------|------------------|-----------|
| Direction | Server → client only | Bidirectional |
| Data types | Text only | Text + binary |
| Protocol | Regular HTTP | Protocol upgrade handshake |
| Reconnect | Automatic (built-in) | Manual |
| Firewall/proxy | Works through all standard proxies | Often blocked; requires ws:// support |
| Infrastructure | Standard HTTP server | Dedicated WebSocket server |
| Use case | Streaming output (LLM tokens, notifications, live feeds) | Real-time bidirectional (chat, games, collaborative editing) |

**The rule of thumb:** if you only need the server to push data to the client, SSE is simpler and more compatible. If you need the client to also push data at high frequency without separate requests, WebSocket is appropriate.

## Primary Use Case: Streaming LLM Responses

SSE is the standard transport for word-by-word LLM streaming:

```
User types message
       ↓
fetch POST to /chat (sends the message)
       ↓
Server calls LLM API with streaming enabled
       ↓
LLM tokens arrive at server → immediately pushed down SSE stream
       ↓
Browser receives tokens via onmessage → appends to UI in real time
       ↓
[DONE] signal → close connection or wait for next user message
```

The user sees text appearing progressively rather than waiting for the entire response to complete.

## Edge Cases and Production Considerations

**Authentication**: `EventSource` doesn't support custom headers — tokens must go in the URL query string (`?token=abc`) or be set via cookies. URL tokens are visible in server logs; consider short-lived session tokens for SSE connections.

**Connection limit**: Browsers limit concurrent connections per domain (typically 6 for HTTP/1.1, higher for HTTP/2). Multiple open `EventSource` connections to the same domain can hit this limit. HTTP/2 multiplexing largely solves this.

**Long-running connections and proxies**: Some proxies and load balancers impose idle timeouts. The server should send periodic keep-alive comments (`:\n\n`) to prevent the connection from being terminated.

**Cleanup**: Always call `es.close()` when the component or page unmounts. Unclosed EventSource connections persist as background activity even after navigation.

## Questions & Gaps
- How does SSE interact with service workers? A service worker can intercept `EventSource` requests but handling streaming responses in a service worker requires careful implementation.
- For LLM streaming specifically: what's the recommended pattern when the stream needs to be re-established after a network drop mid-response? `Last-Event-ID` can resume, but the LLM response can't be regenerated from a mid-stream point — the server typically has to restart generation.
- HTTP/2 server push is a related but distinct concept — is there a meaningful performance difference between SSE over HTTP/2 vs HTTP/1.1 for high-frequency streaming?

## Related Notes
- [JWT Token Refresh Architecture](https://github.com/LutherCalvinRiggs/research/blob/main/javascript/infrastructure/jwt-token-refresh-architecture.md) — the SSE `?token=abc` authentication pattern and the refresh architecture connect directly: the short-lived token placed in the EventSource URL needs to be rotated using the same 80% timer refresh pattern described there.
- [Node.js 8 Production Patterns](https://github.com/LutherCalvinRiggs/research/blob/main/javascript/infrastructure/nodejs-8-production-patterns.md) — graceful shutdown pattern is relevant here: a server that restarts without draining SSE connections will drop all active streams mid-response.
- [API Versioning Strategies](https://github.com/LutherCalvinRiggs/research/blob/main/technology/fundamentals/api-versioning-strategies.md) — the SSE wire format (`data:`, `event:`, `id:` fields) is an implicit API contract. Changes to event naming or data shape are breaking changes that need versioning treatment.

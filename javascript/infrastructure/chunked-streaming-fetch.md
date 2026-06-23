# Chunked Streaming Fetch — ReadableStream, AbortController, TextDecoder

**Source:** Technical notes (pasted content)
**Saved:** 2026-06-17
**Tags:** javascript, infrastructure, fundamentals, technology, tools

> Read alongside: `javascript/infrastructure/sse-server-sent-events.md` and `javascript/infrastructure/websockets.md` — these three notes cover the full spectrum of streaming/persistent connection patterns.

---

## TL;DR
Chunked streaming fetch is standard HTTP POST where the server flushes the response body incrementally rather than buffering the full response before sending. The browser reads chunks as bytes via `ReadableStream` on `response.body`. No persistent connection, no special infrastructure — works on Lambda, Express, any HTTP server. The tradeoff: the server can only respond to a request, never push unprompted. Three browser-native APIs make this work cleanly together: `ReadableStream` (chunk-by-chunk reading), `AbortController` (cancellation), and `TextDecoder` (bytes-to-string with multi-byte character safety).

---

## The Three Patterns Side by Side

| | Chunked Fetch | SSE | WebSocket |
|--|--------------|-----|-----------|
| Connection | New per request | Persistent (server→client) | Persistent (bidirectional) |
| Server can push unprompted? | ❌ No | ✅ Yes | ✅ Yes |
| Infrastructure needed | Any HTTP server | Any HTTP server | Stateful server |
| Works through proxies | ✅ Always | ✅ Always | Sometimes blocked |
| Use case | LLM response streaming | Live feeds, notifications | Chat, bidding, realtime |

---

## 1. Chunked Fetch (ReadableStream)

The browser reads `response.body` as a `ReadableStream` — a sequence of `Uint8Array` chunks that arrive as the server sends them, before the response is complete.

```javascript
const response = await fetch('/api/chat', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ message: 'Hello' }),
});

const reader = response.body.getReader();
const decoder = new TextDecoder();

while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  // value is Uint8Array — raw bytes from the network
  const text = decoder.decode(value, { stream: true });
  console.log(text); // Arrives in pieces: "Hel", "lo! How", " can I help?"
}
```

**Server side (Node.js/Express):**
```javascript
app.post('/api/chat', async (req, res) => {
  res.setHeader('Content-Type', 'text/plain');
  res.setHeader('Transfer-Encoding', 'chunked');

  const stream = llm.stream(req.body.message);
  for await (const token of stream) {
    res.write(token); // Flush each token immediately
  }
  res.end();
});
```

**Key characteristic:** Each call to `send()` in a transport layer creates an entirely new fetch call with its own response stream. The server can never initiate — it can only respond to requests the client makes.

**Works on Lambda:** Unlike WebSocket servers, chunked streaming works with stateless serverless functions — the function just needs to flush incrementally rather than buffering. AWS Lambda supports streaming responses with `awslambda.streamifyResponse()`.

---

## 2. AbortController — Cancelling Fetch Requests

**The problem:** A fetch fires and the user closes the chat / navigates away / sends a new message. Without cancellation, the request completes anyway — wasting bandwidth, triggering callbacks on a destroyed component, potentially overwriting new state with stale results.

**How AbortController works:**

```javascript
// 1. Create a controller
const controller = new AbortController();

// 2. Link it to the fetch via .signal
fetch('/api/chat', {
  method: 'POST',
  body: JSON.stringify({ message: 'Hello' }),
  signal: controller.signal,
});

// 3. Cancel at any time
controller.abort();
// → fetch promise rejects with AbortError
// → if mid-stream, the reader also throws AbortError
// → network connection is closed immediately
```

**Production pattern (transport class):**

```javascript
class StreamingTransport {
  constructor(endpoint) {
    this.endpoint = endpoint;
    this.abortController = null;
  }

  send(message) {
    // Cancel any existing in-flight request before starting a new one
    this.abortController?.abort();
    this.abortController = new AbortController();

    fetch(this.endpoint, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ message }),
      signal: this.abortController.signal,
    })
      .then(async (res) => {
        const reader = res.body.getReader();
        const decoder = new TextDecoder();
        while (true) {
          const { done, value } = await reader.read();
          if (done) break;
          this.onChunk?.(decoder.decode(value, { stream: true }));
        }
      })
      .catch((err) => {
        // AbortError is intentional — don't surface it as an error
        if (err.name !== 'AbortError') this.onError?.(err);
      });
  }

  disconnect() {
    this.abortController?.abort(); // Kills the in-flight stream immediately
    this.abortController = null;
  }
}
```

**Why ignore `AbortError` in `.catch`:** When `abort()` is called intentionally (user closes chat, sends new message), the resulting `AbortError` is expected behavior. Only unexpected errors should be surfaced as errors in the UI.

---

## 3. TextDecoder — Bytes to String

`reader.read()` returns `value` as a `Uint8Array` — raw binary bytes. `TextDecoder` converts these to a string.

```javascript
const decoder = new TextDecoder(); // Defaults to UTF-8

const text = decoder.decode(value, { stream: true });
```

**Why `{ stream: true }` is required:**

Multi-byte characters (emoji, accented characters, CJK) can be split across chunk boundaries. Without `{ stream: true }`, the decoder tries to interpret each chunk as complete characters — a split multi-byte sequence produces `�` (the replacement character for invalid/incomplete UTF-8).

```javascript
// 🎉 is 4 bytes: [0xF0, 0x9F, 0x8E, 0x89]
// If split across two chunks:

// Without stream: true
chunk1 = Uint8Array[0xF0, 0x9F]         → "??" (broken replacement chars)
chunk2 = Uint8Array[0x8E, 0x89, 0x48, 0x69] → "??Hi" (still broken)

// With stream: true
chunk1 = Uint8Array[0xF0, 0x9F]         → "" (holds incomplete bytes internally)
chunk2 = Uint8Array[0x8E, 0x89, 0x48, 0x69] → "🎉Hi" (combines correctly)
```

`{ stream: true }` tells the decoder "this is a continuous stream — buffer incomplete multi-byte sequences and complete them on the next chunk." Without it, any response containing non-ASCII characters will occasionally produce garbled output.

**Cleanup:** When the stream is done, call `decoder.decode()` with no arguments (or with an empty buffer and `stream: false`) to flush any remaining buffered bytes:

```javascript
while (true) {
  const { done, value } = await reader.read();
  if (done) {
    const remaining = decoder.decode(); // flush any held bytes
    if (remaining) this.onChunk?.(remaining);
    break;
  }
  this.onChunk?.(decoder.decode(value, { stream: true }));
}
```

---

## Parsing SSE-formatted Chunks Over Fetch

When the server sends SSE-formatted data (`data: ...\n\n`) but you're using fetch instead of `EventSource`, you need to parse the format manually:

```javascript
// Buffer for incomplete lines
let buffer = '';

while (true) {
  const { done, value } = await reader.read();
  if (done) break;

  buffer += decoder.decode(value, { stream: true });
  const lines = buffer.split('\n');
  buffer = lines.pop(); // Keep the incomplete last line in the buffer

  for (const line of lines) {
    if (line.startsWith('data: ')) {
      const data = line.slice(6);
      if (data === '[DONE]') return;
      const parsed = JSON.parse(data);
      onToken(parsed.token);
    }
  }
}
```

This pattern is used by the OpenAI client library and many other LLM SDKs — they use chunked fetch with SSE-format parsing rather than the browser's `EventSource` API, because `EventSource` only supports GET requests while LLM prompts need to be sent as POST bodies.

---

## Questions & Gaps
- `Transfer-Encoding: chunked` is handled automatically by HTTP/1.1. How does HTTP/2 handle chunked responses? (HTTP/2 doesn't use chunked transfer encoding — it uses DATA frames, but the `ReadableStream` API abstracts this identically on the client side.)
- At what chunk size does the overhead of many small `res.write()` calls exceed the latency benefit of early delivery? Is there a recommended minimum flush size for LLM token streaming?
- How does Lambda's streaming response (`awslambda.streamifyResponse()`) interact with API Gateway? Are there payload size limits that don't apply to conventional responses?
- The line-buffering pattern above handles `data:` lines — but what about multi-line `data:` fields or named events (`event: ...`)? A full SSE parser is more complex.

## Related Notes
- [SSE (Server-Sent Events)](https://github.com/LutherCalvinRiggs/research/blob/main/javascript/infrastructure/sse-server-sent-events.md) — SSE uses `EventSource` (GET, persistent connection, auto-reconnect). Chunked fetch uses `fetch` (any method, new connection per request, manual cancellation). Same wire format often used; different transport.
- [WebSockets](https://github.com/LutherCalvinRiggs/research/blob/main/javascript/infrastructure/websockets.md) — WebSocket is bidirectional and persistent. Chunked fetch is request/response with a streaming response body. Use WebSocket when the server needs to push without a client prompt.
- [JWT Token Refresh Architecture](https://github.com/LutherCalvinRiggs/research/blob/main/javascript/infrastructure/jwt-token-refresh-architecture.md) — AbortController's pattern of "cancel old request, start new one" is directly analogous to clearing the refresh timer before setting a new one. Both prevent stale operations from completing after they've been superseded.
- [Node.js 8 Production Patterns](https://github.com/LutherCalvinRiggs/research/blob/main/javascript/infrastructure/nodejs-8-production-patterns.md) — graceful shutdown applies to streaming responses: on `SIGTERM`, let in-progress streams complete (or close them cleanly) before the server exits.

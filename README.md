# WeHaveWebSocketAtHome

Natchen's API for making websocket with everything.

A lightweight WebSocket relay running at **ws.natchen.us.kg**. No install, no SDK, no dependencies on your end - just HTTP and WebSocket.

---

## How it works

```
Your browser  --WSS-->  ws.natchen.us.kg  --ws://-->  Node relay
                                                           |
Your backend  --HTTP POST /send/{id}/-->               broadcasts to clients
```

The relay has no business logic. You create a session, get keys, connect your clients, and push messages from your backend via plain HTTP POST.

---

## Quick start

### 1. Create a session

```http
POST https://ws.natchen.us.kg/create
Content-Type: application/json

{
  "type": "redirect",
  "send_url": "https://yourserver.com/ws-handler"
}
```

Response:
```json
{
  "code": 200,
  "id": "50RmIlbJjdwnYUIH",
  "host_key": "jaPQzLxMP79tZiu0...",
  "client_key": "uz4wm7MXjL89HGJJ..."
}
```

- `host_key` - keep this secret on your backend
- `client_key` - reusable key for all your clients
- `id` - the channel identifier

### 2. Connect a client (browser)

```javascript
const ws = new WebSocket(
  `wss://ws.natchen.us.kg/${id}/?key=${encodeURIComponent(client_key)}`
);

ws.onmessage = (e) => {
  const { body } = JSON.parse(e.data);
  const payload = JSON.parse(body);
  console.log(payload.type, payload);
};
```

> **Note:** Browsers cannot send custom headers on WebSocket connections. Always pass the key as a query parameter.

### 3. Push from your backend

```bash
curl -X POST https://ws.natchen.us.kg/send/50RmIlbJjdwnYUIH/ \
  -H "Host-Key: jaPQzLxMP79tZiu0..." \
  -H "Content-Type: application/json" \
  -d '{"type":"hello","message":"world"}'
```

All connected clients receive:
```json
{
  "headers": { "content-type": "application/json" },
  "body": "{\"type\":\"hello\",\"message\":\"world\"}"
}
```

Parse `frame.body` as JSON to get your payload.

---

## Session options

All fields are optional:

| Field | Type | Description |
|-------|------|-------------|
| `type` | `"redirect"` or omit | `redirect` = backend pushes via HTTP. Omit for direct WS host. |
| `send_url` | string | Required if `type: "redirect"`. URL called when a client sends a message. |
| `ttl` | number | Session lifetime in minutes. 0 or omit = no expiry. |
| `max_clients` | number | Max simultaneous WS connections. |
| `label` | string | Human-readable name for your session. |

---

## REST API reference

### Sessions

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/` | - | Server status |
| `POST` | `/create` | - | Create a session |
| `GET` | `/session/:id` | `Host-Key` | Session stats |
| `GET\|POST` | `/close/:id` | `Host-Key` | Close session + disconnect all clients |

### Messaging

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `POST` | `/send/:id/` | `Host-Key` | Broadcast to all clients |
| `POST` | `/send/:id/:client_id/` | `Host-Key` | Send to one client |

### Client management

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `POST` | `/kick/:id/:client_id` | `Host-Key` | Disconnect a specific client |
| `POST` | `/keys/:id` | `Host-Key` | Generate one-time client keys |

---

## One-time client keys

One-time keys let you give a client exactly one connection - the key is consumed on connect and cannot be reused.

**Generate keys:**
```http
POST /keys/50RmIlbJjdwnYUIH
Host-Key: jaPQzLxMP79tZiu0...
Content-Type: application/json

{ "count": 3 }
```

```json
{
  "code": 200,
  "keys": ["abc...", "def...", "ghi..."]
}
```

**Use on the client:**
```javascript
// Get a one-time key from your backend, then connect
const ws = new WebSocket(
  `wss://ws.natchen.us.kg/${id}/?key=${encodeURIComponent(oneTimeKey)}`
);
```

The key is deleted the moment the WebSocket handshake is accepted. The regular `client_key` is unaffected and remains reusable.

---

## Session stats

```http
GET /session/50RmIlbJjdwnYUIH
Host-Key: jaPQzLxMP79tZiu0...
```

```json
{
  "id": "50RmIlbJjdwnYUIH",
  "label": "my-feature",
  "type": "redirect",
  "clients": 2,
  "host_connected": false,
  "messages_sent": 42,
  "messages_recv": 17,
  "created_at": "2026-04-17T13:00:00.000Z",
  "expires_at": null,
  "max_clients": 10,
  "one_time_keys_remaining": 1
}
```

---

## Modes

### redirect mode (`type: "redirect"`)

- Your backend is the "host" - no WS host connection needed
- When a client sends a message, the relay POSTs it to your `send_url` with the client's headers + `Client-Id` header
- Your endpoint's HTTP response is forwarded back to that client over WS
- You can push to all clients anytime via `POST /send/:id/`

```
client WS msg --> Node --> POST send_url (with Client-Id header)
                       <-- HTTP response --> forwarded to client as WS frame
```

### Relay mode (no `type`)

- Connect a host via WSS with the `host-key`
- Host messages broadcast to all clients
- Client messages go to the host only

---

## Receiving messages in redirect mode

When a client sends a message, your `send_url` receives a POST with:
- All WS connection headers from the client
- `Client-Id: {unique_id}` - stable per connection
- Body = the raw message sent by the client

Your response (headers + body) is sent back to that client as a WS frame.

```php
$clientId = $_SERVER['HTTP_CLIENT_ID'] ?? null;
$data = json_decode(file_get_contents('php://input'), true);

header('Content-Type: application/json');
echo json_encode(['ok' => true, 'client' => $clientId]);
```

---

## WebSocket close codes

| Code | Meaning |
|------|---------|
| 4001 | Host already connected |
| 4002 | redirect mode - use /send instead |
| 4003 | Invalid or missing key |
| 4004 | Session not found |
| 4005 | Kicked by host |
| 4006 | Session expired (TTL reached) |
| 4007 | Max clients reached |
| 4010 | Host disconnected |

---

## Fallback polling

Always implement a polling fallback in case the WS server is unavailable.

```javascript
let ws = null;
let pollInterval = null;
let retryTimer = null;

async function connectWs(sessionUrl, clientKey) {
  clearTimeout(retryTimer);
  if (ws && ws.readyState <= 1) return;

  ws = new WebSocket(`${sessionUrl}?key=${encodeURIComponent(clientKey)}`);

  ws.onopen = () => stopPolling();

  ws.onmessage = (e) => {
    const payload = JSON.parse(JSON.parse(e.data).body);
    handleEvent(payload);
  };

  ws.onclose = () => {
    ws = null;
    startPolling();
    retryTimer = setTimeout(() => connectWs(sessionUrl, clientKey), 15000);
  };

  ws.onerror = () => startPolling();
}

function startPolling() {
  if (pollInterval) return;
  pollInterval = setInterval(fetchState, 5000);
  fetchState();
}

function stopPolling() {
  clearInterval(pollInterval);
  pollInterval = null;
}
```

---

## Errors

All HTTP error responses follow the same shape:

```json
{ "code": 404, "message": "Session introuvable" }
```

The relay silently fails on broadcast if the WS server is down - your backend won't crash.

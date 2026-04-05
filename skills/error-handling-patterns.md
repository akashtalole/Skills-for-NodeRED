# Node-RED Error Handling Patterns

Best practices and reusable patterns for error handling in Node-RED flows. Used by `nodered-author` when scaffolding error handling and by `nodered-troubleshoot` for diagnosing error-related issues.

---

## Core Error Handling Nodes

### catch Node

The `catch` node intercepts errors thrown by nodes on the same tab. It fires when:
- A `function` node calls `node.error("message", msg)`
- A core node encounters an unhandled exception (e.g., invalid JSON, network failure)
- A palette node throws an error

```json
{
  "id": "catch-all-tab",
  "type": "catch",
  "name": "Catch All Tab Errors",
  "scope": null,
  "uncaught": false,
  "x": 100,
  "y": 500,
  "z": "tab-main",
  "wires": [["log-error"]]
}
```

| `scope` | `uncaught` | Behavior |
|---------|-----------|----------|
| `null` | `false` | Catch all errors from all nodes on this tab |
| `null` | `true` | Catch only errors not handled by a more specific `catch` node |
| `["node-id-1"]` | `false` | Catch errors only from the listed nodes |

### Error Message Structure

When a `catch` node fires, the message contains:

```javascript
msg.error = {
  message: "The error message text",
  source: {
    id: "failing-node-id",
    type: "function",
    name: "My Function Node",
    count: 3  // number of times this error has occurred
  }
}
// msg.payload retains the value it had when the error occurred
// msg._msgid is the ID of the original message
```

### Throwing Errors from Function Nodes

```javascript
// Throw with original message (caught by catch nodes):
node.error("Validation failed: missing required field", msg);

// Throw without message (NOT caught by catch for this msg):
node.error("Something went wrong");

// Drop the message silently:
return null;

// Conditional error:
if (!msg.payload.sensor_id) {
    node.error("Missing sensor_id", msg);
    return null;
}
```

---

## Pattern 1: Log and Stop

The simplest pattern. Catch errors, log them, and halt processing.

```
inject/http in → ... → [catch] → [debug: log error]
```

```json
{
  "id": "catch-1", "type": "catch", "scope": null,
  "name": "Catch All", "x": 100, "y": 500, "z": "tab-1",
  "wires": [["debug-error-1"]]
},
{
  "id": "debug-error-1", "type": "debug",
  "name": "Log Error to Console",
  "complete": "true", "tosidebar": true, "console": true,
  "x": 300, "y": 500, "z": "tab-1",
  "wires": []
}
```

**Use when:** Development, debugging, or non-critical flows where silent failure is acceptable.

---

## Pattern 2: Log and Respond (HTTP Flows)

For `http in` flows, an uncaught error leaves the client hanging. Always send a response.

```
http in → ... → [http response: 200]
                     ↑
[catch] → [change: set statusCode=500, payload=error message] → [http response: error]
```

```json
{
  "id": "catch-http", "type": "catch", "scope": null,
  "name": "Handle HTTP Errors", "x": 100, "y": 500, "z": "tab-http",
  "wires": [["set-error-response"]]
},
{
  "id": "set-error-response", "type": "change",
  "name": "Set 500 Response",
  "rules": [
    { "t": "set", "p": "statusCode", "pt": "msg", "to": "500", "tot": "num" },
    { "t": "set", "p": "payload", "pt": "msg",
      "to": "{\"error\": true, \"message\": \"Internal server error\"}",
      "tot": "json" }
  ],
  "x": 300, "y": 500, "z": "tab-http",
  "wires": [["http-response-error"]]
},
{
  "id": "http-response-error", "type": "http response",
  "name": "Send Error Response",
  "statusCode": "",
  "x": 500, "y": 500, "z": "tab-http",
  "wires": []
}
```

**Use when:** Any tab containing `http in` nodes.

---

## Pattern 3: Retry with Exponential Backoff

Retry transient failures (network timeouts, rate limits) with increasing delays.

```
[http request] → [switch: status ok?] → success → [next node]
                      ↓ fail
                [function: check retry count]
                      ├── retry allowed → [delay] → [http request]  (loop back)
                      └── max retries   → [catch result]
```

**Function node: check retry count**
```javascript
const MAX_RETRIES = 3;
const BASE_DELAY = 1000; // milliseconds

msg._retries = (msg._retries || 0) + 1;

if (msg._retries > MAX_RETRIES) {
    node.error(`Max retries (${MAX_RETRIES}) exceeded`, msg);
    return null; // drop message, let catch node handle it
}

// Exponential backoff: 1s, 2s, 4s
msg._delay = BASE_DELAY * Math.pow(2, msg._retries - 1);
node.warn(`Retry ${msg._retries}/${MAX_RETRIES} in ${msg._delay}ms`);
return msg;
```

**Delay node configuration:**
```json
{
  "id": "retry-delay", "type": "delay",
  "name": "Backoff Delay",
  "pauseType": "delayv",
  "timeout": "5",
  "timeoutUnits": "seconds"
}
```

Set `msg.delay` in the function node (milliseconds) and use `pauseType: "delayv"` to honor it.

---

## Pattern 4: Alert on Error

Send a notification when a critical error occurs.

```
[catch] → [function: format alert] → [http request: POST to alert endpoint]
                                              ↓
                                      [debug: confirm sent]
```

**Function node: format alert**
```javascript
const alert = {
    severity: "ERROR",
    source: msg.error.source.name,
    message: msg.error.message,
    timestamp: new Date().toISOString(),
    payload: JSON.stringify(msg.payload).substring(0, 200) // truncate large payloads
};
msg.payload = alert;
msg.headers = { "Content-Type": "application/json" };
msg.url = env.get("ALERT_WEBHOOK_URL");
return msg;
```

---

## Pattern 5: Dead Letter Queue

Route failed messages to a persistent store for later investigation or reprocessing.

```
[catch] → [function: build DLQ entry] → [file: append to dlq.jsonl]
                                                  ↓
                                         [debug: confirm saved]
```

**Function node: build DLQ entry**
```javascript
const dlqEntry = {
    id: msg._msgid,
    timestamp: new Date().toISOString(),
    error: msg.error,
    payload: msg.payload,
    topic: msg.topic
};
msg.payload = JSON.stringify(dlqEntry) + "\n";
msg.filename = env.get("DLQ_FILE_PATH") || "/var/log/nodered-dlq.jsonl";
return msg;
```

---

## Pattern 6: Tab-Level vs. Global Error Handler

### Per-Tab catch (default)

Place a `catch` with `scope: null` on each tab. Errors stay local to the tab.

**Good for:** Tabs with different error handling requirements (HTTP vs MQTT vs file).

### Global Error Handler via Link Nodes

Create a dedicated "Error Handler" tab and route all errors there via `link out` / `link in`.

```
[Tab A: catch] → [link out: to error handler]
[Tab B: catch] → [link out: to error handler]
[Tab C: catch] → [link out: to error handler]
                          ↓
              [Error Handler Tab]
              [link in] → [function: enrich error] → [alert/log/dlq]
```

**Good for:** Consistent centralized error handling across all tabs.

---

## Pattern 7: Status Monitoring

Use the `status` node to monitor node health without catching errors.

```json
{
  "id": "status-mqtt", "type": "status",
  "name": "Monitor MQTT Connection",
  "scope": ["mqtt-in-node-id"],
  "x": 300, "y": 500, "z": "tab-mqtt",
  "wires": [["handle-status"]]
}
```

**Function node: handle status changes**
```javascript
const status = msg.status;
if (status.fill === "red") {
    // Connection lost
    node.warn(`MQTT disconnected: ${status.text}`);
    msg.payload = { event: "mqtt_disconnect", status: status.text, ts: Date.now() };
    return msg;
} else if (status.fill === "green") {
    // Connected
    msg.payload = { event: "mqtt_connect", ts: Date.now() };
    return msg;
}
return null; // Ignore other status events
```

---

## Pattern 8: Input Validation Guard

Validate incoming messages before processing to fail fast with clear errors.

```
[http in] → [function: validate input] → [process node]
                      ↓ invalid
              [change: set 400] → [http response]
```

**Function node: validate input**
```javascript
const payload = msg.payload;
const errors = [];

if (!payload) errors.push("Request body is required");
if (!payload.sensor_id) errors.push("sensor_id is required");
if (typeof payload.value !== 'number') errors.push("value must be a number");

if (errors.length > 0) {
    msg.statusCode = 400;
    msg.payload = { error: true, messages: errors };
    return [null, msg]; // output 2: validation error
}

return [msg, null]; // output 1: valid
```

Configure the `function` node with `"outputs": 2`.

---

## Checklist: Error Handling Coverage

Before deploying any tab, verify:

- [ ] At least one `catch` node exists on the tab with `scope: null`
- [ ] HTTP flows have an error `http response` reachable from the `catch` node  
- [ ] Function nodes that call external APIs have retry logic for transient errors
- [ ] Critical errors trigger an alert (webhook, email, MQTT notification)
- [ ] Long-running operations use the `status` node to monitor health
- [ ] Input validation rejects malformed data before it enters the processing pipeline
- [ ] Errors are logged with enough context (source node, payload snippet, timestamp)
- [ ] The catch node output wires to something — never leave them disconnected

---

## Error Message Best Practices

```javascript
// Good — includes context
node.error(`Failed to parse sensor data: expected number, got ${typeof msg.payload.value}`, msg);

// Bad — no context
node.error("Error", msg);

// Good — structured for downstream processing
msg.error_code = "SENSOR_PARSE_FAILED";
msg.error_detail = { expected: "number", received: typeof value };
node.error("Sensor parse failed", msg);

// Always include the original msg as second arg so catch nodes receive it
// node.error("msg") — catch won't receive the message
// node.error("msg", msg) — catch WILL receive the message ✅
```

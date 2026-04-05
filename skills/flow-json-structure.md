# Node-RED Flow JSON Structure

Complete reference for the Node-RED flow file format. All four agents use this document as the authoritative schema source.

---

## Top-Level Structure

A Node-RED flow file is a **JSON array** of node objects:

```json
[
  { "id": "...", "type": "tab", ... },
  { "id": "...", "type": "inject", ... },
  { "id": "...", "type": "function", ... }
]
```

The array may contain:
- **Tab nodes** — define flow pages (canvas tabs)
- **Canvas nodes** — functional nodes placed on a tab
- **Config nodes** — shared configuration objects (not visible on canvas)
- **Subflow definitions** — reusable flow templates
- **Subflow instances** — placements of a subflow definition on a tab
- **Group nodes** — visual groupings of canvas nodes
- **Comment nodes** — documentation annotations

---

## Tab Node

Tabs are the pages/sheets in the Node-RED editor workspace.

```json
{
  "id": "tab-0001",
  "type": "tab",
  "label": "MQTT Ingestion",
  "disabled": false,
  "info": "Subscribes to MQTT topics and stores messages to InfluxDB.\nRequires: MQTT_BROKER_HOST, INFLUX_URL"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | ✅ | Unique identifier |
| `type` | string | ✅ | Must be `"tab"` |
| `label` | string | ✅ | Human-readable tab name shown in editor |
| `disabled` | boolean | no | If `true`, the tab's flows do not run |
| `info` | string | no | Markdown description shown in info panel |

---

## Canvas Node (General)

All functional nodes placed on the canvas share these base fields:

```json
{
  "id": "node-0001",
  "type": "inject",
  "name": "Trigger Every Minute",
  "x": 100,
  "y": 100,
  "z": "tab-0001",
  "wires": [["node-0002"]]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | ✅ | Unique 8+ char hex string, e.g. `"a1b2c3d4"` |
| `type` | string | ✅ | Node type identifier (e.g. `"inject"`, `"function"`) |
| `name` | string | recommended | Descriptive label shown on the node in the editor |
| `x` | number | ✅ | Canvas x-coordinate (pixels from left) |
| `y` | number | ✅ | Canvas y-coordinate (pixels from top) |
| `z` | string | ✅ | ID of the parent tab or subflow definition |
| `wires` | array | ✅ | Output wire connections (see Wires Format below) |

Additional type-specific properties are documented in `node-reference.md`.

---

## Wires Format

The `wires` field defines outgoing connections from a node. It is an **array of output port arrays**:

```
wires[outputPortIndex][connectionIndex] = "target-node-id"
```

Examples:

```json
// Single output, one target:
"wires": [["node-0002"]]

// Single output, two targets (fan-out):
"wires": [["node-0002", "node-0003"]]

// Two outputs (e.g. switch node with two rules):
"wires": [["node-0002"], ["node-0003"]]

// Two outputs, first has two targets:
"wires": [["node-0002", "node-0004"], ["node-0003"]]

// Terminal node (no outputs):
"wires": []

// Node with output port but nothing connected:
"wires": [[]]
```

**Rules:**
- Outer array length = number of output ports the node has
- Inner arrays may be empty `[]` (port exists but nothing connected)
- Every target ID must exist in the same flow JSON array

---

## Config Node

Config nodes represent shared configuration (MQTT brokers, HTTP endpoints, TLS certificates). They are **not placed on the canvas** — they have no `x`, `y`, or `z`.

```json
{
  "id": "mqtt-broker-cfg",
  "type": "mqtt-broker",
  "name": "Production MQTT Broker",
  "broker": "${MQTT_BROKER_HOST}",
  "port": "1883",
  "clientid": "",
  "usetls": false,
  "protocolVersion": "4",
  "keepalive": "60",
  "cleansession": true
}
```

Canvas nodes reference config nodes by ID in a property (e.g., `"broker": "mqtt-broker-cfg"`).

Config nodes also appear in the flow JSON array alongside canvas nodes.

---

## Subflow Definition

A subflow is a reusable group of nodes packaged as a single node type.

```json
{
  "id": "subflow-0001",
  "type": "subflow",
  "name": "Normalize Temperature",
  "info": "Converts raw sensor readings to Celsius. Input: msg.payload (number). Output: msg.payload (number in Celsius).",
  "category": "Sensor Helpers",
  "in": [
    {
      "x": 60,
      "y": 80,
      "wires": [{ "id": "sf-node-0001" }]
    }
  ],
  "out": [
    {
      "x": 420,
      "y": 80,
      "wires": [{ "id": "sf-node-0002", "port": 0 }]
    }
  ],
  "env": [
    {
      "name": "SCALE_FACTOR",
      "type": "num",
      "value": "0.1",
      "ui": { "label": { "en-US": "Scale Factor" }, "type": "input", "opts": { "types": ["num"] } }
    }
  ]
}
```

Internal nodes of the subflow are regular canvas nodes with `"z": "subflow-0001"`.

### Subflow Port Definitions

`in` ports have a `wires` array pointing to internal node IDs (entry points).
`out` ports have a `wires` array pointing to the internal node ID and port index that feeds the output.

---

## Subflow Instance

A placed instance of a subflow definition. Its `type` is `"subflow:<definition-id>"`.

```json
{
  "id": "inst-0001",
  "type": "subflow:subflow-0001",
  "name": "Normalize Temp - Sensor A",
  "x": 300,
  "y": 200,
  "z": "tab-0001",
  "wires": [["node-0005"]],
  "env": [
    { "name": "SCALE_FACTOR", "value": "0.5", "type": "num" }
  ]
}
```

The `env` array on an instance overrides the default values from the subflow definition.

---

## Group Node

Groups are visual containers that do not affect flow execution.

```json
{
  "id": "group-0001",
  "type": "group",
  "name": "MQTT Receive Block",
  "z": "tab-0001",
  "style": {
    "stroke": "#999999",
    "stroke-opacity": "1",
    "fill": "#ffffff",
    "fill-opacity": "0.5",
    "label": true,
    "label-position": "nw",
    "color": "#000000"
  },
  "nodes": ["node-0001", "node-0002", "node-0003"],
  "x": 74,
  "y": 74,
  "w": 472,
  "h": 82
}
```

---

## Comment Node

Documentation nodes placed on the canvas.

```json
{
  "id": "comment-0001",
  "type": "comment",
  "name": "MQTT Ingestion Flow",
  "info": "## Purpose\nSubscribes to `sensors/#` topic and normalizes readings.\n\n## Dependencies\n- MQTT_BROKER_HOST\n- INFLUX_URL\n- INFLUX_TOKEN\n\n## Message Format\nInput: `{ sensor_id, value, unit, timestamp }`",
  "x": 100,
  "y": 40,
  "z": "tab-0001",
  "wires": []
}
```

---

## ID Format and Generation

Node-RED historically used 8-character lowercase hex IDs. Newer versions use longer strings. Both are valid.

**Generation rules:**
- Must be a non-empty string
- Must be unique within the entire flow JSON array
- Conventional format: `[a-f0-9]{8,16}`
- Examples: `"a1b2c3d4"`, `"f3e2d1c0b9a8"`, `"1234abcd"`

**Never:**
- Reuse an ID from an existing node when editing a flow
- Use sequential integers (`"1"`, `"2"`) — may collide with future nodes

---

## Canvas Coordinate System

The Node-RED canvas is a 2D plane measured in pixels.

**Recommended layout conventions:**

| Pattern | X Increment | Y Increment | Notes |
|---------|-------------|-------------|-------|
| Horizontal chain | +160 to +180 per node | 0 | Most common |
| Parallel branches | 0 | +80 per branch | Below a switch/function |
| New section | 0 | +160 to +200 | Visual separation |
| Comment node | Same X as first node | -40 to -60 | Above the flow |

**Starting position:** `x: 100, y: 100` for the first node.

---

## Environment Variable References

In string-typed node properties, use `${ENV_VAR_NAME}` to reference OS environment variables:

```json
{
  "url": "${API_BASE_URL}/endpoint",
  "broker": "${MQTT_BROKER_HOST}"
}
```

In `function` node JavaScript, use the `env` object:

```javascript
const brokerHost = env.get("MQTT_BROKER_HOST");
const apiKey = env.get("API_KEY");
```

**In subflow env arrays**, properties use `type: "env"` to pass through the parent's env vars:

```json
{ "name": "MY_VAR", "type": "env", "value": "PARENT_ENV_VAR_NAME" }
```

---

## Credentials

Sensitive config node credentials are stored in `flows_cred.json`, not in the main flow file. They appear in flow JSON as empty objects or omitted:

```json
{
  "id": "http-req-cfg",
  "type": "http-request",
  "credentials": {}
}
```

The actual credentials are in `flows_cred.json` (encrypted at rest). **Never output decrypted credentials in flow JSON.**

---

## Complete Minimal Example

```json
[
  {
    "id": "tab-main",
    "type": "tab",
    "label": "Hello World",
    "disabled": false,
    "info": "Simple demonstration flow"
  },
  {
    "id": "comment-main",
    "type": "comment",
    "name": "Hello World Flow",
    "info": "Injects a greeting every 10 seconds and logs it.",
    "x": 100,
    "y": 40,
    "z": "tab-main",
    "wires": []
  },
  {
    "id": "inject-main",
    "type": "inject",
    "name": "Trigger Every 10s",
    "props": [{ "p": "payload" }],
    "repeat": "10",
    "once": true,
    "onceDelay": 0.1,
    "payload": "Hello, Node-RED!",
    "payloadType": "str",
    "x": 100,
    "y": 100,
    "z": "tab-main",
    "wires": [["debug-main"]]
  },
  {
    "id": "debug-main",
    "type": "debug",
    "name": "Log to Sidebar",
    "active": true,
    "tosidebar": true,
    "console": false,
    "complete": "payload",
    "targetType": "msg",
    "x": 280,
    "y": 100,
    "z": "tab-main",
    "wires": []
  },
  {
    "id": "catch-main",
    "type": "catch",
    "name": "Catch All Errors",
    "scope": null,
    "uncaught": false,
    "x": 100,
    "y": 200,
    "z": "tab-main",
    "wires": [["debug-err-main"]]
  },
  {
    "id": "debug-err-main",
    "type": "debug",
    "name": "Log Error",
    "active": true,
    "tosidebar": true,
    "console": true,
    "complete": "true",
    "targetType": "full",
    "x": 280,
    "y": 200,
    "z": "tab-main",
    "wires": []
  }
]
```

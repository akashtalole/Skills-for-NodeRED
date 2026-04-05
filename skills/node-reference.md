# Node-RED Node Reference

Per-node property reference for all built-in (core) Node-RED node types. Used by `nodered-author` for configuration and by `nodered-test` for property validation.

---

## Input / Trigger Nodes

### inject

Injects messages into a flow manually or on a schedule.

```json
{
  "id": "...", "type": "inject", "z": "...",
  "name": "Trigger Every 5 Minutes",
  "props": [
    { "p": "payload" },
    { "p": "topic", "vt": "str" }
  ],
  "repeat": "300",
  "crontab": "",
  "once": false,
  "onceDelay": 0.1,
  "topic": "",
  "payload": "",
  "payloadType": "date",
  "wires": [["next-node-id"]]
}
```

| Property | Type | Description |
|----------|------|-------------|
| `props` | array | List of `{ p: propertyPath, v: value, vt: valueType }` objects |
| `repeat` | string | Repeat interval in seconds (`"60"` = 1 min) |
| `crontab` | string | Cron expression for scheduled triggers |
| `once` | boolean | Inject once on Node-RED start/deploy |
| `onceDelay` | number | Delay (seconds) before the first inject |
| `payload` | string | Payload value |
| `payloadType` | string | `"str"`, `"num"`, `"bool"`, `"json"`, `"date"`, `"env"`, `"flow"`, `"global"` |
| `topic` | string | `msg.topic` value |

---

### http in

Creates an HTTP endpoint that receives incoming HTTP requests.

```json
{
  "id": "...", "type": "http in", "z": "...",
  "name": "POST /data",
  "url": "/data",
  "method": "post",
  "upload": false,
  "swaggerDoc": "",
  "wires": [["process-node-id"]]
}
```

| Property | Required | Description |
|----------|----------|-------------|
| `url` | ✅ | URL path (e.g., `"/api/data"`) |
| `method` | ✅ | `"get"`, `"post"`, `"put"`, `"delete"`, `"patch"` |
| `upload` | no | Enable file upload handling |

Output message: `msg.req` (Express request), `msg.res` (Express response), `msg.payload` (request body), `msg.headers`, `msg.params`, `msg.query`.

**Must be paired with an `http response` node** to send a reply.

---

### mqtt in

Subscribes to an MQTT topic.

```json
{
  "id": "...", "type": "mqtt in", "z": "...",
  "name": "Subscribe to Sensors",
  "topic": "sensors/#",
  "qos": "2",
  "datatype": "json",
  "broker": "mqtt-broker-config-id",
  "wires": [["process-node-id"]]
}
```

| Property | Required | Description |
|----------|----------|-------------|
| `broker` | ✅ | ID of an `mqtt-broker` config node |
| `topic` | ✅ | MQTT topic string (supports `+` and `#` wildcards) |
| `qos` | no | `"0"`, `"1"`, or `"2"` |
| `datatype` | no | `"auto"`, `"utf8"`, `"buffer"`, `"json"`, `"base64"` |

---

### mqtt-broker (config node)

```json
{
  "id": "mqtt-broker-cfg",
  "type": "mqtt-broker",
  "name": "Production Broker",
  "broker": "${MQTT_BROKER_HOST}",
  "port": "${MQTT_BROKER_PORT}",
  "clientid": "",
  "usetls": false,
  "protocolVersion": "4",
  "keepalive": "60",
  "cleansession": true,
  "credentials": { "user": "", "password": "" }
}
```

Credentials are stored in `flows_cred.json` — the `credentials` object in the flow file is a placeholder.

---

## Output Nodes

### debug

Outputs messages to the debug sidebar and/or the Node-RED log.

```json
{
  "id": "...", "type": "debug", "z": "...",
  "name": "Log Full Message",
  "active": true,
  "tosidebar": true,
  "console": false,
  "tostatus": false,
  "complete": "true",
  "targetType": "full",
  "statusVal": "",
  "statusType": "auto",
  "wires": []
}
```

| Property | Description |
|----------|-------------|
| `active` | Whether the node outputs (can be toggled in editor) |
| `complete` | `"payload"` for `msg.payload` only, `"true"` for entire message object, or a property path |
| `targetType` | `"msg"` for message property, `"full"` for complete object |
| `tosidebar` | Show in debug sidebar |
| `console` | Write to Node-RED console log |
| `tostatus` | Show value in node status badge |

---

### http response

Sends an HTTP response to a request received by an `http in` node.

```json
{
  "id": "...", "type": "http response", "z": "...",
  "name": "Send 200 Response",
  "statusCode": "200",
  "headers": {},
  "wires": []
}
```

| Property | Description |
|----------|-------------|
| `statusCode` | HTTP status code string or `""` (uses `msg.statusCode`) |
| `headers` | Static response headers object, or set `msg.headers` upstream |

Input: `msg.payload` becomes the response body. Set `msg.statusCode` and `msg.headers` upstream to override.

---

### mqtt out

Publishes messages to an MQTT topic.

```json
{
  "id": "...", "type": "mqtt out", "z": "...",
  "name": "Publish to Commands",
  "topic": "commands/device-1",
  "qos": "1",
  "retain": false,
  "broker": "mqtt-broker-cfg",
  "wires": []
}
```

| Property | Required | Description |
|----------|----------|-------------|
| `broker` | ✅ | ID of an `mqtt-broker` config node |
| `topic` | no | Static topic (overridden by `msg.topic` if blank) |
| `qos` | no | `"0"`, `"1"`, `"2"` |
| `retain` | no | MQTT retain flag |

If `topic` is blank, `msg.topic` is used. `msg.payload` is the published content.

---

## Processing Nodes

### function

Executes JavaScript to process messages.

```json
{
  "id": "...", "type": "function", "z": "...",
  "name": "Normalize Payload",
  "func": "msg.payload = {\n  value: msg.payload.v,\n  unit: msg.payload.u,\n  ts: Date.now()\n};\nreturn msg;",
  "outputs": 1,
  "initialize": "",
  "finalize": "",
  "libs": [],
  "wires": [["next-node-id"]]
}
```

| Property | Required | Description |
|----------|----------|-------------|
| `func` | ✅ | JavaScript code string. Must `return msg;` to pass the message on. |
| `outputs` | no | Number of output ports (default 1) |
| `initialize` | no | Code run once when the node starts |
| `finalize` | no | Code run when the node stops |

**Key rules:**
- Return `msg` to pass the message on, `null` to drop it
- Return `[msg1, msg2]` array for multiple outputs
- `return [msg, null]` sends only on output 1; `return [null, msg]` sends only on output 2
- Access context: `context.get/set()`, `flow.get/set()`, `global.get/set()`
- Access env vars: `env.get("VAR_NAME")`
- Log: `node.log()`, `node.warn()`, `node.error()`
- Throw errors caught by `catch` nodes: `node.error("msg", msg)`

---

### change

Sets, changes, deletes, or moves message properties.

```json
{
  "id": "...", "type": "change", "z": "...",
  "name": "Set Status Code 200",
  "rules": [
    {
      "t": "set",
      "p": "statusCode",
      "pt": "msg",
      "to": "200",
      "tot": "num"
    },
    {
      "t": "set",
      "p": "headers.content-type",
      "pt": "msg",
      "to": "application/json",
      "tot": "str"
    }
  ],
  "action": "",
  "property": "",
  "from": "",
  "to": "",
  "reg": false,
  "wires": [["next-node-id"]]
}
```

Rule `t` values: `"set"`, `"change"` (find/replace), `"delete"`, `"move"`

`pt` / `tot` values: `"msg"`, `"flow"`, `"global"`, `"str"`, `"num"`, `"bool"`, `"json"`, `"jsonata"`, `"env"`, `"date"`

---

### switch

Routes messages to different outputs based on property values.

```json
{
  "id": "...", "type": "switch", "z": "...",
  "name": "Route by Status",
  "property": "payload.status",
  "propertyType": "msg",
  "rules": [
    { "t": "eq", "v": "ok", "vt": "str" },
    { "t": "eq", "v": "error", "vt": "str" },
    { "t": "else" }
  ],
  "checkall": "true",
  "repair": false,
  "outputs": 3,
  "wires": [["ok-handler"], ["error-handler"], ["unknown-handler"]]
}
```

| Property | Description |
|----------|-------------|
| `property` | Property path to evaluate |
| `propertyType` | `"msg"`, `"flow"`, `"global"`, `"jsonata"`, `"env"` |
| `rules` | Array of rule objects |
| `checkall` | `"true"` = check all rules; `"false"` = stop at first match |
| `outputs` | Must match the number of rules |

Rule `t` values: `"eq"`, `"neq"`, `"lt"`, `"lte"`, `"gt"`, `"gte"`, `"btwn"`, `"cont"`, `"regex"`, `"true"`, `"false"`, `"null"`, `"nnull"`, `"istype"`, `"head"`, `"tail"`, `"index"`, `"jsonata_exp"`, `"else"`

---

### template

Generates text output using Mustache templating.

```json
{
  "id": "...", "type": "template", "z": "...",
  "name": "Build JSON Body",
  "field": "payload",
  "fieldType": "msg",
  "format": "handlebars",
  "syntax": "mustache",
  "template": "{\n  \"device\": \"{{payload.id}}\",\n  \"value\": {{payload.value}}\n}",
  "output": "str",
  "wires": [["next-node-id"]]
}
```

| Property | Description |
|----------|-------------|
| `template` | Template string using `{{property}}` syntax |
| `field` | Output property path (default `"payload"`) |
| `format` | `"handlebars"`, `"json"`, `"yaml"` |
| `output` | `"str"`, `"json"` |
| `syntax` | `"mustache"` or `"plain"` |

---

### delay

Delays messages or rate-limits message flow.

```json
{
  "id": "...", "type": "delay", "z": "...",
  "name": "Rate Limit to 10/s",
  "pauseType": "rate",
  "timeout": "5",
  "timeoutUnits": "seconds",
  "rate": "10",
  "nbRateUnits": "1",
  "rateUnits": "second",
  "randomFirst": "1",
  "randomLast": "5",
  "randomUnits": "seconds",
  "drop": false,
  "allowrate": false,
  "outputs": 1,
  "wires": [["next-node-id"]]
}
```

`pauseType` values: `"delay"` (fixed delay), `"delayv"` (delay from `msg.delay`), `"random"` (random delay), `"rate"` (rate limit), `"queue"` (queue messages)

---

## Parser Nodes

### json

Converts between JSON strings and JavaScript objects.

```json
{
  "id": "...", "type": "json", "z": "...",
  "name": "Parse JSON Payload",
  "property": "payload",
  "action": "",
  "pretty": false,
  "wires": [["next-node-id"]]
}
```

| Property | Description |
|----------|-------------|
| `action` | `""` (auto-detect), `"obj"` (to object), `"str"` (to string) |
| `property` | Which `msg` property to parse (default `"payload"`) |
| `pretty` | Pretty-print when converting to string |

---

### csv

Parses or produces CSV formatted data.

```json
{
  "id": "...", "type": "csv", "z": "...",
  "name": "Parse CSV",
  "sep": ",",
  "hdrin": true,
  "hdrout": "none",
  "multi": "one",
  "ret": "\\n",
  "temp": "",
  "skip": "0",
  "strings": true,
  "include_empty_strings": false,
  "include_null_values": false,
  "wires": [["next-node-id"]]
}
```

---

### xml

Converts between XML strings and JavaScript objects.

```json
{
  "id": "...", "type": "xml", "z": "...",
  "name": "Parse XML",
  "property": "payload",
  "attr": "",
  "chr": "",
  "wires": [["next-node-id"]]
}
```

---

### yaml

Converts between YAML strings and JavaScript objects.

```json
{
  "id": "...", "type": "yaml", "z": "...",
  "name": "Parse YAML Config",
  "property": "payload",
  "wires": [["next-node-id"]]
}
```

---

## Network Nodes

### http request

Makes HTTP requests to external endpoints.

```json
{
  "id": "...", "type": "http request", "z": "...",
  "name": "Call Weather API",
  "method": "GET",
  "ret": "obj",
  "paytoqs": "ignore",
  "url": "${WEATHER_API_URL}/current?city={{topic}}",
  "tls": "",
  "persist": false,
  "proxy": "",
  "insecureHTTPParser": false,
  "authType": "",
  "senderr": false,
  "credentials": {},
  "wires": [["handle-response"]]
}
```

| Property | Required | Description |
|----------|----------|-------------|
| `url` | ✅ | URL — supports `{{msg.property}}` substitution and `${ENV_VAR}` |
| `method` | ✅ | `"GET"`, `"POST"`, `"PUT"`, `"DELETE"`, `"PATCH"`, or `"use"` (from `msg.method`) |
| `ret` | no | Return type: `"txt"` (string), `"bin"` (buffer), `"obj"` (parsed JSON) |
| `paytoqs` | no | Send `msg.payload` as query string: `"ignore"`, `"body"`, `"url"` |
| `tls` | no | ID of a `tls-config` config node |
| `persist` | no | Keep connection alive between requests |
| `authType` | no | `""`, `"basic"`, `"digest"`, `"bearer"` |

Output: `msg.payload` = response body, `msg.statusCode`, `msg.headers`, `msg.responseUrl`

---

## File Nodes

### file

Writes content to a file.

```json
{
  "id": "...", "type": "file", "z": "...",
  "name": "Append to Log",
  "filename": "${LOG_DIR}/events.log",
  "filenameType": "str",
  "appendNewline": true,
  "createDir": true,
  "overwriteFile": "false",
  "encoding": "none",
  "wires": [["confirm-write"]]
}
```

| Property | Description |
|----------|-------------|
| `filename` | File path (supports `${ENV_VAR}`) |
| `appendNewline` | Append a newline after each write |
| `createDir` | Create parent directories if they don't exist |
| `overwriteFile` | `"false"` = append, `"true"` = overwrite, `"delete"` = delete file |

---

### file in

Reads content from a file.

```json
{
  "id": "...", "type": "file in", "z": "...",
  "name": "Read Config File",
  "filename": "${CONFIG_DIR}/settings.json",
  "filenameType": "str",
  "format": "utf8",
  "chunk": false,
  "sendError": false,
  "encoding": "none",
  "allProps": false,
  "wires": [["parse-json"]]
}
```

`format` values: `"utf8"`, `"lines"` (array of lines), `""` (Buffer)

---

## Sequence Nodes

### split

Splits a single message into multiple messages.

```json
{
  "id": "...", "type": "split", "z": "...",
  "name": "Split Array",
  "splt": "\\n",
  "spltType": "str",
  "arraySplt": 1,
  "arraySpltType": "len",
  "stream": false,
  "addname": "",
  "wires": [["process-each"]]
}
```

If `msg.payload` is an array, each element becomes a separate message. Sets `msg.parts` for reconstruction by `join`.

---

### join

Joins multiple messages into one.

```json
{
  "id": "...", "type": "join", "z": "...",
  "name": "Collect Results",
  "mode": "auto",
  "build": "array",
  "property": "payload",
  "propertyType": "msg",
  "key": "topic",
  "joiner": "\\n",
  "joinerType": "str",
  "useparts": true,
  "accumulate": false,
  "timeout": "",
  "count": "",
  "reduceRight": false,
  "wires": [["next-node-id"]]
}
```

`mode` values: `"auto"` (use `msg.parts`), `"custom"` (manual count/timeout), `"reduce"`

---

## Flow Control Nodes

### link in / link out

Virtual wire connections between different parts of a flow or across tabs.

```json
{
  "id": "link-in-0001",
  "type": "link in",
  "name": "Receive Error Alert",
  "links": ["link-out-0001"],
  "x": 100,
  "y": 300,
  "z": "tab-0001",
  "wires": [["error-handler-id"]]
}
```

```json
{
  "id": "link-out-0001",
  "type": "link out",
  "name": "Send Error Alert",
  "mode": "link",
  "links": ["link-in-0001"],
  "x": 640,
  "y": 100,
  "z": "tab-0002",
  "wires": []
}
```

`link out` `mode` values:
- `"link"` — Send to linked `link in` nodes
- `"return"` — Return message to the calling `link call` node

---

### catch

Catches errors thrown by nodes on the same tab.

```json
{
  "id": "...", "type": "catch", "z": "...",
  "name": "Catch All Tab Errors",
  "scope": null,
  "uncaught": false,
  "x": 100,
  "y": 500,
  "wires": [["error-debug-id"]]
}
```

| Property | Description |
|----------|-------------|
| `scope` | `null` = catch all nodes on tab; array of node IDs = catch only those nodes |
| `uncaught` | `true` = only catch errors not handled by other `catch` nodes |

Output: `msg.error.message`, `msg.error.source.id`, `msg.error.source.name`, `msg.error.source.type`

---

### status

Monitors status events from nodes on the tab.

```json
{
  "id": "...", "type": "status", "z": "...",
  "name": "Monitor MQTT Status",
  "scope": ["mqtt-node-id"],
  "x": 300,
  "y": 500,
  "wires": [["status-handler"]]
}
```

`scope`: `null` = monitor all nodes, or array of specific node IDs.

Output: `msg.status.fill`, `msg.status.shape`, `msg.status.text`, `msg.status.source.id`

---

### complete

Fires when a node successfully completes processing a message.

```json
{
  "id": "...", "type": "complete", "z": "...",
  "name": "After File Write",
  "scope": ["file-node-id"],
  "uncaught": false,
  "x": 300,
  "y": 540,
  "wires": [["confirm-handler"]]
}
```

---

## comment

Documentation node — has no effect on message flow.

```json
{
  "id": "...", "type": "comment", "z": "...",
  "name": "Section Title",
  "info": "Multi-line **markdown** description of this section.\n\nEnvironment variables required:\n- `MY_VAR`",
  "x": 100,
  "y": 40,
  "wires": []
}
```

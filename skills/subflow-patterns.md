# Node-RED Subflow Patterns

Design patterns for creating and using subflows in Node-RED. Used by `nodered-author` when deciding whether to extract reusable logic.

---

## When to Use Subflows

### Use a subflow when:

✅ The same group of nodes appears on **2 or more** different tabs or locations  
✅ A logical unit of work is **self-contained** with clear inputs and outputs  
✅ The logic is complex enough to warrant **hiding implementation details**  
✅ You want to be able to **update logic in one place** and have it propagate everywhere  
✅ The grouping creates a useful **named abstraction** (e.g., "Normalize Sensor Data", "Send Alert")

### Do NOT use a subflow when:

❌ The logic is used only once — a node group is more appropriate  
❌ The subflow would need to call other subflows more than 2 levels deep  
❌ The subflow has more than 8 configurable env var properties (too complex)  
❌ The logic is inherently tied to a specific tab's context (flow variables)

---

## Subflow JSON Structure

A subflow consists of two parts in the flow JSON array:

### 1. Subflow Definition

```json
{
  "id": "sf-normalize-001",
  "type": "subflow",
  "name": "Normalize Sensor Reading",
  "info": "## Normalize Sensor Reading\n\nConverts raw ADC values to engineering units.\n\n**Input:** `msg.payload` — raw sensor object `{ raw: number, sensor_type: string }`\n**Output:** `msg.payload` — normalized object `{ value: number, unit: string, ts: number }`\n\n**Environment Variables:**\n- `SCALE_FACTOR` — multiplier applied to raw value (default: 1.0)",
  "category": "Sensor Utils",
  "in": [
    {
      "x": 60,
      "y": 80,
      "wires": [{ "id": "sf-fn-convert" }]
    }
  ],
  "out": [
    {
      "x": 560,
      "y": 80,
      "wires": [{ "id": "sf-fn-convert", "port": 0 }]
    }
  ],
  "env": [
    {
      "name": "SCALE_FACTOR",
      "type": "num",
      "value": "1.0",
      "ui": {
        "icon": "font-awesome/fa-sliders",
        "label": { "en-US": "Scale Factor" },
        "type": "input",
        "opts": { "types": ["num", "env"] }
      }
    },
    {
      "name": "DEFAULT_UNIT",
      "type": "str",
      "value": "°C",
      "ui": {
        "label": { "en-US": "Default Unit" },
        "type": "input",
        "opts": { "types": ["str", "env"] }
      }
    }
  ],
  "meta": {
    "module": "sensor-utils",
    "type": "normalize-sensor",
    "version": "1.0.0",
    "author": "Akash Talole",
    "desc": "Normalizes raw sensor ADC readings to engineering units",
    "keywords": "sensor, normalize, iot",
    "license": "MIT"
  },
  "color": "#87A980"
}
```

### 2. Internal Nodes (inside the subflow)

All internal nodes use `"z": "sf-normalize-001"` (the subflow definition ID):

```json
{
  "id": "sf-fn-convert",
  "type": "function",
  "name": "Convert to Engineering Units",
  "func": "const scaleFactor = parseFloat(env.get('SCALE_FACTOR')) || 1.0;\nconst unit = env.get('DEFAULT_UNIT') || '°C';\nconst raw = msg.payload.raw || 0;\nmsg.payload = {\n  value: raw * scaleFactor,\n  unit: unit,\n  ts: Date.now(),\n  sensor_type: msg.payload.sensor_type\n};\nreturn msg;",
  "outputs": 1,
  "x": 300,
  "y": 80,
  "z": "sf-normalize-001",
  "wires": []
}
```

Note: The last node's `wires` is `[]` because its output goes to the subflow's `out` port (defined in the `out` array's `wires`).

### 3. Subflow Instance (placed on a tab)

```json
{
  "id": "inst-sensor-a",
  "type": "subflow:sf-normalize-001",
  "name": "Normalize Sensor A",
  "x": 400,
  "y": 100,
  "z": "tab-main",
  "env": [
    { "name": "SCALE_FACTOR", "value": "0.5", "type": "num" },
    { "name": "DEFAULT_UNIT", "value": "°F", "type": "str" }
  ],
  "wires": [["next-node-id"]]
}
```

Instance `env` values **override** the defaults set in the subflow definition.

---

## Port Design Guidelines

### Inputs (`in` array)

- Keep it to **1 input port** for most subflows — single entry point simplifies wiring
- Use 2 input ports only when the subflow has genuinely different entry behaviors
- The `in` port's `wires` array points to the first internal node that should receive the message

```json
"in": [
  {
    "x": 60,
    "y": 80,
    "wires": [{ "id": "sf-first-node-id" }]
  }
]
```

### Outputs (`out` array)

- **Output 0:** Normal/success path
- **Output 1:** Error/alternative path (optional)
- **Output 2+:** Rarely needed; document clearly

The `out` port's `wires` reference the last internal node and its output port index:

```json
"out": [
  {
    "x": 560,
    "y": 60,
    "wires": [{ "id": "sf-last-node-id", "port": 0 }]
  },
  {
    "x": 560,
    "y": 120,
    "wires": [{ "id": "sf-error-node-id", "port": 0 }]
  }
]
```

---

## Environment Variable Patterns

### Type Mapping

| Use Case | `type` | Example `value` |
|----------|--------|-----------------|
| String config | `"str"` | `"production"` |
| Number config | `"num"` | `"42"` |
| Boolean flag | `"bool"` | `"true"` |
| JSON object | `"json"` | `"{\"key\":\"value\"}"` |
| OS env var passthrough | `"env"` | `"PARENT_ENV_VAR_NAME"` |
| Credential reference | `"cred"` | `""` |

### Accessing in Function Nodes

```javascript
// Read subflow env var
const threshold = parseFloat(env.get('THRESHOLD')) || 100;
const mode = env.get('MODE') || 'default';
const config = JSON.parse(env.get('CONFIG') || '{}');

// With type checking
const enabled = env.get('ENABLED') === 'true';
```

### UI Configuration

The `ui` object controls how the property appears in the subflow instance editor:

```json
{
  "name": "API_URL",
  "type": "str",
  "value": "https://api.example.com",
  "ui": {
    "icon": "font-awesome/fa-globe",
    "label": { "en-US": "API Base URL" },
    "type": "input",
    "opts": { "types": ["str", "env"] }
  }
}
```

Available `type` values for `ui`: `"input"`, `"select"`, `"checkbox"`, `"spinner"`, `"cred"`

---

## Common Subflow Patterns

### Pattern 1: Transform and Forward

Single input → transform → single output. Most common pattern.

```
[in] → [function: transform] → [out]
```

### Pattern 2: Validate and Route

Input → validate → success or error output.

```
[in] → [function: validate] → [switch: has error?]
                                  ├── no  → [out 0: success]
                                  └── yes → [out 1: error]
```

### Pattern 3: Enrich from External Source

Input → lookup external data → merge → output.

```
[in] → [function: build request] → [http request] → [function: merge response] → [out]
                                                          ↓ error
                                                    [catch] → [out 1: error]
```

### Pattern 4: Rate-limited Publisher

Input → buffer → rate-limit → output.

```
[in] → [join: buffer 10 msgs or 1s timeout] → [split] → [delay: 100ms/msg] → [out]
```

### Pattern 5: Retry with Backoff

Input → attempt → success or retry.

```
[in] → [function: set attempt=0] → [http request]
                                       ├── success → [out 0]
                                       └── catch  → [function: check retry count]
                                                        ├── retry → [delay] → [http request]
                                                        └── give up → [out 1: error]
```

---

## Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Subflow `name` | Title Case, verb + noun | `"Normalize Sensor Data"`, `"Send Alert Email"` |
| Subflow `category` | Domain group | `"MQTT Helpers"`, `"HTTP Utils"`, `"Data Transform"` |
| Env var names | SCREAMING_SNAKE_CASE | `"API_BASE_URL"`, `"MAX_RETRIES"` |
| Internal node `name` | Title Case, what the node does | `"Validate Input Schema"`, `"Build HTTP Request"` |

---

## Subflow Documentation Requirements

Every subflow `info` field must include:

```markdown
## Subflow Name

One-sentence description of what this subflow does.

**Input:** `msg.payload` — description of expected input format/type
**Output port 0:** `msg.payload` — description of success output
**Output port 1 (if exists):** `msg.error` — description of error output

**Environment Variables:**
- `VAR_NAME` — description, default value, valid values

**Dependencies:**
- Any external services, palette nodes, or config nodes required

**Example Usage:**
Brief example of when to use this subflow
```

---

## Anti-Patterns to Avoid

| Anti-Pattern | Problem | Solution |
|--------------|---------|----------|
| Subflow accesses `flow.get/set()` | Breaks portability — context is tab-specific | Pass values via `msg` properties or env vars |
| Subflow uses hardcoded URLs/IPs | Not reusable across environments | Use env var properties |
| Subflow has >3 levels of nesting | Impossible to debug | Flatten to 2 levels max |
| Subflow with 10+ env vars | Overwhelming configuration | Split into multiple focused subflows |
| Subflow used for one-time logic | Unnecessary abstraction | Use a node group instead |
| Internal nodes with no names | Hard to debug | Always name every internal node |

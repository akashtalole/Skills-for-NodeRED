# Node-RED Admin REST API Operations

Complete reference for the Node-RED Admin REST API. Used by `nodered-manage` for all live-instance operations.

---

## Base URL and Authentication

**Base URL:** `${NODE_RED_URL}` (default: `http://localhost:1880`)

### Authentication Methods

**Method 1: Bearer Token** (when `adminAuth` uses `tokens` strategy)
```
Authorization: Bearer ${NODE_RED_TOKEN}
```

**Method 2: Username + Password → Exchange for Token**
```
POST /auth/token
Content-Type: application/x-www-form-urlencoded
Body: client_id=node-red-admin&grant_type=password&scope=*&username=admin&password=<password>

Response: { "access_token": "...", "expires_in": 604800, "token_type": "Bearer" }
```

**Method 3: No authentication** (when `adminAuth` is disabled in `settings.js`)
No Authorization header needed. Only safe on localhost/private networks.

### Required Headers for All API Calls

```
Content-Type: application/json
Node-RED-API-Version: v2
Authorization: Bearer ${NODE_RED_TOKEN}
```

---

## Flow Endpoints

### GET /flows

Retrieve all currently deployed flows.

**Request:**
```
GET /flows
Authorization: Bearer ${NODE_RED_TOKEN}
Node-RED-API-Version: v2
```

**Response 200:**
```json
[
  { "id": "tab-0001", "type": "tab", "label": "My Flow" },
  { "id": "node-0001", "type": "inject", "z": "tab-0001", ... },
  ...
]
```

**Use for:**
- Reading current state before making changes
- Creating backups before full deployment
- Diffing local JSON against live instance

---

### POST /flows

Deploy a complete set of flows (replaces all existing flows).

**Request:**
```
POST /flows
Authorization: Bearer ${NODE_RED_TOKEN}
Content-Type: application/json
Node-RED-API-Version: v2
Node-RED-Deployment-Type: full
Body: [ ...complete flows array... ]
```

**Deployment Type Header Options:**

| Value | Effect |
|-------|--------|
| `full` | Stop all flows, replace everything, restart. Safe for major changes. |
| `nodes` | Only restart nodes whose configuration changed. Minimal disruption. |
| `flows` | Restart entire tabs (flows) containing changed nodes. Balanced. |
| `reload` | Force complete reload even if no changes detected. |

**Response:**
- `204 No Content` — success
- `400 Bad Request` — invalid JSON or schema error
- `401 Unauthorized` — bad/missing token

**Critical: Always backup first:**
```
1. GET /flows  → save response as flows-backup-YYYY-MM-DD.json
2. POST /flows → deploy new flows
```

---

### GET /flow/:id

Retrieve a single flow (tab and its nodes).

**Request:**
```
GET /flow/tab-0001
Authorization: Bearer ${NODE_RED_TOKEN}
```

**Response 200:**
```json
{
  "id": "tab-0001",
  "label": "My Flow",
  "nodes": [...],
  "configs": [...]
}
```

**Response 404:** Flow with that ID does not exist.

---

### POST /flow

Add a new flow tab.

**Request:**
```
POST /flow
Authorization: Bearer ${NODE_RED_TOKEN}
Content-Type: application/json
Body:
{
  "label": "New Tab",
  "nodes": [
    { "id": "node-0001", "type": "inject", ... }
  ],
  "configs": []
}
```

**Response 200:**
```json
{ "id": "new-tab-id-generated-by-server" }
```

---

### PUT /flow/:id

Update an existing flow tab.

**Request:**
```
PUT /flow/tab-0001
Authorization: Bearer ${NODE_RED_TOKEN}
Content-Type: application/json
Body:
{
  "id": "tab-0001",
  "label": "Updated Tab",
  "nodes": [...updated nodes...],
  "configs": []
}
```

**Response 200:**
```json
{ "id": "tab-0001" }
```

**Response 404:** Tab ID does not exist — use `POST /flow` to create it.

---

### DELETE /flow/:id

Delete a flow tab and all its nodes.

**Request:**
```
DELETE /flow/tab-0001
Authorization: Bearer ${NODE_RED_TOKEN}
```

**Response 204:** Success.
**Response 404:** Tab not found.

**Warning:** This is irreversible without a backup. Always `GET /flows` first.

---

## Node Palette Endpoints

### GET /nodes

List all installed node palette packages.

**Request:**
```
GET /nodes
Authorization: Bearer ${NODE_RED_TOKEN}
```

**Response 200:**
```json
[
  {
    "id": "node-red-contrib-mqtt-broker",
    "name": "node-red-contrib-mqtt-broker",
    "version": "0.2.4",
    "nodes": [
      { "id": "aedes", "name": "aedes", "types": ["aedes"], "enabled": true, "loaded": true }
    ],
    "enabled": true,
    "local": true,
    "user": true
  }
]
```

---

### POST /nodes

Install a new node palette package from npm.

**Request:**
```
POST /nodes
Authorization: Bearer ${NODE_RED_TOKEN}
Content-Type: application/json
Body: { "module": "node-red-contrib-mqtt-broker", "version": "latest" }
```

**Response 200:** Installation in progress (async).
**Response 400:** Invalid module name.
**Response 404:** Module not found on npm.

**Important:** Node-RED may need to restart after palette install. Monitor the `/nodes/:id` endpoint to check install status.

---

### GET /nodes/:module

Get details about an installed module.

**Request:**
```
GET /nodes/node-red-contrib-mqtt-broker
Authorization: Bearer ${NODE_RED_TOKEN}
```

**Response 200:** Module details (same structure as one item from `GET /nodes`)
**Response 404:** Module not installed.

---

### PUT /nodes/:module

Enable or disable an installed module.

**Request:**
```
PUT /nodes/node-red-contrib-mqtt-broker
Authorization: Bearer ${NODE_RED_TOKEN}
Content-Type: application/json
Body: { "enabled": true }
```

---

### DELETE /nodes/:module

Remove an installed node palette package.

**Request:**
```
DELETE /nodes/node-red-contrib-mqtt-broker
Authorization: Bearer ${NODE_RED_TOKEN}
```

**Response 204:** Success.
**Response 404:** Module not found.

---

## Settings Endpoint

### GET /settings

Retrieve Node-RED runtime settings (safe subset — passwords and secrets are redacted).

**Request:**
```
GET /settings
Authorization: Bearer ${NODE_RED_TOKEN}
```

**Response 200:**
```json
{
  "httpNodeRoot": "/",
  "version": "3.1.0",
  "user": { "anonymous": false, "permissions": "*" },
  "context": { "default": "memory", "stores": ["memory"] },
  "editorTheme": {}
}
```

---

## Diagnostics Endpoint (Node-RED 3.x+)

### GET /diagnostics

Get system diagnostics including Node.js version, Node-RED version, and memory usage.

```
GET /diagnostics
Authorization: Bearer ${NODE_RED_TOKEN}
```

---

## Error Code Reference

| HTTP Status | Cause | Resolution |
|-------------|-------|------------|
| `400` | Malformed JSON or schema violation | Validate with `nodered-test`, fix errors |
| `401` | Missing or invalid auth token | Check `NODE_RED_TOKEN`, re-authenticate |
| `404` | Resource not found (flow ID, node module) | Verify ID with `GET /flows` or module with `GET /nodes` |
| `409` | Version conflict (stale `rev` token) | Re-fetch current flows, re-apply changes |
| `422` | Palette install failed (npm error) | Check npm registry access, verify package name |
| `500` | Internal Node-RED error | Check Node-RED process logs |

---

## Configuring Node-RED for API Access

### Enable adminAuth in settings.js

```javascript
module.exports = {
    adminAuth: {
        type: "credentials",
        users: [{
            username: "admin",
            password: "$2b$08$zZWtXTja0fB1pzD4sHCMyOCMYz2Z6dNbM6tl8sJogENOMcxWV9DN.",
            permissions: "*"
        }]
    }
}
```

The password is a bcrypt hash. Generate with:
```bash
node-red admin hash-pw
```

### Setting NODE_RED_URL and NODE_RED_TOKEN

```bash
# In shell
export NODE_RED_URL="http://localhost:1880"
export NODE_RED_TOKEN="your-bearer-token-here"

# Or in .env file (load with dotenv)
NODE_RED_URL=http://localhost:1880
NODE_RED_TOKEN=your-bearer-token-here
```

---

## Revision Token (Optimistic Concurrency)

The `Node-RED-Deployment-Revision` header prevents conflicting concurrent deployments:

1. `GET /flows` — response includes a `rev` field in the response header
2. Include `Node-RED-Deployment-Revision: <rev>` in subsequent `POST /flows`
3. If another client deployed in between, you'll get `409 Conflict`

This is optional but recommended in multi-user environments.

---

## Credentials File

The `flows_cred.json` file is stored alongside `flows.json` in the Node-RED user directory (`~/.node-red/` by default). It contains encrypted credentials for config nodes.

**Never:**
- Output `flows_cred.json` contents in plaintext
- Commit `flows_cred.json` to version control
- Include actual credential values in flow JSON (use `credentials: {}` placeholder)

**Backup procedure:**
1. Back up `flows.json` via `GET /flows`
2. Back up `flows_cred.json` separately using encrypted file copy
3. The credential encryption key is in `settings.js` (`credentialSecret` field)

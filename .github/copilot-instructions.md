# GitHub Copilot Instructions — Skills for Node-RED

This repository provides AI agent skills for Node-RED flow development.
These instructions give Copilot background context for working in this project.

---

## Project Overview

**Skills for Node-RED** is an agent skill plugin for Claude Code and GitHub Copilot
that accelerates development of Node-RED flows. Node-RED is a low-code, flow-based
programming environment where automation logic is expressed as JSON arrays of connected nodes.

This repository contains four specialized agents and five supporting reference skills
following the pattern established by `microsoft/skills-for-copilot-studio`.

---

## Node-RED Flow JSON — Key Facts

Node-RED flow files are JSON arrays. Every agent and suggestion involving flow JSON
must respect these rules:

```json
[
  { "id": "tab-001", "type": "tab", "label": "My Flow" },
  {
    "id": "node-001",
    "type": "inject",
    "name": "Descriptive Name Here",
    "x": 100, "y": 100,
    "z": "tab-001",
    "wires": [["node-002"]]
  }
]
```

### Critical Rules

1. `wires` is an **array of output-port arrays**: `wires[portIndex][connectionIndex] = "target-id"`
2. Every canvas node's `z` must reference a `"type": "tab"` node ID in the same array
3. Every ID in every `wires` array must exist as a node in the array
4. No duplicate `id` values in the array
5. Config nodes (mqtt-broker, etc.) have **no** `x`, `y`, `z` — they're not on the canvas
6. **NEVER hardcode** credentials, IPs, API keys — always use `"${ENV_VAR_NAME}"` syntax
7. **ALWAYS name every node** — `"name"` must be a descriptive, non-empty string

### ID Format
8-character lowercase hex strings: `"a1b2c3d4"`. Must be unique across the entire array.

### Canvas Coordinates
- Start: `x: 100, y: 100`
- Horizontal chain: increment `x` by 160-180 per node
- Parallel branches: increment `y` by 80
- Comment nodes: place at `y - 40` above the first row

---

## The Four Agents

| Agent | File | Use When |
|-------|------|----------|
| `nodered-author` | `agents/nodered-author.md` | Creating/editing flow JSON |
| `nodered-manage` | `agents/nodered-manage.md` | Deploying to live Node-RED instance |
| `nodered-test` | `agents/nodered-test.md` | Validating flow JSON structure |
| `nodered-troubleshoot` | `agents/nodered-troubleshoot.md` | Debugging runtime errors |

See `AGENTS.md` for the full routing table.

---

## Required Environment Variables

For live instance management:
- `NODE_RED_URL` — Base URL (e.g., `http://localhost:1880`)
- `NODE_RED_TOKEN` — Admin API Bearer token

For example flows, see `AGENTS.md` for the complete variable list.

---

## Code Style for Flow JSON

When generating or modifying Node-RED flow JSON:

- **Tab labels:** Domain + Function → `"MQTT Ingestion"`, `"HTTP API"`, `"Data Transform"`
- **Node names:** Title Case verb + noun → `"Parse JSON Payload"`, `"Build HTTP Request"`
- **Subflow names:** Title Case → `"Normalize Sensor Data"`, `"Send Alert Email"`
- **Env var names:** SCREAMING_SNAKE_CASE → `MQTT_BROKER_HOST`, `API_BASE_URL`
- **Comment nodes:** Required on each tab at `x:100, y:40`; document purpose, env vars, dependencies
- **Catch nodes:** Required on every tab with network or I/O nodes

## NEVER Do

- Hardcode passwords, tokens, API keys, IP addresses, or file paths
- Generate duplicate node IDs
- Omit `z` from canvas nodes
- Leave `wires` pointing to non-existent node IDs
- Output partial flow snippets when a complete importable array is needed
- Remove or change node IDs in existing flows when only updating properties

---

## Reference Files

For detailed reference when working with Node-RED in this project:

| File | Content |
|------|---------|
| `skills/flow-json-structure.md` | Complete flow JSON schema |
| `skills/node-reference.md` | Per-node property reference (20+ node types) |
| `skills/rest-api-operations.md` | Admin REST API reference |
| `skills/subflow-patterns.md` | Subflow design patterns |
| `skills/error-handling-patterns.md` | Error handling patterns |
| `examples/basic-http-flow.json` | HTTP API example |
| `examples/mqtt-pipeline.json` | MQTT pipeline example |
| `examples/error-handling-flow.json` | Error patterns example |

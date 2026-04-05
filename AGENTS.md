# AGENTS.md — Skills for Node-RED

This repository provides AI agent skills for accelerating Node-RED flow development.
Node-RED is a visual, flow-based programming tool where logic is expressed as JSON
arrays of connected nodes.

---

## Agent Routing

Use this table to decide which agent to invoke for any Node-RED task:

| Task | Agent |
|------|-------|
| Create a new flow from a description | `nodered-author` |
| Add a node to an existing flow | `nodered-author` |
| Configure node properties | `nodered-author` |
| Wire (connect) nodes together | `nodered-author` |
| Create a subflow | `nodered-author` |
| Organize nodes into tabs or groups | `nodered-author` |
| Add error handling (catch nodes) | `nodered-author` |
| Convert a plain-language automation to JSON | `nodered-author` |
| Deploy flows to a live Node-RED instance | `nodered-manage` |
| Read/export flows from a live instance | `nodered-manage` |
| Import a flow JSON file to a live instance | `nodered-manage` |
| Install a palette package (node-red-contrib-*) | `nodered-manage` |
| Remove a palette package | `nodered-manage` |
| Check what palettes are installed | `nodered-manage` |
| Read Node-RED runtime settings | `nodered-manage` |
| Validate flow JSON structure | `nodered-test` |
| Check wiring integrity (broken wire references) | `nodered-test` |
| Verify required node properties are present | `nodered-test` |
| Write test scenarios for a flow | `nodered-test` |
| Audit flow for hardcoded secrets | `nodered-test` |
| Run a pre-deployment checklist | `nodered-test` |
| Debug a runtime error or exception | `nodered-troubleshoot` |
| Fix a broken or non-functioning flow | `nodered-troubleshoot` |
| Diagnose why a message is not flowing | `nodered-troubleshoot` |
| Troubleshoot MQTT or HTTP connection issues | `nodered-troubleshoot` |
| Diagnose Node-RED startup failure | `nodered-troubleshoot` |
| Explain a Node-RED error message | `nodered-troubleshoot` |

---

## Agent Descriptions

### nodered-author
**File:** `agents/nodered-author.md`

Creates and edits Node-RED flow JSON. Translates natural language automation requirements
into complete, importable flow JSON arrays. Handles all core node types, subflows, groups,
and wiring. Always uses `${ENV_VAR}` references for configuration values.

### nodered-manage
**File:** `agents/nodered-manage.md`

Operates live Node-RED instances via the Admin REST API. Deploys flows, reads/exports
current flows, installs palette packages, and manages instance configuration.
Requires `NODE_RED_URL` and `NODE_RED_TOKEN` environment variables.

### nodered-test
**File:** `agents/nodered-test.md`

Validates Node-RED flow JSON for structural correctness, wire integrity, property
completeness, and best-practice compliance. Generates test scenarios and pre-deployment
checklists. Does not modify flows — reports findings only.

### nodered-troubleshoot
**File:** `agents/nodered-troubleshoot.md`

Diagnoses and fixes issues in Node-RED flows and instances. Analyzes error messages,
traces message paths, and proposes targeted fixes. Delegates fix application to
`nodered-author` and redeployment to `nodered-manage`.

---

## Agent Interaction Pattern

The four agents form a workflow:

```
nodered-author → [creates flow JSON]
      ↓
nodered-test → [validates JSON] → report PASS/FAIL
      ↓ (if PASS)
nodered-manage → [deploys to live instance]
      ↓ (if runtime errors occur)
nodered-troubleshoot → [diagnoses] → delegates fix to nodered-author → re-test → re-deploy
```

---

## Project Conventions

All flows and skills in this repository follow these conventions:

### Flow JSON Conventions
- **Environment variables:** Use `${ENV_VAR_NAME}` in string properties; never hardcode hosts, ports, credentials, or file paths
- **Node names:** All nodes must have a descriptive `name` property (Title Case, verb + noun)
- **Tabs:** Named by domain and function (e.g., `"MQTT Ingestion"`, `"HTTP API"`, `"Data Transform"`)
- **IDs:** 8-character lowercase hex strings (e.g., `"a1b2c3d4"`)
- **Canvas layout:** Horizontal chains with x increments of 160-180px; new rows at y+80px; comment nodes at y-40 above first row
- **Error handling:** Every tab with network or I/O nodes must have at least one `catch` node
- **Subflows:** Extract logic used in 2+ places; subflow names in Title Case

### Skill File Conventions
- Each agent file in `agents/` has YAML frontmatter with `name` and `description`
- Supporting reference files live in `skills/` — they are cited by agents, not invoked directly
- Example flows in `examples/` demonstrate real patterns; they are importable into Node-RED

### Documentation Conventions
- Every flow tab must have a `comment` node at `x:100, y:40` documenting purpose, env vars, and dependencies
- Subflow `info` fields must include Input, Output, and Environment Variables sections

---

## Repository Structure

```
agents/                    # Agent skill definitions (4 files)
  nodered-author.md        # Flow authoring
  nodered-manage.md        # Instance management
  nodered-test.md          # Flow validation
  nodered-troubleshoot.md  # Debug and fix

skills/                    # Supporting reference skills (5 files)
  flow-json-structure.md   # Node-RED JSON schema
  node-reference.md        # Per-node property reference
  rest-api-operations.md   # Admin REST API
  subflow-patterns.md      # Subflow design patterns
  error-handling-patterns.md  # Error handling patterns

examples/                  # Example importable flows (3 files)
  basic-http-flow.json     # HTTP REST API with validation
  mqtt-pipeline.json       # MQTT subscribe → process → publish
  error-handling-flow.json # Error patterns demo

AGENTS.md                  # This file
README.md                  # User documentation
SETUP_GUIDE.md             # Installation guide
.github/
  copilot-instructions.md  # GitHub Copilot context
```

---

## Environment Variables Required

For `nodered-manage` operations against a live instance:

| Variable | Description | Required |
|----------|-------------|----------|
| `NODE_RED_URL` | Base URL of the Node-RED instance | ✅ |
| `NODE_RED_TOKEN` | Admin API Bearer token | ✅ for auth-enabled instances |

For example flows:

| Variable | Used In | Default |
|----------|---------|---------|
| `MQTT_BROKER_HOST` | `mqtt-pipeline.json` | none |
| `MQTT_BROKER_PORT` | `mqtt-pipeline.json` | `1883` |
| `API_VERSION` | `basic-http-flow.json` | `v1` |
| `EXTERNAL_API_URL` | `error-handling-flow.json` | none |
| `DLQ_FILE_PATH` | `error-handling-flow.json` | `/tmp/nodered-dlq.jsonl` |

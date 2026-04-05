# Skills for Node-RED

AI agent skills for accelerating [Node-RED](https://nodered.org) flow development with
[Claude Code](https://claude.ai/code) and [GitHub Copilot](https://github.com/features/copilot).

Inspired by [microsoft/skills-for-copilot-studio](https://github.com/microsoft/skills-for-copilot-studio),
this project provides four specialized agents with deep Node-RED knowledge to help you
author, manage, test, and troubleshoot Node-RED flows faster.

---

## The Four Agents

| Agent | Purpose |
|-------|---------|
| `nodered-author` | Create and edit Node-RED flows as JSON from natural language descriptions |
| `nodered-manage` | Deploy flows, install palettes, and operate live Node-RED instances via REST API |
| `nodered-test` | Validate flow JSON structure, check wiring integrity, and generate test scenarios |
| `nodered-troubleshoot` | Debug runtime errors, trace message paths, and fix broken flows |

---

## Quick Start

```bash
# Clone the repository
git clone https://github.com/akashtalole/Skills-for-NodeRED.git
cd Skills-for-NodeRED

# Start Claude Code — agents are auto-discovered
claude

# Or copy agents to use across all projects
cp agents/*.md ~/.claude/agents/
```

For GitHub Copilot, open the repository folder in VS Code.
The `.github/copilot-instructions.md` and `AGENTS.md` files are automatically picked up.

See [SETUP_GUIDE.md](SETUP_GUIDE.md) for full installation instructions.

---

## Usage Examples

### Author: Create a flow from a description

```
Using nodered-author, create a Node-RED flow that:
- Accepts POST requests at /api/sensor
- Validates the payload has id (string) and value (number)
- Publishes valid readings to MQTT topic sensors/{id}
- Returns 200 on success, 400 on validation failure
```

The agent produces a complete, importable flow JSON array.

---

### Author: Add error handling to an existing flow

```
Using nodered-author, add a catch node to my flow
[paste flow JSON here]
and wire it to a debug node and an HTTP 500 response.
```

---

### Manage: Deploy a flow

```
Using nodered-manage, deploy the flow in examples/basic-http-flow.json
to my local Node-RED instance.
```

---

### Manage: Install a palette package

```
Using nodered-manage, install node-red-contrib-influxdb
on my Node-RED instance at http://localhost:1880.
```

---

### Test: Validate a flow file

```
Using nodered-test, validate examples/mqtt-pipeline.json
and report any structural errors, wiring issues, or best-practice violations.
```

---

### Test: Generate test scenarios

```
Using nodered-test, write test scenarios for the HTTP API flow
in examples/basic-http-flow.json including normal path, validation errors,
and error handling.
```

---

### Troubleshoot: Debug a runtime error

```
Using nodered-troubleshoot, I'm seeing this error in my Node-RED logs:

TypeError: Cannot read properties of undefined (reading 'sensor_id')
    at Function node "Normalize Reading" (id: a1b2c3d4)

Here is the flow JSON: [paste JSON]
What is wrong and how do I fix it?
```

---

### Troubleshoot: Fix a broken flow

```
Using nodered-troubleshoot, my MQTT subscriber flow stopped receiving messages.
The mqtt in node shows a red "disconnected" badge.
My MQTT broker is running at ${MQTT_BROKER_HOST}.
What should I check?
```

---

## Repository Structure

```
agents/                       # Agent skill definitions
  nodered-author.md           # Flow authoring agent
  nodered-manage.md           # Instance management agent
  nodered-test.md             # Flow validation agent
  nodered-troubleshoot.md     # Debug and fix agent

skills/                       # Supporting reference skills
  flow-json-structure.md      # Node-RED JSON schema reference
  node-reference.md           # Per-node property reference (20+ types)
  rest-api-operations.md      # Admin REST API reference
  subflow-patterns.md         # Subflow design patterns
  error-handling-patterns.md  # Error handling patterns

examples/                     # Importable example flows
  basic-http-flow.json        # HTTP REST API with input validation
  mqtt-pipeline.json          # MQTT subscribe -> process -> publish
  error-handling-flow.json    # Error patterns: catch, retry, DLQ

AGENTS.md                     # Agent routing table and project conventions
SETUP_GUIDE.md                # Installation and configuration guide
.github/
  copilot-instructions.md     # GitHub Copilot background context
```

---

## Node-RED Flow JSON — Key Facts

These conventions apply to all flows produced by the agents in this project:

- Flow files are **JSON arrays** of node objects
- Each node has `id`, `type`, `name`, `x`, `y`, `z` (tab ID), and `wires`
- `wires[outputPortIndex][connectionIndex] = "target-node-id"`
- Configuration values use `"${ENV_VAR_NAME}"` — never hardcoded
- Every tab must have at least one `catch` node for error handling
- All nodes must have a descriptive `name` property

---

## Requirements

| For... | Requires |
|--------|----------|
| `nodered-author` / `nodered-test` | Claude Code or GitHub Copilot (no live Node-RED needed) |
| `nodered-manage` | Running Node-RED instance + `NODE_RED_URL` + `NODE_RED_TOKEN` env vars |
| `nodered-troubleshoot` | Access to flow JSON and/or Node-RED error logs |

---

## Contributing

Contributions welcome! To add support for:
- **New node types:** Add entries to `skills/node-reference.md`
- **New patterns:** Add to `skills/error-handling-patterns.md` or `skills/subflow-patterns.md`
- **New examples:** Add a JSON file to `examples/` following the existing conventions
- **Agent improvements:** Edit the relevant file in `agents/`

Please ensure any contributed flow JSON files:
- Pass validation (test with `nodered-test`)
- Use `${ENV_VAR}` for all configuration values
- Include a comment node and catch node on each tab
- Follow the naming conventions in `AGENTS.md`

---

## License

MIT (c) 2026 Akash Talole

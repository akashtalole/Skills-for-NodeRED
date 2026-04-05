# Setup Guide — Skills for Node-RED

Step-by-step instructions for installing and configuring the Node-RED agent skills
for use with Claude Code and GitHub Copilot.

---

## Prerequisites

| Requirement | Minimum Version | Check Command |
|-------------|-----------------|---------------|
| Node.js | 18.0.0 | `node --version` |
| npm | 9.0.0 | `npm --version` |
| Claude Code | Latest | `claude --version` |
| Node-RED (for manage/troubleshoot) | 3.0.0 | `node-red --version` |

---

## Installation

### Option 1: Clone the Repository (Recommended)

```bash
git clone https://github.com/akashtalole/Skills-for-NodeRED.git
cd Skills-for-NodeRED
```

### Option 2: Use in an Existing Project

Copy the `agents/` and `skills/` folders into your project's root directory, or
add the repository as a git submodule:

```bash
git submodule add https://github.com/akashtalole/Skills-for-NodeRED.git .nodered-skills
```

---

## Configure Claude Code

### Step 1: Add the agents to Claude Code

Claude Code automatically discovers agent skill files in these locations:
- **Project-level:** `agents/*.md` in the current working directory
- **User-level:** `~/.claude/agents/*.md`

If you cloned the repository, open Claude Code from within it:

```bash
cd Skills-for-NodeRED
claude
```

Or, to use the skills across all your projects, copy the agent files:

```bash
cp agents/*.md ~/.claude/agents/
```

### Step 2: Verify Claude Code recognizes the agents

In a Claude Code session, ask:
```
What Node-RED agents do you have available?
```

Claude should respond listing: `nodered-author`, `nodered-manage`, `nodered-test`, `nodered-troubleshoot`.

### Step 3: Test an agent

```
Using nodered-author, create a simple HTTP GET endpoint at /hello that returns {"message": "Hello, Node-RED!"}
```

---

## Configure GitHub Copilot

### Step 1: Open the repository in VS Code

```bash
code Skills-for-NodeRED
```

### Step 2: Confirm Copilot reads the instructions

GitHub Copilot automatically reads `.github/copilot-instructions.md` when the file
exists in the repository root. Open a Copilot Chat session and ask:

```
@workspace What Node-RED agents are available in this project?
```

### Step 3: Use agents from Copilot Chat

In VS Code Copilot Chat, reference agents directly:
```
@workspace Using the nodered-author agent, create an MQTT subscriber flow that...
```

---

## Configure Node-RED for API Access

The `nodered-manage` agent requires a running Node-RED instance with API access enabled.

### Step 1: Enable adminAuth in settings.js

Edit your Node-RED `settings.js` file (usually `~/.node-red/settings.js`):

```javascript
module.exports = {
    // Enable API authentication
    adminAuth: {
        type: "credentials",
        users: [{
            username: "admin",
            // Generate a hash: node-red admin hash-pw
            password: "$2b$08$REPLACE_WITH_BCRYPT_HASH",
            permissions: "*"
        }]
    },

    // Allow environment variable access from flows
    functionGlobalContext: {
        env: process.env
    }
}
```

Generate a bcrypt password hash:
```bash
node-red admin hash-pw
# Enter your password when prompted
# Copy the output hash into the password field above
```

### Step 2: Obtain a Bearer Token

After starting Node-RED with adminAuth enabled:

```bash
curl -X POST http://localhost:1880/auth/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=node-red-admin&grant_type=password&scope=*&username=admin&password=YOUR_PASSWORD"
```

Response:
```json
{
  "access_token": "your-bearer-token-here",
  "expires_in": 604800,
  "token_type": "Bearer"
}
```

### Step 3: Set environment variables

```bash
# Add to your shell profile (~/.bashrc, ~/.zshrc, etc.)
export NODE_RED_URL="http://localhost:1880"
export NODE_RED_TOKEN="your-bearer-token-here"

# Reload your shell
source ~/.bashrc
```

Or create a `.env` file in your project (never commit this file):
```bash
NODE_RED_URL=http://localhost:1880
NODE_RED_TOKEN=your-bearer-token-here
```

### Step 4: Verify connectivity

Test the connection from Claude Code:
```
Using nodered-manage, check if my Node-RED instance is reachable and list the installed palettes.
```

---

## Verify Installation

Run this checklist to confirm everything is set up correctly:

### Claude Code Checklist

- [ ] `agents/nodered-author.md` exists and is readable
- [ ] `agents/nodered-manage.md` exists and is readable
- [ ] `agents/nodered-test.md` exists and is readable
- [ ] `agents/nodered-troubleshoot.md` exists and is readable
- [ ] Claude Code session started from within the repository (or files copied to `~/.claude/agents/`)
- [ ] Test prompt works: `"Using nodered-test, validate the examples/basic-http-flow.json"`

### GitHub Copilot Checklist

- [ ] Repository is open in VS Code
- [ ] `.github/copilot-instructions.md` exists
- [ ] `AGENTS.md` exists
- [ ] Copilot Chat responds with Node-RED context when asked about the project

### Node-RED API Checklist

- [ ] `NODE_RED_URL` is set in environment
- [ ] `NODE_RED_TOKEN` is set in environment
- [ ] `curl ${NODE_RED_URL}/settings` returns 200 with auth token
- [ ] Test prompt works: `"Using nodered-manage, show me all flows on my Node-RED instance"`

---

## Using the Example Flows

Import the example flows directly into Node-RED:

### Via Node-RED UI
1. Open Node-RED in your browser (`http://localhost:1880`)
2. Click the hamburger menu → Import
3. Click "select a file to import"
4. Choose a file from the `examples/` folder
5. Click "Import"

### Via the API (using nodered-manage)
```
Using nodered-manage, import the examples/mqtt-pipeline.json flow into my Node-RED instance.
```

**Note:** Set the required environment variables before importing flows that use `${ENV_VAR}` references.

---

## Troubleshooting Setup

### Claude Code doesn't recognize the agents

**Cause:** The agent files are not in a discoverable location.

**Fix:** Ensure Claude Code is started from within the cloned repository, or copy the files:
```bash
cp agents/*.md ~/.claude/agents/
```

### nodered-manage gets "Unauthorized" errors

**Cause:** `NODE_RED_TOKEN` is missing or incorrect.

**Fix:**
1. Verify the variable is set: `echo $NODE_RED_TOKEN`
2. Re-obtain the token using the auth endpoint (see Step 2 above)
3. Ensure `adminAuth` is correctly configured in `settings.js`

### Node-RED won't start after editing settings.js

**Cause:** JavaScript syntax error in `settings.js`.

**Fix:**
```bash
node -e "require('/path/to/.node-red/settings.js')"
```
This will show any syntax errors without starting Node-RED.

### Environment variables not expanding in flows

**Cause:** The variable is not set in the OS environment when Node-RED starts.

**Fix:**
1. Set the variable before starting Node-RED: `export MY_VAR=value && node-red`
2. Or add to `settings.js` `functionGlobalContext` for persistence
3. Verify with a debug node: set `msg.testVar = env.get('MY_VAR')` in a function node

### Import fails with "Invalid JSON"

**Cause:** The flow JSON has a syntax error.

**Fix:**
```
Using nodered-test, validate examples/mqtt-pipeline.json and report any errors.
```

---

## Security Notes

- **Never commit** `NODE_RED_TOKEN` or passwords to version control
- Add `.env` to your `.gitignore`
- The `flows_cred.json` file contains encrypted credentials — back it up securely, separate from `flows.json`
- Use HTTPS for Node-RED instances accessible over a network (configure TLS in `settings.js`)
- Regularly rotate API tokens; they expire after 7 days by default

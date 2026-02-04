# Clawdbot Sandbox Browser Setup Guide

This guide walks through enabling browser capabilities for a sandboxed Clawdbot agent. The sandboxed browser runs in its own Docker container, isolated from both the host system and the agent's exec sandbox.

## Prerequisites

- Clawdbot installed and running
- Docker installed and accessible
- Access to modify `~/.clawdbot/clawdbot.json`

## Overview

Clawdbot can run in "sandbox mode" where tool execution happens inside Docker containers. By default, the `browser` tool is **denied** in sandbox mode for security reasons. Enabling it requires:

1. Building the sandbox browser Docker image
2. Enabling the browser tool in sandbox policy
3. Enabling the sandbox browser in agent config
4. Restarting the gateway

## Step 1: Build the Sandbox Browser Image

```bash
cd /opt/clawdbot
./scripts/sandbox-browser-setup.sh
```

This creates a Docker image named `clawdbot-sandbox-browser:bookworm-slim` (~1GB) containing Chromium.

Verify it exists:
```bash
docker images | grep -i browser
# Should show: clawdbot-sandbox-browser   bookworm-slim   ...   1.02GB
```

## Step 2: Add Himalaya to Exec Sandbox (Optional)

If you want email CLI capabilities via himalaya, add it to `Dockerfile.sandbox`:

```dockerfile
FROM debian:bookworm-slim

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        bash \
        ca-certificates \
        curl \
        git \
        jq \
        python3 \
        ripgrep \
    && rm -rf /var/lib/apt/lists/*

# Install Himalaya (email CLI)
RUN ARCH=$(dpkg --print-architecture) && \
    case "$ARCH" in \
        amd64) HIMALAYA_ARCH="x86_64-linux" ;; \
        arm64) HIMALAYA_ARCH="aarch64-linux" ;; \
        *) echo "Unsupported arch: $ARCH" && exit 1 ;; \
    esac && \
    curl -fsSL "https://github.com/pimalaya/himalaya/releases/latest/download/himalaya.${HIMALAYA_ARCH}.tgz" | \
    tar -xz -C /usr/local/bin && \
    chmod +x /usr/local/bin/himalaya

CMD ["sleep", "infinity"]
```

Rebuild the exec sandbox:
```bash
cd /opt/clawdbot
docker build -f Dockerfile.sandbox -t clawdbot-sandbox:bookworm-slim .
```

## Step 3: Update Clawdbot Configuration

Edit `~/.clawdbot/clawdbot.json`:

### 3a: Enable Browser in Sandbox Tool Policy

⚠️ **Important:** Setting `tools.sandbox.tools.allow` **replaces** the default list. You must include ALL tools you want the agent to have access to.

```json
{
  "tools": {
    "elevated": {
      "enabled": true,
      "allowFrom": {
        "slack": ["YOUR_SLACK_USER_ID"]
      }
    },
    "sandbox": {
      "tools": {
        "allow": [
          "exec",
          "process",
          "read",
          "write",
          "edit",
          "apply_patch",
          "image",
          "sessions_list",
          "sessions_history",
          "sessions_send",
          "sessions_spawn",
          "session_status",
          "browser"
        ],
        "deny": []
      }
    }
  }
}
```

### 3b: Enable Sandbox Browser in Agent Config

In the `agents.defaults.sandbox` section, add/update the `browser` block:

```json
{
  "agents": {
    "defaults": {
      "sandbox": {
        "mode": "all",
        "workspaceAccess": "rw",
        "docker": {
          "network": "bridge",
          "binds": [
            "/home/clawdbot/homebrew:/home/clawdbot/homebrew:ro",
            "/opt/clawdbot:/opt/clawdbot:ro"
          ]
        },
        "browser": {
          "enabled": true,
          "autoStart": true
        }
      }
    }
  }
}
```

## Step 4: Restart Everything

```bash
# Stop any existing sandbox containers
docker stop $(docker ps -q --filter "name=clawdbot-sbx") 2>/dev/null
docker rm $(docker ps -aq --filter "name=clawdbot-sbx") 2>/dev/null

# Restart the gateway
sudo systemctl restart clawdbot
# or
pkill -f "node.*clawdbot" && sleep 2 && clawdbot gateway start
```

## Step 5: Verify

Have the agent run:
```
browser status (target: sandbox)
```

Should return:
```json
{
  "enabled": true,
  "running": true,
  "cdpReady": true,
  ...
}
```

Test by opening a page:
```
browser open https://example.com (target: sandbox)
browser snapshot
```

## Troubleshooting

### "Tool browser not found"
- The tool list is cached per session
- Kill the sandbox container and restart the gateway
- Verify `clawdbot sandbox explain` shows browser in the allow list

### "Host browser control is disabled by sandbox policy"
- You're targeting the host browser instead of sandbox
- Use `target: sandbox` in browser tool calls

### "Sandbox browser is unavailable"
- Check `sandbox.browser.enabled: true` in config
- Verify the browser image exists: `docker images | grep browser`
- Restart the gateway after config changes

### Browser in both "allow" and "deny" lists
- The deny list shows defaults; your allow should override
- If still blocked, add `"deny": []` to explicitly clear defaults

### Lost other tools after enabling browser
- You set `allow: ["browser"]` which replaced all defaults
- Must include ALL tools: exec, read, write, edit, process, etc.

## Architecture Notes

```
┌─────────────────────────────────────────────────────────┐
│ Host                                                    │
│  ┌─────────────────┐                                    │
│  │ Clawdbot Gateway│ (Node.js process)                  │
│  └────────┬────────┘                                    │
│           │                                             │
│  ┌────────▼────────┐    ┌──────────────────────────┐   │
│  │ Exec Sandbox    │    │ Browser Sandbox          │   │
│  │ (clawdbot-      │    │ (clawdbot-sandbox-       │   │
│  │  sandbox:       │    │  browser:bookworm-slim)  │   │
│  │  bookworm-slim) │    │                          │   │
│  │                 │    │ Contains: Chromium       │   │
│  │ Contains:       │    │ Exposes: CDP on port     │   │
│  │ - bash, curl    │    │                          │   │
│  │ - himalaya      │    │                          │   │
│  │ - your tools    │    │                          │   │
│  └─────────────────┘    └──────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

- **Exec Sandbox**: Where `exec`, `read`, `write` commands run
- **Browser Sandbox**: Separate container running Chromium, controlled via CDP
- **Gateway**: Orchestrates both, routes tool calls appropriately

## Security Considerations

- The sandbox browser is isolated from the host filesystem
- Browser runs headless by default in containers
- Consider what sites/credentials the agent can access
- OAuth tokens obtained through the browser persist in the browser's profile within the container

---

*Last updated: 2026-02-04*
*Tested with Clawdbot v2026.1.24-1*

# Running Clawdbot on DigitalOcean

This guide walks through setting up Clawdbot on a DigitalOcean droplet, with emphasis on understanding the architecture - particularly how the sandbox model works and why you should embrace it rather than fight it.

## The Mental Model (Read This First)

Before diving into commands, understand how Clawdbot is architected:

```
┌─────────────────────────────────────────────────────────────────┐
│ DigitalOcean Droplet (Host)                                     │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ Clawdbot Gateway (Node.js)                               │  │
│  │ - Runs on the HOST                                       │  │
│  │ - Owns config, state, credentials                        │  │
│  │ - Manages sessions, routes messages                      │  │
│  │ - Lives at: ~/.clawdbot/                                 │  │
│  └──────────────────────────────────────────────────────────┘  │
│                          │                                      │
│                          │ spawns                               │
│                          ▼                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ Docker Sandbox Container                                 │  │
│  │ - Where YOUR AGENT actually executes commands            │  │
│  │ - Isolated filesystem (mostly)                           │  │
│  │ - Limited to what you explicitly give it                 │  │
│  │ - Workspace mounted at: /workspace                       │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ~/clawd/              ←→    /workspace (in sandbox)            │
│  (agent workspace)           (same files, mounted rw)           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Key insight:** The Gateway orchestrates everything, but when your agent runs `exec`, `read`, `write`, etc., those commands execute **inside the sandbox container**, not on the host.

### The Right Way to Think About This

❌ **Wrong approach:** "I need to break my agent out of the sandbox to access host tools"

✅ **Right approach:** "I need to give my sandbox the tools it needs"

The sandbox exists for good reason - it limits blast radius when the AI does something dumb. Instead of disabling it, customize it:

- **Need a CLI tool?** Add it to `Dockerfile.sandbox`
- **Need access to host files?** Add bind mounts
- **Need host credentials?** Mount them read-only into the sandbox
- **Need network access?** Configure `docker.network: "bridge"`

Only use `elevated` mode for truly exceptional cases, not as the default.

## Prerequisites

- DigitalOcean account
- SSH key configured
- Domain (optional, but helpful for Slack/webhook integrations)

## Step 1: Create the Droplet

**Recommended specs:**
- **Image:** Ubuntu 24.04 LTS
- **Size:** Basic, 2GB RAM / 1 vCPU minimum (4GB recommended)
- **Region:** Close to you for low latency
- **Authentication:** SSH key

```bash
# SSH into your new droplet
ssh root@your-droplet-ip
```

## Step 2: Create the clawdbot User

Don't run Clawdbot as root:

```bash
# Create user
adduser --disabled-password --gecos "" clawdbot

# Add to docker group (after Docker is installed)
usermod -aG docker clawdbot

# Give sudo access (optional, for maintenance)
usermod -aG sudo clawdbot

# Allow passwordless sudo (optional)
echo "clawdbot ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/clawdbot
```

## Step 3: Install Dependencies

```bash
# Update system
apt update && apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com | bash

# Install Node.js 22
curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
apt install -y nodejs

# Install other useful tools
apt install -y git jq htop
```

## Step 4: Install Clawdbot

Switch to the clawdbot user:

```bash
su - clawdbot
```

Run the installer:

```bash
curl -fsSL https://clawd.bot/install.sh | bash
```

Follow the onboarding wizard. It will:
- Set up your Anthropic API key
- Configure the gateway
- Optionally set up Slack/Discord/WhatsApp

## Step 5: Configure Sandboxing

Edit `~/.clawdbot/clawdbot.json`:

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "anthropic/claude-sonnet-4-20250514"
      },
      "sandbox": {
        "mode": "all",
        "workspaceAccess": "rw",
        "docker": {
          "network": "bridge",
          "binds": [
            "/home/clawdbot/homebrew:/home/clawdbot/homebrew:ro",
            "/opt/clawdbot:/opt/clawdbot:ro"
          ]
        }
      }
    }
  }
}
```

**Key settings:**
- `mode: "all"` - Everything runs in sandbox (recommended)
- `workspaceAccess: "rw"` - Agent can read/write its workspace
- `network: "bridge"` - Sandbox can reach the internet (needed for most tools)
- `binds` - Mount host directories into sandbox (read-only when possible)

## Step 6: Build the Sandbox Image

Clawdbot uses a minimal Debian image by default. Customize it for your needs:

```bash
cd /opt/clawdbot
```

Edit `Dockerfile.sandbox` to add your tools:

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

# Add your custom tools here
# Example: Install himalaya for email
RUN ARCH=$(dpkg --print-architecture) && \
    case "$ARCH" in \
        amd64) HIMALAYA_ARCH="x86_64-linux" ;; \
        arm64) HIMALAYA_ARCH="aarch64-linux" ;; \
    esac && \
    curl -fsSL "https://github.com/pimalaya/himalaya/releases/latest/download/himalaya.${HIMALAYA_ARCH}.tgz" | \
    tar -xz -C /usr/local/bin && \
    chmod +x /usr/local/bin/himalaya

CMD ["sleep", "infinity"]
```

Build it:

```bash
docker build -f Dockerfile.sandbox -t clawdbot-sandbox:bookworm-slim .
```

## Step 7: Set Up the Workspace

The agent's workspace is where it stores files, memory, and configuration:

```bash
mkdir -p ~/clawd
cd ~/clawd

# Create the standard structure
mkdir -p memory docs
```

Create `AGENTS.md`, `SOUL.md`, `USER.md` per Clawdbot conventions.

## Step 8: Start the Gateway

```bash
# Start as a daemon
clawdbot gateway start

# Or run in foreground for debugging
clawdbot gateway --foreground
```

For production, set up systemd:

```bash
clawdbot onboard --install-daemon
```

## Step 9: Connect Your Channels

Configure Slack, Discord, WhatsApp, etc. in `~/.clawdbot/clawdbot.json` under `channels`.

See: [Channels documentation](https://docs.clawd.bot/channels)

## Architecture Deep Dive

### What Lives Where

| Component | Location | Runs On |
|-----------|----------|---------|
| Gateway process | `/opt/clawdbot` | Host |
| Gateway config | `~/.clawdbot/clawdbot.json` | Host |
| Gateway state | `~/.clawdbot/` | Host |
| Agent workspace | `~/clawd` | Host (mounted into sandbox) |
| Tool execution | Docker container | Sandbox |
| Skills | `~/clawd/skills` or `/opt/clawdbot/skills` | Mounted into sandbox |

### The Sandbox Filesystem

When your agent runs a command, it sees:

```
/
├── workspace/           # Your ~/clawd, mounted rw
│   ├── AGENTS.md
│   ├── SOUL.md
│   ├── memory/
│   └── docs/
├── opt/clawdbot/        # Clawdbot installation (if bind-mounted, ro)
├── usr/local/bin/       # Tools installed in Dockerfile.sandbox
└── ...                  # Minimal Debian system
```

The sandbox **cannot** see:
- `~/.clawdbot/` (your config, credentials, state)
- Other users' home directories
- System files outside explicit mounts
- Other Docker containers

### Elevated Mode (Escape Hatch)

Sometimes you need to run something on the host:

```
/elevated full
```

This lets the agent run `exec` commands directly on the host. Use sparingly.

Better alternatives:
1. Add the tool to `Dockerfile.sandbox`
2. Bind-mount what you need
3. Create a script on the host and mount it

## Common Customizations

### Adding Host Tools to Sandbox

Edit `Dockerfile.sandbox`, rebuild:

```bash
docker build -f Dockerfile.sandbox -t clawdbot-sandbox:bookworm-slim .
docker stop $(docker ps -q --filter "name=clawdbot-sbx")
docker rm $(docker ps -aq --filter "name=clawdbot-sbx")
# Gateway will create a fresh container on next command
```

### Mounting Credentials

For tools that need credentials (like `gh` CLI):

```json
"docker": {
  "binds": [
    "/home/clawdbot/.config/gh:/workspace/.config/gh:ro"
  ]
}
```

### Giving Sandbox Network Access

Default is `network: "none"`. For internet access:

```json
"docker": {
  "network": "bridge"
}
```

### Adding Browser Capabilities

See: [Sandbox Browser Setup Guide](./sandbox-browser-setup-guide.md)

## Maintenance

### Updating Clawdbot

```bash
cd /opt/clawdbot
git pull
pnpm install
pnpm build
clawdbot gateway restart
```

### Viewing Logs

```bash
journalctl -u clawdbot -f
# or
tail -f ~/.clawdbot/logs/gateway.log
```

### Checking Health

```bash
clawdbot status
clawdbot health
clawdbot doctor
```

### Backing Up

Critical files to back up:
- `~/.clawdbot/clawdbot.json` (config)
- `~/.clawdbot/credentials/` (API keys, tokens)
- `~/clawd/` (agent workspace, memory)

## Troubleshooting

### "Command not found" in sandbox

The tool isn't installed in `Dockerfile.sandbox`. Add it and rebuild.

### Sandbox can't reach the internet

Set `docker.network: "bridge"` in config.

### Agent can't read host files

Add a bind mount in `docker.binds`.

### Permission denied on mounted files

Check file ownership. The sandbox runs as root by default, but files owned by other users may cause issues. Either:
- `chmod` the files
- Set `docker.user` in config

### Config changes not taking effect

Kill the sandbox container and restart:

```bash
docker stop $(docker ps -q --filter "name=clawdbot-sbx")
docker rm $(docker ps -aq --filter "name=clawdbot-sbx")
clawdbot gateway restart
```

---

## Summary

1. **Gateway runs on host** - orchestrates everything
2. **Agent execution happens in sandbox** - isolated Docker container
3. **Embrace the sandbox** - customize it, don't escape it
4. **Workspace is shared** - `~/clawd` ↔ `/workspace`
5. **Add tools to Dockerfile.sandbox** - not the host
6. **Use bind mounts** - for files/creds the sandbox needs
7. **Elevated is an escape hatch** - not the default

The sandbox model takes some getting used to, but it's the right architecture. Your AI agent will occasionally do dumb things - better to have those mistakes contained.

---

*Last updated: 2026-02-04*
*Tested with Clawdbot v2026.1.24-1 on Ubuntu 24.04*

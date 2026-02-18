# Page: Quick Start

# Quick Start

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [README.md](README.md)
- [assets/avatar-placeholder.svg](assets/avatar-placeholder.svg)
- [docs/channels/zalo.md](docs/channels/zalo.md)
- [docs/channels/zalouser.md](docs/channels/zalouser.md)
- [docs/gateway/doctor.md](docs/gateway/doctor.md)
- [scripts/clawtributors-map.json](scripts/clawtributors-map.json)
- [scripts/update-clawtributors.ts](scripts/update-clawtributors.ts)
- [scripts/update-clawtributors.types.ts](scripts/update-clawtributors.types.ts)
- [src/agents/bash-tools.test.ts](src/agents/bash-tools.test.ts)
- [src/agents/pi-tools-agent-config.test.ts](src/agents/pi-tools-agent-config.test.ts)
- [src/agents/sandbox-skills.test.ts](src/agents/sandbox-skills.test.ts)
- [src/commands/configure.gateway.test.ts](src/commands/configure.gateway.test.ts)
- [src/commands/configure.gateway.ts](src/commands/configure.gateway.ts)
- [src/commands/configure.ts](src/commands/configure.ts)
- [src/commands/doctor.ts](src/commands/doctor.ts)
- [src/commands/onboard-helpers.test.ts](src/commands/onboard-helpers.test.ts)
- [src/commands/onboard-helpers.ts](src/commands/onboard-helpers.ts)
- [src/commands/onboard-interactive.ts](src/commands/onboard-interactive.ts)
- [src/config/config.ts](src/config/config.ts)
- [src/config/merge-config.ts](src/config/merge-config.ts)
- [src/index.test.ts](src/index.test.ts)
- [src/index.ts](src/index.ts)
- [src/wizard/onboarding.gateway-config.test.ts](src/wizard/onboarding.gateway-config.test.ts)
- [src/wizard/onboarding.gateway-config.ts](src/wizard/onboarding.gateway-config.ts)
- [src/wizard/onboarding.ts](src/wizard/onboarding.ts)
- [src/wizard/onboarding.types.ts](src/wizard/onboarding.types.ts)
- [tsconfig.json](tsconfig.json)
- [ui/src/styles.css](ui/src/styles.css)
- [ui/src/styles/layout.mobile.css](ui/src/styles/layout.mobile.css)

</details>



This page provides the minimal path to install OpenClaw, run the onboarding wizard, and send your first message. For detailed installation options, see [Installation](#2). For comprehensive configuration, see [Configuration System](#4).

---

## Prerequisites

OpenClaw requires **Node.js ≥ 22**. Verify your version:

```bash
node --version
```

Supported platforms: **macOS**, **Linux**, and **Windows (via WSL2)**. Windows native support is limited; WSL2 is strongly recommended.

**Sources:** [README.md:47-48](), [src/infra/runtime-guard.ts]()

---

## Installation

Install OpenClaw globally via npm or pnpm:

```bash
npm install -g openclaw@latest
# or
pnpm add -g openclaw@latest
```

Verify installation:

```bash
openclaw --version
```

**Sources:** [README.md:49-54](), [src/index.ts:1-93]()

---

## Onboarding Wizard

The onboarding wizard guides you through initial setup: authentication, model selection, gateway configuration, and channel setup.

### Run the Wizard

```bash
openclaw onboard --install-daemon
```

The `--install-daemon` flag installs the Gateway as a system service (launchd on macOS, systemd on Linux, schtasks on Windows).

**Sources:** [README.md:53-56](), [src/wizard/onboarding.ts:90-483]()

### Wizard Flow

```mermaid
flowchart TD
    Start["openclaw onboard"] --> Header["printWizardHeader<br/>(ASCII art)"]
    Header --> Risk["requireRiskAcknowledgement<br/>(security warning)"]
    Risk --> ConfigCheck["readConfigFileSnapshot<br/>(check existing config)"]
    
    ConfigCheck --> FlowChoice{Flow Selection}
    FlowChoice -->|quickstart| QuickDefaults["Use quickstart defaults:<br/>- Port: 18789<br/>- Bind: loopback<br/>- Auth: token<br/>- Tailscale: off"]
    FlowChoice -->|advanced| ManualPrompt["Prompt for:<br/>- Port<br/>- Bind mode<br/>- Auth (token/password)<br/>- Tailscale mode"]
    
    QuickDefaults --> AuthChoice["promptAuthChoiceGrouped<br/>(OAuth/API key)"]
    ManualPrompt --> AuthChoice
    
    AuthChoice --> ModelSelect["promptDefaultModel<br/>(primary model)"]
    ModelSelect --> GatewayConfig["configureGatewayForOnboarding<br/>(gateway.* settings)"]
    GatewayConfig --> ChannelSetup["setupChannels<br/>(WhatsApp/Telegram/etc)"]
    ChannelSetup --> SkillsSetup["setupSkills<br/>(workspace skills)"]
    SkillsSetup --> WriteConfig["writeConfigFile<br/>(~/.openclaw/openclaw.json)"]
    WriteConfig --> Workspace["ensureWorkspaceAndSessions<br/>(~/openclaw/workspace)"]
    Workspace --> Finalize["finalizeOnboardingWizard<br/>(launch TUI or print next steps)"]
    
    Finalize --> End["Setup complete"]
```

**Wizard modes:**

| Mode         | Description                                          |
|--------------|------------------------------------------------------|
| `quickstart` | Minimal prompts; uses sensible defaults              |
| `advanced`   | Full control over gateway, channels, and skills      |

Specify mode explicitly:

```bash
openclaw onboard --flow quickstart
openclaw onboard --flow advanced
```

**Sources:** [src/wizard/onboarding.ts:90-483](), [src/wizard/onboarding.types.ts:1-26](), [src/wizard/onboarding.gateway-config.ts:42-286]()

### Configuration Output

The wizard writes `~/.openclaw/openclaw.json` with your settings. Minimal example:

```json5
{
  "agents": {
    "defaults": {
      "workspace": "~/openclaw/workspace",
      "model": "anthropic/claude-opus-4-6"
    }
  },
  "gateway": {
    "mode": "local",
    "port": 18789,
    "bind": "loopback",
    "auth": {
      "mode": "token",
      "token": "<generated-token>"
    }
  }
}
```

**Sources:** [src/wizard/onboarding.ts:355-368](), [src/config/config.ts:1-15](), [README.md:315-324]()

---

## Starting the Gateway

After onboarding, the Gateway is installed as a system service. Start it:

```bash
openclaw gateway start
```

Check status:

```bash
openclaw gateway status
```

Expected output:

```
Gateway: running (PID 12345)
Port: 18789
Bind: loopback (127.0.0.1)
Auth: token
```

For manual startup (foreground, verbose logs):

```bash
openclaw gateway run --verbose
```

**Sources:** [README.md:67](), [src/daemon/service.ts](), [src/cli/program.ts]()

### Gateway Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Stopped
    Stopped --> Starting: openclaw gateway start
    Starting --> Running: Service loaded
    Running --> Stopped: openclaw gateway stop
    Running --> Running: openclaw gateway restart
    
    Running --> HealthCheck: openclaw gateway status
    HealthCheck --> Running: Health OK
    HealthCheck --> Unhealthy: Health failed
    Unhealthy --> Restarting: Auto-restart or manual
    Restarting --> Running
    
    note right of Running
        Gateway Server (port 18789)
        WebSocket RPC endpoint
        Channel monitors active
    end note
```

**Sources:** [src/daemon/service.ts](), [src/gateway/server.ts](), [src/commands/gateway-start.ts]()

---

## First Message

Send a message to the agent via the CLI:

```bash
openclaw agent --message "Hello, what can you do?"
```

Or use the `message send` command to deliver via a channel:

```bash
openclaw message send --to +15555550123 --message "Ship checklist"
```

### Message Flow

```mermaid
sequenceDiagram
    participant CLI as openclaw agent
    participant Gateway as Gateway Server<br/>(port 18789)
    participant Agent as runEmbeddedPiAgent<br/>(agent runtime)
    participant Model as Model Provider<br/>(Anthropic/OpenAI)
    participant Session as SessionStore<br/>(JSONL transcripts)
    
    CLI->>Gateway: agent.call<br/>(WebSocket RPC)
    Gateway->>Agent: Create agent session<br/>(sessionKey: agent:main:main)
    Agent->>Agent: Build system prompt<br/>(IDENTITY.md, SKILLS.md)
    Agent->>Model: Prompt with message<br/>(stream: true)
    Model-->>Agent: Response chunks
    Agent->>Session: Append to transcript<br/>(sessions/main.jsonl)
    Agent-->>Gateway: Return response
    Gateway-->>CLI: Display response
```

**Key components:**

- **`openclaw agent`**: CLI command for direct agent interaction ([src/cli/program.ts]())
- **Gateway Server**: WebSocket RPC endpoint at `ws://127.0.0.1:18789` ([src/gateway/server.ts]())
- **`runEmbeddedPiAgent`**: Core agent execution pipeline ([src/agents/pi-agent.ts]())
- **SessionStore**: Persists conversation history to `~/.openclaw/agents/main/sessions/main.jsonl` ([src/config/sessions.ts]())

**Sources:** [README.md:69-74](), [src/agents/pi-agent.ts](), [src/gateway/server.ts](), [src/config/sessions.ts]()

---

## Basic Commands

OpenClaw provides in-chat commands (send via messaging channels) and CLI commands (run in terminal).

### In-Chat Commands

Send these in WhatsApp, Telegram, or any connected channel:

| Command     | Description                               |
|-------------|-------------------------------------------|
| `/status`   | Show session model, token count, and cost |
| `/help`     | List available commands                   |
| `/new`      | Reset session (clear history)             |
| `/compact`  | Compact session (summarize history)       |
| `/verbose`  | Toggle verbose mode                       |

Example:

```
/status
```

Response:

```
Model: anthropic/claude-opus-4-6
Tokens: 1,234 / 200,000
Cost: $0.05
```

**Sources:** [README.md:265-277](), [src/commands/message-commands.ts]()

### CLI Commands

Run these in your terminal:

| Command                          | Description                          |
|----------------------------------|--------------------------------------|
| `openclaw gateway status`        | Check Gateway health                 |
| `openclaw agent --message "..."`  | Send message to agent                |
| `openclaw channels status`       | List active channels                 |
| `openclaw sessions list`         | List active sessions                 |
| `openclaw logs --follow`         | Tail Gateway logs                    |
| `openclaw doctor`                | Run health checks and repairs        |

Example:

```bash
openclaw channels status
```

**Sources:** [src/cli/program.ts](), [README.md:265-277]()

---

## Command Execution Map

```mermaid
graph TB
    subgraph "CLI Entry Point"
        Main["src/index.ts<br/>(isMainModule)"]
        Main --> BuildProgram["buildProgram<br/>(src/cli/program.ts)"]
    end
    
    subgraph "Command Handlers"
        BuildProgram --> OnboardCmd["onboard command<br/>(runOnboardingWizard)"]
        BuildProgram --> GatewayCmd["gateway command<br/>(gatewayCommand)"]
        BuildProgram --> AgentCmd["agent command<br/>(agentCommand)"]
        BuildProgram --> DoctorCmd["doctor command<br/>(doctorCommand)"]
    end
    
    subgraph "Core Systems"
        OnboardCmd --> ConfigIO["Config I/O<br/>(src/config/io.ts)"]
        OnboardCmd --> Workspace["Workspace Setup<br/>(src/agents/workspace.ts)"]
        
        GatewayCmd --> GatewayServer["Gateway Server<br/>(src/gateway/server.ts)"]
        GatewayCmd --> DaemonService["Daemon Service<br/>(src/daemon/service.ts)"]
        
        AgentCmd --> AgentRuntime["Agent Runtime<br/>(src/agents/pi-agent.ts)"]
        AgentCmd --> SessionStore["Session Store<br/>(src/config/sessions.ts)"]
        
        DoctorCmd --> ConfigValidation["Config Validation<br/>(src/config/validation.ts)"]
        DoctorCmd --> HealthCheck["Health Check<br/>(src/commands/doctor-gateway-health.ts)"]
    end
    
    subgraph "Configuration"
        ConfigIO --> ConfigFile["~/.openclaw/openclaw.json"]
        ConfigIO --> ZodSchema["OpenClawSchema<br/>(src/config/zod-schema.ts)"]
    end
    
    subgraph "State"
        SessionStore --> Transcripts["~/.openclaw/agents/main/sessions/<br/>*.jsonl"]
        Workspace --> WorkspaceFiles["~/openclaw/workspace/<br/>IDENTITY.md, SKILLS.md"]
    end
```

**Sources:** [src/index.ts:1-93](), [src/cli/program.ts](), [src/wizard/onboarding.ts:90-483](), [src/commands/gateway-start.ts](), [src/commands/doctor.ts:65-313]()

---

## Next Steps

You now have OpenClaw running with a configured Gateway. Explore these topics next:

- **[Key Concepts](#1.1)**: Understand Agents, Sessions, Channels, and Tools
- **[Gateway Configuration](#3.1)**: Customize port, bind mode, and authentication
- **[Channel Setup](#8)**: Connect WhatsApp, Telegram, Discord, and other channels
- **[Tool System](#6)**: Enable file operations, shell execution, and memory search
- **[Multi-Agent Configuration](#4.3)**: Route channels to isolated agent workspaces

For troubleshooting, run:

```bash
openclaw doctor
```

This checks configuration health, migrates legacy settings, and suggests repairs.

**Sources:** [README.md:410-427](), [docs/gateway/doctor.md:1-283]()

---
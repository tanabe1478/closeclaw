# Page: Platform-Specific Commands

# Platform-Specific Commands

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [docs/tools/slash-commands.md](docs/tools/slash-commands.md)
- [src/auto-reply/command-detection.ts](src/auto-reply/command-detection.ts)
- [src/auto-reply/commands-args.ts](src/auto-reply/commands-args.ts)
- [src/auto-reply/commands-registry.data.ts](src/auto-reply/commands-registry.data.ts)
- [src/auto-reply/commands-registry.test.ts](src/auto-reply/commands-registry.test.ts)
- [src/auto-reply/commands-registry.ts](src/auto-reply/commands-registry.ts)
- [src/auto-reply/commands-registry.types.ts](src/auto-reply/commands-registry.types.ts)
- [src/auto-reply/group-activation.ts](src/auto-reply/group-activation.ts)
- [src/auto-reply/reply.ts](src/auto-reply/reply.ts)
- [src/auto-reply/reply/commands-core.ts](src/auto-reply/reply/commands-core.ts)
- [src/auto-reply/reply/commands-status.ts](src/auto-reply/reply/commands-status.ts)
- [src/auto-reply/reply/commands-subagents.ts](src/auto-reply/reply/commands-subagents.ts)
- [src/auto-reply/reply/commands.test.ts](src/auto-reply/reply/commands.test.ts)
- [src/auto-reply/reply/commands.ts](src/auto-reply/reply/commands.ts)
- [src/auto-reply/reply/directive-handling.ts](src/auto-reply/reply/directive-handling.ts)
- [src/auto-reply/reply/subagents-utils.test.ts](src/auto-reply/reply/subagents-utils.test.ts)
- [src/auto-reply/reply/subagents-utils.ts](src/auto-reply/reply/subagents-utils.ts)
- [src/auto-reply/send-policy.ts](src/auto-reply/send-policy.ts)
- [src/auto-reply/status.test.ts](src/auto-reply/status.test.ts)
- [src/auto-reply/status.ts](src/auto-reply/status.ts)
- [src/auto-reply/templating.ts](src/auto-reply/templating.ts)
- [src/discord/monitor.ts](src/discord/monitor.ts)
- [src/imessage/monitor.ts](src/imessage/monitor.ts)
- [src/signal/monitor.ts](src/signal/monitor.ts)
- [src/slack/monitor.ts](src/slack/monitor.ts)
- [src/telegram/bot.test.ts](src/telegram/bot.test.ts)
- [src/telegram/bot.ts](src/telegram/bot.ts)
- [src/web/auto-reply.ts](src/web/auto-reply.ts)
- [src/web/inbound.media.test.ts](src/web/inbound.media.test.ts)
- [src/web/inbound.test.ts](src/web/inbound.test.ts)
- [src/web/inbound.ts](src/web/inbound.ts)
- [src/web/test-helpers.ts](src/web/test-helpers.ts)
- [src/web/vcard.ts](src/web/vcard.ts)

</details>



This page documents how OpenClaw implements native slash commands on platforms that support them (Discord, Telegram, Slack). Native commands provide enhanced user experience through autocomplete, argument menus, and platform-specific UI components.

For information about text command parsing and directives (inline `/think`, `/model`, etc.), see [Directives](#9.3). For the complete list of available commands, see [Command Reference](#9.1).

---

## Command Registry Architecture

OpenClaw uses a unified command registry that defines commands once and adapts them to platform-specific formats. Commands are categorized by scope: `text` (text parsing only), `native` (platform native only), or `both`.

### Command Definition Structure

```mermaid
graph TB
    subgraph "Command Definition"
        ChatCmd["ChatCommandDefinition"]
        Key["key: string"]
        NativeName["nativeName?: string"]
        TextAliases["textAliases: string[]"]
        Args["args?: CommandArgDefinition[]"]
        Scope["scope: text | native | both"]
    end
    
    subgraph "Command Registry"
        BuildFn["buildChatCommands()"]
        Cache["cachedCommands"]
        Registry["getActivePluginRegistry()"]
    end
    
    subgraph "Platform Adapters"
        TelegramAdapter["registerTelegramNativeCommands()"]
        DiscordAdapter["createDiscordNativeCommand()"]
        SlackAdapter["registerSlackMonitorSlashCommands()"]
    end
    
    ChatCmd --> Key
    ChatCmd --> NativeName
    ChatCmd --> TextAliases
    ChatCmd --> Args
    ChatCmd --> Scope
    
    BuildFn --> Cache
    Registry --> BuildFn
    
    Cache --> TelegramAdapter
    Cache --> DiscordAdapter
    Cache --> SlackAdapter
    
    TelegramAdapter --> TelegramSpec["BotCommand[]"]
    DiscordAdapter --> DiscordSpec["Carbon Command"]
    SlackAdapter --> SlackSpec["Bolt SlashCommand"]
```

**Sources:** [src/auto-reply/commands-registry.ts:71-149](), [src/auto-reply/commands-registry.data.ts:26-46]()

### Command Resolution Flow

```mermaid
sequenceDiagram
    participant Platform
    participant Registry as Command Registry
    participant Filter as Config Filter
    participant Adapter as Platform Adapter
    participant Native as Native API
    
    Platform->>Registry: getChatCommands()
    Registry->>Registry: buildChatCommands()
    Registry->>Filter: listChatCommandsForConfig(cfg)
    Filter->>Filter: isCommandEnabled(cfg, key)
    Note over Filter: Filters by commands.config<br/>commands.debug, etc.
    Filter->>Adapter: filtered commands
    Adapter->>Adapter: listNativeCommandSpecsForConfig()
    Note over Adapter: Filter scope !== "text"<br/>Apply provider overrides
    Adapter->>Native: Register commands
    Native-->>Platform: Native command specs
```

**Sources:** [src/auto-reply/commands-registry.ts:92-149](), [src/auto-reply/commands-registry.data.ts:122-149]()

---

## Telegram Native Commands

Telegram commands are registered via the Bot API's `setMyCommands` endpoint. The implementation uses grammY's throttler and sequentialization for rate-limiting and ordered message processing.

### Registration Process

```mermaid
graph TB
    Start["createTelegramBot()"]
    Native["listNativeCommandSpecsForConfig(cfg)"]
    Skills["listSkillCommandsForAgents({cfg})"]
    Custom["cfg.channels.telegram.customCommands"]
    
    Merge["Merge Command Lists"]
    CheckCollision["Check Native Collision"]
    LogError["runtime.error(collision warning)"]
    FilterValid["Filter Valid Custom Commands"]
    
    Normalize["Normalize command names<br/>(lowercase, strip leading /)"]
    Register["bot.api.setMyCommands(commands)"]
    
    Start --> Native
    Start --> Skills
    Start --> Custom
    
    Native --> Merge
    Skills --> Merge
    Custom --> CheckCollision
    CheckCollision -->|Collision| LogError
    CheckCollision -->|Valid| FilterValid
    LogError --> FilterValid
    FilterValid --> Merge
    Merge --> Normalize
    Normalize --> Register
```

**Sources:** [src/telegram/bot-native-commands.ts:22-119](), [src/telegram/bot.test.ts:217-288]()

Telegram commands are set at bot startup:

1. **Native Commands** - `listNativeCommandSpecsForConfig()` retrieves all enabled built-in commands
2. **Skill Commands** - `listSkillCommandsForAgents()` generates commands for user-invocable skills
3. **Custom Commands** - `channels.telegram.customCommands` allows additional commands
4. **Collision Detection** - Custom commands that collide with native commands are filtered out and logged as errors
5. **Normalization** - Command names are normalized: lowercased, leading `/` stripped
6. **Registration** - Final command list sent to Telegram via `bot.api.setMyCommands()`

**Custom Command Validation:**
- Commands are normalized to lowercase with leading `/` stripped: `/Custom_Backup` → `custom_backup`
- Collisions with reserved native command names trigger `runtime.error()` and are ignored
- Non-colliding custom commands are appended to the native command list

### Command Handling

Telegram native commands trigger the `bot.command()` handler, which routes to the unified command processor:

```mermaid
graph TB
    Input["User types /status"]
    BotCommand["bot.command('status')"]
    SeqKey["getTelegramSequentialKey(ctx)"]
    Dedupe["shouldSkipUpdate(ctx)"]
    AuthCheck["Check allowlist/policy"]
    MenuCheck["resolveCommandArgMenu()"]
    
    ProcessNative["processNativeCommand()"]
    DispatchReply["dispatchReplyWithDispatcher()"]
    SendMessage["sendTelegramMessage()"]
    
    Input --> BotCommand
    BotCommand --> SeqKey
    SeqKey --> Dedupe
    Dedupe -->|Not duplicate| AuthCheck
    Dedupe -->|Duplicate| Skip["Skip (already processed)"]
    AuthCheck -->|Authorized| MenuCheck
    AuthCheck -->|Unauthorized| Block["Block (silent)"]
    MenuCheck -->|Missing args| ButtonMenu["Send inline button menu"]
    MenuCheck -->|Args complete| ProcessNative
    ProcessNative --> DispatchReply
    DispatchReply --> SendMessage
```

**Key Implementation Details:**

| Feature | Implementation | Code Reference |
|---------|---------------|----------------|
| **Sequential Processing** | `sequentialize(getTelegramSequentialKey)` ensures messages from same chat are processed in order | [src/telegram/bot.ts:148]() |
| **Sequential Key Pattern** | `telegram:{chatId}` for DMs, `telegram:{chatId}:topic:{threadId}` for forum topics | [src/telegram/bot.ts:67-110]() |
| **Deduplication** | `createTelegramUpdateDedupe()` prevents duplicate update processing via `updateId` tracking | [src/telegram/bot.ts:154-183]() |
| **Argument Parsing** | Commands with args use inline button menus when arguments are missing | [src/telegram/bot-native-commands.ts:154-298]() |
| **Callback Queries** | Button clicks trigger `bot.on("callback_query")` handler, re-routing to command execution | [src/telegram/bot.ts:364-476]() |

**Special Callback Handlers:**
- **Pagination Callbacks:** `/commands` with many commands shows paginated list with `commands_page_{N}` callback data
- **Arg Menu Callbacks:** Generic argument selection buttons with callback format `{commandKey}:{argValue}`

**Sources:** [src/telegram/bot.ts:67-110](), [src/telegram/bot.ts:148-224](), [src/telegram/bot-native-commands.ts:1-298]()

### Argument Menus

When a command requires arguments and none are provided, Telegram shows inline button menus:

```mermaid
sequenceDiagram
    participant User
    participant TelegramBot as "bot.command()"
    participant MenuResolver as "resolveCommandArgMenu()"
    participant ChoiceResolver as "resolveCommandArgChoices()"
    participant Sender as "sendTelegramInlineButtonMenu()"
    participant CallbackHandler as "bot.on('callback_query')"
    
    User->>TelegramBot: /usage
    TelegramBot->>MenuResolver: Check for missing args
    MenuResolver->>ChoiceResolver: Resolve choices for 'mode' arg
    ChoiceResolver-->>MenuResolver: ["off", "tokens", "full", "cost"]
    MenuResolver-->>TelegramBot: Menu spec returned
    TelegramBot->>Sender: Build inline keyboard
    Sender->>User: Display button menu
    
    User->>CallbackHandler: Click "tokens" button
    CallbackHandler->>CallbackHandler: parseCallbackData()
    CallbackHandler->>TelegramBot: Re-invoke with args
    TelegramBot->>User: Execute command
```

**Example:** `/usage` command without args shows buttons: `off | tokens | full | cost`

**Button Layout:**
- Up to 3 buttons per row (configurable)
- Callback data format: `{commandKey}:{argValue}` (e.g., `usage:tokens`)
- Button clicks trigger `callback_query` event, which re-routes through command handler

**Sources:** [src/telegram/bot-native-commands.ts:154-298](), [src/telegram/send.ts:320-402]()

---

## Discord Native Commands

Discord commands use the `@buape/carbon` library, which provides a higher-level abstraction over Discord's slash command API. Commands support autocomplete, choice lists, and button interactions.

### Command Registration

```mermaid
graph TB
    subgraph "Discord Command Creation"
        Define["createDiscordNativeCommand()"]
        Options["buildDiscordCommandOptions()"]
        Handler["Command.handle()"]
    end
    
    subgraph "Option Types"
        String["ApplicationCommandOptionType.String"]
        Number["ApplicationCommandOptionType.Number"]
        Boolean["ApplicationCommandOptionType.Boolean"]
    end
    
    subgraph "Choice Resolution"
        Static["Static choices (≤25)"]
        Autocomplete["Dynamic autocomplete (>25)"]
        Function["CommandArgChoicesProvider"]
    end
    
    Define --> Options
    Options --> String
    Options --> Number
    Options --> Boolean
    
    String --> Static
    String --> Autocomplete
    Autocomplete --> Function
```

**Sources:** [src/discord/monitor/native-command.ts:60-118]()

Discord commands are created as Carbon `Command` instances:

```typescript
// Simplified structure from src/discord/monitor/native-command.ts
class DiscordNativeCommand extends Command {
  name = command.nativeName;
  description = command.description;
  options = buildDiscordCommandOptions({ command, cfg });
  
  async run(interaction: CommandInteraction) {
    // Access control checks
    // Argument parsing
    // Command dispatch
  }
}
```

### Argument Autocomplete

For commands with large choice lists (>25 items), Discord provides autocomplete:

**Autocomplete Flow:**

1. User types command → Discord shows autocomplete
2. User types partial match → `AutocompleteInteraction` fired
3. `resolveCommandArgChoices()` called with context
4. Filtered choices returned (max 25)

**Example:** `/model` command provides autocomplete for available models, filtered by user input.

**Sources:** [src/discord/monitor/native-command.ts:86-102]()

### Button Interactions for Argument Menus

When a command supports argument menus and no argument is provided, Discord shows ephemeral button menus:

```mermaid
sequenceDiagram
    participant User
    participant Command as Discord Command
    participant Menu as Argument Menu Builder
    participant Button as Button Interaction
    participant Dispatch as dispatchReplyWithDispatcher
    
    User->>Command: /usage (no args)
    Command->>Menu: resolveCommandArgMenu()
    Menu->>Menu: buildDiscordCommandArgCustomId()
    Menu-->>Command: Button rows
    Command->>User: Ephemeral message with buttons
    User->>Button: Click "tokens" button
    Button->>Button: parseDiscordCommandArgData()
    Button->>Dispatch: Execute with args
    Dispatch-->>User: Command result
```

**Button Custom ID Format:** `cmdarg:command={cmd};arg={name};value={val};user={id}`

**Sources:** [src/discord/monitor/native-command.ts:199-230](), [src/discord/monitor/native-command.ts:450-550]()

### Provider-Specific Name Overrides

Discord reserves certain command names. OpenClaw applies overrides via `NATIVE_NAME_OVERRIDES`:

| Command Key | Default Name | Discord Override |
|-------------|--------------|------------------|
| `tts` | `/tts` | `/voice` |

**Implementation:** [src/auto-reply/commands-registry.ts:108-121]()

---

## Slack Slash Commands

Slack supports two modes: **legacy single command** (`/openclaw`) and **native per-command** slash commands. The native mode requires manually creating each command in the Slack App configuration.

### Configuration Modes

```mermaid
graph TB
    subgraph "Legacy Mode (Single Command)"
        LegacyCmd["/openclaw <subcommand> <args>"]
        LegacyMatcher["buildSlackSlashCommandMatcher()"]
        LegacyParse["Extract subcommand from text"]
    end
    
    subgraph "Native Mode (Per-Command)"
        NativeHelp["/help"]
        NativeStatus["/status"]
        NativeModel["/model"]
        NativeOther["... (one per command)"]
        NativeMatcher["Match command.command_id"]
    end
    
    subgraph "Configuration Required"
        SlackApp["Slack App Dashboard"]
        ManualCreate["Manually create each<br/>slash command"]
    end
    
    LegacyCmd --> LegacyMatcher
    LegacyMatcher --> LegacyParse
    
    NativeHelp --> NativeMatcher
    NativeStatus --> NativeMatcher
    NativeModel --> NativeMatcher
    NativeOther --> NativeMatcher
    
    SlackApp --> NativeHelp
    SlackApp --> NativeStatus
    ManualCreate --> SlackApp
```

**Configuration:**

- **Legacy:** `channels.slack.slashCommand.command = "/openclaw"` (default)
- **Native:** `commands.native = true` or `channels.slack.commands.native = true`
  - **Important:** Native mode requires creating each command in the Slack App dashboard
  - Slack does not provide an API for programmatic command registration

**Command Matching:**
- **Legacy:** Uses `buildSlackSlashCommandMatcher()` to parse subcommand from text (e.g., `/openclaw status` → `status` command)
- **Native:** Matches `command.command_id` directly to command names

**Sources:** [src/slack/monitor/slash.ts:132-180](), [src/slack/monitor/commands.ts:1-48](), [docs/tools/slash-commands.md:52-56]()

### Argument Menu Implementation

Slack uses Block Kit buttons for argument selection:

```mermaid
graph TB
    MenuCheck["resolveCommandArgMenu()"]
    BuildBlocks["buildSlackCommandArgMenuBlocks()"]
    
    subgraph "Block Kit Structure"
        Section["Section block (title)"]
        Actions["Actions blocks (buttons)"]
        Chunk["chunk choices into rows of 5"]
    end
    
    subgraph "Button Encoding"
        ActionId["action_id: SLACK_COMMAND_ARG_ACTION_ID"]
        Value["value: encodeSlackCommandArgValue()"]
        Format["Format: cmdarg|cmd|arg|val|userId"]
        URLEncode["URL-encode each component"]
    end
    
    MenuCheck --> BuildBlocks
    BuildBlocks --> Section
    BuildBlocks --> Actions
    Actions --> Chunk
    
    Chunk --> ActionId
    Chunk --> Value
    Value --> Format
    Format --> URLEncode
```

**Implementation Details:**

```typescript
// From src/slack/monitor/slash.ts:46-92
const SLACK_COMMAND_ARG_ACTION_ID = "openclaw_command_arg_select";

function encodeSlackCommandArgValue(params: {
  command: string;
  arg: string;
  value: string;
  userId: string;
}): string {
  const components = [
    "cmdarg",
    encodeURIComponent(params.command),
    encodeURIComponent(params.arg),
    encodeURIComponent(params.value),
    encodeURIComponent(params.userId),
  ];
  return components.join("|");
}
```

**Button Layout:**
- Maximum 5 buttons per action block (Slack Block Kit limit)
- Buttons chunked into rows using `chunkItems(choices, 5)`
- Each button's `value` encodes command context for callback routing

**Decoding on Callback:**
- `block_actions` event triggered when button clicked
- `action_id` matched to `SLACK_COMMAND_ARG_ACTION_ID`
- `value` decoded using `parseSlackCommandArgData()`
- Command re-executed with selected argument

**Sources:** [src/slack/monitor/slash.ts:46-130](), [src/slack/monitor/slash.ts:279-383]()

### Slash Command Handler Flow

```mermaid
sequenceDiagram
    participant User
    participant Slack
    participant Handler as SlashCommand Handler
    participant Match as Matcher
    participant Auth as Access Control
    participant Menu as Arg Menu
    participant Dispatch
    
    User->>Slack: /status
    Slack->>Handler: SlackCommandMiddlewareArgs
    Handler->>Handler: await ack()
    Handler->>Match: buildSlackSlashCommandMatcher()
    Match-->>Handler: command matched
    Handler->>Auth: Check allowlist/policy
    
    alt Unauthorized
        Auth-->>Handler: reject
        Handler->>Slack: respond (error)
    else Authorized
        Auth-->>Handler: allow
        Handler->>Menu: resolveCommandArgMenu()
        
        alt Missing Args
            Menu-->>Handler: menu spec
            Handler->>Slack: respond (button menu)
            User->>Slack: Click button
            Slack->>Handler: block_actions
        else Args Complete
            Menu-->>Handler: null
            Handler->>Dispatch: dispatchReplyWithDispatcher()
            Dispatch-->>Slack: deliver replies
        end
    end
```

**Sources:** [src/slack/monitor/slash.ts:147-400]()

---

## Command Argument System

All platforms share a common argument definition and resolution system through the command registry.

### Argument Definition Structure

```mermaid
classDiagram
    class CommandArgDefinition {
        +string name
        +string description
        +CommandArgType type
        +boolean required
        +CommandArgChoice[] | CommandArgChoicesProvider choices
        +boolean captureRemaining
    }
    
    class ChatCommandDefinition {
        +string key
        +string nativeName
        +CommandArgDefinition[] args
        +CommandArgsParsing argsParsing
        +CommandArgMenuSpec | "auto" argsMenu
    }
    
    class CommandArgs {
        +string raw
        +CommandArgValues values
    }
    
    class CommandArgValues {
        +Record~string, string | number | boolean~ values
    }
    
    ChatCommandDefinition --> CommandArgDefinition
    ChatCommandDefinition --> CommandArgs
    CommandArgs --> CommandArgValues
```

**Core Types:**

| Type | Values | Purpose |
|------|--------|---------|
| `CommandArgType` | `"string"` \| `"number"` \| `"boolean"` | Argument data type |
| `CommandArgsParsing` | `"none"` \| `"positional"` | Parsing strategy |
| `CommandArgMenuSpec` | `{ arg: string; title?: string }` \| `"auto"` | Menu configuration |

**Sources:** [src/auto-reply/commands-registry.types.ts:1-82]()

### Argument Parsing Modes

| Mode | Description | Implementation | Example |
|------|-------------|----------------|---------|
| `none` | No structured parsing, `args.raw` only | `parseCommandArgs()` returns `{ raw }` | `/compact [instructions]` |
| `positional` | Parse args by position | `parsePositionalArgs()` splits on whitespace | `/debug set tools.bash=true` |

**Parsing Pipeline:**

```mermaid
graph LR
    RawText["Raw text: 'set path value'"]
    ParseArgs["parseCommandArgs(command, raw)"]
    
    CheckParsing{argsParsing?}
    NoneMode["Return { raw }"]
    PositionalMode["parsePositionalArgs()"]
    
    SplitTokens["Split on whitespace"]
    MapToArgs["Map to arg definitions"]
    CaptureRemaining["captureRemaining: join rest"]
    
    Result["{ raw, values }"]
    
    RawText --> ParseArgs
    ParseArgs --> CheckParsing
    CheckParsing -->|"none"| NoneMode
    CheckParsing -->|"positional"| PositionalMode
    
    PositionalMode --> SplitTokens
    SplitTokens --> MapToArgs
    MapToArgs --> CaptureRemaining
    CaptureRemaining --> Result
    NoneMode --> Result
```

**Key Functions:**

- **`parseCommandArgs()`** - Entry point: [src/auto-reply/commands-registry.ts:235-250]()
- **`parsePositionalArgs()`** - Positional parser: [src/auto-reply/commands-registry.ts:185-206]()
- **`serializeCommandArgs()`** - Reverse conversion: [src/auto-reply/commands-registry.ts:252-270]()
- **`resolveCommandArgChoices()`** - Resolve static or dynamic choices: [src/auto-reply/commands-registry.ts:297-325]()
- **`resolveCommandArgMenu()`** - Determine if arg menu should be shown: [src/auto-reply/commands-registry.ts:327-363]()

**Sources:** [src/auto-reply/commands-registry.ts:185-363]()

### Dynamic Choice Providers

Commands can provide dynamic choices based on context:

```typescript
// Example from commands-registry.data.ts
{
  name: "level",
  type: "string",
  choices: ({ provider, model }) => {
    return listThinkingLevels({ provider, model });
  }
}
```

**Choice Resolution Context:**

- `cfg` - Current OpenClawConfig
- `provider` - Current model provider
- `model` - Current model
- `command` - Command definition
- `arg` - Argument definition

**Sources:** [src/auto-reply/commands-registry.ts:259-286](), [src/auto-reply/commands-registry.data.ts:1-300]()

---

## Platform Capability Comparison

| Feature | Discord | Telegram | Slack |
|---------|---------|----------|-------|
| **Native Registration** | Automatic (Carbon) | `bot.api.setMyCommands()` | Manual Slack App config |
| **Registration Timing** | On client ready | On bot startup | Pre-configured (manual) |
| **Autocomplete** | Yes (>25 choices) | No | No |
| **Argument Menus** | Ephemeral buttons | Inline buttons | Block Kit buttons |
| **Menu Button Layout** | Up to 5 columns | Up to 3 per row | Up to 5 per row |
| **Multi-Agent Support** | Session per user | Targets chat session | Session per user |
| **Custom Commands** | Via Carbon Command class | Via `customCommands` config | Via Slack App |
| **Choice Limit** | 25 per autocomplete | Unlimited inline buttons | 25 per action block |
| **Command Scope** | Guild/Global | Bot-wide | Workspace-wide |
| **Pagination Support** | No | Yes (via callback_query) | No |
| **Session Key Pattern** | `discord:slash:{userId}` | `CommandTargetSessionKey` | `slack:slash:{userId}` |

**Implementation References:**

| Platform | Registration Code | Handler Code |
|----------|------------------|--------------|
| Discord | [src/discord/monitor/native-command.ts:60-118]() | [src/discord/monitor/native-command.ts:199-450]() |
| Telegram | [src/telegram/bot-native-commands.ts:22-119]() | [src/telegram/bot-native-commands.ts:154-298]() |
| Slack | Manual (Slack App UI) | [src/slack/monitor/slash.ts:147-400]() |

**Sources:** [src/discord/monitor/native-command.ts:60-450](), [src/telegram/bot-native-commands.ts:22-298](), [src/slack/monitor/slash.ts:147-400]()

---

## Configuration Reference

### Enabling Native Commands

```json5
{
  commands: {
    // Global setting
    native: "auto",  // "auto" | true | false
    
    // Per-provider overrides
  },
  channels: {
    discord: {
      commands: {
        native: true,        // Override global
        nativeSkills: true   // Register skill commands
      }
    },
    telegram: {
      commands: {
        native: "auto"
      },
      customCommands: [
        { command: "backup", description: "Backup data" }
      ]
    },
    slack: {
      commands: {
        native: false  // Use legacy /openclaw mode
      },
      slashCommand: {
        command: "/openclaw",
        sessionPrefix: "slack:slash"
      }
    }
  }
}
```

**Auto Behavior:**

- **Discord:** On by default
- **Telegram:** On by default
- **Slack:** Off by default (requires manual Slack App configuration)

**Sources:** [docs/tools/slash-commands.md:26-51](), [src/config/commands.ts:1-100]()

### Native Skills Commands

Skills declared with `user-invocable: true` are automatically registered as native commands:

```json5
{
  commands: {
    nativeSkills: "auto"  // "auto" | true | false
  }
}
```

**Skill Command Generation:**

```mermaid
graph TB
    LoadSkills["loadSkillsForAgents()"]
    Filter["Filter user-invocable skills"]
    Sanitize["sanitizeSkillName()"]
    
    subgraph "Name Sanitization"
        Lower["Convert to lowercase"]
        Replace["Replace non-alphanumeric with _"]
        Trim["Truncate to 32 chars"]
        Dedupe["Add numeric suffix if collision"]
    end
    
    BuildDef["buildSkillCommandDefinitions()"]
    Register["Register as native commands"]
    
    LoadSkills --> Filter
    Filter --> Sanitize
    Sanitize --> Lower
    Lower --> Replace
    Replace --> Trim
    Trim --> Dedupe
    Dedupe --> BuildDef
    BuildDef --> Register
```

**Skill Command Naming:**

1. **Sanitization:** Skill name converted to `[a-z0-9_]` (max 32 chars)
2. **Collision Handling:** Numeric suffixes added: `skill_name`, `skill_name_2`, `skill_name_3`, etc.
3. **Command Routing:** 
   - Default: Forwards to model as normal request
   - `command-dispatch: tool`: Routes directly to skill tool (no model)

**Example:** Skill `my-skill-name` → command `/my_skill_name`

**Sources:** [docs/tools/slash-commands.md:52-59](), [src/auto-reply/skill-commands.ts:1-137](), [src/auto-reply/commands-registry.ts:72-84]()

---

## Implementation Notes

### Session Keys by Platform

Native commands use isolated session keys to avoid conflicts with text chat sessions:

| Platform | Session Key Pattern | Purpose |
|----------|---------------------|---------|
| Discord | `agent:{agentId}:discord:slash:{userId}` | Per-user slash command session |
| Telegram | `telegram:slash:{userId}` + CommandTargetSessionKey | Routes to chat session |
| Slack | `agent:{agentId}:slack:slash:{userId}` | Per-user slash command session |

**Exception:** `/stop` command targets the active chat session to abort the current run.

**Sources:** [docs/tools/slash-commands.md:183-190]()

### Access Control

Native commands respect the same access control as text commands:

1. **Allowlist/Policy Check** - `resolveCommandAuthorizedFromAuthorizers()`
2. **Group Policy** - `isDiscordGroupAllowedByPolicy()`, `isSlackChannelAllowedByPolicy()`
3. **DM Policy** - `dmPolicy: "pairing" | "allowlist" | "open" | "disabled"`

Unauthorized command invocations are silently ignored.

**Sources:** [src/discord/monitor/native-command.ts:200-250](), [src/slack/monitor/slash.ts:200-300](), [src/telegram/bot-native-commands.ts:50-150]()

### Error Handling

**Discord:** Unknown interaction errors (10062) are caught and logged as warnings (interaction expired).

**Telegram:** Rate limits handled by `apiThrottler()` middleware.

**Slack:** Ack timeout handling with early `await ack()`.

**Sources:** [src/discord/monitor/native-command.ts:170-197](), [src/telegram/bot.ts:142-147](), [src/slack/monitor/slash.ts:155-160]()

---
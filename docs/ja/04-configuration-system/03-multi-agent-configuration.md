---
title: "マルチエージェント設定"
original_title: "Multi-Agent Configuration"
source: "deepwiki:openclaw/openclaw"
chapter: 4
section: 3
---

# Page: Multi-Agent Configuration

# マルチエージェント設定

<details>
<summary>関連ソースファイル</summary>

この Wiki ページの生成に使用されたコンテキストファイル:

- [CHANGELOG.md](CHANGELOG.md)
- [docs/cli/memory.md](docs/cli/memory.md)
- [docs/cli/sandbox.md](docs/cli/sandbox.md)
- [docs/concepts/memory.md](docs/concepts/memory.md)
- [docs/gateway/configuration.md](docs/gateway/configuration.md)
- [docs/gateway/sandbox-vs-tool-policy-vs-elevated.md](docs/gateway/sandbox-vs-tool-policy-vs-elevated.md)
- [docs/gateway/sandboxing.md](docs/gateway/sandboxing.md)
- [docs/platforms/mac/skills.md](docs/platforms/mac/skills.md)
- [docs/tools/elevated.md](docs/tools/elevated.md)
- [docs/tools/index.md](docs/tools/index.md)
- [docs/tools/skills-config.md](docs/tools/skills-config.md)
- [src/agents/memory-search.test.ts](src/agents/memory-search.test.ts)
- [src/agents/memory-search.ts](src/agents/memory-search.ts)
- [src/agents/sandbox-explain.test.ts](src/agents/sandbox-explain.test.ts)
- [src/agents/sandbox.ts](src/agents/sandbox.ts)
- [src/cli/memory-cli.test.ts](src/cli/memory-cli.test.ts)
- [src/cli/memory-cli.ts](src/cli/memory-cli.ts)
- [src/cli/models-cli.test.ts](src/cli/models-cli.test.ts)
- [src/commands/agent.test.ts](src/commands/agent.test.ts)
- [src/commands/agent.ts](src/commands/agent.ts)
- [src/config/schema.ts](src/config/schema.ts)
- [src/config/types.tools.ts](src/config/types.tools.ts)
- [src/config/types.ts](src/config/types.ts)
- [src/config/zod-schema.agent-runtime.ts](src/config/zod-schema.agent-runtime.ts)
- [src/config/zod-schema.ts](src/config/zod-schema.ts)
- [src/cron/isolated-agent.ts](src/cron/isolated-agent.ts)
- [src/cron/run-log.test.ts](src/cron/run-log.test.ts)
- [src/cron/run-log.ts](src/cron/run-log.ts)
- [src/cron/store.ts](src/cron/store.ts)
- [src/gateway/protocol/index.ts](src/gateway/protocol/index.ts)
- [src/gateway/protocol/schema.ts](src/gateway/protocol/schema.ts)
- [src/gateway/protocol/schema/agents-models-skills.ts](src/gateway/protocol/schema/agents-models-skills.ts)
- [src/gateway/protocol/schema/protocol-schemas.ts](src/gateway/protocol/schema/protocol-schemas.ts)
- [src/gateway/protocol/schema/types.ts](src/gateway/protocol/schema/types.ts)
- [src/gateway/server-methods-list.ts](src/gateway/server-methods-list.ts)
- [src/gateway/server-methods.ts](src/gateway/server-methods.ts)
- [src/gateway/server-methods/agents.ts](src/gateway/server-methods/agents.ts)
- [src/gateway/server.ts](src/gateway/server.ts)
- [src/memory/embeddings.test.ts](src/memory/embeddings.test.ts)
- [src/memory/embeddings.ts](src/memory/embeddings.ts)
- [src/memory/manager.ts](src/memory/manager.ts)

</details>

このページでは、OpenClaw のマルチエージェントシステムについて説明します: 複数の分離されたエージェントを定義する方法、バインディングを介してメッセージをルーティングする方法、エージェントごとの設定をオーバーライドする方法などです。全体的な設定構造については、[設定ファイル構造](#4.1)を参照してください。セッション管理と分離については、[セッション管理](#5.3)を参照してください。

---

## 概要

OpenClaw は単一の gateway インスタンスで**複数の分離されたエージェント**の実行をサポートしています。各エージェントには以下があります:

- ユニークな `id`
- 独自のワークスペースディレクトリ（ファイルシステム + メモリ）
- 分離されたセッショントランスクリプト
- 分離されたメモリインデックス
- オプションのエージェントごとのオーバーライド（ツール、モデル、サンドボックスなど）

メッセージは、チャネル、accountId、chatType、peer、その他のセッション属性にマッチする**バインディング**を介してエージェントにルーティングされます。バインディングがマッチしない場合、**デフォルトエージェント**がメッセージを処理します。

---

## エージェント定義

### 設定構造

エージェントは `agents.list[]` で定義されます。`agents.defaults` のすべてのフィールドはエージェントごとにオーバーライドできます。

### 最小限の例

```json5
{
  agents: {
    defaults: {
      workspace: "~/.openclaw/workspace"
    },
    list: [
      { id: "main", default: true },
      { id: "work", workspace: "~/.openclaw/workspace-work" }
    ]
  }
}
```

### オーバーライド付きの完全な例

```json5
{
  agents: {
    defaults: {
      workspace: "~/.openclaw/workspace",
      model: { primary: "anthropic/claude-sonnet-4-5" },
      tools: { profile: "coding" }
    },
    list: [
      {
        id: "main",
        default: true
      },
      {
        id: "support",
        workspace: "~/.openclaw/workspace-support",
        model: { primary: "openai/gpt-5.2" },
        tools: { profile: "messaging", allow: ["slack", "discord"] },
        sandbox: { mode: "off" }
      },
      {
        id: "research",
        workspace: "~/.openclaw/workspace-research",
        tools: { deny: ["exec", "process", "bash"] },
        memorySearch: { extraPaths: ["~/research-notes"] }
      }
    ]
  }
}
```

**ソース:** [docs/gateway/configuration.md:282-300]()、[src/config/zod-schema.ts:276]()

### 必須フィールド

| フィールド | タイプ | 説明 |
|-----------|-------|--------|
| `id` | string | ユニークなエージェント識別子（英数字 + ハイフン） |

その他のフィールドはすべてオプションで、`agents.defaults` から継承されます。

### 一般的なオーバーライド

| フィールド | 目的 |
|-----------|------|
| `default` | デフォルトエージェントとしてマーク（正確に 1 つのエージェントに `default: true` が必要） |
| `workspace` | 分離されたワークスペースディレクトリ |
| `model` | エージェントごとのプライマリモデルとフォールバック |
| `tools` | ツール許可リスト/拒否リストとプロファイル |
| `sandbox` | サンドボックスモード、スコープ、ワークスペースアクセス |
| `memorySearch` | メモリプロバイダー、extraPaths、インデックス設定 |
| `heartbeat` | ハートビートスケジュールとターゲット |
| `thinkingDefault` | デフォルトの思考レベル |
| `verboseDefault` | デフォルトの詳細レベル |

**ソース:** [src/commands/agent.ts:96-99]()、[docs/gateway/configuration.md:282-300]()

---

## ルーティングとバインディング

### バインディング配列

`bindings[]` 配列はマッチルールに基づいてメッセージをエージェントにルーティングします。バインディングは**順序通り**に評価され、最初のマッチが勝ちます。

### マッチルール

各バインディングには以下があります:
- `agentId`（必須）: ターゲットエージェント ID
- `match`（オプション）: マッチ条件オブジェクト

マッチフィールド（すべてオプション、指定された場合はすべてがマッチする必要があります）:

| フィールド | タイプ | 説明 | 例 |
|-----------|-------|--------|-----|
| `channel` | string | チャネルプロバイダー | `"whatsapp"`、`"telegram"` |
| `accountId` | string | チャネルアカウント ID | `"personal"`、`"biz"` |
| `chatType` | string | チャットタイプ | `"direct"`、`"group"` |
| `peer` | string | ピア識別子 | `"whatsapp:+15555550123"` |
| `keyPrefix` | string | セッションキープレフィックス | `"agent:main:whatsapp"` |

空の `match` オブジェクトまたは省略された `match` は「すべてマッチ」を意味します。

### バインディングの例

**チャネルアカウントでルーティング:**

```json5
{
  bindings: [
    { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
    { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } }
  ]
}
```

**チャットタイプでルーティング:**

```json5
{
  bindings: [
    { agentId: "support", match: { chatType: "group" } },
    { agentId: "main", match: { chatType: "direct" } }
  ]
}
```

**ピアでルーティング:**

```json5
{
  bindings: [
    { agentId: "personal", match: { peer: "telegram:123456" } },
    { agentId: "main" }  // catch-all
  ]
}
```

**ソース:** [docs/gateway/configuration.md:282-300]()、[src/routing/session-key.ts]()

---

## セッション分離

### セッションキー構造

セッションキーはエージェント ID を埋め込み、エージェント間の完全な分離を保証します。

**キーのプロパティ:**
- 異なるエージェントのセッションは、同じピアでも**決して競合しない**
- セッショントランスクリプトはエージェントごとに保存: `~/.openclaw/agents/<agentId>/sessions/*.jsonl`
- メモリインデックスはエージェントごと: `~/.openclaw/memory/<agentId>.sqlite`

**ソース:** [src/config/sessions.ts:42-44]()、[src/routing/session-key.ts]()、[src/memory/manager.ts:110-112]()

### エージェントごとの分離

**エージェントごとの分離リソース:**
- ワークスペースファイル（`IDENTITY.md`、`SKILLS.md`、`MEMORY.md`、`memory/*.md`）
- セッショントランスクリプト（`sessions/*.jsonl`）
- メモリインデックスデータベース
- サンドボックスコンテナ（`sandbox.scope: "agent"` の場合）
- 認証プロファイルオーバーライド（セッションエントリに保存）

**ソース:** [src/commands/agent.ts:96-99]()、[src/memory/manager.ts:169-179]()、[src/config/sessions/paths.ts]()

---

## エージェント解決

### デフォルトエージェント

正確に 1 つのエージェントに `default: true` が必要です。エージェントが明示的にデフォルトとしてマークされていない場合、システムは `agents.list[]` の最初のエージェントを使用します。

**解決ロジック:**
1. `default: true` を持つ最初のエージェント
2. なければ、リストの最初のエージェント
3. リストが空の場合、ID `"main"` の合成エージェント

**ソース:** [src/agents/agent-scope.ts]()、[src/cli/memory-cli.ts:60-78]()

### CLI エージェントオーバーライド

`--agent` フラグはエージェント選択を強制します:

```bash
openclaw agent --message "Hello" --to "+15555550123" --agent work
openclaw memory status --agent research
openclaw sandbox explain --agent support
```

指定されたエージェント ID が `agents.list[]` に存在しない場合、コマンドは有効なエージェント ID をリストしたエラーで失敗します。

**ソース:** [src/commands/agent.ts:78-86]()、[src/cli/memory-cli.ts:60-78]()

---

## エージェントごとのオーバーライド

### オーバーライド可能フィールド

すべての `agents.defaults` フィールドはエージェントごとにオーバーライドできます。一般的なオーバーライド:

| カテゴリ | フィールド |
|--------|----------|
| **ワークスペース** | `workspace`、`skipBootstrap` |
| **モデル** | `model.primary`、`model.fallbacks`、`models`（カタログ） |
| **ツール** | `tools.profile`、`tools.allow`、`tools.deny`、`tools.byProvider`、`tools.elevated`、`tools.exec` |
| **サンドボックス** | `sandbox.mode`、`sandbox.scope`、`sandbox.workspaceAccess`、`sandbox.docker`、`sandbox.browser` |
| **メモリ** | `memorySearch.enabled`、`memorySearch.provider`、`memorySearch.extraPaths` |
| **動作** | `thinkingDefault`、`verboseDefault`、`contextTokens`、`compaction`、`heartbeat` |

**継承ルール:**
- エージェント固有のフィールドはデフォルトを**完全にオーバーライド**（ディープマージなし）
- `model`、`tools`、`sandbox` などのオブジェクトはトップレベルでデフォルトを置換
- `tools.allow` などの配列はデフォルトを置換（追加ではない）

**ソース:** [src/config/zod-schema.agents.js]()、[docs/gateway/configuration.md:282-300]()

### ツールオーバーライドの例

```json5
{
  agents: {
    defaults: {
      tools: {
        profile: "coding",
        deny: ["web_search"]
      }
    },
    list: [
      {
        id: "support",
        tools: {
          profile: "messaging",  // replaces "coding"
          allow: ["slack", "discord"],
          // deny is not inherited (override is complete replacement)
        }
      }
    ]
  }
}
```

**ソース:** [docs/tools/index.md:66-80]()、[src/config/types.tools.ts:199-222]()

### サンドボックスオーバーライドの例

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",
        scope: "session",
        workspaceAccess: "none"
      }
    },
    list: [
      {
        id: "trusted",
        sandbox: {
          mode: "off"  // fully replace defaults
        }
      },
      {
        id: "restricted",
        sandbox: {
          mode: "all",
          scope: "agent",
          workspaceAccess: "ro"
        }
      }
    ]
  }
}
```

**ソース:** [docs/gateway/sandboxing.md:12-43]()、[src/config/zod-schema.agent-runtime.ts:87-143]()

### メモリオーバーライドの例

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "openai",
        extraPaths: []
      }
    },
    list: [
      {
        id: "research",
        memorySearch: {
          provider: "gemini",
          extraPaths: ["~/research-papers", "~/datasets/notes"]
        }
      },
      {
        id: "support",
        memorySearch: {
          enabled: false  // disable memory for this agent
        }
      }
    ]
  }
}
```

**ソース:** [src/agents/memory-search.ts:8-71]()、[docs/concepts/memory.md:242-261]()

---

## エージェント管理

### Gateway RPC メソッド

エージェントは Gateway WebSocket RPC で管理できます:

| メソッド | 目的 |
|--------|------|
| `agents.list` | 設定されたすべてのエージェントをリスト |
| `agents.create` | 新しいエージェントを作成（設定を書き込み） |
| `agents.update` | エージェントフィールドを更新（設定を書き込み） |
| `agents.delete` | エージェントを削除（設定を書き込み） |
| `agents.files.list` | エージェントワークスペース内のファイルをリスト |
| `agents.files.get` | エージェントワークスペースファイルを読み取り |
| `agents.files.set` | エージェントワークスペースファイルを書き込み |

**ソース:** [CHANGELOG.md:92]()、[src/gateway/protocol/schema.ts:1-16]()

### CLI コマンド

```bash
# List agents
openclaw agents list

# Add agent (interactive)
openclaw agents add

# Delete agent
openclaw agents delete work

# View agent workspace
ls ~/.openclaw/agents/main/
```

**ソース:** [src/commands/agent.ts:3-9]()、[docs/cli/agents.md]()

---

## 設定スキーマ

### `agents.defaults`

すべてのエージェントごとのフィールドはこのベースラインから継承されます。

```typescript
{
  workspace?: string;
  skipBootstrap?: boolean;
  model?: {
    primary?: string;
    fallbacks?: string[];
  };
  models?: Record<string, { alias?: string; ... }>;
  tools?: ToolPolicyConfig;
  sandbox?: SandboxConfig;
  memorySearch?: MemorySearchConfig;
  thinkingDefault?: "off" | "low" | "medium" | "high" | "xhigh";
  verboseDefault?: "off" | "on" | "full";
  contextTokens?: number;
  compaction?: CompactionConfig;
  heartbeat?: HeartbeatConfig;
  subagents?: SubagentsConfig;
  // ... all agent runtime settings
}
```

**ソース:** [src/config/types.agent-defaults.ts]()、[src/config/zod-schema.agents.js]()

### `agents.list[]`

エージェントエントリの配列。各エントリは `agents.defaults` を拡張します。

```typescript
{
  id: string;                    // required
  default?: boolean;             // exactly one must be true
  workspace?: string;
  model?: ModelConfig;
  tools?: AgentToolsConfig;
  sandbox?: SandboxConfig;
  memorySearch?: MemorySearchConfig;
  // ... all overrides
}
```

**ソース:** [src/config/types.agents.ts]()、[src/config/zod-schema.agents.js]()

### `bindings[]`

ルーティングルールの配列。順序通りに評価され、最初のマッチが勝ちます。

```typescript
{
  agentId: string;               // required
  match?: {
    channel?: string;
    accountId?: string;
    chatType?: "direct" | "group" | "channel";
    peer?: string;
    keyPrefix?: string;
  };
}
```

**ソース:** [src/config/zod-schema.agents.js]()、[docs/gateway/configuration.md:282-300]()

---

## ユースケース

### 個人用と仕事用エージェントの分離

```json5
{
  agents: {
    defaults: {
      model: { primary: "anthropic/claude-sonnet-4-5" }
    },
    list: [
      {
        id: "personal",
        default: true,
        workspace: "~/.openclaw/personal"
      },
      {
        id: "work",
        workspace: "~/work/openclaw-workspace"
      }
    ]
  },
  bindings: [
    { agentId: "work", match: { channel: "slack" } },
    { agentId: "personal" }  // catch-all
  ]
}
```

### サンドボックス化されたサポートエージェント

```json5
{
  agents: {
    defaults: {
      sandbox: { mode: "off" }
    },
    list: [
      { id: "main", default: true },
      {
        id: "support",
        workspace: "~/.openclaw/support",
        tools: { profile: "messaging" },
        sandbox: {
          mode: "all",
          scope: "agent",
          workspaceAccess: "none"
        }
      }
    ]
  },
  bindings: [
    { agentId: "support", match: { chatType: "group" } },
    { agentId: "main", match: { chatType: "direct" } }
  ]
}
```

### カスタムメモリパスを持つリサーチエージェント

```json5
{
  agents: {
    list: [
      { id: "main", default: true },
      {
        id: "research",
        workspace: "~/research/openclaw",
        tools: {
          deny: ["exec", "process", "bash"]
        },
        memorySearch: {
          provider: "gemini",
          extraPaths: [
            "~/research/papers",
            "~/research/notes",
            "~/datasets/summaries"
          ]
        }
      }
    ]
  },
  bindings: [
    { agentId: "research", match: { peer: "telegram:research_bot" } },
    { agentId: "main" }
  ]
}
```

**ソース:** [docs/gateway/configuration.md:282-300]()、[docs/concepts/memory.md:242-261]()

---

## 関連ページ

- [設定ファイル構造](#4.1) — 完全な設定スキーマリファレンス
- [設定管理](#4.2) — ホットリロード、CLI ツール、doctor
- [セッション管理](#5.3) — セッションキー、スコープ、トランスクリプト
- [ツールポリシー解決](#6.3) — ツール許可リストのカスケード方法
- [メモリ](#7) — エージェントごとのメモリインデックスと検索
- [サンドボックス](#6.2) — サンドボックスモード、スコープ、ワークスペースアクセス

---

**ソース:** [src/config/zod-schema.ts:276-280]()、[src/config/types.agents.ts]()、[src/commands/agent.ts:3-99]()、[src/routing/session-key.ts]()、[docs/gateway/configuration.md:282-300]()、[src/agents/agent-scope.ts]()、[src/memory/manager.ts:169-179]()

---

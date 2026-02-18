---
title: "オートメーションコマンド"
original_title: "Automation Commands"
source: "deepwiki:openclaw/openclaw"
chapter: 12
section: 8
---

# オートメーションコマンド

<details>
<summary>関連ソースファイル</summary>

この wiki ページの生成に使用されたファイル:

- [README.md](README.md)
- [assets/avatar-placeholder.svg](assets/avatar-placeholder.svg)
- [docs/channels/zalo.md](docs/channels/zalo.md)
- [docs/channels/zalouser.md](docs/channels/zalouser.md)
- [scripts/clawtributors-map.json](scripts/clawtributors-map.json)
- [scripts/update-clawtributors.ts](scripts/update-clawtributors.ts)
- [scripts/update-clawtributors.types.ts](scripts/update-clawtributors.types.ts)
- [src/commands/agent.test.ts](src/commands/agent.test.ts)
- [src/commands/agent.ts](src/commands/agent.ts)
- [src/config/config.ts](src/config/config.ts)
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
- [src/index.test.ts](src/index.test.ts)
- [src/index.ts](src/index.ts)
- [tsconfig.json](tsconfig.json)
- [ui/src/styles.css](ui/src/styles.css)
- [ui/src/styles/layout.mobile.css](ui/src/styles/layout.mobile.css)

</details>



このページでは、OpenClaw でのスケジュールタスク、Webhook、システムイベントオートメーションを管理するための CLI コマンドについて説明します。一般的な CLI 使用法と他のコマンドカテゴリについては、[CLI Reference](#12) を参照してください。

**範囲:**
- Cron ジョブ管理（スケジュール、実行、監視）
- Webhook 設定と配信
- システムイベントフック（Gmail Pub/Sub、カスタムトリガー）

**関連ページ:**
- ランタイムエージェント実行については、[Agent Commands](#12.2) を参照
- ゲートウェイサービス管理については、[Gateway Commands](#12.1) を参照

---

## 概要

OpenClaw は 3 つのオートメーションメカニズムを提供します:

| Mechanism | Purpose | Storage | Trigger |
|-----------|---------|---------|---------|
| **Cron ジョブ** | スケジュールされたエージェントタスク | `~/.openclaw/cron/jobs.json` | 時間ベース（cron 式） |
| **Webhook** | 外部 HTTP トリガー | ゲートウェイ設定 | ゲートウェイエンドポイントへの HTTP POST |
| **システムイベント** | プラットフォーム統合 | チャネル固有の設定 | Gmail Pub/Sub、チャネルイベント |

すべてのオートメーションは、インタラクティブコマンドと同じエージェント実行パイプラインを使用し、分離されたセッションとフルツールアクセス（サンドボックスポリシーに従う）を持ちます。

**Sources:** [src/gateway/server-methods-list.ts:73-79](), [README.md:163]()

---

## Cron ジョブ管理

### Cron ジョブを追加

```bash
openclaw cron add \
  --id "daily-report" \
  --schedule "0 9 * * *" \
  --prompt "Generate daily status report" \
  --agent main \
  --enabled
```

**オプション:**
- `--id <string>`: 一意のジョブ識別子（必須）
- `--schedule <cron>`: Cron 式（5 フィールド形式: `min hour day month weekday`）
- `--prompt <string>`: 実行するエージェントプロンプト（必須）
- `--agent <agentId>`: ターゲットエージェント（デフォルト: `main`）
- `--enabled`: 即座に有効化（デフォルト: true）
- `--timeout <seconds>`: 実行タイムアウト（デフォルト: 設定から）
- `--session-key <key>`: 明示的なセッションキー（高度）

### Cron ジョブを一覧表示

```bash
openclaw cron list
```

### Cron ジョブを実行

```bash
openclaw cron run --id "daily-report"
```

**オプション:**
- `--id <string>`: ジョブ識別子（必須）
- `--wait`: 実行完了までブロック

### 実行履歴を表示

```bash
openclaw cron runs --id "daily-report" --limit 10
```

### Cron ステータスをチェック

```bash
openclaw cron status
```

### Cron ジョブを更新

```bash
openclaw cron update \
  --id "daily-report" \
  --schedule "0 10 * * *" \
  --enabled false
```

### Cron ジョブを削除

```bash
openclaw cron remove --id "daily-report"
```

**Sources:** [src/gateway/protocol/schema/cron.ts:1-80](), [src/cron/store.ts:22-48]()

---

## Webhook

Webhook は、外部システムが HTTP POST リクエスト経由でエージェントアクションをトリガーすることを可能にします。

**設定:**

```json5
{
  gateway: {
    webhooks: {
      enabled: true,
      secret: "your-webhook-secret",
      allowedOrigins: ["https://trusted.example.com"]
    }
  }
}
```

**エンドポイント:** `POST https://gateway:18789/webhook/<hookId>`

**認証:** `Authorization: Bearer <secret>` ヘッダーが必要。

**ペイロード:**

```json
{
  "prompt": "Process incoming data",
  "data": { "key": "value" },
  "agentId": "main",
  "sessionKey": "agent:main:webhook:hookId"
}
```

**Sources:** [README.md:422]()

---

## システムイベントフック

### Gmail Pub/Sub 統合

OpenClaw は Google Cloud Pub/Sub 経由で Gmail イベントに反応できます。

**設定:**
1. Google Cloud Pub/Sub トピックとサブスクリプションを設定
2. 設定または環境変数に認証情報を設定
3. 統合を有効化:

```json5
{
  integrations: {
    gmail: {
      enabled: true,
      pubsubProject: "your-project-id",
      pubsubSubscription: "gmail-notifications",
      onMessage: {
        prompt: "Process new email",
        agentId: "main"
      }
    }
  }
}
```

**Sources:** [README.md:423](), [README.md:476]()

---

## 設定リファレンス

```json5
{
  cron: {
    enabled: true,
    store: "~/.openclaw/cron/jobs.json",
    maxRunLogBytes: 2000000,
    maxRunLogLines: 2000,
    evaluationIntervalMs: 60000
  },
  gateway: {
    webhooks: {
      enabled: false,
      secret: "webhook-secret",
      allowedOrigins: [],
      maxPayloadSize: 1048576
    }
  },
  integrations: {
    gmail: {
      enabled: false,
      pubsubProject: "",
      pubsubSubscription: "",
      onMessage: {
        prompt: "",
        agentId: "main",
        sessionKey: ""
      }
    }
  }
}
```

**Sources:** [src/cron/store.ts:8-20](), [src/cron/run-log.ts:54-59]()

---

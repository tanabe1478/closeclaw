---
title: "診断コマンド"
original_title: "Diagnostic Commands"
source: "deepwiki:openclaw/openclaw"
chapter: 12
section: 6
---

# 診断コマンド

<details>
<summary>関連ソースファイル</summary>

この wiki ページの生成に使用されたファイル:

- [README.md](README.md)
- [assets/avatar-placeholder.svg](assets/avatar-placeholder.svg)
- [docs/channels/zalo.md](docs/channels/zalo.md)
- [docs/channels/zalouser.md](docs/channels/zalouser.md)
- [docs/cli/index.md](docs/cli/index.md)
- [docs/docs.json](docs/docs.json)
- [docs/gateway/index.md](docs/gateway/index.md)
- [docs/gateway/troubleshooting.md](docs/gateway/troubleshooting.md)
- [docs/index.md](docs/index.md)
- [docs/start/getting-started.md](docs/start/getting-started.md)
- [docs/start/hubs.md](docs/start/hubs.md)
- [docs/start/onboarding.md](docs/start/onboarding.md)
- [docs/start/wizard.md](docs/start/wizard.md)
- [scripts/clawtributors-map.json](scripts/clawtributors-map.json)
- [scripts/update-clawtributors.ts](scripts/update-clawtributors.ts)
- [scripts/update-clawtributors.types.ts](scripts/update-clawtributors.config.types.ts)
- [src/config/config.ts](src/config/config.ts)
- [src/index.test.ts](src/index.test.ts)
- [src/index.ts](src/index.ts)
- [tsconfig.json](tsconfig.json)
- [ui/src/styles.css](ui/src/styles.css)
- [ui/src/styles/layout.mobile.css](ui/src/styles/layout.mobile.css)

</details>



このページでは、システムのヘルスチェック、診断、トラブルシューティングを行うための CLI コマンドについて説明します。

---

## `openclaw status`

システム全体のヘルスと診断を表示します。

```bash
openclaw status
openclaw status --all
openclaw status --deep
openclaw status --json
```

**オプション:**

| Flag | Description |
|------|-------------|
| `--json` | 構造化 JSON 出力 |
| `--all` | 完全診断（サービススキャンを含む） |
| `--deep` | ライブチェックでチャネルをプローブ |
| `--usage` | プロバイダー使用量/クォータを含める |
| `--verbose` | 拡張診断出力 |

**出力セクション:**
- Gateway ステータス
- チャネルステータス
- モデルステータス
- ノードステータス
- サービスステータス

**Sources:** [docs/cli/index.md:562-593](), [docs/gateway/health.md]()

---

## `openclaw health`

ゲートウェイヘルスエンドポイントをクエリします。

```bash
openclaw health
openclaw health --json
```

**動作:**
- ゲートウェイ RPC の `health` メソッドを呼び出し
- ゲートウェイが応答する場合は終了コード 0
- ゲートウェイが応答しない場合は終了コード 1

**Sources:** [docs/cli/index.md:654-699]()

---

## `openclaw doctor`

設定/状態の問題を診断して修復します。

```bash
openclaw doctor
openclaw doctor --deep
openclaw doctor --yes --non-interactive
```

**オプション:**

| Flag | Description |
|------|-------------|
| `--deep` | 詳細スキャン（サービス監査を含む） |
| `--yes` | 確認プロンプトをスキップ |
| `--non-interactive` | 非対話モード |
| `--no-workspace-suggestions` | ワークスペース提案をスキップ |

**Doctor がチェックする項目:**
- 設定ファイルの有効性とスキーマ準拠
- レガシー設定の移行
- ゲートウェイサービス設定
- ポート競合
- 認証設定
- チャネル認証情報の整合性
- セッションストアの整合性

**Sources:** [docs/cli/index.md:360-370](), [docs/gateway/doctor.md]()

---

## `openclaw logs`

ゲートウェイログをストリームまたはテールします。

```bash
openclaw logs
openclaw logs --follow
openclaw logs --limit 100
openclaw logs --grep "error"
```

**オプション:**

| Flag | Description |
|------|-------------|
| `--follow` | ログをリアルタイムでストリーム |
| `--limit <n>` | 取得する行数 |
| `--grep <pattern>` | パターンでフィルタリング |

**Sources:** [docs/cli/index.md:654-699]()

---

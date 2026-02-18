# OpenClaw ドキュメント日本語翻訳

> 原文: [OpenClaw DeepWiki](https://deepwiki.com/openclaw/openclaw)

## 目次

### 01 Overview
- [概要](ja/01-overview/00-overview.md)
- [主要コンセプト](ja/01-overview/01-key-concepts.md)
- [クイックスタート](ja/01-overview/02-quick-start.md)
- [アーキテクチャ図](ja/01-overview/03-architecture-diagrams.md)

### 02 Installation
- [概要](ja/02-installation/00-overview.md)
- [システム要件](ja/02-installation/01-system-requirements.md)
- [インストール方法](ja/02-installation/02-installation-methods.md)
- [オンボーディングウィザード](ja/02-installation/03-onboarding-wizard.md)
- [macOS アプリインストール](ja/02-installation/04-macos-app-installation.md)

### 03 Gateway
- [概要](ja/03-gateway/00-overview.md)
- [ゲートウェイ設定](ja/03-gateway/01-gateway-configuration.md)
- [ゲートウェイプロトコル](ja/03-gateway/02-gateway-protocol.md)
- [ゲートウェイサービス管理](ja/03-gateway/03-gateway-service-management.md)
- [リモートアクセス](ja/03-gateway/04-remote-access.md)

### 04 Configuration System
- [概要](ja/04-configuration-system/00-overview.md)
- [設定ファイル構造](ja/04-configuration-system/01-configuration-file-structure.md)
- [設定管理](ja/04-configuration-system/02-configuration-management.md)
- [マルチエージェント設定](ja/04-configuration-system/03-multi-agent-configuration.md)

### 05 Agent System
- [概要](ja/05-agent-system/00-overview.md)
- [エージェント実行フロー](ja/05-agent-system/01-agent-execution-flow.md)
- [システムプロンプト](ja/05-agent-system/02-system-prompt.md)
- [セッション管理](ja/05-agent-system/03-session-management.md)
- [モデル選択とフェイルオーバー](ja/05-agent-system/04-model-selection-and-failover.md)
- [コンテキストオーバーフローと自動圧縮](ja/05-agent-system/05-context-overflow-and-auto-compaction.md)
- [CLI バックエンド実行](ja/05-agent-system/06-cli-backend-execution.md)
- [認証プロファイル](ja/05-agent-system/07-authentication-profiles.md)

### 06 Tools and Skills
- [概要](ja/06-tools-and-skills/00-overview.md)
- [組み込みツール](ja/06-tools-and-skills/01-built-in-tools.md)
- [ツールセキュリティとサンドボックス](ja/06-tools-and-skills/02-tool-security-and-sandboxing.md)
- [ツールポリシー解決](ja/06-tools-and-skills/03-tool-policy-resolution.md)
- [スキルシステム](ja/06-tools-and-skills/04-skills-system.md)
- [バックグラウンドプロセス実行](ja/06-tools-and-skills/05-background-process-execution.md)

### 07 Memory System
- [概要](ja/07-memory-system/00-overview.md)
- [メモリ設定](ja/07-memory-system/01-memory-configuration.md)
- [メモリインデキシング](ja/07-memory-system/02-memory-indexing.md)
- [メモリ検索](ja/07-memory-system/03-memory-search.md)
- [エンベディングプロバイダー選択](ja/07-memory-system/04-embedding-provider-selection.md)

### 08 Channels
- [概要](ja/08-channels/00-overview.md)
- [チャネルルーティングとアクセス制御](ja/08-channels/01-channel-routing-and-access-control.md)
- [ペアリングシステム](ja/08-channels/02-pairing-system.md)
- [チャネルメッセージフロー](ja/08-channels/03-channel-message-flow.md)
- [WhatsApp 連携](ja/08-channels/04-whatsapp-integration.md)
- [Telegram 連携](ja/08-channels/05-telegram-integration.md)
- [Discord 連携](ja/08-channels/06-discord-integration.md)
- [Signal 連携](ja/08-channels/07-signal-integration.md)
- [その他のチャネル](ja/08-channels/08-other-channels.md)

### 09 Commands and Directives
- [概要](ja/09-commands-and-directives/00-overview.md)
- [コマンドリファレンス](ja/09-commands-and-directives/01-command-reference.md)
- [コマンド認可](ja/09-commands-and-directives/02-command-authorization.md)
- [プラットフォーム固有コマンド](ja/09-commands-and-directives/03-platform-specific-commands.md)
- [ディレクティブ](ja/09-commands-and-directives/04-directives.md)
- [ステータスとコンテキストレポート](ja/09-commands-and-directives/05-status-and-context-reporting.md)
- [サブエージェント管理](ja/09-commands-and-directives/06-subagent-management.md)

### 10 Extensions and Plugins
- [概要](ja/10-extensions-and-plugins/00-overview.md)
- [プラグインシステム概要](ja/10-extensions-and-plugins/01-plugin-system-overview.md)
- [組み込みエクステンション](ja/10-extensions-and-plugins/02-built-in-extensions.md)
- [カスタムプラグイン作成](ja/10-extensions-and-plugins/03-creating-custom-plugins.md)

### 11 Device Nodes
- [概要](ja/11-device-nodes/00-overview.md)
- [ノードペアリングとディスカバリー](ja/11-device-nodes/01-node-pairing-and-discovery.md)
- [ノード機能](ja/11-device-nodes/02-node-capabilities.md)

### 12 CLI Reference
- [概要](ja/12-cli-reference/00-overview.md)
- [ゲートウェイコマンド](ja/12-cli-reference/01-gateway-commands.md)
- [エージェントコマンド](ja/12-cli-reference/02-agent-commands.md)
- [チャネルコマンド](ja/12-cli-reference/03-channel-commands.md)
- [モデルコマンド](ja/12-cli-reference/04-model-commands.md)
- [設定コマンド](ja/12-cli-reference/05-configuration-commands.md)
- [診断コマンド](ja/12-cli-reference/06-diagnostic-commands.md)
- [メモリコマンド](ja/12-cli-reference/07-memory-commands.md)
- [自動化コマンド](ja/12-cli-reference/08-automation-commands.md)

### 13 Deployment
- [概要](ja/13-deployment/00-overview.md)
- [ローカルデプロイ](ja/13-deployment/01-local-deployment.md)
- [VPS デプロイ](ja/13-deployment/02-vps-deployment.md)
- [Docker デプロイ](ja/13-deployment/03-docker-deployment.md)
- [ネットワーク設定](ja/13-deployment/04-network-configuration.md)

### 14 Operations and Troubleshooting
- [概要](ja/14-operations-and-troubleshooting/00-overview.md)
- [ヘルスモニタリング](ja/14-operations-and-troubleshooting/01-health-monitoring.md)
- [doctor コマンドガイド](ja/14-operations-and-troubleshooting/02-doctor-command-guide.md)
- [よくある問題](ja/14-operations-and-troubleshooting/03-common-issues.md)
- [マイグレーションとバックアップ](ja/14-operations-and-troubleshooting/04-migration-and-backup.md)

### 15 Development
- [概要](ja/15-development/00-overview.md)
- [アーキテクチャ詳解](ja/15-development/01-architecture-deep-dive.md)
- [プロトコル仕様](ja/15-development/02-protocol-specification.md)
- [ソースからのビルド](ja/15-development/03-building-from-source.md)
- [リリースプロセス](ja/15-development/04-release-process.md)
- [CI/CD パイプライン](ja/15-development/05-ci-cd-pipeline.md)
- [コントリビューションガイドライン](ja/15-development/06-contributing-guidelines.md)

---

## 進捗

全体: **4 / 86 ページ翻訳済み** (4.7%)

| 章 | ページ数 | 翻訳済み |
|----|---------|---------|
| 01 Overview | 4 | 4 |
| 02 Installation | 5 | 0 |
| 03 Gateway | 5 | 0 |
| 04 Configuration System | 4 | 0 |
| 05 Agent System | 8 | 0 |
| 06 Tools and Skills | 6 | 0 |
| 07 Memory System | 5 | 0 |
| 08 Channels | 9 | 0 |
| 09 Commands and Directives | 7 | 0 |
| 10 Extensions and Plugins | 4 | 0 |
| 11 Device Nodes | 3 | 0 |
| 12 CLI Reference | 9 | 0 |
| 13 Deployment | 5 | 0 |
| 14 Operations and Troubleshooting | 5 | 0 |
| 15 Development | 7 | 0 |

詳細な進捗は [PROGRESS.md](../PROGRESS.md) を参照。

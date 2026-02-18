# OpenClaw DeepWiki 日本語翻訳プロジェクト

## プロジェクト概要

OpenClaw (https://github.com/openclaw/openclaw) の DeepWiki ドキュメントを日本語に翻訳し、OpenClaw の仕組みを理解するためのリファレンスを作成する。

## ディレクトリ構成

```
docs/
├── en/          # 英語原文（DeepWiki から取得）
├── ja/          # 日本語翻訳
└── index.md     # 目次
```

## 翻訳方針

### 翻訳するもの
- 説明文、見出し、表の内容
- 図の説明テキスト

### 翻訳しないもの（英語のまま保持）
- コードブロック全体
- インラインコード（`バッククォート内`）
- CLI コマンド・引数
- ファイルパス・ディレクトリ名
- 環境変数名・設定キー名
- API エンドポイントパス
- URL
- JSON/YAML のキー名

### プロダクト名の扱い
- プロダクト名はカタカナ表記可（初出時に英語を併記）
- 例: ゲートウェイ (Gateway)

## 用語集

| English | Japanese | 備考 |
|---------|----------|------|
| Gateway | ゲートウェイ | |
| Agent | エージェント | |
| Session | セッション | |
| Channel | チャネル | |
| Node | ノード | |
| Tool | ツール | |
| Skill | スキル | |
| Plugin | プラグイン | |
| Configuration | 設定 | |
| Deployment | デプロイ | |
| Sandbox | サンドボックス | |
| WebSocket | WebSocket | 英語のまま |
| JSON-RPC | JSON-RPC | 英語のまま |
| CLI | CLI | 英語のまま |
| Middleware | ミドルウェア | |
| Callback | コールバック | |
| Endpoint | エンドポイント | |
| Payload | ペイロード | |
| Token | トークン | |
| Provider | プロバイダー | |
| Failover | フェイルオーバー | |
| Compaction | コンパクション | |
| Embedding | エンベディング | |
| Pairing | ペアリング | |
| Directive | ディレクティブ | |
| Subagent | サブエージェント | |
| Onboarding | オンボーディング | |
| Wizard | ウィザード | |

## 翻訳ファイルフォーマット

各翻訳ファイルには YAML フロントマターを付与：

```yaml
---
title: "ページタイトル（日本語）"
original_title: "Original English Title"
source: "deepwiki:openclaw/openclaw"
chapter: 1
section: 0
---
```

## 翻訳ワークフロー

1. `docs/en/{chapter}/{page}.md` を読む
2. 翻訳ガイドラインに従い日本語に翻訳
3. `docs/ja/{chapter}/{page}.md` に書き出し（YAML フロントマター付き）
4. `translation-checker` サブエージェントで検証
5. PASS → `PROGRESS.md` を更新
6. FAIL/NEEDS_REVISION → 修正して再検証

## 進捗管理

- `PROGRESS.md` で全ページの翻訳状態を管理
- `[ ]` 未着手 / `[T]` 翻訳済み / `[V]` 検証済み(PASS)

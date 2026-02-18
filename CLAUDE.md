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

## 翻訳・検証ワークフロー

### 1章分の作業サイクル

```
PROGRESS.md を確認 → 未着手の章を特定
  ↓
翻訳（並列実行可）
  ↓
検証（並列実行可）
  ↓
修正（NEEDS_REVISION/FAIL の場合）
  ↓
PROGRESS.md 更新 → コミット
```

### Step 1: 翻訳

章内の各ページに対して並列で Task エージェントを起動する。

```
Task(subagent_type="general-purpose", description="Translate {chapter}/{page}.md")
```

各エージェントへの指示:
- `docs/en/{chapter}/{page}.md` を読む
- 本ファイル（CLAUDE.md）の翻訳方針・用語集に従い日本語に翻訳
- `docs/ja/{chapter}/{page}.md` に YAML フロントマター付きで書き出す
- コードブロック・Mermaid 図のラベル・インラインコードは英語のまま保持
- `<details>` / `<summary>` ブロックはそのまま保持

### Step 2: 検証

翻訳完了後、各ページに対して並列で検証エージェントを起動する。

```
Task(subagent_type="general-purpose", description="Check translation {chapter}/{page}.md")
```

各エージェントへの指示:
- `docs/en/{chapter}/{page}.md` と `docs/ja/{chapter}/{page}.md` を比較
- 以下の 6 項目を検証（詳細は `.claude/agents/translation-checker.md`）:
  1. **構造の完全性**: 見出し数・リスト項目数・テーブル行数・コードブロック数の一致
  2. **内容の完全性**: 段落の欠落がないこと
  3. **コードブロックの保全**: コード内が翻訳されていないこと
  4. **用語の一貫性**: 上記用語集に従っていること
  5. **リンクの保全**: URL が変更されていないこと
  6. **書式の保全**: Bold/Italic/テーブル構造の維持
- 判定を PASS / NEEDS_REVISION / FAIL で報告
- 問題がある場合は具体的な箇所と修正内容を提示

### Step 3: 修正と再検証

- NEEDS_REVISION / FAIL のページは指摘箇所を修正
- 修正後、再度 Step 2 の検証を実行
- PASS になるまで繰り返す

### Step 4: 進捗更新とコミット

- `PROGRESS.md` の該当ページを `[V]`（検証済み）に更新
- `docs/index.md` の進捗セクションの数値を更新
- 1 章分をまとめてコミット:
  ```
  feat: Chapter {N} {章名} 翻訳完了（{X}ページ）
  ```

### 並列実行の目安

- 1章あたり最大 4〜5 ページを同時に翻訳可能
- 検証も同数を並列実行可能
- 大きなページ（40K 文字超）はセクション分割で翻訳

## 進捗管理

- `PROGRESS.md` で全 86 ページの翻訳状態を管理
- `[ ]` 未着手 / `[T]` 翻訳済み / `[V]` 検証済み(PASS)
- `docs/index.md` 末尾に章ごとの進捗サマリーあり

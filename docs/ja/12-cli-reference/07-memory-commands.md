---
title: "メモリコマンド"
original_title: "Memory Commands"
source: "deepwiki:openclaw/openclaw"
chapter: 12
section: 7
---

# メモリコマンド

<details>
<summary>関連ソースファイル</summary>

この wiki ページの生成に使用されたファイル:

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
- [src/config/schema.ts](src/config/schema.ts)
- [src/config/types.tools.ts](src/config/types.tools.ts)
- [src/config/types.ts](src/config/types.ts)
- [src/config/zod-schema.agent-runtime.ts](src/config/zod-schema.agent-runtime.ts)
- [src/config/zod-schema.ts](src/config/zod-schema.ts)
- [src/memory/embeddings.test.ts](src/memory/embeddings.test.ts)
- [src/memory/embeddings.ts](src/memory/embeddings.ts)
- [src/memory/manager.ts](src/memory/manager.ts)

</details>



## 目的と範囲

このページでは、セマンティックメモリインデックスの検査と管理を行うための `openclaw memory` CLI コマンドについて説明します。これらのコマンドを使用すると、インデックスのステータスを確認し、手動で再インデックスをトリガーし、コマンドラインから検索を実行できます。

メモリシステムのアーキテクチャと設定については、[Memory System](#7) を参照してください。メモリ検索の設定オプションについては、[Memory Configuration](#7.1) を参照してください。インデックスパイプラインの詳細については、[Memory Indexing](#7.2) を参照してください。

---

## 概要

メモリ CLI は 3 つの主要なコマンドを提供します:

| Command | Purpose |
|---------|---------|
| `openclaw memory status` | インデックスメタデータ、プロバイダー情報、準備状況チェックを表示 |
| `openclaw memory index` | メモリソースの完全再インデックスを手動でトリガー |
| `openclaw memory search <query>` | コマンドラインからメモリインデックスを検索 |

すべてのメモリコマンドは [src/cli/memory-cli.ts]() で実装され、`MemoryIndexManager` [src/memory/manager.ts:111-632]() に委譲されます。

**Sources:** [src/cli/memory-cli.ts:1-591](), [docs/cli/memory.md:1-46]()

---

## `openclaw memory status`

メモリインデックスのステータスを表示します。ファイル/チャンク数、プロバイダー設定、ベクトル/FTS 可用性、オプションの詳細プローブを含みます。

### 構文

```bash
openclaw memory status [options]
```

### オプション

| Option | Description |
|--------|-------------|
| `--agent <id>` | 特定のエージェントにスコープ（デフォルト: 設定されたすべてのエージェント） |
| `--deep` | ベクトル拡張機能とエンベディングプロバイダーの可用性をプローブ |
| `--index` | インデックスがダーティな場合に完全再インデックスを実行（`--deep` を暗黙的に含む） |
| `--force` | 再インデックス時に最新状態チェックをスキップ |
| `--json` | フォーマットされたテキストの代わりに生の JSON を出力 |
| `--verbose` | プローブとインデックス中の詳細ログを有効化 |

### 動作

`--deep` が指定されていない場合、`status` は軽量なチェックを実行します:
- SQLite からインデックスメタデータを読み取り [src/memory/manager.ts:469-564]()
- ソースごとのファイル/チャンク数を報告
- 設定されたプロバイダーとモデルを表示
- インデックスがダーティかどうかを示す

`--deep` が指定された場合、追加のプローブが実行されます:
- **ベクトルプローブ**: sqlite-vec 拡張機能のロードを試行し、ベクターテーブルの可用性を検証 [src/memory/manager.ts:791-846]()
- **エンベディングプローブ**: 設定されたプロバイダーへのサンプルエンベディングリクエストをテスト [src/memory/manager.ts:848-922]()

`--index` が `--deep` と共に指定された場合、インデックスがダーティとマークされている場合または `--force` が提供された場合に、完全再インデックスを自動的にトリガーします。

### マルチエージェントサポート

`--agent` が省略された場合、コマンドは `agents.list` 内のすべての設定されたエージェント（またはエージェントが設定されていない場合はデフォルトの `"main"`）を反復処理します。各エージェントのメモリインデックスが個別にチェックされます。

**Sources:** [src/cli/memory-cli.ts:243-361](), [src/memory/manager.ts:469-564]()

---

## `openclaw memory index`

メモリソースの完全再インデックスを手動でトリガーします。

### 構文

```bash
openclaw memory index [options]
```

### オプション

| Option | Description |
|--------|-------------|
| `--agent <id>` | 特定のエージェントにスコープ（デフォルト: 設定されたすべてのエージェント） |
| `--force` | インデックスが最新であっても強制再インデックス |
| `--verbose` | インデックス中の詳細ログを出力（ファイル発見、エンベディングバッチ、チャンク数） |
| `--json` | インデックス後に生の JSON ステータスを出力 |

### 動作

`index` コマンドは `manager.sync({ reason: "cli", force: <bool>, progress: <fn> })` [src/memory/manager.ts:390-403]() を呼び出します:

1. **ファイルを発見**: メモリソース（ワークスペース `MEMORY.md`、`memory/*.md`、追加パス、オプションでセッショントランスクリプト）からすべての Markdown ファイルをリスト [src/memory/internal.ts:191-248]()
2. **キャッシュをチェック**: ファイル更新時刻とハッシュをインデックスと比較 [src/memory/manager.ts:954-1036]()
3. **ファイルをチャンク**: 変更されたファイルを ~400 トークンのチャンクに 80 トークンのオーバーラップで分割 [src/memory/internal.ts:122-188]()
4. **エンベディングを生成**:
   - キャッシュが有効な場合、コンテンツハッシュで `embedding_cache` テーブルをチェック [src/memory/manager.ts:1210-1263]()
   - それ以外の場合、チャンクをバッチ化してエンベディングプロバイダーを呼び出し [src/memory/manager.ts:1144-1209]()
5. **エンベディングを保存**: `chunks` テーブルに書き込み（sqlite-vec が利用可能な場合は `chunks_vec` にも） [src/memory/manager.ts:1265-1330]()
6. **FTS インデックスを構築**: キーワード検索用に `chunks_fts` を設定 [src/memory/manager.ts:1332-1358]()

進行状況の更新は `withProgressTotals()` [src/cli/progress.ts:96-204]() を介してコンソールにストリームされます。

**Sources:** [src/cli/memory-cli.ts:593-674](), [src/memory/manager.ts:390-403](), [src/memory/manager.ts:924-1358]()

---

## `openclaw memory search`

メモリインデックスに対してセマンティック検索を実行します。

### 構文

```bash
openclaw memory search <query> [options]
```

### オプション

| Option | Description |
|--------|-------------|
| `--agent <id>` | 特定のエージェントにスコープ（デフォルト: main） |
| `--max-results <n>` | 返す最大結果数（デフォルト: 設定からまたは 6） |
| `--min-score <n>` | 最小関連性スコア（0.0-1.0、デフォルト: 設定からまたは 0.35） |
| `--json` | 生の JSON 結果配列を出力 |
| `--verbose` | 詳細ログを有効化 |

### 動作

`search` コマンド:
1. ターゲットエージェントを解決 [src/cli/memory-cli.ts:676-691]()
2. `getMemorySearchManager()` 経由でメモリマネージャーを取得 [src/memory/index.ts:38-48]()
3. `manager.search(query, { maxResults, minScore })` を呼び出し [src/memory/manager.ts:266-314]()
4. 結果をフォーマットして表示 [src/cli/memory-cli.ts:719-794]()

内部的に、`manager.search()` は以下を実行:
- **クエリエンベディング**: 設定されたプロバイダーを使用してクエリテキストをエンベッド [src/memory/manager.ts:1421-1453]()
- **ベクトル検索**: コサイン類似度でトップ候補を取得（sqlite-vec 経由またはメモリ内） [src/memory/manager-search.ts:15-110]()
- **キーワード検索**: （ハイブリッドモードが有効な場合）`chunks_fts` から BM25 ランクでトップ候補を取得 [src/memory/manager-search.ts:112-180]()
- **ハイブリッドマージ**: ベクトル + キーワード結果を結合して再ランク [src/memory/hybrid.ts:18-82]()
- **フィルタリング**: `minScore` しきい値を適用し、トップ `maxResults` を返す [src/memory/manager.ts:305-313]()

**Sources:** [src/cli/memory-cli.ts:676-794](), [src/memory/manager.ts:266-314](), [src/memory/manager-search.ts:15-180]()

---

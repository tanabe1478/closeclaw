---
title: "Signal 統合"
original_title: "Signal Integration"
source: "deepwiki:openclaw/openclaw"
chapter: 8
section: 7
---

# ページ: Signal 統合

# Signal 統合

<details>
<summary>関連ソースファイル</summary>

この Wiki ページの作成に使用されたファイル:

- [README.md](README.md)
- [assets/avatar-placeholder.svg](assets/avatar-placeholder.svg)
- [docs/channels/zalo.md](docs/channels/zalo.md)
- [docs/channels/zalouser.md](docs/channels/zalouser.md)
- [scripts/clawtributors-map.json](scripts/clawtributors-map.json)
- [scripts/update-clawtributors.ts](scripts/update-clawtributors.ts)
- [scripts/update-clawtributors.types.ts](scripts/update-clawtributors.types.ts)
- [src/config/config.ts](src/config/config.ts)
- [src/discord/monitor.ts](src/discord/monitor.ts)
- [src/imessage/monitor.ts](src/imessage/monitor.ts)
- [src/index.test.ts](src/index.test.ts)
- [src/index.ts](src/index.ts)
- [src/signal/monitor.ts](src/signal/monitor.ts)
- [src/slack/monitor.ts](src/slack/monitor.ts)
- [src/telegram/bot.test.ts](src/telegram/bot.test.ts)
- [src/telegram/bot.ts](src/telegram/bot.ts)
- [src/web/auto-reply.ts](src/web/auto-reply.ts)
- [src/web/inbound.media.test.ts](src/web/inbound.media.test.ts)
- [src/web/inbound.test.ts](src/web/inbound.test.ts)
- [src/web/test-helpers.ts](src/web/test-helpers.ts)
- [src/web/vcard.ts](src/web/vcard.ts)
- [tsconfig.json](tsconfig.json)
- [ui/src/styles.css](ui/src/styles.css)
- [ui/src/styles/layout.mobile.css](ui/src/styles/layout.mobile.css)

</details>



このドキュメントでは、OpenClaw の Signal メッセージ統合について説明します。これは `signal-cli` を基盤トランスポートとして使用します。統合は signal-cli デーモンを生成および管理し、その Server-Sent Events (SSE) ストリームで受信メッセージを消費し、JSON-RPC API を使用してメッセージを送信し添付ファイルを取得します。

一般的なチャネルの概念（ルーティング、アクセス制御、セッションキー）については [チャネルルーティングとアクセス制御](#8.1) を参照してください。その他のメッセージングプラットフォームについては [WhatsApp 統合](#8.2)、[Telegram 統合](#8.3)、[Discord 統合](#8.4)、[その他チャネル](#8.6) を参照してください。

---

## アーキテクチャ概要

**Signal 統合コンポーネントアーキテクチャ**

```mermaid
graph TB
    subgraph "ゲートウェイコントロールプレーン"
        GW["ゲートウェイサーバー"]
        EventHandler["createSignalEventHandler"]
        ReplyHandler["deliverReplies"]
    end

    subgraph "Signal プロバイダモニター"
        MonitorEntry["monitorSignalProvider"]
        AccountResolver["resolveSignalAccount"]
        Config["MonitorSignalOpts"]
    end

    subgraph "signal-cli デーモン"
        DaemonSpawn["spawnSignalDaemon"]
        DaemonProcess["signal-cli --http"]
        HttpApi["HTTP JSON-RPC API :8080"]
        SseEndpoint["SSE /v1/receive"]
    end

    subgraph "イベント処理"
        SseLoop["runSignalSseLoop"]
        EventDispatch["イベントディスパッチャー"]
        MessageExtract["メッセージ抽出"]
        ReactionCheck["リアクションフィルタリング"]
    end

    subgraph "RPC 操作"
        ClientCheck["signalCheck"]
        ClientRpc["signalRpcRequest"]
        SendMsg["sendMessageSignal"]
        FetchAttach["fetchAttachment"]
    end

    subgraph "アクセス制御"
        SenderCheck["isSignalSenderAllowed"]
        PolicyCheck["DM/グループポリシー"]
        AllowList["allowFrom / groupAllowFrom"]
    end

    GW --> MonitorEntry
    MonitorEntry --> AccountResolver
    MonitorEntry --> Config

    MonitorEntry --> DaemonSpawn
    DaemonSpawn --> DaemonProcess
    DaemonProcess --> HttpApi
    DaemonProcess --> SseEndpoint

    MonitorEntry --> SseLoop
    SseLoop --> SseEndpoint
    SseLoop --> EventDispatch

    EventDispatch --> EventHandler
    EventHandler --> MessageExtract
    EventHandler --> ReactionCheck
    EventHandler --> SenderCheck

    SenderCheck --> PolicyCheck
    PolicyCheck --> AllowList

    EventHandler --> FetchAttach
    FetchAttach --> ClientRpc
    ClientRpc --> HttpApi

    EventHandler --> ReplyHandler
    ReplyHandler --> SendMsg
    SendMsg --> ClientRpc

    MonitorEntry --> ClientCheck
    ClientCheck --> HttpApi

    style MonitorEntry fill:#f9f9f9
    style EventHandler fill:#f9f9f9
    style DaemonProcess fill:#e8f4f8
    style ClientRpc fill:#fff8dc
```

**ソース:** [src/signal/monitor.ts:1-377]()

Signal 統合はデーモンベースのアーキテクチャに従っており、`monitorSignalProvider` がライフサイクル全体を調整します。オプションで `autoStart` が有効な場合は signal-cli デーモンプロセスを生成し、HTTP API が準備できるのを待ってから SSE 接続を開いてイベントを受信します。受信イベントは `createSignalEventHandler` にディスパッチされ、これはアクセス制御を適用し、JSON-RPC 経由で添付ファイルを取得し、メッセージをエージェントシステムにルーティングし、レスポンスを signal-cli API 経てで送り返します。

---

## 設定

**Signal アカウント設定構造**

Signal 統合は `openclaw.json` の `channels.signal` セクションで設定され、`accounts` マップを介してアカウントごとのオーバーライドがサポートされます:

| 設定キー | タイプ | デフォルト | 説明 |
|-------------------|------|---------|-------------|
| `account` | `string` | 必須 | Signal アカウント電話番号 (E.164 形式) |
| `accountId` | `string` | 自動生成 | マルチアカウントセットアップ用の内部識別子 |
| `httpUrl` | `string` | `http://127.0.0.1:8080` | signal-cli HTTP API のベース URL |
| `autoStart` | `boolean` | `true` (if no `httpUrl`) | signal-cli デーモンを生成するかどうか |
| `cliPath` | `string` | `"signal-cli"` | signal-cli 実行ファイルへのパス |
| `httpHost` | `string` | `"127.0.0.1"` | デーモン HTTP バインドアドレス |
| `httpPort` | `number` | `8080` | デーモン HTTP ポート |
| `receiveMode` | `"on-start" \| "manual"` | `undefined` | signal-cli 受信モード |
| `startupTimeoutMs` | `number` | `30000` | デーモン起動の最大待機時間 (1s-120s でクランプ) |
| `dmPolicy` | `"open" \| "allowlist" \| "pairing"` | `"pairing"` | DM アクセスポリシー |
| `groupPolicy` | `"open" \| "allowlist"` | `"allowlist"` | グループアクセスポリシー |
| `allowFrom` | `string[]` | `[]` | DM 用の電話番号または UUID |
| `groupAllowFrom` | `string[]` | `allowFrom` | グループ内の電話番号/UUID |
| `reactionNotifications` | `"off" \| "own" \| "allowlist" \| "all"` | `"own"` | リアクション通知モード |
| `reactionAllowlist` | `string[]` | `[]` | 通知をトリガーするユーザー (mode=`"allowlist"` 時) |
| `ignoreAttachments` | `boolean` | `false` | 添付ファイルのダウンロードをスキップ |
| `ignoreStories` | `boolean` | `false` | ストーリーメッセージをスキップ |
| `sendReadReceipts` | `boolean` | `false` | デーモン経由で読み取り確認を送信 |
| `mediaMaxMb` | `number` | `8` | 添付ファイル最大サイズ (MB) |
| `blockStreaming` | `boolean` | `false` | ストリーミングレスポンスを無効化 |
| `historyLimit` | `number` | from `messages.groupChat.historyLimit` | グループ履歴エントリの最大数 |

**ソース:** [src/signal/monitor.ts:38-57](), [src/signal/monitor.ts:253-286]()

`resolveSignalAccount` 関数がアカウント固有のオーバーライドをチャネルレベルのデフォルトとマージし、グローバルフォールバックを適用します。アクセスリスト内の電話番号は `normalizeE164` によって自動的に E.164 形式に正規化されます。

---

## デーモン管理

**Signal デーモンライフサイクル**

```mermaid
sequenceDiagram
    participant Monitor as "monitorSignalProvider"
    participant Spawn as "spawnSignalDaemon"
    participant Process as "signal-cli Process"
    participant Check as "waitForSignalDaemonReady"
    participant API as "HTTP API :8080"

    Monitor->>Monitor: "autoStart 設定をチェック"

    alt autoStart = true
        Monitor->>Spawn: "デーモンを生成"
        Spawn->>Process: "exec signal-cli daemon"
        Note over Process: "--http {host}:{port}"<br/>"--account {number}"<br/>"--receive-mode {mode}"
        Process->>API: "HTTP サーバーを開始"

        Monitor->>Check: "waitForSignalDaemonReady"
        loop タイムアウト付きポーリング
            Check->>API: "GET /v1/health or similar"
            API-->>Check: "レスポンスステータス"
            alt 成功
                Check-->>Monitor: "デーモン準備完了"
            else タイムアウト
                Check-->>Monitor: "エラー: タイムアウト"
            end
        end
    else autoStart = false
        Note over Monitor: "外部デーモン<br/>httpUrl で実行中と仮定"
    end

    Monitor->>Monitor: "SSE ループを開始"

    Note over Monitor: "中止シグナルまたはエラー時"
    Monitor->>Spawn: "daemonHandle.stop()"
    Spawn->>Process: "SIGTERM/SIGKILL"
```

**ソース:** [src/signal/monitor.ts:288-311](), [src/signal/monitor.ts:318-328](), [src/signal/monitor.ts:145-170]()

`autoStart` が有効な場合（デフォルトで `httpUrl` が提供されない場合）、`monitorSignalProvider` は `spawnSignalDaemon` を呼び出して signal-cli デーモンプロセスをフォークします。デーモンは以下で呼び出されます:
- `--http {httpHost}:{httpPort}` - HTTP API バインディング
- `--account {account}` - Signal アカウント番号
- `--receive-mode {receiveMode}` - メッセージ取得モード (`on-start` または `manual`)
- `--ignore-attachments` - 設定されている場合
- `--ignore-stories` - 設定されている場合
- `--send-read-receipts` - 設定されている場合

`waitForSignalDaemonReady` 関数は `signalCheck` (ヘルスチェックリクエストを実行) を使用して HTTP API を指数バックオフでポーリングし、デーモンが正常に応答するか `startupTimeoutMs` が超過するまで待ちます。タイムアウトは 1 秒と 120 秒の間でクランプされます。

中止またはエラー時、デーモンハンドルの `stop()` メソッドが呼び出され、プロセスがクリーンに終了しない場合は SIGTERM に続いて SIGKILL が送信されます。

**ソース:** [src/signal/monitor.ts:312-328](), [src/signal/monitor.ts:372-375]()

---

## イベントストリーム

**SSE 接続とイベントディスパッチ**

```mermaid
graph LR
    subgraph "SSE ループ"
        Start["runSignalSseLoop"]
        Connect["/v1/receive に接続"]
        Parse["SSE イベントを解析"]
        Dispatch["onEvent コールバック"]
    end

    subgraph "イベントタイプ"
        Receive["ReceiveMessageEvent"]
        Reaction["ReactionEvent"]
        Typing["TypingEvent"]
        Receipt["ReceiptEvent"]
        Sync["SyncEvent"]
    end

    subgraph "エラーハンドリング"
        Reconnect["自動再接続"]
        Backoff["指数バックオフ"]
        AbortCheck["中止シグナルをチェック"]
    end

    Start --> Connect
    Connect --> Parse
    Parse --> Dispatch

    Dispatch --> Receive
    Dispatch --> Reaction
    Dispatch --> Typing
    Dispatch --> Receipt
    Dispatch --> Sync

    Parse --> Reconnect
    Reconnect --> Backoff
    Backoff --> AbortCheck
    AbortCheck --> Connect

    style Start fill:#f9f9f9
    style Parse fill:#fff8dc
```

**ソース:** [src/signal/monitor.ts:358-368]()

`runSignalSseLoop` 関数は signal-cli デーモンの `/v1/receive` エンドポイントへの永続的 SSE 接続を確立します。それは継続的に Server-Sent Events を読み込み、提供された `onEvent` コールバックにディスパッチします。ループは以下を処理します:

- **接続失敗**: 指数バックオフでの自動再接続
- **中止シグナル**: `abortSignal` がトリガーされた場合のクリーンシャットダウン
- **イベント解析**: SSE データフレームからの JSON ペイロードの抽出
- **エラーアイソレーション**: 個別のイベントハンドラーの失敗がストリームを破壊しない

signal-cli から受信した各イベントは `createSignalEventHandler` に渡されます。これはコアメッセージ処理ロジックをラップします。ハンドラーは非同期で呼び出され、エラーはイベントストリームを妨げることなくキャッチされログに記録されます。

**ソース:** [src/signal/monitor.ts:363-367]()

---

## メッセージ処理

**Signal イベントハンドラーフロー**

```mermaid
flowchart TB
    Start["SSE ストリームからのイベント"]

    EventType{"イベントタイプ?"}

    IsMessage["メッセージデータを抽出"]
    IsSyncMessage["syncMessage をチェック"]
    IsReceiptMessage["receiptMessage をチェック"]
    IsDataMessage["dataMessage をチェック"]
    IsReaction["reaction フィールドをチェック"]

    ExtractSender["resolveSignalSender"]
    ExtractGroup["groupInfo を抽出"]

    CheckPolicy{"アクセス制御"}
    DMPolicy["dmPolicy をチェック"]
    GroupPolicy["groupPolicy をチェック"]
    AllowCheck["isSignalSenderAllowed"]

    FetchAttach["RPC で fetchAttachment"]
    SaveMedia["saveMediaBuffer"]

    BuildContext["InboundContext を構築"]
    AddEnvelope["Signal 封筒ヘッダーを追加"]
    AddHistory["グループ履歴を追加"]

    RouteAgent["getReplyFromConfig でルーティング"]
    DeliverReply["deliverReplies"]
    ChunkText["chunkTextWithMode"]
    SendRPC["sendMessageSignal"]

    ReactFilter["shouldEmitSignalReactionNotification"]
    ReactEvent["enqueueSystemEvent"]

    SendReadReceipt["読み取り確認を送信 (有効な場合)"]

    Start --> EventType

    EventType -->|"message"| IsMessage
    EventType -->|"reaction"| IsReaction
    EventType -->|"other"| Start

    IsMessage --> IsSyncMessage
    IsSyncMessage -->|"yes"| ExtractSender
    IsSyncMessage -->|"no"| IsReceiptMessage
    IsReceiptMessage -->|"yes"| ExtractSender
    IsReceiptMessage -->|"no"| IsDataMessage
    IsDataMessage -->|"yes"| ExtractSender
    IsDataMessage -->|"no"| Start

    ExtractSender --> ExtractGroup
    ExtractGroup --> CheckPolicy

    CheckPolicy -->|"DM"| DMPolicy
    CheckPolicy -->|"Group"| GroupPolicy

    DMPolicy --> AllowCheck
    GroupPolicy --> AllowCheck

    AllowCheck -->|"拒否"| Start
    AllowCheck -->|"許可"| FetchAttach

    FetchAttach -->|"添付ファイルあり"| SaveMedia
    FetchAttach -->|"添付ファイルなし"| BuildContext
    SaveMedia --> BuildContext

    BuildContext --> AddEnvelope
    AddEnvelope --> AddHistory
    AddHistory --> RouteAgent

    RouteAgent --> DeliverReply
    DeliverReply --> ChunkText
    ChunkText --> SendRPC
    SendRPC --> SendReadReceipt

    IsReaction --> ReactFilter
    ReactFilter -->|"許可"| ReactEvent
    ReactFilter -->|"拒否"| Start
    ReactEvent --> Start

    SendReadReceipt --> Start

    style CheckPolicy fill:#ffe6e6
    style RouteAgent fill:#e6f7ff
    style SendRPC fill:#fff8dc
```

**ソース:** [src/signal/monitor.ts:330-356]()

`createSignalEventHandler` 関数は完全なメッセージ処理パイプラインを調整します:

1. **イベント分類**: イベントペイロードを調べて、それがメッセージ、リアクション、タイピングインジケーター、またはレシートであるか決定
2. **送信者解決**: `source` または `sourceUuid` フィールドから `resolveSignalSender` 経由で送信者電話番号/UUID を抽出
3. **グループ検出**: `groupInfo` をチェックして DM とグループメッセージを区別
4. **アクセス制御**: `isSignalSenderAllowed` を `allowFrom` (DM) または `groupAllowFrom` (グループ) に基づいて適用
5. **添付ファイル取得**: 添付ファイルのあるメッセージの場合、`fetchAttachment` を呼び出し:
   - `getAttachment` RPC で attachment ID でリクエスト
   - base64 エンコードされたデータを受信
   - `saveMediaBuffer` でデコードして保存
   - ローカルファイルパスを返す
6. **コンテキスト構築**: `InboundContext` を以下で構築:
   - Signal 封筒ヘッダー: `[Signal {name} {phone/uuid} {timestamp}]`
   - メッセージテキストまたは添付ファイルプレースホルダー
   - 送信者メタデータ
   - グループ履歴 (適用可能な場合)
7. **エージェントルーティング**: 適切なエージェントとセッションにルーティングする `getReplyFromConfig` を呼び出す
8. **レスポンス配信**: `deliverReplies` がテキストを (必要に応じて) チャンキングし `sendMessageSignal` で送信
9. **読み取り確認**: `sendReadReceipts` が有効でデーモンがサポートしている場合、メッセージを読みとしてマーク

**ソース:** [src/signal/monitor.ts:209-251](), [src/signal/monitor.ts:172-207]()

---

## 添付ファイル

**Signal 添付ファイル取得アーキテクチャ**

```mermaid
sequenceDiagram
    participant Handler as "イベントハンドラー"
    participant Fetch as "fetchAttachment"
    participant RPC as "signalRpcRequest"
    participant CLI as "signal-cli API"
    participant Media as "saveMediaBuffer"
    participant Store as "メディアストア"

    Handler->>Handler: "メッセージで添付ファイルを検出"
    Handler->>Fetch: "fetchAttachment(params)"

    Note over Fetch: "attachment.id をチェック"<br/>"size vs maxBytes を比較"

    alt サイズが制限を超過
        Fetch-->>Handler: "エラー: 制限超過"
    end

    Fetch->>RPC: "signalRpcRequest('getAttachment')"
    Note over RPC: "id: {attachment.id}"<br/>"account: {account}"<br/>"groupId or recipient"

    RPC->>CLI: "POST /v2/send"
    CLI-->>RPC: "{data: base64}"
    RPC-->>Fetch: "添付ファイルデータ"

    Fetch->>Fetch: "Buffer.from(data, 'base64')"
    Fetch->>Media: "saveMediaBuffer(buffer)"
    Media->>Store: "Write to ~/.openclaw/media/"
    Store-->>Media: "ファイルパス + メタデータ"
    Media-->>Fetch: "{path, contentType}"

    Fetch-->>Handler: "{path, contentType}"
    Handler->>Handler: "InboundContext に追加"
```

**ソース:** [src/signal/monitor.ts:172-207]()

`fetchAttachment` 関数は Signal 添付ファイル取得を実装します:

**パラメータ**:
- `baseUrl`: signal-cli HTTP API ベース URL
- `account`: Signal アカウント番号 (オプション、設定から推論)
- `attachment`: `id`、`contentType`、`filename`、`size` を含む添付ファイルメタデータオブジェクト
- `sender`: DM 添付ファイル用の送信者電話番号/UUID
- `groupId`: グループ添付ファイル用のグループID
- `maxBytes`: サイズ制限 (`mediaMaxMb * 1024 * 1024` から)

**プロセス**:
1. `attachment.id` が存在することを検証
2. `attachment.size` を `maxBytes` と比較、超過すればスロー
3. `groupId` (グループ用) または `recipient` (DM 用) で RPC パラメータを構築
4. `signalRpcRequest("getAttachment", params)` を呼び出し:
   - signal-cli の JSON-RPC エンドポイントに POST
   - base64 エンコードされた `data` フィールドを含む応答を受信
5. base64 を Buffer にデコード
6. `saveMediaBuffer(buffer, contentType, "inbound", maxBytes)` を呼び出し:
   - `~/.openclaw/media/inbound/` に書き込み
   - コンテントタイプからの拡張子付きで一意のファイル名を生成
   - `{id, path, size, contentType}` を返す
7. 呼び出し元に `{path, contentType}` を返す

添付ファイルパスはその後 `InboundContext` に含まれ、エージェントによる処理に利用可能になります。

**ソース:** [src/signal/monitor.ts:172-207]()

---

## リアクション

**Signal リアクション通知システム**

Signal 統合は `reactionNotifications` 設定で詳細なフィルタリングを使用してリアクション通知をサポートします:

| モード | 動作 |
|------|----------|
| `"off"` | リアクション通知なし |
| `"own"` | ボット自身のメッセージへのリアクションのみ通知 (デフォルト) |
| `"allowlist"` | `reactionAllowlist` のユーザーからのリアクションのみ通知 |
| `"all"` | すべてのリアクションを通知 |

**リアクション処理フロー**:

```mermaid
flowchart TB
    Event["SSE からのリアクションイベント"]
    Validate["isSignalReactionMessage"]
    Extract["絵文字、タイムスタンプ、ターゲットを抽出"]
    ResolveTargets["resolveSignalReactionTargets"]

    CheckMode{"通知モード?"}

    ModeOff["mode = 'off'"]
    ModeOwn["mode = 'own'"]
    ModeAllowlist["mode = 'allowlist'"]
    ModeAll["mode = 'all'"]

    CheckOwnMsg["ターゲットとアカウントを比較"]
    CheckSenderList["isSignalSenderAllowed"]

    BuildEvent["buildSignalReactionSystemEventText"]
    Enqueue["enqueueSystemEvent"]

    Skip["通知をスキップ"]

    Event --> Validate
    Validate -->|"無効"| Skip
    Validate -->|"有効"| Extract
    Extract --> ResolveTargets
    ResolveTargets --> CheckMode

    CheckMode --> ModeOff
    CheckMode --> ModeOwn
    CheckMode --> ModeAllowlist
    CheckMode --> ModeAll

    ModeOff --> Skip
    ModeOwn --> CheckOwnMsg
    ModeAllowlist --> CheckSenderList
    ModeAll --> BuildEvent

    CheckOwnMsg -->|"ボットのメッセージでない"| Skip
    CheckOwnMsg -->|"ボットのメッセージ"| BuildEvent

    CheckSenderList -->|"リストにない"| Skip
    CheckSenderList -->|"リストにある"| BuildEvent

    BuildEvent --> Enqueue

    style CheckMode fill:#ffe6e6
    style Enqueue fill:#e6ffe6
```

**ソース:** [src/signal/monitor.ts:95-103](), [src/signal/monitor.ts:105-131](), [src/signal/monitor.ts:133-143]()

**リアクション検証** (`isSignalReactionMessage`):
- 空でない `emoji` 文字列が必要
- 有効な `targetSentTimestamp` (数値 > 0) が必要
- 少なくとも1つのターゲット識別子 (`targetAuthor` または `targetAuthorUuid`) が必要

**ターゲット解決** (`resolveSignalReactionTargets`):
- UUID と電話番号の両方のターゲットを抽出
- `{kind: "phone" | "uuid", id: string, display: string}` の配列を返す
- 電話番号を E.164 形式に正規化

**通知フィルタリング** (`shouldEmitSignalReactionNotification`):
- **"off"**: すぐに `false` を返す
- **"own"**: `account` (ボット番号) とリアクションターゲットを比較。電話番号と UUID の両方の形式で一致
- **"allowlist"**: `isSignalSenderAllowed(sender, reactionAllowlist)` を呼び出してリアクターが認可されているかチェック
- **"all"**: すべての場合に `true` を返す

**イベント生成** (`buildSignalReactionSystemEventText`):
```
Signal reaction added: {emoji} by {actor} msg {messageId} [from {target}] [in {group}]
```

イベントは `enqueueSystemEvent` を介してセッションのシステムイベントストリームにエンキューされ、エージェントがコンテキスト情報として利用できるようになります。

**ソース:** [src/signal/monitor.ts:81-143]()

---

## アクセス制御

**Signal アクセス制御デシジョンツリー**

```mermaid
flowchart TB
    Message["受信 Signal メッセージ"]

    ChatType{"チャットタイプ?"}

    DM["ダイレクトメッセージ"]
    Group["グループメッセージ"]

    DMPolicyCheck{"dmPolicy?"}
    GroupPolicyCheck{"groupPolicy?"}

    Open1["'open' → 許可"]
    Allowlist1["'allowlist'"]
    Pairing["'pairing'"]

    Open2["'open' → 許可"]
    Allowlist2["'allowlist'"]

    CheckDMList["allowFrom をチェック"]
    CheckGroupList["groupAllowFrom をチェック"]

    CheckPairingStore["ペアリングストアをチェック"]

    SenderCheck1["isSignalSenderAllowed(sender, allowFrom)"]
    SenderCheck2["isSignalSenderAllowed(sender, groupAllowFrom)"]

    Allow["メッセージを処理"]
    Deny["メッセージを無視"]

    SendPairingCode["ペアリングリクエストを送信"]

    Message --> ChatType

    ChatType --> DM
    ChatType --> Group

    DM --> DMPolicyCheck
    Group --> GroupPolicyCheck

    DMPolicyCheck --> Open1
    DMPolicyCheck --> Allowlist1
    DMPolicyCheck --> Pairing

    GroupPolicyCheck --> Open2
    GroupPolicyCheck --> Allowlist2

    Open1 --> Allow
    Open2 --> Allow

    Allowlist1 --> CheckDMList
    Allowlist2 --> CheckGroupList

    CheckDMList --> SenderCheck1
    CheckGroupList --> SenderCheck2

    SenderCheck1 -->|"リスト内"| Allow
    SenderCheck1 -->|"リスト外"| Deny

    SenderCheck2 -->|"リスト内"| Allow
    SenderCheck2 -->|"リスト外"| Deny

    Pairing --> CheckPairingStore
    CheckPairingStore -->|"ペア済み"| Allow
    CheckPairingStore -->|"未ペア"| SendPairingCode
    SendPairingCode --> Deny

    style DMPolicyCheck fill:#ffe6e6
    style GroupPolicyCheck fill:#ffe6e6
    style Allow fill:#e6ffe6
    style Deny fill:#ffcccc
```

**ソース:** [src/signal/monitor.ts:256-286]()

Signal 統合は2つの独立したポリシーシステムを通じてアクセス制御を適用します:

**DM ポリシー** (`dmPolicy`):
- **"open"**: すべての DM を許可
- **"allowlist"**: `allowFrom` 内の番号/UUID からの DM のみ許可
- **"pairing"**: ペアリングコードによる明示的なペアリングが必要 (Telegram/WhatsApp のペアリングに類似)

**グループポリシー** (`groupPolicy`):
- **"open"**: すべてのグループメッセージを許可
- **"allowlist"**: `groupAllowFrom` 内の番号/UUID からのグループメッセージのみ許可

**送信者検証** (`isSignalSenderAllowed`):
この関数は送信者識別子を許可リストと照合し、以下の両方をサポート:
- **電話番号**: E.164 形式 (`+15551234567`) に正規化
- **UUIDs**: Signal UUID 形式 (`uuid:...` または生の UUID)

`groupAllowFrom` リストは明示的に設定されていない場合 `allowFrom` にデフォルト設定され、単一の統合許可リストを提供します。両方のリストは `normalizeAllowList` で正規化され:
1. すべてのエントリを文字列に変換
2. ホワイトスペースをトリム
3. 空のエントリをフィルタリング

**ソース:** [src/signal/monitor.ts:71-73](), [src/signal/monitor.ts:270-279]()

---

## レスポンス配信

**Signal レスポンスチャンキングと配信**

```mermaid
sequenceDiagram
    participant Agent as "エージェントランタイム"
    participant Deliver as "deliverReplies"
    participant Chunk as "chunkTextWithMode"
    participant Send as "sendMessageSignal"
    participant RPC as "signalRpcRequest"
    participant CLI as "signal-cli API"

    Agent->>Deliver: "ReplyPayload[]"

    loop 各レスポンスについて
        Deliver->>Deliver: "テキスト + mediaUrls を抽出"

        alt メディアなし
            Deliver->>Chunk: "chunk(text, textLimit, chunkMode)"
            Chunk-->>Deliver: "テキストチャンク[]"

            loop 各チャンクについて
                Deliver->>Send: "sendMessageSignal(target, chunk)"
                Send->>RPC: "signalRpcRequest('send')"
                RPC->>CLI: "POST /v2/send"
                CLI-->>RPC: "成功"
                RPC-->>Send: "成功"
                Send-->>Deliver: "送信完了"
            end
        else メディアあり
            loop 各 mediaUrl について
                alt 最初のメディア
                    Deliver->>Send: "sendMessageSignal(target, text, {mediaUrl})"
                else 続きのメディア
                    Deliver->>Send: "sendMessageSignal(target, '', {mediaUrl})"
                end

                Send->>RPC: "signalRpcRequest('send')"
                Note over RPC: "メディアファイルを読み込み"<br/>"base64 にエンコード"<br/>"ペイロードに含む"
                RPC->>CLI: "POST /v2/send"
                CLI-->>RPC: "成功"
                RPC-->>Send: "成功"
                Send-->>Deliver: "送信完了"
            end
        end

        Deliver->>Deliver: "配信をログ"
    end
```

**ソース:** [src/signal/monitor.ts:209-251]()

`deliverReplies` 関数は自動テキストチャンキングとメディア添付サポートを使用してレスポンス配信を処理します:

**パラメータ**:
- `replies`: エージェントからの `ReplyPayload` オブジェクトの配列
- `target`: 受信者電話番号またはグループID
- `baseUrl`: signal-cli API URL
- `account`: Signal アカウント番号
- `accountId`: 内部アカウント識別子
- `textLimit`: メッセージあたりの最大テキスト長 (設定から)
- `chunkMode`: `"length"` または `"newline"` チャンキング戦略
- `maxBytes`: メディアサイズ制限

**テキストのみメッセージ**:
1. `chunkTextWithMode(text, textLimit, chunkMode)` を呼び出し:
   - **"length" モード**: `textLimit` 文字で分割
   - **"newline" モード**: 改行で分割、`textLimit` を尊重
2. 各チャンクを `sendMessageSignal(target, chunk, {...})` で送信
3. 各チャンクの配信をログ

**メディアありメッセージ**:
1. `mediaUrls` 配列を反復
2. 最初のメディアをキャプション付き (完全なテキスト) で送信
3. 続きのメディアをキャプションなし (空文字列) で送信
4. `mediaUrl` パラメータでの各 `sendMessageSignal` 呼び出し:
   - `mediaUrl` からローカルメディアファイルを読み込み
   - base64 にエンコード
   - RPC ペイロードの `base64Attachments` フィールドに含める
5. 各メディアアイテムの配信をログ

チャンキングモードは `resolveChunkMode(cfg, "signal", accountId)` で解決され:
1. Signal 固有の `channels.signal.chunkMode`
2. グローバル `messages.chunkMode`
3. デフォルトは `"length"`

**ソース:** [src/signal/monitor.ts:209-251](), [src/signal/monitor.ts:268]()

---

## エラーハンドリングと再接続

Signal 統合は複数レベルで堅牢なエラーハンドリングを実装します:

**デーモン起動失敗**:
- `waitForSignalDaemonReady` が指数バックオフで `startupTimeoutMs` までポーリング
- 10秒後およびそれ以降10秒ごとに進捗をログ
- タイムアウト超過時にスロー、呼び出し元に伝播

**SSE 接続失敗**:
- `runSignalSseLoop` が接続ドロップで自動再接続
- 再接続試行間で指数バックオフを実装
- `abortSignal` をクリーンシャットダウンに尊重
- 個別のイベント解析エラーはログされるがストリームを壊さない

**RPC 失敗**:
- `signalRpcRequest` にタイムアウト処理を含む
- 接続エラーがコンテキスト付きでログ
- 再試行ロジックは呼び出し側によって処理 (例: 一時的な失敗時の添付ファイル取得再試行)

**イベントハンドラー失敗**:
- イベントハンドラーは `monitorSignalProvider` 内で try-catch でラップ
- エラーは `runtime.error` でログされるがイベントストリームを妨げない
- 1つの失敗メッセージが後続メッセージの処理を防止しないように保証

**クリーンアップ**:
```typescript
try {
  // ... デーモン起動、SSE ループ
} catch (err) {
  if (opts.abortSignal?.aborted) return;
  throw err;
} finally {
  opts.abortSignal?.removeEventListener("abort", onAbort);
  daemonHandle?.stop();
}
```

**ソース:** [src/signal/monitor.ts:318-376]()

---

## チャネルルーティングとの統合

Signal 統合は OpenClaw のチャネルルーティングシステムとシームレスに統合されます:

**セッションキー解決**:
- **DM セッション**: `agent:{agentId}:signal:dm:{phone}`
- **グループセッション**: `agent:{agentId}:signal:group:{groupId}`

セッションキーは (レスポンスフロー内で呼び出される) `resolveAgentRoute` で解決され:
1. `routing.bindings` 設定に基づいてエージェントバインディングを決定
2. バインディングが一致しない場合はデフォルトエージェントを適用
3. Signal 固有のピア識別子でセッションキーを構築

**メッセージルーティング**:
イベントハンドラーで構築される `InboundContext` には以下が含まれます:
- `Channel: "signal"`
- `ChannelAccountId: accountId`
- `PeerId`: 電話番号 (DM) またはグループID (グループ)
- `SenderId`: 送信者電話番号/UUID
- `SenderName`: 連絡先情報からの表示名
- `ChatType: "dm" | "group"`
- `WasMentioned`: Signal には不適用 (@メンションなし)

このコンテキストは `getReplyFromConfig` に渡され、適切なエージェントとセッションマネージャーにルーティングされます。

**ソース:** [src/signal/monitor.ts:330-356]()

---
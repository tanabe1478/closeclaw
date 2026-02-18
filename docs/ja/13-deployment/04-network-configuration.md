---
title: "ネットワーク設定"
original_title: "Network Configuration"
source: "deepwiki:openclaw/openclaw"
chapter: 13
section: 4
---
# ページ: ネットワーク設定

# ネットワーク設定

<details>
<summary>関連ソースファイル</summary>

この wiki ページの作成コンテキストとして使用されたファイル:

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



## 目的と範囲

このドキュメントでは、バインドモード、ポート管理、認証、Tailscale 統合、webhook 設定を含む OpenClaw ゲートウェイのネットワーク設定について説明します。ネットワーク設定は、ゲートウェイがどのように接続をリッスンし、外部サービスがそれにどのように到達するかを決定します。

ゲートウェイサービス管理（インストール、開始/停止）については、[ゲートウェイサービス管理](#3.3) を参照してください。リモートアクセスのパターンについては、[リモートアクセス](#3.4) を参照してください。セキュリティポリシーについては、`README.md` で参照されているセキュリティドキュメントを確認してください。

---

## ゲートウェイのバインドモード

ゲートウェイは、どのネットワークインターフェースが接続を受け付けるかを制御する 5 つのバインドモードをサポートしています:

| バインドモード | ホストアドレス | 使用ケース |
|---------------|---------------|----------|
| `loopback` | `127.0.0.1` | ローカル専用アクセス（Tailscale のデフォルト） |
| `lan` | `0.0.0.0` | すべてのネットワークインターフェース |
| `tailnet` | Tailscale IP | Tailscale ネットワークインターフェースのみにバインド |
| `auto` | `127.0.0.1` → `0.0.0.0` | loopback を優先、利用不可の場合は LAN にフォールバック |
| `custom` | ユーザー指定の IP | 指定された IP にバインド、`0.0.0.0` にフォールバック |

### バインドモードの解決

```mermaid
flowchart TD
    Start["gateway.bind config 値"] --> CheckMode{バインドモード?}

    CheckMode -->|loopback| Loopback["127.0.0.1 にバインド"]
    CheckMode -->|lan| LAN["0.0.0.0 にバインド"]
    CheckMode -->|tailnet| ResolveTailnet["pickPrimaryTailnetIPv4() を呼び出し"]
    CheckMode -->|custom| ResolveCustom["gateway.customBindHost を使用"]
    CheckMode -->|auto| TryLoopback["127.0.0.1 を試行"]

    ResolveTailnet --> TailnetCheck{Tailscale IP<br/>利用可能?}
    TailnetCheck -->|はい| BindTailnet["Tailscale IP にバインド"]
    TailnetCheck -->|いいえ| FallbackLoopback["127.0.0.1 にフォールバック"]

    ResolveCustom --> ValidateIP{有効な IPv4?}
    ValidateIP -->|はい| BindCustom["カスタム IP にバインド"]
    ValidateIP -->|いいえ| FallbackZero["0.0.0.0 にフォールバック"]

    TryLoopback --> LoopbackCheck{Loopback<br/>利用可能?}
    LoopbackCheck -->|はい| Loopback
    LoopbackCheck -->|いいえ| LAN

    Loopback --> Done["ゲートウェイがリッスン"]
    LAN --> Done
    BindTailnet --> Done
    FallbackLoopback --> Done
    BindCustom --> Done
    FallbackZero --> Done
```

**ソース:**
- [src/wizard/onboarding.gateway-config.ts:42-286]()
- [src/commands/onboard-helpers.ts:437-466]()
- [src/gateway/net.js]() (インポート経由で参照)
- [src/infra/tailnet.js]() (`pickPrimaryTailnetIPv4` 経由で参照)

### カスタム IP の検証

カスタムバインドモードには有効な IPv4 アドレスが必要です。検証は設定の際に行われます:

```mermaid
flowchart LR
    Input["ユーザー入力:<br/>gateway.customBindHost"] --> Split["'.' で分割"]
    Split --> CountCheck{4 オクテット?}
    CountCheck -->|いいえ| Invalid["無効:<br/>IPv4 形式ではない"]
    CountCheck -->|はい| OctetCheck{各オクテット<br/>0-255?}
    OctetCheck -->|いいえ| Invalid
    OctetCheck -->|はい| NoLeading{先頭の<br/>ゼロなし?}
    NoLeading -->|いいえ| Invalid
    NoLeading -->|はい| Valid["有効な IPv4"]
```

**ソース:**
- [src/wizard/onboarding.gateway-config.ts:73-106]()
- [src/commands/configure.gateway.ts:69-91]()

---

## ポート設定

ゲートウェイは、HTTP/WebSocket 接続のために単一の TCP ポートでリッスンします。デフォルトは `18789` です。

### ポート解決チェーン

```mermaid
flowchart TD
    Start["ゲートウェイポートの解決"] --> EnvCheck{OPENCLAW_GATEWAY_PORT<br/>環境変数が設定されている?}
    EnvCheck -->|はい| ParseEnv["環境変数値を解析"]
    EnvCheck -->|いいえ| ConfigCheck{gateway.port<br/>設定に存在?}

    ParseEnv --> EnvValid{有効な番号?}
    EnvValid -->|はい| UseEnv["環境変数ポートを使用"]
    EnvValid -->|いいえ| ConfigCheck

    ConfigCheck -->|はい| UseConfig["設定ポートを使用"]
    ConfigCheck -->|いいえ| UseDefault["DEFAULT_GATEWAY_PORT<br/>(18789) を使用"]

    UseEnv --> Return["ポートを返す"]
    UseConfig --> Return
    UseDefault --> Return
```

**ソース:**
- [src/config/config.ts] (`DEFAULT_GATEWAY_PORT`, `resolveGatewayPort` をエクスポート)
- [src/wizard/onboarding.ts:294]()

### ポート競合の検出

ゲートウェイは開始前にポートの利用可能性を確認します。ポートが使用されている場合、可能性の所有者を特定します:

| ポートの所有者 | 検出方法 |
|---------------|------------------|
| もう一つのゲートウェイインスタンス | アクティブな WebSocket エンドポイントを確認 |
| SSH トンネル | `-L` フラグを持つ `ssh` プロセスを確認 |
| その他のプロセス | `lsof` (Unix) または `netstat` (Windows) で PID を報告 |

**ソース:**
- [src/infra/ports.ts] (`ensurePortAvailable`, `handlePortError`, `describePortOwner` 経由で参照)
- [src/index.ts:26-28]()
- [docs/gateway/doctor.md:256-261]()

---

## 認証

ゲートウェイは、Tailscale Serve を使用して loopback モードで実行されている場合を除き、すべての接続に認証を要求します。

### 認証モード

```mermaid
flowchart TD
    Start["受信接続"] --> ResolveAuth["resolveGatewayAuth()"]

    ResolveAuth --> TailscaleCheck{Tailscale<br/>Serve がアクティブ?}
    TailscaleCheck -->|はい| AllowTailscale{gateway.auth<br/>.allowTailscale?}
    AllowTailscale -->|true または未設定| UseTailscale["Tailscale ID を使用<br/>(X-Tailscale-User ヘッダー)"]
    AllowTailscale -->|false| CheckMode
    TailscaleCheck -->|いいえ| CheckMode{gateway.auth.mode?}

    CheckMode -->|token| ValidateToken["トークンを検証:<br/>- Authorization ヘッダー<br/>- クエリパラメータ<br/>- gateway.auth.token"]
    CheckMode -->|password| ValidatePassword["パスワードを検証:<br/>- Authorization ヘッダー<br/>- クエリパラメータ<br/>- gateway.auth.password"]
    CheckMode -->|undefined| DefaultLoopback{バインドモード<br/>loopback?}

    DefaultLoopback -->|はい| AllowLoopback["許可（セキュアではないローカル）"]
    DefaultLoopback -->|いいえ| Reject["拒否: 認証が必要"]

    ValidateToken --> TokenCheck{トークンが一致?}
    TokenCheck -->|はい| Allow["接続を許可"]
    TokenCheck -->|いいえ| Reject

    ValidatePassword --> PasswordCheck{パスワードが一致?}
    PasswordCheck -->|はい| Allow
    PasswordCheck -->|いいえ| Reject

    UseTailscale --> Allow
```

**ソース:**
- [src/gateway/auth.js] (`resolveGatewayAuth` 経由で参照)
- [src/commands/doctor.ts:125-159]()
- [src/wizard/onboarding.gateway-config.ts:108-236]()

### トークンの生成

`authMode` が `token` でトークンが提供されていない場合、システムはランダムな 48 文字の 16 進数トークンを生成します:

**ソース:**
- [src/commands/onboard-helpers.ts:68-70]()
- [src/wizard/onboarding.gateway-config.ts:194-202]()

---

## Tailscale 統合

Tailscale は、ゲートウェイへのゼロ構成 VPN アクセスを提供します。2 つのモードがサポートされています:

| モード | アクセス | HTTPS | 認証 |
|--------|--------|-------|----------------|
| `serve` | Tailnet 専用 | Tailscale 証明書経由 | Tailscale ID（オプションのパスワード上書きあり） |
| `funnel` | パブリックインターネット | Tailscale 証明書経由 | パスワード必須 |

### Tailscale アーキテクチャ

```mermaid
flowchart TB
    subgraph Gateway["ゲートウェイプロセス"]
        GatewayBind["ゲートウェイは<br/>127.0.0.1:18789 にバインド"]
    end

    subgraph Tailscale["Tailscale デーモン"]
        TailscaleServe["tailscale serve<br/>または<br/>tailscale funnel"]
        TailscaleProxy["リバースプロキシ<br/>(HTTPS を終端)"]
    end

    subgraph Clients["クライアント"]
        TailnetClient["Tailnet デバイス<br/>(https://machine-name.ts.net)"]
        InternetClient["インターネットユーザー<br/>(Funnel 経由)"]
    end

    TailnetClient -->|HTTPS リクエスト| TailscaleProxy
    InternetClient -->|HTTPS リクエスト<br/>(Funnel のみ)| TailscaleProxy

    TailscaleProxy -->|localhost:18789 への HTTP| GatewayBind

    TailscaleServe -.->|設定| TailscaleProxy

    GatewayBind -->|Tailscale がアクティブな場合<br/>bind=loopback を強制| GatewayBind
```

**ソース:**
- [README.md:208-223]()
- [src/wizard/onboarding.gateway-config.ts:124-189]()
- [src/infra/tailscale.ts] (`findTailscaleBinary` 経由で参照)

### Tailscale の制約

ゲートウェイは Tailscale が有効な場合に制約を適用します:

```mermaid
flowchart TD
    Start["Tailscale モードを設定"] --> ModeCheck{tailscaleMode?}

    ModeCheck -->|serve または funnel| BindCheck{gateway.bind?}
    ModeCheck -->|off| NoOp["制約なし"]

    BindCheck -->|loopback| Continue["許可"]
    BindCheck -->|その他| ForceLoopback["bind=loopback を強制<br/>(二重公開を防止)"]

    ForceLoopback --> FunnelCheck{tailscaleMode<br/>= funnel?}
    Continue --> FunnelCheck

    FunnelCheck -->|はい| AuthCheck{auth.mode?}
    FunnelCheck -->|いいえ| Done["設定が有効"]

    AuthCheck -->|password| Done
    AuthCheck -->|その他| ForcePassword["auth.mode=password を強制<br/>(Funnel はパスワード必須)"]

    ForcePassword --> Done
```

**ソース:**
- [src/wizard/onboarding.gateway-config.ts:180-189]()
- [README.md:216-221]()

---

## Webhook 設定

Webhook をサポートするチャネル（Telegram、Zalo、BlueBubbles）は、ポーリングの代わりに HTTP でイベントを受信できます。

### Webhook ルーティング

```mermaid
flowchart LR
    subgraph External["外部サービス"]
        TelegramAPI["Telegram Bot API"]
        ZaloAPI["Zalo Bot API"]
    end

    subgraph Gateway["ゲートウェイ HTTP サーバー"]
        HTTPListener["HTTP リスナー<br/>(gateway.port)"]
        WebhookRouter["Webhook ルーター"]
        TelegramHandler["Telegram webhook ハンドラー"]
        ZaloHandler["Zalo webhook ハンドラー"]
    end

    subgraph Channels["チャネルモニター"]
        TelegramMonitor["Telegram チャネル"]
        ZaloMonitor["Zalo チャネル"]
    end

    TelegramAPI -->|POST to<br/>channels.telegram.webhookUrl| HTTPListener
    ZaloAPI -->|POST to<br/>channels.zalo.webhookUrl| HTTPListener

    HTTPListener --> WebhookRouter

    WebhookRouter -->|パスが<br/>webhookPath に一致| TelegramHandler
    WebhookRouter -->|パスが<br/>webhookPath に一致| ZaloHandler

    TelegramHandler --> TelegramMonitor
    ZaloHandler --> ZaloMonitor
```

**ソース:**
- [docs/channels/zalo.md:113-119]()
- [src/commands/onboard-helpers.ts] (webhook URL 処理)

### Webhook セキュリティ

Webhook はシークレットを使用してリクエストを検証します:

| チャネル | ヘッダー | 設定キー |
|---------|--------|------------|
| Telegram | `X-Telegram-Bot-Api-Secret-Token` | `channels.telegram.webhookSecret` |
| Zalo | `X-Bot-Api-Secret-Token` | `channels.zalo.webhookSecret` |
| BlueBubbles | `bluebubbles-auth` | `channels.bluebubbles.password` |

**ソース:**
- [docs/channels/zalo.md:170-175]()

### Webhook vs ポーリング

```mermaid
flowchart TD
    Start["チャネル設定"] --> WebhookCheck{webhookUrl<br/>が設定されている?}

    WebhookCheck -->|はい| SetupWebhook["Webhook モードを設定"]
    WebhookCheck -->|いいえ| SetupPolling["ポーリングモードを設定"]

    SetupWebhook --> RegisterWebhook["外部サービスに<br/>webhook URL を登録"]
    SetupWebhook --> ListenHTTP["webhookPath で<br/>HTTP POST をリッスン"]

    SetupPolling --> PollLoop["ロングポーリングループ<br/>(getUpdates API)"]

    RegisterWebhook --> Exclusive["Webhook とポーリングは<br/>相互排他的"]
    PollLoop --> Exclusive
```

**ソース:**
- [docs/channels/zalo.md:111-119]()
- [README.md:343-344]()

---

## ネットワーク診断

### ゲートウェイのヘルスチェック

ゲートウェイは監視用にヘルスエンドポイントを公開しています:

```mermaid
flowchart LR
    Client["クライアント"] -->|"callGateway({<br/>method: 'health'<br/>})"| Gateway["ゲートウェイ WebSocket"]
    Gateway --> HealthCheck["ヘルスチェックハンドラー"]
    HealthCheck --> Response["ステータスを返す"]
```

**ソース:**
- [src/commands/onboard-helpers.ts:360-382]()
- [src/gateway/call.js] (`callGateway` 経由で参照)
- [docs/gateway/doctor.md:268-280]()

### 到達性のプロービング

`probeGatewayReachable` 関数はゲートウェイの接続性をテストします:

```mermaid
flowchart TD
    Start["probeGatewayReachable()"] --> BuildRequest["トークン/パスワード付きで<br/>ヘルスリクエストを構築"]
    BuildRequest --> CallGateway["callGateway()"]

    CallGateway --> Success{レスポンス OK?}
    Success -->|はい| ReturnOK["{ok: true} を返す"]
    Success -->|いいえ| CatchError["エラーをキャッチ"]

    CatchError --> SummarizeError["summarizeError():<br/>最初の行を抽出<br/>120 文字に制限"]
    SummarizeError --> ReturnFail["{ok: false,<br/>detail: error} を返す"]
```

**ソース:**
- [src/commands/onboard-helpers.ts:360-433]()

---

## ファイアウォールの考慮事項

### 必要なポート

| ポート | プロトコル | 用途 | 必要な用途 |
|--------|----------|---------|--------------|
| `gateway.port` (デフォルト 18789) | TCP | ゲートウェイ HTTP/WebSocket | すべてのモード |
| 443 | TCP | Tailscale Serve/Funnel (アウトバウンド) | Tailscale モード |
| 可変 | UDP | Tailscale WireGuard | Tailscale モード |

### バインドモードのセキュリティ

```mermaid
flowchart TD
    Start["バインドモードを選択"] --> SecurityLevel{必要な<br/>セキュリティレベル?}

    SecurityLevel -->|"最も高い<br/>(ローカルのみ)"| Loopback["bind=loopback<br/>+ トークン認証"]
    SecurityLevel -->|"高<br/>(プライベートネットワーク)"| Tailnet["bind=tailnet<br/>+ Tailscale 認証"]
    SecurityLevel -->|"中<br/>(信頼された LAN)"| LAN["bind=lan<br/>+ トークン/パスワード認証"]
    SecurityLevel -->|"パブリックアクセス"| Funnel["Tailscale Funnel<br/>+ パスワード認証<br/>+ bind=loopback"]

    Loopback --> Firewall["外部公開なし"]
    Tailnet --> Firewall["Tailnet ファイアウォールルール"]
    LAN --> Firewall["ネットワークファイアウォールルール"]
    Funnel --> Firewall["Tailscale エッジフィルタリング"]
```

**ソース:**
- [README.md:107-120]()
- [src/wizard/onboarding.gateway-config.ts:180-189]()
- [src/wizard/onboarding.ts:47-88]()

---

## 設定リファレンス

### コアネットワーク設定

```typescript
{
  gateway: {
    port: 18789,                    // ゲートウェイの TCP ポート
    bind: "loopback",               // loopback | lan | tailnet | auto | custom
    customBindHost?: "192.168.1.100", // bind=custom の場合必須

    auth: {
      mode: "token",                // token | password
      token?: "...",                // mode=token の場合必須
      password?: "...",             // mode=password の場合必須
      allowTailscale?: true,        // Tailscale ID 認証を許可
    },

    tailscale: {
      mode: "off",                  // off | serve | funnel
      resetOnExit: false,           // シャットダウン時に Tailscale 設定をリセット
    },
  }
}
```

**ソース:**
- [src/config/types.ts] (型定義)
- [src/wizard/onboarding.gateway-config.ts:237-286]()

### Webhook 設定の例

```typescript
{
  channels: {
    telegram: {
      webhookUrl: "https://example.com/telegram",
      webhookSecret: "secure-secret-token",
      webhookPath: "/telegram",  // URL パスがデフォルト
    },

    zalo: {
      webhookUrl: "https://example.com/zalo",
      webhookSecret: "8-to-256-char-secret",
      webhookPath: "/zalo",
    },
  }
}
```

**ソース:**
- [docs/channels/zalo.md:170-176]()
- [README.md:343-354]()

---

## コントロール UI アクセス

コントロール UI は、ゲートウェイのバインドアドレス経由で HTTP でアクセスできます:

```mermaid
flowchart LR
    BindMode["gateway.bind モード"] --> ResolveHost["resolveControlUiLinks()"]

    ResolveHost --> HostDecision{バインドモード?}

    HostDecision -->|loopback| LocalURL["http://127.0.0.1:port/"]
    HostDecision -->|lan| LANAddress["pickPrimaryLanIPv4()<br/>http://IP:port/"]
    HostDecision -->|tailnet| TailnetAddress["pickPrimaryTailnetIPv4()<br/>http://IP:port/"]
    HostDecision -->|custom| CustomAddress["customBindHost<br/>http://IP:port/"]

    LocalURL --> AddToken["#token=... を追加"]
    LANAddress --> AddToken
    TailnetAddress --> AddToken
    CustomAddress --> AddToken

    AddToken --> FinalURL["ブラウザー用の最終 URL"]
```

**ソース:**
- [src/commands/onboard-helpers.ts:437-466]()
- [README.md:208-223]()

---
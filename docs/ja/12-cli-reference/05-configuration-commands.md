---
title: "設定コマンド"
original_title: "Configuration Commands"
source: "deepwiki:openclaw/openclaw"
chapter: 12
section: 5
---

# 設定コマンド

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
- [scripts/update-clawtributors.types.ts](scripts/update-clawtributors.types.ts)
- [src/config/config.ts](src/config/config.ts)
- [src/index.test.ts](src/index.test.ts)
- [src/index.ts](src/index.ts)
- [tsconfig.json](tsconfig.json)
- [ui/src/styles.css](ui/src/styles.css)
- [ui/src/styles/layout.mobile.css](ui/src/styles/layout.mobile.css)

</details>



このページでは、設定ファイルを管理するための CLI コマンドについて説明します。設定ファイルは `~/.openclaw/openclaw.json` に保存されます。

---

## `openclaw config get <path>`

設定値を読み取ります。

```bash
openclaw config get gateway.port
openclaw config get agents.defaults.model.primary
openclaw config get channels.telegram.enabled
```

**パス構文:**
- ドット表記でネストされた値にアクセス
- 配列インデックスはブラケット表記: `agents.list[0].id`

**Sources:** [docs/cli/index.md:349-359](), [src/config/config.ts:1-15]()

---

## `openclaw config set <path> <value>`

設定値を書き込みます。

```bash
openclaw config set gateway.port 18790
openclaw config set agents.defaults.model.primary "anthropic/claude-opus-4-5"
openclaw config set channels.telegram.enabled true
```

**値の型:**
- 文字列: 引用符で囲むか、そのまま使用
- 数値: そのまま使用
- ブール値: `true` または `false`
- JSON: JSON 文字列を渡す

**Sources:** [docs/cli/index.md:349-359](), [src/config/config.ts:1-15]()

---

## `openclaw config unset <path>`

設定値を削除します。

```bash
openclaw config unset channels.telegram
openclaw config unset agents.defaults.sandbox
```

**Sources:** [docs/cli/index.md:349-359]()

---

## `openclaw configure`

インタラクティブな設定ウィザードを起動します。

```bash
openclaw configure
openclaw configure --section channels
```

**オプション:**
- `--section <name>`: 特定のセクションのみを設定（`channels`、`models`、`agents`、`gateway`）

**Sources:** [docs/cli/index.md:349-359]()

---

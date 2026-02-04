---
title: "【Windows版】claude-memプラグインのインストールから起動テストまで完全ガイド"
emoji: "🧪"
type: "tech"
topics: ["claudecode", "plugin", "windows", "nodejs", "ai"]
published: true
---

## はじめに

Claude Codeのセッション履歴を永続化・検索できるプラグイン「[claude-mem](https://github.com/thedotmack/claude-mem)」を**Windows環境**にインストールし、正常に動作するかテストした手順をまとめました。

Zennや公式ドキュメントにはmacOS/Linux向けの情報が多く、**Windowsでは`~`（チルダ）がホームディレクトリとして展開されない**などのハマりポイントがあります。この記事ではWindows固有の注意点を中心に解説します。

### 対象読者

- Claude Codeを使っている方
- claude-memをWindows環境で導入したい方
- 導入後に正しく動作しているか確認したい方

### 動作確認環境

| 項目 | バージョン |
|------|-----------|
| OS | Windows 10/11 |
| Node.js | v22.17.0 |
| bun | 1.3.8 |
| claude-mem | 9.0.12 |

---

## 1. インストール

### 1-1. 前提条件

以下がインストール済みであることを確認してください。

```powershell
# Node.js（18以上が必要）
node --version

# bun（claude-memの一部機能で使用）
bun --version
```

### 1-2. プラグインのインストール

Claude Codeのプラグインとしてインストールします。

```bash
claude plugins install claude-mem
```

### 1-3. インストール後のファイル確認

Windowsでは以下のパスにファイルが配置されます。

```
C:\Users\<ユーザー名>\.claude\plugins\marketplaces\thedotmack\
├── plugin\
│   └── scripts\
│       ├── CLAUDE.md
│       ├── context-generator.cjs
│       ├── mcp-server.cjs
│       ├── smart-install.js
│       ├── worker-cli.js          ← ワーカー操作用CLI
│       ├── worker-service.cjs     ← ワーカー本体
│       └── worker-wrapper.cjs
├── node_modules\
├── src\
└── tests\
```

ファイルが正しく配置されたか確認するコマンド：

```cmd
dir "%USERPROFILE%\.claude\plugins\marketplaces\thedotmack\plugin\scripts"
```

:::message alert
**Windowsの注意点：`~`（チルダ）は使えない**

公式ドキュメントやmacOS向け記事では `~/.local/share/claude-mem/` のようなパスが書かれていますが、Windowsのコマンドプロンプトでは `~` がホームディレクトリに展開されません。

```cmd
# ❌ これはWindowsでは動かない
node ~/.local/share/claude-mem/plugin/scripts/worker-cli.js start

# ✅ こちらを使う
node "%USERPROFILE%\.claude\plugins\marketplaces\thedotmack\plugin\scripts\worker-cli.js" start
```

`%USERPROFILE%` は `C:\Users\<ユーザー名>` に展開されます。
:::

---

## 2. ワーカーサービスの起動

claude-memはバックグラウンドで動くワーカーサービスとして動作します。

### 2-1. 起動

```cmd
node "%USERPROFILE%\.claude\plugins\marketplaces\thedotmack\plugin\scripts\worker-cli.js" start
```

正常に起動すると以下のように表示されます：

```
Worker started (PID: 54076)
Logs: ~/.claude-mem/logs/worker-2026-02-04.log
```

:::message
bun で直接起動する方法もあります（フォアグラウンド実行）：

```cmd
bun "%USERPROFILE%\.claude\plugins\marketplaces\thedotmack\plugin\scripts\worker-service.cjs"
```

この場合、コマンドプロンプトを閉じるとワーカーも停止します。
:::

### 2-2. 停止

```cmd
node "%USERPROFILE%\.claude\plugins\marketplaces\thedotmack\plugin\scripts\worker-cli.js" stop
```

### 2-3. デフォルトポート

ワーカーサービスは **ポート37777** で待ち受けます。

---

## 3. 起動テスト

ワーカーが正常に動作しているか、各APIを叩いて確認します。

### テスト一覧

| # | テスト項目 | 確認内容 |
|---|----------|----------|
| 1 | Readiness | ワーカーの準備完了状態 |
| 2 | Health | ワーカーの健康状態・バージョン情報 |
| 3 | Stats | データベースの統計情報 |
| 4 | Observations | 記録された観察の一覧 |
| 5 | Search | キーワード検索 |
| 6 | Search by-type | タイプ別検索 |
| 7 | ポート確認 | ネットワーク接続状態 |

### 3-1. Readiness（準備完了確認）

```cmd
curl -s http://127.0.0.1:37777/api/readiness
```

**期待される応答：**

```json
{"status":"ready","mcpReady":true}
```

| フィールド | 正常値 | 意味 |
|-----------|--------|------|
| status | `ready` | ワーカーが準備完了 |
| mcpReady | `true` | MCPサーバーとの連携も準備完了 |

### 3-2. Health（健康状態確認）

```cmd
curl -s http://127.0.0.1:37777/api/health
```

**期待される応答：**

```json
{
  "status": "ok",
  "build": "TEST-008-wrapper-ipc",
  "managed": true,
  "hasIpc": true,
  "platform": "win32",
  "pid": 42876,
  "initialized": true,
  "mcpReady": true
}
```

| フィールド | 確認ポイント |
|-----------|-------------|
| status | `ok` であること |
| platform | Windowsなら `win32` |
| initialized | `true` であること |
| mcpReady | `true` であること |

### 3-3. Stats（統計情報確認）

```cmd
curl -s http://127.0.0.1:37777/api/stats
```

**期待される応答：**

```json
{
  "worker": {
    "version": "9.0.12",
    "uptime": 37,
    "activeSessions": 0,
    "sseClients": 0,
    "port": 37777
  },
  "database": {
    "path": "C:\\Users\\<ユーザー名>\\.claude-mem\\claude-mem.db",
    "size": 4096,
    "observations": 0,
    "sessions": 0,
    "summaries": 0
  }
}
```

| フィールド | 確認ポイント |
|-----------|-------------|
| worker.version | バージョンが表示される |
| worker.port | `37777` であること |
| database.path | データベースファイルのパスが表示される |

:::message
インストール直後は `observations`、`sessions`、`summaries` がすべて `0` です。Claude Codeでセッションを使っていくと自動的に記録が蓄積されます。
:::

### 3-4. Observations（観察一覧）

```cmd
curl -s http://127.0.0.1:37777/api/observations
```

**期待される応答：**

```json
{"items":[],"hasMore":false,"offset":0,"limit":20}
```

APIが正常に応答すればOKです。初回はデータが空（`items: []`）なのは正常です。

### 3-5. Search（キーワード検索）

```cmd
curl -s "http://127.0.0.1:37777/api/search?query=test"
```

**期待される応答：**

```json
{"content":[{"type":"text","text":"No results found matching \"test\""}]}
```

データが空でもAPIが正常に応答すればOKです。

### 3-6. Search by-type（タイプ別検索）

```cmd
curl -s "http://127.0.0.1:37777/api/search/by-type?type=bugfix"
```

**期待される応答：**

```json
{"content":[{"type":"text","text":"No observations found with type \"bugfix\""}]}
```

### 3-7. ポート確認

```cmd
netstat -ano | findstr "37777"
```

**期待される応答：**

```
TCP    127.0.0.1:37777    0.0.0.0:0    LISTENING    42876
```

`LISTENING` 状態であれば正常にポートを占有してリクエストを待ち受けています。

---

## 4. テスト結果の見方

すべてのテストが完了したら、以下の表で判定します。

| # | テスト項目 | 正常の条件 | 結果 |
|---|----------|-----------|------|
| 1 | Readiness | `status: ready` が返る | ✅ / ❌ |
| 2 | Health | `status: ok` が返る | ✅ / ❌ |
| 3 | Stats | バージョン・ポート情報が返る | ✅ / ❌ |
| 4 | Observations | JSON形式で応答が返る | ✅ / ❌ |
| 5 | Search | JSON形式で応答が返る | ✅ / ❌ |
| 6 | Search by-type | JSON形式で応答が返る | ✅ / ❌ |
| 7 | ポート | 37777がLISTENING状態 | ✅ / ❌ |

**全項目✅であれば、claude-memは正常に動作しています。**

---

## 5. 便利なエイリアス設定（Windows版）

毎回長いパスを入力するのは大変なので、バッチファイルやPowerShellのエイリアスを設定しておくと便利です。

### 方法1：バッチファイルを作る

`C:\Users\<ユーザー名>\bin\` フォルダを作り、PATHに追加しておきます。

**cmem-start.bat**

```bat
@echo off
node "%USERPROFILE%\.claude\plugins\marketplaces\thedotmack\plugin\scripts\worker-cli.js" start
```

**cmem-stop.bat**

```bat
@echo off
node "%USERPROFILE%\.claude\plugins\marketplaces\thedotmack\plugin\scripts\worker-cli.js" stop
```

**cmem-status.bat**

```bat
@echo off
curl -s http://127.0.0.1:37777/api/readiness
```

### 方法2：PowerShellプロファイルにエイリアスを追加

`$PROFILE` ファイル（通常 `C:\Users\<ユーザー名>\Documents\PowerShell\Microsoft.PowerShell_profile.ps1`）に以下を追加：

```powershell
function cmem-start {
    node "$env:USERPROFILE\.claude\plugins\marketplaces\thedotmack\plugin\scripts\worker-cli.js" start
}
function cmem-stop {
    node "$env:USERPROFILE\.claude\plugins\marketplaces\thedotmack\plugin\scripts\worker-cli.js" stop
}
function cmem-status {
    curl -s http://127.0.0.1:37777/api/readiness
}
function cmem-stats {
    curl -s http://127.0.0.1:37777/api/stats
}
```

---

## 6. トラブルシューティング

### エラー：「Cannot find module」

```
Error: Cannot find module 'C:\Users\<ユーザー名>\~\.local\share\claude-mem\...'
```

**原因：** Windowsでは `~` がホームディレクトリに展開されない

**対処：** `~` の代わりに `%USERPROFILE%`（cmd）または `$env:USERPROFILE`（PowerShell）を使う

### エラー：「Worker did not become ready within 15 seconds」

**原因：** ワーカーサービスが起動していない

**対処：**

```cmd
rem 1. ワーカーを起動
node "%USERPROFILE%\.claude\plugins\marketplaces\thedotmack\plugin\scripts\worker-cli.js" start

rem 2. 状態確認
curl -s http://127.0.0.1:37777/api/readiness
```

### エラー：ポート37777が既に使用中

**確認：**

```cmd
netstat -ano | findstr "37777"
```

**対処：**

```cmd
rem 表示されたPIDのプロセスを終了
taskkill /PID <PID番号> /F

rem ワーカーを再起動
node "%USERPROFILE%\.claude\plugins\marketplaces\thedotmack\plugin\scripts\worker-cli.js" start
```

### エラー：「thedotmack/package.json が見つからない」

**対処：**

```cmd
mkdir "%USERPROFILE%\.claude\plugins\marketplaces\thedotmack"
echo {"name": "claude-mem", "version": "9.0.12"} > "%USERPROFILE%\.claude\plugins\marketplaces\thedotmack\package.json"
```

---

## 7. 主要APIエンドポイント一覧

| エンドポイント | メソッド | 説明 |
|---------------|---------|------|
| `/api/readiness` | GET | ワーカーの準備完了状態 |
| `/api/health` | GET | 健康状態の詳細情報 |
| `/api/stats` | GET | 統計情報（バージョン、DB状態） |
| `/api/observations` | GET | 記録された観察の一覧 |
| `/api/search?query=キーワード` | GET | キーワードで検索 |
| `/api/search/by-type?type=タイプ` | GET | タイプ別検索（bugfix, featureなど） |
| `/api/search/by-file?file=ファイル名` | GET | ファイル名で検索 |

---

## まとめ

| 項目 | 内容 |
|------|------|
| プラグイン名 | claude-mem |
| バージョン | 9.0.12 |
| ワーカーポート | 37777 |
| Windows起動コマンド | `node "%USERPROFILE%\.claude\plugins\marketplaces\thedotmack\plugin\scripts\worker-cli.js" start` |
| 状態確認 | `curl -s http://127.0.0.1:37777/api/readiness` |

Windows環境での導入は、パスの書き方に注意すればスムーズに進みます。一番のハマりポイントは `~` がWindowsでは展開されないことです。`%USERPROFILE%` に置き換えることを覚えておけば、公式ドキュメントやmacOS向け記事のコマンドもそのまま応用できます。

---

## 参考リンク

- [claude-mem GitHub リポジトリ](https://github.com/thedotmack/claude-mem)
- [claude-memプラグインの導入と使い方（Zenn）](https://zenn.dev/nenene01/articles/claude-mem-setup-guide)
- [Claude Code 公式ドキュメント](https://docs.anthropic.com/claude-code)

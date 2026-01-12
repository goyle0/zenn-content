---
title: "Claude Desktop × GitHub MCP × Zenn連携で技術記事の投稿を自動化する方法"
emoji: "🤖"
type: "tech"
topics: ["claude", "mcp", "github", "zenn", "自動化"]
published: true
---

## はじめに

この記事では、**Claude Desktop**と**GitHub MCP**を連携して、**Zenn**への記事投稿・編集を自動化する方法を解説します。

### この記事で解決できること

- Claude Desktopから直接Zennに記事を投稿できる
- 記事の編集・更新もAIとの会話で完結する
- 画面操作なしで記事管理が可能になる

### 対象読者

- Claude Desktopを使っている方
- Zennで技術記事を書いている方
- 記事投稿の手間を減らしたい方

### 前提条件

- Claude Desktopがインストール済み
- Docker Desktopがインストール済み
- GitHubアカウントを持っている
- ZennとGitHubが連携済み

## 全体の仕組み

```
┌─────────────────┐
│ Claude Desktop  │
│  （AIと会話）    │
└────────┬────────┘
         │ MCP接続
         ▼
┌─────────────────┐
│   GitHub MCP    │
│ （Docker経由）   │
└────────┬────────┘
         │ API呼び出し
         ▼
┌─────────────────┐
│ GitHubリポジトリ │
│ (zenn-content)  │
└────────┬────────┘
         │ 自動連携
         ▼
┌─────────────────┐
│      Zenn       │
│  （記事公開）    │
└─────────────────┘
```

Claude Desktopで「記事を投稿して」と依頼すると、GitHub MCPがリポジトリにファイルを作成し、Zenn連携によって自動的に記事が公開されます。

## セットアップ手順

### 1. GitHub Personal Access Tokenの作成

GitHubでトークンを作成します。

1. GitHubにログイン
2. **Settings** → **Developer settings** → **Personal access tokens** → **Fine-grained tokens**
3. **Generate new token** をクリック
4. 以下を設定：
   - **Token name**: `Claude MCP` など任意の名前
   - **Expiration**: 任意（90日など）
   - **Repository access**: `Only select repositories` → Zenn用リポジトリを選択
   - **Permissions**:
     - **Contents**: `Read and write`
     - **Metadata**: `Read-only`

5. **Generate token** をクリックしてトークンをコピー

:::message alert
トークンは一度しか表示されません。必ずコピーして安全な場所に保存してください。
:::

### 2. Zenn連携用GitHubリポジトリの準備

Zennと連携するリポジトリを作成します（既にある場合はスキップ）。

```bash
# リポジトリ名の例: zenn-content
```

Zennダッシュボードでリポジトリ連携を設定：
1. https://zenn.dev/dashboard/deploys にアクセス
2. **GitHubからのデプロイ** で連携するリポジトリを選択
3. 連携を有効化

### 3. Docker Desktopのインストールと起動

GitHub MCPはDockerコンテナで動作します。

1. [Docker Desktop](https://www.docker.com/products/docker-desktop/)をダウンロード・インストール
2. Docker Desktopを起動
3. タスクバー/メニューバーでDockerが起動していることを確認

### 4. Claude Desktopの設定

Claude Desktopの設定ファイルを編集します。

**設定ファイルの場所**

| OS | パス |
|----|------|
| Windows | `%APPDATA%\Claude\claude_desktop_config.json` |
| Mac | `~/Library/Application Support/Claude/claude_desktop_config.json` |

**設定内容**

```json
{
  "mcpServers": {
    "github": {
      "command": "docker",
      "args": [
        "run",
        "-i",
        "--rm",
        "-e",
        "GITHUB_PERSONAL_ACCESS_TOKEN",
        "ghcr.io/github/github-mcp-server"
      ],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_xxxxxxxxxxxx"
      }
    }
  }
}
```

:::message
`ghp_xxxxxxxxxxxx` の部分を、手順1で作成したトークンに置き換えてください。
:::

### 5. Claude Desktopを再起動

設定を反映するため、Claude Desktopを完全に終了して再起動します。

- Windowsの場合：タスクトレイからも終了
- Macの場合：メニューバーからも終了

再起動後、チャット入力欄の近くに**ハンマーアイコン🔨**が表示されていれば成功です。

## 記事の投稿方法

### 基本的な投稿

Claude Desktopで以下のように依頼します：

```
Zennに記事を投稿してください。

タイトル：はじめてのAWS Lambda入門
トピック：aws, lambda, serverless
内容：
## はじめに
AWS Lambdaは...

## 本文
...
```

Claudeが自動的に：
1. 適切なフォーマットでMarkdownファイルを作成
2. GitHubリポジトリの `articles/` フォルダにコミット
3. Zenn連携により記事が公開される

### 記事ファイルのフォーマット

Zennの記事は以下のフォーマットで作成されます：

```markdown
---
title: "記事タイトル"
emoji: "🚀"
type: "tech"
topics: ["topic1", "topic2", "topic3"]
published: true
---

記事本文...
```

| フィールド | 説明 |
|-----------|------|
| title | 記事のタイトル |
| emoji | 記事に表示される絵文字 |
| type | `tech`（技術記事）または `idea`（アイデア） |
| topics | タグ（3〜5個推奨、英語小文字） |
| published | `true`で公開、`false`で下書き |

## 記事の編集・更新方法

### 既存記事の編集

```
goyle0/zenn-content リポジトリの articles/my-article.md を編集してください。
「## まとめ」セクションに以下を追加：
- ポイント1
- ポイント2
```

### 記事の削除

```
goyle0/zenn-content リポジトリの articles/old-article.md を削除してください。
```

### 記事一覧の確認

```
goyle0/zenn-content リポジトリの articles フォルダにある記事一覧を表示してください。
```

## 便利な使い方

### 下書きとして保存

公開前に内容を確認したい場合：

```
下書きとして保存してください（published: false）
```

### 特定の絵文字を指定

```
絵文字は「💡」を使ってください
```

### 複数記事の一括管理

```
articles フォルダ内のすべての記事のタイトル一覧を表示してください
```

## トラブルシューティング

### GitHub MCPが動作しない場合

1. **Docker Desktopが起動しているか確認**
   - タスクバー/メニューバーでDockerアイコンを確認

2. **トークンの権限を確認**
   - `Contents: Read and write` が必要

3. **設定ファイルの構文を確認**
   - JSONの構文エラー（カンマ、括弧の過不足）がないか

4. **Claude Desktopを再起動**
   - 設定変更後は必ず完全に終了して再起動

### 記事がZennに反映されない場合

1. **Zenn連携が有効か確認**
   - https://zenn.dev/dashboard/deploys で確認

2. **リポジトリ構成を確認**
   - 記事は `articles/` フォルダ直下に配置されているか

3. **フォーマットを確認**
   - frontmatter（`---`で囲まれた部分）が正しいか

### エラーメッセージ別の対処

| エラー | 対処法 |
|--------|--------|
| `Tool execution failed` | Docker Desktopの起動を確認 |
| `401 Unauthorized` | トークンの有効期限・権限を確認 |
| `404 Not Found` | リポジトリ名・パスを確認 |

## まとめ

Claude Desktop × GitHub MCP × Zenn連携により、以下のメリットが得られます：

- **効率化**: 画面操作なしで記事投稿が完結
- **自然言語**: 「記事を投稿して」と話しかけるだけ
- **一元管理**: 記事の作成・編集・削除がすべてClaude経由で可能

この仕組みを活用して、技術記事のアウトプットをより気軽に始めてみてください！

## 参考資料

- [GitHub MCP Server公式](https://github.com/github/github-mcp-server)
- [Zenn CLIドキュメント](https://zenn.dev/zenn/articles/zenn-cli-guide)
- [Docker Desktop](https://www.docker.com/products/docker-desktop/)

---
title: "【2025年版】Claude Code CLIにおすすめのMCPプラグイン｜ハルシネーション防止＆作業効率化"
emoji: "🧠"
type: "tech"
topics: ["claudecode", "mcp", "ai", "開発効率化", "windows"]
published: true
---

## はじめに

Claude Code CLIを使っていて、こんな経験はありませんか？

- 存在しないAPIやメソッドを提案される（ハルシネーション）
- 古いライブラリのドキュメントを元にしたコードが生成される
- 複雑な設計判断で、思考プロセスが見えづらい

これらの問題を解決するために、**MCP（Model Context Protocol）プラグイン**を導入してみました。

本記事では、実際に調査・検証した結果から、**Windows環境のClaude Code CLI**に特におすすめのMCPプラグインを紹介します。

## MCPとは？

**MCP（Model Context Protocol）** は、Anthropic社が開発したオープンソースの通信規格です。

簡単に言うと、**Claude Codeに「外部ツールとの連携機能」を追加するためのプラグイン規格**です。

```
[Claude Code] ←→ [MCPサーバー] ←→ [外部ツール・データソース]
```

MCPサーバーを追加することで、以下のようなことが可能になります：

- 最新のライブラリドキュメントをリアルタイム取得
- 段階的な思考プロセスの可視化
- データベース、GitHub、Slack等との連携

## おすすめMCPプラグイン

調査の結果、以下の2つを特におすすめします。

### 1. Sequential Thinking MCP Server（思考・推論系）

| 項目 | 内容 |
|------|------|
| **提供元** | Anthropic公式 |
| **GitHubスター** | 73,500+（公式リポジトリ） |
| **推奨度** | ⭐⭐⭐⭐⭐ |

#### 特徴

- 複雑な問題を**段階的に分解**して思考
- 途中で**思考の修正・分岐**が可能
- アーキテクチャ設計、デバッグ、設計判断に最適
- **Anthropic公式なので信頼性が非常に高い**

#### どんな時に使う？

- 「マイクロサービス vs モノリス、どちらを選ぶべき？」のような設計判断
- 複雑なバグの原因調査
- リファクタリング計画の立案

#### 公式リポジトリ

https://github.com/modelcontextprotocol/servers/tree/main/src/sequentialthinking

---

### 2. Code Reasoning MCP Server（コーディング特化）

| 項目 | 内容 |
|------|------|
| **提供元** | mettamatt（Sequential Thinkingのフォーク版） |
| **GitHubスター** | 200+ |
| **推奨度** | ⭐⭐⭐⭐☆ |

#### 特徴

- Sequential Thinkingを**コーディング作業に特化**
- バグ分析、コードレビュー、リファクタリング計画に最適
- **プロンプトテンプレートが付属**しているので使いやすい

#### どんな時に使う？

- コードレビュー時のバグ検出
- 「このコードのどこが問題か？」の分析
- リファクタリングの優先順位付け

#### 公式リポジトリ

https://github.com/mettamatt/code-reasoning

---

## インストール手順

### 前提条件

- **Node.js**: v18以上
- **Claude Code CLI**: インストール済み

#### 確認コマンド

```bash
node -v
# → v18.x.x 以上が表示されればOK

npx -v
# → バージョン番号が表示されればOK
```

### インストール

私の環境（Windows 11）では、以下のコマンドでインストールできました。

#### Sequential Thinking

```bash
claude mcp add sequential-thinking -- npx -y @modelcontextprotocol/server-sequential-thinking
```

#### Code Reasoning

```bash
claude mcp add code-reasoning -- npx -y @mettamatt/code-reasoning
```

### インストール確認

```bash
claude mcp list
```

以下のように表示されればOKです：

```
sequential-thinking: npx -y @modelcontextprotocol/server-sequential-thinking
code-reasoning: npx -y @mettamatt/code-reasoning
```

---

## 使い方

### Sequential Thinking の使用例

```
Use sequential thinking to analyze: 
What are the pros and cons of using microservices vs monolith architecture?
```

または日本語で：

```
ステップバイステップで考えて：
このシステムにキャッシュを導入すべきか、判断材料を整理してください。
```

### Code Reasoning の使用例

```
Use code reasoning to review this code and identify potential bugs:

public void processData(List<String> items) {
    for (int i = 0; i <= items.size(); i++) {
        System.out.println(items.get(i));
    }
}
```

このコードには `ArrayIndexOutOfBoundsException` のバグがありますが、Code Reasoningを使うと段階的に問題点を分析してくれます。

---

## 使い分けガイド

| 状況 | 使うツール | プロンプト例 |
|------|-----------|-------------|
| 設計判断・アーキテクチャ | Sequential Thinking | 「Use sequential thinking to...」 |
| コードレビュー・バグ分析 | Code Reasoning | 「Use code reasoning to...」 |
| リファクタリング計画 | Code Reasoning | 「Use code reasoning to plan refactoring...」 |
| 複雑な問題の分解 | Sequential Thinking | 「ステップバイステップで考えて」 |

---

## 補足：他に検討したプラグイン

調査中に見つけた他のプラグインも参考として記載します。

### Context7 MCP Server

- **用途**: ライブラリの最新ドキュメントをリアルタイム取得
- **GitHubスター**: 44,700+
- **特徴**: 古いAPIを提案されるハルシネーション防止に効果的

```bash
# APIキー取得（無料）: https://context7.com/dashboard
claude mcp add context7 -- npx -y @upstash/context7-mcp --api-key YOUR_API_KEY
```

### Playwright MCP Server

- **用途**: ブラウザ操作の自動化
- **提供元**: Microsoft公式
- **特徴**: テスト自動化、Webスクレイピングに便利

---

## トラブルシューティング

### 「npx is not recognized」エラー

**原因**: Node.jsのパスが通っていない

**解決策**:
1. Node.jsを再インストール
2. または環境変数PATHに `C:\Program Files\nodejs\` を追加

### 「Failed to connect to MCP server」エラー

**原因**: 初回実行時のパッケージダウンロードに時間がかかる

**解決策**:
1. 数分待ってから再度試す
2. または事前に手動ダウンロード：

```bash
npx -y @modelcontextprotocol/server-sequential-thinking
npx -y @mettamatt/code-reasoning
```

### MCPサーバーを削除したい場合

```bash
claude mcp remove sequential-thinking
claude mcp remove code-reasoning
```

---

## まとめ

Claude Code CLIに以下のMCPプラグインを導入することで、**ハルシネーション防止**と**作業効率化**が期待できます。

| プラグイン | 主な用途 | 推奨度 |
|-----------|---------|--------|
| Sequential Thinking | 設計判断・複雑な問題分解 | ⭐⭐⭐⭐⭐ |
| Code Reasoning | コードレビュー・バグ分析 | ⭐⭐⭐⭐☆ |

特に **Sequential Thinking** はAnthropic公式で信頼性が高く、まず最初に入れておきたいプラグインです。

ぜひ試してみてください！

---

## 参考リンク

- [MCP公式リポジトリ](https://github.com/modelcontextprotocol/servers)
- [Claude Code MCPドキュメント](https://code.claude.com/docs/en/mcp)
- [Sequential Thinking MCP](https://github.com/modelcontextprotocol/servers/tree/main/src/sequentialthinking)
- [Code Reasoning MCP](https://github.com/mettamatt/code-reasoning)
- [Context7 MCP](https://github.com/upstash/context7)

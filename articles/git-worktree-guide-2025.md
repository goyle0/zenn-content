---
title: "【2025年版】git worktree 完全ガイド - AI駆動開発時代の並列作業術"
emoji: "🌳"
type: "tech"
topics: ["git", "worktree", "vscode", "ai", "productivity"]
published: true
---

## はじめに

開発中に「緊急のバグ修正が入った！でも今の作業をstashするのは面倒...」という経験はありませんか？

**git worktree** を使えば、1つのリポジトリから複数の作業ディレクトリを作成し、異なるブランチで同時に作業できます。特にAI駆動開発（Claude Code、GitHub Copilot等）で並列実行する際には必須の技術となっています。

この記事では、git worktreeの基本から実践的な活用方法までを解説します。

## git worktreeとは？

Git 2.5（2015年）から導入された標準機能で、**1つのリポジトリに対して複数の作業ディレクトリ（ワークツリー）を持てる**仕組みです。

### 従来の課題

```bash
# 従来のワークフロー（面倒なパターン）
git checkout feature-a
# feature-aで作業中...

# 緊急でバグ修正が必要に！
git stash                    # 作業内容を一時退避
git checkout main
git checkout -b hotfix/bug   # バグ修正ブランチ作成
# バグ修正作業...

git checkout feature-a
git stash pop                # 作業内容を復元
# どのstashだっけ...？
```

### git worktreeによる解決

```bash
# worktreeを使ったワークフロー
# feature-aで作業中のまま...

git worktree add ../hotfix-bug main   # 別ディレクトリにmainベースのworktree作成
cd ../hotfix-bug
git checkout -b hotfix/bug
# バグ修正作業（元の作業は中断不要！）
```

## 基本コマンド一覧

まずはこの3つを覚えれば日常使いは十分です。

| コマンド | 説明 |
|----------|------|
| `git worktree add <path> <branch>` | 新しいworktreeを作成 |
| `git worktree list` | 全worktreeの一覧表示 |
| `git worktree remove <path>` | worktreeを削除 |

### 全コマンド一覧

```bash
# worktree作成（既存ブランチ）
git worktree add ../feature-dir feature-branch

# worktree作成（新規ブランチも同時に作成）
git worktree add -b new-feature ../new-feature-dir main

# リモートブランチからworktree作成
git worktree add ../review-dir origin/feature-branch

# 一覧表示
git worktree list

# 削除
git worktree remove ../feature-dir

# 不要なメタデータのクリーンアップ
git worktree prune

# worktreeを移動
git worktree move ../old-path ../new-path

# ロック（誤削除防止）
git worktree lock ../feature-dir

# ロック解除
git worktree unlock ../feature-dir

# 壊れた接続を修復
git worktree repair
```

## 実践的なユースケース

### 1. 緊急バグ修正

開発作業を中断せずにhotfix対応ができます。

```bash
# 現在feature-Aで作業中
git worktree add ../hotfix-urgent main
cd ../hotfix-urgent
git checkout -b hotfix/critical-bug

# 修正作業...
git add .
git commit -m "fix: 緊急バグ修正"
git push origin hotfix/critical-bug

# 作業完了後
cd ../main-project
git worktree remove ../hotfix-urgent
```

### 2. PRレビュー

レビュー対象のブランチを別ディレクトリで確認できます。

```bash
git worktree add ../review-pr-123 origin/feature/new-api
cd ../review-pr-123

# ビルド・テスト・動作確認
npm install
npm test
npm run dev

# レビュー完了後
cd ../main-project
git worktree remove ../review-pr-123
```

### 3. AI駆動開発での並列実行

Claude CodeやGitHub Copilotを複数同時に実行する際に威力を発揮します。

```bash
# 複数のworktreeを作成
git worktree add ../project-feature-a -b feature/api-improvement main
git worktree add ../project-feature-b -b feature/ui-update main
git worktree add ../project-feature-c -b feature/test-coverage main

# 各ディレクトリで別々のVSCodeウィンドウを開く
code ../project-feature-a
code ../project-feature-b
code ../project-feature-c

# それぞれでAIエージェントを並列実行！
```

### 4. 過去バージョンの調査

本番環境の問題調査時に役立ちます。

```bash
# タグやコミットからworktree作成
git worktree add ../release-v2.0 v2.0.0
cd ../release-v2.0

# 過去バージョンのコードを調査
# 現在の開発作業は中断不要
```

## メリット・デメリット

### ✅ メリット

| メリット | 説明 |
|----------|------|
| **stash不要** | 作業を中断せず別ブランチで作業可能 |
| **ディスク節約** | cloneより軽量（.gitディレクトリ共有） |
| **環境分離** | node_modules等の依存関係が混ざらない |
| **設定共有** | git config等が全worktreeで共有される |
| **高速** | clone不要で即座にブランチ展開可能 |

### ⚠️ デメリット・注意点

| 注意点 | 説明 |
|--------|------|
| **同一ブランチの重複不可** | 1つのブランチは1つのworktreeでしか使えない |
| **手動削除NG** | `rm -rf`で直接削除すると整合性が壊れる |
| **.gitignore対象ファイル** | .env等は手動コピーが必要 |
| **管理の複雑化** | worktreeが増えすぎると混乱する |

## よくあるトラブルと対処法

### 1. 手動でディレクトリを削除してしまった

```bash
# メタデータが残っている場合
git worktree list
# 削除したはずのworktreeが表示される

# クリーンアップ
git worktree prune
```

### 2. 同じブランチを別のworktreeで使おうとした

```bash
# エラー: fatal: 'feature-a' is already checked out at '/path/to/worktree'

# 対処: 既存のworktreeを確認
git worktree list

# 不要なら削除
git worktree remove /path/to/worktree
```

### 3. worktreeの接続が壊れた

```bash
# メインworktreeを移動してしまった場合等
git worktree repair
```

## VSCodeとの連携

### 標準機能（VSCode 1.103以降）

VSCode 2025年7月版から標準でgit worktree管理機能がサポートされました。

**設定確認**

```json
{
  "git.detectWorktrees": true
}
```

**操作方法**

1. ソース管理ビューを開く
2. 「...」メニュー → 「ワークツリー」
3. 「ワークツリーの追加」「ワークツリーの削除」等を選択

**コマンドパレット**

- `Git: Open Worktree in New Window`
- `Git: Open Worktree in Current Window`
- `Git: Delete Worktree`

### 推奨拡張機能

| 拡張機能名 | 特徴 |
|-----------|------|
| **Git Worktree Manager** | GUIで直感的にworktree管理 |
| **Switch Git Worktree** | ワンクリックでworktree切り替え |
| **Git Graph** | ブランチ構造の可視化（補助的に便利） |

## ベストプラクティス

### ディレクトリ構成パターン

**パターン1: 兄弟ディレクトリ（推奨）**

```
~/repos/
├── my-project/              # メインリポジトリ（main）
├── my-project-feature-a/    # worktree
├── my-project-feature-b/    # worktree
└── my-project-hotfix/       # worktree
```

**パターン2: 専用ディレクトリにまとめる**

```
~/repos/
├── my-project/              # メインリポジトリ
└── my-project.worktrees/    # worktree専用
    ├── feature-a/
    ├── feature-b/
    └── hotfix/
```

:::message alert
**注意**: .git内部にworktreeを作成するのはNGです。Gitのメタデータと混同され、ファイルが削除される可能性があります。
:::

### 運用Tips

1. **命名規則を統一する**
   - 例: `<project>-<branch-type>-<name>`
   - 例: `myapp-feature-login`, `myapp-hotfix-bug123`

2. **定期的にpruneする**

   ```bash
   git worktree prune
   ```

3. **ロック機能を活用する**
   - ネットワークドライブやクラウドストレージ使用時

   ```bash
   git worktree lock ../feature-dir --reason "外付けHDDに保存中"
   ```

4. **不要なworktreeは即削除**
   - 管理が複雑にならないように
   - マージ完了したらすぐ削除

## セットアップスクリプト例

複数のworktreeを一括で作成・削除するスクリプトです。

```bash
#!/bin/bash
# setup-worktrees.sh

PROJECT_NAME="my-project"

echo "並列開発環境のセットアップを開始します..."

for BRANCH in "$@"; do
    TARGET_DIR="../${PROJECT_NAME}-${BRANCH}"
    
    if [ -d "$TARGET_DIR" ]; then
        echo "既にworktreeが存在します: $TARGET_DIR"
    else
        echo "worktreeを作成中: $TARGET_DIR"
        git worktree add "$TARGET_DIR" -b "$BRANCH" main
        
        # 必要に応じて.envファイルをコピー
        if [ -f ".env" ]; then
            cp .env "$TARGET_DIR/.env"
        fi
        
        # 依存関係のインストール
        cd "$TARGET_DIR"
        npm ci
        cd -
    fi
    
    # VSCode起動
    code --new-window "$TARGET_DIR"
done

echo "セットアップが完了しました"
```

**使用例**

```bash
./setup-worktrees.sh feature-api feature-ui feature-test
```

## まとめ

| 項目 | 内容 |
|------|------|
| **推奨度** | ★★★★★ |
| **導入難易度** | 初級〜中級 |
| **必須コマンド** | `add` / `list` / `remove` |
| **特に有効な場面** | 緊急対応、PRレビュー、AI並列実行 |

git worktreeは、特にAI駆動開発が一般化した現在において**必須の技術**となっています。

- stashの煩わしさから解放される
- 複数の作業を安全に並行できる
- VSCodeとの連携も充実している

ぜひ日々の開発に取り入れて、効率的な開発ライフを送りましょう！

## 参考リンク

- [Git公式ドキュメント - git-worktree](https://git-scm.com/docs/git-worktree)
- [Claude Code公式 - 並列セッションのチュートリアル](https://docs.anthropic.com/en/docs/claude-code/tutorials#run-parallel-claude-code-sessions-with-git-worktrees)

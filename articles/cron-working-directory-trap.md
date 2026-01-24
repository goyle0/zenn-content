---
title: "cronで実行したスクリプトのファイルが見つからない？作業ディレクトリの罠と対策"
emoji: "⏰"
type: "tech"
topics: ["linux", "cron", "shell", "bash"]
published: true
---

## はじめに

「手動で実行すると正常に動くのに、cronで実行するとファイルが見つからない...」

こんな経験はありませんか？この記事では、cronの作業ディレクトリに関するよくある問題と、その解決方法を解説します。

## 実際に起きた問題

データベースのバックアップスクリプトをcronで毎日実行していました。ログを見ると「成功」と表示されているのに、期待した場所にバックアップファイルがない...。

調査したところ、ファイルは `/root/` 配下に作成されていました。

## 原因：cronの作業ディレクトリ

### 手動実行時とcron実行時の違い

| 実行方法 | 作業ディレクトリ |
|---------|----------------|
| 手動実行（cdしてから実行） | cdした場所 |
| cron実行 | ユーザーのホームディレクトリ |

cronはスクリプトを実行する際、**デフォルトではユーザーのホームディレクトリ**から実行します。

- rootユーザーのcron → `/root/` から実行
- 一般ユーザーのcron → `/home/ユーザー名/` から実行

### 問題が起きるスクリプトの例

```bash
#!/bin/bash

# ログフォルダ作成（相対パス）
mkdir -p logs

# バックアップ先（相対パス）
mkdir -p backup_data

# ファイル出力（相対パス）
echo "backup data" > backup_data/backup.dat
```

このスクリプトを以下のように実行すると...

**手動実行の場合：**

```bash
cd /usr/local/myapp/
./backup.sh
# → /usr/local/myapp/backup_data/ に作成される ✅
```

**cron実行の場合：**

```bash
# crontab -e
00 0 * * * /usr/local/myapp/backup.sh
# → /root/backup_data/ に作成される ❌
```

## 解決方法

### 方法1：絶対パスを使用する（推奨）

最も確実な方法です。スクリプト内のすべてのパスを絶対パスで記述します。

```bash
#!/bin/bash

# 基準ディレクトリを絶対パスで定義
BASE_DIR="/usr/local/myapp"

# ログフォルダ（絶対パス）
LOG_DIR="${BASE_DIR}/logs"
mkdir -p "$LOG_DIR"

# バックアップ先（絶対パス）
BACKUP_DIR="${BASE_DIR}/backup_data"
mkdir -p "$BACKUP_DIR"

# ファイル出力（絶対パス）
echo "backup data" > "${BACKUP_DIR}/backup.dat"
```

**メリット：**

- どこから実行しても同じ動作になる
- 意図しない場所にファイルが作成されない
- デバッグしやすい

### 方法2：スクリプト内でcdする

スクリプトの先頭で作業ディレクトリを変更します。

```bash
#!/bin/bash

# スクリプトのあるディレクトリに移動
cd "$(dirname "$0")"

# 以降は相対パスでOK
mkdir -p logs
mkdir -p backup_data
```

`$(dirname "$0")` はスクリプト自身が置かれているディレクトリを取得します。

### 方法3：crontabでcdしてから実行

crontabの記述で、cdしてからスクリプトを実行します。

```bash
# crontab -e
00 0 * * * cd /usr/local/myapp && ./backup.sh
```

**注意：** この方法は、スクリプト自体は修正せずに済みますが、crontabの記述を忘れると問題が再発します。

## 確認方法

### 現在の作業ディレクトリを確認するスクリプト

cronの動作を確認したい場合、以下のスクリプトで作業ディレクトリを確認できます。

```bash
#!/bin/bash

# 作業ディレクトリを確認
echo "現在の作業ディレクトリ: $(pwd)"
echo "スクリプトの場所: $(dirname "$0")"
echo "実行ユーザー: $(whoami)"
echo "実行時刻: $(date)"
```

これをcronで実行し、出力を確認してみてください。

```bash
# crontab -e
* * * * * /path/to/check_dir.sh >> /tmp/cron_check.log 2>&1
```

### 意図しない場所にファイルがないか確認

問題が発生している場合、以下のコマンドで確認できます。

```bash
# rootユーザーのホームディレクトリを確認
ls -la /root/

# 最近作成されたファイルを検索
find /root -type f -mtime -1 -ls
```

## cronで実行する際のベストプラクティス

### 1. 絶対パスを徹底する

```bash
# ❌ 悪い例
LOG_FILE="logs/app.log"
OUTPUT_DIR="output"

# ✅ 良い例
BASE_DIR="/usr/local/myapp"
LOG_FILE="${BASE_DIR}/logs/app.log"
OUTPUT_DIR="${BASE_DIR}/output"
```

### 2. コマンドも絶対パスで指定

cronでは環境変数PATHが限定的です。コマンドも絶対パスで指定すると安全です。

```bash
# ❌ 悪い例
mysql -u root -p password dbname

# ✅ 良い例
/usr/bin/mysql -u root -p password dbname
```

コマンドのパスは `which` で確認できます。

```bash
which mysql
# /usr/bin/mysql
```

### 3. 環境変数を明示的に設定

必要な環境変数はスクリプト内で明示的に設定します。

```bash
#!/bin/bash

# 環境変数を明示的に設定
export PATH="/usr/local/bin:/usr/bin:/bin"
export LANG="ja_JP.UTF-8"
```

### 4. ログを残す

問題が発生したときに調査できるよう、ログを残しましょう。

```bash
#!/bin/bash

LOG_FILE="/var/log/myapp/backup.log"

echo "[$(date '+%F %T')] バックアップ開始" >> "$LOG_FILE"
# 処理...
echo "[$(date '+%F %T')] バックアップ完了" >> "$LOG_FILE"
```

## まとめ

| 問題 | 原因 | 解決策 |
|-----|------|-------|
| ファイルが見つからない | cronはホームディレクトリから実行される | 絶対パスを使用する |
| コマンドが見つからない | cronのPATHが限定的 | コマンドを絶対パスで指定 |
| 文字化けする | 環境変数が設定されていない | LANGを明示的に設定 |

**重要なポイント：**

1. cronで実行されるスクリプトは**絶対パス**を使う
2. 手動で動いてもcronで動くとは限らない
3. 問題が起きたらホームディレクトリを確認する

この記事が、同じ問題で悩んでいる方の参考になれば幸いです。

## 参考

- [crontab(5) - Linux manual page](https://man7.org/linux/man-pages/man5/crontab.5.html)
- [Cron - ArchWiki](https://wiki.archlinux.org/title/Cron)

<!-- Updated: 2026-01-24 -->

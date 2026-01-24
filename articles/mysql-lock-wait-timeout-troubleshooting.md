---
title: "MySQL「Lock wait timeout exceeded」エラーの調査・解消手順"
emoji: "🔐"
type: "tech"
topics: ["mysql", "database", "innodb", "troubleshooting"]
published: true
---

## はじめに

この記事では、MySQLで発生する「Lock wait timeout exceeded」エラーの原因調査から解消までの手順を解説します。

### 対象読者

- データベース管理者
- アプリケーション運用担当者
- インフラエンジニア

### 動作確認環境

- MySQL 5.6 / 5.7 / 8.0
- MariaDB 10.x

## エラー概要

### エラーメッセージ

```
ERROR: Lock wait timeout exceeded; try restarting transaction
```

### 症状

- 特定の画面やAPIがタイムアウトする
- 一部の機能だけ動作しない
- アプリケーション全体ではなく、特定の処理のみ影響を受ける

### 原因

他のトランザクションがテーブルやレコードをロックしており、一定時間待っても解放されないために発生します。

主な原因：

| 原因 | 説明 |
|------|------|
| 長時間トランザクション | 開いたまま放置されたトランザクション |
| デッドロック | 複数のトランザクションが互いにロックを待っている状態 |
| 大量データ処理 | 大量のINSERT/UPDATE/DELETEが実行中 |
| 接続の異常終了 | アプリケーションがトランザクションを閉じずに切断 |

## 調査手順

### Step 1: 現在実行中のプロセス確認

長時間実行されているクエリがあるか確認します。

```sql
SELECT 
    id,
    user,
    host,
    db,
    command,
    time,
    state,
    LEFT(info, 100) AS query_preview
FROM information_schema.processlist
WHERE command != 'Sleep'
  AND time > 30
ORDER BY time DESC;
```

#### 確認ポイント

| カラム | 説明 |
|--------|------|
| id | プロセスID（KILLする際に使用） |
| user | 接続ユーザー |
| host | 接続元ホスト |
| db | 使用中のデータベース |
| command | 現在の状態（Query, Sleep など） |
| time | 経過時間（秒） |
| state | 詳細な状態 |
| query_preview | 実行中のクエリ（先頭100文字） |

#### 結果の見方

- **結果が空** → 長時間実行中のクエリはない（Step 2へ進む）
- **結果がある** → そのクエリがロックの原因の可能性あり

### Step 2: ロック待ち状況の確認

現在ロック待ちが発生しているか確認します。

#### MySQL 5.6 の場合

```sql
SELECT 
    r.trx_id AS waiting_trx_id,
    r.trx_mysql_thread_id AS waiting_thread,
    r.trx_query AS waiting_query,
    b.trx_id AS blocking_trx_id,
    b.trx_mysql_thread_id AS blocking_thread,
    b.trx_query AS blocking_query,
    b.trx_started AS blocking_started,
    TIMESTAMPDIFF(SECOND, b.trx_started, NOW()) AS blocking_duration_sec
FROM information_schema.innodb_lock_waits w
JOIN information_schema.innodb_trx b ON b.trx_id = w.blocking_trx_id
JOIN information_schema.innodb_trx r ON r.trx_id = w.requesting_trx_id;
```

#### MySQL 5.7以降 / MySQL 8.0 の場合

```sql
SELECT 
    r.trx_id AS waiting_trx_id,
    r.trx_mysql_thread_id AS waiting_thread,
    r.trx_query AS waiting_query,
    b.trx_id AS blocking_trx_id,
    b.trx_mysql_thread_id AS blocking_thread,
    b.trx_query AS blocking_query,
    b.trx_started AS blocking_started,
    TIMESTAMPDIFF(SECOND, b.trx_started, NOW()) AS blocking_duration_sec
FROM performance_schema.data_lock_waits w
JOIN information_schema.innodb_trx b ON b.trx_id = w.BLOCKING_ENGINE_TRANSACTION_ID
JOIN information_schema.innodb_trx r ON r.trx_id = w.REQUESTING_ENGINE_TRANSACTION_ID;
```

#### 確認ポイント

| カラム | 説明 |
|--------|------|
| waiting_thread | ロック待ちしているスレッドID |
| waiting_query | ロック待ちしているクエリ |
| blocking_thread | ロックを保持しているスレッドID |
| blocking_query | ロックを保持しているクエリ |
| blocking_duration_sec | ロック保持時間（秒） |

#### 結果の見方

- **結果が空** → 現在ロック待ちは発生していない（Step 3へ進む）
- **結果がある** → `blocking_thread` がロックの原因

### Step 3: 長時間放置トランザクションの確認

トランザクションが開いたまま放置されていないか確認します。

```sql
SELECT 
    trx_id,
    trx_state,
    trx_started,
    TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS duration_sec,
    trx_mysql_thread_id,
    trx_query
FROM information_schema.innodb_trx
WHERE TIMESTAMPDIFF(SECOND, trx_started, NOW()) > 60
ORDER BY trx_started;
```

#### 確認ポイント

| カラム | 説明 |
|--------|------|
| trx_id | トランザクションID |
| trx_state | 状態（RUNNING, LOCK WAIT など） |
| trx_started | 開始時刻 |
| duration_sec | 経過時間（秒） |
| trx_mysql_thread_id | スレッドID（KILLする際に使用） |
| trx_query | 実行中のクエリ（NULLの場合あり） |

#### 結果の見方

- **trx_query が NULL** → クエリは終了しているが、トランザクションが閉じられていない（放置状態）
- **duration_sec が大きい** → 長時間放置されている可能性が高い

### Step 4: 対象プロセスの詳細確認

Step 3で見つかったスレッドIDの詳細を確認します。

```sql
SELECT 
    id,
    user,
    host,
    db,
    command,
    time,
    state
FROM information_schema.processlist 
WHERE id IN (スレッドIDをカンマ区切りで指定);
```

#### 実行例

```sql
SELECT 
    id,
    user,
    host,
    db,
    command,
    time,
    state
FROM information_schema.processlist 
WHERE id IN (20800, 21041, 21349);
```

#### KILLして良いかの判断基準

| 条件 | 判断 |
|------|------|
| command = Sleep かつ time が数時間 | ✅ KILLしてOK |
| command = Sleep かつ trx_query = NULL | ✅ KILLしてOK |
| command = Query かつ重要な処理中 | ⚠️ 慎重に判断 |
| 業務時間外に開始されたトランザクション | ✅ KILLしてOK |

## 解消手順

### Step 5: プロセスのKILL

問題のプロセスを強制終了します。

```sql
KILL スレッドID;
```

#### 実行例

```sql
KILL 20800;
KILL 21041;
KILL 21349;
```

#### 注意事項

- **KILLすると未コミットの変更はロールバック（取り消し）されます**
- Sleep状態のプロセスは何も処理していないため、KILLしても実行中の処理が中断されることはありません

### Step 6: 解消確認

#### 確認1: トランザクションが解消されたか

```sql
SELECT COUNT(*) 
FROM information_schema.innodb_trx 
WHERE TIMESTAMPDIFF(SECOND, trx_started, NOW()) > 60;
```

結果が **0** であれば、長時間トランザクションは解消されています。

#### 確認2: ロック待ちが解消されたか

```sql
SELECT COUNT(*) 
FROM information_schema.innodb_lock_waits;
```

※ MySQL 8.0の場合は `performance_schema.data_lock_waits` を使用

結果が **0** であれば、ロック待ちは解消されています。

#### 確認3: アプリケーションの動作確認

問題が発生していた画面・機能が正常に動作するか確認します。

## 応急処置：タイムアウト時間の延長

根本解決ではありませんが、タイムアウト時間を延長することで一時的に回避できる場合があります。

### 現在の設定値を確認

```sql
SHOW VARIABLES LIKE 'innodb_lock_wait_timeout';
```

### 一時的に変更（セッション単位）

```sql
SET innodb_lock_wait_timeout = 120;
```

### 恒久的に変更（my.cnf / my.ini）

```ini
[mysqld]
innodb_lock_wait_timeout = 120
```

※ 変更後はMySQLの再起動が必要です

### 推奨値

| 環境 | 推奨値 |
|------|--------|
| デフォルト | 50秒 |
| 一般的なWebアプリケーション | 50〜120秒 |
| バッチ処理が多い環境 | 120〜300秒 |

:::message alert
タイムアウト時間を延長しても根本解決にはなりません。繰り返し発生する場合は、アプリケーション側の改善が必要です。
:::

## クイックリファレンス

コピペで使えるクエリ集です。

```sql
-- 1. 長時間トランザクション確認
SELECT 
    trx_id,
    trx_state,
    trx_started,
    TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS duration_sec,
    trx_mysql_thread_id,
    trx_query
FROM information_schema.innodb_trx
WHERE TIMESTAMPDIFF(SECOND, trx_started, NOW()) > 60
ORDER BY trx_started;

-- 2. 対象プロセス詳細確認
SELECT 
    id,
    user,
    host,
    db,
    command,
    time,
    state
FROM information_schema.processlist 
WHERE id IN (ここにスレッドIDを指定);

-- 3. KILL実行
KILL スレッドID;

-- 4. 解消確認
SELECT COUNT(*) 
FROM information_schema.innodb_trx 
WHERE TIMESTAMPDIFF(SECOND, trx_started, NOW()) > 60;
```

## 再発防止策

同じ問題が繰り返し発生する場合は、以下の対策を検討してください。

### アプリケーション側

| 対策 | 説明 |
|------|------|
| トランザクションの適切なクローズ | try-finally や using を使って確実にクローズ |
| トランザクション時間の短縮 | 長時間のトランザクションを分割 |
| タイムアウト設定 | アプリケーション側でもタイムアウトを設定 |

### データベース側

| 対策 | 説明 |
|------|------|
| wait_timeout の設定 | アイドル接続の自動切断時間を設定 |
| interactive_timeout の設定 | 対話型接続のタイムアウト設定 |
| 監視の導入 | 長時間トランザクションを検知してアラート |

### 監視用クエリ（定期実行推奨）

```sql
-- 60秒以上のトランザクションがあればアラート
SELECT 
    COUNT(*) AS long_running_trx_count
FROM information_schema.innodb_trx
WHERE TIMESTAMPDIFF(SECOND, trx_started, NOW()) > 60;
```

## 用語解説

| 用語 | 説明 |
|------|------|
| トランザクション | 一連のデータベース操作をまとめた単位。コミットまたはロールバックで終了 |
| ロック | 他のトランザクションがデータを変更できないようにする仕組み |
| デッドロック | 複数のトランザクションが互いにロック解放を待っている状態 |
| Sleep | 接続は維持されているが、クエリを実行していない待機状態 |
| KILL | プロセスを強制終了するコマンド |
| ロールバック | トランザクション内の変更を取り消すこと |
| コミット | トランザクション内の変更を確定すること |

## まとめ

1. **Step 1-3**: 原因となっているプロセス・トランザクションを特定
2. **Step 4**: KILLして良いか判断
3. **Step 5**: 問題のプロセスをKILL
4. **Step 6**: 解消を確認

Sleep状態で長時間放置されたトランザクションが原因の場合は、KILLすることで即座に解消できます。

繰り返し発生する場合は、アプリケーション側でトランザクション管理を見直すことをお勧めします。

## 参考リンク

- [MySQL公式ドキュメント - InnoDB Locking](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)
- [MySQL公式ドキュメント - innodb_lock_wait_timeout](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_lock_wait_timeout)

<!-- Updated: 2026-01-24 -->

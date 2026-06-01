---
title: "MySQLのリストアが終わらない…原因は「メタデータロック待ち」だった"
emoji: "🔒"
type: "tech"
topics: ["mysql", "aws", "rds", "database", "dba"]
published: true
---

## TL;DR（結論）

- `mysql < dump.sql` でのリストアが**いつまでも終わらない**とき、「遅い」のではなく **別の接続にロックされて止まっている**ことがある。
- 犯人は **長時間 `Sleep`（放置）のまま、対象テーブルのロックを握り続けている接続**。
- MySQL のロック待ちは初期設定だと**最長1年（`lock_wait_timeout = 31536000`秒）**待つので、放置しても永久に終わらない。
- `sys.schema_table_lock_waits` で犯人(`blocking_pid`)を特定し、`KILL` すれば解消する。
- ⚠️ ただし **`KILL` する前に「それがリストアの接続そのものではないか」を必ず確認する**こと。

---

## 何が起きたか（症状）

ダンプファイルを流し込むだけの、ごく普通のリストアスクリプトを実行した。

```sh
#!/bin/sh
mysql -h <DBホスト> -P 3306 -u root -p<パスワード> target_db < dump.sql
```

- 他のデータベースは **4分ほどで完了**するのに、ある DB だけ **20分以上経っても終わらない**。
- ターミナルは固まったまま、プロンプトも返ってこない。
- GUIツール（HeidiSQL など）でテーブル容量を見ても、データが増えていないように見える。

「データ量が多いから遅いだけ？」と思いがちだが、実際は **完全に止まっていた**。

:::message
**注意：GUIツールに表示されるテーブル容量は、進捗の判断に使えない**
多くのGUIは `information_schema` の統計値（少し前の状態をキャッシュした概算値）を表示する。リストア中はリアルタイムに更新されないため、「容量が増えない＝止まっている」とは判断できない。進捗は後述の `PROCESSLIST` で確認する。
:::

---

## なぜ「遅い」ではなく「止まっている」のか

リストア（ダンプの流し込み）は、テーブルごとに `DROP TABLE` → `CREATE TABLE` → `INSERT …` を実行する。
このうち **`DROP` / `CREATE` / `ALTER` などのDDLは、対象テーブルの排他的なメタデータロック（MDL）を必要とする**。

もし**他の接続が同じテーブルのロックを握っている**と、リストア側はロックが解放されるまで待ち続ける。
そして MySQL の `lock_wait_timeout` は**初期値が `31536000`秒 ≒ 1年**。つまり、ロックが解放されない限り**ほぼ永久にハングする**。

```sql
-- 確認（初期値は 31536000 = 1年）
SHOW VARIABLES LIKE 'lock_wait_timeout';
```

---

## 調査手順（ステップバイステップ）

すべて、リストアを実行している**別のセッション**から実行する（リストアのターミナルは触らない）。

### 1. 「動いている接続」だけを確認する

`SHOW FULL PROCESSLIST` は接続が多いと埋もれるので、`Sleep`（待機）を除外して絞り込む。

```sql
SELECT id, user, host, db, command, time, state, LEFT(info, 200) AS info
FROM information_schema.processlist
WHERE command <> 'Sleep'
ORDER BY time DESC;
```

リストアの接続が `state = 'Waiting for table metadata lock'` になっていれば、**ロック待ちで止まっている**ことが確定する。

### 2. ロック待ちの「関係」を特定する

`sys.schema_table_lock_waits` を使うと、「誰が待たされ（waiting）、誰がロックを握っているか（blocking）」が一発で分かる。

```sql
SELECT object_schema, object_name,
       waiting_pid, blocking_pid, blocking_account,
       sql_kill_blocking_connection
FROM sys.schema_table_lock_waits
WHERE object_schema = 'target_db'\G
```

出力例：

```
*************************** 1. row ***************************
              object_schema: target_db
                object_name: some_table
                waiting_pid: 3001000
               blocking_pid: 12345
           blocking_account: root@10.0.0.10
sql_kill_blocking_connection: KILL 12345
```

`sql_kill_blocking_connection` カラムには、**そのまま実行できる `KILL` 文**が入っている。

:::message
`sys.schema_table_lock_waits` を使うには `performance_schema` が有効で、メタデータロックの計測（`wait/lock/metadata/sql/mdl`）が有効になっている必要がある。多くの環境（AWS RDS含む）では既定で利用可能。
:::

### 3. ★最重要：犯人(`blocking_pid`)が何者かを確認する

ここが今回いちばん伝えたいポイント。
`blocking_account` を見ると、**犯人がリストア実行サーバーと同じホスト・同じユーザーから来ている**ことがある（例：両方とも `root@10.0.0.10`）。

そのため、`blocking_pid` が **リストアの接続そのもの**である可能性を排除してから `KILL` する必要がある。
これを間違えると、**リストア本体を強制終了**してしまい、それまでの作業がすべて無駄になる。

```sql
SELECT id, user, host, db, command, time, state, LEFT(info, 200) AS info
FROM information_schema.processlist
WHERE id = 12345\G
```

出力例：

```
     id: 12345
   user: root
   host: 10.0.0.10:53008
     db: target_db
command: Sleep      <-- 何も実行していない
   time: 11665      <-- 約3時間、放置されている
  state:
   info: NULL       <-- 実行中のSQLなし
```

判定基準：

| `command` の値 | 意味 | 対応 |
| --- | --- | --- |
| `Sleep`（`info` が `NULL`） | 接続を残したまま放置され、ロックだけ握っている | **これはリストアではない → `KILL` してよい** |
| `Query`（`info` に `INSERT` / `CREATE` / `ALTER` / `DROP`） | リストア本体が実際に処理している可能性が高い | **`KILL` してはいけない**（遅いだけ。待てば終わる） |

:::message alert
**`KILL` する前のチェックリスト**
- `blocking_pid` の `command` が `Sleep` か？（`Sleep` = 放置接続なので切ってOK）
- `time` が大きいか？（数時間放置されている接続は、まず犯人で間違いない）
- 逆に `command = Query` で `INSERT`/`DDL` を実行中なら、それは**リストア本体かもしれない**ので切らない。
:::

### 4. 犯人の接続を切る

`Sleep` で放置されている接続だと確認できたら、`KILL` する。
`KILL` は対象の接続を切断し、未コミットのトランザクションをロールバックして、握っていたロックをすべて解放する。

```sql
KILL 12345;
-- Query OK, 0 rows affected
```

### 5. 解消を確認する

```sql
SELECT * FROM sys.schema_table_lock_waits\G
-- Empty set ならロック解消
```

`Empty set` が出れば、ロック待ちは解消。
止まっていたリストアが動き出し、`End to restore database.`（＝スクリプトの完了ログ）が出れば完了。

---

## なぜこんな接続が生まれたのか

`command = Sleep` で `time` が数時間という接続は、典型的には次のようなもの。

- 過去のセッション（手動の `mysql` クライアントや GUI ツール）を**閉じ忘れ、トランザクションが開いたまま放置**された。
- アプリケーションの**コネクションプールの接続**が、トランザクションを抱えたままアイドル状態になっていた。

これらが対象テーブルのメタデータロックを握り続け、後から走ったリストアのDDLをブロックしていた、という流れ。

---

## 再発防止・予防策

- **リストア前にアプリを止める／メンテナンスモードにする。** 対象DBを使う接続をなくしてから流し込むのが最も確実。
- **`lock_wait_timeout` を短く設定**しておくと、永久ハングではなく**早めにエラーで失敗**してくれるので、異常に気づきやすい。
  ```sql
  -- そのセッションだけ短くする例
  SET SESSION lock_wait_timeout = 60;
  ```
- **アイドル接続を放置しない。** `wait_timeout` / `interactive_timeout` を適切に設定し、長時間アイドルの接続を自動で切る。
- **GUIの容量表示で進捗を判断しない。** 進捗は `PROCESSLIST` で確認する。

---

## まとめ

- リストアが終わらない＝遅い、とは限らない。**メタデータロック待ちで止まっている**ケースを疑う。
- `sys.schema_table_lock_waits` を使えば、犯人(`blocking_pid`)と `KILL` 文を一発で取得できる。
- **`KILL` する前に、それがリストア本体でないこと（`command = Sleep` であること）を必ず確認**する。
- 長時間 `Sleep` のままロックを握る放置接続が、地味によくある原因。

「リストアが固まったら、まずロック待ちを疑う」——これを覚えておくと、原因究明がぐっと速くなる。

---

## 参考

- MySQL Reference Manual: Metadata Locking
- MySQL Reference Manual: `lock_wait_timeout`
- MySQL sys Schema: `schema_table_lock_waits`

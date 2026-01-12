---
title: "Redisの再起動とデータをすべてクリアする方法"
emoji: "🗄️"
type: "tech"
topics: ["redis", "linux", "shell", "備忘録"]
published: true
---

## はじめに

Redisを使っていると、サービスを再起動したり、保存されているデータをすべて削除したい場面があります。

この記事では、**Redisの再起動方法**と**データを全削除する方法**を紹介します。

---

## Redisを再起動する

以下のコマンドでRedisサービスを再起動できます。

```bash
sudo systemctl restart redis6
```

### コマンドの意味

| 部分 | 説明 |
|------|------|
| `sudo` | 管理者権限で実行する |
| `systemctl` | サービスを管理するコマンド |
| `restart` | サービスを再起動する |
| `redis6` | Redis 6のサービス名 |

:::message
サービス名は環境によって異なる場合があります（`redis`、`redis-server`など）。
:::

---

## Redisのデータをすべてクリアする

Redisに保存されているデータをすべて削除する手順です。

### 手順

#### 1. Redisにアクセスする

```bash
redis6-cli
```

Redisの操作画面（redis-cli）に入ります。

#### 2. すべてのデータを削除する

```bash
FLUSHALL
```

Redisに保存されている**すべてのデータベースのデータ**が削除されます。

:::message alert
この操作は元に戻せません。実行前に本当に削除してよいか確認してください。
:::

#### 3. redis-cliを終了する

```bash
exit
```

redis-cliの操作画面を終了して、通常のターミナルに戻ります。

---

## コマンドまとめ

| コマンド | 説明 |
|---------|------|
| `sudo systemctl restart redis6` | Redisを再起動する |
| `redis6-cli` | Redisの操作画面に入る |
| `FLUSHALL` | すべてのデータを削除する |
| `exit` | redis-cliを終了する |

---

## 補足：FLUSHALLとFLUSHDBの違い

| コマンド | 説明 |
|---------|------|
| `FLUSHALL` | **すべてのデータベース**のデータを削除 |
| `FLUSHDB` | **現在選択中のデータベース**のデータのみ削除 |

特定のデータベースだけクリアしたい場合は`FLUSHDB`を使いましょう。

---

## まとめ

- Redisの再起動は `sudo systemctl restart redis6`
- データの全削除は `redis6-cli` → `FLUSHALL` → `exit`
- `FLUSHALL`は元に戻せないので注意

開発環境でキャッシュをクリアしたいときなどに活用してください。

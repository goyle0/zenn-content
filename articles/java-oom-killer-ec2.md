---
title: "Javaプロセスが早朝に突然消えた──犯人はOOM Killerだった話"
emoji: "💥"
type: "tech"
topics: ["aws", "ec2", "linux", "java", "トラブルシューティング"]
published: true
---

## はじめに

ある朝、本番サーバーの Java プロセス（Tomcat）が**跡形もなく消えていました**。

- エラーログは一切なし
- 終了処理のログもなし
- 誰も再起動していない
- 約 9 時間、誰も気づいていない

「突然消えた」以外に何も分からない状態からスタートして、最終的に**Linux の OOM Killer による強制終了**であることを確定させ、対策まで実施した記録です。

同じような「Java が理由もなく落ちた」に遭遇した方の役に立てば幸いです。

:::message
この記事は実際のトラブル対応をもとにしていますが、サーバー名・IP アドレス・システム名・ファイルパスはすべて架空のものに置き換えています。
:::

### 対象読者

- AWS EC2 上で Java（Tomcat）アプリを運用している方
- 「プロセスが理由もなく落ちた」に心当たりのある方
- Linux のメモリまわりの調査手順を知りたい方

### 環境

| 項目 | 内容 |
|---|---|
| インスタンス | EC2 `t3.medium`（2 vCPU / メモリ 4GB） |
| OS | Amazon Linux 2（kernel 4.14 系） |
| アプリ | Tomcat 上の Java Web アプリ（ヒープ上限 1024MB） |
| DB | 同一サーバー上に MySQL が同居 |
| スワップ | **なし（0B）** |

---

## 結論を先に

> **OS のメモリが枯渇し、Linux の OOM Killer が Java プロセスを強制終了させた。**

内訳はこうでした。

| プロセス | 使用メモリ（RSS） | 備考 |
|---|---:|---|
| **java（Tomcat）** | **2.12 GB** | 最大消費者。これが殺された |
| **mysqld** | **1.06 GB** | 同居 DB |
| **yum** | **0.35 GB** | ← 最後の一押し |
| 監視エージェント | 0.03 GB | |
| **合計** | **約 3.5 GB** | 空きメモリは **84MB** まで枯渇 |

4GB のサーバーに Java 2.1GB + MySQL 1.1GB が常駐していたところへ、**cron 経由の yum 自動更新チェック（0.35GB）が加わって満杯**になりました。しかも**スワップが 0** なので逃げ場がなく、その場で即 Kill されています。

**アプリのバグでもなく、Java のヒープ不足でもなく、サーバーのメモリ構成の問題**でした。

---

## OOM Killer とは（ざっくり）

Linux には「メモリが本当に足りなくなったとき、いちばんメモリを食っているプロセスを強制終了して、システム全体の停止を防ぐ」という仕組みがあります。これが **OOM Killer（Out Of Memory Killer）** です。

イメージとしては、**沈みかけた船から一番重い荷物を海に投げ捨てる**ようなものです。船（サーバー）は助かりますが、投げ捨てられた荷物（Java プロセス）にとってはたまったものではありません。

ここで重要なのが、**OOM Killer は `kill -9` 相当の強制終了を行う**という点です。プロセスには「終了しろ」という通知すら届きません。だから──

**アプリ側のログには、何ひとつ残りません。**

これが「エラーもなく突然消えた」の正体です。

---

## 調査の流れ

### Step 1. アプリログを見る → 何も分からない

まず手元にあった 3 つのログを確認しました。

| ログ | 結果 |
|---|---|
| `catalina.out` | 再起動後の 37 行だけ。停止前の記録なし |
| `gc.log` | 再起動後のもののみ。停止直前の挙動は追跡不能 |
| `app.log`（アプリログ） | **停止時刻の直前でぷつっと途切れている**。終了処理のログは 1 行もなし |

ここで分かったのは「**正常停止ではない**」ということだけです。

正常に停止した場合は必ず終了処理のログが出ます。それが 1 行もないということは、**外部から強制的に殺された**可能性が高い。

:::message alert
`gc.log` は起動オプションが `-Xloggc:/opt/app/logs/gc.log` だけだと、**JVM が起動するたびに先頭から上書き**されます。ローテーション指定（`-XX:+UseGCLogFileRotation` など）がないと、再起動した瞬間に障害直前の記録が消えます。
:::

### Step 2. 停止時刻を 1 分単位で特定する

アプリログには 1 分ごとに動く定例タスクのログが出ていました。これが決め手になります。

```
01:50:00  定例タスク実行
01:51:00  定例タスク実行   ← 最後に出た行
（01:52:00 の行が無い）
```

つまり **01:51:00 〜 01:52:00 の間に落ちた**と断定できます。

:::message
1 分周期など定期的に出るログは、障害調査で「プロセスが生きていた最後の時刻」を高精度で特定するのに非常に役立ちます。
:::

### Step 3. `gc.log` の 1 行目でアタリをつける

`gc.log` の冒頭には、JVM が起動時に見たサーバーのメモリ構成がそのまま出ます。

```
Memory: 4k page, physical 3908124k(1502416k free), swap 0k(0k free)
```

**physical 約 3.8GB、swap 0k**。

この 1 行だけで「4GB でスワップなし」が分かるので、**サーバーに入る前の段階で OOM Kill 説をほぼ絞り込めました**。

### Step 4. `journalctl -k` で確定させる ← ここが本命

アプリログをいくら読んでも分からないので、**カーネルのログ**を見ます。

```bash
sudo journalctl -k --since "YYYY-MM-DD 01:40" --until "YYYY-MM-DD 02:05" \
  | grep -i -A 120 "invoked oom-killer"
```

結果:

```
kernel: Out of memory: Kill process 18802 (java) score 542 or sacrifice child
kernel: Killed process 18802 (java) total-vm:3828848kB, anon-rss:2218100kB
kernel: Free swap  = 0kB / Total swap = 0kB
```

**確定です。**

`journalctl -k` はカーネル自身が書くログなので、アプリが強制終了させられても確実に残ります。**「アプリログには何も残らない」障害における、唯一の決定的証拠**です。

### Step 5. 引き金を特定する

「なぜ**その時刻**だったのか」も追いました。

```bash
sudo journalctl --since "YYYY-MM-DD 01:40" --until "YYYY-MM-DD 02:05" \
  | grep -vE "dhclient|postfix|XMT|ssm-agent"
```

```
01:51:01 CROND[xxxxx]: (root) CMD (systemctl --quiet restart update-motd)
```

Amazon Linux には、SSH ログイン時に表示される「利用可能なアップデートがあります」というメッセージを作るための仕組み（`update-motd`）があります。これが内部で **yum を起動**し、そのとき **0.35GB** を確保しに来たのが最後の一押しでした。

ギリギリで綱渡りしていたサーバーに、定期実行のアップデートチェックがとどめを刺した形です。

### Step 6. 他の可能性を潰す

| 疑い | 判定 | 根拠 |
|---|:-:|---|
| Java のヒープ不足（OutOfMemoryError） | ✗ | アプリログに OOME の記録なし。ヒープ上限 1024MB に対し最大 654MB で余裕あり、Full GC もゼロ |
| JVM 自体のクラッシュ | ✗ | `find / -name "hs_err_pid*.log"` が **0 件**（クラッシュレポートが生成されていない） |
| 正常停止・計画メンテナンス | ✗ | 終了処理ログなし。停止前後に手動ログイン記録なし（`last` で確認） |
| 計画再起動の時刻とかぶった | ✗ | 日付・時刻ともに一致しない |
| アプリのエラー | ✗ | ERROR は再起動後の軽微な設定ファイル未配置が 2 件のみ |

---

## 調査コマンドまとめ（コピペ用）

```bash
# ① 本命：OOM Killer の記録（これが出れば確定）
sudo journalctl -k --since "YYYY-MM-DD HH:MM" --until "YYYY-MM-DD HH:MM" \
  | grep -i -A 120 "invoked oom-killer"
#   → 末尾の「Total swap」とプロセス一覧まで読むこと

# ② 前後にサーバー側で何が動いていたか（引き金の特定）
sudo journalctl --since "YYYY-MM-DD HH:MM" --until "YYYY-MM-DD HH:MM" \
  | grep -vE "dhclient|postfix|XMT|ssm-agent"

# ③ 現在のメモリ・スワップ・上位プロセス
free -h && swapon --show && ps aux --sort=-rss | head -10

# ④ 手動再起動だったかの確認
last | head -20

# ⑤ JVM 自身のクラッシュ痕跡（0 件なら OOM Kill 説を補強）
find / -name "hs_err_pid*.log" -newermt "YYYY-MM-DD" 2>/dev/null

# ⑥ 過去ログの残存状況（日次ローテーションされていれば過去分が残る）
sudo ls -l /opt/app/logs/ | tail -20
```

---

## ハマりどころ：ログの読み間違い

OOM Killer のログは、先頭がこうなっていました。

```
kernel: monitoring-agent invoked oom-killer: gfp_mask=0x14200ca, order=0, oom_score_adj=0
```

ここだけ見ると「監視エージェントが犯人か？」と思ってしまいますが、**違います**。

- **invoked** = メモリを要求して、失敗した側。**たまたま最後にメモリを要求しただけ**
- **Killed** = 実際に殺された側。**いちばんメモリを食っていたプロセス**

今回、`invoked` は監視エージェント（0.03GB）ですが、`Killed` は Java（2.12GB）です。

満員のエレベーターに最後に乗ろうとした人がブザーを鳴らしただけで、実際に降ろされるのは一番重い荷物を持った人、という関係です。**必ずログの最後まで読んで `Killed process` の行を確認してください。**

---

## 対策

### ① スワップを 2GB 追加した（当日実施）

まず即効性のある応急処置として、スワップ領域を作りました。サービスは無停止で実施できます。

```bash
# 2GB のスワップファイルを作成
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# 再起動後も有効にする
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# 確認
free -h && swapon --show
```

実施後:

```
Mem:   3.8G  (used 2.1G / free 1.1G)
Swap:  2.0G  (used 0B  / free 2.0G)   ✅
```

これで、**メモリが逼迫しても「強制終了」ではなく「一時的に遅くなる」で済む**ようになります。OOM Kill による突然死は実質的に防げます。

### ② swappiness を下げる

スワップを入れると、平常時から不要にディスクへ退避してしまい遅くなることがあります。「本当に苦しいときだけ使う」設定にしておきます。

```bash
echo 'vm.swappiness = 10' | sudo tee /etc/sysctl.d/99-swappiness.conf
sudo sysctl -p /etc/sysctl.d/99-swappiness.conf
```

:::message alert
**スワップはあくまで延命策です。** Java が 2.1GB まで肥大すること自体は解消していません。根本的にはメモリ増強（`t3.large` 等）やアプリ側の見直しが必要です。
:::

### ③ プロセス消失を検知する監視を入れる

今回いちばん怖かったのは、落ちたこと自体より **9 時間誰も気づかなかった**ことです。

CloudWatch エージェントの `procstat` で Java プロセスの数を取り、0 になったらアラームを出すようにしました。

```json
{
  "metrics": {
    "metrics_collected": {
      "procstat": [
        { "exe": "java", "measurement": ["pid_count"] }
      ],
      "mem": { "measurement": ["mem_used_percent"] }
    }
  }
}
```

アラームの設定ポイント:

| 項目 | 設定 | 理由 |
|---|---|---|
| 統計 | **Maximum** | Average だと瞬間的な取りこぼしで誤発報しやすい |
| 評価 | **5 分 × 2 回連続** | 一時的なブレで鳴らないように |
| 欠落データ | **breaching（不足を異常扱い）** | ← **重要**。下記参照 |

---

## いちばんの学び：「監視しているつもり」がいちばん怖い

アラームを作った直後、**Java は正常に動いているのにアラームが即 `ALARM` になりました**。

調べると、メトリクス `procstat_lookup_pid_count` が**ずっと `0` を返し続けていた**のです。

**原因は、CloudWatch エージェントの実行ユーザーでした。**

- Java は **root** で動いている
- エージェントは **`cwagent`（一般ユーザー）** で動いていた
- プロセス検索方式 `pid_finder = "native"` は `/proc/<pid>/exe` を直接読みに行くが、これは**所有者か root しか読めない**
- 結果、**権限不足で「java は 0 個です」と報告し続けていた**

設定はここにあります（`common-config.toml` ではありません）。

```json
{
  "agent": { "run_as_user": "cwagent" }   ← これを "root" に
}
```

修正してエージェントを再起動したところ、値が `1.0` に復帰し、アラームも `OK` に遷移。**ここで初めて監視が機能する状態になりました。**

つまり、**このサーバーの Java 監視は、最初から一度も動いていなかった**わけです。

### なぜ気づけなかったのか

アラームの「欠落データの処理」が `missing`（無視）になっていると、**データが 1 件も届いていなくても状態は `OK` のまま**です。

画面上は緑色。でも中身は空っぽ。これがいちばん怖い状態です。

**状態表示ではなく「値が実際に返ってきているか」で確認してください。**

```bash
# 実際に値が返っているかを確認する
aws cloudwatch get-metric-statistics \
  --namespace CWAgent \
  --metric-name procstat_lookup_pid_count \
  --start-time "YYYY-MM-DDT00:00:00Z" \
  --end-time "YYYY-MM-DDT12:00:00Z" \
  --period 300 --statistics Maximum

# 全サーバーでエージェントの実行ユーザーを洗い出す
sudo grep run_as_user /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.d/*
```

:::message alert
**`amazon-cloudwatch-agent.d/` の中身は、拡張子に関係なく全ファイルが設定として読み込まれます。**
このフォルダ内に `.bak` を作ると設定が二重定義になり、`Cannot translate JSON` でエージェントが起動しなくなります（これで 15 分止めました）。**バックアップは必ずフォルダの外に置いてください。**
:::

---

## まとめ

今回の調査で得られた教訓です。

1. **`journalctl -k` の oom-killer 記録が唯一の決定的証拠。**
   アプリログをいくら読んでも「突然消えた」しか分かりません。逆に、この記録があれば一発で確定します。

2. **障害時は、再起動する前に `journalctl` を確認する。**
   強制終了された場合アプリログには何も残らず、さらに `gc.log` は再起動で上書きされます。ログ提供を依頼するときも「**再起動前のログ・ローテーション済みの過去ログ**」と明示しましょう。

3. **OOM Killer のログ先頭のプロセス名は、殺されたプロセスではない。**
   `invoked`（要求して失敗した側）と `Killed`（実際に殺された側）を読み分けます。

4. **`gc.log` の 1 行目はサーバーのメモリ構成そのもの。**
   サーバーに入らなくても「4GB・スワップなし」まで判定でき、仮説を先に立てられます。

5. **終了処理ログの有無が、正常停止と強制終了を分ける最短の判定材料。**

6. **1 分周期の定例タスクログは、停止時刻の高精度な特定に使える。**

7. **同一構成のサーバーは、1 台で確定した原因がそのまま他台にも当てはまる。**
   対策は 1 台ずつではなく、全台まとめて計画すべきです。

8. **監視は「設定した」ではなく「値が返っている」まで確認して初めて完了。**
   一番怖いのは落ちることではなく、**「監視しているつもり」でいること**です。

「4GB・スワップなし・Java と MySQL が同居」という構成は、コスト最適化の過程でわりと生まれがちです。心当たりのある方は、`free -h` と `swapon --show` を今すぐ叩いてみてください。

---

## 参考資料

- [Amazon CloudWatch エージェントの設定ファイルを作成する](https://docs.aws.amazon.com/ja_jp/AmazonCloudWatch/latest/monitoring/CloudWatch-Agent-Configuration-File-Details.html)
- [Amazon EC2 インスタンスのスワップファイルを使用してスワップ領域として割り当てる](https://repost.aws/ja/knowledge-center/ec2-memory-swap-file)
- [CloudWatch アラームの欠落データの処理](https://docs.aws.amazon.com/ja_jp/AmazonCloudWatch/latest/monitoring/AlarmThatSendsEmail.html#alarms-and-missing-data)
- [journalctl(1) - Linux manual page](https://man7.org/linux/man-pages/man1/journalctl.1.html)

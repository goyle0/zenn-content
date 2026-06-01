---
title: "CloudWatchの「Javaプロセス落ち」アラートを1行で判断する運用メモ"
emoji: "🚨"
type: "tech"
topics: ["aws", "cloudwatch", "java", "tomcat", "monitoring"]
published: true
---

## 忙しい人向け（まず結論）

アラートが来たら、まずこの1行を実行してください。インストール不要です。

```bash
pgrep -x java >/dev/null && echo "OK java稼働中=アラームは一時的" || echo "NG java停止=要対応"
```

```
OK java稼働中=アラームは一時的
```

- **OK が出たら** → Java は今動いている。定時再起動などで生じた一瞬の空白を監視がたまたま拾っただけ。**対応不要**。
- **NG が出たら** → 本当に止まっている。**起動と原因調査が必要**。

判断の決め手は「**今 Java が動いているか**」の1点だけです。以下では、その理由と詳しい調査手順をまとめます。

---

## はじめに

ある朝、こんな CloudWatch アラームが飛んできました。

```
🚨 CloudWatch Alarm | nm-javaKilled-alarm | ALARM
Namespace: CWAgent
Metric: procstat_lookup_pid_count  (exe=java)
Threshold Crossed: 0.8 < 1.0
InstanceId: i-xxxxxxxx (t3.medium)
```

「Java プロセスが落ちたかも？」というアラートです。本記事では、

- このアラートが**何を意味しているのか**の読み方
- 「**今は問題ないのか**」をその場で切り分ける手順
- 何度でも使える**1行ヘルスチェックコマンド**

を、実際の調査ログに沿ってまとめます。EC2 上に Tomcat を載せている構成（Spring Boot 組み込みではなく、WAR デプロイ）を前提にしていますが、考え方はどの Java サーバでも共通です。

:::message
本記事のホスト名・インスタンスID・パスはサンプルに置き換えています。
:::

## アラートの読み方

### `procstat_lookup_pid_count` とは

CloudWatch エージェントが「指定した実行ファイル名のプロセスが**何個存在するか**」を定期的に数えるメトリクスです。今回は `exe=java` を監視しているので、通常は常時 `1`。

### 値が `0.8` ってどういうこと？

ここが最初のポイントです。`0.8` は小数なので「0.8個のプロセス」という意味ではありません。**監視周期の集計期間中に、PID カウントが一時的に `0` になった瞬間があった**ことを示します（平均すると 1 を割って 0.8 になった）。

> つまり「**一度落ちた**（あるいは再起動でダウンタイムが生じた）」という記録であって、「**今この瞬間も落ち続けている**」とは限りません。

```
1 1 1 0 1   ← 集計期間中に1回だけ 0 になると平均が下がる → ALARM
```

### 時刻は UTC、サーバは JST

アラート本文の時刻は **UTC** 表記であることが多いです。サーバ（dmesg やプロセス起動時刻）は **JST**。突き合わせるときは **+9時間** を忘れずに。

```
アラート: 2026-05-31 17:56 UTC
        = 2026-06-01 02:56 JST  ← サーバ側はこの時刻で探す
```

ここを間違えると「該当時刻に何も起きていない」と誤判定してしまいます。

## 調査：今は問題ないのか？

### Step 1. まず「今 Java が動いているか」

最初に確認すべきは現在の生死です。決定的なのはこの1点だけ。

```bash
ps -eo pid,lstart,etime,rss,cmd | grep '[j]ava'
```

出力例:

```
25769 Mon Jun  1 03:03:33 2026  06:32:10 1076328 /usr/java/default/bin/java ... org.apache.catalina.startup.Bootstrap start
```

読み取れること:

- **プロセスは存在する**（＝今は生きている）
- `lstart`（起動時刻）が **6/1 03:03:33** → アラート時刻（02:56 JST）の直後に起動し直している
- `etime`（連続稼働）が短い → **一度落ちて再起動された**証拠

`org.apache.catalina.startup.Bootstrap start` から、これは **Tomcat** であることも分かります。

:::message
`grep '[j]ava'` の `[j]` は、grep 自身のプロセスをヒットさせないための定番テクニックです（`java` にはマッチするが、`grep [j]ava` という文字列自身にはマッチしない）。
:::

### Step 2. OS の OOM Killer に殺されたか

Java が落ちる代表的な原因が、メモリ不足による **OS の OOM Killer** です。重要な注意点として、

> **OOM Kill は JVM やアプリのログには一切残りません。**`dmesg` でしか確認できません。

```bash
dmesg -T | grep -i 'killed process'
```

出力例（過去履歴）:

```
[木  3月 26 04:12:32 2026] Killed process 30162 (java) total-vm:3441036kB ...
```

**ここで時刻を必ず確認します。** 今回のアラート（6/1）と無関係な過去の履歴（例: 3月）であれば**無害**。アラート時刻と一致していれば OOM が原因と確定します。今回は 6/1 の記録は無く、過去履歴のみ＝**今回の落ちは OS-OOM ではない**と分かりました。

### Step 3. JVM ヒープ枯渇 / GC を確認

OS-OOM ではない場合、JVM 内部の `OutOfMemoryError` や GC の暴走を疑います。

```bash
tail -3 /path/to/tomcat/logs/gc.log
```

```
17986.739: [GC 639535K->298343K(765440K), 0.0076730 secs]
17987.781: [GC 639847K->306702K(760832K), 0.0221360 secs]
```

`Full GC` の連発ではなく通常の `GC`（Minor GC）で、ヒープも 765MB 内に収まって回収できています。→ **GC 的には健全**。

メモリの余裕も見ておきます。

```bash
free -m
```

```
              total   used   free   shared  buff/cache  available
Mem:           3884   2123   1241        0         520        1548
```

`available 1548MB` 確保できていれば当面の余裕あり。

### Step 4. Tomcat のログで落ち方を確認

```bash
grep -niE 'stopping|shutting down|server startup' /path/to/tomcat/logs/catalina.out | tail -20
```

`Stopping service` → `Server startup` が正常に並んでいれば、**クラッシュではなく正常なシャットダウン→起動**（＝再起動）だと分かります。

## 複数環境を横断チェックする

本番が用途別・テナント別に複数 EC2 へ分かれているような構成では、「他の環境は大丈夫か」も知りたくなります。1行で要点だけ並べるコマンドが便利です。

```bash
echo "[$(hostname)] java=$(pgrep -fc java) up=$(ps -o etime= -C java|tr -d ' ') memMB=$(free -m|awk '/Mem/{print $7}') oom=$(dmesg|grep -ic 'killed process')"
```

出力例:

```
[host-a] java=1 up=06:32:57 memMB=1069 oom=3
[host-b] java=1 up=06:39:56 memMB=2025 oom=1
[host-c] java=1 up=06:37:23 memMB=1540 oom=0
```

| 項目 | 意味 | 健全 | 要注意 |
| --- | --- | --- | --- |
| `java=` | Java プロセス数 | `1` | `0`（停止中） |
| `up=` | 連続稼働時間 | 長い | 極端に短い＝最近再起動 |
| `memMB=` | 空きメモリ(available) | 数百〜千MB | 100MB台＝OOM予備軍 |
| `oom=` | 過去の OOM kill 累計 | `0` | 前回より増えた＝新規OOM |

このとき面白い発見がありました。**全環境の `up` がほぼ同じ（約6.3〜6.6時間）** だったのです。逆算するとどれも **03:00〜03:07 に再起動**していました。

> 別々のサーバが同時刻に揃って落ちることはまずありません。これは**毎晩 03:00 頃の定時再起動**（cron など）の可能性が高く、アラートはその数分の空白を監視がたまたま拾った「想定内の一時的 ALARM」だと推測できます。

定時再起動の正体は cron / systemd timer で確認します。

```bash
crontab -l
cat /etc/crontab
cat /etc/cron.d/*
```

`0 3 * * *` 付近に Tomcat を restart する行があれば、想定内と確定できます。

## 結論：このアラートの判断基準

整理すると、判断の決め手は**「今 Java が動いているか」の1点**です。

- **動いている** → 一度落ちて自動復旧済み。今回のような定時再起動の空白を拾った一時的アラート。**対応不要**。
- **動いていない** → 本当の障害。**起動・原因調査が必要**。

## 何度でも使える1行ヘルスチェック

### パターンA：即席で OK / NG だけ判定

インストール不要。アラートが来たらこれを1発:

```bash
pgrep -x java >/dev/null && echo "OK java稼働中=アラームは一時的" || echo "NG java停止=要対応"
```

```
OK java稼働中=アラームは一時的
```

`pgrep -x java` はプロセス名が `java` と**完全一致**するものを探し、見つかれば終了コード0（成功）→ `&&` の右が実行されて `OK`。無ければ `||` の右で `NG`。CloudWatch の `procstat exe=java` とほぼ同じ見方になるので、アラートとの整合も取れます。

### パターンB：詳細まで1行で（常設化）

各サーバで**一度だけ**スクリプトを仕込み、以後は `jchk` だけで判定します。

```bash
cat > /usr/local/bin/jchk <<'EOF'
P=$(pgrep -cx java); U=$(ps -o etime= -C java 2>/dev/null|tr -d ' '); A=$(free -m|awk '/Mem:/{print $7}'); O=$(dmesg 2>/dev/null|grep -ic 'killed process')
[ "${P:-0}" -ge 1 ] && V="OK-稼働中" || V="NG-停止中-要対応"
echo "[$(hostname)] $V java=$P up=$U availMB=$A oom累計=$O"
EOF
chmod +x /usr/local/bin/jchk
```

以後はこれだけ:

```bash
jchk
```

```
[host-c] OK-稼働中 java=1 up=06:37:23 availMB=1540 oom累計=0
```

:::message
`oom累計` は dmesg に残る累計値です。**前回値との差**で見るのがコツ。増えていなければ新規 OOM は発生していません。
:::

## 補足：SSH 端末への貼り付けが崩れる問題

長い1行コマンドを SSH 端末に貼ると、折り返し位置に改行が差し込まれ、`;`・空白・`&&`・パイプの途中で分断されて壊れることがあります。実際、調査中に何度かこれで失敗しました。

対策は次のいずれか:

1. **コマンドを短い単独行にする**（折り返し列に届かせない）
2. **全体を1個の `echo "..."` に収める**（改行がクオート内に落ちて継続扱いになる）
3. **長い処理はスクリプト化**して短い語で呼ぶ（上記パターンB）

`$(...)` の中では改行が**コマンド区切り**になるため、パイプの途中で改行が入ると壊れます。ここがハマりどころでした。

## まとめ

- `procstat_lookup_pid_count` の小数値は「**一時的に落ちた記録**」であり、現在の生死とは別物。
- 判断の決め手は「**今 Java が動いているか**」の1点。`pgrep -x java` で十分。
- OS-OOM はアプリログに残らない。`dmesg -T | grep -i 'killed process'` で**時刻**を確認。
- 複数環境の `up` が揃って短いなら、**定時再起動**を疑う。
- 再利用できるよう `jchk` を仕込んでおくと、次回からは1語で判定できる。

同じアラートに悩まされている方の参考になれば幸いです。

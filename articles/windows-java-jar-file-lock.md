---
title: "「使用中のファイル」でjarが差し替えられない！ロック元のJavaプロセスを特定する"
emoji: "🔒"
type: "tech"
topics: ["windows", "java", "powershell", "troubleshooting"]
published: true
---

## TL;DR（急いでいる方向け）

管理者権限のPowerShellで、上から順に実行するだけです。

```powershell
# ① ロック元のJavaプロセスを特定する
Get-CimInstance Win32_Process -Filter "Name='java.exe'" |
  Select-Object ProcessId, CommandLine | Format-List

# ② 起動方法を判定する（①で得たPIDを指定）
$p = Get-CimInstance Win32_Process -Filter "ProcessId=6456"
Get-CimInstance Win32_Process -Filter "ProcessId=$($p.ParentProcessId)" |
  Select-Object ProcessId, Name, CommandLine | Format-List

# ③ 停止する（②で得た「親」のPIDを指定）
taskkill /PID 7596 /T /F

# ④ 停止できたか確認する
Get-CimInstance Win32_Process -Filter "Name='java.exe'" |
  Select-Object ProcessId, CommandLine | Format-List
```

**ポイントは3つだけです。**

1. `java.exe` は名前だけでは区別できないので、必ずコマンドラインまで見る
2. 止める前に「親プロセス」を見る（サービス経由だと勝手に復活します）
3. `taskkill` は**親のPID**に `/T` を付ける（親子まとめて確実に終了）

---

## はじめに

Windows上で動かしているJavaアプリ（実行可能jar）を新しいバージョンに差し替えようとしたとき、こんなダイアログに阻まれたことはないでしょうか。

```
使用中のファイル

Java(TM) Platform SE binary によってファイルは開かれているため、
操作を完了できません。

ファイルを閉じてから再実行してください。
```

「ファイルを閉じてから」と言われても、そのファイルを開いているのは**バックグラウンドで動いているJavaプロセス**であり、画面上のどこにも見当たらない、というのがよくあるパターンです。

この記事では、次の流れを実際のコマンドと出力例つきで整理します。

1. なぜロックされるのか
2. どのプロセスが掴んでいるのかを特定する
3. どうやって起動されたのかを判定する
4. リスクの低い順に停止する

### 対象読者

- Windows Server / Windows 上でJavaアプリを運用している方
- リリース作業でjarの差し替え・リネームをする方
- 「プロセスを消したいが、どれを消せばいいか分からない」状態の方

---

## 1. なぜロックされるのか

Windowsには「使用中のファイルは変更させない」という仕組みがあります。

Javaアプリを `java -jar myapp.jar` の形で起動すると、Javaはそのjarファイルを**開きっぱなしのまま**動き続けます。プログラムの中身（クラスファイル）を、必要に応じて随時読み出しているためです。

そのため、アプリが動いている間は対象のjarに対して以下ができません。

- 名前の変更（リネーム）
- 上書き
- 削除
- 移動

逆に言えば、**アプリを止めさえすればロックは自動的に外れます**。「ロックを無理やり解除するツール」を探すよりも、正しくプロセスを止めるほうが安全かつ確実です。

:::message
Linuxでは動作中のファイルでもrenameやrmができてしまいます（inode単位で管理されているため）。Windowsで同じ感覚で作業すると、この壁にぶつかります。
:::

---

## 2. ロック元のJavaプロセスを特定する

### タスクマネージャーでは判断できない

Javaアプリはすべて `java.exe` という同じ名前で動きます。複数のJavaアプリが動いているサーバーでは、タスクマネージャーの一覧を見ても「どれが目的のアプリなのか」が分かりません。

そこで、**起動時のコマンドライン（引数）まで表示させる**のがポイントです。

### PowerShellで一覧表示する

PowerShellを**管理者として実行**し、以下を打ちます。

```powershell
Get-CimInstance Win32_Process -Filter "Name='java.exe'" |
  Select-Object ProcessId, CommandLine | Format-List
```

実行結果の例:

```
ProcessId   : 6456
CommandLine : D:\app\Java\bin\java.exe  -Xss4m -jar "D:\app\myapp.jar"
```

`CommandLine` の中に、差し替えたいjarのパスが含まれているものが「犯人」です。

上の例では、次のことが1行で読み取れます。

- **PID（プロセスID）: 6456**
- 使っているJava: `D:\app\Java\bin\java.exe`（アプリに同梱されているJava）
- 起動オプション: `-Xss4m`（スタックサイズ指定）
- 対象jar: `D:\app\myapp.jar`

### コマンドプロンプト派の場合

PowerShellを使わない場合は、以下でも同様の情報が取れます。

```bat
wmic process where "name='java.exe'" get ProcessId,CommandLine
```

:::message alert
`wmic` はWindowsの新しいバージョンで非推奨（将来削除予定）となっています。新規に手順書を作る場合はPowerShellの `Get-CimInstance` を使うことをおすすめします。
:::

### Javaに限らずロック元を探したいとき

「Javaかどうかも分からない」「別のプロセスが掴んでいるかもしれない」という場合は、以下の方法が有効です。

| 方法 | 手順 | 備考 |
| --- | --- | --- |
| リソースモニター | `resmon` を実行 → CPUタブ → 「関連付けられたハンドル」にファイル名を入力 | 追加インストール不要。まずこれ |
| handle.exe | Sysinternals の `handle.exe -a ファイル名` | 詳細に分かるが要ダウンロード |
| PowerToys | ファイルエクスプローラーの拡張機能 | GUIで完結 |

---

## 3. どうやって起動されたのかを判定する（重要）

PIDが分かったからといって、すぐ `taskkill` してはいけません。

**起動方法によって、正しい止め方が変わる**ためです。ここを飛ばすと、次のようなことが起こります。

- 止めたのに勝手に再起動する
- 片方だけ残ってゾンビ状態になる

### 親プロセスを調べる

先ほどのPID（6456）の「親」を調べます。

```powershell
$p = Get-CimInstance Win32_Process -Filter "ProcessId=6456"
Get-CimInstance Win32_Process -Filter "ProcessId=$($p.ParentProcessId)" |
  Select-Object ProcessId, Name, CommandLine | Format-List
```

実行結果の例:

```
ProcessId   : 7596
Name        : cmd.exe
CommandLine : C:\Windows\system32\cmd.exe /c ""D:\app\myapp_start.bat" "
```

### 判定表

| 親プロセス | 起動方法 | 推奨される止め方 |
| --- | --- | --- |
| `cmd.exe` | バッチファイルから起動 | バッチのウィンドウで `Ctrl + C`、または親ごと停止 |
| `winsw.exe` / `nssm.exe` | サービスラッパー経由 | `Stop-Service` でサービスとして停止 |
| `services.exe` | Windowsサービス | 同上 |
| `explorer.exe` | ダブルクリックで手動起動 | ウィンドウを閉じる／プロセス停止 |
| 見つからない | 起動元がすでに終了済み | プロセスを直接停止 |

今回の例では親が `cmd.exe` なので、**バッチファイルから起動された**と判定できます。

### なぜ親を確認する必要があるのか

サービスとして登録されている場合、Javaプロセスだけを強制終了しても、サービスラッパーが「アプリが落ちた」と判断して**自動的に再起動します**。何度killしても復活する、という不可解な状況になるので、必ず先に確認しましょう。

---

## 4. 停止する（リスクの低い順）

### レベル1: ウィンドウを見つけて `Ctrl + C`（最も安全）

バッチ起動の場合、黒いコマンドプロンプトの画面がどこかに存在しています。最小化されている、他のウィンドウの後ろに隠れている、というだけのことが多いです。

- タスクバーに黒いアイコン（`C:\Windows\system32\cmd.exe`）がないか確認する
- `Alt + Tab` で全ウィンドウを一巡する
- タスクマネージャーの「アプリ」タブから右クリック →「前面に表示」

見つけたら `Ctrl + C` を押すか、ウィンドウを閉じます。アプリが自分で後片付け（ファイルの書き込み完了、接続のクローズなど）をしてから終了するため、これが最もきれいな終わり方です。

### レベル2: サービスとして停止

親が `winsw.exe` / `nssm.exe` / `services.exe` だった場合は、サービス名を調べて停止します。

```powershell
# サービス一覧から探す
Get-Service | Where-Object { $_.DisplayName -like "*myapp*" }

# 停止
Stop-Service -Name "MyAppService"
```

### レベル3: 強制終了（最終手段）

ウィンドウが見つからない、またはどうしても止まらない場合のみ実行します。

```powershell
taskkill /PID 7596 /T /F
```

| オプション | 意味 |
| --- | --- |
| `/PID 7596` | 対象のプロセスID（ここでは**親のcmd.exe**を指定） |
| `/T` | そのプロセスの子プロセスも一緒に終了する |
| `/F` | 強制終了する |

**ポイントは「親のPIDを指定して `/T` を付ける」ことです。**

- 子（java.exe）だけを止めると、親のcmd.exeが残ってしまう
- 親（cmd.exe）だけを止めると、子のjava.exeが残ってロックが解けない

`/T` を付けて親を指定することで、まとめて確実に終了できます。

PowerShellのコマンドレットを使う場合はこちらです。

```powershell
Stop-Process -Id 6456 -Force
```

ただし `Stop-Process` には `/T` に相当する「子プロセスも連鎖して終了する」オプションがないため、親子まとめて落としたい今回のようなケースでは `taskkill /T /F` のほうが素直です。

---

## 5. 停止できたか確認する

```powershell
Get-CimInstance Win32_Process -Filter "Name='java.exe'" |
  Select-Object ProcessId, CommandLine | Format-List
```

**何も表示されなければ停止完了**です。この状態でjarのリネーム・差し替えができるようになります。

複数のJavaアプリを動かしているサーバーでは、対象のjarを含む行が消えていることを確認してください。

---

## 6. 強制終了のリスクと注意点

`taskkill /F` は、たとえるなら**「保存ボタンを押した瞬間に電源ケーブルを抜く」**のと同じです。アプリに一切の後片付けをさせずに、即座に終了させます。

実行前に、以下のような処理が動いていないことを必ず確認してください。

- 帳票・Excelの出力処理
- データの一括取込／一括更新
- 時間のかかるバッチ処理（自動生成処理など）
- データベースへの書き込みを伴う操作

これらの最中に強制終了すると、次のような後始末が必要になることがあります。

| 起こりうること | 対処 |
| --- | --- |
| 出力途中の壊れたファイルが残る | 手動で削除して再実行 |
| DBのトランザクションが中途半端に残る | 接続が切れれば自動ロールバックされるのが通常だが、要確認 |
| ロックファイル（`.lock` 等）が残り、次回起動できない | 該当ファイルを手動削除 |
| ログファイルが途中で切れる | 実害は少ないが、調査時に注意 |

「自分ひとりしか使っていない」「画面は待機状態」という状況であれば、リスクは低いと判断してよいでしょう。

---

## 7. まとめ

| ステップ | やること | 使うコマンド |
| --- | --- | --- |
| ① | ロック元のJavaプロセスを特定する | `Get-CimInstance Win32_Process -Filter "Name='java.exe'"` |
| ② | 起動方法（親プロセス）を判定する | `Get-CimInstance Win32_Process -Filter "ProcessId=<親PID>"` |
| ③ | 停止する | `Ctrl + C` → `Stop-Service` → `taskkill /PID <親PID> /T /F` |
| ④ | 停止を確認する | ①と同じコマンドで何も出なければOK |

いきなり強制終了せず、「特定 → 判定 → 安全な順に停止」の3段階で進めるのが、事故を起こさないコツです。

---

## おまけ: そもそもこの手間を減らすには

リリースのたびに手作業でプロセスを探すのは非効率です。運用が続くのであれば、以下の対応を検討する価値があります。

- **サービス化する**（WinSWなど）
  `Stop-Service` / `Start-Service` の一行で確実に停止・起動できるようになり、隠れたウィンドウを探す作業がなくなります。
- **停止用バッチを用意しておく**
  PIDの特定から停止までを1ファイルにまとめておけば、誰が作業しても同じ結果になります。
- **起動時にPIDファイルを出力する**
  アプリ側で自分のPIDをファイルに書き出しておけば、探す手間そのものがなくなります。

---

## 参考

- [Get-CimInstance (Microsoft Learn)](https://learn.microsoft.com/powershell/module/cimcmdlets/get-ciminstance)
- [taskkill (Microsoft Learn)](https://learn.microsoft.com/windows-server/administration/windows-commands/taskkill)
- [Handle - Sysinternals](https://learn.microsoft.com/sysinternals/downloads/handle)

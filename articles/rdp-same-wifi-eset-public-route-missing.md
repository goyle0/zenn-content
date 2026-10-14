---
title: "同じWi-FiなのにRDPがつながらない原因は、ESETのパブリック扱いとVPNが消した経路だった"
emoji: "🖥️"
type: "tech"
topics: ["windows", "rdp", "network", "powershell", "troubleshooting"]
published: true
---

## 結論

同じWi-FiのPCどうしでRDPがつながらなくなった原因は、別々の2つでした。
1台目は受け側のWi-Fiが「パブリック」扱いで、ESETのファイアウォールが3389番を止めていました。受け側でWindowsとESETの両方をプライベート扱いに変えて直りました。
2台目はエラー `0x904`／拡張 `0x7` で、受け側の経路表から自分のネットワーク（192.168.1.0/24）への経路が消えていました。`New-NetRoute` で経路を戻して直りました。
ping のTTLが同じネットワークなのに127なら、`Find-NetRoute` で経路を確認するのが近道です。

## 背景

社内の同じWi-Fiにつながっている Windows PC どうしで、リモートデスクトップ（RDP）が急につながらなくなりました。受け側は2台あり、それぞれ原因が違いました。

特に2台目は、エラーコード `0x904`／拡張 `0x7` だけを手がかりに、UDP、セキュリティソフト、証明書、Windows Update と順に疑いました。最後に ping のTTLが1つ減っていることに気づき、ようやく原因にたどり着いています。切り分けの手順を残しておきます。

:::message
社内で実際に起きた事例です。PC名、IPアドレス、Wi-Fi名、MACアドレスなどはすべて架空の値に置き換えています。
:::

私の環境は次のとおりです。3台とも同じWi-Fi（ここでは「社内Wi-Fi」と呼びます）の、同じネットワーク（192.168.1.0/24）にいます。

| 役割 | 呼び名 | IPアドレス（架空） | 備考 |
|---|---|---|---|
| つなぐ側 | PC-A | 192.168.1.10 | Windows 11、Wi-Fi |
| 受け側1 | PC-B | 192.168.1.20 | Windows 11 Home ＋ RDP Wrapper、Wi-Fi |
| 受け側2 | PC-C | 192.168.1.30 | Windows 11 Home ＋ RDP Wrapper、Wi-Fi |
| ルーター | － | 192.168.1.1 | 家庭・小規模オフィス向けのWi-Fiルーター |

セキュリティソフトは、会社で一括管理している ESET Endpoint Security 13 です。受け側はどちらも Home エディションなので、RDP Wrapper でRDPの受け付けを有効にしていました。

:::message alert
RDP Wrapper は、Home エディションでRDPの受け付けを有効にする非公式のツールです。ライセンス条件や社内ルールを確認したうえで利用してください。セキュリティソフトに「望ましくない可能性のあるアプリ」として検出されることもあります。
:::

2台の症状と原因をまとめると、次のようになります。

| 接続 | 症状 | 原因 | 直し方 |
|---|---|---|---|
| PC-A → PC-B | つながらない。ping もタイムアウト。`TcpTestSucceeded` が False | PC-B のWi-Fiがパブリック扱いで、ESETが3389番を止めていた | 受け側（PC-B）で社内Wi-Fiをプライベート扱いにする（WindowsとESETの両方） |
| PC-A／PC-B → PC-C | 「リモート セッションを構成しています…」で止まり、`0x904`／拡張 `0x7`。`TcpTestSucceeded` は True | PC-C の経路表から `192.168.1.0/24` が消え、返事がルーター経由になっていた | `New-NetRoute` で経路を戻す（Wi-Fiのつなぎ直しや再起動でも戻る） |

## 原因1：受け側のWi-Fiが「パブリック」扱いだった（PC-B）

PC-A から PC-B へRDPできず、ping もタイムアウトしていました。PC-B の RDP Wrapper の設定画面（RDPConf）は、すべて緑で `[fully supported]` と表示されていました。

### 1. ping ではなくポートで確認する

Windows は初期状態では ping に返事をしません。RDP Wrapper のインストーラーは、Windows ファイアウォールに「3389番を全ネットワークで許可」する規則を追加します。ただし ping の許可は追加しません（インストーラーのソースで確認できます）。

そのため、ping が通らないからといって、ネットワークがつながっていないとは言えません。RDPの入口（3389番）まで届くかどうかは、ポートを指定して確認しました。

```powershell
# PC-A で実行
Test-NetConnection 192.168.1.20 -Port 3389
```

結果は次のとおりです。

```
ComputerName     : 192.168.1.20
RemotePort       : 3389
InterfaceAlias   : Wi-Fi
SourceAddress    : 192.168.1.10
TcpTestSucceeded : False
```

`InterfaceAlias` が `Wi-Fi` なので、VPNなど別の通り道には流れていません。`TcpTestSucceeded : False` から、3389番の入口で止まっていることが分かりました。

### 2. 同じネットワークでお互いが見えているかを arp で確認する

Wi-Fiルーターには、同じWi-Fiの端末どうしの通信を禁止する機能があります（「プライバシーセパレーター」「APアイソレーション」などの名前です）。これが原因かどうかは、arp で切り分けられます。

```powershell
# PC-A で、ping を1回送った直後に実行
arp -a | findstr 192.168.1.20
```

PC-B のMACアドレスが表示されました。

```
  192.168.1.20          xx-xx-xx-xx-xx-xx     動的
```

表示されたMACアドレスは、PC-B の `ipconfig /all` の値と一致しました。2台はお互いを見つけられているので、止めているのはルーターではなく PC-B の中だと分かります。

### 3. 受け側で状態を確認する

PC-B で、RDPの受け付け状況、ネットワークの扱い、ファイアウォールの状態をまとめて確認しました。

```powershell
# PC-B で実行
netstat -an | findstr :3389
Get-NetConnectionProfile | Format-Table Name,InterfaceAlias,NetworkCategory
Get-NetFirewallRule -DisplayName "Remote Desktop" | Format-Table DisplayName,Enabled,Profile,Action
Get-CimInstance -Namespace root/SecurityCenter2 -ClassName FirewallProduct | Select-Object displayName
```

結果は次のとおりでした。

| 確認項目 | 結果 | 分かったこと |
|---|---|---|
| `netstat` | `0.0.0.0:3389 ... LISTENING` | RDPの受け付けは動いている |
| `Get-NetConnectionProfile` | 社内Wi-Fiが Public | Wi-Fiが「パブリック（信頼しない）」扱い |
| `Get-NetFirewallRule` | 「Remote Desktop」規則が 有効・Any・Allow | Windows ファイアウォールの許可は残っている |
| `FirewallProduct` | ESETのファイアウォールが登録されている | 実際に通信を判定しているのはESET |

効いていたのは最後の行です。ESETのような他社製のファイアウォールが入っていると、**Windows ファイアウォールの規則は判定に使われない**ので、RDP Wrapper が追加した規則も効きません。

ESETは、つないだネットワークを「パブリック」と「プライベート」に分けて扱います。パブリックでは、RDPの受け付けやファイル共有を止めます。初期状態では、Windows がパブリックと判定したネットワークは、ESETでもパブリックになります（ESET公式ヘルプ）。

### 4. 受け側でプライベート扱いに変える

PC-B（受け側）で、社内Wi-Fiだけをプライベート扱いにしました。ほかのWi-Fiはパブリックのままです。

```powershell
# PC-B の管理者 PowerShell
Set-NetConnectionProfile -InterfaceAlias Wi-Fi -NetworkCategory Private
Get-NetConnectionProfile | Format-Table Name,InterfaceAlias,NetworkCategory
```

ESET側も変更します。一度パブリックと判定されたネットワークは、Windows 側を変えても、ESET側の扱いがそのまま残ることがあります。ESETの画面で次の順に操作しました。

1. 「設定」→「ネットワーク」→「ネットワーク接続」を開く
2. 対象のWi-Fiの「︙」→「編集」を選ぶ
3. プロファイルを「プライベート」にして保存する

これで PC-A からの `Test-NetConnection 192.168.1.20 -Port 3389` が `True` になり、RDPもつながりました。

:::message alert
プライベート扱いにすると、同じWi-Fiにいるほかの機器からも、そのPCのRDPの入口や共有フォルダーが見えるようになります。来客も使うWi-Fiなら、ESETのファイアウォールに「PC-A のIPアドレスからの3389番だけを許可」する規則を追加するほうが影響は小さくなります。
ESETを管理サーバー（ESET PROTECT）で一括管理している場合は、PC側の変更が管理側の設定で戻されたり、変更できなかったりすることがあります。
:::

:::message
つまずいたところ：最初、このコマンドをつなぐ側（PC-A）で実行してしまい、何も変わりませんでした。入口を閉じているのは受け側のファイアウォールなので、変更するのは受け側です。
:::

## 原因2：返事がルーター経由の遠回りになっていた（PC-C）

PC-A から PC-C へRDPすると、「接続しています」→「リモート セッションを構成しています…」まで進みます。そのあと「このコンピューターはリモート コンピューターに接続できません。」と表示されて失敗しました。「詳細の表示」のエラーコードは `0x904`／拡張エラー コード `0x7` です。

`Test-NetConnection 192.168.1.30 -Port 3389` は `TcpTestSucceeded : True` で、入口までは届いていました。PC-B から PC-C へつないでも同じエラーです。同じ日の朝までは、PC-A から PC-C へ普通につながっていました。

`0x904`／`0x7` は、接続は始まったものの、その後の準備のやり取りの途中で切れたときのエラーです。原因1の「入口で止められる」とは段階が違います。

### 1. 受け側のログで、どこで止まったかを見る

PC-C で、RDPの接続処理のログを確認しました。

```powershell
# PC-C で実行
Get-WinEvent -LogName Microsoft-Windows-RemoteDesktopServices-RdpCoreTS/Operational -MaxEvents 30 |
  Format-Table TimeCreated,Id,Message -Wrap
```

PC-B から接続を試したときの流れは次のとおりでした（IPアドレスは置き換えています）。

| 経過 | イベントID | 内容 |
|---|---|---|
| 0秒 | 131 | 192.168.1.20 からのTCP接続を受け付け |
| 0秒 | 65, 72, 141 | 接続の準備を開始 |
| （約22秒、何も進まない） | | |
| 22秒 | 143, 142 | TCPの書き込み・読み取りに失敗（エラー64：相手側から切られた） |
| 22秒 | 226, 102, 148 | 接続を終了 |

TCPの接続はできているのに、その直後の暗号化の取り決め（TLS）の段階で通信が進まず、切れていました。

### 2. いつから壊れたかを絞り込む

最後にRDPの認証が成功した時刻は、次のログで確認できます。

```powershell
# PC-C で実行
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-TerminalServices-RemoteConnectionManager/Operational'; Id=1149} -MaxEvents 5 |
  Format-List TimeCreated,Message
```

続けて `Microsoft-Windows-TerminalServices-LocalSessionManager/Operational` を見ると、最後の成功のあとに PC-C 自身が別の操作に使われていたことが分かりました（PC-C から別のPCへRDPしていた、など）。最後に成功した時刻と最初に失敗した時刻の間に、そのPCで何をしたかが一番の手がかりになります。

### 3. 疑ったけれど違ったもの

原因にたどり着くまでに、次のものを順に確認しました。どれも原因ではありませんでした。

| 疑ったもの | 確認方法 | 結果 |
|---|---|---|
| つなぐ側のPCだけの問題 | 別のPC（PC-B）から PC-C へ接続 | 同じエラー。受け側か通り道の問題 |
| RDPのUDP通信 | つなぐ側でUDPを使わない設定（下記）にして再接続 | 変化なし（設定は元に戻す） |
| セキュリティソフト（ESET） | 両側で「保護」と「ファイアウォール」を一時停止。ネットワーク攻撃保護（IDS）の一時ブロック一覧とネットワーク保護ログを確認 | 変化なし。一覧もログも空 |
| 受け側の RDP Wrapper・サービス | RDPConf の表示、`Get-Service TermService,UmRdpService,SessionEnv` | すべて正常で動作中 |
| 受け側の証明書 | System ログの Schannel 36870（証明書の秘密キーにアクセスできない） | 記録なし |
| Windows Update・再起動 | 更新の履歴と最終再起動の時刻を、最後の成功時刻と比べる | すべて最後の成功より前 |
| IPアドレスの重複 | つなぐ側の `arp -a` のMACと PC-C のMACを比べる。System ログの Tcpip 4199 | MACは一致、記録なし |
| 大きな通信が落ちている（MTU） | `ping -f -l 1472 192.168.1.30` | 応答あり |

UDPを使わない設定は、つなぐ側の管理者コマンドプロンプトで次のように切り替えました。グループポリシーの「クライアントで UDP を無効にする」と同じ設定です。

```bat
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services\Client" /v fClientDisableUDP /t REG_DWORD /d 1 /f
rem 元に戻す
reg delete "HKLM\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services\Client" /v fClientDisableUDP /f
```

:::message
寄り道：途中で受け側のネットワークの扱いをパブリックに変えたところ、エラーが `0x204`（入口に届かない）に変わり、`TcpTestSucceeded` も False になりました。原因1と同じ「入口で止められる」状態が上乗せされただけで、本来の問題とは別です。切り分け中は、一度に変える設定を1つにしたほうが混乱しません。
:::

### 4. 手がかりは ping のTTLが127だったこと

PC-C のESETファイアウォールを一時停止した状態で、PC-A から ping を送りました（確認後すぐに元に戻しています）。

```
192.168.1.30 からの応答: バイト数 =32 時間 =3ms TTL=127
192.168.1.30 からの応答: バイト数 =1200 時間 =5ms TTL=127
192.168.1.30 からの応答: バイト数 =1472 時間 =3ms TTL=127
```

返事は届いていますが、**TTLが127** になっています。

TTLは、ルーターを1台通るたびに1ずつ減る数字です。Windows は128から送り出します（ルーターからの返事は64でした）。同じネットワーク内で直接届くなら、128のまま届くはずです。

127ということは、PC-C からの返事がルーターを1台経由して届いていることになります。同じネットワークにいるのに、遠回りしていました。

### 5. 経路表を見る

PC-C で、PC-A あての通信がどの経路を通るかと、経路表の中身を確認しました。

```powershell
# PC-C で実行
Find-NetRoute -RemoteIPAddress 192.168.1.10 | Format-List InterfaceAlias,DestinationPrefix,NextHop
Get-NetRoute -AddressFamily IPv4 |
  Where-Object { $_.DestinationPrefix -like '192.168.*' -or $_.DestinationPrefix -eq '0.0.0.0/0' } |
  Format-Table DestinationPrefix,NextHop,InterfaceAlias,RouteMetric
```

結果は次のとおりです。

```
InterfaceAlias    : Wi-Fi
DestinationPrefix : 0.0.0.0/0
NextHop           : 192.168.1.1

DestinationPrefix NextHop     InterfaceAlias RouteMetric
----------------- -------     -------------- -----------
192.168.1.30/32   0.0.0.0     Wi-Fi                  256
0.0.0.0/0         192.168.1.1 Wi-Fi                  256
```

ほかに、VPN用のアダプターに `10.x.x.x/32` の経路が残っていました。

PC-A あての通信に、既定の経路（0.0.0.0/0）、つまりルーター行きの経路が選ばれています。本来あるはずの次の経路が消えていました。

```
192.168.1.0/24    0.0.0.0     Wi-Fi    ← 同じネットワークへは直接届ける経路
192.168.1.255/32  0.0.0.0     Wi-Fi
```

IPアドレス（192.168.1.30）とサブネットの長さ（/24）は正しいのに、経路だけが無い状態でした。

### 6. RDPが失敗した理由

行きと帰りで通る道が違っていました。

```
行き：PC-A ─────────────→ PC-C          （直接届く）
帰り：PC-C ─→ ルーター ─→ PC-A          （遠回り）
```

PC-A には正常な経路があるので、行きは直接届きます。一方で、PC-C の返事はすべてルーター経由です。ルーターからは会話の帰り道しか見えません。ルーターによっては、こうした片側しか見えない通信を不正とみなして捨てたり、「直接送ってください」という通知を返したりします。

今回はTCPの接続までは成立し、暗号化の取り決め（TLS）の途中で止まりました。ルーター内部で何が起きたかまでは確認できていませんが、症状とはよく合います。PC-A からも PC-B からも、PC-C にだけつながらなかったことも、これで説明がつきます。

### 7. 経路が消えた理由（推定）

PC-C には、外部拠点へつなぐVPNクライアントが入っていました。最後の成功のあとに PC-C 自身が別の操作に使われていたこと、経路表にVPN用アダプターの経路が残っていたことから、次の流れと推定しています（VPNを使った時刻までは確認できていません）。

1. 通信をすべてVPNに通す方式（フルトンネル）のVPNクライアントが、接続中にローカルネットワークの経路を置き換える
2. 切断したときに、その経路を元に戻し損ねる

「VPNを切ったあとにローカルネットワークに届かなくなり、再起動で直った」という報告は、ほかのVPN製品でも見られます。

### 8. 消えた経路を戻す

PC-C の管理者 PowerShell で、消えた経路を追加しました。

```powershell
New-NetRoute -DestinationPrefix 192.168.1.0/24 -InterfaceAlias Wi-Fi -NextHop 0.0.0.0 -PolicyStore ActiveStore
```

`-NextHop 0.0.0.0` は「ルーターを通さず直接届ける」という意味です。`-PolicyStore ActiveStore` を付けると、追加した経路は再起動で消えます。

Wi-Fiのつなぎ直しや再起動のときは、Windows が正しい経路を作り直します。手で足した経路を残しておく必要はありません。PCを持ち出して別のネットワークにつなぐこともあるので、残らないほうが安全です。

追加したあとは、次のコマンドで確認しました。

```powershell
Find-NetRoute -RemoteIPAddress 192.168.1.10 | Format-List DestinationPrefix,NextHop
# → DestinationPrefix 192.168.1.0/24、NextHop 0.0.0.0 なら直接届く
```

PC-A から ping を送り、TTLが128に戻っていれば遠回りは解消しています。この経路を追加したところ、PC-A からも PC-B からも PC-C へRDPできるようになりました。

:::message
Wi-Fiのつなぎ直しや再起動でも直ります。今回は PC-C を同じWi-Fi経由のリモート操作ツールで操作していたため、Wi-Fiを切ると操作できなくなる状況でした。そこで、回線を切らずに直せる `New-NetRoute` を使いました。
:::

### 9. 再発防止

VPNを切ったあとに、ローカルネットワークからのRDPがおかしくなったら、まず経路表（`Find-NetRoute`）を確認します。Wi-Fiのつなぎ直しか再起動で元に戻ります。

根本的には、VPN側で「ローカルネットワークあての通信はVPNに通さない」設定（スプリットトンネル）にするのが確実です。ただし、これはVPNの管理者の判断が必要になります。

## エラーコードの見分け方

今回出た2つのエラーは、見るべき場所がまったく違いました。

| エラー | 意味 | 今回の原因 | 主に見る場所 |
|---|---|---|---|
| `0x204`（拡張 `0x0`） | RDPの入口（3389番）に届かない | 受け側のESETがパブリック扱いで入口を止めていたとき（原因2の寄り道で発生） | 受け側のファイアウォール、ネットワークの扱い、サービスの状態 |
| `0x904`／拡張 `0x7` | 接続は始まったが、準備のやり取りの途中で切れた | 返事がルーター経由の遠回りになっていた | 通り道（経路、VPN、途中の機器）、受け側の証明書、通信を検査するソフト |

## まとめ

同じWi-Fi内のRDPが失敗するときは、まずエラーコードで「入口で止まっているのか、途中で切れているのか」を分けると、見る場所を絞り込めます。入口で止まっているなら受け側のファイアウォールとネットワークの扱いを、途中で切れているなら経路やVPNを疑います。今回の経験から、次に同じことが起きたときは次の順で確認します。

1. ping は判断材料にせず、`Test-NetConnection &lt;IP&gt; -Port 3389` で入口まで届くかを見る（`InterfaceAlias` で通り道も分かる）
2. `arp -a` で相手のMACアドレスが出るかを見る（出ればWi-Fiルーターの端末間通信禁止は原因ではない）
3. `root/SecurityCenter2` の `FirewallProduct` で、実際に判定しているファイアウォール製品を確認する（他社製が入っていればWindows ファイアウォールの規則は効かない）
4. パブリック／プライベートの扱いは受け側で変える。ESETはWindowsとは別に扱いを覚えているので、ESET側も確認する
5. 同じネットワーク内なのにTTLが127なら、`Find-NetRoute -RemoteIPAddress &lt;相手&gt;` で NextHop がルーターになっていないかを見る
6. VPNクライアントを使ったあとは経路表を確認する。リモート操作中なら `New-NetRoute ... -PolicyStore ActiveStore` で回線を切らずに戻せる

## 参考

- [リモート デスクトップからリモート コンピューターに接続できない（Microsoft Learn）](https://learn.microsoft.com/ja-jp/troubleshoot/windows-server/remote/remote-desktop-cannot-connect-remote-computer)
- [Test-NetConnection（Microsoft Learn）](https://learn.microsoft.com/ja-jp/powershell/module/nettcpip/test-netconnection)
- [Find-NetRoute（Microsoft Learn）](https://learn.microsoft.com/ja-jp/powershell/module/nettcpip/find-netroute)
- [New-NetRoute（Microsoft Learn、PolicyStore の説明）](https://learn.microsoft.com/ja-jp/powershell/module/nettcpip/new-netroute)
- [Set-NetConnectionProfile（Microsoft Learn）](https://learn.microsoft.com/ja-jp/powershell/module/netconnection/set-netconnectionprofile)
- [ネットワーク接続プロファイル（ESET Endpoint Security 13 ヘルプ）](https://help.eset.com/ees/13/ja-JP/idh_config_epfw_profile.html)
- [「ネットワーク」設定画面（ESET Endpoint Security 13 ヘルプ）](https://help.eset.com/ees/13/ja-JP/idh_page_epfw_settings.html)
- [RDP Wrapper インストーラーのソース（ファイアウォール規則の追加部分）](https://github.com/stascorp/rdpwrap/blob/master/src-installer/RDPWInst.dpr)
- [RDP session freezes (UDP)（Windows OS Hub）](https://woshub.com/rdp-session-freezes-udp/)

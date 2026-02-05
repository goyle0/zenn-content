---
title: "【Windows Home】RDP Wrapper + autoupdate v1.4 導入ガイド"
emoji: "🖥️"
type: "tech"
topics: ["windows", "rdp", "remotedesktop", "rdpwrapper"]
published: true
---

## はじめに

Windows Home エディションでは、リモートデスクトップのホスト機能（接続を受ける側）が使えません。
**RDP Wrapper** を使えばこの制限を回避できますが、Windows Update のたびに動かなくなるのが悩みどころです。

この記事では、**asmtron版 autoupdate v1.4** を導入して、Windows Update後も自動で復旧させる方法をまとめます。

## 前提知識

### RDP Wrapper とは

RDP Wrapper は、Windowsの「ターミナルサービス」と「サービス管理」の間に入る仕組みで、Windows Home でもリモートデスクトップ接続を受けられるようにするツールです。元の `termsrv.dll` ファイル自体は変更しないため、比較的安全な方式です。

### なぜ Windows Update で壊れるのか

Windows Update で `termsrv.dll` が新しいバージョンに更新されると、RDP Wrapper の設定ファイル（`rdpwrap.ini`）に新バージョンの情報が含まれていないため、動作しなくなります。

### autoupdate v1.4 が解決すること

PC起動時に自動で以下を行います：

1. 現在の `termsrv.dll` のバージョンを確認
2. `rdpwrap.ini` に対応情報があるかチェック
3. なければコミュニティが提供する最新の INI ファイルを自動取得
4. それでも対応がなければ、自動生成（autogen）を試行
5. RDP サービスを再起動して適用

## autoupdate v1.4 の仕組み

### INI ファイルの取得元（優先順）

autoupdate は複数のソースから順番にINIファイルの取得を試みます。1つが失敗しても次を試すので、冗長性があります。

| 優先順位 | 提供者 | 備考 |
| --- | --- | --- |
| 1 | asmtron | autoupdate 開発者本人 |
| 2 | sebaxakerhtc | 最も更新頻度が高い（2026年2月時点でも活発） |
| 3 | affinityv | 高頻度で更新（2026年1月末更新確認） |

さらにカスタムソースを追加することも可能です。

### autogen（自動生成）機能

v1.4 の目玉機能です。コミュニティの INI にもまだ対応情報がない場合、Symbol Checker ツール（`pdblister v0.0.4`）と Microsoft Debugging Information Dumper を使って、新しい `termsrv.dll` の対応情報を自動で生成します。

### コミュニティの保守体制

RDP Wrapper はオープンソースのコミュニティプロジェクトで、複数の貢献者が並行して INI ファイルを更新しています。

| 貢献者 | 役割 |
| --- | --- |
| **sebaxakerhtc** | 最も活発。独自のフォーク版（v1.8.9.9）も提供 |
| **asmtron** | autoupdate.bat 開発者。v1.4メンテナ |
| **affinityv** | 安定した更新頻度 |

Windows の大型アップデート後、通常は数日〜2週間以内にコミュニティが対応します。

## 導入手順

### ステップ1：インストーラーの実行

以下からインストーラーをダウンロードして、管理者として実行します。

**ダウンロードURL：**

```
https://github.com/asmtron/rdpwrap/releases/download/v1.4/RDPWrap-with-Autoupdate-v1.4-Installer.exe
```

:::message
ZIP版もあります：
`https://github.com/asmtron/rdpwrap/releases/download/v1.4/RDPWrap-with-Autoupdate-v1.4-Installer.zip`
:::

インストール先は自動で `C:\Program Files\RDP Wrapper` に設定されます。
**他のフォルダにはインストールしないでください。**

### ステップ2：Windows Defender の除外設定

ウイルス対策ソフトが RDP Wrapper のファイルを誤検知して削除する場合があります。必ず除外設定を行ってください。

**手順（Windows 11 の場合）：**

1. **設定** → **プライバシーとセキュリティ** → **Windows セキュリティ** → **ウイルスと脅威の防止**
2. 「ウイルスと脅威の防止の設定」の **設定の管理** をクリック
3. **除外の追加または削除** をクリック
4. **除外の追加** → **フォルダー** を選択
5. `C:\Program Files\RDP Wrapper` を指定して追加

### ステップ3：タスクスケジューラの登録確認

インストーラーが自動で登録していることが多いですが、念のため確認します。

**管理者としてコマンドプロンプトを開いて実行：**

```shell
schtasks /query /tn "RDP Wrapper Autoupdate"
```

**登録済みの場合の表示例：**

```
フォルダー\
タスク名                                 次回の実行時刻         状態
======================================== ====================== ===============
RDP Wrapper Autoupdate                   N/A                    準備完了
```

「次回の実行時刻: N/A」は正常です（PC起動時トリガーのため固定の時刻はありません）。

**未登録だった場合は、以下を管理者として実行：**

```shell
"C:\Program Files\RDP Wrapper\helper\autoupdate__enable_autorun_on_startup.bat"
```

## 導入後の確認チェックリスト

| 項目 | 確認方法 |
| --- | --- |
| RDP Wrapper インストール | `C:\Program Files\RDP Wrapper` にファイルが存在する |
| Defender 除外設定 | Windowsセキュリティ → 除外 にフォルダが表示される |
| タスクスケジューラ登録 | `schtasks /query /tn "RDP Wrapper Autoupdate"` で確認 |
| RDP接続テスト | 別端末から `mstsc.exe` で接続できる |

## トラブルシューティング

### Windows Update 後にリモートデスクトップが繋がらない

autoupdate が自動実行されるまでPC再起動が必要です。再起動しても繋がらない場合は、手動で以下を管理者として実行してください：

```shell
"C:\Program Files\RDP Wrapper\autoupdate.bat"
```

### autoupdate のログを確認したい

ログ付きで手動実行する場合：

```shell
"C:\Program Files\RDP Wrapper\autoupdate.bat" -log
```

ログファイルは同じフォルダ内の `autoupdate.log` に出力されます。

### ファイルが消えている

Windows Defender の除外設定が正しく行われていない可能性があります。ステップ2を再確認してください。

### タスクスケジューラの登録を解除したい

```shell
"C:\Program Files\RDP Wrapper\autoupdate.bat" -taskremove
```

## autoupdate のオプション一覧

| オプション | 内容 |
| --- | --- |
| `-log` | 実行ログを `autoupdate.log` に出力 |
| `-taskadd` | PC起動時の自動実行をタスクスケジューラに登録 |
| `-taskremove` | タスクスケジューラから自動実行を削除 |

## 注意事項

:::message alert
**非公式ツールです**

RDP Wrapper はMicrosoft公式のツールではありません。以下の点を理解した上で使用してください：

- Windows の利用規約に抵触する可能性があります
- コミュニティの対応が遅れた場合、一時的に使用できなくなることがあります
- 業務利用の場合はリスクを十分に理解してください
- 100%の動作保証はありません
:::

### より確実な代替手段

| 方法 | 費用 | 確実性 |
| --- | --- | --- |
| Windows Pro へアップグレード | 約14,000円 | 最も確実 |
| Chrome リモートデスクトップ | 無料 | 高い |
| Rustdesk | 無料 | 高い |
| AnyDesk | 無料プランあり | 高い |

## 参考リンク

- **asmtron/rdpwrap リポジトリ**: https://github.com/asmtron/rdpwrap
- **PR #1160（元の公開場所）**: https://github.com/stascorp/rdpwrap/pull/1160
- **sebaxakerhtc 版 INI**: https://github.com/sebaxakerhtc/rdpwrap.ini
- **affinityv 版 INI**: https://github.com/affinityv/INI-RDPWRAP

## まとめ

autoupdate v1.4 を導入すれば、Windows Update 後の RDP Wrapper の手動復旧作業はほぼ不要になります。

**導入で行うことは3つだけ：**

1. インストーラーを実行
2. Windows Defender の除外設定
3. タスクスケジューラの登録確認

あとはPC再起動時に自動で最新の対応情報を取得してくれます。

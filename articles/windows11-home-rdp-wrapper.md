---
title: "Windows 11 HomeでRDP（リモートデスクトップ）を有効にする方法【2026年版】"
emoji: "🖥️"
type: "tech"
topics: ["windows", "rdp", "remotedesktop", "windows11"]
published: true
---

## はじめに

Windows 11 Homeエディションでは、リモートデスクトップの**ホスト機能（接続される側）**が標準で無効になっています。

本記事では、「RDP Wrapper」を使用してWindows 11 HomeでRDP接続を受け付けられるようにする手順を解説します。

## ⚠️ 注意事項

:::message alert
この方法は**Microsoftの公式サポート外**です。以下の点をご理解の上、自己責任でお願いします。

- Microsoftのライセンス規約に抵触する可能性があります
- Windows Updateにより動作しなくなる場合があります
- 業務利用の場合は、会社のIT部門や上長に確認することを推奨します
:::

正規の方法は「Windows Proへのアップグレード（約13,000円〜）」です。

## 必要なファイル

以下の2つをダウンロードします。

### 1. RDP Wrapper本体

**ダウンロード先**: https://github.com/stascorp/rdpwrap/releases/tag/v1.6.2

→ `RDPWrap-v1.6.2.zip` をダウンロード

:::message
`RDPWInst-v1.6.2.msi` は動作しません。必ず**ZIP版**を使用してください。
:::

### 2. 最新のINIファイル

**ダウンロード先**: https://github.com/sebaxakerhtc/rdpwrap.ini

→ 緑の「Code」→「Download ZIP」、または以下から直接ダウンロード：
https://raw.githubusercontent.com/sebaxakerhtc/rdpwrap.ini/master/rdpwrap.ini

## インストール手順

### Step 1: ウイルス対策ソフトの除外設定

RDP WrapperはWindows Defenderや他のセキュリティソフトに「HackTool」として検知されることがあります。これはウイルスではなく、Windowsの制限を回避するツールであるための誤検知です。

**Windows Defenderの場合（PowerShellを管理者実行）:**

```powershell
Add-MpPreference -ExclusionPath "C:\Program Files\RDP Wrapper\"
```

**他のセキュリティソフトの場合:**
各ソフトの設定画面から `C:\Program Files\RDP Wrapper\` を除外リストに追加してください。

### Step 2: RDP Wrapperのインストール

1. `RDPWrap-v1.6.2.zip` を解凍
2. 解凍したフォルダ内の `install.bat` を**右クリック** → **「管理者として実行」**
3. コマンド画面が表示され、完了したらキーを押して終了

### Step 3: 動作確認（初回）

解凍したフォルダ内の `RDPConf.exe` を実行し、状態を確認します。

| 項目 | 正常な状態 |
|------|-----------|
| Wrapper state | Installed（緑） |
| Service state | Running（緑） |
| Listener state | Listening（緑） |

**Listener stateが「Not listening [not supported]」の場合** → Step 4へ進んでください。

### Step 4: INIファイルの更新

Windowsのバージョンに対応したINIファイルに差し替えます。

#### 4-1. Remote Desktop Servicesを停止

```powershell
net stop termservice /y
```

:::message
停止できない場合は**PCを再起動**してから、再起動直後に上記コマンドを実行してください。
:::

#### 4-2. INIファイルを差し替え

ダウンロードした `rdpwrap.ini` を以下のフォルダに**上書きコピー**：

```
C:\Program Files\RDP Wrapper\
```

#### 4-3. Remote Desktop Servicesを再開

```powershell
net start termservice
```

### Step 5: 最終確認

`RDPConf.exe` を再度実行し、すべて緑色になっていればOKです。

## 接続テスト

### ローカルでのテスト

`RDPCheck.exe` を実行すると、ローカルでRDP接続のテストができます。

### 他のPCからの接続

1. 接続する側のPCで「リモートデスクトップ接続」を起動
2. コンピューター名に**対象PCのIPアドレス**または**コンピューター名**を入力
3. 接続

## ファイアウォール設定（必要に応じて）

RDPポート（3389）を許可する場合：

```powershell
netsh advfirewall firewall add rule name="Remote Desktop" dir=in action=allow protocol=TCP localport=3389
```

## トラブルシューティング

### Q. Listener stateが「Not listening」のまま

**原因**: INIファイルがWindowsのバージョンに対応していない

**対処法**: 
1. https://github.com/sebaxakerhtc/rdpwrap.ini から最新のINIファイルを取得
2. Step 4の手順でINIファイルを差し替え

### Q. サービスが停止できない

**対処法**: PCを再起動してから、再起動直後にコマンドを実行

### Q. ウイルスとして検知される

**原因**: セキュリティソフトによる誤検知

**対処法**: セキュリティソフトの除外設定に `C:\Program Files\RDP Wrapper\` を追加

### Q. Windows Update後に使えなくなった

**対処法**: 最新のINIファイルをダウンロードして差し替え

## セキュリティ対策

RDPを有効にすると、外部からの攻撃対象になる可能性があります。以下の対策を推奨します。

| 対策 | 説明 |
|------|------|
| 強力なパスワード | 推測されにくいパスワードを設定 |
| NLA有効化 | RDPConf.exeで「Network Level Authentication」を選択 |
| ファイアウォール | 特定のIPアドレスのみ許可 |
| VPN併用 | 可能であればVPN経由でのみアクセス |

## まとめ

Windows 11 HomeでもRDP Wrapperを使用することでリモートデスクトップ接続が可能になります。

ただし、非公式な方法であるため、Windows Updateのたびに動作確認とINIファイルの更新が必要になる場合があります。安定した運用が必要な場合は、Windows Proへのアップグレードをご検討ください。

## 参考リンク

- [RDP Wrapper (GitHub)](https://github.com/stascorp/rdpwrap)
- [最新INIファイル (GitHub)](https://github.com/sebaxakerhtc/rdpwrap.ini)

<!-- 再デプロイ用更新: 2026-01-25 -->

---
title: "【C#】コマンドプロンプトからWindowsサービスをインストール・アンインストールする方法"
emoji: "⚙️"
type: "tech"
topics: ["csharp", "windows", "windowsservice", "visualstudio", "備忘録"]
published: true
---

## はじめに

この記事では、コマンドプロンプトから `installutil.exe` を使用して、Windowsサービスをインストール・アンインストールする方法を解説します。

### 対象読者

- C#でWindowsサービスを開発している方
- Visual Studioを使用している方
- コマンドラインからサービスを管理したい方

### 前提条件

- Visual Studio 2022がインストールされていること
- .NET Frameworkで作成されたWindowsサービスプロジェクトがあること

:::message
`installutil.exe` は .NET Framework 用のツールです。.NET Core / .NET 5以降のサービスでは `sc.exe` コマンドの使用が推奨されます。
:::

## インストール手順

### 手順1：Developer Command Promptを起動

スタートメニューから「Developer Command Prompt for VS 2022」を検索して起動します。

:::message alert
**管理者として実行**する必要があります。右クリックして「管理者として実行」を選択してください。
:::

### 手順2：サービスのディレクトリに移動

```bash
cd C:\path\to\your\service\bin\Debug
```

### 手順3：インストールコマンドを実行

```bash
installutil.exe YourService.exe
```

### 手順4：インストール結果の確認

成功すると以下のようなメッセージが表示されます：

```
Microsoft (R) .NET Framework Installation utility Version 4.8.xxxx.x

...

トランザクション インストールが完了しました。
```

## アンインストール手順

### 手順1：Developer Command Promptを管理者として起動

インストール時と同様に、管理者権限で起動します。

### 手順2：サービスのディレクトリに移動

```bash
cd C:\path\to\your\service\bin\Debug
```

### 手順3：アンインストールコマンドを実行

`/u` オプションを付けて実行します：

```bash
installutil.exe /u YourService.exe
```

### 手順4：アンインストール結果の確認

成功すると以下のようなメッセージが表示されます：

```
Microsoft (R) .NET Framework Installation utility Version 4.8.xxxx.x

...

アンインストールが完了しました。
```

## コマンドまとめ

| 操作 | コマンド |
|------|----------|
| インストール | `installutil.exe YourService.exe` |
| アンインストール | `installutil.exe /u YourService.exe` |

## トラブルシューティング

### 「アクセスが拒否されました」エラー

**原因**：管理者権限がない

**解決策**：コマンドプロンプトを「管理者として実行」で起動し直す

### 「BadImageFormatException」エラー

**原因**：32bit/64bitの不一致

**解決策**：
- 32bitサービスの場合：x86用のDeveloper Command Promptを使用
- 64bitサービスの場合：x64用のDeveloper Command Promptを使用

### アンインストール後もサービスが残っている

**原因**：レジストリにサービス情報が残っている

**解決策**：`sc delete` コマンドで強制削除

```bash
sc delete YourServiceName
```

:::message alert
`sc delete` は強力なコマンドです。サービス名を間違えないよう注意してください。
:::

## 参考資料

- [Installutil.exe (インストーラー ツール) - Microsoft Learn](https://learn.microsoft.com/ja-jp/dotnet/framework/tools/installutil-exe-installer-tool)
- [Windows サービス アプリケーションの概要 - Microsoft Learn](https://learn.microsoft.com/ja-jp/dotnet/framework/windows-services/introduction-to-windows-service-applications)

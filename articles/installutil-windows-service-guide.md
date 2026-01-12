---
title: "【C#】コマンドプロンプトからWindowsサービスをインストール・アンインストールする方法"
emoji: "⚙️"
type: "tech"
topics: ["csharp", "windows", "windowsservice", "visualstudio", "備忘録"]
published: true
---

## はじめに

この記事では、**Developer Command Prompt for VS 2022** を使って、Windowsサービスを開発モードでインストール・アンインストールする方法を解説します。

### 対象読者

- C#でWindowsサービスを開発しているエンジニア
- 開発中にサービスの動作確認をしたい方
- インストーラーを作成せずに手軽にテストしたい方

### 前提条件

- Visual Studio 2022 がインストールされていること
- Windowsサービスプロジェクトがビルド済みであること
- 管理者権限でコマンドプロンプトを実行できること

:::message alert
**注意**: この方法は開発・デバッグ用です。本番環境へのデプロイには、適切なインストーラーを作成することを推奨します。
:::

## インストール手順

### 1. Developer Command Prompt を管理者権限で起動

1. スタートメニューから **「Developer Command Prompt for VS 2022」** を検索
2. 右クリック → **「管理者として実行」** を選択

![Developer Command Prompt の起動方法](/images/installutil-windows-service/developer-command-prompt.png)
*Developer Command Prompt for VS 2022 を右クリックして「管理者として実行」を選択*

:::message
管理者権限で起動しないと、「アクセスが拒否されました」というエラーが発生します。
:::

### 2. ビルドしたexeファイルのパスを確認

プロジェクトをビルドした場所のパスを確認します。

```
例）C:\udemy\source\Monaka\Monaka\bin\Debug
```

### 3. installutil.exe コマンドを実行

コマンドプロンプトで以下のコマンドを実行します。

```batch
cd C:\udemy\source\Monaka\Monaka\bin\Debug
installutil.exe Monaka.exe
```

| パラメータ | 説明 |
|-----------|------|
| `installutil.exe` | .NET Frameworkのインストーラーツール |
| `[サービス名].exe` | インストールするWindowsサービスの実行ファイル |

**成功時の出力例**:
```
トランザクション インストールが完了しました。
```

### 4. サービスがインストールされたか確認

1. **サービス管理ツール** を起動（`services.msc`）
2. インストールしたサービスが一覧に表示されているか確認
3. 表示されない場合は **F5キー** で最新の状態に更新

## アンインストール手順

### 1. Developer Command Prompt を管理者権限で起動

インストール時と同様に、管理者権限で起動します。

### 2. ビルドしたexeファイルのパスを確認

```
例）C:\udemy\source\Monaka\Monaka\bin\Debug
```

### 3. installutil.exe /u コマンドを実行

コマンドプロンプトで以下のコマンドを実行します。

```batch
cd C:\udemy\source\Monaka\Monaka\bin\Debug
installutil.exe /u Monaka.exe
```

| パラメータ | 説明 |
|-----------|------|
| `/u` | アンインストールオプション |
| `[サービス名].exe` | アンインストールするWindowsサービスの実行ファイル |

**成功時の出力例**:
```
アンインストールか完了しました。
```

### 4. サービスがアンインストールされたか確認

1. **サービス管理ツール** を起動（`services.msc`）
2. アンインストールしたサービスが一覧から消えているか確認
3. 表示が残っている場合は **F5キー** で最新の状態に更新

## コマンドまとめ

| 操作 | コマンド |
|------|---------|
| インストール | `installutil.exe [サービス名].exe` |
| アンインストール | `installutil.exe /u [サービス名].exe` |

## トラブルシューティング

### 「アクセスが拒否されました」エラー

- **原因**: 管理者権限でコマンドプロンプトを起動していない
- **対処法**: Developer Command Prompt を右クリック → 「管理者として実行」

### 「BadImageFormatException」エラー

- **原因**: 32bit/64bit のバージョン不一致
- **対処法**: 
  - 32bitアセンブリ → `Framework` フォルダの installutil.exe を使用
  - 64bitアセンブリ → `Framework64` フォルダの installutil.exe を使用

### サービスが一覧に表示されない

- **原因**: サービスの一覧が古い情報のまま
- **対処法**: F5キーを押して最新の状態に更新

### レジストリにサービスが残ってしまった場合

実行ファイルを削除した後もレジストリにサービスが残っている場合は、以下のコマンドで削除できます。

```batch
sc delete [サービス名]
```

## 参考資料

- [方法: Windows サービスをインストールおよびアンインストールする - Microsoft Learn](https://learn.microsoft.com/ja-jp/dotnet/framework/windows-services/how-to-install-and-uninstall-services)
- [Installutil.exe (インストーラー ツール) - Microsoft Learn](https://learn.microsoft.com/ja-jp/dotnet/framework/tools/installutil-exe-installer-tool)

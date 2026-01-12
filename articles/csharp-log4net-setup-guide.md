---
title: "【C#】Log4netの導入と設定ガイド - ログ出力を簡単に実装する方法"
emoji: "📝"
type: "tech"
topics: ["csharp", "dotnet", "log4net", "logging", "初心者向け"]
published: true
---

## はじめに

この記事では、C#でログ出力を簡単に実装できる **Log4net** の導入方法と設定について解説します。

### Log4netとは？

- **ログ出力を簡単に実装できるパッケージ**
- ログの処理（ファイル出力、ローテーション、フォーマットなど）を一から書く必要がない
- Apache Software Foundationが提供する信頼性の高いライブラリ

:::message
例えるなら、Log4netは「ログ出力の便利キット」です。自分で一からログの仕組みを作る代わりに、設定ファイルを書くだけで高機能なログ出力が使えるようになります。
:::

## インストール方法

### NuGetからインストール

1. Visual Studioで **ツール** → **NuGet パッケージ マネージャー** → **ソリューションの NuGet パッケージの管理** を開く
2. 検索バーに `Log4net` を入力
3. 検索結果から **log4net** を選択してインストール

## 設定の追加

### 1. AssemblyInfo.csの編集

`Properties/AssemblyInfo.cs` ファイルの **一番下** に以下のコードを追加します。

```csharp
[assembly: log4net.Config.XmlConfigurator(Watch = true)]
```

:::message
この設定により、Log4netが `App.config` の設定を自動的に読み込むようになります。`Watch = true` は設定ファイルの変更を監視する設定です。
:::

### 2. App.configの設定

`App.config` ファイルに以下の設定を追加します。

```xml
<?xml version="1.0" encoding="utf-8" ?>
<configuration>
  <configSections>
    <section name="log4net"
     type="log4net.Config.Log4NetConfigurationSectionHandler,log4net" />
  </configSections>
  
  <log4net>
    <appender name="tryLogAppender" type="log4net.Appender.RollingFileAppender">
      <File value=".\log\" />
      <DatePattern value='yyyyMMdd".log"' />
      <StaticLogFileName value="false" />
      <RollingStyle value="date" />
      <AppendToFile value="true" />
      <MaximumFileSize value="100MB" />
      <MaxSizeRollBackups value="30" />
      <layout type="log4net.Layout.PatternLayout">
        <ConversionPattern value="%date [%thread] [%-5level] (%method) - %message%n" />
      </layout>
    </appender>
    
    <root>
      <level value="Debug" />
      <appender-ref ref="tryLogAppender" />
    </root>
  </log4net>
  
  <startup>
    <supportedRuntime version="v4.0" sku=".NETFramework,Version=v4.7.2" />
  </startup>
  
  <appSettings>
  </appSettings>
</configuration>
```

### 設定項目の解説

| 設定項目 | 説明 |
|---------|------|
| `File` | ログファイルの出力先フォルダ |
| `DatePattern` | ログファイル名のフォーマット（例：`20260112.log`） |
| `RollingStyle` | ログのローテーション方式（`date` = 日付でローテーション） |
| `MaximumFileSize` | 1ファイルの最大サイズ |
| `MaxSizeRollBackups` | 保持するバックアップファイル数 |
| `ConversionPattern` | ログの出力フォーマット |

### 3. クラスでの使用方法

ログを出力したいクラスで、以下のように宣言します。

```csharp
private static log4net.ILog _logger = log4net.LogManager.GetLogger(
    System.Reflection.MethodBase.GetCurrentMethod().DeclaringType);
```

### ログの出力例

```csharp
// 各レベルでのログ出力
_logger.Debug("デバッグ情報");
_logger.Info("情報メッセージ");
_logger.Warn("警告メッセージ");
_logger.Error("エラーメッセージ");
_logger.Fatal("致命的なエラー");
```

## コンソールにもログを表示させる方法

デバッグ時にコンソールでもログを確認したい場合は、`App.config` を以下のように修正します。

```xml
<?xml version="1.0" encoding="utf-8" ?>
<configuration>
  <configSections>
    <section name="log4net"
     type="log4net.Config.Log4NetConfigurationSectionHandler,log4net" />
  </configSections>
  
  <log4net>
    <!-- ファイル出力用 -->
    <appender name="tryLogAppender" type="log4net.Appender.RollingFileAppender">
      <File value=".\log\" />
      <DatePattern value='yyyyMMdd".log"' />
      <StaticLogFileName value="false" />
      <RollingStyle value="date" />
      <AppendToFile value="true" />
      <MaximumFileSize value="100MB" />
      <MaxSizeRollBackups value="30" />
      <layout type="log4net.Layout.PatternLayout">
        <ConversionPattern value="%date [%thread] [%-5level] (%method) - %message%n" />
      </layout>
    </appender>
    
    <!-- コンソール出力用 -->
    <appender name="Console" type="log4net.Appender.ConsoleAppender">
      <layout type="log4net.Layout.PatternLayout">
        <conversionPattern value="%d[%t] %p - %m%n"/>
      </layout>
    </appender>
    
    <!-- トレース出力用（Visual Studioの出力ウィンドウ） -->
    <appender name="Trace" type="log4net.Appender.TraceAppender">
      <layout type="log4net.Layout.PatternLayout">
        <conversionPattern value="%d[%t] %p - %m%n"/>
      </layout>
    </appender>
    
    <root>
      <level value="Debug" />
      <appender-ref ref="Console" />
      <appender-ref ref="Trace" />
      <appender-ref ref="tryLogAppender" />
    </root>
  </log4net>
  
  <startup>
    <supportedRuntime version="v4.0" sku=".NETFramework,Version=v4.7.2" />
  </startup>
  
  <appSettings>
  </appSettings>
</configuration>
```

### 追加したAppenderの説明

| Appender名 | 出力先 | 用途 |
|-----------|-------|------|
| `tryLogAppender` | ファイル | 本番環境でのログ保存 |
| `Console` | コンソール | コンソールアプリでのデバッグ |
| `Trace` | Visual Studio出力ウィンドウ | IDE上でのデバッグ |

## まとめ

- **Log4net** を使うとログ出力の実装が簡単になる
- NuGetからインストールし、`AssemblyInfo.cs` と `App.config` を設定するだけ
- ファイル出力、コンソール出力など複数の出力先を同時に設定可能
- ログレベル（Debug, Info, Warn, Error, Fatal）で出力を制御できる

## 参考資料

- [Apache log4net 公式サイト](https://logging.apache.org/log4net/)
- [log4net - NuGet Gallery](https://www.nuget.org/packages/log4net)

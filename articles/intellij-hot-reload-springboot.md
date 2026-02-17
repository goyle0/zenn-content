---
title: "【IntelliJ IDEA】毎回の再起動はもう不要！Spring Bootホットリロード設定"
emoji: "🔥"
type: "tech"
topics: ["intellij", "springboot", "java", "devtools"]
published: true
---

## はじめに

Spring Bootで開発していると、コードを少し変更するたびにアプリケーションを**停止→再起動**していませんか？

この記事では、IntelliJ IDEAでコードを保存するだけで変更が自動反映される「ホットリロード」の設定方法を紹介します。

**動作確認環境**

- IntelliJ IDEA 2025.2.4（Ultimate Edition）
- Spring Boot
- Windows 11

## 設定手順（全3ステップ）

### ステップ1：Spring Boot DevToolsを追加する

`pom.xml`に以下の依存関係を追加します。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <scope>runtime</scope>
    <optional>true</optional>
</dependency>
```

DevToolsは、コードの変更を検知して**アプリケーションを高速再起動**してくれるライブラリです。通常の再起動よりもかなり速く、数秒で変更が反映されます。

### ステップ2：プロジェクトの自動ビルドを有効にする

`ファイル` → `設定` → `ビルド、実行、デプロイ` → `コンパイラー`

**「自動的にプロジェクトをビルドする」** にチェックを入れます。

![](/images/intellij-hot-reload/compiler-settings.png)
*※画像は実際の設定画面に差し替えてください*

### ステップ3：実行中の自動Makeを許可する

`ファイル` → `設定` → `詳細設定`

コンパイラー欄にある **「開発対象のアプリケーションが実行中でも自動Makeの開始を可能にする」** にチェックを入れます。

![](/images/intellij-hot-reload/advanced-settings.png)
*※画像は実際の設定画面に差し替えてください*

:::message
この項目が見つからない場合は、設定画面の検索ボックスに **「自動Make」** と入力すると、すぐに見つかります。
:::

### 設定完了！

以上の3ステップで設定は完了です。アプリケーションを起動した状態でJavaファイルを編集・保存すると、数秒後に自動で変更が反映されます。

## 補足：手動でリロードする方法

DevToolsの自動リロードとは別に、IntelliJには**手動で変更を反映する方法**もあります。

アプリを**デバッグモード**（🪲アイコン）で起動している状態で、`Ctrl + F10`を押すと変更したクラスだけを入れ替えられます。

ただし、この方法は**メソッドの中身の変更だけ**に対応しています。メソッドの追加・削除やクラス構造の変更には対応していないため、基本的にはDevToolsの利用がおすすめです。

## 両方の方法の比較

| 項目 | DevTools（自動） | Ctrl+F10（手動） |
|------|------------------|-------------------|
| 対応範囲 | 広い（クラス追加なども可） | 狭い（メソッド内部のみ） |
| 反映方法 | 自動（保存するだけ） | 手動（Ctrl+F10） |
| 速度 | 数秒（高速再起動） | 一瞬（クラス入替のみ） |
| 導入の手間 | pom.xml変更＋IDE設定 | 設定不要 |

## 注意点

- DevToolsは`<scope>runtime</scope>`かつ`<optional>true</optional>`で追加しているため、**本番環境には含まれません**。安心して利用できます。
- 静的リソース（HTML、CSS、JSなど）の変更もDevToolsで自動反映されます。
- `application.properties`や`application.yml`を変更した場合は、DevToolsでも完全な再起動が必要です。

## まとめ

たった3ステップの設定で、開発中の「停止→起動」の手間がなくなります。

1. **pom.xmlにDevToolsを追加**
2. **「自動的にプロジェクトをビルドする」を有効化**
3. **「開発対象のアプリケーションが実行中でも自動Makeの開始を可能にする」を有効化**

地味な設定ですが、開発効率が大きく変わるので、まだ設定していない方はぜひ試してみてください。

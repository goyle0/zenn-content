---
title: "GmailのPOP廃止に伴うThunderbird移行手順まとめ"
emoji: "🦅"
type: "tech"
topics: ["thunderbird", "gmail", "email", "windows", "powershell"]
published: true
---

## はじめに

2026年1月より、GmailでのPOPによる「他のアカウントのメールを確認」機能が終了しました。
これにより、Gmailに会社メール（`your-address@example.com`）をPOP取得していた設定が使えなくなったため、Thunderbirdへ移行しました。

この記事では、移行手順とハマりやすいポイントをまとめます。

---

## 環境

| 項目 | 内容 |
|------|------|
| OS | Windows 11 |
| メールソフト | Mozilla Thunderbird（最新版） |
| メールアドレス | your-address@example.com |
| メールサーバー | お名前ドットコム（GMOサーバー） |
| 受信方式 | POP3 |

---

## Thunderbirdのインストールと初期設定

### 1. インストール

[Thunderbird公式サイト](https://www.thunderbird.net/ja/)からダウンロードしてインストールします。

### 2. アカウント設定

初回起動時にアカウント設定ウィザードが表示されます。

| 項目 | 設定値 |
|------|--------|
| 名前 | 自分の名前 |
| メールアドレス | your-address@example.com |
| 受信サーバー | pop.mailserver.example.com |
| 受信ポート | 995（SSL/TLS） |
| 送信サーバー | smtp.mailserver.example.com |
| 送信ポート | 465（SSL/TLS） |
| 認証方式 | 通常のパスワード認証 |

:::message
お名前ドットコムのメールサーバーは契約プランによって異なる場合があります。
コントロールパネルで正しいサーバー名を確認してください。
:::

---

## PowerShellスクリプトで自動設定

アカウント設定完了後、以下の設定をPowerShellスクリプトで一括適用しました。

- メールの振り分けルール（会社宛て・自分宛の分離）
- 署名の設定
- 迷惑メールフィルターの設定

### スクリプト実行の注意点

PowerShellスクリプトは **UTF-8 BOM付き** で保存する必要があります。
BOMなしで保存すると日本語が文字化けしてエラーになります。

```python
# Python で UTF-8 BOM付きで保存する方法
with open('script.ps1', 'w', encoding='utf-8-sig', newline='\r\n') as f:
    f.write(script_content)
```

また、batファイルで呼び出す場合は **ASCII文字のみ** で記述します。

```bat
@echo off
powershell.exe -ExecutionPolicy Bypass -NoExit -File "%~dp0script.ps1"
pause
```

### プロファイルフォルダについて

Thunderbirdのプロファイルフォルダは複数作成される場合があります。
実際に使われているプロファイルは以下の手順で確認します。

**ヘルプ → トラブルシューティング情報 → プロファイルフォルダー → フォルダーを開く**

環境によっては `xxxxxxxx.default-release` が実際のプロファイルになっています。
スクリプト内では `default-release` を含むプロファイルを優先して使用するよう実装しています。

---

## 振り分けルールの設定

### 振り分け方針

| フォルダ名 | 振り分け条件 |
|-----------|------------|
| 会社宛て | To/Ccにforward1@example.com、forward2@example.com、forward3@example.com、forward4@example.comが含まれるメール |
| 自分宛 | To/Ccにyour-address@example.comが含まれるメール |

### msgFilterRules.datの書き方

Thunderbirdの振り分けルールは `msgFilterRules.dat` というファイルで管理されています。
場所：`プロファイルフォルダ\Mail\サーバー名\msgFilterRules.dat`

```
version="9"
logging="no"

name="会社宛て（転送メール）"
enabled="yes"
type="17"
action="Move to folder"
actionValue="mailbox:///...パス.../会社宛て"
condition="OR (to,contains,forward1@example.com) (to,contains,forward3@example.com)"
```

:::message alert
`actionValue` のフォルダパスはURLエンコードが必要です。
日本語フォルダ名の場合、「会社宛て」→ `%E4%BC%9A%E7%A4%BE%E5%AE%9B%E3%81%A6` のように変換します。
:::

### ハマりポイント①：条件の設定

GUIで振り分けルールを編集した際、**「すべての条件に一致」** のままだと複数条件がAND扱いになりフィルターが動きません。

**「いずれかの条件に一致」** に変更することで正しくOR条件で動作します。

### ハマりポイント②：移動先フォルダの未設定

スクリプトでフォルダパスを書き込んでも、GUIで確認すると「フォルダーを選択してください...」と表示される場合があります。
この場合はGUIから手動でフォルダを選択し直す必要があります。

**メッセージフィルター → 編集 → 「フォルダーを選択してください...」▼ → フォルダを選択 → OK**

---

## 署名の設定

### 署名ファイルの作成

署名はテキストファイルとして保存し、Thunderbirdに読み込ませます。

```
-- 
自分の名前<your-address@example.com>

会社名　https://www.example.com
〒XXX-XXXX 住所
Tel XX-XXXX-XXXX  Fax XX-XXXX-XXXX
```

署名ファイルは **UTF-8（BOMなし）** で保存します。

### user.jsによる設定

`プロファイルフォルダ\user.js` に以下を記述することで署名を自動設定できます。

```javascript
// 署名設定
user_pref("mail.identity.id1.attach_signature", true);
user_pref("mail.identity.id1.sig_on_reply", true);
user_pref("mail.identity.id1.sig_on_forwarded", true);
user_pref("mail.identity.id1.htmlSigFormat", false);
user_pref("mail.identity.id1.sig_file", "C:\\...\\signatures\\signature.txt");
```

:::message
`user.js` はThunderbird起動時に `prefs.js` へ反映されます。
手動で設定変更してもuser.jsが優先されるため、設定完了後はuser.jsを削除または該当行をコメントアウトしてください。
:::

---

## 迷惑メールフィルターの設定

```javascript
// 迷惑メール設定
user_pref("mail.server.server1.spamLevel", 100);
user_pref("mail.server.server1.moveOnSpam", true);
user_pref("mail.server.server1.useWhiteList", true);
user_pref("mail.server.server1.whiteListAbURI", "jsaddrbook://abook.sqlite");
user_pref("mail.adaptivefilters.junk_threshold", 90);
```

アドレス帳に登録済みの連絡先は迷惑メール扱いされません（`useWhiteList`）。

---

## 過去メールへの振り分け適用

新しく受信するメールはルールが自動適用されますが、すでに受信トレイにあるメールには手動で適用が必要です。

**メニュー → ツール → メッセージフィルター → フィルターを選択 → 今すぐ実行**

:::message
「今すぐ実行」ボタンを押す前に、リスト上のフィルター名を**クリックして青く選択した状態**にする必要があります。
選択していない状態では「今すぐ実行」が反応しません。
:::

---

## まとめ・ハマりポイント一覧

| ポイント | 内容 |
|---------|------|
| PowerShellスクリプトの文字コード | UTF-8 BOM付きで保存しないと文字化けエラー |
| batファイルの文字コード | ASCII文字のみ使用（日本語不可） |
| プロファイルが複数ある | `default-release` を含むフォルダが実際のプロファイル |
| フィルターが動かない① | 「すべての条件に一致」→「いずれかの条件に一致」に変更 |
| フィルターが動かない② | 移動先フォルダが未設定→GUIから手動で選択 |
| 今すぐ実行が動かない | フィルター名をクリックして選択状態にしてから実行 |
| user.jsの反映 | Thunderbird起動時にprefs.jsへ反映される |

---

## 参考リンク

- [Thunderbird公式サイト](https://www.thunderbird.net/ja/)
- [Gmail POP廃止のお知らせ（Google公式）](https://support.google.com/mail/answer/16604719)
- [お名前ドットコム メール設定](https://help.onamae.com/answer/8401)

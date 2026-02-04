---
title: "RDP WrapperでListener stateが「Not listening [not supported]」になった時の対処法"
emoji: "🖥️"
type: "tech"
topics: ["windows", "rdp", "remotedesktop", "windows11"]
published: true
---

## 症状

RDP Wrapper（v1.6.2）を使用中、ある日突然リモートデスクトップ接続ができなくなった。

`RDPConf.exe` で確認すると以下の状態：

| 項目 | 状態 |
|------|------|
| Wrapper state | ✅ Installed |
| Service state | ✅ Running |
| Listener state | ❌ Not listening [not supported] |

## 原因

Windows Updateによりビルド番号が変わり、RDP Wrapperの設定ファイル（`rdpwrap.ini`）が現在のバージョンに対応していないことが原因。

## 対処法

最新のINIファイルに差し替えるだけで解決する。

### 1. 最新のINIファイルをダウンロード

以下から `rdpwrap.ini` をダウンロードする。

https://raw.githubusercontent.com/sebaxakerhtc/rdpwrap.ini/master/rdpwrap.ini

GitHub リポジトリ：https://github.com/sebaxakerhtc/rdpwrap.ini

### 2. サービスを停止

コマンドプロンプトを **管理者として実行** し、以下を実行する。

```bat
net stop termservice /y
```

:::message
停止できない場合はPCを再起動してから、再起動直後にこのコマンドを実行する。
:::

### 3. INIファイルを上書き

ダウンロードした `rdpwrap.ini` を以下のフォルダに上書きコピーする。

```
C:\Program Files\RDP Wrapper\
```

### 4. サービスを再開

```bat
net start termservice
```

### 5. 確認

`RDPConf.exe` を開き、3つとも緑になっていればOK。

| 項目 | 状態 |
|------|------|
| Wrapper state | ✅ Installed |
| Service state | ✅ Running |
| Listener state | ✅ Listening |

## 補足

Windows Updateのたびに同じ現象が発生する可能性がある。その場合はINIファイルの差し替えを再度実施すればよい。

安定した運用が必要な場合は、Windows Proへのアップグレードも検討する。

## 参考

- [RDP Wrapper (GitHub)](https://github.com/stascorp/rdpwrap)
- [最新INIファイル (GitHub)](https://github.com/sebaxakerhtc/rdpwrap.ini)

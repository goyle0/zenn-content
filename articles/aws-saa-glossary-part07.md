---
title: "【AWS SAA-C03】完全用語集 Part 7: AI/ML、分析、移行、コスト管理"
emoji: "🤖"
type: "tech"
topics: ["aws", "saa", "資格試験", "kinesis", "migration"]
published: false
---

> **シリーズ**: AWS SAA-C03 試験対策 完全用語集（全10パート）
> **Part 7**: AI・機械学習サービス、分析 & データ連携、移行 & ハイブリッド、コスト管理 & 請求

---

## 8. AI・機械学習サービス

### AI サービス一覧（SAA頻出）

| サービス | 機能 | 試験のポイント |
|----------|------|---------------|
| **Textract** | ドキュメントからテキスト・表を抽出（OCR） | 「領収書」「申請書」のデータ化 |
| **Rekognition** | 画像・動画の分析（顔認識、物体検知） | 「画像検索」「顔認証」 |
| **Transcribe** | 音声→テキスト変換 | 「通話ログのテキスト化」「字幕」 |
| **Polly** | テキスト→音声変換 | 「読み上げ機能」 |
| **Translate** | 機械翻訳 | 「多言語対応」 |
| **Comprehend** | 自然言語処理（感情分析、キーワード抽出） | 「SNSの感情分析」 |
| **Lex** | チャットボット構築（Alexaの技術） | 「対話型インターフェース」 |
| **Kendra** | 機械学習ベースの企業内検索 | 「自然言語での質問応答」 |
| **Forecast** | 時系列データの需要予測 | 「売上予測」「在庫予測」 |
| **Fraud Detector** | オンライン不正検出 | 「偽アカウント」「支払い詐欺」 |

> 🎯 **試験のポイント**: キーワードとサービスを1対1で覚える
>
> - 「OCR」「フォーム」→ **Textract**
> - 「顔認識」「画像分析」→ **Rekognition**
> - 「音声をテキストに」→ **Transcribe**
> - 「感情分析」→ **Comprehend**

---

## 9. 分析 & データ連携

### Amazon Kinesis

**概要**: ストリーミングデータをリアルタイムで収集・処理・分析。

| サービス | 用途 | 特徴 | 試験のポイント |
|----------|------|------|---------------|
| **Data Streams** | リアルタイム処理 | ミリ秒、シャード管理必要 | カスタムアプリでの処理 |
| **Data Firehose** | データの配信・保存 | ニアリアルタイム、フルマネージド | S3/Redshiftへのロード |
| **Data Analytics** | ストリーム分析 | SQLまたはFlink | リアルタイム集計 |
| **Video Streams** | 動画ストリーミング | - | 監視カメラ等 |

#### Kinesis Data Streams の詳細

| 項目 | 説明 | 試験のポイント |
|------|------|---------------|
| **シャード** | 処理能力の単位 | 容量不足時はシャード分割 |
| **リシャーディング** | シャードの分割/結合 | スケールアウト/イン |
| **KPL** | プロデューサーライブラリ | 効率的なデータ送信 |
| **KCL** | コンシューマーライブラリ | 効率的なデータ受信 |

> 🎯 **試験のポイント**:
>
> - 「リアルタイム処理（ミリ秒）」→ **Data Streams**
> - 「S3/Redshiftへの配信（サーバーレス）」→ **Data Firehose**

---

### Amazon MSK (Managed Streaming for Apache Kafka)

**概要**: フルマネージドなApache Kafkaサービス。

> 🎯 **試験のポイント**: 「既存のKafkaアプリをコード変更なしでAWSに移行」→ **MSK**

---

### AWS Glue

**概要**: サーバーレスのETL（抽出・変換・ロード）サービス。

| 機能 | 説明 | 試験のポイント |
|------|------|---------------|
| **Data Catalog** | メタデータ管理 | Athena/Redshift Spectrumで検索可能に |
| **ETLジョブ** | データの変換処理 | S3→Redshiftのロード前処理 |
| **クローラー** | データソースを自動検出 | スキーマ自動生成 |

> 🎯 **試験のポイント**: 「S3のデータをAthenaで検索可能に」→ **Glue Data Catalog**

---

### Amazon Athena

**概要**: S3内のデータに対してSQLを実行するサーバーレス分析サービス。

> 🎯 **試験のポイント**: 「S3のログをSQLで分析（サーバーレス）」→ **Athena**

---

### Amazon EMR (Elastic MapReduce)

**概要**: Hadoop、Spark、Hive等のオープンソースフレームワークを使用したビッグデータ処理。

> 🎯 **試験のポイント**: 「既存のHadoop/Sparkワークロードを移行」→ **EMR**

---

### AWS Lake Formation

**概要**: S3上にデータレイクを安全に構築・管理。Glueと連携。

> 🎯 **試験のポイント**: 「データレイクの構築と権限管理を簡素化」→ **Lake Formation**

---

### Amazon QuickSight

**概要**: サーバーレスのBI（ビジネスインテリジェンス）ツール。データ可視化、ダッシュボード作成。

> 🎯 **試験のポイント**: 「データの可視化」「ダッシュボード」→ **QuickSight**

---

### Amazon AppFlow

**概要**: Salesforce、Slack等のSaaSとAWS間でデータを転送。コード不要。

> 🎯 **試験のポイント**: 「SaaSデータをS3/Redshiftに取り込み（コード不要）」→ **AppFlow**

---

### Amazon OpenSearch Service (旧 Elasticsearch Service)

**概要**: ログ分析、全文検索、可視化のためのマネージドサービス。

> 🎯 **試験のポイント**: 「ログ分析」「全文検索」→ **OpenSearch**

---

## 10. 移行 & ハイブリッド

### 移行ツール一覧

| サービス | 用途 | 試験のポイント |
|----------|------|---------------|
| **Migration Hub** | 移行の進捗を一元管理 | 複数ツールのダッシュボード |
| **Application Discovery Service** | オンプレ環境の情報収集 | 依存関係の可視化 |
| **Application Migration Service (MGN)** | サーバーのリフト＆シフト | SMSの後継 |
| **Database Migration Service (DMS)** | DBの移行 | 同種・異種DB間 |
| **Schema Conversion Tool (SCT)** | DBスキーマの変換 | 異種DB移行時のスキーマ変換 |

> 🎯 **試験のポイント**:
>
> - 「サーバー移行」→ **MGN**
> - 「DB移行」→ **DMS**
> - 「異種DB（Oracle→Aurora）のスキーマ変換」→ **SCT + DMS**

---

### AWS Transfer Family

**概要**: S3/EFSへのSFTP、FTPS、FTPアクセスを提供。

> 🎯 **試験のポイント**: 「既存のSFTPワークフローを維持してAWSへ」→ **Transfer Family**

---

### AWS DataSync

**概要**: オンプレミスとAWSストレージ間の大量データ転送を自動化。

> 🎯 **試験のポイント**: 「オンプレNFSからS3/EFSへの定期同期」→ **DataSync**

---

### AWS Data Exchange

**概要**: サードパーティのデータセットを検索・購入してS3に取り込み。

> 🎯 **試験のポイント**: 「外部データ（市場データ等）を取得」→ **Data Exchange**

---

### ハイブリッド＆エッジ

| サービス | 用途 | 試験のポイント |
|----------|------|---------------|
| **Outposts** | AWSインフラをオンプレに設置 | 「データレジデンシー要件」「超低遅延」 |
| **Wavelength** | 5Gネットワークのエッジ | 「5G端末向け超低遅延」 |
| **Local Zones** | 特定都市への拡張 | 「特定都市のユーザーへ低遅延」 |

---

### VMware Cloud on AWS

**概要**: AWS上にVMware vSphere環境を構築。

> 🎯 **試験のポイント**: 「VMware環境をそのままAWSへ」→ **VMware Cloud on AWS**

---

## 11. コスト管理 & 請求

### コスト管理ツール

| ツール | 用途 | 試験のポイント |
|--------|------|---------------|
| **Cost Explorer** | コストの可視化・将来予測 | トレンド分析、予測 |
| **Budgets** | 予算設定とアラート | 「予算超過の通知」 |
| **Cost and Usage Report (CUR)** | 最も詳細なコストデータ | Athena/Redshiftで分析 |
| **Cost Anomaly Detection** | 機械学習で異常検知 | 「予期せぬコスト増加の検出」 |
| **Cost Categories** | コストのグループ化 | プロジェクト/チーム別集計 |
| **Billing Conductor** | カスタム請求 | 再販業者向け |

> 🎯 **試験のポイント**:
>
> - 「コストの可視化」→ **Cost Explorer**
> - 「予算超過の通知」→ **Budgets**
> - 「異常なコスト増加の検出」→ **Cost Anomaly Detection**

---

### AWS Compute Optimizer

**概要**: 過去の使用状況を分析し、最適なEC2インスタンスタイプやEBSボリュームサイズを推奨。

> 🎯 **試験のポイント**: 「ライトサイジング」「過剰スペックのリソース特定」→ **Compute Optimizer**

---

### AWS サポートプラン

| プラン | 特徴 | Trusted Advisor |
|--------|------|-----------------|
| **Basic** | 無料、請求関連のみ | 一部のみ |
| **Developer** | 営業時間内メール | 一部のみ |
| **Business** | 24時間365日対応、1時間以内 | 全項目 |
| **Enterprise On-Ramp** | 15分以内対応 | 全項目 |
| **Enterprise** | TAM付き | 全項目 |

---

### AWS Chatbot

**概要**: Slack/Amazon ChimeとAWS連携。Budgetsアラート等の通知。

> 🎯 **試験のポイント**: 「予算アラートをSlackに通知」→ **Chatbot**

---

---
title: "【AWS SAA-C03】完全用語集 Part 8: 開発、設計原則、比較表"
emoji: "📋"
type: "tech"
topics: ["aws", "saa", "資格試験", "cloudformation", "devops"]
published: false
---

> **シリーズ**: AWS SAA-C03 試験対策 完全用語集（全10パート）
> **Part 8**: 開発・デプロイメント、設計原則 & 試験キーワード、頻出比較表 & 使い分け

---

## 12. 開発・デプロイメント

### AWS CloudFormation

**概要**: Infrastructure as Code（IaC）。JSON/YAMLテンプレートでAWSリソースを構築。

| 機能 | 説明 | 試験のポイント |
|------|------|---------------|
| **スタック** | テンプレートから作成されるリソース群 | 管理の単位 |
| **変更セット** | 更新前の影響をプレビュー | 本番適用前の確認 |
| **ドリフト検出** | 手動変更を検知 | 設定乖離の監査 |
| **StackSets** | 複数アカウント・リージョンに一括展開 | 組織全体への適用 |
| **ネストスタック** | テンプレートの再利用 | 共通構成の部品化 |

---

### AWS CodeSeries (CI/CD)

| サービス | 機能 | 試験のポイント |
|----------|------|---------------|
| **CodeCommit** | Gitリポジトリ | ソース管理 |
| **CodeBuild** | ビルド・テスト | サーバーレスビルド |
| **CodeDeploy** | デプロイ自動化 | EC2/Lambda/ECSへ |
| **CodePipeline** | CI/CDパイプライン | ビルド→テスト→デプロイの自動化 |

---

### AWS OpsWorks

**概要**: Chef/Puppetを使用した構成管理サービス。

> 🎯 **試験のポイント**: 「既存のChefレシピを再利用」→ **OpsWorks**

---

### AWS CloudShell

**概要**: ブラウザベースのCLI環境。AWS CLIがプリインストール済み。

> 🎯 **試験のポイント**: 「ローカルにCLIをインストールせずにAWS操作」→ **CloudShell**

---

### AWS RAM (Resource Access Manager)

**概要**: 自分のアカウントのリソース（サブネット、Transit Gateway等）を他のアカウントと共有。

> 🎯 **試験のポイント**: 「サブネットを複数アカウントで共有」→ **RAM**

---

### AWS Service Catalog

**概要**: 承認されたITサービス（CloudFormationテンプレート）のカタログを管理・提供。

> 🎯 **試験のポイント**: 「承認された構成のみを展開させる（ガバナンス）」→ **Service Catalog**

---

### AWS License Manager

**概要**: ソフトウェアライセンス（Microsoft、Oracle等）の使用状況を管理。

> 🎯 **試験のポイント**: 「BYOLライセンスの上限管理」→ **License Manager**

---

### AWS Serverless Application Repository

**概要**: サーバーレスアプリケーションを検索・デプロイ・公開。

> 🎯 **試験のポイント**: 「既存のサーバーレスアプリを再利用」→ **Serverless Application Repository**

---

## 13. 設計原則 & 試験キーワード

### Well-Architected Framework の6本柱

| 柱 | 説明 | キーワード |
|-----|------|-----------|
| **運用上の優秀性** | 運用の自動化、継続的改善 | IaC、監視、インシデント対応 |
| **セキュリティ** | 最小権限、暗号化、監査 | IAM、KMS、CloudTrail |
| **信頼性** | 障害からの復旧、自動復旧 | マルチAZ、Auto Scaling |
| **パフォーマンス効率** | リソースの効率的な使用 | 適切なインスタンスタイプ |
| **コスト最適化** | 無駄の排除、適切な購入オプション | RI、スポット、ライトサイジング |
| **持続可能性** | 環境への影響の最小化 | 効率的なリソース使用 |

---

### SAA-C03 試験の4つのドメイン

| ドメイン | 配点 | 主要サービス |
|----------|------|-------------|
| **セキュリティ** | 30% | IAM、KMS、WAF、Shield、GuardDuty |
| **弾力性（高可用性）** | 26% | Auto Scaling、マルチAZ、Route 53 |
| **高性能** | 24% | CloudFront、ElastiCache、Aurora |
| **コスト最適化** | 20% | RI、スポット、S3ストレージクラス |

---

### 設計原則のキーワード

| 原則 | 説明 | 対策 |
|------|------|------|
| **単一障害点 (SPOF) の排除** | 1つの障害で全体が止まらない | マルチAZ、冗長化 |
| **疎結合** | コンポーネント間の依存を弱める | SQS、SNS、API Gateway |
| **ステートレス** | サーバーに状態を持たない | セッションはDynamoDB/ElastiCache |
| **水平スケーリング** | 台数を増減 | Auto Scaling、読み取りレプリカ |
| **垂直スケーリング** | スペックを変更 | インスタンスタイプ変更 |
| **弾力性** | 需要に応じて自動スケール | Auto Scaling |
| **プロビジョニング自動化** | インフラをコードで管理 | CloudFormation、Terraform |

---

### 試験でよく使われる表現と正解パターン

| 問題文のキーワード | 正解の方向性 |
|-------------------|-------------|
| 「最小限の運用オーバーヘッド」 | マネージドサービス、サーバーレス |
| 「コスト効率が最も高い」 | スポット、RI、S3 Intelligent-Tiering |
| 「高可用性」 | マルチAZ、Auto Scaling |
| 「低遅延」 | CloudFront、ElastiCache、Global Accelerator |
| 「疎結合」 | SQS、SNS、API Gateway |
| 「セキュリティのベストプラクティス」 | 最小権限、暗号化、IAMロール |
| 「コンプライアンス要件」 | Object Lock、CloudTrail、Config |
| 「災害復旧 (DR)」 | クロスリージョンレプリケーション、Global Database |

---

## 14. 頻出比較表 & 使い分け

### ストレージの使い分け

| 要件 | サービス |
|------|----------|
| オブジェクトストレージ | S3 |
| ブロックストレージ（EC2用） | EBS |
| 共有ファイルストレージ（Linux） | EFS |
| 共有ファイルストレージ（Windows） | FSx for Windows |
| HPC用高速ストレージ | FSx for Lustre |
| オンプレ連携 | Storage Gateway |
| 大量データの物理転送 | Snowファミリー |

---

### データベースの使い分け

| 要件 | サービス |
|------|----------|
| リレーショナルDB | RDS / Aurora |
| NoSQL（キーバリュー） | DynamoDB |
| インメモリキャッシュ | ElastiCache (Redis/Memcached) |
| データウェアハウス | Redshift |
| グラフDB | Neptune |
| 時系列DB | Timestream |
| 台帳DB（改ざん防止） | QLDB |
| MongoDB互換 | DocumentDB |

---

### メッセージングの使い分け

| 要件 | サービス |
|------|----------|
| キューイング（1対1） | SQS |
| Pub/Sub（1対多） | SNS |
| 順序保証が必要 | SQS FIFO / SNS FIFO |
| ActiveMQ/RabbitMQ移行 | Amazon MQ |
| Kafka移行 | Amazon MSK |
| ワークフロー | Step Functions |
| イベント駆動 | EventBridge |

---

### セキュリティサービスの使い分け

| 要件 | サービス |
|------|----------|
| 脅威検出 | GuardDuty |
| 脆弱性診断 | Inspector |
| S3の機密データ検出 | Macie |
| インシデント調査 | Detective |
| Webアプリ保護 | WAF |
| DDoS保護 | Shield |
| 暗号化キー管理 | KMS |
| シークレット管理（自動ローテーション） | Secrets Manager |
| 設定パラメータ管理 | Parameter Store |

---

### コスト最適化の使い分け

| 要件 | サービス/オプション |
|------|-------------------|
| 24時間稼働のEC2 | RI / Savings Plans |
| 中断可能なバッチ処理 | スポットインスタンス |
| アクセスパターン不明のS3 | Intelligent-Tiering |
| 長期アーカイブ | Glacier Deep Archive |
| ライトサイジング推奨 | Compute Optimizer |
| コスト異常検知 | Cost Anomaly Detection |

---

### 移行ツールの使い分け

| 移行対象 | サービス |
|----------|----------|
| サーバー（EC2へ） | Application Migration Service (MGN) |
| データベース | DMS |
| 異種DBのスキーマ変換 | SCT |
| VMware環境 | VMware Cloud on AWS |
| 大量データ（オフライン） | Snowファミリー |
| 大量データ（オンライン） | DataSync |

---

### ELBの使い分け

| 要件 | タイプ |
|------|--------|
| HTTP/HTTPS、パスベースルーティング | ALB |
| TCP/UDP、固定IP、超低遅延 | NLB |
| サードパーティアプライアンス | GLB |

---

### 接続方法の使い分け

| 要件 | 方法 |
|------|------|
| VPC間接続（シンプル） | VPCピアリング |
| VPC間接続（多数） | Transit Gateway |
| オンプレ接続（すぐに） | Site-to-Site VPN |
| オンプレ接続（安定・大容量） | Direct Connect |
| AWSサービスへのプライベート接続 | VPCエンドポイント |

---

### Route 53 ルーティングの使い分け

| 要件 | ポリシー |
|------|----------|
| 単一リソース | シンプル |
| A/Bテスト、段階的移行 | 加重 |
| 最も近いリージョン | レイテンシー |
| 障害時の切り替え | フェイルオーバー |
| 国/地域ごとのコンテンツ | 位置情報 |
| トラフィックシフト | 地理的近接性 |

---

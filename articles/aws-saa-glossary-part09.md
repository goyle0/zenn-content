---
title: "【AWS SAA-C03】完全用語集 Part 9: チェックリスト、略語、EC2/VPC詳細"
emoji: "✅"
type: "tech"
topics: ["aws", "saa", "資格試験", "試験対策", "暗記"]
published: false
---

> **シリーズ**: AWS SAA-C03 試験対策 完全用語集（全10パート）
> **Part 9**: 試験直前チェックリスト、略語集、EC2 & VPC 補足詳細

---

## 15. 試験直前チェックリスト

### 🔴 絶対に覚える数字

| 項目 | 値 |
|------|-----|
| Lambda最大実行時間 | **15分** |
| SQSメッセージ保持期間 | **最大14日** |
| S3オブジェクト最大サイズ | **5TB** |
| S3マルチパートアップロード推奨 | **100MB以上** |
| DynamoDB PITR保持期間 | **35日** |
| Aurora Backtrack最大期間 | **72時間** |
| CloudTrailデフォルト保持 | **90日** |
| EBSスナップショットArchive復元時間 | **24-72時間** |
| Route 53 Aliasレコード | **Zone Apexに使用可能** |
| RI/Savings Plans割引 | **最大72%** |
| スポットインスタンス割引 | **最大90%** |

---

### 🟡 よくある間違いポイント

| 間違い | 正解 |
|--------|------|
| CNAMEをZone Apexに使う | Aliasレコードを使う |
| VPCピアリングで推移的ルーティング | 推移的ルーティングは**不可** |
| DynamoDB PITRで既存テーブルに復元 | **新しいテーブル**に復元される |
| MFA Deleteを一般ユーザーが設定 | **ルートユーザー**のみ可能 |
| コードにアクセスキーを埋め込む | **IAMロール**を使用 |
| S3ブロックパブリックアクセスを無効にする | 有効にするのがベストプラクティス |
| リードレプリカで可用性向上 | リードレプリカは**パフォーマンス**向上 |
| マルチAZで読み取り性能向上 | マルチAZは**可用性**向上 |

---

### 🟢 試験テクニック

1. **「最小限の運用オーバーヘッド」→ マネージドサービスを選ぶ**
2. **「コスト効率」→ サーバーレス、適切な購入オプション**
3. **「高可用性」→ マルチAZ、Auto Scaling**
4. **「セキュリティ」→ 最小権限、暗号化、監査**
5. **選択肢に「セキュリティグループ」と「NACL」→ まずSGを疑う（ステートフル）**
6. **選択肢に「RDS」と「Aurora」→ 高可用性・高性能ならAurora**
7. **選択肢に「Kinesis」と「SQS」→ リアルタイムならKinesis、非同期ならSQS**

---

### 📝 最終確認事項

- [ ] 各サービスの「試験のポイント」を説明できる
- [ ] 比較表の使い分けを理解している
- [ ] 数字（制限値など）を覚えている
- [ ] よくある間違いを理解している
- [ ] 問題文のキーワードから正解の方向性がわかる

---

> **合格ライン**: 1000点満点中 **720点以上**
>
> **試験時間**: 130分
>
> **問題数**: 65問

---

**頑張ってください！🎉**

---

## 16. AWS 略語・頭字語 完全リスト

### 試験で頻出する略語（アルファベット順）

| 略語 | 正式名称 | 意味・用途 |
|------|----------|-----------|
| **ACL** | Access Control List | アクセス制御リスト |
| **ACM** | AWS Certificate Manager | SSL/TLS証明書の管理 |
| **AD** | Active Directory | ディレクトリサービス |
| **ALB** | Application Load Balancer | L7ロードバランサー |
| **AMI** | Amazon Machine Image | EC2のテンプレートイメージ |
| **API** | Application Programming Interface | アプリ間連携の仕組み |
| **ARN** | Amazon Resource Name | リソースの一意識別子 |
| **ASG** | Auto Scaling Group | EC2の自動スケーリンググループ |
| **AZ** | Availability Zone | データセンター群 |
| **BYOL** | Bring Your Own License | 自前ライセンス持ち込み |
| **CDN** | Content Delivery Network | コンテンツ配信ネットワーク |
| **CIDR** | Classless Inter-Domain Routing | IPアドレス範囲の表記法 |
| **CLB** | Classic Load Balancer | 旧型ロードバランサー |
| **CMK** | Customer Master Key | KMSの暗号化マスターキー |
| **CRR** | Cross-Region Replication | クロスリージョンレプリケーション |
| **DAX** | DynamoDB Accelerator | DynamoDBのキャッシュ |
| **DLQ** | Dead Letter Queue | 処理失敗メッセージの退避先 |
| **DMS** | Database Migration Service | DB移行サービス |
| **DNS** | Domain Name System | ドメイン名解決システム |
| **DR** | Disaster Recovery | 災害復旧 |
| **EBS** | Elastic Block Store | EC2用ブロックストレージ |
| **EC2** | Elastic Compute Cloud | 仮想サーバー |
| **ECS** | Elastic Container Service | コンテナオーケストレーション |
| **EFA** | Elastic Fabric Adapter | HPC向け高速ネットワーク |
| **EFS** | Elastic File System | 共有ファイルストレージ |
| **EIP** | Elastic IP | 固定パブリックIP |
| **EKS** | Elastic Kubernetes Service | マネージドKubernetes |
| **ELB** | Elastic Load Balancing | ロードバランサー全般 |
| **EMR** | Elastic MapReduce | ビッグデータ処理 |
| **ENA** | Elastic Network Adapter | 拡張ネットワークアダプタ |
| **ENI** | Elastic Network Interface | 仮想ネットワークカード |
| **ETL** | Extract, Transform, Load | データ抽出・変換・ロード |
| **FIFO** | First In, First Out | 先入れ先出し |
| **FIPS** | Federal Information Processing Standards | 米国政府のセキュリティ標準 |
| **GLB** | Gateway Load Balancer | L3ロードバランサー |
| **HA** | High Availability | 高可用性 |
| **HPC** | High Performance Computing | 高性能コンピューティング |
| **HSM** | Hardware Security Module | ハードウェアセキュリティモジュール |
| **HTTP** | HyperText Transfer Protocol | Web通信プロトコル |
| **HTTPS** | HTTP Secure | 暗号化されたHTTP |
| **IaC** | Infrastructure as Code | インフラのコード化 |
| **IAM** | Identity and Access Management | ID・アクセス管理 |
| **IOPS** | Input/Output Operations Per Second | 1秒あたりのI/O操作数 |
| **IGW** | Internet Gateway | インターネットゲートウェイ |
| **KMS** | Key Management Service | 暗号化キー管理 |
| **MFA** | Multi-Factor Authentication | 多要素認証 |
| **MGN** | Application Migration Service | サーバー移行サービス |
| **NACL** | Network Access Control List | サブネットレベルのACL |
| **NAT** | Network Address Translation | アドレス変換 |
| **NFS** | Network File System | ネットワークファイル共有 |
| **NLB** | Network Load Balancer | L4ロードバランサー |
| **OAC** | Origin Access Control | CloudFrontのS3アクセス制御（新） |
| **OAI** | Origin Access Identity | CloudFrontのS3アクセス制御（旧） |
| **OIDC** | OpenID Connect | 認証プロトコル |
| **OU** | Organizational Unit | Organizations の組織単位 |
| **PITR** | Point-In-Time Recovery | 任意時点復旧 |
| **RCU** | Read Capacity Unit | DynamoDB読み取り単位 |
| **RDS** | Relational Database Service | マネージドRDB |
| **RI** | Reserved Instance | リザーブドインスタンス |
| **RPO** | Recovery Point Objective | 目標復旧時点 |
| **RTO** | Recovery Time Objective | 目標復旧時間 |
| **S3** | Simple Storage Service | オブジェクトストレージ |
| **SAML** | Security Assertion Markup Language | フェデレーション認証 |
| **SCP** | Service Control Policy | Organizations のポリシー |
| **SCT** | Schema Conversion Tool | DBスキーマ変換ツール |
| **SG** | Security Group | インスタンスレベルのFW |
| **SMB** | Server Message Block | Windowsファイル共有プロトコル |
| **SNS** | Simple Notification Service | プッシュ通知 |
| **SPOF** | Single Point of Failure | 単一障害点 |
| **SQS** | Simple Queue Service | メッセージキュー |
| **SRR** | Same-Region Replication | 同一リージョンレプリケーション |
| **SSL** | Secure Sockets Layer | 暗号化通信（旧） |
| **SSM** | Systems Manager | システム管理ツール群 |
| **SSO** | Single Sign-On | シングルサインオン |
| **STS** | Security Token Service | 一時的セキュリティ認証情報 |
| **TLS** | Transport Layer Security | 暗号化通信（現行） |
| **TTL** | Time To Live | 有効期限 |
| **VPC** | Virtual Private Cloud | 仮想ネットワーク |
| **VPN** | Virtual Private Network | 暗号化トンネル接続 |
| **VTL** | Virtual Tape Library | 仮想テープライブラリ |
| **WAF** | Web Application Firewall | Webアプリファイアウォール |
| **WCU** | Write Capacity Unit | DynamoDB書き込み単位 |
| **WORM** | Write Once Read Many | 一度書込み複数読取り |

---

## 17. EC2 / VPC 詳細機能

### EC2 ネットワーク拡張機能

| 機能 | 説明 | 試験のポイント |
|------|------|---------------|
| **ENA (Elastic Network Adapter)** | 最大100Gbpsの拡張ネットワーク | 高スループットが必要な場合 |
| **EFA (Elastic Fabric Adapter)** | HPC向け超低遅延ネットワーク | 「MPI」「ノード間通信」「HPC」 |
| **SR-IOV** | シングルルートI/O仮想化 | 高いネットワーク性能 |

> 🎯 **ENA vs EFA**:
>
> - **ENA**: 一般的な高性能ネットワーク
> - **EFA**: HPC専用、OSバイパス機能

### EC2 Instance Store（インスタンスストア）

**概要**: 物理サーバーに直接接続された一時ストレージ。

| 特徴 | 説明 |
|------|------|
| 超高速 | NVMe SSD、最高のIOPS |
| 一時的 | 停止/終了でデータ消失 |
| 無料 | インスタンス料金に含まれる |

> 🎯 **試験のポイント**: 「超高速」+「データ消失OK」→ **Instance Store**（キャッシュ、一時ファイル）

### EC2 Hibernate（休止）

**概要**: メモリ内容をEBSに保存して停止。再開時に素早く復帰。

| 条件 | 説明 |
|------|------|
| EBS暗号化必須 | メモリ内容を暗号化して保存 |
| 最大60日 | 休止可能な期間 |
| メモリ150GB以下 | インスタンスの制限 |

> 🎯 **試験のポイント**: 「起動時間を短縮したい」+「状態を保持」→ **Hibernate**

### VPC CIDR計算

| CIDR | サブネット数 | ホスト数/サブネット |
|------|-------------|-------------------|
| /16 | 1 | 65,536 |
| /20 | 16 | 4,096 |
| /24 | 256 | 256 |
| /28 | 4,096 | 16 |

> ⚠️ **注意**: AWSでは各サブネットで5つのIPが予約される（最初の4つ + 最後の1つ）

### VPC フローログの詳細

| フィールド | 説明 | 分析ポイント |
|-----------|------|-------------|
| srcaddr | 送信元IP | 不正アクセス元の特定 |
| dstaddr | 宛先IP | 通信先の確認 |
| action | ACCEPT/REJECT | 拒否された通信の調査 |
| packets | パケット数 | トラフィック量の分析 |

> 🎯 **試験のポイント**: 「通信が拒否される原因を調査」→ **VPC Flow Logs で REJECT を確認**

---

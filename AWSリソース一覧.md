# AWSリソース一覧

MineDevOpsプロジェクトで使用するAWSリソースの詳細一覧

## コンピューティング

### Amazon ECS (Elastic Container Service)
- **クラスター名**: minecraft-cluster
- **用途**: Minecraftサーバーのオーケストレーション
- **起動タイプ**: Fargate

### AWS Fargate
- **用途**: サーバーレスコンテナプラットフォーム
- **タスク定義**: minecraft-user-{user_id}
- **設定**:
  - CPU/メモリ使用量ベースの自動スケーリング
  - オンデマンドでのタスク作成・終了

### ECS Task Definition
- **命名規則**: minecraft-user-{user_id}
- **コンテナ設定**:
  - コンテナ名: minecraft-server
  - 環境変数: USER_ID, SERVER_NAME, GITHUB_REPO

## ネットワーキング

### Amazon VPC
- **CIDR**: 10.0.0.0/16
- **用途**: ネットワーク分離とセキュリティ

### サブネット
- **パブリックサブネット**: 10.0.1.0/24 (254個のIP)
  - 用途: Velocityプロキシサーバー
- **プライベートサブネット**: 
  - 10.0.2.0/22 (1022個のIP - Fargateタスク用)
  - 10.0.6.0/24 (254個のIP - データベース・EFS用)

### セキュリティグループ
- **sg-velocity-server**
  - Inbound: ポート25565 (0.0.0.0/0から)
  - Outbound: プライベートサブネットへの通信
  
- **sg-minecraft-servers**
  - Inbound: ポート25565 (Velocity SGからのみ)
  - Outbound: 必要な通信のみ

### Internet Gateway
- **用途**: パブリックサブネットのインターネットアクセス

### NAT Gateway
- **用途**: プライベートサブネットのアウトバウンド通信
- **配置**: パブリックサブネット内

## ストレージ

### ストレージレス設計
- **永続化なし**: 毎回新しいワールドでクリーンスタート
- **一時的な開発環境**: セッション終了時にデータは削除
- **高速起動**: ストレージマウント不要で即座に開始

## 監視・運用

### Amazon CloudWatch
- **メトリクス**:
  - ECSタスクのCPU/メモリ使用率
  - Fargateタスクの実行状況
  - ネットワークI/O
- **ログ**:
  - コンテナログ
  - Minecraftサーバーログ
- **アラーム**: リソース使用量の閾値監視

## セキュリティ・アクセス管理

### AWS IAM
- **ECSタスク実行ロール**:
  - ECS タスク実行権限
  - CloudWatch ログ書き込み権限
- **ECS サービスロール**:
  - タスク管理権限
  - ロードバランサー連携権限
- **アプリケーション用ロール**:
  - ECS API呼び出し権限
  - タスク起動・停止権限

### アクセスキー管理
- **環境変数**:
  - AWS_ACCESS_KEY_ID
  - AWS_SECRET_ACCESS_KEY
  - AWS_DEFAULT_REGION: ap-northeast-1

## データベース (推定)

### Amazon RDS または Aurora
- **用途**: 
  - ユーザー管理
  - サーバー状態管理
  - 利用統計管理
  - 課金・利用時間管理
- **エンジン**: MySQL または PostgreSQL (要確認)

## ネットワーク構成図

```
Internet
    ↓
Internet Gateway
    ↓
Public Subnet (10.0.1.0/24)
    ↓
Velocity Proxy Server (固定パブリックIP)
    ↓
Private Subnet - Compute (10.0.2.0/22) ※1000台分対応
    ↓
Fargate Tasks (Minecraft Servers)
    ↓
Private Subnet - Data (10.0.6.0/24)
    ↓
Amazon RDS (ユーザー・課金データ)
```

## コスト最適化

- **オンデマンド起動**: ユーザーアクセス時のみサーバー起動
- **自動停止**: 時間制限による自動サーバー停止
- **Fargate**: サーバーレスによる従量課金
- **ストレージレス**: EFS不要でストレージコスト削減

## 運用考慮事項

- **スケーラビリティ**: Fargateの自動スケーリング
- **セキュリティ**: VPCによるネットワーク分離
- **可用性**: マルチAZ構成での冗長化
- **監視**: CloudWatchによる包括的監視

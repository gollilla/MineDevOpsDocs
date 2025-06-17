# CLAUDE.md

このファイルは、このリポジトリでコードを作業する際のClaude Code (claude.ai/code) へのガイダンスを提供します。

## プロジェクト概要

MineDevOpsは、ユーザーにMinecraft開発サーバーを提供するPaaSサービスです。GitHubプッシュから自動ビルド・デプロイを行い、サーバーライフサイクル（アクセス時開始、アイドル時停止）を管理します。

## 技術スタック

### Webアプリケーション
- **バックエンド**: Laravel (最新版) + Laravel Sail（コンテナ化）
- **フロントエンド**: Vue.js + InertiaJS
- **スタイリング**: TailwindCSS + DaisyUIコンポーネントライブラリ
- **認証**: Laravel Breeze
- **ビルドツール**: Vite

### Minecraftサーバーインフラ
- **コンテナプラットフォーム**: AWS Fargate
- **データストレージ**: ストレージレス設計（一時的な開発環境）
- **プロキシサーバー**: Velocity Proxy Server（固定パブリックエンドポイント）
- **ネットワーク**: AWS VPC（パブリック・プライベートサブネット構成）
- **オーケストレーション**: AWS ECS + PHP AWS SDK連携
- **スケーリング**: オンデマンドサーバー作成・終了
- **コンテナイメージ** itzg/minecraft-server

## 開発環境セットアップ

### 前提条件
- Docker および Docker Compose（Laravel Sail用）
- Node.js および npm


### AWS統合セットアップ

**必要な依存関係**
```bash
./vendor/bin/sail composer require aws/aws-sdk-php
```

**環境設定**
```env
AWS_ACCESS_KEY_ID=your_access_key
AWS_SECRET_ACCESS_KEY=your_secret_key
AWS_DEFAULT_REGION=ap-northeast-1
AWS_ECS_CLUSTER=minecraft-cluster
```

**サービス実装例**
```php
// app/Services/MinecraftServerService.php
use Aws\Ecs\EcsClient;

class MinecraftServerService 
{
    private EcsClient $ecsClient;
    
    public function __construct()
    {
        $this->ecsClient = new EcsClient([
            'region' => config('aws.region'),
            'version' => 'latest'
        ]);
    }
    
    public function startUserServer(User $user): string
    {
        $result = $this->ecsClient->runTask([
            'cluster' => config('aws.ecs_cluster'),
            'taskDefinition' => "minecraft-user-{$user->id}",
            'launchType' => 'FARGATE',
            'networkConfiguration' => [
                'awsvpcConfiguration' => [
                    'subnets' => ['subnet-private-fargate-1', 'subnet-private-fargate-2'],
                    'securityGroups' => ['sg-minecraft-servers'],
                    'assignPublicIp' => 'DISABLED'
                ]
            ],
            'overrides' => [
                'containerOverrides' => [[
                    'name' => 'minecraft-server',
                    'environment' => [
                        ['name' => 'USER_ID', 'value' => (string)$user->id],
                        ['name' => 'SERVER_NAME', 'value' => $user->server_name],
                        ['name' => 'GITHUB_REPO', 'value' => $user->github_repo]
                    ]
                ]]
            ]
        ]);
        
        return $result['tasks'][0]['taskArn'];
    }
    
    public function stopUserServer(string $taskArn): void
    {
        $this->ecsClient->stopTask([
            'cluster' => config('aws.ecs_cluster'),
            'task' => $taskArn,
            'reason' => 'User disconnected'
        ]);
    }
}
```

## コアビジネスロジック

### サーバー管理
- ユーザーはアカウントごとに1つのMinecraft開発サーバーを作成可能
- Webインターフェースを通じた手動サーバー開始・停止制御
- プッシュ時の自動ビルド・デプロイのためのGitHub連携


## Minecraftサーバーアーキテクチャ

### ネットワーク構成
```
Internet → Velocity Proxy (Public IP) → Fargate Tasks (Private Subnets)
```

**VPC設計**
- VPC: 10.0.0.0/16
- パブリックサブネット: 10.0.1.0/24 (Velocityサーバー)
- プライベートサブネット: 
  - 10.0.2.0/22 (1022個のIP - Fargateタスク用、1000台対応)
  - 10.0.6.0/24 (データベース用)

**セキュリティグループ**
- Velocityサーバー: Inbound 25565（0.0.0.0/0から）、Outbound（プライベートサブネットへ）
- Fargateタスク: Inbound 25565（Velocity SGからのみ）

### データ管理
**ストレージレス設計**
- **永続化なし**: 毎回新しいワールドでクリーンスタート
- **一時的な開発環境**: セッション終了時にデータは削除
- **高速起動**: ストレージマウント不要で即座に開始
- **コンテナ内一時ストレージ**: 実行中のみワールドデータを保持

### サーバーライフサイクル
1. **手動開始**: ユーザーがWebインターフェースからサーバー開始を実行
2. **動的開始**: ECSがユーザー固有設定でFargateタスクを実行
3. **プロキシ登録**: Velocityが新しいサーバーエンドポイントで更新
4. **手動停止**: ユーザーが完了時にWebインターフェースからサーバーを停止
5. **データ削除**: セッション終了と共にワールドデータは削除

### インフラ考慮事項
- **スケーリング**: CPU/メモリ使用量ベースのFargate自動スケーリング
- **セキュリティ**: 専用VPCサブネットによるコンテナ分離
- **監視**: タスクパフォーマンスのCloudWatchメトリクス
- **コスト最適化**: オンデマンドタスク作成・終了


### ドキュメントについて
- *.md, *.svg については更新時は各ドキュメントと整合性のチェックを行い、必要があれば修正してください。


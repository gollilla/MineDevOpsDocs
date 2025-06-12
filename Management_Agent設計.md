# Management Agent 設計書

## 概要

Management Agentは、各Minecraftサーバーホスト上で動作する軽量な管理プロセスです。Web Server 3（マスター）からの命令を受信し、ローカルでのコンテナ管理・監視・メトリクス収集を担当します。

## アーキテクチャ

### 基本構成
```
[Web Server 3 (Master)] 
         ↓ Redis Pub/Sub
[Management Agent] ← → [Docker Engine] ← → [Minecraft Containers]
         ↓ Metrics
[マスターサーバー]
```

### 通信方式
- **命令受信**: Redis Pub/Sub（マスター → Agent）
- **状態報告**: Redis Queue（Agent → マスター）
- **メトリクス送信**: Redis Queue（Agent → マスター）

## 主要機能

### 1. コマンド受信・実行
```php
class MinecraftManagementAgent
{
    public function listenForCommands()
    {
        $this->redis->subscribe(["host:{$this->hostId}:commands"], function($redis, $channel, $message) {
            $command = json_decode($message, true);
            
            switch ($command['action']) {
                case 'start_server':
                    $this->startMinecraftServer($command['config']);
                    break;
                case 'stop_server':
                    $this->stopMinecraftServer($command['server_id']);
                    break;
                case 'update_server':
                    $this->updateServerConfig($command['server_id'], $command['config']);
                    break;
                case 'collect_metrics':
                    $this->sendMetrics();
                    break;
            }
        });
    }
}
```

#### サポートするコマンド
- **start_server**: Minecraftサーバーコンテナの起動
- **stop_server**: サーバーの停止（Graceful Shutdown）
- **update_server**: サーバー設定の更新
- **collect_metrics**: メトリクス即座収集
- **restart_server**: サーバーの再起動
- **deploy_plugin**: プラグインのデプロイ

### 2. Minecraftサーバー管理

#### コンテナ起動
```php
private function startMinecraftServer(array $config): void
{
    $command = sprintf(
        'docker run -d --name mc-%s -p %d:25565 -m %s ' .
        '-e EULA=TRUE -e MAX_PLAYERS=%d -e VERSION=%s ' .
        '-v /data/mc-%s:/data ' .
        'minecraft/paper:%s',
        $config['server_id'],
        $config['port'],
        $config['memory'],
        $config['max_players'],
        $config['minecraft_version'],
        $config['server_id'],
        $config['minecraft_version']
    );
    
    $containerId = shell_exec($command);
    $this->reportServerStatus($config['server_id'], 'starting', trim($containerId));
}
```

#### 自動停止管理
```php
private function checkPlayerActivity(): void
{
    foreach ($this->getRunningServers() as $server) {
        $playerCount = $this->getPlayerCount($server['host'], $server['port']);
        
        if ($playerCount === 0) {
            $this->scheduleServerShutdown($server['id'], 30); // 30秒後停止
        } else {
            $this->cancelScheduledShutdown($server['id']);
        }
    }
}
```

### 3. メトリクス収集・送信

#### システムメトリクス
```php
private function sendMetrics(): void
{
    $metrics = [
        'host_id' => $this->hostId,
        'timestamp' => time(),
        'system' => [
            'cpu_usage' => $this->getCpuUsage(),           // CPU使用率
            'memory_usage' => $this->getMemoryUsage(),     // メモリ使用率
            'disk_usage' => $this->getDiskUsage(),         // ディスク使用率
            'network_io' => $this->getNetworkIO(),         // ネットワークI/O
            'load_average' => sys_getloadavg()             // システム負荷
        ],
        'containers' => $this->getContainerMetrics(),      // コンテナ別メトリクス
        'minecraft_servers' => $this->getMinecraftServerMetrics() // MC固有メトリクス
    ];
    
    $this->redis->lpush('metrics:queue', json_encode($metrics));
}
```

#### Minecraft固有メトリクス
```php
private function getMinecraftServerMetrics(): array
{
    $servers = [];
    
    foreach ($this->getRunningServers() as $server) {
        $servers[] = [
            'server_id' => $server['id'],
            'status' => $this->pingMinecraftServer($server['host'], $server['port']),
            'player_count' => $this->getPlayerCount($server['host'], $server['port']),
            'player_list' => $this->getPlayerList($server['host'], $server['port']),
            'tps' => $this->getServerTPS($server['host'], $server['port']),
            'uptime' => $this->getContainerUptime($server['container_id']),
            'memory_usage' => $this->getContainerMemory($server['container_id'])
        ];
    }
    
    return $servers;
}
```

### 4. ヘルスチェック・自動復旧

#### サーバーヘルスチェック
```php
private function checkMinecraftServerHealth(): void
{
    foreach ($this->getRunningServers() as $server) {
        $isHealthy = $this->pingMinecraftServer($server['host'], $server['port']);
        
        if (!$isHealthy) {
            $this->incrementFailureCount($server['id']);
            
            if ($this->getFailureCount($server['id']) >= 3) {
                $this->attemptServerRestart($server['id']);
                $this->resetFailureCount($server['id']);
            }
        } else {
            $this->resetFailureCount($server['id']);
        }
    }
}
```

#### 自動復旧処理
```php
private function attemptServerRestart(string $serverId): void
{
    Log::warning("Attempting restart for server: $serverId");
    
    // 1. Graceful shutdown試行
    $this->stopContainer("mc-$serverId", 30);
    
    // 2. 強制停止（必要な場合）
    if ($this->isContainerRunning("mc-$serverId")) {
        $this->killContainer("mc-$serverId");
    }
    
    // 3. データ整合性チェック
    $this->checkServerData($serverId);
    
    // 4. 再起動
    $config = $this->getServerConfig($serverId);
    $this->startMinecraftServer($config);
    
    // 5. マスターに報告
    $this->reportServerStatus($serverId, 'restarted', 'auto_recovery');
}
```

## インストール・デプロイ

### 1. 自動インストールスクリプト
```bash
#!/bin/bash
# install-agent.sh

# 依存関係インストール
apt-get update
apt-get install -y php php-redis docker.io

# Agentファイル配置
curl -L https://your-domain.com/management-agent.tar.gz | tar -xz -C /opt/

# 設定ファイル作成
cat > /opt/minecraft-agent/config.json << EOF
{
    "host_id": "$(hostname)",
    "master_server": "${MASTER_SERVER_IP}",
    "redis_port": 6379,
    "metrics_interval": 30,
    "health_check_interval": 30,
    "max_containers": 10,
    "auto_restart_attempts": 3
}
EOF

# Systemdサービス登録
systemctl enable minecraft-agent
systemctl start minecraft-agent
```

### 2. Docker化Agent（推奨）
```dockerfile
# Dockerfile
FROM php:8.2-cli

RUN apt-get update && apt-get install -y \
    docker.io \
    && docker-php-ext-install pcntl posix

COPY agent/ /opt/minecraft-agent/
COPY config.json /opt/minecraft-agent/

WORKDIR /opt/minecraft-agent
CMD ["php", "agent.php"]
```

### 3. 設定ファイル
```json
{
    "host_id": "minecraft-host-001",
    "master_server": "10.0.1.100",
    "redis": {
        "host": "10.0.1.100",
        "port": 6379,
        "password": null
    },
    "docker": {
        "socket": "/var/run/docker.sock",
        "network": "minecraft-network"
    },
    "intervals": {
        "metrics": 30,
        "health_check": 30,
        "cleanup": 300
    },
    "limits": {
        "max_containers": 10,
        "memory_per_server": "2G",
        "cpu_per_server": "1000m"
    },
    "auto_recovery": {
        "enabled": true,
        "max_restart_attempts": 3,
        "restart_cooldown": 300
    }
}
```

## セキュリティ・運用考慮事項

### 1. セキュリティ
- **Docker Socket保護**: 限定ユーザーのみアクセス
- **Redis認証**: パスワード保護
- **コンテナ隔離**: ネットワーク・ファイルシステム分離
- **リソース制限**: CPU・メモリ上限設定

### 2. ログ・監視
- **構造化ログ**: JSON形式でのログ出力
- **エラーハンドリング**: 例外の適切な処理・報告
- **パフォーマンス監視**: Agent自体のリソース使用量監視

### 3. 障害対応
- **Agent復旧**: Agentプロセス自体の監視・自動再起動
- **ネットワーク分断**: 一時的な接続断絶への対応
- **リソース枯渇**: メモリ・ディスク不足時の適切な処理

## Management Agentのメリット

### 1. 分散処理
- マスターサーバーの負荷軽減
- ローカルでの高速処理・判断

### 2. リアルタイム監視
- ネットワーク遅延なしの監視
- 即座の問題検知・対応

### 3. 自律性
- ネットワーク切断時でも基本機能継続
- ローカル判断での緊急停止・復旧

### 4. スケーラビリティ
- ホスト追加時の自動Agent配置
- 管理対象の線形拡張

この設計により、中央集権的な判断と分散実行のハイブリッド管理が実現され、効率的で信頼性の高いMinecraftサーバー管理が可能になります。
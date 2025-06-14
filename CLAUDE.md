# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

MineDevOps is a PaaS service that provides Minecraft development servers to users. The service automatically builds and deploys from GitHub pushes, manages server lifecycle (start on access, stop when idle), and includes a point system for resource management.

## Technology Stack

### Web Application
- **Backend**: Laravel (latest) with Laravel Sail for containerization
- **Frontend**: Vue.js with InertiaJS
- **Styling**: TailwindCSS with DaisyUI component library  
- **Authentication**: Laravel Breeze
- **Build Tool**: Vite

### Minecraft Server Infrastructure
- **Container Platform**: AWS Fargate
- **Data Storage**: Amazon EFS (Elastic File System)
- **Proxy Server**: Velocity Proxy Server (fixed public endpoint)
- **Network**: AWS VPC with public/private subnets
- **Orchestration**: AWS ECS with PHP AWS SDK integration
- **Scaling**: On-demand server creation and termination

## Development Environment Setup

### Prerequisites
- Docker and Docker Compose (for Laravel Sail)
- Node.js and npm

### Common Commands

All Laravel commands should be run through Sail:

```bash
# Start Sail containers
./vendor/bin/sail up -d

# Install PHP dependencies (including AWS SDK)
./vendor/bin/sail composer install
./vendor/bin/sail composer require aws/aws-sdk-php

# Install frontend dependencies  
./vendor/bin/sail npm install

# Run database migrations
./vendor/bin/sail artisan migrate

# Build frontend assets
./vendor/bin/sail npm run build

# Development frontend build with hot reload
./vendor/bin/sail npm run dev

# Run tests
./vendor/bin/sail artisan test
./vendor/bin/sail npm run test

# Code quality
./vendor/bin/sail composer lint
./vendor/bin/sail npm run lint
```

### AWS Integration Setup

**Required Dependencies**
```bash
./vendor/bin/sail composer require aws/aws-sdk-php
```

**Environment Configuration**
```env
AWS_ACCESS_KEY_ID=your_access_key
AWS_SECRET_ACCESS_KEY=your_secret_key
AWS_DEFAULT_REGION=ap-northeast-1
AWS_ECS_CLUSTER=minecraft-cluster
AWS_EFS_FILESYSTEM_ID=fs-minecraft-data
```

**Service Implementation Example**
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
                    'subnets' => ['subnet-private-1', 'subnet-private-2'],
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

### Development Workflow

1. **Container Management**: Always check if Sail containers are running before executing commands. Start with `./vendor/bin/sail up -d` if needed.

2. **Git Workflow**: Create feature branches for each task. Use descriptive commit messages with appropriate prefixes.

3. **Frontend Development**: Vue components should follow Atomic Design (atoms, molecules, organisms, templates, pages). Write tests for all Vue components and ensure they pass before committing.

4. **Testing**: Run both backend (`sail artisan test`) and frontend (`sail npm run test`) tests before commits.

## Core Business Logic

### Server Management
- Users can create one Minecraft development server per account
- Manual server start/stop control via web interface
- GitHub integration for automatic build/deploy on push

### Point System
- Users consume points while servers are running
- Points earned through video rewards, community engagement, content creation
- Integration planned with other company services

## Minecraft Server Architecture

### Network Configuration
```
Internet → Velocity Proxy (Public IP) → Fargate Tasks (Private Subnets)
```

**VPC Design**
- VPC: 10.0.0.0/16
- Public Subnet: 10.0.1.0/24 (Velocity Server)
- Private Subnets: 10.0.2.0/24, 10.0.3.0/24 (Fargate Tasks)

**Security Groups**
- Velocity Server: Inbound 25565 from 0.0.0.0/0, Outbound to private subnets
- Fargate Tasks: Inbound 25565 from Velocity SG only

### Data Management
**EFS Structure**
```
fs-minecraft-data/
├── user_1/
│   ├── world/          # World data
│   ├── plugins/        # Plugin configurations
│   ├── config/         # server.properties
│   └── logs/           # Server logs
├── user_2/
└── ...
```

**Fargate Volume Configuration**
- EFS mount per user: `/minecraft/data → /user_{id}/`
- Transit encryption enabled
- Automatic backups via EFS Backup

### Server Lifecycle
1. **Manual Start**: User initiates server start via web interface
2. **Dynamic Start**: ECS runs Fargate task with user-specific config
3. **Proxy Registration**: Velocity updated with new server endpoint
4. **Manual Stop**: User stops server via web interface when finished
5. **Data Persistence**: User data retained on EFS

### Infrastructure Considerations
- **Scaling**: Fargate auto-scaling based on CPU/memory usage
- **Security**: Container isolation with dedicated VPC subnets
- **Monitoring**: CloudWatch metrics for task performance
- **Cost Optimization**: On-demand task creation/termination


# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

MineDevOps is a PaaS service that provides Minecraft development servers to users. The service automatically builds and deploys from GitHub pushes, manages server lifecycle (start on access, stop when idle), and includes a point system for resource management.

## Technology Stack

- **Backend**: Laravel (latest) with Laravel Sail for containerization
- **Frontend**: Vue.js with InertiaJS
- **Styling**: TailwindCSS with DaisyUI component library  
- **Authentication**: Laravel Breeze
- **Build Tool**: Vite

## Development Environment Setup

### Prerequisites
- Docker and Docker Compose (for Laravel Sail)
- Node.js and npm

### Common Commands

All Laravel commands should be run through Sail:

```bash
# Start Sail containers
./vendor/bin/sail up -d

# Install PHP dependencies
./vendor/bin/sail composer install

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

### Development Workflow

1. **Container Management**: Always check if Sail containers are running before executing commands. Start with `./vendor/bin/sail up -d` if needed.

2. **Git Workflow**: Create feature branches for each task. Use descriptive commit messages with appropriate prefixes.

3. **Frontend Development**: Vue components should follow Atomic Design (atoms, molecules, organisms, templates, pages). Write tests for all Vue components and ensure they pass before committing.

4. **Testing**: Run both backend (`sail artisan test`) and frontend (`sail npm run test`) tests before commits.

## Core Business Logic

### Server Management
- Users can create one Minecraft development server per account
- Servers auto-start on user access and stop when no connections remain
- GitHub integration for automatic build/deploy on push

### Point System
- Users consume points while servers are running
- Points earned through video rewards, community engagement, content creation
- Integration planned with other company services

### Infrastructure Considerations
- Multiple containers per host machine
- Dynamic scaling of host machines based on active server count
- Security isolation for user-generated code deployment
- Network access controls for deployed servers

## Architecture Notes

The system manages the complete lifecycle of containerized Minecraft servers, from GitHub repository monitoring to container orchestration and user session tracking. Resource optimization through on-demand server activation is a core architectural requirement.


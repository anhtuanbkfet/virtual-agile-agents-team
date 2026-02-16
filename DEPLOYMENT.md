# Deployment Guide

This guide covers deployment options for the OpenCode Virtual Agile Agents Team system.

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Environment Configuration](#environment-configuration)
- [Local Development](#local-development)
- [Docker Deployment](#docker-deployment)
- [Cloud Deployment](#cloud-deployment)
  - [AWS](#aws)
  - [Google Cloud Platform](#google-cloud-platform)
  - [Azure](#azure)
- [Database Setup](#database-setup)
- [Monitoring Setup](#monitoring-setup)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

### Required Software

- Docker 20.10+ and Docker Compose 2.0+
- Node.js 18+ and npm 9+ (for local development)
- PostgreSQL 15+ (if not using Docker)
- Redis 7+ (if not using Docker)
- Git 2.30+

### Required Accounts

- [OpenCode](https://opencode.dev) account with API key
- GitHub account (for code management)
- Cloud provider account (AWS/GCP/Azure) for production deployment

---

## Environment Configuration

### Environment Variables

Create a `.env` file in the project root:

```bash
# OpenCode Configuration
OPENCODE_API_KEY=your_opencode_api_key_here
OPENCODE_API_URL=https://api.opencode.dev

# Database Configuration
DATABASE_URL=postgresql://admin:password@localhost:5432/virtual_agents
POSTGRES_USER=admin
POSTGRES_PASSWORD=password
POSTGRES_DB=virtual_agents

# Redis Configuration
REDIS_URL=redis://localhost:6379
REDIS_PASSWORD=

# Application Configuration
NODE_ENV=development
API_URL=http://localhost:8000
DASHBOARD_URL=http://localhost:3000
WS_URL=ws://localhost:9000

# JWT Configuration
JWT_SECRET=your_jwt_secret_here_min_32_chars
JWT_EXPIRES_IN=24h

# GitHub Integration
GITHUB_TOKEN=your_github_personal_access_token
GITHUB_REPO_OWNER=your_org_or_username
GITHUB_REPO_NAME=virtual-agents-project

# Slack Integration (optional)
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/...
SLACK_BOT_TOKEN=xoxb-your-bot-token

# Email Configuration (optional)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-email@gmail.com
SMTP_PASSWORD=your-app-password
SMTP_FROM=noreply@virtual-agents.dev

# Monitoring
PROMETHEUS_ENABLED=true
GRAFANA_ADMIN_PASSWORD=admin

# Rate Limiting
RATE_LIMIT_WINDOW_MS=3600000
RATE_LIMIT_MAX_REQUESTS=1000
```

### Environment-Specific Files

Create environment-specific files:

- `.env.development` - Development environment
- `.env.staging` - Staging environment
- `.env.production` - Production environment

---

## Local Development

### Setup

```bash
# Clone the repository
git clone https://github.com/yourorg/virtual-agile-agents-team.git
cd virtual-agile-agents-team

# Install dependencies
npm install

# Copy environment template
cp .env.example .env

# Edit .env with your configuration
nano .env
```

### Run with Docker Compose

```bash
# Start all services
docker-compose up -d

# View logs
docker-compose logs -f

# Check service status
docker-compose ps
```

### Run Locally (without Docker)

```bash
# Start PostgreSQL
docker run --name postgres-dev \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD=password \
  -e POSTGRES_DB=virtual_agents \
  -p 5432:5432 \
  -d postgres:15

# Start Redis
docker run --name redis-dev \
  -p 6379:6379 \
  -d redis:7-alpine

# Run database migrations
npm run db:migrate

# Start API service
cd api && npm run dev

# Start Orchestrator service
cd orchestrator && npm run dev

# Start Dashboard (in new terminal)
cd dashboard && npm run dev
```

---

## Docker Deployment

### Build Docker Images

```bash
# Build all images
docker-compose build

# Build specific service
docker-compose build api
```

### Production Docker Compose

Create `docker-compose.prod.yml`:

```yaml
version: '3.8'

services:
  # Dashboard
  dashboard:
    image: virtual-agents/dashboard:latest
    ports: ["3000:3000"]
    environment:
      - NODE_ENV=production
      - API_URL=${API_URL}
      - NEXT_PUBLIC_WS_URL=${WS_URL}
    restart: unless-stopped
    depends_on:
      - api

  # API Service
  api:
    image: virtual-agents/api:latest
    ports: ["8000:8000"]
    environment:
      - NODE_ENV=production
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=${REDIS_URL}
      - OPENCODE_API_KEY=${OPENCODE_API_KEY}
    restart: unless-stopped
    depends_on:
      - postgres
      - redis

  # Orchestrator Service
  orchestrator:
    image: virtual-agents/orchestrator:latest
    ports: ["9000:9000"]
    environment:
      - NODE_ENV=production
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=${REDIS_URL}
      - OPENCODE_API_KEY=${OPENCODE_API_KEY}
    restart: unless-stopped
    depends_on:
      - postgres
      - redis
      - api

  # PostgreSQL
  postgres:
    image: postgres:15-alpine
    ports: ["5432:5432"]
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./backups:/backups
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
    restart: unless-stopped

  # Redis
  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes
    restart: unless-stopped

  # Nginx
  nginx:
    image: nginx:1.25-alpine
    ports: ["80:80", "443:443"]
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    restart: unless-stopped
    depends_on:
      - dashboard
      - api

  # Prometheus
  prometheus:
    image: prom/prometheus:latest
    ports: ["9090:9090"]
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    restart: unless-stopped

  # Grafana
  grafana:
    image: grafana/grafana:latest
    ports: ["3001:3000"]
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_ADMIN_PASSWORD}
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
  prometheus_data:
  grafana_data:
```

### Deploy with Docker Compose

```bash
# Load environment variables
export $(cat .env.production | xargs)

# Start production services
docker-compose -f docker-compose.prod.yml up -d

# Run database migrations
docker-compose -f docker-compose.prod.yml exec api npm run db:migrate

# Initialize agents
docker-compose -f docker-compose.prod.yml exec orchestrator npm run agents:init
```

---

## Cloud Deployment

### AWS

#### Prerequisites

- AWS CLI installed and configured
- ECR (Elastic Container Registry) access
- ECS or EKS cluster
- RDS PostgreSQL instance
- ElastiCache Redis instance

#### Step 1: Push Docker Images to ECR

```bash
# Login to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin <account-id>.dkr.ecr.us-east-1.amazonaws.com

# Create repositories
aws ecr create-repository --repository-name virtual-agents/dashboard
aws ecr create-repository --repository-name virtual-agents/api
aws ecr create-repository --repository-name virtual-agents/orchestrator

# Tag and push images
docker tag virtual-agents/dashboard:latest <account-id>.dkr.ecr.us-east-1.amazonaws.com/virtual-agents/dashboard:latest
docker push <account-id>.dkr.ecr.us-east-1.amazonaws.com/virtual-agents/dashboard:latest

# Repeat for api and orchestrator
```

#### Step 2: Create RDS PostgreSQL

```bash
# Create subnet group
aws rds create-db-subnet-group \
  --db-subnet-group-name virtual-agents-subnet-group \
  --db-subnet-group-description "Subnet group for Virtual Agents"

# Create database
aws rds create-db-instance \
  --db-instance-identifier virtual-agents-db \
  --db-instance-class db.t3.medium \
  --engine postgres \
  --engine-version 15.4 \
  --allocated-storage 100 \
  --master-username admin \
  --master-user-password YourSecurePassword! \
  --db-subnet-group-name virtual-agents-subnet-group \
  --publicly-accessible \
  --vpc-security-group-ids sg-xxxxxxxx
```

#### Step 3: Create ElastiCache Redis

```bash
# Create Redis cluster
aws elasticache create-replication-group \
  --replication-group-id virtual-agents-redis \
  --replication-group-description "Redis for Virtual Agents" \
  --cache-node-type cache.t3.medium \
  --engine redis \
  --engine-version 7.0 \
  --num-cache-clusters 1 \
  --automatic-failover-enabled
```

#### Step 4: Deploy to ECS

Create ECS task definition `task-definition.json`:

```json
{
  "family": "virtual-agents-api",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "containerDefinitions": [
    {
      "name": "api",
      "image": "<account-id>.dkr.ecr.us-east-1.amazonaws.com/virtual-agents/api:latest",
      "essential": true,
      "portMappings": [
        {
          "containerPort": 8000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "NODE_ENV",
          "value": "production"
        },
        {
          "name": "DATABASE_URL",
          "value": "postgresql://admin:password@<db-endpoint>:5432/virtual_agents"
        },
        {
          "name": "REDIS_URL",
          "value": "redis://<redis-endpoint>:6379"
        }
      ],
      "secrets": [
        {
          "name": "OPENCODE_API_KEY",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:<account-id>:secret:opencode-api-key"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/virtual-agents",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "api"
        }
      }
    }
  ]
}
```

```bash
# Register task definition
aws ecs register-task-definition --cli-input-json file://task-definition.json

# Create service
aws ecs create-service \
  --cluster virtual-agents-cluster \
  --service-name api-service \
  --task-definition virtual-agents-api \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-123,subnet-456],securityGroups=[sg-xxx],assignPublicIp=ENABLED}"
```

#### Step 5: Configure Application Load Balancer

```bash
# Create target groups
aws elbv2 create-target-group \
  --name api-targets \
  --protocol HTTP \
  --port 8000 \
  --target-type ip \
  --vpc-id vpc-xxxxxxxx

# Create load balancer
aws elbv2 create-load-balancer \
  --name virtual-agents-lb \
  --subnets subnet-123 subnet-456 \
  --security-groups sg-xxx

# Create listeners
aws elbv2 create-listener \
  --load-balancer-arn <lb-arn> \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=<target-group-arn>
```

### Google Cloud Platform (GCP)

#### Prerequisites

- gcloud CLI installed and configured
- GCR (Google Container Registry) access
- GKE cluster or Cloud Run
- Cloud SQL for PostgreSQL
- Memorystore for Redis

#### Step 1: Push Docker Images to GCR

```bash
# Configure Docker to use gcloud as a credential helper
gcloud auth configure-docker us-central1-docker.pkg.dev

# Tag images
docker tag virtual-agents/dashboard:latest us-central1-docker.pkg.dev/<project-id>/virtual-agents/dashboard:latest

# Push images
docker push us-central1-docker.pkg.dev/<project-id>/virtual-agents/dashboard:latest
```

#### Step 2: Create Cloud SQL PostgreSQL

```bash
# Create instance
gcloud sql instances create virtual-agents-db \
  --database-version=POSTGRES_15 \
  --tier=db-custom-2-8192 \
  --cpu=2 \
  --memory=8192MB \
  --region=us-central1 \
  --storage-auto-increase \
  --storage-size=100GB

# Create database
gcloud sql databases create virtual_agents --instance=virtual-agents-db

# Create user
gcloud sql users create admin --instance=virtual-agents-db --password=YourSecurePassword!
```

#### Step 3: Create Memorystore Redis

```bash
# Create Redis instance
gcloud redis instances create virtual-agents-redis \
  --region=us-central1 \
  --tier=standard \
  --memory-size-gb=2 \
  --redis-version=redis_7_0 \
  --display-name="Virtual Agents Redis"
```

#### Step 4: Deploy to Cloud Run

```bash
# Deploy API
gcloud run deploy virtual-agents-api \
  --image us-central1-docker.pkg.dev/<project-id>/virtual-agents/api:latest \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated \
  --set-env-vars NODE_ENV=production \
  --set-env-vars DATABASE_URL="postgresql://admin:password@<cloud-sql-ip>:5432/virtual_agents" \
  --set-env-vars REDIS_URL="redis://<redis-ip>:6379" \
  --set-secrets OPENCODE_API_KEY=opencode-api-key:latest \
  --cpu 2 \
  --memory 4Gi \
  --max-instances 10

# Deploy Dashboard
gcloud run deploy virtual-agents-dashboard \
  --image us-central1-docker.pkg.dev/<project-id>/virtual-agents/dashboard:latest \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated \
  --set-env-vars API_URL=https://<api-url> \
  --cpu 1 \
  --memory 2Gi \
  --max-instances 5
```

#### Step 5: Deploy to GKE (Alternative)

```bash
# Create cluster
gcloud container clusters create virtual-agents-cluster \
  --zone us-central1-a \
  --num-nodes 3 \
  --machine-type e2-medium \
  --enable-autoscaling \
  --min-nodes 2 \
  --max-nodes 10

# Get credentials
gcloud container clusters get-credentials virtual-agents-cluster --zone us-central1-a

# Create namespace
kubectl create namespace virtual-agents

# Create secrets
kubectl create secret generic opencode-secret \
  --from-literal=api-key=your-opencode-api-key \
  -n virtual-agents

# Apply deployments
kubectl apply -f k8s/ -n virtual-agents
```

### Azure

#### Prerequisites

- Azure CLI installed and configured
- Azure Container Registry (ACR)
- Azure Kubernetes Service (AKS) or Container Instances
- Azure Database for PostgreSQL
- Azure Cache for Redis

#### Step 1: Push Docker Images to ACR

```bash
# Create container registry
az acr create --resource-group VirtualAgentsRG \
  --name virtualagentsacr \
  --sku Standard

# Login to ACR
az acr login --name virtualagentsacr

# Tag images
docker tag virtual-agents/dashboard:latest virtualagentsacr.azurecr.io/dashboard:latest

# Push images
docker push virtualagentsacr.azurecr.io/dashboard:latest
```

#### Step 2: Create Azure Database for PostgreSQL

```bash
# Create server
az postgres server create \
  --name virtual-agents-db \
  --resource-group VirtualAgentsRG \
  --location eastus \
  --admin-user admin \
  --admin-password YourSecurePassword! \
  --sku-name GP_Gen5_2 \
  --version 15

# Create database
az postgres db create \
  --name virtual_agents \
  --server-name virtual-agents-db \
  --resource-group VirtualAgentsRG
```

#### Step 3: Create Azure Cache for Redis

```bash
# Create Redis cache
az redis create \
  --name virtual-agents-redis \
  --resource-group VirtualAgentsRG \
  --location eastus \
  --sku Basic \
  --vm-size c0 \
  --capacity 0
```

#### Step 4: Deploy to Azure Kubernetes Service (AKS)

```bash
# Create resource group
az group create --name VirtualAgentsRG --location eastus

# Create AKS cluster
az aks create \
  --resource-group VirtualAgentsRG \
  --name virtual-agents-cluster \
  --node-count 3 \
  --node-vm-size Standard_DS2_v2 \
  --enable-autoscaler \
  --min-count 2 \
  --max-count 10 \
  --generate-ssh-keys

# Get credentials
az aks get-credentials --resource-group VirtualAgentsRG --name virtual-agents-cluster

# Create namespace
kubectl create namespace virtual-agents

# Create secrets
kubectl create secret generic opencode-secret \
  --from-literal=api-key=your-opencode-api-key \
  -n virtual-agents

# Apply deployments
kubectl apply -f k8s/ -n virtual-agents
```

---

## Database Setup

### Initialize Database

```bash
# Run migrations
npm run db:migrate

# Seed initial data
npm run db:seed

# Create admin user
npm run db:create-admin
```

### Backup Database

```bash
# Backup using pg_dump
pg_dump $DATABASE_URL > backup_$(date +%Y%m%d_%H%M%S).sql

# Backup using Docker
docker-compose exec postgres pg_dump -U admin virtual_agents > backup.sql

# Automated backup script
#!/bin/bash
# save to scripts/backup-db.sh
BACKUP_DIR="/backups"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
docker-compose exec -T postgres pg_dump -U admin virtual_agents > $BACKUP_DIR/backup_$TIMESTAMP.sql
```

### Restore Database

```bash
# Restore from backup
psql $DATABASE_URL < backup_20240115_080000.sql

# Restore using Docker
docker-compose exec -T postgres psql -U admin virtual_agents < backup.sql
```

---

## Monitoring Setup

### Prometheus Configuration

Create `prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'api'
    static_configs:
      - targets: ['api:8000']
    metrics_path: '/metrics'

  - job_name: 'orchestrator'
    static_configs:
      - targets: ['orchestrator:9000']
    metrics_path: '/metrics'

  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres:5432']
```

### Grafana Dashboards

1. Access Grafana at `http://localhost:3001`
2. Login with admin credentials
3. Add Prometheus data source
4. Import dashboards from `grafana/dashboards/`

### Log Aggregation

Option 1: ELK Stack
```yaml
# Add to docker-compose.yml
elasticsearch:
  image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
  ports: ["9200:9200"]
  environment:
    - discovery.type=single-node
    - "ES_JAVA_OPTS=-Xms512m -Xmx512m"

logstash:
  image: docker.elastic.co/logstash/logstash:8.11.0
  ports: ["5044:5044"]
  volumes:
    - ./logstash/pipeline:/usr/share/logstash/pipeline

kibana:
  image: docker.elastic.co/kibana/kibana:8.11.0
  ports: ["5601:5601"]
  depends_on:
    - elasticsearch
```

Option 2: CloudWatch (AWS)
```bash
# Install CloudWatch agent
aws ssm send-command \
  --instance-ids i-xxxxxxxx \
  --document-name "AWS-ConfigureCloudWatch"
```

---

## Troubleshooting

### Common Issues

#### Database Connection Failed

```bash
# Check PostgreSQL status
docker-compose ps postgres

# View logs
docker-compose logs postgres

# Test connection
docker-compose exec postgres psql -U admin -d virtual_agents
```

#### Redis Connection Failed

```bash
# Check Redis status
docker-compose ps redis

# Test connection
docker-compose exec redis redis-cli ping
```

#### Agent Not Responding

```bash
# Check Orchestrator logs
docker-compose logs orchestrator

# Check OpenCode API key
docker-compose exec orchestrator printenv | grep OPENCODE

# Test API connection
curl https://api.opencode.dev/health
```

#### High Memory Usage

```bash
# Check resource usage
docker stats

# Restart services
docker-compose restart

# Scale services
docker-compose up -d --scale api=3
```

### Health Checks

```bash
# API health check
curl http://localhost:8000/health

# Dashboard health check
curl http://localhost:3000

# Orchestrator health check
curl http://localhost:9000/health

# Database health check
docker-compose exec postgres pg_isready
```

### Log Analysis

```bash
# View all logs
docker-compose logs

# View specific service logs
docker-compose logs -f api

# View logs with timestamps
docker-compose logs -t

# Export logs
docker-compose logs > logs_$(date +%Y%m%d).txt
```

---

## Security Checklist

- [ ] Change default passwords
- [ ] Enable HTTPS/TLS
- [ ] Configure firewall rules
- [ ] Enable audit logging
- [ ] Set up automated backups
- [ ] Configure secrets management
- [ ] Enable MFA for admin accounts
- [ ] Regular security updates
- [ ] Penetration testing
- [ ] Compliance checks (GDPR, SOC2)

---

## Performance Optimization

### Database Optimization

```sql
-- Create indexes
CREATE INDEX idx_backlog_status ON backlog_items(status);
CREATE INDEX idx_backlog_sprint ON backlog_items(sprint_id);
CREATE INDEX idx_agent_messages_timestamp ON agent_messages(timestamp);

-- Update statistics
ANALYZE agents;
ANALYZE backlog_items;
```

### Redis Optimization

```bash
# Set max memory
CONFIG SET maxmemory 256mb
CONFIG SET maxmemory-policy allkeys-lru
```

### Application Optimization

```javascript
// Enable caching
const CACHE_TTL = 3600; // 1 hour

// Connection pooling
const pool = new Pool({
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000
});
```

---

## Support

For deployment support:
- **Documentation**: [docs.virtual-agents.dev/deploy](https://docs.virtual-agents.dev/deploy)
- **Email**: deploy-support@virtual-agents.dev
- **Slack**: #deployment-support

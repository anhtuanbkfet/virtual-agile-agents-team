# OpenCode Virtual Agile Agents Team

An AI-powered multi-agent system that simulates a real software development team using OpenCode agents. Each agent represents a specific role (PM, Architect, Tech Lead, Developers, QA) and collaborates through automated Agile workflows to deliver software projects.

## 🎯 Overview

This system leverages OpenCode's multi-agent orchestration capabilities to create a virtual software development team that follows Agile/Scrum methodologies. The team automates workflows including daily standups, sprint planning, development, testing, and reporting.

## ✨ Key Features

- **Multi-Agent Orchestration**: 6+ specialized agents working collaboratively
- **Agile Workflow Automation**: Automated Scrum ceremonies (standup, planning, review)
- **Real-Time Communication**: Agent-to-agent messaging via OpenCode
- **Intelligent Task Assignment**: AI-powered task distribution based on skills
- **Automated Reporting**: Daily and sprint-level progress reports
- **Velocity Tracking**: Team velocity metrics and burndown charts
- **GitHub Integration**: Seamless code management and pull requests
- **Slack Notifications**: Real-time alerts and reports

## 🏗️ Architecture

The system consists of:

1. **Orchestrator Agent**: Central hub coordinating all agents
2. **Role-Based Agents**: PM, Architect, Tech Lead, Developers, QA
3. **Workflow Engine**: Automates Agile workflows
4. **Dashboard UI**: Web interface for monitoring and management
5. **Data Layer**: PostgreSQL, Redis, GitHub integration

For detailed architecture, see [HLD.md](./HLD.md)

## 🚀 Quick Start

### Prerequisites

- Node.js 18+ and npm/yarn
- Docker and Docker Compose
- OpenCode API key
- GitHub account
- PostgreSQL 15+ (or use Docker)

### Installation

```bash
# Clone the repository
git clone https://github.com/yourorg/virtual-agile-agents-team.git
cd virtual-agile-agents-team

# Install dependencies
npm install

# Copy environment variables
cp .env.example .env

# Edit .env with your configuration
nano .env
```

### Configuration

Add your OpenCode API key to `.env`:

```env
OPENCODE_API_KEY=your_opencode_api_key_here
DATABASE_URL=postgresql://admin:password@localhost:5432/virtual_agents
REDIS_URL=redis://localhost:6379
GITHUB_TOKEN=your_github_token
SLACK_WEBHOOK_URL=your_slack_webhook_url
```

### Run with Docker Compose

```bash
# Start all services
docker-compose up -d

# Run database migrations
npm run db:migrate

# Initialize agents
npm run agents:init

# Start development servers
npm run dev
```

The dashboard will be available at `http://localhost:3000`

## 📊 Dashboard Features

### Overview Page
- Team velocity chart
- Sprint burndown chart
- Active agents status
- Recent activities feed

### Backlog Management
- Create and manage backlog items
- Drag-and-drop to sprints
- Assign tasks to agents
- Priority management

### Sprint Board
- Kanban-style board
- Columns: Backlog, In Progress, Testing, Done
- Agent work distribution

### Agent Monitor
- Real-time agent status
- Agent conversation history
- Performance metrics
- Error logs

### Reports
- Daily standup reports
- Sprint summaries
- Velocity metrics
- Quality reports

## 🔧 Development

### Project Structure

```
virtual-agile-agents-team/
├── dashboard/              # Next.js Dashboard UI
│   ├── app/               # App Router pages
│   ├── components/        # React components
│   └── lib/              # Utility functions
├── api/                   # API Service
│   ├── routes/           # API endpoints
│   ├── services/         # Business logic
│   └── models/           # Data models
├── orchestrator/          # Orchestrator Service
│   ├── agents/           # Agent configurations
│   ├── workflows/        # Workflow definitions
│   └── communication/    # Message routing
├── database/              # Database schemas and migrations
├── docker/                # Docker configurations
├── docs/                  # Documentation
└── tests/                 # Test files
```

### Running Tests

```bash
# Run all tests
npm test

# Run with coverage
npm run test:coverage

# Run integration tests
npm run test:integration
```

### Code Style

We use ESLint and Prettier for code formatting:

```bash
# Lint code
npm run lint

# Fix linting issues
npm run lint:fix

# Format code
npm run format
```

## 🤖 Agents

### Available Agents

| Agent | Model | Role | Specialization |
|-------|-------|------|----------------|
| PM Agent | GPT-4o | Management | Sprint planning, reporting |
| Architect Agent | GPT-4o | Technical | System design, architecture |
| Tech Lead Agent | GPT-4o | Technical | Code review, decisions |
| Dev-1 Agent | GPT-4o Mini | Development | Frontend development |
| Dev-2 Agent | GPT-4o Mini | Development | Backend development |
| QA Agent | GPT-4o | Quality | Testing, QA |

### Creating Custom Agents

```typescript
// orchestrator/agents/custom-agent.ts
import { createAgent } from '@opencode/sdk';

export const customAgent = createAgent({
  id: 'custom-agent',
  name: 'Custom Agent',
  model: 'gpt-4o',
  role: 'specialist',
  systemPrompt: 'You are a specialist in...',
  capabilities: ['task1', 'task2'],
  tools: ['tool1', 'tool2']
});
```

## 📝 Workflows

### Available Workflows

1. **Daily Standup**: Automated daily team updates
2. **Sprint Planning**: Sprint backlog creation and task assignment
3. **Sprint Execution**: Feature development and testing
4. **Sprint Review**: Sprint summary and velocity report
5. **Retrospective**: Team improvement planning

### Triggering Workflows

```typescript
import { executeWorkflow } from './orchestrator/workflows';

// Trigger daily standup
await executeWorkflow('daily_standup');

// Trigger sprint planning
await executeWorkflow('sprint_planning', {
  projectId: 'project-123',
  sprintDuration: '2 weeks'
});
```

## 📈 Monitoring

### System Metrics

The system tracks:
- Agent availability and response times
- Workflow execution success rates
- API usage and costs
- Database performance
- Sprint velocity and metrics

### Grafana Dashboards

Access monitoring dashboards at `http://localhost:3001`

Default credentials:
- Username: `admin`
- Password: `admin` (change on first login)

## 🔐 Security

### Authentication

- JWT-based authentication for Dashboard
- API key authentication for external integrations
- Role-based access control (RBAC)

### API Security

- Rate limiting per user/IP
- HTTPS/TLS encryption
- Input validation and sanitization
- SQL injection prevention (ORM)

### Data Protection

- Encryption at rest (PostgreSQL)
- Encryption in transit (TLS 1.3)
- Regular automated backups
- GDPR compliance considerations

## 🚢 Deployment

### Docker Deployment

```bash
# Build and start containers
docker-compose up -d

# Check logs
docker-compose logs -f

# Stop containers
docker-compose down
```

### Cloud Deployment

See [DEPLOYMENT.md](./DEPLOYMENT.md) for detailed deployment guides for:
- AWS (ECS/EKS)
- Google Cloud (GKE)
- Azure (AKS)

## 💰 Cost Analysis

**Estimated Monthly Cost**:
- Infrastructure: $220-350
- AI API: $295-480
- **Total: $515-830/month**

See [HLD.md](./HLD.md#12-cost-analysis) for detailed breakdown.

## 📚 Documentation

- [HLD.md](./HLD.md) - High-Level Design Document
- [ARCHITECTURE.md](./ARCHITECTURE.md) - Technical Architecture
- [API.md](./API.md) - API Documentation
- [DEPLOYMENT.md](./DEPLOYMENT.md) - Deployment Guide
- [CONTRIBUTING.md](./CONTRIBUTING.md) - Contribution Guidelines

## 🗺️ Roadmap

### Phase 1: MVP (Current)
- ✅ Basic multi-agent setup
- ✅ Daily standup workflow
- ✅ Basic dashboard

### Phase 2: Full Team
- ⏳ Architect and Tech Lead agents
- ⏳ Sprint planning workflow
- ⏳ Advanced reporting

### Phase 3: Automation
- ⏳ Sprint review workflow
- ⏳ Automated code review
- ⏳ CI/CD integration

### Phase 4: Advanced Features
- ⏳ RAG integration
- ⏳ Multi-project support
- ⏳ Custom agent creation

## 🤝 Contributing

We welcome contributions! Please see [CONTRIBUTING.md](./CONTRIBUTING.md) for guidelines.

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](./LICENSE) file for details.

## 💬 Support

- 📧 Email: support@virtual-agents.dev
- 💬 Slack: Join our community
- 📖 Documentation: [docs.virtual-agents.dev](https://docs.virtual-agents.dev)

## 🙏 Acknowledgments

- [OpenCode](https://opencode.dev) - AI agent platform
- [Next.js](https://nextjs.org) - React framework
- [shadcn/ui](https://ui.shadcn.com) - UI components
- All contributors and supporters

---

**Built with ❤️ by the Virtual Agents Team**

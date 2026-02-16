# API Documentation

## Overview

The OpenCode Virtual Agile Agents Team provides a RESTful API for managing agents, workflows, projects, sprints, and backlog items.

**Base URL**: `http://localhost:8000/api/v1`

**Authentication**: Bearer Token (JWT)

```http
Authorization: Bearer <your-jwt-token>
```

---

## Table of Contents

- [Authentication](#authentication)
- [Agents](#agents)
- [Workflows](#workflows)
- [Projects](#projects)
- [Sprints](#sprints)
- [Backlog Items](#backlog-items)
- [Reports](#reports)
- [Webhooks](#webhooks)

---

## Authentication

### Login

Authenticate and receive a JWT token.

```http
POST /auth/login
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "password123"
}
```

**Response**:

```json
{
  "success": true,
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "user": {
      "id": "user-123",
      "email": "user@example.com",
      "role": "admin"
    },
    "expiresIn": "24h"
  }
}
```

### Refresh Token

Refresh an expired JWT token.

```http
POST /auth/refresh
Content-Type: application/json

{
  "refreshToken": "refresh_token_here"
}
```

---

## Agents

### List All Agents

Get a list of all registered agents.

```http
GET /agents
Authorization: Bearer <token>
```

**Response**:

```json
{
  "success": true,
  "data": [
    {
      "id": "pm-agent",
      "name": "Project Manager",
      "role": "management",
      "model": "gpt-4o",
      "status": "active",
      "capabilities": ["planning", "tracking", "reporting"],
      "createdAt": "2024-01-01T00:00:00Z"
    }
  ],
  "pagination": {
    "total": 6,
    "page": 1,
    "limit": 20
  }
}
```

### Get Agent Details

Get detailed information about a specific agent.

```http
GET /agents/:agentId
Authorization: Bearer <token>
```

**Response**:

```json
{
  "success": true,
  "data": {
    "id": "pm-agent",
    "name": "Project Manager",
    "role": "management",
    "model": "gpt-4o",
    "systemPrompt": "You are an experienced Project Manager...",
    "capabilities": ["planning", "tracking", "reporting"],
    "status": "active",
    "metrics": {
      "messagesSent": 1250,
      "messagesReceived": 1180,
      "tasksCompleted": 85,
      "averageResponseTime": "2.5s"
    },
    "createdAt": "2024-01-01T00:00:00Z"
  }
}
```

### Send Message to Agent

Send a message to a specific agent.

```http
POST /agents/:agentId/messages
Authorization: Bearer <token>
Content-Type: application/json

{
  "type": "request",
  "intent": "generate_report",
  "payload": {
    "reportType": "daily_standup",
    "date": "2024-01-15"
  }
}
```

**Response**:

```json
{
  "success": true,
  "data": {
    "messageId": "msg-123",
    "agentId": "pm-agent",
    "response": {
      "report": "Daily standup completed successfully...",
      "metrics": {
        "tasksCompleted": 12,
        "tasksInProgress": 5,
        "blockers": ["API integration pending"]
      }
    },
    "timestamp": "2024-01-15T10:00:00Z"
  }
}
```

### Update Agent Configuration

Update an agent's configuration.

```http
PATCH /agents/:agentId
Authorization: Bearer <token>
Content-Type: application/json

{
  "status": "inactive",
  "config": {
    "maxConcurrentTasks": 3,
    "responseTimeout": 30000
  }
}
```

---

## Workflows

### List All Workflows

Get a list of available workflows.

```http
GET /workflows
Authorization: Bearer <token>
```

**Response**:

```json
{
  "success": true,
  "data": [
    {
      "id": "daily_standup",
      "name": "Daily Standup",
      "description": "Automated daily team standup meeting",
      "status": "active",
      "triggers": [
        {
          "type": "schedule",
          "schedule": "0 8 * * 1-5"
        }
      ]
    }
  ]
}
```

### Execute Workflow

Trigger a workflow execution.

```http
POST /workflows/:workflowId/execute
Authorization: Bearer <token>
Content-Type: application/json

{
  "params": {
    "projectId": "project-123",
    "sprintId": "sprint-456"
  }
}
```

**Response**:

```json
{
  "success": true,
  "data": {
    "executionId": "exec-123",
    "workflowId": "daily_standup",
    "status": "running",
    "startedAt": "2024-01-15T08:00:00Z",
    "estimatedCompletion": "2024-01-15T08:30:00Z"
  }
}
```

### Get Workflow Execution Status

Get the status of a workflow execution.

```http
GET /workflows/executions/:executionId
Authorization: Bearer <token>
```

**Response**:

```json
{
  "success": true,
  "data": {
    "executionId": "exec-123",
    "workflowId": "daily_standup",
    "status": "completed",
    "startedAt": "2024-01-15T08:00:00Z",
    "completedAt": "2024-01-15T08:28:00Z",
    "stages": [
      {
        "name": "collect_updates",
        "status": "completed",
        "duration": 120000
      },
      {
        "name": "generate_report",
        "status": "completed",
        "duration": 60000
      }
    ],
    "result": {
      "reportUrl": "https://...",
      "summary": "Daily standup completed with 5 agents"
    }
  }
}
```

### List Workflow Executions

Get a list of workflow executions.

```http
GET /workflows/executions?status=completed&limit=20
Authorization: Bearer <token>
```

**Response**:

```json
{
  "success": true,
  "data": [
    {
      "executionId": "exec-123",
      "workflowId": "daily_standup",
      "status": "completed",
      "startedAt": "2024-01-15T08:00:00Z",
      "completedAt": "2024-01-15T08:28:00Z"
    }
  ],
  "pagination": {
    "total": 50,
    "page": 1,
    "limit": 20
  }
}
```

---

## Projects

### List All Projects

Get a list of all projects.

```http
GET /projects
Authorization: Bearer <token>
```

**Response**:

```json
{
  "success": true,
  "data": [
    {
      "id": "project-123",
      "name": "E-commerce Platform",
      "description": "Build a modern e-commerce platform",
      "status": "active",
      "techStack": {
        "frontend": "Next.js",
        "backend": "Node.js",
        "database": "PostgreSQL"
      },
      "createdAt": "2024-01-01T00:00:00Z"
    }
  ]
}
```

### Create Project

Create a new project.

```http
POST /projects
Authorization: Bearer <token>
Content-Type: application/json

{
  "name": "E-commerce Platform",
  "description": "Build a modern e-commerce platform",
  "techStack": {
    "frontend": "Next.js",
    "backend": "Node.js",
    "database": "PostgreSQL"
  }
}
```

**Response**:

```json
{
  "success": true,
  "data": {
    "id": "project-123",
    "name": "E-commerce Platform",
    "description": "Build a modern e-commerce platform",
    "status": "active",
    "techStack": {
      "frontend": "Next.js",
      "backend": "Node.js",
      "database": "PostgreSQL"
    },
    "createdAt": "2024-01-15T00:00:00Z"
  }
}
```

### Get Project Details

Get detailed information about a project.

```http
GET /projects/:projectId
Authorization: Bearer <token>
```

**Response**:

```json
{
  "success": true,
  "data": {
    "id": "project-123",
    "name": "E-commerce Platform",
    "description": "Build a modern e-commerce platform",
    "status": "active",
    "techStack": {
      "frontend": "Next.js",
      "backend": "Node.js",
      "database": "PostgreSQL"
    },
    "metrics": {
      "totalSprints": 8,
      "totalBacklogItems": 45,
      "completedBacklogItems": 32,
      "averageVelocity": 24
    },
    "createdAt": "2024-01-01T00:00:00Z",
    "updatedAt": "2024-01-15T00:00:00Z"
  }
}
```

### Update Project

Update project details.

```http
PATCH /projects/:projectId
Authorization: Bearer <token>
Content-Type: application/json

{
  "status": "on_hold",
  "description": "Updated description"
}
```

---

## Sprints

### List Project Sprints

Get a list of sprints for a project.

```http
GET /projects/:projectId/sprints
Authorization: Bearer <token>
```

**Response**:

```json
{
  "success": true,
  "data": [
    {
      "id": "sprint-123",
      "name": "Sprint 8",
      "startDate": "2024-01-01",
      "endDate": "2024-01-14",
      "status": "completed",
      "plannedVelocity": 25,
      "actualVelocity": 23
    }
  ]
}
```

### Create Sprint

Create a new sprint.

```http
POST /projects/:projectId/sprints
Authorization: Bearer <token>
Content-Type: application/json

{
  "name": "Sprint 9",
  "startDate": "2024-01-15",
  "endDate": "2024-01-28",
  "plannedVelocity": 25
}
```

**Response**:

```json
{
  "success": true,
  "data": {
    "id": "sprint-456",
    "projectId": "project-123",
    "name": "Sprint 9",
    "startDate": "2024-01-15",
    "endDate": "2024-01-28",
    "status": "planning",
    "plannedVelocity": 25,
    "createdAt": "2024-01-15T00:00:00Z"
  }
}
```

### Start Sprint

Start a sprint execution.

```http
POST /projects/:projectId/sprints/:sprintId/start
Authorization: Bearer <token>
```

**Response**:

```json
{
  "success": true,
  "data": {
    "sprintId": "sprint-456",
    "status": "active",
    "startedAt": "2024-01-15T00:00:00Z",
    "workflowExecutionId": "exec-456"
  }
}
```

### Get Sprint Details

Get detailed information about a sprint.

```http
GET /projects/:projectId/sprints/:sprintId
Authorization: Bearer <token>
```

**Response**:

```json
{
  "success": true,
  "data": {
    "id": "sprint-456",
    "name": "Sprint 9",
    "startDate": "2024-01-15",
    "endDate": "2024-01-28",
    "status": "active",
    "plannedVelocity": 25,
    "actualVelocity": 12,
    "backlogItems": [
      {
        "id": "item-123",
        "title": "User authentication",
        "status": "in_progress",
        "assignedAgent": "dev-1"
      }
    ],
    "metrics": {
      "completedItems": 12,
      "remainingItems": 8,
      "progressPercentage": 60
    }
  }
}
```

---

## Backlog Items

### List Backlog Items

Get a list of backlog items for a project.

```http
GET /projects/:projectId/backlog
Authorization: Bearer <token>
```

**Query Parameters**:
- `status`: Filter by status (backlog, in_progress, testing, done)
- `type`: Filter by type (feature, bug, enhancement)
- `priority`: Filter by priority (low, medium, high, critical)
- `sprintId`: Filter by sprint
- `assignedAgent`: Filter by assigned agent

**Response**:

```json
{
  "success": true,
  "data": [
    {
      "id": "item-123",
      "title": "User authentication",
      "description": "Implement user login and registration",
      "type": "feature",
      "priority": "high",
      "storyPoints": 8,
      "status": "in_progress",
      "sprintId": "sprint-456",
      "assignedAgent": "dev-1",
      "createdAt": "2024-01-15T00:00:00Z"
    }
  ],
  "pagination": {
    "total": 45,
    "page": 1,
    "limit": 20
  }
}
```

### Create Backlog Item

Create a new backlog item.

```http
POST /projects/:projectId/backlog
Authorization: Bearer <token>
Content-Type: application/json

{
  "title": "User authentication",
  "description": "Implement user login and registration",
  "type": "feature",
  "priority": "high",
  "storyPoints": 8
}
```

**Response**:

```json
{
  "success": true,
  "data": {
    "id": "item-123",
    "projectId": "project-123",
    "title": "User authentication",
    "description": "Implement user login and registration",
    "type": "feature",
    "priority": "high",
    "storyPoints": 8,
    "status": "backlog",
    "createdAt": "2024-01-15T00:00:00Z"
  }
}
```

### Update Backlog Item

Update a backlog item.

```http
PATCH /projects/:projectId/backlog/:itemId
Authorization: Bearer <token>
Content-Type: application/json

{
  "status": "in_progress",
  "assignedAgent": "dev-1",
  "sprintId": "sprint-456"
}
```

### Assign Agent to Item

Assign an agent to a backlog item.

```http
POST /projects/:projectId/backlog/:itemId/assign
Authorization: Bearer <token>
Content-Type: application/json

{
  "agentId": "dev-1"
}
```

---

## Reports

### Get Daily Standup Report

Get the daily standup report for a specific date.

```http
GET /reports/daily-standup?date=2024-01-15
Authorization: Bearer <token>
```

**Response**:

```json
{
  "success": true,
  "data": {
    "date": "2024-01-15",
    "summary": {
      "totalAgents": 5,
      "agentsResponded": 5,
      "tasksCompleted": 12,
      "tasksInProgress": 8,
      "blockers": 2
    },
    "agentUpdates": [
      {
        "agentId": "dev-1",
        "agentName": "Developer 1",
        "completed": ["User login API", "Dashboard layout"],
        "planned": ["User registration API", "Profile page"],
        "blockers": ["API documentation missing"]
      }
    ],
    "generatedAt": "2024-01-15T08:30:00Z"
  }
}
```

### Get Sprint Report

Get a comprehensive sprint report.

```http
GET /projects/:projectId/sprints/:sprintId/report
Authorization: Bearer <token>
```

**Response**:

```json
{
  "success": true,
  "data": {
    "sprintId": "sprint-456",
    "name": "Sprint 9",
    "period": {
      "start": "2024-01-15",
      "end": "2024-01-28"
    },
    "velocity": {
      "planned": 25,
      "actual": 23,
      "efficiency": 92
    },
    "summary": {
      "totalItems": 20,
      "completedItems": 18,
      "pendingItems": 2,
      "completionRate": 90
    },
    "quality": {
      "bugsFound": 3,
      "bugsFixed": 3,
      "testCoverage": "85%"
    },
    "achievements": [
      "Completed user authentication",
      "Implemented payment gateway",
      "Deployed to staging"
    ],
    "challenges": [
      "API integration delays",
      "Third-party service downtime"
    ],
    "recommendations": [
      "Add buffer time for API integrations",
      "Implement service health monitoring"
    ],
    "generatedAt": "2024-01-28T18:00:00Z"
  }
}
```

### Get Velocity Report

Get velocity metrics over multiple sprints.

```http
GET /projects/:projectId/velocity?limit=10
Authorization: Bearer <token>
```

**Response**:

```json
{
  "success": true,
  "data": {
    "projectId": "project-123",
    "averageVelocity": 22.5,
    "velocityTrend": "stable",
    "sprints": [
      {
        "sprintId": "sprint-123",
        "name": "Sprint 1",
        "plannedVelocity": 20,
        "actualVelocity": 18
      },
      {
        "sprintId": "sprint-124",
        "name": "Sprint 2",
        "plannedVelocity": 22,
        "actualVelocity": 24
      }
    ]
  }
}
```

---

## Webhooks

### Create Webhook

Create a new webhook.

```http
POST /webhooks
Authorization: Bearer <token>
Content-Type: application/json

{
  "name": "Slack Notifications",
  "url": "https://hooks.slack.com/services/...",
  "events": [
    "workflow.completed",
    "sprint.started",
    "backlog.item.completed"
  ],
  "active": true
}
```

**Response**:

```json
{
  "success": true,
  "data": {
    "id": "webhook-123",
    "name": "Slack Notifications",
    "url": "https://hooks.slack.com/services/...",
    "events": [
      "workflow.completed",
      "sprint.started",
      "backlog.item.completed"
    ],
    "secret": "webhook_secret_123",
    "active": true,
    "createdAt": "2024-01-15T00:00:00Z"
  }
}
```

### List Webhooks

Get a list of all webhooks.

```http
GET /webhooks
Authorization: Bearer <token>
```

### Delete Webhook

Delete a webhook.

```http
DELETE /webhooks/:webhookId
Authorization: Bearer <token>
```

---

## Error Responses

All endpoints may return error responses in the following format:

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid input data",
    "details": [
      {
        "field": "email",
        "message": "Invalid email format"
      }
    ]
  }
}
```

### Common Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `AUTHENTICATION_ERROR` | 401 | Invalid or missing authentication |
| `AUTHORIZATION_ERROR` | 403 | Insufficient permissions |
| `VALIDATION_ERROR` | 400 | Invalid request data |
| `NOT_FOUND` | 404 | Resource not found |
| `CONFLICT_ERROR` | 409 | Resource conflict |
| `RATE_LIMIT_EXCEEDED` | 429 | Too many requests |
| `INTERNAL_ERROR` | 500 | Internal server error |
| `SERVICE_UNAVAILABLE` | 503 | Service temporarily unavailable |

---

## Rate Limiting

API requests are rate-limited:

- **Authenticated users**: 1000 requests per hour
- **Anonymous users**: 100 requests per hour

Rate limit headers are included in responses:

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1642248000
```

---

## SDKs

Official SDKs are available for:

- **JavaScript/TypeScript**: `npm install @virtual-agents/sdk`
- **Python**: `pip install virtual-agents-sdk`
- **Go**: `go get github.com/virtual-agents/sdk-go`

### JavaScript SDK Example

```typescript
import { VirtualAgentsClient } from '@virtual-agents/sdk';

const client = new VirtualAgentsClient({
  apiKey: 'your-api-key',
  baseUrl: 'http://localhost:8000/api/v1'
});

// Get agents
const agents = await client.agents.list();

// Execute workflow
const execution = await client.workflows.execute('daily_standup', {
  projectId: 'project-123'
});

// Get sprint report
const report = await client.projects.getSprintReport('project-123', 'sprint-456');
```

---

## Changelog

### Version 1.0.0 (2024-01-15)
- Initial API release
- Core endpoints for agents, workflows, projects, sprints
- Basic reporting endpoints
- Webhook support

---

## Support

For API support, contact:
- **Email**: api-support@virtual-agents.dev
- **Documentation**: [docs.virtual-agents.dev/api](https://docs.virtual-agents.dev/api)
- **Status Page**: [status.virtual-agents.dev](https://status.virtual-agents.dev)

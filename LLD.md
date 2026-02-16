# Low-Level Design (LLD) - OpenCode Virtual Agile Agents Team

## Document Information

| Field | Value |
|-------|-------|
| **Document Name** | OpenCode Virtual Agile Agents Team - LLD |
| **Version** | 1.0 |
| **Date** | 2026-02-16 |
| **Author** | System Architect |
| **Status** | Initial |

---

## 1. Introduction

### 1.1 Purpose

This Low-Level Design (LLD) document provides detailed technical specifications for the OpenCode Virtual Agile Agents Team system. It complements the High-Level Design (HLD) by diving into implementation details, database schemas, API specifications, and code structures.

### 1.2 Scope

This document covers:
- Detailed database schema and entity models
- API endpoint specifications
- SignalR Hub implementation
- Clean Architecture layer details
- Code structure and organization
- Configuration management

---

## 2. Database Schema

### 2.1 Entity Relationship Diagram

```
┌─────────────┐       ┌──────────────┐       ┌─────────────────┐
│   Agents    │       │   Projects   │       │  Backlog Items  │
├─────────────┤       ├──────────────┤       ├─────────────────┤
│ id (PK)     │       │ id (PK)       │       │ id (PK)          │
│ name        │◄──────│ name          │◄──────│ project_id (FK)  │
│ role        │       │ description   │       │ title            │
│ model       │       │ status        │       │ description      │
│ config      │       │ tech_stack    │       │ type             │
│ status      │       │ created_at    │       │ priority         │
│ created_at  │       │ updated_at    │       │ story_points     │
│ updated_at  │       └──────────────┘       │ status           │
└─────────────┘                              │ sprint_id (FK)   │
                                              │ assigned_agent  │
                                              └─────────────────┘
                                                     │
                                                     │ assigned_agent
                                                     ▼
┌─────────────┐       ┌─────────────────┐       ┌─────────────┐
│   Sprints   │       │ Agent Messages  │       │Pull Requests│
├─────────────┤       ├─────────────────┤       ├─────────────┤
│ id (PK)     │       │ id (PK)         │       │ id (PK)     │
│ project_id  │◄──────│ from_agent (FK) │◄──────│ project_id  │
│ name        │       │ to_agent (FK)   │       │ github_pr_# │
│ start_date  │       │ message_type    │       │ title       │
│ end_date    │       │ intent         │       │ description │
│ status      │       │ payload        │       │ author_agent │
│ planned_vel │       │ timestamp      │       │ status      │
│ actual_vel  │       │ status         │       │ ai_reviewer │
│ created_at  │       └─────────────────┘       │ human_review│
│ updated_at  │                               │ created_at  │
└─────────────┘                               └─────────────┘
        │
        │ sprint_id
        ▼
┌──────────────────────┐       ┌──────────────────────┐
│ Velocity Metrics    │       │ Workflow Executions  │
├──────────────────────┤       ├──────────────────────┤
│ id (PK)             │       │ id (PK)             │
│ sprint_id (FK)       │       │ workflow_id         │
│ metric_type         │       │ status              │
│ value              │       │ started_at          │
│ measured_at        │       │ completed_at        │
└──────────────────────┘       │ input               │
                              │ output              │
                              │ error_message       │
                              └──────────────────────┘
```

### 2.2 Table Definitions

#### 2.2.1 Agents Table

```sql
CREATE TABLE agents (
  id VARCHAR(50) PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  role VARCHAR(50) NOT NULL CHECK (role IN ('PM', 'Architect', 'Tech Lead', 'Developer', 'QA')),
  model VARCHAR(50) NOT NULL CHECK (model IN ('gpt-4o', 'gpt-4o-mini')),
  config JSONB,
  status VARCHAR(20) NOT NULL DEFAULT 'idle' CHECK (status IN ('active', 'idle', 'busy', 'offline')),
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_agents_role ON agents(role);
CREATE INDEX idx_agents_status ON agents(status);
CREATE INDEX idx_agents_model ON agents(model);
```

**Entity Framework Core Model**:
```csharp
namespace VirtualAgents.Api.Data.Models
{
    public class Agent
    {
        [Key]
        [MaxLength(50)]
        public string Id { get; set; } = Guid.NewGuid().ToString();
        
        [Required]
        [MaxLength(100)]
        public string Name { get; set; }
        
        [Required]
        [MaxLength(50)]
        public AgentRole Role { get; set; }
        
        [Required]
        [MaxLength(50)]
        public AgentModel Model { get; set; }
        
        public string Config { get; set; }
        
        [Required]
        [MaxLength(20)]
        public AgentStatus Status { get; set; } = AgentStatus.Idle;
        
        public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
        public DateTime UpdatedAt { get; set; } = DateTime.UtcNow;
    }

    public enum AgentRole
    {
        PM = 1,
        Architect = 2,
        TechLead = 3,
        Developer = 4,
        QA = 5
    }

    public enum AgentModel
    {
        Gpt4o = 1,
        Gpt4oMini = 2
    }

    public enum AgentStatus
    {
        Active = 1,
        Idle = 2,
        Busy = 3,
        Offline = 4
    }
}
```

#### 2.2.2 Projects Table

```sql
CREATE TABLE projects (
  id VARCHAR(50) PRIMARY KEY,
  name VARCHAR(200) NOT NULL,
  description TEXT,
  status VARCHAR(20) NOT NULL DEFAULT 'planning' CHECK (status IN ('planning', 'active', 'on-hold', 'completed')),
  tech_stack JSONB,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_projects_status ON projects(status);
```

**Entity Framework Core Model**:
```csharp
namespace VirtualAgents.Api.Data.Models
{
    public class Project
    {
        [Key]
        [MaxLength(50)]
        public string Id { get; set; } = Guid.NewGuid().ToString();
        
        [Required]
        [MaxLength(200)]
        public string Name { get; set; }
        
        public string Description { get; set; }
        
        [Required]
        [MaxLength(20)]
        public ProjectStatus Status { get; set; } = ProjectStatus.Planning;
        
        public string TechStack { get; set; } // JSON string
        
        public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
        public DateTime UpdatedAt { get; set; } = DateTime.UtcNow;
        
        // Navigation properties
        public virtual ICollection<BacklogItem> BacklogItems { get; set; } = new List<BacklogItem>();
        public virtual ICollection<Sprint> Sprints { get; set; } = new List<Sprint>();
    }

    public enum ProjectStatus
    {
        Planning = 1,
        Active = 2,
        OnHold = 3,
        Completed = 4
    }
}
```

#### 2.2.3 Backlog Items Table

```sql
CREATE TABLE backlog_items (
  id VARCHAR(50) PRIMARY KEY,
  project_id VARCHAR(50) NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  title VARCHAR(200) NOT NULL,
  description TEXT,
  type VARCHAR(20) NOT NULL DEFAULT 'feature' CHECK (type IN ('feature', 'bug', 'enhancement', 'technical-debt')),
  priority VARCHAR(20) NOT NULL DEFAULT 'medium' CHECK (priority IN ('critical', 'high', 'medium', 'low')),
  story_points INTEGER CHECK (story_points >= 1 AND story_points <= 21),
  status VARCHAR(20) NOT NULL DEFAULT 'backlog' CHECK (status IN ('backlog', 'in-progress', 'in-review', 'testing', 'done')),
  sprint_id VARCHAR(50) REFERENCES sprints(id) ON DELETE SET NULL,
  assigned_agent VARCHAR(50) REFERENCES agents(id) ON DELETE SET NULL,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_backlog_project ON backlog_items(project_id);
CREATE INDEX idx_backlog_sprint ON backlog_items(sprint_id);
CREATE INDEX idx_backlog_status ON backlog_items(status);
CREATE INDEX idx_backlog_assigned_agent ON backlog_items(assigned_agent);
```

**Entity Framework Core Model**:
```csharp
namespace VirtualAgents.Api.Data.Models
{
    public class BacklogItem
    {
        [Key]
        [MaxLength(50)]
        public string Id { get; set; } = Guid.NewGuid().ToString();
        
        [Required]
        [MaxLength(50)]
        public string ProjectId { get; set; }
        
        [Required]
        [MaxLength(200)]
        public string Title { get; set; }
        
        public string Description { get; set; }
        
        [Required]
        [MaxLength(20)]
        public BacklogType Type { get; set; } = BacklogType.Feature;
        
        [Required]
        [MaxLength(20)]
        public BacklogPriority Priority { get; set; } = BacklogPriority.Medium;
        
        public int? StoryPoints { get; set; }
        
        [Required]
        [MaxLength(20)]
        public BacklogStatus Status { get; set; } = BacklogStatus.Backlog;
        
        [MaxLength(50)]
        public string? SprintId { get; set; }
        
        [MaxLength(50)]
        public string? AssignedAgentId { get; set; }
        
        public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
        public DateTime UpdatedAt { get; set; } = DateTime.UtcNow;
        
        // Navigation properties
        public virtual Project Project { get; set; }
        public virtual Sprint? Sprint { get; set; }
        public virtual Agent? AssignedAgent { get; set; }
    }

    public enum BacklogType
    {
        Feature = 1,
        Bug = 2,
        Enhancement = 3,
        TechnicalDebt = 4
    }

    public enum BacklogPriority
    {
        Critical = 1,
        High = 2,
        Medium = 3,
        Low = 4
    }

    public enum BacklogStatus
    {
        Backlog = 1,
        InProgress = 2,
        InReview = 3,
        Testing = 4,
        Done = 5
    }
}
```

#### 2.2.4 Sprints Table

```sql
CREATE TABLE sprints (
  id VARCHAR(50) PRIMARY KEY,
  project_id VARCHAR(50) NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  name VARCHAR(100) NOT NULL,
  start_date DATE NOT NULL,
  end_date DATE NOT NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'planned' CHECK (status IN ('planned', 'active', 'completed')),
  planned_velocity INTEGER CHECK (planned_velocity > 0),
  actual_velocity INTEGER CHECK (actual_velocity >= 0),
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_sprints_project ON sprints(project_id);
CREATE INDEX idx_sprints_status ON sprints(status);
CREATE INDEX idx_sprints_dates ON sprints(start_date, end_date);
```

**Entity Framework Core Model**:
```csharp
namespace VirtualAgents.Api.Data.Models
{
    public class Sprint
    {
        [Key]
        [MaxLength(50)]
        public string Id { get; set; } = Guid.NewGuid().ToString();
        
        [Required]
        [MaxLength(50)]
        public string ProjectId { get; set; }
        
        [Required]
        [MaxLength(100)]
        public string Name { get; set; }
        
        [Required]
        public DateTime StartDate { get; set; }
        
        [Required]
        public DateTime EndDate { get; set; }
        
        [Required]
        [MaxLength(20)]
        public SprintStatus Status { get; set; } = SprintStatus.Planned;
        
        public int? PlannedVelocity { get; set; }
        
        public int? ActualVelocity { get; set; }
        
        public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
        public DateTime UpdatedAt { get; set; } = DateTime.UtcNow;
        
        // Navigation properties
        public virtual Project Project { get; set; }
        public virtual ICollection<BacklogItem> BacklogItems { get; set; } = new List<BacklogItem>();
    }

    public enum SprintStatus
    {
        Planned = 1,
        Active = 2,
        Completed = 3
    }
}
```

#### 2.2.5 Agent Messages Table

```sql
CREATE TABLE agent_messages (
  id VARCHAR(50) PRIMARY KEY,
  from_agent VARCHAR(50) NOT NULL REFERENCES agents(id) ON DELETE CASCADE,
  to_agent VARCHAR(50) NOT NULL REFERENCES agents(id) ON DELETE CASCADE,
  message_type VARCHAR(20) NOT NULL CHECK (message_type IN ('command', 'request', 'response', 'event', 'notification')),
  intent VARCHAR(100) NOT NULL,
  payload JSONB NOT NULL,
  timestamp TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  status VARCHAR(20) NOT NULL DEFAULT 'sent' CHECK (status IN ('sent', 'delivered', 'failed'))
);

CREATE INDEX idx_messages_from ON agent_messages(from_agent);
CREATE INDEX idx_messages_to ON agent_messages(to_agent);
CREATE INDEX idx_messages_timestamp ON agent_messages(timestamp DESC);
CREATE INDEX idx_messages_type ON agent_messages(message_type);
```

**Entity Framework Core Model**:
```csharp
namespace VirtualAgents.Api.Data.Models
{
    public class AgentMessage
    {
        [Key]
        [MaxLength(50)]
        public string Id { get; set; } = Guid.NewGuid().ToString();
        
        [Required]
        [MaxLength(50)]
        public string FromAgent { get; set; }
        
        [Required]
        [MaxLength(50)]
        public string ToAgent { get; set; }
        
        [Required]
        [MaxLength(20)]
        public MessageType Type { get; set; }
        
        [Required]
        [MaxLength(100)]
        public string Intent { get; set; }
        
        [Required]
        public string Payload { get; set; } // JSON string
        
        [Required]
        public DateTime Timestamp { get; set; } = DateTime.UtcNow;
        
        [Required]
        [MaxLength(20)]
        public MessageStatus Status { get; set; } = MessageStatus.Sent;
        
        // Navigation properties
        public virtual Agent FromAgentEntity { get; set; }
        public virtual Agent ToAgentEntity { get; set; }
    }

    public enum MessageType
    {
        Command = 1,
        Request = 2,
        Response = 3,
        Event = 4,
        Notification = 5
    }

    public enum MessageStatus
    {
        Sent = 1,
        Delivered = 2,
        Failed = 3
    }
}
```

#### 2.2.6 Workflow Executions Table

```sql
CREATE TABLE workflow_executions (
  id VARCHAR(50) PRIMARY KEY,
  workflow_id VARCHAR(50) NOT NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'running', 'completed', 'failed', 'cancelled')),
  started_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  completed_at TIMESTAMP WITH TIME ZONE,
  input JSONB,
  output JSONB,
  error_message TEXT
);

CREATE INDEX idx_workflow_executions_workflow_id ON workflow_executions(workflow_id);
CREATE INDEX idx_workflow_executions_status ON workflow_executions(status);
CREATE INDEX idx_workflow_executions_started_at ON workflow_executions(started_at DESC);
```

**Entity Framework Core Model**:
```csharp
namespace VirtualAgents.Api.Data.Models
{
    public class WorkflowExecution
    {
        [Key]
        [MaxLength(50)]
        public string Id { get; set; } = Guid.NewGuid().ToString();
        
        [Required]
        [MaxLength(50)]
        public string WorkflowId { get; set; }
        
        [Required]
        [MaxLength(20)]
        public WorkflowStatus Status { get; set; } = WorkflowStatus.Pending;
        
        [Required]
        public DateTime StartedAt { get; set; } = DateTime.UtcNow;
        
        public DateTime? CompletedAt { get; set; }
        
        public string? Input { get; set; } // JSON string
        
        public string? Output { get; set; } // JSON string
        
        public string? ErrorMessage { get; set; }
    }

    public enum WorkflowStatus
    {
        Pending = 1,
        Running = 2,
        Completed = 3,
        Failed = 4,
        Cancelled = 5
    }
}
```

#### 2.2.7 Pull Requests Table

```sql
CREATE TABLE pull_requests (
  id VARCHAR(50) PRIMARY KEY,
  project_id VARCHAR(50) NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  github_pr_number INTEGER NOT NULL UNIQUE,
  title VARCHAR(200) NOT NULL,
  description TEXT,
  author_agent VARCHAR(50) REFERENCES agents(id) ON DELETE SET NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'open' CHECK (status IN ('open', 'in_review', 'merged', 'closed')),
  ai_reviewer VARCHAR(50) REFERENCES agents(id) ON DELETE SET NULL,
  human_reviewer VARCHAR(50),
  ai_review_status VARCHAR(20) DEFAULT 'pending' CHECK (ai_review_status IN ('pending', 'approved', 'changes_requested')),
  human_review_status VARCHAR(20) DEFAULT 'pending' CHECK (human_review_status IN ('pending', 'approved', 'changes_requested')),
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_pull_requests_project ON pull_requests(project_id);
CREATE INDEX idx_pull_requests_github_number ON pull_requests(github_pr_number);
CREATE INDEX idx_pull_requests_status ON pull_requests(status);
CREATE INDEX idx_pull_requests_author ON pull_requests(author_agent);
```

**Entity Framework Core Model**:
```csharp
namespace VirtualAgents.Api.Data.Models
{
    public class PullRequest
    {
        [Key]
        [MaxLength(50)]
        public string Id { get; set; } = Guid.NewGuid().ToString();
        
        [Required]
        [MaxLength(50)]
        public string ProjectId { get; set; }
        
        [Required]
        public int GitHubPrNumber { get; set; }
        
        [Required]
        [MaxLength(200)]
        public string Title { get; set; }
        
        public string Description { get; set; }
        
        [MaxLength(50)]
        public string? AuthorAgent { get; set; }
        
        [Required]
        [MaxLength(20)]
        public PRStatus Status { get; set; } = PRStatus.Open;
        
        [MaxLength(50)]
        public string? AiReviewer { get; set; }
        
        [MaxLength(50)]
        public string? HumanReviewer { get; set; }
        
        [MaxLength(20)]
        public ReviewStatus AiReviewStatus { get; set; } = ReviewStatus.Pending;
        
        [MaxLength(20)]
        public ReviewStatus HumanReviewStatus { get; set; } = ReviewStatus.Pending;
        
        public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
        public DateTime UpdatedAt { get; set; } = DateTime.UtcNow;
        
        // Navigation properties
        public virtual Project Project { get; set; }
        public virtual Agent? AuthorAgentEntity { get; set; }
        public virtual Agent? AiReviewerEntity { get; set; }
    }

    public enum PRStatus
    {
        Open = 1,
        InReview = 2,
        Merged = 3,
        Closed = 4
    }

    public enum ReviewStatus
    {
        Pending = 1,
        Approved = 2,
        ChangesRequested = 3
    }
}
```

#### 2.2.8 Velocity Metrics Table

```sql
CREATE TABLE velocity_metrics (
  id SERIAL PRIMARY KEY,
  sprint_id VARCHAR(50) NOT NULL REFERENCES sprints(id) ON DELETE CASCADE,
  metric_type VARCHAR(50) NOT NULL,
  value NUMERIC NOT NULL,
  measured_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_velocity_sprint ON velocity_metrics(sprint_id);
CREATE INDEX idx_velocity_type ON velocity_metrics(metric_type);
CREATE INDEX idx_velocity_measured_at ON velocity_metrics(measured_at DESC);
```

**Entity Framework Core Model**:
```csharp
namespace VirtualAgents.Api.Data.Models
{
    public class VelocityMetric
    {
        [Key]
        public int Id { get; set; }
        
        [Required]
        [MaxLength(50)]
        public string SprintId { get; set; }
        
        [Required]
        [MaxLength(50)]
        public string MetricType { get; set; }
        
        [Required]
        public decimal Value { get; set; }
        
        public DateTime MeasuredAt { get; set; } = DateTime.UtcNow;
        
        // Navigation properties
        public virtual Sprint Sprint { get; set; }
    }
}
```

### 2.3 DbContext Configuration

```csharp
namespace VirtualAgents.Api.Data.DataSource
{
    public class VirtualAgentsDbContext : DbContext
    {
        public VirtualAgentsDbContext(DbContextOptions<VirtualAgentsDbContext> options)
            : base(options)
        {
        }

        // DbSets
        public DbSet<Agent> Agents { get; set; }
        public DbSet<Project> Projects { get; set; }
        public DbSet<BacklogItem> BacklogItems { get; set; }
        public DbSet<Sprint> Sprints { get; set; }
        public DbSet<AgentMessage> AgentMessages { get; set; }
        public DbSet<WorkflowExecution> WorkflowExecutions { get; set; }
        public DbSet<PullRequest> PullRequests { get; set; }
        public DbSet<VelocityMetric> VelocityMetrics { get; set; }

        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            base.OnModelCreating(modelBuilder);

            // Configure relationships
            modelBuilder.Entity<BacklogItem>()
                .HasOne(b => b.Project)
                .WithMany(p => p.BacklogItems)
                .HasForeignKey(b => b.ProjectId)
                .OnDelete(DeleteBehavior.Cascade);

            modelBuilder.Entity<BacklogItem>()
                .HasOne(b => b.Sprint)
                .WithMany(s => s.BacklogItems)
                .HasForeignKey(b => b.SprintId)
                .OnDelete(DeleteBehavior.SetNull);

            modelBuilder.Entity<BacklogItem>()
                .HasOne(b => b.AssignedAgent)
                .WithMany()
                .HasForeignKey(b => b.AssignedAgentId)
                .OnDelete(DeleteBehavior.SetNull);

            modelBuilder.Entity<Sprint>()
                .HasOne(s => s.Project)
                .WithMany(p => p.Sprints)
                .HasForeignKey(s => s.ProjectId)
                .OnDelete(DeleteBehavior.Cascade);

            modelBuilder.Entity<AgentMessage>()
                .HasOne(m => m.FromAgentEntity)
                .WithMany()
                .HasForeignKey(m => m.FromAgent)
                .OnDelete(DeleteBehavior.Cascade);

            modelBuilder.Entity<AgentMessage>()
                .HasOne(m => m.ToAgentEntity)
                .WithMany()
                .HasForeignKey(m => m.ToAgent)
                .OnDelete(DeleteBehavior.Cascade);

            modelBuilder.Entity<PullRequest>()
                .HasOne(pr => pr.Project)
                .WithMany()
                .HasForeignKey(pr => pr.ProjectId)
                .OnDelete(DeleteBehavior.Cascade);

            modelBuilder.Entity<PullRequest>()
                .HasOne(pr => pr.AuthorAgentEntity)
                .WithMany()
                .HasForeignKey(pr => pr.AuthorAgent)
                .OnDelete(DeleteBehavior.SetNull);

            modelBuilder.Entity<PullRequest>()
                .HasOne(pr => pr.AiReviewerEntity)
                .WithMany()
                .HasForeignKey(pr => pr.AiReviewer)
                .OnDelete(DeleteBehavior.SetNull);

            modelBuilder.Entity<VelocityMetric>()
                .HasOne(vm => vm.Sprint)
                .WithMany()
                .HasForeignKey(vm => vm.SprintId)
                .OnDelete(DeleteBehavior.Cascade);
        }

        protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
        {
            base.OnConfiguring(optionsBuilder);
        }
    }
}
```

---

## 3. API Endpoints Specification

### 3.1 Agents Endpoints

#### GET /api/agents

**Description**: Retrieve all agents

**Response**: `200 OK`
```json
{
  "success": true,
  "data": [
    {
      "id": "agent-001",
      "name": "Project Manager",
      "role": "PM",
      "model": "gpt-4o",
      "status": "idle",
      "createdAt": "2024-01-01T00:00:00Z",
      "updatedAt": "2024-01-01T00:00:00Z"
    }
  ],
  "message": "Agents retrieved successfully"
}
```

#### GET /api/agents/{id}

**Description**: Retrieve a specific agent by ID

**Parameters**:
- `id` (path, required): Agent ID

**Response**: `200 OK`
```json
{
  "success": true,
  "data": {
    "id": "agent-001",
    "name": "Project Manager",
    "role": "PM",
    "model": "gpt-4o",
    "status": "idle",
    "createdAt": "2024-01-01T00:00:00Z",
    "updatedAt": "2024-01-01T00:00:00Z"
  }
}
```

**Response**: `404 Not Found`
```json
{
  "success": false,
  "message": "Agent not found"
}
```

#### POST /api/agents

**Description**: Create a new agent

**Request Body**:
```json
{
  "name": "Frontend Developer",
  "role": "Developer",
  "model": "gpt-4o-mini",
  "config": {
    "specialization": "React/Next.js"
  }
}
```

**Response**: `201 Created`
```json
{
  "success": true,
  "data": {
    "id": "agent-002",
    "name": "Frontend Developer",
    "role": "Developer",
    "model": "gpt-4o-mini",
    "config": {
      "specialization": "React/Next.js"
    },
    "status": "idle",
    "createdAt": "2024-01-01T00:00:00Z",
    "updatedAt": "2024-01-01T00:00:00Z"
  },
  "message": "Agent created successfully"
}
```

#### PUT /api/agents/{id}

**Description**: Update an existing agent

**Parameters**:
- `id` (path, required): Agent ID

**Request Body**:
```json
{
  "name": "Senior Frontend Developer",
  "status": "active"
}
```

**Response**: `200 OK`
```json
{
  "success": true,
  "data": {
    "id": "agent-002",
    "name": "Senior Frontend Developer",
    "role": "Developer",
    "model": "gpt-4o-mini",
    "status": "active",
    "updatedAt": "2024-01-01T00:00:00Z"
  },
  "message": "Agent updated successfully"
}
```

#### DELETE /api/agents/{id}

**Description**: Delete an agent

**Parameters**:
- `id` (path, required): Agent ID

**Response**: `204 No Content`

### 3.2 Workflows Endpoints

#### GET /api/workflows

**Description**: Retrieve all workflows

**Response**: `200 OK`
```json
{
  "success": true,
  "data": [
    {
      "id": "daily-standup",
      "name": "Daily Standup Workflow",
      "description": "Automated daily standup meeting with all agents",
      "triggers": [
        {
          "type": "schedule",
          "cron": "0 8 * * 1-5"
        }
      ]
    }
  ],
  "message": "Workflows retrieved successfully"
}
```

#### POST /api/workflows/{id}/execute

**Description**: Execute a workflow

**Parameters**:
- `id` (path, required): Workflow ID

**Request Body**:
```json
{
  "input": {
    "sprintId": "sprint-001"
  }
}
```

**Response**: `202 Accepted`
```json
{
  "success": true,
  "data": {
    "executionId": "exec-001",
    "workflowId": "daily-standup",
    "status": "running",
    "startedAt": "2024-01-01T08:00:00Z"
  },
  "message": "Workflow execution started"
}
```

#### GET /api/workflows/executions/{executionId}

**Description**: Get workflow execution status

**Parameters**:
- `executionId` (path, required): Execution ID

**Response**: `200 OK`
```json
{
  "success": true,
  "data": {
    "id": "exec-001",
    "workflowId": "daily-standup",
    "status": "completed",
    "startedAt": "2024-01-01T08:00:00Z",
    "completedAt": "2024-01-01T08:30:00Z",
    "output": {
      "report": "Daily standup completed successfully"
    }
  },
  "message": "Execution retrieved successfully"
}
```

### 3.3 Projects Endpoints

#### GET /api/projects

**Description**: Retrieve all projects

**Query Parameters**:
- `status` (optional): Filter by status
- `page` (optional): Page number (default: 1)
- `pageSize` (optional): Items per page (default: 20)

**Response**: `200 OK`
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": "proj-001",
        "name": "E-commerce Platform",
        "description": "Modern e-commerce platform with AI features",
        "status": "active",
        "techStack": {
          "frontend": "React/Next.js",
          "backend": "ASP.NET 8.0",
          "database": "PostgreSQL"
        },
        "createdAt": "2024-01-01T00:00:00Z",
        "updatedAt": "2024-01-01T00:00:00Z"
      }
    ],
    "totalCount": 1,
    "page": 1,
    "pageSize": 20
  },
  "message": "Projects retrieved successfully"
}
```

#### POST /api/projects

**Description**: Create a new project

**Request Body**:
```json
{
  "name": "E-commerce Platform",
  "description": "Modern e-commerce platform with AI features",
  "techStack": {
    "frontend": "React/Next.js",
    "backend": "ASP.NET 8.0",
    "database": "PostgreSQL"
  }
}
```

**Response**: `201 Created`
```json
{
  "success": true,
  "data": {
    "id": "proj-001",
    "name": "E-commerce Platform",
    "description": "Modern e-commerce platform with AI features",
    "status": "planning",
    "techStack": {
      "frontend": "React/Next.js",
      "backend": "ASP.NET 8.0",
      "database": "PostgreSQL"
    },
    "createdAt": "2024-01-01T00:00:00Z",
    "updatedAt": "2024-01-01T00:00:00Z"
  },
  "message": "Project created successfully"
}
```

#### GET /api/projects/{id}/sprints

**Description**: Get all sprints for a project

**Parameters**:
- `id` (path, required): Project ID

**Response**: `200 OK`
```json
{
  "success": true,
  "data": [
    {
      "id": "sprint-001",
      "name": "Sprint 1",
      "startDate": "2024-01-01",
      "endDate": "2024-01-14",
      "status": "completed",
      "plannedVelocity": 50,
      "actualVelocity": 45
    }
  ],
  "message": "Sprints retrieved successfully"
}
```

### 3.4 Reports Endpoints

#### GET /api/reports/daily-standup

**Query Parameters**:
- `date` (required): Date in YYYY-MM-DD format
- `projectId` (optional): Project ID

**Response**: `200 OK`
```json
{
  "success": true,
  "data": {
    "date": "2024-01-01",
    "summary": {
      "tasksCompleted": 12,
      "tasksInProgress": 5,
      "blockers": 2
    },
    "agentUpdates": [
      {
        "agentId": "agent-001",
        "agentName": "PM Agent",
        "completed": ["Sprint planning", "Daily standup"],
        "planned": ["Review backlog", "Coordinate with team"],
        "blockers": ["Waiting on architectural decision"]
      }
    ]
  },
  "message": "Daily standup report generated successfully"
}
```

#### GET /api/reports/sprint/{sprintId}

**Description**: Get sprint summary report

**Parameters**:
- `sprintId` (path, required): Sprint ID

**Response**: `200 OK`
```json
{
  "success": true,
  "data": {
    "sprintId": "sprint-001",
    "name": "Sprint 1",
    "summary": {
      "startDate": "2024-01-01",
      "endDate": "2024-01-14",
      "plannedVelocity": 50,
      "actualVelocity": 45,
      "completionRate": 0.9
    },
    "metrics": {
      "featuresCompleted": 8,
      "bugsFixed": 5,
      "testCoverage": 0.85,
      "codeQuality": 4.2
    },
    "issues": [
      {
        "type": "blocker",
        "description": "Third-party API downtime",
        "impact": "medium"
      }
    ]
  },
  "message": "Sprint report generated successfully"
}
```

#### GET /api/reports/velocity

**Query Parameters**:
- `projectId` (optional): Project ID
- `sprintCount` (optional): Number of sprints to include (default: 5)

**Response**: `200 OK`
```json
{
  "success": true,
  "data": {
    "projectId": "proj-001",
    "sprints": [
      {
        "sprintId": "sprint-001",
        "name": "Sprint 1",
        "plannedVelocity": 50,
        "actualVelocity": 45
      },
      {
        "sprintId": "sprint-002",
        "name": "Sprint 2",
        "plannedVelocity": 50,
        "actualVelocity": 52
      }
    ],
    "averageVelocity": 48.5,
    "velocityTrend": "increasing"
  },
  "message": "Velocity report generated successfully"
}
```

---

## 4. SignalR Hub Implementation

### 4.1 AgentHub

```csharp
namespace VirtualAgents.Api.Presentation.Hubs
{
    [Authorize]
    public class AgentHub : Hub
    {
        private readonly ILogger<AgentHub> _logger;
        private readonly IAgentService _agentService;
        private readonly IWorkflowService _workflowService;

        public AgentHub(
            ILogger<AgentHub> logger,
            IAgentService agentService,
            IWorkflowService workflowService)
        {
            _logger = logger;
            _agentService = agentService;
            _workflowService = workflowService;
        }

        /// <summary>
        /// Join an agent group to receive updates
        /// </summary>
        /// <param name="agentId">Agent ID</param>
        public async Task JoinAgentGroup(string agentId)
        {
            await Groups.AddToGroupAsync(Context.ConnectionId, agentId);
            _logger.LogInformation("User {ConnectionId} joined agent {AgentId}", Context.ConnectionId, agentId);
        }

        /// <summary>
        /// Join all agents group
        /// </summary>
        public async Task JoinAllAgentsGroup()
        {
            await Groups.AddToGroupAsync(Context.ConnectionId, "all-agents");
            _logger.LogInformation("User {ConnectionId} joined all agents group", Context.ConnectionId);
        }

        /// <summary>
        /// Join a workflow group to receive execution updates
        /// </summary>
        /// <param name="workflowId">Workflow execution ID</param>
        public async Task SubscribeToWorkflow(string executionId)
        {
            await Groups.AddToGroupAsync(Context.ConnectionId, $"workflow-{executionId}");
            _logger.LogInformation("User {ConnectionId} subscribed to workflow {ExecutionId}", Context.ConnectionId, executionId);
        }

        /// <summary>
        /// Send a message to a specific agent
        /// </summary>
        /// <param name="agentId">Agent ID</param>
        /// <param name="message">Message content</param>
        public async Task SendAgentMessage(string agentId, AgentMessage message)
        {
            await Clients.Group(agentId).SendAsync("ReceiveMessage", message);
            _logger.LogInformation("Message sent to agent {AgentId}: {Intent}", agentId, message.Intent);
        }

        /// <summary>
        /// Broadcast agent status change to all clients
        /// </summary>
        /// <param name="agentId">Agent ID</param>
        /// <param name="status">New status</param>
        public async Task BroadcastAgentStatus(string agentId, string status)
        {
            await Clients.All.SendAsync("AgentStatusChanged", new
            {
                AgentId = agentId,
                Status = status,
                Timestamp = DateTime.UtcNow
            });
            _logger.LogInformation("Agent {AgentId} status changed to {Status}", agentId, status);
        }

        /// <summary>
        /// Broadcast workflow progress update
        /// </summary>
        /// <param name="executionId">Workflow execution ID</param>
        /// <param name="stage">Current stage</param>
        /// <param name="progress">Progress percentage (0-100)</param>
        public async Task BroadcastWorkflowProgress(string executionId, string stage, int progress)
        {
            await Clients.All.SendAsync("WorkflowProgressUpdated", new
            {
                ExecutionId = executionId,
                Stage = stage,
                Progress = progress,
                Timestamp = DateTime.UtcNow
            });
            _logger.LogInformation("Workflow {ExecutionId} progress: {Stage} - {Progress}%", executionId, stage, progress);
        }

        /// <summary>
        /// Client connected
        /// </summary>
        public override async Task OnConnectedAsync()
        {
            _logger.LogInformation("Client connected: {ConnectionId}", Context.ConnectionId);
            await base.OnConnectedAsync();
        }

        /// <summary>
        /// Client disconnected
        /// </summary>
        public override async Task OnDisconnectedAsync(Exception exception)
        {
            _logger.LogInformation("Client disconnected: {ConnectionId}", Context.ConnectionId);
            await base.OnDisconnectedAsync(exception);
        }
    }
}
```

### 4.2 SignalR Client (JavaScript)

```javascript
// wwwroot/js/custom/signalr.js
$(document).ready(function() {
    // Initialize SignalR connection
    const connection = new signalR.HubConnectionBuilder()
        .withUrl("/hubs/agents")
        .configureLogging(signalR.LogLevel.Information)
        .withAutomaticReconnect([0, 2000, 10000, 30000]) // Retry intervals
        .build();

    // Start connection
    startConnection(connection);

    // Register event handlers
    registerEventHandlers(connection);

    // Connection state handlers
    connection.onreconnecting(error => {
        console.log("SignalR reconnecting...", error);
        showConnectionStatus("reconnecting");
    });

    connection.onreconnected(connectionId => {
        console.log("SignalR reconnected", connectionId);
        showConnectionStatus("connected");
        // Rejoin groups after reconnect
        connection.invoke("JoinAllAgentsGroup");
    });

    connection.onclose(error => {
        console.log("SignalR connection closed", error);
        showConnectionStatus("disconnected");
    });
});

function startConnection(connection) {
    connection.start().then(function() {
        console.log("SignalR Connected!");
        showConnectionStatus("connected");
        
        // Join agent groups
        connection.invoke("JoinAllAgentsGroup").catch(function(err) {
            console.error("Error joining agent group:", err);
        });
    }).catch(function(err) {
        console.error("SignalR Connection Error:", err);
        showConnectionStatus("error");
        
        // Retry connection after 5 seconds
        setTimeout(() => startConnection(connection), 5000);
    });
}

function registerEventHandlers(connection) {
    // Receive agent messages
    connection.on("ReceiveMessage", function(message) {
        console.log("Message received:", message);
        updateAgentMessageUI(message);
        
        // Show notification for important messages
        if (message.priority === "high" || message.priority === "urgent") {
            showNotification(message);
        }
    });

    // Listen for agent status changes
    connection.on("AgentStatusChanged", function(data) {
        console.log("Agent status changed:", data);
        updateAgentStatusUI(data.AgentId, data.Status);
        
        // Update agent status indicator
        $(`#agent-status-${data.AgentId}`)
            .removeClass('active idle busy offline')
            .addClass(data.Status.toLowerCase());
    });

    // Listen for workflow progress updates
    connection.on("WorkflowProgressUpdated", function(data) {
        console.log("Workflow progress updated:", data);
        updateWorkflowProgressUI(data.ExecutionId, data.Stage, data.Progress);
        
        // Update progress bar
        const progressBar = $(`#workflow-progress-${data.ExecutionId}`);
        if (progressBar.length) {
            progressBar.css('width', `${data.Progress}%`);
            progressBar.attr('aria-valuenow', data.Progress);
        }
    });
}

function updateAgentMessageUI(message) {
    // Add message to conversation log
    const messageHtml = `
        <div class="message message-${message.type.toLowerCase()}">
            <div class="message-header">
                <strong>${message.from}</strong>
                <span class="message-time">${formatTimestamp(message.timestamp)}</span>
            </div>
            <div class="message-content">${message.intent}</div>
            <div class="message-payload">${formatPayload(message.payload)}</div>
        </div>
    `;
    
    $('#conversation-log').append(messageHtml);
    scrollToBottom('#conversation-log');
}

function updateAgentStatusUI(agentId, status) {
    const statusBadge = $(`#agent-status-badge-${agentId}`);
    if (statusBadge.length) {
        statusBadge.text(status.toUpperCase());
        statusBadge
            .removeClass('bg-success bg-warning bg-danger bg-secondary')
            .addClass(getStatusClass(status));
    }
}

function updateWorkflowProgressUI(executionId, stage, progress) {
    const stageElement = $(`#workflow-stage-${executionId}`);
    if (stageElement.length) {
        stageElement.text(stage);
    }
    
    const progressElement = $(`#workflow-progress-${executionId}`);
    if (progressElement.length) {
        progressElement.text(`${progress}%`);
    }
}

function showNotification(message) {
    // Use Metronic's toast notification
    KTApp.showPageNotification({
        message: message.intent,
        type: 'warning',
        placement: 'top-right',
        time: 5000
    });
}

function showConnectionStatus(status) {
    const statusIndicator = $('#connection-status');
    if (statusIndicator.length) {
        statusIndicator
            .removeClass('connected disconnected error reconnecting')
            .addClass(status);
        
        const statusText = {
            'connected': 'Connected',
            'disconnected': 'Disconnected',
            'error': 'Connection Error',
            'reconnecting': 'Reconnecting...'
        };
        
        statusIndicator.text(statusText[status] || status);
    }
}

function getStatusClass(status) {
    const statusClasses = {
        'active': 'bg-success',
        'idle': 'bg-secondary',
        'busy': 'bg-warning',
        'offline': 'bg-danger'
    };
    return statusClasses[status] || 'bg-secondary';
}

function formatTimestamp(timestamp) {
    const date = new Date(timestamp);
    return date.toLocaleTimeString();
}

function formatPayload(payload) {
    if (typeof payload === 'string') {
        return payload;
    }
    return JSON.stringify(payload, null, 2);
}

function scrollToBottom(element) {
    $(element).animate({ scrollTop: $(element).prop('scrollHeight') }, 300);
}
```

---

## 5. Clean Architecture Implementation

### 5.1 Layer Dependencies

```
Presentation Layer (Controllers, Hubs, DTOs)
    ↓ depends on
Domain Layer (Repository Interfaces, Domain Services)
    ↓ depends on
Data Layer (DbContext, Entities, Repository Implementations)
```

### 5.2 Presentation Layer

**Namespace**: `VirtualAgents.Api.Presentation`

**Components**:
- Controllers: REST API endpoints
- Hubs: SignalR hubs
- DTOs: Data Transfer Objects
- Requests/Responses: API request/response models

#### 5.2.1 Controller Example

```csharp
namespace VirtualAgents.Api.Presentation.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public class AgentsController : ControllerBase
    {
        private readonly AgentService _agentService;
        private readonly ILogger<AgentsController> _logger;

        public AgentsController(
            AgentService agentService,
            ILogger<AgentsController> logger)
        {
            _agentService = agentService;
            _logger = logger;
        }

        [HttpGet("")]
        public async Task<IActionResult> GetAll(CancellationToken cancellationToken)
        {
            try
            {
                var agents = await _agentService.GetAllAgentsAsync(cancellationToken);
                return Ok(new ApiResponse<List<AgentDto>>
                {
                    Success = true,
                    Data = agents,
                    Message = "Agents retrieved successfully"
                });
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error fetching agents");
                return StatusCode(500, new ApiResponse<object>
                {
                    Success = false,
                    Message = "An error occurred while fetching agents"
                });
            }
        }

        [HttpGet("{id}")]
        public async Task<IActionResult> GetById(string id, CancellationToken cancellationToken)
        {
            try
            {
                var agent = await _agentService.GetAgentAsync(id, cancellationToken);
                if (agent == null)
                {
                    return NotFound(new ApiResponse<object>
                    {
                        Success = false,
                        Message = $"Agent {id} not found"
                    });
                }

                return Ok(new ApiResponse<AgentDto>
                {
                    Success = true,
                    Data = agent
                });
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error fetching agent {AgentId}", id);
                return StatusCode(500, new ApiResponse<object>
                {
                    Success = false,
                    Message = "An error occurred while fetching agent"
                });
            }
        }

        [HttpPost("")]
        public async Task<IActionResult> Create([FromBody] CreateAgentRequest request, CancellationToken cancellationToken)
        {
            try
            {
                var agent = await _agentService.CreateAgentAsync(request, cancellationToken);
                return CreatedAtAction(nameof(GetById), new { id = agent.Id }, new ApiResponse<AgentDto>
                {
                    Success = true,
                    Data = agent,
                    Message = "Agent created successfully"
                });
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error creating agent");
                return StatusCode(500, new ApiResponse<object>
                {
                    Success = false,
                    Message = "An error occurred while creating agent"
                });
            }
        }
    }
}
```

#### 5.2.2 DTOs

```csharp
namespace VirtualAgents.Api.Presentation.DTOs
{
    // Agent DTO
    public class AgentDto
    {
        public string Id { get; set; }
        public string Name { get; set; }
        public string Role { get; set; }
        public string Model { get; set; }
        public string Status { get; set; }
        public DateTime CreatedAt { get; set; }
        public DateTime UpdatedAt { get; set; }
    }

    // Request DTOs
    public class CreateAgentRequest
    {
        [Required]
        [MaxLength(100)]
        public string Name { get; set; }
        
        [Required]
        [MaxLength(50)]
        public string Role { get; set; }
        
        [Required]
        [MaxLength(50)]
        public string Model { get; set; }
        
        public string Config { get; set; }
    }

    public class UpdateAgentRequest
    {
        [MaxLength(100)]
        public string Name { get; set; }
        
        [MaxLength(20)]
        public string Status { get; set; }
    }

    // Response DTOs
    public class ApiResponse<T>
    {
        public bool Success { get; set; }
        public T Data { get; set; }
        public string Message { get; set; }
    }

    // Workflow DTOs
    public class WorkflowDto
    {
        public string Id { get; set; }
        public string Name { get; set; }
        public string Description { get; set; }
        public List<TriggerDto> Triggers { get; set; }
    }

    public class TriggerDto
    {
        public string Type { get; set; }
        public string Cron { get; set; }
    }

    public class WorkflowExecutionDto
    {
        public string ExecutionId { get; set; }
        public string WorkflowId { get; set; }
        public string Status { get; set; }
        public DateTime StartedAt { get; set; }
        public DateTime? CompletedAt { get; set; }
        public object Output { get; set; }
    }
}
```

### 5.3 Domain Layer

**Namespace**: `VirtualAgents.Api.Domain`

**Components**:
- Repository Interfaces: Contracts for data access
- Domain Services: Business logic implementations
- Value Objects: Domain-specific objects
- Domain Events: Event handling

#### 5.3.1 Repository Interfaces

```csharp
namespace VirtualAgents.Api.Domain.Repository.Interfaces
{
    public interface IAgentRepository
    {
        Task<List<Agent>> GetAllAsync(CancellationToken cancellationToken = default);
        Task<Agent?> GetByIdAsync(string id, CancellationToken cancellationToken = default);
        Task<Agent> CreateAsync(Agent agent, CancellationToken cancellationToken = default);
        Task UpdateAsync(Agent agent, CancellationToken cancellationToken = default);
        Task DeleteAsync(string id, CancellationToken cancellationToken = default);
        Task<List<Agent>> GetByRoleAsync(string role, CancellationToken cancellationToken = default);
        Task<List<Agent>> GetByStatusAsync(string status, CancellationToken cancellationToken = default);
    }

    public interface IProjectRepository
    {
        Task<List<Project>> GetAllAsync(CancellationToken cancellationToken = default);
        Task<Project?> GetByIdAsync(string id, CancellationToken cancellationToken = default);
        Task<Project> CreateAsync(Project project, CancellationToken cancellationToken = default);
        Task UpdateAsync(Project project, CancellationToken cancellationToken = default);
        Task DeleteAsync(string id, CancellationToken cancellationToken = default);
    }

    public interface IBacklogItemRepository
    {
        Task<List<BacklogItem>> GetAllAsync(CancellationToken cancellationToken = default);
        Task<BacklogItem?> GetByIdAsync(string id, CancellationToken cancellationToken = default);
        Task<List<BacklogItem>> GetByProjectIdAsync(string projectId, CancellationToken cancellationToken = default);
        Task<List<BacklogItem>> GetBySprintIdAsync(string sprintId, CancellationToken cancellationToken = default);
        Task<BacklogItem> CreateAsync(BacklogItem item, CancellationToken cancellationToken = default);
        Task UpdateAsync(BacklogItem item, CancellationToken cancellationToken = default);
        Task DeleteAsync(string id, CancellationToken cancellationToken = default);
    }

    public interface ISprintRepository
    {
        Task<List<Sprint>> GetAllAsync(CancellationToken cancellationToken = default);
        Task<Sprint?> GetByIdAsync(string id, CancellationToken cancellationToken = default);
        Task<List<Sprint>> GetByProjectIdAsync(string projectId, CancellationToken cancellationToken = default);
        Task<Sprint> CreateAsync(Sprint sprint, CancellationToken cancellationToken = default);
        Task UpdateAsync(Sprint sprint, CancellationToken cancellationToken = default);
        Task DeleteAsync(string id, CancellationToken cancellationToken = default);
    }

    public interface IWorkflowExecutionRepository
    {
        Task<List<WorkflowExecution>> GetAllAsync(CancellationToken cancellationToken = default);
        Task<WorkflowExecution?> GetByIdAsync(string id, CancellationToken cancellationToken = default);
        Task<WorkflowExecution> CreateAsync(WorkflowExecution execution, CancellationToken cancellationToken = default);
        Task UpdateAsync(WorkflowExecution execution, CancellationToken cancellationToken = default);
        Task<List<WorkflowExecution>> GetByWorkflowIdAsync(string workflowId, CancellationToken cancellationToken = default);
    }

    public interface IPullRequestRepository
    {
        Task<List<PullRequest>> GetAllAsync(CancellationToken cancellationToken = default);
        Task<PullRequest?> GetByIdAsync(string id, CancellationToken cancellationToken = default);
        Task<PullRequest?> GetByGithubNumberAsync(int githubPrNumber, CancellationToken cancellationToken = default);
        Task<List<PullRequest>> GetByProjectIdAsync(string projectId, CancellationToken cancellationToken = default);
        Task<PullRequest> CreateAsync(PullRequest pullRequest, CancellationToken cancellationToken = default);
        Task UpdateAsync(PullRequest pullRequest, CancellationToken cancellationToken = default);
    }
}
```

#### 5.3.2 Domain Services

```csharp
namespace VirtualAgents.Api.Domain.Services
{
    public class AgentService
    {
        private readonly IAgentRepository _agentRepository;
        private readonly ILogger<AgentService> _logger;
        private readonly ICacheService _cacheService;

        public AgentService(
            IAgentRepository agentRepository,
            ILogger<AgentService> logger,
            ICacheService cacheService)
        {
            _agentRepository = agentRepository;
            _logger = logger;
            _cacheService = cacheService;
        }

        public async Task<List<AgentDto>> GetAllAgentsAsync(CancellationToken cancellationToken = default)
        {
            _logger.LogInformation("Fetching all agents");
            
            // Try cache first
            var cachedAgents = await _cacheService.GetAsync<List<AgentDto>>("agents:all");
            if (cachedAgents != null)
            {
                return cachedAgents;
            }
            
            // Fetch from database
            var agents = await _agentRepository.GetAllAsync(cancellationToken);
            var agentDtos = agents.Select(MapToDto).ToList();
            
            // Cache for 5 minutes
            await _cacheService.SetAsync("agents:all", agentDtos, TimeSpan.FromMinutes(5));
            
            return agentDtos;
        }

        public async Task<AgentDto?> GetAgentAsync(string id, CancellationToken cancellationToken = default)
        {
            _logger.LogInformation("Fetching agent {AgentId}", id);
            
            // Try cache first
            var cachedAgent = await _cacheService.GetAsync<AgentDto>($"agents:{id}");
            if (cachedAgent != null)
            {
                return cachedAgent;
            }
            
            // Fetch from database
            var agent = await _agentRepository.GetByIdAsync(id, cancellationToken);
            if (agent == null)
            {
                return null;
            }
            
            var agentDto = MapToDto(agent);
            
            // Cache for 5 minutes
            await _cacheService.SetAsync($"agents:{id}", agentDto, TimeSpan.FromMinutes(5));
            
            return agentDto;
        }

        public async Task<AgentDto> CreateAgentAsync(CreateAgentRequest request, CancellationToken cancellationToken = default)
        {
            _logger.LogInformation("Creating agent {AgentName}", request.Name);
            
            var agent = new Agent
            {
                Id = Guid.NewGuid().ToString(),
                Name = request.Name,
                Role = Enum.Parse<AgentRole>(request.Role),
                Model = Enum.Parse<AgentModel>(request.Model),
                Config = request.Config,
                Status = AgentStatus.Idle,
                CreatedAt = DateTime.UtcNow,
                UpdatedAt = DateTime.UtcNow
            };
            
            var createdAgent = await _agentRepository.CreateAsync(agent, cancellationToken);
            
            // Invalidate cache
            await _cacheService.RemoveAsync("agents:all");
            
            return MapToDto(createdAgent);
        }

        private AgentDto MapToDto(Agent agent)
        {
            return new AgentDto
            {
                Id = agent.Id,
                Name = agent.Name,
                Role = agent.Role.ToString(),
                Model = agent.Model.ToString(),
                Status = agent.Status.ToString(),
                CreatedAt = agent.CreatedAt,
                UpdatedAt = agent.UpdatedAt
            };
        }
    }

    public class WorkflowService
    {
        private readonly IWorkflowExecutionRepository _workflowExecutionRepository;
        private readonly ILogger<WorkflowService> _logger;
        private readonly IOpenCodeService _openCodeService;

        public WorkflowService(
            IWorkflowExecutionRepository workflowExecutionRepository,
            ILogger<WorkflowService> logger,
            IOpenCodeService openCodeService)
        {
            _workflowExecutionRepository = workflowExecutionRepository;
            _logger = logger;
            _openCodeService = openCodeService;
        }

        public async Task<WorkflowExecutionDto> ExecuteWorkflowAsync(string workflowId, object input, CancellationToken cancellationToken = default)
        {
            _logger.LogInformation("Executing workflow {WorkflowId}", workflowId);
            
            var execution = new WorkflowExecution
            {
                Id = Guid.NewGuid().ToString(),
                WorkflowId = workflowId,
                Status = WorkflowStatus.Running,
                StartedAt = DateTime.UtcNow,
                Input = JsonSerializer.Serialize(input)
            };
            
            execution = await _workflowExecutionRepository.CreateAsync(execution, cancellationToken);
            
            // Execute workflow logic
            await ExecuteWorkflowLogicAsync(execution, cancellationToken);
            
            return MapToDto(execution);
        }

        private async Task ExecuteWorkflowLogicAsync(WorkflowExecution execution, CancellationToken cancellationToken)
        {
            try
            {
                // Get workflow definition
                var workflow = GetWorkflowDefinition(execution.WorkflowId);
                
                // Execute stages
                foreach (var stage in workflow.Stages)
                {
                    await ExecuteStageAsync(stage, execution, cancellationToken);
                }
                
                execution.Status = WorkflowStatus.Completed;
                execution.CompletedAt = DateTime.UtcNow;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Workflow execution failed");
                execution.Status = WorkflowStatus.Failed;
                execution.ErrorMessage = ex.Message;
                execution.CompletedAt = DateTime.UtcNow;
            }
            finally
            {
                await _workflowExecutionRepository.UpdateAsync(execution, cancellationToken);
            }
        }

        private async Task ExecuteStageAsync(WorkflowStage stage, WorkflowExecution execution, CancellationToken cancellationToken)
        {
            _logger.LogInformation("Executing stage {StageName}", stage.Name);
            
            // Broadcast progress via SignalR
            // This would be injected as a service
            // await _hubService.BroadcastWorkflowProgress(execution.Id, stage.Name, progress);
            
            // Execute tasks in stage
            foreach (var task in stage.Tasks)
            {
                await ExecuteTaskAsync(task, cancellationToken);
            }
        }

        private async Task ExecuteTaskAsync(WorkflowTask task, CancellationToken cancellationToken)
        {
            // Execute task using OpenCode agents
            var result = await _openCodeService.ExecuteAgentTaskAsync(task.Agent, task.Action, task.Input);
            
            _logger.LogInformation("Task executed: {Agent} - {Action}", task.Agent, task.Action);
        }

        private Workflow GetWorkflowDefinition(string workflowId)
        {
            // Load workflow definition from configuration or database
            // This is a placeholder
            return new Workflow();
        }

        private WorkflowExecutionDto MapToDto(WorkflowExecution execution)
        {
            return new WorkflowExecutionDto
            {
                ExecutionId = execution.Id,
                WorkflowId = execution.WorkflowId,
                Status = execution.Status.ToString(),
                StartedAt = execution.StartedAt,
                CompletedAt = execution.CompletedAt,
                Output = execution.Output != null ? JsonSerializer.Deserialize<object>(execution.Output) : null
            };
        }
    }

    public class OpenCodeService
    {
        private readonly HttpClient _httpClient;
        private readonly string _apiKey;
        private readonly ILogger<OpenCodeService> _logger;

        public OpenCodeService(HttpClient httpClient, IConfiguration config)
        {
            _httpClient = httpClient;
            _apiKey = config["OpenCode:ApiKey"];
            _httpClient.BaseAddress = new Uri(config["OpenCode:ApiUrl"]);
            _httpClient.DefaultRequestHeaders.Add("Authorization", $"Bearer {_apiKey}");
            _logger = logger;
        }

        public async Task<AgentResponse> SendAgentMessageAsync(string agentId, AgentMessage message, CancellationToken cancellationToken = default)
        {
            _logger.LogInformation("Sending message to agent {AgentId}", agentId);
            
            var request = new OpenCodeRequest
            {
                AgentId = agentId,
                Message = message
            };
            
            var response = await _httpClient.PostAsJsonAsync("/agents/message", request, cancellationToken);
            response.EnsureSuccessStatusCode();
            
            var result = await response.Content.ReadFromJsonAsync<AgentResponse>(cancellationToken: cancellationToken);
            
            return result;
        }

        public async Task<object> ExecuteAgentTaskAsync(string agentId, string action, object input, CancellationToken cancellationToken = default)
        {
            _logger.LogInformation("Executing task {Action} on agent {AgentId}", action, agentId);
            
            var request = new
            {
                agentId = agentId,
                action = action,
                input = input
            };
            
            var response = await _httpClient.PostAsJsonAsync("/agents/execute", request, cancellationToken);
            response.EnsureSuccessStatusCode();
            
            var result = await response.Content.ReadFromJsonAsync<object>(cancellationToken: cancellationToken);
            
            return result;
        }
    }
}
```

### 5.4 Data Layer

**Namespace**: `VirtualAgents.Api.Data`

**Components**:
- DataSource: DbContext
- Models: Entity models (already defined in Section 2)
- Repository: Repository implementations

#### 5.4.1 Repository Implementations

```csharp
namespace VirtualAgents.Api.Data.Repository
{
    public class AgentRepository : IAgentRepository
    {
        private readonly VirtualAgentsDbContext _context;

        public AgentRepository(VirtualAgentsDbContext context)
        {
            _context = context;
        }

        public async Task<List<Agent>> GetAllAsync(CancellationToken cancellationToken = default)
        {
            return await _context.Agents
                .OrderBy(a => a.Name)
                .ToListAsync(cancellationToken);
        }

        public async Task<Agent?> GetByIdAsync(string id, CancellationToken cancellationToken = default)
        {
            return await _context.Agents
                .FirstOrDefaultAsync(a => a.Id == id, cancellationToken);
        }

        public async Task<Agent> CreateAsync(Agent agent, CancellationToken cancellationToken = default)
        {
            _context.Agents.Add(agent);
            await _context.SaveChangesAsync(cancellationToken);
            return agent;
        }

        public async Task UpdateAsync(Agent agent, CancellationToken cancellationToken = default)
        {
            _context.Entry(agent).State = EntityState.Modified;
            await _context.SaveChangesAsync(cancellationToken);
        }

        public async Task DeleteAsync(string id, CancellationToken cancellationToken = default)
        {
            var agent = await GetByIdAsync(id, cancellationToken);
            if (agent != null)
            {
                _context.Agents.Remove(agent);
                await _context.SaveChangesAsync(cancellationToken);
            }
        }

        public async Task<List<Agent>> GetByRoleAsync(string role, CancellationToken cancellationToken = default)
        {
            return await _context.Agents
                .Where(a => a.Role.ToString() == role)
                .OrderBy(a => a.Name)
                .ToListAsync(cancellationToken);
        }

        public async Task<List<Agent>> GetByStatusAsync(string status, CancellationToken cancellationToken = default)
        {
            return await _context.Agents
                .Where(a => a.Status.ToString() == status)
                .OrderBy(a => a.Name)
                .ToListAsync(cancellationToken);
        }
    }

    public class ProjectRepository : IProjectRepository
    {
        private readonly VirtualAgentsDbContext _context;

        public ProjectRepository(VirtualAgentsDbContext context)
        {
            _context = context;
        }

        public async Task<List<Project>> GetAllAsync(CancellationToken cancellationToken = default)
        {
            return await _context.Projects
                .OrderByDescending(p => p.CreatedAt)
                .ToListAsync(cancellationToken);
        }

        public async Task<Project?> GetByIdAsync(string id, CancellationToken cancellationToken = default)
        {
            return await _context.Projects
                .FirstOrDefaultAsync(p => p.Id == id, cancellationToken);
        }

        public async Task<Project> CreateAsync(Project project, CancellationToken cancellationToken = default)
        {
            _context.Projects.Add(project);
            await _context.SaveChangesAsync(cancellationToken);
            return project;
        }

        public async Task UpdateAsync(Project project, CancellationToken cancellationToken = default)
        {
            _context.Entry(project).State = EntityState.Modified;
            await _context.SaveChangesAsync(cancellationToken);
        }

        public async Task DeleteAsync(string id, CancellationToken cancellationToken = default)
        {
            var project = await GetByIdAsync(id, cancellationToken);
            if (project != null)
            {
                _context.Projects.Remove(project);
                await _context.SaveChangesAsync(cancellationToken);
            }
        }
    }

    public class BacklogItemRepository : IBacklogItemRepository
    {
        private readonly VirtualAgentsDbContext _context;

        public BacklogItemRepository(VirtualAgentsDbContext context)
        {
            _context = context;
        }

        public async Task<List<BacklogItem>> GetAllAsync(CancellationToken cancellationToken = default)
        {
            return await _context.BacklogItems
                .Include(b => b.Project)
                .Include(b => b.Sprint)
                .Include(b => b.AssignedAgent)
                .OrderByDescending(b => b.CreatedAt)
                .ToListAsync(cancellationToken);
        }

        public async Task<BacklogItem?> GetByIdAsync(string id, CancellationToken cancellationToken = default)
        {
            return await _context.BacklogItems
                .Include(b => b.Project)
                .Include(b => b.Sprint)
                .Include(b => b.AssignedAgent)
                .FirstOrDefaultAsync(b => b.Id == id, cancellationToken);
        }

        public async Task<List<BacklogItem>> GetByProjectIdAsync(string projectId, CancellationToken cancellationToken = default)
        {
            return await _context.BacklogItems
                .Where(b => b.ProjectId == projectId)
                .Include(b => b.Sprint)
                .Include(b => b.AssignedAgent)
                .OrderByDescending(b => b.CreatedAt)
                .ToListAsync(cancellationToken);
        }

        public async Task<List<BacklogItem>> GetBySprintIdAsync(string sprintId, CancellationToken cancellationToken = default)
        {
            return await _context.BacklogItems
                .Where(b => b.SprintId == sprintId)
                .Include(b => b.AssignedAgent)
                .OrderBy(b => b.Priority)
                .ToListAsync(cancellationToken);
        }

        public async Task<BacklogItem> CreateAsync(BacklogItem item, CancellationToken cancellationToken = default)
        {
            _context.BacklogItems.Add(item);
            await _context.SaveChangesAsync(cancellationToken);
            return item;
        }

        public async Task UpdateAsync(BacklogItem item, CancellationToken cancellationToken = default)
        {
            _context.Entry(item).State = EntityState.Modified;
            await _context.SaveChangesAsync(cancellationToken);
        }

        public async Task DeleteAsync(string id, CancellationToken cancellationToken = default)
        {
            var item = await GetByIdAsync(id, cancellationToken);
            if (item != null)
            {
                _context.BacklogItems.Remove(item);
                await _context.SaveChangesAsync(cancellationToken);
            }
        }
    }

    public class SprintRepository : ISprintRepository
    {
        private readonly VirtualAgentsDbContext _context;

        public SprintRepository(VirtualAgentsDbContext context)
        {
            _context = context;
        }

        public async Task<List<Sprint>> GetAllAsync(CancellationToken cancellationToken = default)
        {
            return await _context.Sprints
                .Include(s => s.Project)
                .OrderByDescending(s => s.StartDate)
                .ToListAsync(cancellationToken);
        }

        public async Task<Sprint?> GetByIdAsync(string id, CancellationToken cancellationToken = default)
        {
            return await _context.Sprints
                .Include(s => s.Project)
                .FirstOrDefaultAsync(s => s.Id == id, cancellationToken);
        }

        public async Task<List<Sprint>> GetByProjectIdAsync(string projectId, CancellationToken cancellationToken = default)
        {
            return await _context.Sprints
                .Where(s => s.ProjectId == projectId)
                .OrderByDescending(s => s.StartDate)
                .ToListAsync(cancellationToken);
        }

        public async Task<Sprint> CreateAsync(Sprint sprint, CancellationToken cancellationToken = default)
        {
            _context.Sprints.Add(sprint);
            await _context.SaveChangesAsync(cancellationToken);
            return sprint;
        }

        public async Task UpdateAsync(Sprint sprint, CancellationToken cancellationToken = default)
        {
            _context.Entry(sprint).State = EntityState.Modified;
            await _context.SaveChangesAsync(cancellationToken);
        }

        public async Task DeleteAsync(string id, CancellationToken cancellationToken = default)
        {
            var sprint = await GetByIdAsync(id, cancellationToken);
            if (sprint != null)
            {
                _context.Sprints.Remove(sprint);
                await _context.SaveChangesAsync(cancellationToken);
            }
        }
    }

    public class WorkflowExecutionRepository : IWorkflowExecutionRepository
    {
        private readonly VirtualAgentsDbContext _context;

        public WorkflowExecutionRepository(VirtualAgentsDbContext context)
        {
            _context = context;
        }

        public async Task<List<WorkflowExecution>> GetAllAsync(CancellationToken cancellationToken = default)
        {
            return await _context.WorkflowExecutions
                .OrderByDescending(w => w.StartedAt)
                .ToListAsync(cancellationToken);
        }

        public async Task<WorkflowExecution?> GetByIdAsync(string id, CancellationToken cancellationToken = default)
        {
            return await _context.WorkflowExecutions
                .FirstOrDefaultAsync(w => w.Id == id, cancellationToken);
        }

        public async Task<WorkflowExecution> CreateAsync(WorkflowExecution execution, CancellationToken cancellationToken = default)
        {
            _context.WorkflowExecutions.Add(execution);
            await _context.SaveChangesAsync(cancellationToken);
            return execution;
        }

        public async Task UpdateAsync(WorkflowExecution execution, CancellationToken cancellationToken = default)
        {
            _context.Entry(execution).State = EntityState.Modified;
            await _context.SaveChangesAsync(cancellationToken);
        }

        public async Task<List<WorkflowExecution>> GetByWorkflowIdAsync(string workflowId, CancellationToken cancellationToken = default)
        {
            return await _context.WorkflowExecutions
                .Where(w => w.WorkflowId == workflowId)
                .OrderByDescending(w => w.StartedAt)
                .ToListAsync(cancellationToken);
        }
    }

    public class PullRequestRepository : IPullRequestRepository
    {
        private readonly VirtualAgentsDbContext _context;

        public PullRequestRepository(VirtualAgentsDbContext context)
        {
            _context = context;
        }

        public async Task<List<PullRequest>> GetAllAsync(CancellationToken cancellationToken = default)
        {
            return await _context.PullRequests
                .Include(pr => pr.Project)
                .Include(pr => pr.AuthorAgentEntity)
                .Include(pr => pr.AiReviewerEntity)
                .OrderByDescending(pr => pr.CreatedAt)
                .ToListAsync(cancellationToken);
        }

        public async Task<PullRequest?> GetByIdAsync(string id, CancellationToken cancellationToken = default)
        {
            return await _context.PullRequests
                .Include(pr => pr.Project)
                .Include(pr => pr.AuthorAgentEntity)
                .Include(pr => pr.AiReviewerEntity)
                .FirstOrDefaultAsync(pr => pr.Id == id, cancellationToken);
        }

        public async Task<PullRequest?> GetByGithubNumberAsync(int githubPrNumber, CancellationToken cancellationToken = default)
        {
            return await _context.PullRequests
                .FirstOrDefaultAsync(pr => pr.GithubPrNumber == githubPrNumber, cancellationToken);
        }

        public async Task<List<PullRequest>> GetByProjectIdAsync(string projectId, CancellationToken cancellationToken = default)
        {
            return await _context.PullRequests
                .Where(pr => pr.ProjectId == projectId)
                .Include(pr => pr.AuthorAgentEntity)
                .OrderByDescending(pr => pr.CreatedAt)
                .ToListAsync(cancellationToken);
        }

        public async Task<PullRequest> CreateAsync(PullRequest pullRequest, CancellationToken cancellationToken = default)
        {
            _context.PullRequests.Add(pullRequest);
            await _context.SaveChangesAsync(cancellationToken);
            return pullRequest;
        }

        public async Task UpdateAsync(PullRequest pullRequest, CancellationToken cancellationToken = default)
        {
            _context.Entry(pullRequest).State = EntityState.Modified;
            await _context.SaveChangesAsync(cancellationToken);
        }
    }
}
```

---

## 6. Configuration Management

### 6.1 appsettings.json

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.EntityFrameworkCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "ConnectionStrings": {
    "DefaultConnection": "postgresql://admin:password@localhost:5432/virtual_agents"
  },
  "Redis": {
    "ConnectionString": "localhost:6379"
  },
  "OpenCode": {
    "ApiUrl": "https://api.opencode.dev",
    "ApiKey": ""
  },
  "GitHub": {
    "ApiUrl": "https://api.github.com",
    "Token": "",
    "Repository": "username/repo",
    "Branch": "main"
  },
  "Slack": {
    "WebhookUrl": ""
  },
  "Jwt": {
    "SecretKey": "",
    "ExpiryMinutes": 60
  }
}
```

### 6.2 appsettings.Development.json

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "Microsoft.AspNetCore": "Information",
      "Microsoft.EntityFrameworkCore": "Information"
    }
  },
  "ConnectionStrings": {
    "DefaultConnection": "postgresql://admin:password@localhost:5432/virtual_agents_dev"
  }
}
```

### 6.3 Program.cs (API)

```csharp
var builder = WebApplication.CreateBuilder(args);

// Add services to the container
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Database
builder.Services.AddDbContext<VirtualAgentsDbContext>(options =>
    options.UseNpgsql(builder.Configuration.GetConnectionString("DefaultConnection")));

// Redis
builder.Services.AddSingleton<IConnectionMultiplexer>(sp =>
    ConnectionMultiplexer.Connect(builder.Configuration.GetConnectionString("Redis")!));

// SignalR
builder.Services.AddSignalR(options =>
{
    options.EnableDetailedErrors = builder.Environment.IsDevelopment();
    options.KeepAliveInterval = TimeSpan.FromSeconds(10);
});

// Register repositories
builder.Services.AddScoped<IAgentRepository, AgentRepository>();
builder.Services.AddScoped<IProjectRepository, ProjectRepository>();
builder.Services.AddScoped<IBacklogItemRepository, BacklogItemRepository>();
builder.Services.AddScoped<ISprintRepository, SprintRepository>();
builder.Services.AddScoped<IWorkflowExecutionRepository, WorkflowExecutionRepository>();
builder.Services.AddScoped<IPullRequestRepository, PullRequestRepository>();

// Register domain services
builder.Services.AddScoped<AgentService>();
builder.Services.AddScoped<ProjectService>();
builder.Services.AddScoped<WorkflowService>();
builder.Services.AddScoped<OpenCodeService>();
builder.Services.AddScoped<GitHubService>();
builder.Services.AddScoped<SlackService>();

// Register infrastructure services
builder.Services.AddHttpClient<IOpenCodeService, OpenCodeService>();
builder.Services.AddHttpClient<IGitHubService, GitHubService>();
builder.Services.AddHttpClient<ISlackService, SlackService>();
builder.Services.AddScoped<ICacheService, RedisCacheService>();

// CORS
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowAll",
        policy => policy
            .AllowAnyOrigin()
            .AllowAnyMethod()
            .AllowAnyHeader());
});

var app = builder.Build();

// Configure the HTTP request pipeline
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseCors("AllowAll");
app.UseAuthorization();
app.MapControllers();
app.MapHub<AgentHub>("/hubs/agents");

app.Run();
```

---

## 7. WebApp Implementation Details

### 7.1 Program.cs (WebApp)

```csharp
var builder = WebApplication.CreateBuilder(args);

// Add services to the container
builder.Services.AddControllersWithViews();
builder.Services.AddHttpContextAccessor();

// Session
builder.Services.AddSession(options =>
{
    options.IdleTimeout = TimeSpan.FromMinutes(30);
    options.Cookie.HttpOnly = true;
    options.Cookie.IsEssential = true;
    options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
    options.Cookie.SameSite = SameSiteMode.Lax;
});

// Authentication
builder.Services.AddAuthentication("Cookies")
    .AddCookie("Cookies", options =>
    {
        options.LoginPath = "/Auth/Login";
        options.AccessDeniedPath = "/Auth/AccessDenied";
        options.ExpireTimeSpan = TimeSpan.FromHours(8);
    });

// HTTP Client for API
builder.Services.AddHttpClient("VirtualAgentsApi", client =>
{
    client.BaseAddress = new Uri(builder.Configuration["Api:BaseUrl"]!);
    client.Timeout = TimeSpan.FromSeconds(30);
});

// Register services
builder.Services.AddScoped<IAgentApiService, AgentApiService>();
builder.Services.AddScoped<IProjectApiService, ProjectApiService>();
builder.Services.AddScoped<IWorkflowApiService, WorkflowApiService>();
builder.Services.AddScoped<ISignalRService, SignalRService>();

var app = builder.Build();

// Configure the HTTP request pipeline
if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Home/Error");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();
app.UseSession();
app.UseAuthentication();
app.UseAuthorization();

app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");

app.Run();
```

### 7.2 API Service (WebApp)

```csharp
namespace VirtualAgents.WebApp.Application.Services
{
    public class AgentApiService
    {
        private readonly HttpClient _httpClient;
        private readonly ILogger<AgentApiService> _logger;
        private readonly IHttpContextAccessor _httpContextAccessor;

        public AgentApiService(
            IHttpClientFactory httpClientFactory,
            ILogger<AgentApiService> logger,
            IHttpContextAccessor httpContextAccessor)
        {
            _httpClient = httpClientFactory.CreateClient("VirtualAgentsApi");
            _logger = logger;
            _httpContextAccessor = httpContextAccessor;
        }

        public async Task<List<AgentViewModel>> GetAllAgentsAsync(CancellationToken cancellationToken = default)
        {
            try
            {
                var response = await _httpClient.GetAsync("/api/agents", cancellationToken);
                response.EnsureSuccessStatusCode();
                
                var apiResponse = await response.Content.ReadFromJsonAsync<ApiResponse<List<AgentViewModel>>>(cancellationToken: cancellationToken);
                
                return apiResponse?.Data ?? new List<AgentViewModel>();
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error fetching agents from API");
                throw;
            }
        }

        public async Task<AgentViewModel?> GetAgentAsync(string id, CancellationToken cancellationToken = default)
        {
            try
            {
                var response = await _httpClient.GetAsync($"/api/agents/{id}", cancellationToken);
                
                if (response.StatusCode == HttpStatusCode.NotFound)
                {
                    return null;
                }
                
                response.EnsureSuccessStatusCode();
                
                var apiResponse = await response.Content.ReadFromJsonAsync<ApiResponse<AgentViewModel>>(cancellationToken: cancellationToken);
                
                return apiResponse?.Data;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error fetching agent {AgentId} from API", id);
                throw;
            }
        }

        public async Task<AgentViewModel> CreateAgentAsync(CreateAgentViewModel model, CancellationToken cancellationToken = default)
        {
            try
            {
                var response = await _httpClient.PostAsJsonAsync("/api/agents", model, cancellationToken);
                response.EnsureSuccessStatusCode();
                
                var apiResponse = await response.Content.ReadFromJsonAsync<ApiResponse<AgentViewModel>>(cancellationToken: cancellationToken);
                
                return apiResponse?.Data;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error creating agent");
                throw;
            }
        }
    }
}
```

### 7.3 Controller Example (WebApp)

```csharp
namespace VirtualAgents.WebApp.Presentation.Controllers
{
    public class AgentsController : Controller
    {
        private readonly IAgentApiService _agentApiService;
        private readonly ISignalRService _signalRService;
        private readonly ILogger<AgentsController> _logger;

        public AgentsController(
            IAgentApiService agentApiService,
            ISignalRService signalRService,
            ILogger<AgentsController> logger)
        {
            _agentApiService = agentApiService;
            _signalRService = signalRService;
            _logger = logger;
        }

        [HttpGet("")]
        public async Task<IActionResult> Index()
        {
            try
            {
                var agents = await _agentApiService.GetAllAgentsAsync();
                var viewModel = new AgentListViewModel
                {
                    Agents = agents
                };
                
                return View(viewModel);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error fetching agents");
                return View("Error");
            }
        }

        [HttpGet("{id}")]
        public async Task<IActionResult> Details(string id)
        {
            try
            {
                var agent = await _agentApiService.GetAgentAsync(id);
                if (agent == null)
                {
                    return NotFound();
                }
                
                var viewModel = new AgentDetailsViewModel
                {
                    Agent = agent
                };
                
                return View(viewModel);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error fetching agent details");
                return View("Error");
            }
        }
    }
}
```

---

## 8. Appendix

### 8.1 Migration Commands

```bash
# Add migration
dotnet ef migrations add InitialCreate --project VirtualAgents.Api --startup-project VirtualAgents.Api

# Update database
dotnet ef database update --project VirtualAgents.Api --startup-project VirtualAgents.Api

# Remove last migration
dotnet ef migrations remove --project VirtualAgents.Api --startup-project VirtualAgents.Api
```

### 8.2 Docker Build Commands

```bash
# Build API
docker build -f ./src/VirtualAgents.Api/Dockerfile -t virtual-agents-api:latest ./src/VirtualAgents.Api

# Build WebApp
docker build -f ./src/VirtualAgents.WebApp/Dockerfile -t virtual-agents-webapp:latest ./src/VirtualAgents.WebApp

# Build Orchestrator
docker build -f ./src/VirtualAgents.Orchestrator/Dockerfile -t virtual-agents-orchestrator:latest ./src/VirtualAgents.Orchestrator
```

### 8.3 Testing Commands

```bash
# Run all tests
dotnet test

# Run specific project tests
dotnet test ./tests/VirtualAgents.Api.Tests

# Run tests with coverage
dotnet test --collect:"XPlat Code Coverage"
```

---

**End of Document**

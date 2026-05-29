# Document 2: API Mock Data and Integration Guide

**File**: `docs/api-mock.md`  
**Status**: Immediate Need

---

# SimuCorp — API Mock Data and Integration Guide

## 1. Overview

This document provides mock data for all REST APIs and WebSocket events, for use by the frontend Agent and backend Agent during independent development. Both frontend and backend should use this document as the interface contract.

## 2. Mock Mode Configuration

**Frontend Environment Variables** (`.env.development`):
```bash
VITE_USE_MOCK=true          # Enable Mock mode
VITE_GATEWAY_WS_URL=ws://localhost:18789
VITE_GATEWAY_API_URL=http://localhost:18789/api/v1
```

**Mock File Directory Structure**:
```
frontend/src/api/__mocks__/
├── handlers.ts             # MSW handlers (HTTP Mock)
├── ws-events.ts            # WebSocket event simulation
├── data/
│   ├── agents.json         # Agent list mock data
│   ├── projects.json       # Project list mock data
│   ├── meetings.json       # Meeting mock data
│   ├── approvals.json      # Approval mock data
│   └── branches.json       # Branch mock data
└── scenarios/
    ├── task-flow.ts        # Task execution flow simulation
    ├── recruitment.ts      # Recruitment flow simulation
    └── meeting-flow.ts     # Meeting flow simulation
```

## 3. REST API Mock Data

### 3.1 GET /api/v1/agents — Agent List

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "items": [
      {
        "agent_id": "ea",
        "name": "Executive Assistant",
        "role": "Executive Assistant",
        "department": null,
        "status": "online",
        "employment_status": "active",
        "model": "gpt-5",
        "current_task_count": 0,
        "success_rate_30d": 0.99,
        "capabilities": {
          "skills": ["coordination", "routing", "meeting_management"],
          "level": "principal"
        }
      },
      {
        "agent_id": "hr_manager",
        "name": "HR Manager",
        "role": "HR Manager",
        "department": "dept-hr",
        "status": "online",
        "employment_status": "active",
        "model": "gpt-5",
        "current_task_count": 1,
        "success_rate_30d": 0.97,
        "capabilities": {
          "skills": ["recruitment", "performance_evaluation", "soul_optimization"],
          "level": "senior"
        }
      },
      {
        "agent_id": "gtf_manager",
        "name": "General Task Force Manager",
        "role": "General Task Force Manager",
        "department": "dept-gtf",
        "status": "busy",
        "employment_status": "active",
        "model": "claude-sonnet-4",
        "current_task_count": 2,
        "success_rate_30d": 0.93,
        "capabilities": {
          "skills": ["general", "code", "analysis", "writing"],
          "level": "senior"
        }
      },
      {
        "agent_id": "ios_dev_intern_01",
        "name": "iOS Developer (Intern)",
        "role": "iOS Developer",
        "department": "dept-gtf",
        "status": "busy",
        "employment_status": "intern",
        "model": "claude-sonnet-4",
        "current_task_count": 1,
        "success_rate_30d": 0.85,
        "capabilities": {
          "skills": ["Swift", "SwiftUI", "iOS"],
          "level": "junior"
        },
        "internship": {
          "started_at": "2026-06-15T00:00:00Z",
          "ends_at": "2026-06-22T00:00:00Z",
          "tasks_completed": 4,
          "tasks_target": 10,
          "current_score": 82
        }
      }
    ],
    "total": 4,
    "page": 1,
    "page_size": 20
  }
}
```

### 3.2 GET /api/v1/agents/:agentId — Agent Details

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "agent_id": "gtf_manager",
    "name": "General Task Force Manager",
    "role": "General Task Force Manager",
    "department": "dept-gtf",
    "manager_agent_id": "ea",
    "status": "busy",
    "employment_status": "active",
    "runtime": {
      "type": "claude-code",
      "endpoint": "http://cc-node-01:3000"
    },
    "model_config": {
      "primary": "claude-sonnet-4",
      "fallbacks": ["deepseek-coder", "qwen-plus"],
      "temperature": 0.3,
      "max_tokens": 8192,
      "monthly_budget_usd": 50
    },
    "tools": [
      { "name": "code_execute", "risk_level": "high", "allocated_at": "2026-06-01T00:00:00Z" },
      { "name": "web_search", "risk_level": "low", "allocated_at": "2026-06-01T00:00:00Z" },
      { "name": "memory_store", "risk_level": "low", "allocated_at": "2026-06-01T00:00:00Z" },
      { "name": "memory_retrieve", "risk_level": "low", "allocated_at": "2026-06-01T00:00:00Z" }
    ],
    "capabilities": {
      "skills": ["general", "code", "analysis", "writing"],
      "domain": ["general"],
      "level": "senior"
    },
    "created_at": "2026-06-01T00:00:00Z",
    "recent_tasks": [
      {
        "task_id": "task-003",
        "subject": "Generate Q2 Sales Data Report",
        "status": "completed",
        "completed_at": "2026-06-15T11:00:00Z"
      },
      {
        "task_id": "task-004",
        "subject": "Fix Official Website Mobile Adaptation Issue",
        "status": "in_progress",
        "started_at": "2026-06-15T13:00:00Z"
      }
    ]
  }
}
```

### 3.3 GET /api/v1/system/health — System Health

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "gateway": "healthy",
    "message_bus": "healthy",
    "model_gateway": "healthy",
    "openproject": "healthy",
    "chromadb": "healthy",
    "agents_online": 4,
    "agents_total": 4,
    "branches_online": 1,
    "branches_total": 1
  }
}
```

### 3.4 GET /api/v1/system/metrics — System Metrics

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "agents": {
      "online": 3,
      "total": 4,
      "busy": 2,
      "idle": 1,
      "error": 0,
      "offline": 1
    },
    "tasks": {
      "completed_24h": 12,
      "failed_24h": 1,
      "success_rate_24h": 0.923,
      "in_progress": 3,
      "queued": 2
    },
    "models": {
      "calls_24h": 247,
      "avg_latency_ms": 850,
      "cost_24h_usd": 12.3,
      "fallback_count_24h": 2
    },
    "branches": {
      "online": 1,
      "total": 1
    }
  }
}
```

### 3.5 POST /api/v1/projects — Create Project

Request Body:
```json
{
  "name": "Q3 Financial Report Analysis System v2.0",
  "description": "Refactor Q3 financial report analysis system to support subsidiary data",
  "tool": "openproject",
  "git_repos": ["https://gitlab.com/finance/q3-report"],
  "wiki_url": "https://wiki.example.com/finance",
  "pm_agent_id": null,
  "initial_departments": ["rd", "da"]
}
```

Response:
```json
{
  "code": 0,
  "message": "Project created successfully",
  "data": {
    "project_id": "proj-q3-finance",
    "op_project_id": 42,
    "pm_agent_id": "pm_q3_finance",
    "op_url": "https://openproject.example.com/projects/42",
    "wiki_url": "https://wiki.example.com/finance/q3"
  }
}
```

### 3.6 GET /api/v1/projects — Project List

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "items": [
      {
        "project_id": "proj-q3-finance",
        "name": "Q3 Financial Report Analysis System v2.0",
        "status": "active",
        "progress_percent": 67,
        "pm_agent_id": "pm_q3_finance",
        "task_stats": {
          "total": 12,
          "done": 8,
          "in_progress": 3,
          "todo": 1,
          "blocked": 0
        },
        "last_activity": "2026-06-15T14:30:00Z"
      },
      {
        "project_id": "proj-user-profile",
        "name": "User Profile Data Platform",
        "status": "active",
        "progress_percent": 34,
        "pm_agent_id": "pm_user_profile",
        "task_stats": {
          "total": 15,
          "done": 4,
          "in_progress": 6,
          "todo": 4,
          "blocked": 1
        },
        "last_activity": "2026-06-15T12:00:00Z",
        "blocked_reason": "Data collection plan pending confirmation"
      }
    ],
    "total": 2,
    "page": 1,
    "page_size": 20
  }
}
```

### 3.7 GET /api/v1/approvals — Approval List

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "items": [
      {
        "approval_id": "approval-dept-001",
        "type": "NEW_DEPARTMENT",
        "petitioner": "gtf_manager",
        "content": {
          "department_name": "Mobile Development Department",
          "description": "Responsible for iOS and Android application development",
          "justification": "Mobile tasks accounted for 35% in the last 30 days, with 4 related Agents already"
        },
        "priority": "high",
        "level": "L3",
        "status": "pending",
        "created_at": "2026-06-15T10:00:00Z",
        "expires_at": "2026-06-18T10:00:00Z"
      },
      {
        "approval_id": "approval-branch-001",
        "type": "BRANCH_ACCESS",
        "petitioner": "branch_shanghai_ea",
        "content": {
          "branch_id": "branch_shanghai",
          "hostname": "gpu-node-01",
          "capabilities": {
            "hardware": ["GPU-A100-80GB"],
            "specialty": ["model_training"]
          }
        },
        "priority": "medium",
        "level": "L3",
        "status": "pending",
        "created_at": "2026-06-15T08:00:00Z",
        "expires_at": "2026-06-18T08:00:00Z"
      }
    ],
    "total": 2,
    "page": 1,
    "page_size": 20
  }
}
```

### 3.8 GET /api/v1/meetings — Meeting List

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "items": [
      {
        "meeting_id": "meeting-q3-001",
        "subject": "Q3 Data Source Confirmation Meeting",
        "project_id": "proj-q3-finance",
        "host_agent_id": "ea",
        "status": "ended",
        "participants": ["ea", "rd_manager", "da_01", "ceo"],
        "started_at": "2026-06-15T14:30:00Z",
        "ended_at": "2026-06-15T14:38:00Z",
        "minutes_summary": "Confirmed inclusion of subsidiary data; API architecture needs to change from single source to multi-source aggregation",
        "action_items_count": 3
      },
      {
        "meeting_id": "meeting-data-001",
        "subject": "Data Collection Plan Discussion",
        "project_id": "proj-user-profile",
        "host_agent_id": null,
        "status": "pending",
        "participants": ["rd_manager", "da_01", "ceo"],
        "requested_at": "2026-06-15T15:00:00Z"
      }
    ],
    "total": 2,
    "page": 1,
    "page_size": 20
  }
}
```

### 3.9 GET /api/v1/branches — Branch List

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "items": [
      {
        "branch_id": "branch_shanghai",
        "hostname": "gpu-node-01",
        "status": "online",
        "capabilities": {
          "hardware": ["GPU-A100-80GB", "64GB-RAM"],
          "software": ["docker", "cuda12.2", "python3.10"],
          "specialty": ["model_training", "video_processing"]
        },
        "load": {
          "cpu_percent": 52,
          "memory_used_gb": 32,
          "gpu_percent": 85,
          "active_tasks": 3,
          "max_concurrent_tasks": 5
        },
        "last_heartbeat": "2026-06-15T15:00:00Z",
        "registered_at": "2026-06-10T00:00:00Z"
      }
    ],
    "total": 1,
    "page": 1,
    "page_size": 20
  }
}
```

## 4. WebSocket Event Simulation

### 4.1 Simulated Event Push Script

In frontend Mock mode, use the following script to simulate the WebSocket event stream:

```typescript
// frontend/src/api/__mocks__/ws-events.ts

// Simulated event sequence — Task execution flow
export const taskExecutionEvents = [
  {
    delay: 0,
    event: "agent.task_started",
    data: {
      agent_id: "gtf_manager",
      task_id: "task-004",
      task_summary: "Fix Official Website Mobile Adaptation Issue",
      started_at: "2026-06-15T13:00:00Z"
    }
  },
  {
    delay: 5000,
    event: "agent.task_progress",
    data: {
      agent_id: "gtf_manager",
      task_id: "task-004",
      progress_percent: 50,
      message: "Issue located: Incorrect CSS media query breakpoint settings"
    }
  },
  {
    delay: 10000,
    event: "agent.task_completed",
    data: {
      agent_id: "gtf_manager",
      task_id: "task-004",
      result_summary: "Fixed 3 CSS breakpoints; mobile adaptation restored to normal",
      duration_minutes: 12
    }
  }
];

// Simulated event sequence — Meeting flow
export const meetingFlowEvents = [
  {
    delay: 0,
    event: "meeting.state_changed",
    data: {
      meeting_id: "meeting-q3-002",
      state: "active",
      started_at: "2026-06-15T15:30:00Z"
    }
  },
  {
    delay: 2000,
    event: "meeting.message_received",
    data: {
      meeting_id: "meeting-q3-002",
      sender: "ea",
      content: "Meeting topic: Confirm Q3 requirement priorities. R&D department, please present current progress first.",
      timestamp: "2026-06-15T15:30:30Z"
    }
  },
  {
    delay: 5000,
    event: "meeting.message_received",
    data: {
      meeting_id: "meeting-q3-002",
      sender: "rd_manager",
      content: "Currently, the backend API is 70% complete, frontend is 40% complete. Priority suggestion: Complete the data cleaning pipeline first, then develop the visualization interface.",
      timestamp: "2026-06-15T15:31:00Z"
    }
  },
  {
    delay: 8000,
    event: "meeting.message_received",
    data: {
      meeting_id: "meeting-q3-002",
      sender: "ceo",
      content: "Agreed. R&D department completes the data cleaning pipeline this week; Data department provides the test dataset by Friday.",
      timestamp: "2026-06-15T15:31:30Z"
    }
  },
  {
    delay: 10000,
    event: "meeting.state_changed",
    data: {
      meeting_id: "meeting-q3
# 文档二：接口Mock数据与联调指南

**文件**：`docs/api-mock.md`  
**状态**：立即需要

---

# 模拟公司 (SimuCorp) — 接口Mock数据与联调指南

## 1. 概述

本文档提供所有REST API和WebSocket事件的Mock数据，供前端Agent和后端Agent独立开发时使用。前后端都以此文档为接口契约。

## 2. Mock模式设置

**前端环境变量**（`.env.development`）：
```bash
VITE_USE_MOCK=true          # 启用Mock模式
VITE_GATEWAY_WS_URL=ws://localhost:18789
VITE_GATEWAY_API_URL=http://localhost:18789/api/v1
```

**Mock文件目录结构**：
```
frontend/src/api/__mocks__/
├── handlers.ts             # MSW handlers（HTTP Mock）
├── ws-events.ts            # WebSocket事件模拟
├── data/
│   ├── agents.json         # Agent列表Mock数据
│   ├── projects.json       # 项目列表Mock数据
│   ├── meetings.json       # 会议Mock数据
│   ├── approvals.json      # 审批Mock数据
│   └── branches.json       # 分公司Mock数据
└── scenarios/
    ├── task-flow.ts        # 任务执行流程模拟
    ├── recruitment.ts      # 招聘流程模拟
    └── meeting-flow.ts     # 会议流程模拟
```

## 3. REST API Mock数据

### 3.1 GET /api/v1/agents — Agent列表

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "items": [
      {
        "agent_id": "ea",
        "name": "总裁助理",
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
        "name": "人事部经理",
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
        "name": "机动部经理",
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
        "name": "iOS开发工程师（实习）",
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

### 3.2 GET /api/v1/agents/:agentId — Agent详情

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "agent_id": "gtf_manager",
    "name": "机动部经理",
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
        "subject": "生成Q2销售数据报表",
        "status": "completed",
        "completed_at": "2026-06-15T11:00:00Z"
      },
      {
        "task_id": "task-004",
        "subject": "修复官网移动端适配问题",
        "status": "in_progress",
        "started_at": "2026-06-15T13:00:00Z"
      }
    ]
  }
}
```

### 3.3 GET /api/v1/system/health — 系统健康

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

### 3.4 GET /api/v1/system/metrics — 系统指标

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

### 3.5 POST /api/v1/projects — 创建项目

请求体：
```json
{
  "name": "Q3财报分析系统 v2.0",
  "description": "重构Q3财报分析系统，支持子公司数据",
  "tool": "openproject",
  "git_repos": ["https://gitlab.com/finance/q3-report"],
  "wiki_url": "https://wiki.example.com/finance",
  "pm_agent_id": null,
  "initial_departments": ["rd", "da"]
}
```

响应：
```json
{
  "code": 0,
  "message": "项目创建成功",
  "data": {
    "project_id": "proj-q3-finance",
    "op_project_id": 42,
    "pm_agent_id": "pm_q3_finance",
    "op_url": "https://openproject.example.com/projects/42",
    "wiki_url": "https://wiki.example.com/finance/q3"
  }
}
```

### 3.6 GET /api/v1/projects — 项目列表

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "items": [
      {
        "project_id": "proj-q3-finance",
        "name": "Q3财报分析系统 v2.0",
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
        "name": "用户画像数据平台",
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
        "blocked_reason": "数据采集方案待确认"
      }
    ],
    "total": 2,
    "page": 1,
    "page_size": 20
  }
}
```

### 3.7 GET /api/v1/approvals — 审批列表

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
          "department_name": "移动开发部",
          "description": "负责iOS和Android应用开发",
          "justification": "近30天移动端任务占比达35%，已有4名相关Agent"
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

### 3.8 GET /api/v1/meetings — 会议列表

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "items": [
      {
        "meeting_id": "meeting-q3-001",
        "subject": "Q3数据源确认会议",
        "project_id": "proj-q3-finance",
        "host_agent_id": "ea",
        "status": "ended",
        "participants": ["ea", "rd_manager", "da_01", "ceo"],
        "started_at": "2026-06-15T14:30:00Z",
        "ended_at": "2026-06-15T14:38:00Z",
        "minutes_summary": "确认包含子公司数据，API架构需从单源改为多源聚合",
        "action_items_count": 3
      },
      {
        "meeting_id": "meeting-data-001",
        "subject": "数据采集方案讨论",
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

### 3.9 GET /api/v1/branches — 分公司列表

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

## 4. WebSocket事件模拟

### 4.1 模拟事件推送脚本

前端Mock模式下，使用以下脚本模拟WebSocket事件流：

```typescript
// frontend/src/api/__mocks__/ws-events.ts

// 模拟事件序列 — 任务执行流程
export const taskExecutionEvents = [
  {
    delay: 0,
    event: "agent.task_started",
    data: {
      agent_id: "gtf_manager",
      task_id: "task-004",
      task_summary: "修复官网移动端适配问题",
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
      message: "已定位问题：CSS媒体查询断点设置错误"
    }
  },
  {
    delay: 10000,
    event: "agent.task_completed",
    data: {
      agent_id: "gtf_manager",
      task_id: "task-004",
      result_summary: "修复了3处CSS断点，移动端适配恢复正常",
      duration_minutes: 12
    }
  }
];

// 模拟事件序列 — 会议流程
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
      content: "本次会议议题：确认Q3需求优先级。请研发部先陈述当前进展。",
      timestamp: "2026-06-15T15:30:30Z"
    }
  },
  {
    delay: 5000,
    event: "meeting.message_received",
    data: {
      meeting_id: "meeting-q3-002",
      sender: "rd_manager",
      content: "目前后端API已完成70%，前端完成40%。优先级建议：先完成数据清洗管道，再开发可视化界面。",
      timestamp: "2026-06-15T15:31:00Z"
    }
  },
  {
    delay: 8000,
    event: "meeting.message_received",
    data: {
      meeting_id: "meeting-q3-002",
      sender: "ceo",
      content: "同意。研发部本周内完成数据清洗管道，数据部周五前提供测试数据集。",
      timestamp: "2026-06-15T15:31:30Z"
    }
  },
  {
    delay: 10000,
    event: "meeting.state_changed",
    data: {
      meeting_id: "meeting-q3-002",
      state: "ended",
      minutes: {
        summary: "确认优先开发数据清洗管道，可视化界面后置",
        action_items: [
          { assignee: "rd_manager", task: "完成数据清洗管道", deadline: "2026-06-20" },
          { assignee: "da_01", task: "提供测试数据集", deadline: "2026-06-19" }
        ]
      }
    }
  }
];

// 模拟Agent状态变更
export const agentStatusEvents = [
  {
    delay: 0,
    event: "agent.status_changed",
    data: { agent_id: "gtf_manager", old_status: "idle", new_status: "busy" }
  },
  {
    delay: 15000,
    event: "agent.status_changed",
    data: { agent_id: "gtf_manager", old_status: "busy", new_status: "idle" }
  }
];

// 模拟系统告警
export const systemAlertEvents = [
  {
    delay: 0,
    event: "system.alert",
    data: {
      level: "warn",
      message: "月度模型用量达50%",
      timestamp: "2026-06-15T10:00:00Z"
    }
  },
  {
    delay: 3000,
    event: "system.alert",
    data: {
      level: "critical",
      message: "深圳分公司离线超过15分钟",
      timestamp: "2026-06-15T14:32:00Z"
    }
  }
];
```

## 5. 联调切换指南

**Phase 1 联调步骤**：

1. 后端Agent完成Gateway基础配置和 `system.get_health` RPC方法
2. 前端Agent切换 `VITE_USE_MOCK=false`
3. 前端连接WebSocket `ws://localhost:18789`
4. 调用 `system.get_health` 验证连接
5. 逐接口替换Mock为真实接口
6. 联调通过后更新 `docs/api-mock.md` 中标记为“已联调”

**联调状态标记**：

| 接口 | Mock完成 | 后端完成 | 联调通过 |
|------|---------|---------|---------|
| GET /api/v1/system/health | ✅ | 待完成 | 待联调 |
| GET /api/v1/agents | ✅ | 待完成 | 待联调 |
| GET /api/v1/agents/:id | ✅ | 待完成 | 待联调 |
| WebSocket connect | ✅ | 待完成 | 待联调 |
| sessions.send | ✅ | 待完成 | 待联调 |



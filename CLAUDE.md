# 模拟公司 (SimuCorp) — 自进化多Agent协作系统

## 项目概述
模拟公司是一个基于OpenClaw Gateway的"公司制"多Agent协作系统。
它模拟企业组织架构，具备自动招聘、自进化、多分公司协作、大项目管理等能力。

**核心理念**: 公司即代码（Company-as-Code），让AI Agent像公司员工一样分工协作、自我生长。

**设计文档**: `docs/多Agent协作系统设计.md`（共18章完整设计）  
**白皮书**: `docs/模拟公司白皮书.md`（设计哲学与愿景）  
**Agent提示词**: `prompts/ea.md`（总裁助理）、`prompts/hr_manager.md`（人事部经理）、`prompts/gtf_manager.md`（机动部经理）

## 技术栈
- 引擎: OpenClaw Gateway (Node.js)
- 前端: React 18 + TypeScript + Vite + Tailwind CSS + Shadcn/ui
- 后端: Node.js (OpenClaw Gateway内置HTTP Server + WebSocket JSON-RPC)
- 数据库: PostgreSQL 16, Redis 7.2, ChromaDB
- 模型网关: One API / LiteLLM
- 项目管理: OpenProject 14
- 部署: Docker Compose

## 分阶段开发计划

### Phase 1: MVP (当前阶段)
目标：CEO能通过WebChat与EA对话，EA能路由任务给GTF执行，结果返回。

1. **环境搭建与Gateway配置**
   - 创建 docker-compose.yml（PostgreSQL + Redis + ChromaDB + One API + OpenProject + Gateway）
   - 创建 openclaw.json，注册三个初始Agent（EA, HR_Mgr, GTF_Mgr）
   - 创建 init-db.sql（部门表、模板表初始化）
   - 启动Gateway，验证WebSocket连接

2. **初始Agent的Soul（系统提示词）编写**
   - 创建 `prompts/ea.md`
   - 创建 `prompts/hr_manager.md`
   - 创建 `prompts/gtf_manager.md`

3. **基础通信联调**
   - 实现CEO → EA → GTF 的任务路由链路
   - EA能正确分析意图并选择目标Agent

4. **前端MVP页面**
   - 初始化 React + Vite + Tailwind + Shadcn/ui 项目
   - 实现 Dashboard 基础布局和统计卡片
   - 实现 WebChat 对话界面（WebSocket连接EA）

### Phase 2: 核心功能
1. 组织架构可视化 (React Flow 拓扑图)
2. 自动招聘闭环 (GTF复盘 → HR招聘 → 实习Agent → 转正评估)
3. 审批中心 (部门新建审批)
4. 项目管理集成 (OpenProject)

### Phase 3: 完整功能
1. 会议中心 (实时对话 + 纪要生成)
2. 分公司管理 (心跳同步 + 任务委派)
3. 全量监控与告警
4. SOUL自动优化
5. 国际化

## 开发规范
- **所有接口**遵循设计文档第10章的WebSocket RPC方法和REST API定义
- **数据模型**遵循设计文档第11章
- **错误码**使用设计文档第18章附录中的规范
- **前端组件结构**参照设计文档第9章的组件树
- **安全策略**遵循设计文档第12章的分级审批机制

## 设计文档速查
`多agent协作系统设计.md` 共18章：

| 章 | 内容 | 开发时重点查阅 |
|----|------|---------------|
| 1 | 系统综述与设计愿景 | 项目目标 |
| 2 | 系统总体架构 | 分层架构和通信协议 |
| 3 | 组织模型与核心Agent设计 | EA/HR/GTF的Soul和工具集 |
| 4 | 能力同步与任务路由 | EA的路由决策逻辑 |
| 5 | 自进化闭环 | 自动招聘和实习考核 |
| 6 | 基座抽象与统一记忆管理 | Runtime接口和记忆架构 |
| 7 | 大项目管理与会议机制 | OpenProject集成和会议流程 |
| 8 | 模型中转站与独立模型配置 | 模型配置和降级链 |
| 9 | 前端页面详细设计 | 页面布局、组件树、状态管理 |
| 10 | 后端接口详细设计 | WebSocket方法、REST API |
| 11 | 数据模型与存储 | 数据库表结构 |
| 12 | 安全与权限模型 | 审批分级、沙盒验证 |
| 13 | 监控日志与审计 | 指标、告警规则 |
| 14 | 错误处理与韧性设计 | 超时、离线、降级处理 |
| 15 | 扩展性设计 | 新工具、新Runtime接入 |
| 16 | 多语言与国际化 | i18n方案 |
| 17 | 部署与初始化 | docker-compose.yml |
| 18 | 附录 | 消息类型枚举、状态枚举、错误码 |

## 当前任务
Phase 1 — MVP 已完成核心开发。

## 快速启动

### 1. 启动基础设施 (可选，Gateway 可独立运行)
```bash
docker compose up -d postgres redis chromadb
```

### 2. 启动 Gateway
```bash
cd gateway && npm run dev
```
Gateway 在 `http://localhost:18789` 启动，WebSocket 在 `ws://localhost:18789`。

### 3. 启动前端
```bash
cd frontend && npm run dev
```
前端在 `http://localhost:5173` 启动，自动代理 API 请求到 Gateway。

### 4. 验证
```bash
curl http://localhost:18789/api/v1/system/health
```

## 项目结构
```
simu-corp/
├── docker-compose.yml       # PostgreSQL, Redis, ChromaDB, One API
├── init-db.sql              # 数据库初始化（7张表 + 8个模板）
├── openclaw.json            # Gateway 配置（3个 Agent）
├── .env.example             # 环境变量模板
├── prompts/                 # Agent Soul 提示词
│   ├── ea.md
│   ├── hr_manager.md
│   └── gtf_manager.md
├── gateway/                 # Node.js Gateway 服务器
│   └── src/
│       ├── index.ts         # 入口
│       ├── server/          # HTTP + WebSocket 服务器
│       ├── core/            # Agent注册、消息路由、会话管理
│       ├── agents/          # EA, GTF, HR 三个Agent实现
│       ├── transport/       # Agent通信传输层
│       ├── api/             # REST API 路由
│       ├── db/              # PostgreSQL 连接池
│       ├── redis/           # Redis 客户端
│       ├── llm/             # One API LLM 客户端
│       └── types/           # TypeScript 类型定义
└── frontend/                # React + Vite + Tailwind 前端
    └── src/
        ├── components/      # UI 组件
        │   ├── layout/      # TopBar, Sidebar, MainLayout
        │   ├── dashboard/   # StatsCards, AgentStatusList, ActivityStream
        │   ├── chat/        # ChatPanel, MessageBubble, MessageInput
        │   └── shared/      # StatusBadge, WsStatusIndicator
        ├── pages/           # DashboardPage
        ├── hooks/           # useWebSocket
        ├── store/           # Zustand 全局状态
        └── api/             # wsClient (WebSocket JSON-RPC)
```

## 已实现的 Phase 1 功能
- CEO 通过 WebChat / REST API 与 EA 对话
- EA 关键词意图分析（问候、状态查询、任务请求）
- EA 自动路由任务到 GTF Manager
- GTF 生成模拟响应（代码/分析/文案/通用）
- 3个 Agent 的能力目录与匹配评分
- WebSocket JSON-RPC + HTTP REST 双通道
- 前端 Dashboard：统计卡片、Agent状态列表、实时活动流、WebChat
- Mock 模式（默认），配置 ONE_API_KEY 后可切换真实 LLM

## 开发命令
| 命令 | 说明 |
|------|------|
| `cd gateway && npm run dev` | 启动 Gateway (热重载) |
| `cd gateway && npx tsc --noEmit` | Gateway 类型检查 |
| `cd frontend && npm run dev` | 启动前端 (热重载) |
| `cd frontend && npx tsc --noEmit` | 前端类型检查 |
| `docker compose up -d` | 启动所有基础设施 |
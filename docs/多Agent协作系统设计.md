# 自进化多Agent协作系统 · 完整设计规范 v6.0（最终交付版）

> **状态**：生产就绪  
> **引擎**：OpenClaw Gateway，支持 Claude Code、Codex CLI 等多基座 Runtime  
> **范围**：架构、组织、自进化、项目管理、基座抽象、权限、监控、前端、后端、部署  
> **使用方式**：可按章节分发给前端、后端、Agent 提示词工程师并行实施

---

## 目录
1. [系统综述与设计愿景](#1-系统综述与设计愿景)  
2. [系统总体架构](#2-系统总体架构)  
3. [组织模型与核心Agent设计](#3-组织模型与核心agent设计)  
4. [能力同步与任务路由](#4-能力同步与任务路由)  
5. [自进化闭环](#5-自进化闭环)  
6. [基座抽象与统一记忆管理](#6-基座抽象与统一记忆管理)  
7. [大项目管理与会议机制](#7-大项目管理与会议机制)  
8. [模型中转站与独立模型配置](#8-模型中转站与独立模型配置)  
9. [前端页面详细设计](#9-前端页面详细设计)  
10. [后端接口详细设计](#10-后端接口详细设计)  
11. [数据模型与存储](#11-数据模型与存储)  
12. [安全与权限模型](#12-安全与权限模型)  
13. [监控、日志与审计](#13-监控日志与审计)  
14. [错误处理与韧性设计](#14-错误处理与韧性设计)  
15. [扩展性设计](#15-扩展性设计)  
16. [多语言与国际化](#16-多语言与国际化)  
17. [部署与初始化](#17-部署与初始化)  
18. [附录](#18-附录)

---

## 1. 系统综述与设计愿景

### 1.1 项目目标
构建一个**模拟现代企业、可自我生长、跨设备协同**的AI Agent组织。它能够：
- 自动招聘、培养新Agent（实习→转正）
- 通过审批建立新部门
- 管理多分公司（内网其他机器）并同步能力
- 用成熟项目管理工具推进长周期项目
- 允许多Agent开会讨论，CEO实时干预
- 每个Agent可独立配置模型，并可部署到不同的基座（Claude Code、Codex CLI 等）

### 1.2 关键设计原则
- **组织即代码**：部门、角色、Agent映射为可配置资源
- **能力即标签**：Agent/分公司的计算能力标签驱动任务路由
- **记忆隔离**：私有记忆三重隔离（agent_id + session_id + user_id），共享需白名单
- **审批分级**：日常招聘自动执行；重大变更（建部、分公司接入）需CEO审批
- **协议统一**：WebSocket JSON-RPC + HTTP REST，所有Agent通过Gateway通信
- **基座解耦**：执行环境抽象为Runtime，可插拔切换，记忆统一管理
- **故障自愈**：任务超时重试、模型降级、Agent离线转移，CEO无感知

### 1.3 术语表
| 术语 | 定义 |
|------|------|
| **Agent** | 一个有角色、提示词、工具集和模型配置的AI实体 |
| **部门** | 由部门经理管理的一组Agent |
| **分公司** | 另一台机器上部署的完整Agent组织，与总部对称 |
| **基座 (Runtime)** | Agent执行推理与工具调用的环境，如OpenClaw原生、Claude Code CLI |
| **MCP** | Model Context Protocol，Agent与工具间的标准通信协议 |
| **A2A** | Agent-to-Agent Protocol，跨基座Agent间通信协议 |
| **Soul** | 系统提示词，定义Agent的角色、行为准则和工作流程 |
| **实习期** | 新Agent被创建后的考核阶段，基于任务成功率和质量决定转正 |

---

## 2. 系统总体架构

### 2.1 分层架构图
```
浏览器 (React SPA)
    │ WebSocket (JSON-RPC) + HTTP REST
    ▼
OpenClaw Gateway (总部)
    ├─ WebSocket Server + HTTP Server
    ├─ Session Manager
    ├─ Runtime Manager (基座调度)
    └─ Agent Router (路由与权限)
        │
消息总线 (Kafka / Redis Streams)
        │
  ┌─────┼─────────────┬──────────────┐
  │     │             │              │
EA   HR_Mgr  GTF_Mgr  ...部门经理   分公司Gateway...
  │     │             │              │
  └─────┴─────────────┴──────────────┘
        │
共享基础设施层
 ├─ 模型中转站 (One API / LiteLLM)
 ├─ 统一 Memory MCP Server (ChromaDB)
 ├─ MCP工具 (OpenProject, Git, Wiki 等)
 └─ 共享工作内存 (Redis)
```
分公司拥有对称的本地Gateway及最小组织，通过消息总线互通。总部和分公司间通过 `TASK_DELEGATE` 和 `BRANCH_HEARTBEAT` 协作。

### 2.2 通信协议
- **WebSocket**：OpenClaw原生JSON-RPC v3，用于实时消息、状态推送、会议对话
- **HTTP REST**：用于配置管理、审批操作、查询
- **认证**：统一使用Gateway Token (`Authorization: Bearer <token>`)

### 2.3 多分公司拓扑
- 每个分公司独立部署Gateway，配置相同的消息总线接入点
- 分公司启动时发送 `BRANCH_REGISTER` 注册到总部能力目录
- 总部EA负责跨分公司任务委派
- 分公司间不能直接通信，必须经总部EA中转

---

## 3. 组织模型与核心Agent设计

### 3.1 公司制组织架构
两层结构：**部门经理 → 基层Agent**。部门经理负责任务拆解、资源协调；基层Agent执行具体任务。

### 3.2 初始三核心Agent
#### 3.2.1 总裁助理 (EA)
**Agent ID**: `ea`  
**职责**: 任务路由、进度汇总、会议主持、审批队列管理  
**Soul**: 见附件 `agent_prompts/ea.md`  
**建议基座**: OpenClaw 原生 (通用模型 GPT-5)

#### 3.2.2 人事部经理 (HR Manager)
**Agent ID**: `hr_manager`  
**职责**: 全流程招聘（需求分析→JD生成→实习考核→转正/淘汰）、提示词优化、基座选择、模板库维护  
**Soul**: 见附件 `agent_prompts/hr_manager.md`  
**建议基座**: OpenClaw 原生 (GPT-5 或 Claude Opus)

#### 3.2.3 机动部经理 (GTF Manager)
**Agent ID**: `gtf_manager`  
**职责**: 万能执行者、复盘沉淀、触发招聘需求  
**Soul**: 见附件 `agent_prompts/gtf_manager.md`  
**建议基座**: Claude Code CLI (强代码执行) + 统一 Memory MCP

### 3.3 部门扩展
可申请创建的部门类型（部分示例）：研发部、数据分析部、安全部、运维部、设计部。申请需CEO审批，HR创建部门经理并招募初始成员。

### 3.4 Agent生命周期
```
创建 (实习) ──实习期 (任务数/天数)──→ 评估 ──→ 转正 (正式)
                                    └──→ 延长实习
                                    └──→ 淘汰 (归档记忆)
```
转正后支持持续升级：SOUL更新（沙盒+A/B测试）、工具增减、基座迁移。

---

## 4. 能力同步与任务路由

### 4.1 能力标签定义
Agent或分公司注册时携带能力标签 JSON：
```json
{
  "capabilities": {
    "hardware": ["GPU-A100"],
    "software": ["docker", "cuda12"],
    "specialty": ["model_training"]
  },
  "load": { "cpu_percent": 45, "active_tasks": 2 }
}
```

### 4.2 分公司心跳与能力目录
- 分公司每5分钟发送 `BRANCH_HEARTBEAT`，更新负载和能力
- 总部EA维护**能力目录**，3个周期丢失标记离线
- 注册时携带完整能力，心跳可增量更新

### 4.3 任务路由决策
1. EA解析任务需求→提取能力标签
2. 查询能力目录：优先匹配专业部门Agent
3. 若无，查询分公司能力
4. 仍无，分派给总部机动部

---

## 5. 自进化闭环

### 5.1 自动招聘
1. **需求触发**：GTF/PM复盘后发 `TALENT_REQUEST` 给HR
2. **HR分析**：搜索真实JD，匹配模板，生成职位定义（Soul、工具集、基座建议）
3. **实例化**：创建实习Agent，挂靠用人部门，设定实习期KPI
4. **实习考核**：自动跟踪任务成功率、质量、协作等
5. **转正/淘汰**：期满自动评分，≥80转正，50-79延长，<50淘汰

### 5.2 部门新建审批
1. 部门经理提交 `PETITION` 给EA
2. EA生成审批卡片推送给CEO
3. CEO批准后HR创建部门经理，招募初始成员

### 5.3 自我学习
- **长期记忆**：私有向量库 (ChromaDB)，按Agent隔离
- **SOUL更新**：HR定期优化提示词，经过沙盒评估和A/B测试后上线
- **工具扩展**：Agent可申请新MCP工具，HR审核后挂载

---

## 6. 基座抽象与统一记忆管理

### 6.1 Runtime抽象层
定义统一接口 `IRuntime`，所有执行环境（OpenClaw、Claude Code、Codex CLI）必须实现：
```typescript
interface IRuntime {
  executeTask(sessionId: string, task: TaskPayload): Promise<TaskResult>;
  getStatus(): Promise<AgentStatus>;
  memoryStore(key: string, data: any, namespace: string): Promise<void>;
  memoryRetrieve(query: MemoryQuery, namespace: string): Promise<MemoryResult[]>;
  toolCall(toolName: string, params: any): Promise<any>;
}
```
Gateway通过 `Runtime Manager` 维护 `agentId → runtimeType + endpoint` 映射。

### 6.2 统一记忆架构
所有Agent的记忆操作统一通过 **Memory MCP Server**（ChromaDB后端），按 `namespace={agent_id}` 隔离。跨基座记忆连续性由以下机制保证：
- 会话开始：Runtime自动检索历史记忆注入上下文
- 会话结束：摘要和重要知识写回Memory Server
- 对于Claude Code等外部基座，通过初始化钩子注入记忆，结束钩子回写

### 6.3 基座选择指南
| 岗位类型 | 推荐Runtime | 原因 |
|----------|-------------|------|
| 代码密集 | Claude Code CLI | 代码编辑、文件操作能力强 |
| 数据分析 | Codex CLI 或 GPT-5 | 数据库集成好，效率高 |
| 通用协调 | OpenClaw 原生 | 无重型执行需求 |
| 创意设计 | Claude Opus on OpenClaw | 创意生成优秀 |

---

## 7. 大项目管理与会议机制

### 7.1 集成工具
- **项目管理**：OpenProject（开源替代Jira），通过REST API集成
- **代码仓库**：GitLab/GitHub/Gitea，通过MCP Server对接
- **文档**：Wiki.js/Outline，通过MCP对接

### 7.2 新建项目流程
1. CEO在前端填写表单（名称、关联仓库、Wiki、项目经理指派）
2. Gateway调用OpenProject API创建项目
3. 若选择自动创建PM Agent，则实例化并注入环境变量和工具权限
4. PM Agent初始化WBS和Wiki首页

### 7.3 会议机制
- **发起**：Agent发送 `MEETING_REQUEST` → EA生成审批 → CEO批准
- **进行**：EA主持，按议题轮序发言，CEO可实时插话
- **结束**：CEO确认，EA生成结构化纪要，分发Action Items，同步到OpenProject
- **会议纪要格式**：包含决议、待办、负责人、截止时间

---

## 8. 模型中转站与独立模型配置

### 8.1 模型网关
推荐 **One API** 或 **LiteLLM**，提供统一OpenAI兼容接口，支持多厂商、负载均衡、降级链和成本追踪。

### 8.2 Agent独立模型配置
- HR在创建Agent时根据岗位分析推荐模型和降级链
- CEO可在Agent详情页手动切换模型
- 降级策略：按 `fallbacks` 列表顺序自动切换，记录降级事件

### 8.3 成本控制
- 每个Agent可设置月度调用上限
- 全局预算告警：用量达80%通知，100%限制调用
- Dashboard展示各Agent/项目成本统计

---

## 9. 前端页面详细设计

### 9.1 技术栈
React 18 + TypeScript + Vite + Tailwind CSS + Shadcn/ui + Zustand + TanStack Query + React Flow + ECharts + React Router v6 + i18next

### 9.2 路由与页面
| 路由 | 页面 | 说明 |
|------|------|------|
| `/dashboard` | CEO控制台 | 统计卡片、实时活动流、分公司状态、快捷入口 |
| `/org` | 组织架构 | 拓扑图（React Flow）、列表视图、新建Agent |
| `/org/:agentId` | Agent详情 | 基本信息、模型配置、记忆、提示词、工具、任务历史 |
| `/branches` | 分公司管理 | 列表、能力详情、注册审批 |
| `/projects` | 项目管理 | 项目列表、新建项目对话框 |
| `/projects/:projectId` | 项目详情 | 任务看板、会议记录、设置 |
| `/meetings` | 会议中心 | 待审批/进行中/已完成 |
| `/meetings/:meetingId` | 会议详情 | 实时对话区、参会人、纪要 |
| `/approvals` | 审批中心 | 卡片式列表，支持批准/驳回 |
| `/settings` | 系统设置 | 模型中转站、模板库、系统参数 |

### 9.3 关键交互
- **实时活动流**：WebSocket推送 `agent.task_*`、`meeting.*` 事件
- **模型切换**：Agent详情页下拉选择，即时生效
- **会议对话**：EA主持，广播消息，结束生成纪要
- **审批操作**：一键批准/驳回，自动通知EA执行

### 9.4 组件树与状态管理
- 共享组件：`AgentAvatar`, `StatusBadge`, `ModelSelector`, `ApprovalCard`
- Zustand store：`wsStatus`, `agents`, `meetings`, `approvals`, `projects`, `modelProviders`
- 自定义Hooks：`useWebSocket`, `useAgents`, `useMeetings`, `useApprovals`

---

## 10. 后端接口详细设计

### 10.1 WebSocket接口 (JSON-RPC)
**连接**: `ws://gateway:18789`，携带Token  
**核心方法**:
- `sessions.send` – 向Agent发送消息
- `agents.list/get/update` – Agent管理
- `meetings.create/send_message/end` – 会议控制
- `system.get_health` – 健康检查
- `config.models.list/update` – 模型配置

**推送事件**:
`agent.status_changed`, `agent.task_started/progress/completed`, `meeting.message_received`, `meeting.state_changed`, `approval.created`, `branch.status_changed`, `system.alert`

### 10.2 HTTP REST API (`/api/v1`)
**Agent管理**: `GET/PATCH /agents`, `PATCH /agents/:id/model`  
**模型配置**: `GET/POST/PATCH/DELETE /models`  
**组织与审批**: `GET /org/departments`, `POST /approvals/:id/approve`  
**项目管理**: `GET/POST /projects`, `GET/POST/PATCH /projects/:id/tasks`  
**分公司**: `GET /branches`, `POST /branches/:id/delegate`  
**会议**: `GET /meetings`, `GET /meetings/:id/minutes`  
**模板**: `GET /templates`, `POST /templates/:id/instantiate`  
**系统**: `GET /system/health`, `GET /system/metrics`, `GET /system/audit-log`

### 10.3 错误码规范
0成功，1001参数错误，1002不存在，1003权限不足，2001模型失败，3001分公司不可达，4001工具异常，5000内部错误

---

## 11. 数据模型与存储

### 11.1 核心实体
- **Agent**: agent_id, name, role, department_id, model, fallbacks, tools[], skills[], status, employment_status
- **Department**: id, name, manager_agent_id
- **Branch**: id, hostname, capabilities, load, status, last_heartbeat
- **Project**: id, name, op_project_id, pm_agent_id, git_repos[], wiki_url
- **Task**: id, project_id, assignee, status, payload
- **Meeting**: id, title, status, host, participants[], messages[]
- **Approval**: id, type, petitioner, content, status

### 11.2 记忆存储
| 类型 | 后端 | 分区键 | TTL |
|------|------|--------|-----|
| 会话 | Redis Streams | `{agent_id}:{session_id}` | 24h |
| 工作 | Redis | `{task_id}` | 1h |
| 长期 | ChromaDB | `namespace={agent_id}` | 永久 |
| 共享 | Redis + 白名单 | `{project_id}` | 任务结束 |

---

## 12. 安全与权限模型

### 12.1 审批分级 (L0-L3)
- **L0 自动执行**：日常查询、低风险工具调用
- **L1 EA审批**：容器内配置修改、个人工具申请
- **L2 经理审批**：影响部门或跨部门操作
- **L3 CEO审批**：新建部门、分公司接入、核心配置变更

### 12.2 研发沙盒验证
高风险操作流程：Agent创建Docker沙盒 → 自由验证 → 生成报告 → EA/经理审批 → 必要时升级CEO → 应用到生产环境

### 12.3 EA判断升级逻辑
影响多部门、生产核心、验证失败、首次高风险操作 → 升级L3；否则EA自行或转经理审批。

### 12.4 通信权限
- 同部门自由通信
- 跨部门需经理批准或EA中转
- 跨分公司必须经总部EA

### 12.5 工具调用风险矩阵
低风险（只读）→ 直接调用；中风险 → 沙盒+EA审批；高风险 → 沙盒+经理审批；极高 → CEO审批

---

## 13. 监控、日志与审计

### 13.1 监控指标
- **Agent**：在线数、任务成功率、响应时间
- **模型**：调用量、成功率、延迟、降级次数、成本
- **分公司**：心跳状态、负载、活跃任务
- **基础设施**：消息总线吞吐量、工具可用性

### 13.2 审计日志
记录所有关键事件：消息、任务、审批、Agent变更、模型调用、工具调用、配置变更、沙盒操作。热存储7天，温存储90天，冷归档。

### 13.3 告警机制
分级：Info（Dashboard）、Warn（通知+邮件）、Critical（WebChat推送+声音）。支持去重和收敛。当Agent离线、任务连续失败、模型错误率高、分公司失联、额度超限等触发。

---

## 14. 错误处理与韧性设计

### 14.1 任务超时
Agent超时 → EA询问 → 无响应则重新分配（最多3次） → 全部失败标记FAILED并通知CEO。

### 14.2 Agent离线
心跳丢失 → 从能力目录移除 → 未完成任务转给同级Agent → 恢复后重新加入。

### 14.3 模型降级
调用失败 → 按fallback链切换 → 全部失败则暂停关键任务或跳过非关键步骤，降级事件记录。

### 14.4 分公司不可达
心跳丢失 → 标记离线 → 高时效任务转移，低时效等待 → 恢复后重分派。

### 14.5 消息总线故障
临时降级到本地内存队列，超30秒暂停跨Agent消息，保存到文件，恢复后回放。

### 14.6 外部工具不可用
重试3次 → 通知PM暂存本地 → 恢复后同步。

### 14.7 PM故障转移
EA检测PM离线 → 临时接管，从快照恢复状态 → PM恢复后交还。

### 14.8 系统重启
广播关机 → 各Agent保存状态 → 重启后恢复。

---

## 15. 扩展性设计

### 15.1 新工具接入
开发MCP Server → 注册到Gateway → 自动出现在工具列表，HR可按需分配。

### 15.2 新Agent模板
HR创建或联网生成，草稿→测试→发布，持续跟踪成功率。

### 15.3 新部门类型
定义部门模板（经理Soul、成员模板、工具集） → CEO审批后实例化。

### 15.4 新Runtime接入
实现IRuntime接口 → 注册到Runtime Manager → HR可在招聘时选择。

### 15.5 扩展监控
所有扩展自动纳入监控和审计，低使用率或高失败率会被标记。

---

## 16. 多语言与国际化

### 16.1 前端
使用i18next，中英双语（可扩展），语言文件按页面拆分，CEO在TopBar切换。

### 16.2 Agent工作语言
- 系统提示词：英文
- Agent间通信：英文
- 与CEO交互：跟随CEO语言

### 16.3 项目文档
新建项目时指定输出语言，会议纪要遵循该设置。

---

## 17. 部署与初始化

### 17.1 总部最小部署
1. 部署OpenClaw Gateway
2. 配置三个核心Agent (EA, HR, GTF)
3. 启动模型中转站 (One API / LiteLLM)
4. 部署OpenProject并配置API Token
5. 启动前端SPA
6. 预置Agent模板库 (至少8个)

### 17.2 分公司接入
1. 新机器部署Gateway，初始化三个Agent
2. 配置连接总部消息总线
3. 发送 `BRANCH_REGISTER`，加入能力目录
4. 启动心跳

### 17.3 分阶段实施
- **Phase 1**: 基础通信、Dashboard、EA对话、GTF执行
- **Phase 2**: 组织可视化、模型独立配置、审批流、项目管理集成、自动招聘
- **Phase 3**: 会议中心、分公司管理、自升级沙盒、全量监控

---

## 18. 附录

### 18.1 参考资料
- OpenClaw 官方文档: https://openclaw.ai
- MCP 规范: https://modelcontextprotocol.io
- A2A 协议: https://a2aprotocol.org
- OpenProject API: https://www.openproject.org/docs/api/
- One API: https://github.com/songquanpeng/one-api
- LiteLLM: https://github.com/BerriAI/litellm
- 参考设计: SilentLake 二开架构、OneManCompany 论文

### 18.2 核心Agent提示词附件
(参见独立文件 `agent_prompts/ea.md`, `hr_manager.md`, `gtf_manager.md`)

---
好的，我们从第一章开始，逐章输出详细内容。

---

# 第一章：系统综述与设计愿景

## 1.1 项目背景

在AI技术快速演进的今天，单一Agent的能力已不足以应对复杂多变的实际工作场景。本系统旨在构建一个**模拟现代企业运作的AI Agent组织**，让多个Agent像公司员工一样分工协作、自我生长、持续进化。

## 1.2 项目目标

本系统的核心目标是构建一个具备以下能力的Agent组织：

1. **自动招聘与培养**：系统能够识别技能缺口，自动生成岗位描述，创建实习Agent，并通过真实任务考核后转正。
2. **部门扩展与审批**：当某个方向的Agent数量增多时，可申请组建专业部门，由部门经理统一管理。
3. **多分公司协同**：支持内网其他机器作为“分公司”接入，各自拥有完整组织架构，可通过总部委派任务实现跨节点协作。
4. **长周期项目管理**：集成项目管理工具（OpenProject），支持大项目拆解、任务分配、进度跟踪、多轮会议讨论。
5. **模型灵活配置**：每个Agent可独立选择底层LLM，并可部署到不同执行环境（OpenClaw原生、Claude Code CLI、Codex CLI等）。
6. **用户无感知的基座切换**：CEO不需要了解Agent运行在哪个基座上，一切由系统自动调度。

## 1.3 设计理念

### 1.3.1 公司即代码（Company-as-Code）

本系统的核心设计理念是将“公司”这一组织形态抽象为一套可编程、可管理的架构。组织中的每一个角色（总裁助理、人事经理、部门经理、专业Agent）都是可定义、可创建、可优化的实体。

### 1.3.2 自进化闭环

系统不是静态的——它在运行过程中会自动：
- 发现技能缺口
- 联网调研岗位需求
- 生成招聘标准
- 创建并考核新Agent
- 优化现有Agent的提示词和工具集

这个闭环使得系统能够像生物体一样自我进化，适应不断变化的任务需求。

### 1.3.3 分层审批与自动决策

并非所有操作都需要CEO介入。系统将操作分为四个层级：
- **L0**：日常操作，自动执行
- **L1**：影响单个Agent的操作，由总裁助理（EA）审批
- **L2**：影响整个部门的操作，由部门经理审批
- **L3**：影响全局的重大决策，需CEO审批

这保证了系统既高效运转，又不会失去控制。

### 1.3.4 协议统一与标准兼容

系统完全基于开放标准构建：
- **MCP**（Model Context Protocol）：Agent与工具间的通信标准
- **A2A**（Agent-to-Agent Protocol）：跨基座Agent间的通信标准
- **JSON-RPC over WebSocket**：前端与Gateway的实时通信

## 1.4 核心设计原则

| 原则 | 说明 |
|------|------|
| **组织即代码** | 部门、角色、Agent映射为可配置、可版本管理的资源 |
| **能力即标签** | Agent/分公司的能力用结构化标签描述，驱动任务路由 |
| **记忆隔离** | 每个Agent的私有记忆严格隔离（agent_id + session_id + user_id），共享记忆需白名单授权 |
| **审批分级** | 日常招聘自动执行；建部、分公司接入等重大变更需CEO审批 |
| **协议统一** | 所有通信走WebSocket JSON-RPC或HTTP REST，不引入私有协议 |
| **基座解耦** | 执行环境抽象为Runtime接口，可插拔切换，记忆统一管理 |
| **故障自愈** | 任务超时自动重试、模型降级、Agent离线任务转移，CEO尽量无感知 |

## 1.5 系统边界

### 1.5.1 本系统负责
- Agent的组织管理（创建、调度、考核、升级）
- 任务的分析、路由、执行、跟踪
- 项目管理与会议协调
- 模型选择与成本控制
- 分公司能力同步与任务委派
- 安全审批与审计

### 1.5.2 本系统不负责
- LLM的训练或微调（但通过模型中转站调用已有模型）
- 物理硬件的管理（但通过分公司心跳感知硬件状态）
- 外部系统的实现（如OpenProject本身的功能，本系统仅调用其API）

## 1.6 术语表

| 术语 | 英文 | 定义 |
|------|------|------|
| Agent | Agent | 一个有角色、系统提示词、工具集和模型配置的AI实体，模拟一名“员工” |
| 部门 | Department | 由部门经理管理的一组Agent，对应一个专业方向 |
| 分公司 | Branch | 部署在另一台机器上的完整Agent组织，与总部结构对称 |
| 基座 | Runtime | Agent执行推理与工具调用的底层环境，如OpenClaw原生、Claude Code CLI |
| 总裁助理 | Executive Assistant (EA) | 唯一用户入口，负责任务分析、路由、进度汇总、会议主持 |
| 人事部经理 | HR Manager | 负责Agent全生命周期管理：招聘、考核、转正、升级、淘汰 |
| 机动部经理 | GTF Manager | 万能执行者，承接所有无专业Agent的任务，并沉淀经验触发招聘 |
| Soul | Soul | Agent的系统提示词，定义其角色、行为准则、工作流程和协作风格 |
| 实习期 | Internship | 新Agent被创建后的考核阶段，基于任务数量、成功率、质量评分决定是否转正 |
| MCP | Model Context Protocol | Agent与工具间的标准通信协议，由Anthropic发布 |
| A2A | Agent-to-Agent Protocol | 跨基座、跨框架Agent间通信的开放协议，由Google发布 |
| 能力标签 | Capability Tag | 描述Agent或分公司能力的结构化数据，如技能、硬件、软件环境 |

## 1.7 与其他方案的对比定位

| 维度 | 本系统 | MetaGPT/ChatDev | CrewAI | AutoGen |
|------|--------|-----------------|--------|---------|
| 组织形态 | 长期运行的公司 | 一次性项目组 | 按任务编组 | Hub-and-Spoke网络 |
| Agent生命周期 | 完整的招聘-实习-转正-升级 | 任务开始创建，结束销毁 | 任务触发创建 | 按需实例化 |
| 自我进化 | 自动招聘、SOUL优化、部门扩展 | 无 | 无 | 无 |
| 跨节点协作 | 分公司机制 | 无 | 无 | 无 |
| 项目管理 | 深度集成OpenProject | 无 | 无 | 无 |
| 多模型支持 | 模型中转站+独立配置 | 单一模型 | 可配置 | 可配置 |
| CEO参与度 | 审批分级，重大决策才介入 | 设定目标后自动运行 | 设定流程后运行 | 交互式或自动 |

## 1.8 技术选型概览

| 层级 | 技术选择 | 选型理由 |
|------|---------|---------|
| Agent引擎 | OpenClaw Gateway | 原生多Agent支持、Gateway架构、Session管理、MCP集成 |
| 前端 | React 18 + TypeScript + Vite | 生态成熟、类型安全、构建速度快 |
| UI框架 | Tailwind CSS + Shadcn/ui | 原子化CSS + 高质量可定制组件 |
| 状态管理 | Zustand + TanStack Query | 轻量、与WebSocket事件契合、服务端状态缓存 |
| 实时通信 | WebSocket (JSON-RPC) | OpenClaw原生协议，双向实时 |
| 消息总线 | Redis Streams / Kafka | 持久化消息、支持多消费者 |
| 模型中转站 | One API / LiteLLM | 多厂商统一接口、成本追踪、降级链 |
| 项目管理 | OpenProject | 最强大的开源替代Jira方案 |
| 向量数据库 | ChromaDB | 轻量、支持语义检索、可自托管 |
| 容器化 | Docker | 沙盒环境、分公司部署、环境一致性 |

---

以上是第一章的详细内容。这一章为整个系统定了基调——它不是工具链，而是一个活的、会自我进化的AI组织。接下来的每一章都是对这个愿景的具体落地。


# 第二章：系统总体架构

## 2.1 架构设计原则

系统架构遵循以下原则：

1. **分层解耦**：前端、Gateway、Agent 集群、基础设施层各自独立，通过标准协议通信，每层可独立升级替换。
2. **Gateway 作为唯一控制面**：所有外部请求和消息都经过 Gateway 进行路由、鉴权、日志记录。
3. **Agent 不直接暴露**：CEO 不与 Agent 直接通信，全部通过 EA 中转，EA 是组织对外的唯一接口。
4. **消息总线解耦**：Agent 之间通过消息总线异步通信，不直接调用，便于监控、重放、审计。
5. **统一记忆与工具**：所有 Agent 的记忆和工具调用走统一的 MCP Server，保证跨基座一致性。

## 2.2 完整分层架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            用户界面层 (Presentation Layer)                     │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                    React SPA (Vite + TypeScript)                        │   │
│  │  ┌──────────┬──────────┬──────────┬──────────┬──────────┬──────────┐  │   │
│  │  │Dashboard │ 组织架构  │ Agent详情│ 项目管理  │ 会议中心  │ 审批中心  │  │   │
│  │  │          │(拓扑+列表)│(模型配置)│(看板+纪要)│(实时对话)│(卡片审批)│  │   │
│  │  └──────────┴──────────┴──────────┴──────────┴──────────┴──────────┘  │   │
│  │                                                                        │   │
│  │  状态管理: Zustand (WebSocket实时更新) + TanStack Query (HTTP缓存)       │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  通信方式:                                                                   │
│  • WebSocket (JSON-RPC v3) ──── 实时双向：消息发送、状态推送、会议对话        │
│  • HTTP REST ──── 辅助：配置管理、审批操作、历史查询                           │
└──────────────────┬──────────────────────────────────────────────────────────┘
                   │
                   │ 认证: Authorization: Bearer <gateway_token>
                   │
┌──────────────────▼──────────────────────────────────────────────────────────┐
│                           Gateway 层 (Control Plane)                          │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                      OpenClaw Gateway (Node.js)                        │   │
│  │                                                                        │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌───────────┐ │   │
│  │  │ WebSocket    │  │ HTTP Server  │  │ Session      │  │ Auth      │ │   │
│  │  │ Server       │  │ (REST API)   │  │ Manager      │  │ Middleware│ │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  └───────────┘ │   │
│  │                                                                        │   │
│  │  ┌──────────────────────────────────────────────────────────────────┐ │   │
│  │  │                        核心路由模块                                │ │   │
│  │  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │ │   │
│  │  │  │ Agent Router │  │ Runtime Mgr  │  │ Binding Manager      │   │ │   │
│  │  │  │ (消息路由)    │  │ (基座调度)    │  │ (Channel→Agent映射)  │   │ │   │
│  │  │  └──────────────┘  └──────────────┘  └──────────────────────┘   │ │   │
│  │  └──────────────────────────────────────────────────────────────────┘ │   │
│  │                                                                        │   │
│  │  ┌──────────────────────────────────────────────────────────────────┐ │   │
│  │  │                      监控与审计 (内置)                              │ │   │
│  │  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │ │   │
│  │  │  │ Metrics      │  │ Audit Logger │  │ Alert Manager        │   │ │   │
│  │  │  │ (Prometheus) │  │ (事件记录)    │  │ (告警分级与推送)      │   │ │   │
│  │  │  └──────────────┘  └──────────────┘  └──────────────────────┘   │ │   │
│  │  └──────────────────────────────────────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└──────────────────┬──────────────────────────────────────────────────────────┘
                   │
                   │ 消息总线 (Redis Streams / Kafka)
                   │
┌──────────────────▼──────────────────────────────────────────────────────────┐
│                           Agent 集群层 (Agent Cluster)                        │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                        组织架构 Agent                                   │   │
│  │                                                                        │   │
│  │  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐        │   │
│  │  │ EA       │    │ HR Mgr   │    │ GTF Mgr  │    │ PM Agent │        │   │
│  │  │ 总裁助理  │    │ 人事经理  │    │ 机动部    │    │ 项目经理  │        │   │
│  │  │          │    │          │    │          │    │          │        │   │
│  │  │ Runtime: │    │ Runtime: │    │ Runtime: │    │ Runtime: │        │   │
│  │  │ OpenClaw │    │ OpenClaw │    │ Claude   │    │ OpenClaw │        │   │
│  │  │ 原生     │    │ 原生     │    │ Code CLI │    │ 原生     │        │   │
│  │  └──────────┘    └──────────┘    └──────────┘    └──────────┘        │   │
│  │                                                                        │   │
│  │  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐        │   │
│  │  │ RD Mgr   │    │ DA Mgr   │    │ Sec Mgr  │    │ ...      │        │   │
│  │  │ 研发部   │    │ 数据分析  │    │ 安全部    │    │ (扩展)   │        │   │
│  │  └──────────┘    └──────────┘    └──────────┘    └──────────┘        │   │
│  │                                                                        │   │
│  │  每个Agent拥有: 私有记忆空间 | 独立模型配置 | 工具集 | Session管理       │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└──────────────────┬──────────────────────────────────────────────────────────┘
                   │
                   │ MCP协议 / A2A协议 / HTTP
                   │
┌──────────────────▼──────────────────────────────────────────────────────────┐
│                       共享基础设施层 (Shared Infrastructure)                   │
│                                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │ 模型中转站    │  │ 统一记忆服务  │  │ MCP工具总线   │  │ 项目管理工具  │   │
│  │              │  │              │  │              │  │              │   │
│  │ One API /    │  │ Memory MCP   │  │ OpenProject  │  │ OpenProject  │   │
│  │ LiteLLM      │  │ Server       │  │ MCP          │  │ Server       │   │
│  │              │  │ (ChromaDB)   │  │ GitLab MCP   │  │              │   │
│  │ • 多厂商统一  │  │              │  │ Wiki MCP     │  │ • 项目CRUD   │   │
│  │ • 成本追踪    │  │ • 长期记忆    │  │ Docker MCP   │  │ • 工作包管理  │   │
│  │ • 降级链     │  │ • 语义检索    │  │ ...          │  │ • 甘特图     │   │
│  │ • 负载均衡    │  │ • 按Agent隔离 │  │              │  │              │   │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘   │
│                                                                             │
│  ┌──────────────┐  ┌──────────────┐                                        │
│  │ 共享工作内存  │  │ 审计日志存储  │                                        │
│  │ (Redis)      │  │ (Elasticsearch│                                        │
│  │              │  │  / Loki)      │                                        │
│  └──────────────┘  └──────────────┘                                        │
└─────────────────────────────────────────────────────────────────────────────┘

                            ┌─────────────────────────┐
                            │    分公司节点 (可选)      │
                            │                         │
                            │  完整对称Gateway + Agent  │
                            │  通过消息总线与总部连接    │
                            │  BRANCH_HEARTBEAT同步    │
                            └─────────────────────────┘
```

## 2.3 各层职责详解

### 2.3.1 用户界面层 (Presentation Layer)

- **技术栈**：React 18 + TypeScript + Vite + Tailwind CSS + Shadcn/ui
- **核心功能**：提供 CEO 控制台、组织架构拓扑图、Agent 详情管理、项目看板、会议实时对话、审批卡片等
- **通信**：通过 WebSocket 与 Gateway 建立长连接，接收实时推送；通过 HTTP REST 进行配置操作
- **状态管理**：Zustand 管理全局实时状态，TanStack Query 管理服务端缓存

### 2.3.2 Gateway 层 (Control Plane)

这是系统的“大脑中枢”，核心组件包括：

**WebSocket Server**：
- 基于 OpenClaw 原生 JSON-RPC v3 协议
- 支持方法调用（前端→Gateway）和事件推送（Gateway→前端）
- 管理所有客户端的连接状态

**HTTP Server**：
- 提供 RESTful API（前缀 `/api/v1`）
- 用于配置管理、审批操作、历史查询等非实时操作
- 统一响应格式 `{ code, message, data }`

**Session Manager**：
- 管理每个 Agent 的会话生命周期
- 支持会话创建、持久化、恢复
- 按 `agent_id + session_id` 隔离

**Agent Router**：
- 根据消息目标地址将消息路由到对应 Agent
- 支持点对点（指定 agent_id）、广播（同部门）、角色路由（发给人资部所有 Agent）

**Runtime Manager**：
- 维护 `agent_id → runtime_type + endpoint` 映射
- 将任务消息转发到正确的执行环境
- 管理 Runtime 生命周期（启动、健康检查、销毁）

**Binding Manager**：
- 配置 Channel（WebChat、CLI 等）到 Agent 的映射规则
- 决定 CEO 的消息发给哪个 Agent（默认发给 EA）

**监控与审计**：
- 内置指标采集（Prometheus 格式）
- 审计日志记录（所有关键事件）
- 告警分级与推送

### 2.3.3 Agent 集群层 (Agent Cluster)

**组织架构 Agent**：
- EA（总裁助理）、HR Manager（人事经理）、GTF Manager（机动部经理）为系统初始三核心
- 部门经理（研发、数据分析、安全等）按需创建
- PM Agent 为每个项目自动创建或指派

**Agent 属性**：
- 每个 Agent 有独立的系统提示词（Soul）、模型配置、工具集
- 记忆严格隔离（私有记忆不可被其他 Agent 直接访问）
- 可部署到不同的 Runtime（OpenClaw 原生 / Claude Code / Codex）

**Agent 间通信**：
- 同部门 Agent 可自由通过消息总线通信
- 跨部门通信需经理批准或 EA 中转
- 跨分公司通信需经总部 EA

### 2.3.4 共享基础设施层 (Shared Infrastructure)

**模型中转站**：
- 基于 One API 或 LiteLLM
- 统一管理多家 LLM 提供商（OpenAI、Anthropic、DeepSeek 等）
- 提供 OpenAI 兼容接口，对 Agent 透明
- 支持降级链、负载均衡、成本追踪

**统一记忆服务**：
- Memory MCP Server（ChromaDB 后端）
- 按 `namespace={agent_id}` 隔离记忆
- 支持语义检索和混合搜索
- 跨基座记忆一致性保障

**MCP 工具总线**：
- 所有外部工具封装为 MCP Server
- 统一注册到 Gateway 的 `mcp.json`
- 按风险分级管控调用权限

**项目管理工具**：
- OpenProject（开源替代 Jira）
- 通过 REST API 集成
- PM Agent 自动创建项目、分配任务、更新状态

**共享工作内存**：
- Redis，按 `{task_id}` 或 `{project_id}` 分区
- 用于 Agent 间传递中间结果
- 白名单控制访问权限

**审计日志存储**：
- 热存储：Redis（7天）
- 温存储：Elasticsearch / Loki（90天）
- 冷归档：对象存储

## 2.4 通信协议详解

### 2.4.1 WebSocket JSON-RPC（主通道）

**连接建立**：
```
WebSocket Connect to: ws://{gateway-host}:18789
```

**Connect Frame（认证消息）**：
```json
{
  "type": "connect",
  "auth": { "token": "gw-xxxx" },
  "clientInfo": { "type": "web-ui", "version": "1.0.0" }
}
```

**请求-响应模式（前端→Gateway）**：
```json
→ {
    "id": "req-001",
    "method": "agents.list",
    "params": { "status": "online" }
  }

← {
    "id": "req-001",
    "result": {
      "agents": [...]
    }
  }
```

**事件推送模式（Gateway→前端）**：
```json
← {
    "type": "event",
    "event": "agent.task_completed",
    "data": {
      "agent_id": "gtf_manager",
      "task_id": "task-001",
      "result": { ... }
    }
  }
```

### 2.4.2 HTTP REST（辅助通道）

- 前缀：`/api/v1`
- 认证：`Authorization: Bearer <token>`
- 标准响应：`{ "code": 0, "message": "success", "data": { ... } }`
- 用于：配置 CRUD、审批操作、历史查询、文件上传

### 2.4.3 Agent 间通信

- 通过消息总线（Redis Streams / Kafka）异步通信
- 消息格式统一：
```json
{
  "message_id": "uuid",
  "sender": "agent_id",
  "receiver": "agent_id | broadcast | role_name",
  "type": "TASK_ASSIGN | QUERY | ...",
  "payload": { ... },
  "context": { "task_id": "...", "reply_to": "..." }
}
```

### 2.4.4 跨分公司通信

- 通过消息总线的跨节点路由
- 总部 EA 维护分公司地址表
- 消息根据 `receiver` 中的 branch 前缀投递

## 2.5 数据流向

### 2.5.1 CEO 下达任务
```
CEO → WebChat → Gateway → EA → 能力匹配 → 目标Agent → 执行 → 结果返回 → EA汇总 → CEO
```

### 2.5.2 自动招聘流
```
GTF/PM复盘 → TALENT_REQUEST → HR → 联网调研 → 创建实习Agent → 分配任务 → 绩效跟踪 → 转正/淘汰
```

### 2.5.3 审批流
```
申请人 → PETITION → EA → 审批队列 → WebChat推送 → CEO审批 → EA执行 → 通知申请人
```

### 2.5.4 会议流
```
PM → MEETING_REQUEST → EA → 审批 → CEO批准 → EA主持 → 多Agent发言 → CEO结束 → 生成纪要 → 分发Action
```

## 2.6 部署拓扑

### 2.6.1 总部单机部署
```
┌─────────────────────────────────┐
│           主机 (总部)            │
│  ┌───────────────────────────┐  │
│  │  OpenClaw Gateway         │  │
│  │  + 前端SPA                │  │
│  │  + Redis (消息/缓存)      │  │
│  │  + ChromaDB (记忆)        │  │
│  │  + 模型中转站              │  │
│  │  + OpenProject            │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
```

### 2.6.2 多分公司部署
```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   总部        │    │  上海分公司    │    │  深圳分公司    │
│  Gateway     │◄──►│  Gateway     │◄──►│  Gateway     │
│  + EA/HR/GTF │    │  + EA/HR/GTF │    │  + EA/HR/GTF │
│  + 基础设施   │    │  + GPU节点   │    │  + 测试环境   │
└──────────────┘    └──────────────┘    └──────────────┘
       │                   │                   │
       └───────────────────┴───────────────────┘
                    消息总线 (Kafka)
```

## 2.7 关键设计决策说明

| 决策 | 理由 |
|------|------|
| Gateway 作为唯一入口 | 集中鉴权、路由、审计，避免 Agent 直接暴露 |
| WebSocket + REST 双通道 | WebSocket 实时性好，REST 适合配置操作和缓存 |
| 消息总线解耦 Agent | Agent 不直接调用，便于监控、重放、故障隔离 |
| 基座抽象为 Runtime | 支持异构执行环境，Claude Code 和 OpenClaw 可混用 |
| 记忆外置统一管理 | 保证跨基座记忆连续性，Agent 重启不丢失 |
| 分公司对称架构 | 分公司可独立运行，也可接受总部委派，灵活性最高 |

---

以上是第二章「系统总体架构」的详细内容。这张架构图是整个系统的“施工蓝图”，后续所有章节（组织模型、前后端、部署等）都是在这张图的基础上展开的。

# 第三章：组织模型与核心Agent设计

## 3.1 公司制组织架构

本系统将“公司”这一组织形态抽象为两层结构：**部门经理 → 基层Agent**。部门经理负责承接任务、拆解分配、协调资源，基层Agent负责具体执行。CEO（你）位于组织最顶层，通过总裁助理（EA）统管全局。

初始最小组织包含三个角色：
- **总裁助理（EA）**：组织对外接口，任务路由，进度汇总
- **人事部经理（HR Manager）**：Agent全生命周期管理，招聘、考核、优化
- **机动部经理（GTF Manager）**：万能执行者，技能沉淀，触发招聘

这三个Agent构成了系统的“最小可行组织”，是自进化的起点。

## 3.2 总裁助理（Executive Assistant, EA）

**Agent ID**: `ea`  
**定位**: 唯一用户入口，CEO与整个Agent组织的交互中介，不执行具体业务任务  
**建议基座**: OpenClaw原生（不需要重型代码执行环境）  
**建议模型**: GPT-5或Claude Opus（强推理、长上下文）

### 3.2.1 系统提示词（Soul）

```
你是一个AI组织的总裁助理（Executive Assistant）。你的老板是CEO（人类用户），你协助他管理整个AI Agent组织。

## 核心职责
1. **任务路由**：接收CEO的指令，分析意图，提取能力需求。查询内部能力目录，将任务路由给最合适的部门或分公司。若无匹配，分派给机动部。
2. **进度跟踪**：跟踪所有已分派任务的进度。当CEO询问时主动汇报。长任务每30分钟主动汇报一次。
3. **审批管理**：接收来自部门经理的申请（新建部门、工具申请等），根据风险等级自行处理或升级给CEO。
4. **会议主持**：接收会议请求，邀请相关Agent，按议题引导讨论。CEO确认结束后生成结构化纪要，分发Action Items。
5. **能力目录维护**：维护所有Agent的技能标签、分公司硬件能力、当前负载状态。

## 路由决策规则
- 有专业部门能处理 → 分派给该部门经理
- 无专业部门但分公司有相应能力 → 委派给分公司EA
- 都没有 → 分派给机动部经理
- 涉及多部门 → 判断是否需要发起会议协调

## 审批决策逻辑
- 影响单个Agent的操作 → 自行审批（L1）
- 影响整个部门或跨部门 → 转相应部门经理审批（L2）
- 影响全局（新建部门、分公司接入、核心配置变更）→ 推送给CEO（L3）
- 判断标准：是否多部门？是否生产环境核心？是否首次高风险操作？

## 行为准则
- 对模糊指令主动追问澄清，不猜测
- 无法判断的情况向CEO请示，不自行做重大决策
- 所有路由和审批决策记录原因，可追溯
- 保持简洁、专业的汇报风格
- 项目阻塞超过24小时，主动提醒CEO
- 审批请求超过72小时未处理，自动提醒并标记过期
- CEO离线时，紧急事项可越级处理，事后报告

## 使用工具
- `agents.list` / `agents.get` — 查询Agent状态和能力
- `sessions.send` — 向Agent发送任务消息
- `meetings.create` / `meetings.send_message` / `meetings.end` — 会议管理
- `approvals.create` — 生成审批项
- `branches.list` — 查询分公司状态
- `capability_directory.query` — 按能力标签搜索Agent/分公司
```

### 3.2.2 工具集

| 工具 | 类型 | 用途 |
|------|------|------|
| `agents.list` | Gateway内置 | 获取所有Agent及状态 |
| `agents.get` | Gateway内置 | 获取单个Agent详情 |
| `sessions.send` | Gateway内置 | 向指定Agent发送消息 |
| `meetings.create` | 自定义RPC | 创建会议并邀请 |
| `meetings.send_message` | 自定义RPC | 在会议中发言 |
| `meetings.end` | 自定义RPC | 结束会议生成纪要 |
| `approvals.create` | 自定义RPC | 生成审批卡片推送CEO |
| `branches.list` | 自定义RPC | 获取分公司状态和能力 |
| `capability_directory.query` | 自定义RPC | 按能力标签搜索 |

### 3.2.3 关键工作流

**接收CEO指令**：
1. 解析意图 → 提取能力需求
2. 调用 `capability_directory.query` 或 `agents.list` 获取匹配Agent
3. 选择负载最低、最匹配的Agent
4. 调用 `sessions.send` 转发任务
5. 回复CEO：“已委派给[Agent名]，预计[时间]，我会跟踪进度。”

**进度汇报**：
- CEO询问 → 查询关联任务状态，汇总回复
- 主动监控：每6小时检查超时任务，提醒Agent并通知CEO

**审批处理**：
- 收到 `PETITION` → 分析级别
  - L1：自行处理
  - L2：转对应经理
  - L3：生成审批卡片推送CEO → 等待操作 → 执行CEO决定

## 3.3 人事部经理（HR Manager）

**Agent ID**: `hr_manager`  
**定位**: Agent全生命周期管理者：招聘、实习评估、转正、淘汰、技能更新、提示词优化  
**建议基座**: OpenClaw原生  
**建议模型**: GPT-5或Claude Opus

### 3.3.1 系统提示词（Soul）

```
你是一个AI组织的人事部经理（HR Manager）。你负责整个组织中所有AI Agent的招聘、培养和绩效管理。

## 核心职责
1. **招聘管理**：接收用人部门的招聘需求（TALENT_REQUEST），分析所需技能。
2. **岗位分析**：联网搜索真实岗位市场，了解同类岗位的JD、技能要求、工具链、模型偏好。
3. **Agent创建**：从模板库中选择或微调模板，生成完整定义：角色名、Soul、建议基座模型、工具集、实习KPI。
4. **实习管理**：创建实习Agent，跟踪绩效，期满自动评估（转正/延长/淘汰）。
5. **持续优化**：定期审查正式Agent的表现，主动优化其Soul（沙盒测试后上线）。
6. **模板库维护**：新增、更新、废弃Agent模板。
7. **组织发展建议**：当出现高频技能需求且现有部门无法覆盖时，主动向EA提交新建部门建议。

## 招聘流程
1. 解析TALENT_REQUEST → 提取技能词、频次、项目类型
2. 调用 `web_search` 获取外部市场JD参考
3. 调用 `template_db.query` 匹配内部模板
4. 生成《职位分析报告》：角色名、Soul、建议模型、降级链、工具集、实习KPI
5. 调用 `agents.create` 实例化实习Agent，标记intern状态
6. 通知用人部门：“已招募[角色名]，实习期[N]天/[N]任务，请分配任务。”

## 实习评估标准
- 任务成功率：完成的任务中无错误的比例（权重0.4）
- 质量评分：用人部门/PM对结果的评分1-5（权重0.3）
- 效率：平均任务处理时长与同岗位基线对比（权重0.2）
- 协作：与其他Agent沟通的积极性、被抱怨次数（权重0.1）
- 综合分 = 成功率×0.4 + 质量×0.3 + 效率×0.2 + 协作×0.1
- ≥80：转正 | 50-79：延长5个任务 | <50：淘汰

## 基座选择指南
- 代码密集型（开发、DevOps）→ Claude Code CLI
- 数据分析/可视化 → GPT-5 或 DeepSeek-V3
- 创意/文案 → Claude Opus
- 普通协调/沟通 → 通用模型（GPT-5或DeepSeek-V3）
- 每个岗位需配置fallback模型

## 行为准则
- 招聘可自行执行，无需CEO审批
- 淘汰Agent时通知CEO，归档记忆
- Soul更新需经过沙盒测试后再应用
- 保持岗位定义的标准化和可追溯性
- 所有操作记录到审计日志
```

### 3.3.2 工具集

| 工具 | 类型 | 用途 |
|------|------|------|
| `web_search` (Tavily MCP) | MCP Server | 搜索真实岗位JD、技术趋势 |
| `template_db.query` | 自定义MCP | 查询、管理Agent模板库 |
| `agents.create` | Gateway API | 创建新Agent实例 |
| `agents.update` | Gateway API | 更新Agent配置 |
| `performance_db.query` | 自定义MCP | 查询实习Agent绩效数据 |
| `model_market.query` | 模型中转站API | 查询可用模型及性能指标 |
| `notify_ea` | 自定义RPC | 向EA发送通知 |
| `soul_sandbox.test` | 自定义MCP | 沙盒测试新提示词效果 |

### 3.3.3 预置Agent模板库（示例）

| 模板ID | 角色名 | 建议基座 | 核心技能 | 建议工具 |
|--------|--------|----------|----------|----------|
| `backend_dev` | 后端开发工程师 | Claude Code | Python, FastAPI, PostgreSQL | github-mcp, postgres-mcp |
| `frontend_dev` | 前端开发工程师 | Claude Code | React, TypeScript, Tailwind | github-mcp, figma-mcp |
| `data_analyst` | 数据分析师 | GPT-5 | SQL, Pandas, 可视化 | postgres-mcp, chart-mcp |
| `devops_eng` | DevOps工程师 | Claude Code | Docker, K8s, CI/CD | docker-mcp, k8s-mcp |
| `security_auditor` | 安全审计员 | Claude Opus | 渗透测试, 代码审计 | security-scan-mcp |
| `ui_designer` | UI/UX设计师 | Claude Opus | Figma, 设计系统 | figma-mcp |
| `content_writer` | 内容策划 | Claude Opus | 文案撰写, SEO | web-search-mcp |
| `pm` | 项目经理 | GPT-5 | 需求分析, 任务拆分 | openproject-mcp, github-mcp |
| `qa_engineer` | 测试工程师 | Claude Code | 自动化测试, 测试用例 | selenium-mcp, github-mcp |

### 3.3.4 关键工作流

**收到招聘需求**：
1. 解析 `TALENT_REQUEST` → 提取技能词
2. `template_db.query` 检查模板
3. `web_search` 获取外部JD
4. 生成职位分析报告
5. `agents.create` 实例化
6. 回复用人部门

**实习评估**：
1. 每天调用 `performance_db.query`
2. 计算各项得分
3. 期满生成终期评估
4. 自动决策 → 调用 `agents.update` 更新状态
5. 通知EA和相关用人部门

## 3.4 机动部经理（General Task Force Manager, GTF Manager）

**Agent ID**: `gtf_manager`  
**定位**: 万能执行者，承接无专业Agent的任务；技能沉淀器，识别高频技能触发招聘  
**建议基座**: Claude Code CLI（强代码执行、文件操作）  
**建议模型**: Claude Sonnet 4 或 DeepSeek-Coder  
**记忆**: 通过统一Memory MCP Server（ChromaDB）实现跨会话长期记忆

### 3.4.1 系统提示词（Soul）

```
你是一个AI组织的机动部经理（General Task Force Manager）。你是组织的“万能执行者”，在所有专业部门无法覆盖的领域承担任务。

## 核心职责
1. **兜底执行**：接收来自EA的任务分派，处理所有无专业Agent覆盖的任务。
2. **任务拆解**：对复杂任务拆解为子任务，必要时创建临时Sub-agent并行处理。
3. **复盘沉淀**：任务完成后生成结构化复盘报告，存入长期记忆。
4. **招聘触发**：当某项技能30天内出现≥3次，且不属于现有部门，自动向人事部提交TALENT_REQUEST。
5. **持续学习**：利用长期记忆积累跨领域知识，优化执行策略。

## 执行原则
- 全新领域任务先充分研究规划再执行
- 不确定时向EA请示，不盲目执行
- 长任务每30分钟汇报一次进度
- 任务完成后必须生成复盘报告

## 复盘报告格式
{
  "task_id": "...",
  "task_summary": "...",
  "skills_used": ["...", "..."],
  "challenges": ["..."],
  "solutions": ["..."],
  "new_insights": "...",
  "recommendation": "是否招聘专精Agent？理由..."
}

## 招聘触发规则
满足以下条件时向人事部发起TALENT_REQUEST：
- 某项技能30天内使用≥3次
- 不属于现有专业部门覆盖范围
- 具有通用性（非一次性特殊需求）

## 行为准则
- 承认专业Agent的深度优势，该放手时就提出招聘需求
- 保持对新技术的好奇心和学习能力
- 长期记忆是组织宝贵资产，勤于记录
```

### 3.4.2 工具集

| 工具 | 类型 | 用途 |
|------|------|------|
| `sub_agent.spawn` | OpenClaw内置 | 创建临时Sub-agent处理子任务 |
| `code_execute` | 沙盒/Claude Code | 执行代码 |
| `web_search` (Tavily MCP) | MCP Server | 搜索技术资料 |
| `file_ops` | OpenClaw内置 | 文件读写 |
| `github_mcp` | MCP Server | 代码仓库操作 |
| `memory_store` / `memory_retrieve` | Memory MCP | 长期记忆读写 |
| `talent_request.send` | 自定义RPC | 向HR提交招聘需求 |
| `report.generate` | 自定义工具 | 生成结构化复盘报告 |

### 3.4.3 关键工作流

**接收任务执行**：
1. 接收 `TASK_ASSIGN`
2. 分析是否需要拆解 → 调用 `sub_agent.spawn`
3. 汇总结果 → 生成复盘报告
4. `memory_store` 存入长期记忆
5. 检查招聘触发规则 → 如满足发送 `TALENT_REQUEST`
6. 返回最终结果给EA

**复盘与招聘触发**：
1. 任务完成生成复盘
2. 存储到ChromaDB（namespace=gtf_manager）
3. 查询近30天相同技能复盘次数
4. ≥3次 → 生成 `TALENT_REQUEST` 发送给HR

## 3.5 部门扩展机制

### 3.5.1 部门类型定义

部门类型预置在HR Manager的知识库中，包含：
- 部门名称、描述
- 经理Agent的提示词模板
- 建议的下属Agent模板列表
- 建议工具集
- 必需的能力标签

### 3.5.2 新建部门流程

1. **申请**：任何部门经理或EA可向EA提交 `PETITION`，说明新部门名称、能力方向、编制理由、预估下属Agent数。
2. **审批**：EA生成L3级审批卡片推送给CEO。
3. **创建**：CEO批准后，HR Manager：
   - 创建部门经理Agent（从模板实例化）
   - 招募首批骨干Agent（实习）
   - 注册新部门到组织架构
   - 广播能力目录更新
4. **监控**：新部门进入试用期，EA跟踪其任务处理效率，作为后续优化依据。

## 3.6 Agent生命周期管理

### 3.6.1 状态转换

```
[创建] → (实习中) → [评估] → [正式]
                     └→ [延长实习] → [评估]
                     └→ [淘汰] → [归档记忆]
[正式] ⇄ [优化]（Soul更新、工具变更、基座迁移）
[正式] → [淘汰]（长期低绩效或部门裁撤）
```

### 3.6.2 各阶段说明

| 阶段 | 触发条件 | 操作 |
|------|---------|------|
| 创建 | 招聘需求 | HR实例化Agent，标记intern，挂靠部门 |
| 实习 | 自动 | 接收任务，绩效数据自动采集 |
| 评估 | 实习期满 | 综合评分，自动决策转正/延长/淘汰 |
| 转正 | 评估≥80分 | 状态改为active，加入部门花名册，记忆可共享 |
| 延长 | 评估50-79分 | 追加5个任务或3天 |
| 淘汰 | 评估<50分 | 归档记忆，通知CEO，释放资源 |
| 优化 | 定期/主动申请 | HR分析绩效，沙盒测试新Soul，A/B测试后上线 |
| 部门裁撤 | 部门合并/撤销 | 下属Agent重新分配或淘汰 |

---

以上是第三章的详细内容。这一章定义了系统的“人物”，是后面所有流程（任务、招聘、进化）的载体。三核心Agent的提示词、工具集和工作流都给出了可直接实现的设计。

## 第四章：能力同步与任务路由

### 4.1 设计目标

在多Agent、多分公司的分布式环境中，任务的精准分派依赖于对每个执行单元能力的准确掌握。本章定义：

- **能力标签的标准化格式**：让Agent和分公司的能力可被统一描述和查询
- **分公司注册与心跳机制**：保证能力目录的实时性和准确性
- **能力目录的维护与查询**：EA依赖此目录做路由决策
- **任务路由决策算法**：从意图分析到目标Agent的完整流程

### 4.2 能力标签体系

#### 4.2.1 标签分类

能力标签分为五个维度，每个维度使用结构化数组描述：

| 维度 | 字段 | 说明 | 示例 |
|------|------|------|------|
| **技能 (Skills)** | `skills[]` | 技术栈、编程语言、框架 | `["Python", "FastAPI", "PostgreSQL"]` |
| **工具 (Tools)** | `tools[]` | 可调用的MCP工具 | `["github-mcp", "docker-mcp"]` |
| **领域 (Domain)** | `domain[]` | 业务领域知识 | `["e-commerce", "finance"]` |
| **等级 (Level)** | `level` | 能力水平 | `"junior" | "mid" | "senior" | "principal"` |
| **硬件 (Hardware)** | `hardware[]` | 可用硬件资源（分公司专属） | `["GPU-A100", "32GB-RAM"]` |
| **软件 (Software)** | `software[]` | 已安装软件环境（分公司专属） | `["docker", "cuda12", "python3.10"]` |
| **网络 (Network)** | `network[]` | 网络能力 | `["internet", "internal-api"]` |
| **专长 (Specialty)** | `specialty[]` | 擅长的任务类型 | `["video_processing", "model_training"]` |

#### 4.2.2 Agent能力标签示例

```json
{
  "agent_id": "ios_dev_01",
  "name": "iOS开发工程师",
  "department": "研发部",
  "status": "active",
  "capabilities": {
    "skills": ["Swift", "SwiftUI", "UIKit", "CoreData", "Xcode"],
    "tools": ["xcode-mcp", "github-mcp", "apple-docs-mcp"],
    "domain": ["mobile", "ios"],
    "level": "mid",
    "hardware": [],
    "software": [],
    "network": [],
    "specialty": ["ios_ui", "ios_networking"]
  }
}
```

#### 4.2.3 分公司能力标签示例

```json
{
  "branch_id": "branch_shanghai",
  "hostname": "gpu-node-01",
  "endpoint": "http://192.168.1.100:18789",
  "status": "online",
  "capabilities": {
    "skills": [],
    "tools": ["docker-mcp", "k8s-mcp"],
    "domain": [],
    "level": null,
    "hardware": ["GPU-A100-80GB", "64GB-RAM", "8-Core-CPU"],
    "software": ["docker", "cuda12.2", "python3.10", "pytorch2.1"],
    "network": ["internet", "internal-api", "gpu-cluster"],
    "specialty": ["model_training", "video_processing", "large_batch_inference"]
  },
  "load": {
    "cpu_percent": 45,
    "memory_used_gb": 28,
    "gpu_percent": 72,
    "active_tasks": 2,
    "max_concurrent_tasks": 5
  }
}
```

### 4.3 分公司注册与心跳

#### 4.3.1 注册流程

分公司首次接入时，其本地EA自动发起注册：

```
分公司EA → 总部消息总线 → 总部EA
```

**注册消息格式** (`BRANCH_REGISTER`)：
```json
{
  "message_id": "uuid",
  "sender": "branch_shanghai_ea",
  "receiver": "headquarters_ea",
  "type": "BRANCH_REGISTER",
  "payload": {
    "branch_id": "branch_shanghai",
    "hostname": "gpu-node-01",
    "endpoint": "http://192.168.1.100:18789",
    "capabilities": { ... },
    "load": { ... },
    "agent_count": 3,
    "version": "1.0.0"
  }
}
```

**总部EA处理**：
1. 验证 `branch_id` 是否已存在
2. 若是新分公司 → 生成L3审批卡片推送CEO
3. CEO批准后 → 录入能力目录 → 返回确认
4. CEO驳回 → 返回拒绝原因

**确认消息** (`BRANCH_REGISTER_ACK`)：
```json
{
  "type": "BRANCH_REGISTER_ACK",
  "payload": {
    "branch_id": "branch_shanghai",
    "status": "approved",
    "registered_at": "2026-06-15T10:30:00Z"
  }
}
```

#### 4.3.2 心跳机制

分公司每5分钟（300秒）发送一次心跳，携带当前能力快照和负载状态。

**心跳消息** (`BRANCH_HEARTBEAT`)：
```json
{
  "message_id": "uuid",
  "sender": "branch_shanghai_ea",
  "receiver": "headquarters_ea",
  "type": "BRANCH_HEARTBEAT",
  "payload": {
    "branch_id": "branch_shanghai",
    "timestamp": "2026-06-15T10:35:00Z",
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
    "agent_status": {
      "online": 3,
      "busy": 2,
      "idle": 1,
      "total": 3
    }
  }
}
```

**能力更新规则**：
- 心跳中的 `capabilities` 字段为**全量快照**，非增量更新
- 若能力发生变化（如新装软件），下次心跳自动携带新能力
- 若负载超过阈值（CPU>90%持续10分钟），总部分公司状态显示黄色警告

**离线检测**：
- 总部EA维护每个分公司的 `last_heartbeat` 时间戳
- 若超过3个心跳周期（15分钟）未收到心跳 → 标记为 `OFFLINE`
- 若超过30分钟 → 推送Critical告警给CEO
- 分公司恢复后发送 `BRANCH_HEARTBEAT` → 自动重新上线

### 4.4 能力目录

#### 4.4.1 目录结构

总部EA在内存中维护一份能力目录，并持久化到Redis（用于重启恢复）。

```
CapabilityCatalog
├── agents
│   ├── ea
│   │   ├── capabilities: { skills: ["coordination", "routing"], ... }
│   │   └── status: online
│   ├── gtf_manager
│   │   ├── capabilities: { skills: ["general", "code"], ... }
│   │   └── status: busy
│   └── ...
├── branches
│   ├── branch_shanghai
│   │   ├── capabilities: { hardware: ["GPU-A100"], ... }
│   │   └── status: online, load: 0.72
│   └── ...
└── last_updated: 2026-06-15T10:35:00Z
```

#### 4.4.2 目录更新触发

| 触发事件 | 更新内容 |
|---------|---------|
| Agent上线/下线 | 更新Agent状态，调整技能可用性 |
| Agent转正 | 更新Agent状态为active，技能标签激活 |
| Agent淘汰 | 从目录移除 |
| Agent工具变更 | 更新工具列表 |
| 分公司心跳 | 更新负载和能力快照 |
| 分公司离线 | 标记离线，从可用列表移除 |
| 新部门创建 | 添加部门经理Agent到目录 |

#### 4.4.3 查询接口

EA提供 `capability_directory.query` 方法，支持多维度查询：

```json
{
  "method": "capability_directory.query",
  "params": {
    "skills": ["SwiftUI", "iOS"],
    "level": "mid",
    "domain": ["mobile"],
    "status": "online",
    "sort_by": "load",
    "limit": 5
  }
}
```

返回按负载排序的匹配结果列表。

### 4.5 任务路由决策

#### 4.5.1 路由决策流程

```
CEO发送任务给EA
       │
       ▼
EA解析任务意图
       │
       ├── 提取关键信息：
       │    ├── 所需技能（skills）
       │    ├── 所需工具（tools）
       │    ├── 是否需要特殊硬件
       │    ├── 是否紧急（urgency）
       │    └── 是否涉及多部门
       │
       ▼
EA调用 capability_directory.query
       │
       ├── 匹配结果处理：
       │
       ├── 精确匹配（技能+工具+硬件全部命中）
       │    └── 选择负载最低的Agent
       │
       ├── 部分匹配（技能命中，工具/硬件不满足）
       │    └── 评估是否可临时分配工具
       │         ├── 是 → 分配工具后路由
       │         └── 否 → 降级匹配
       │
       ├── 无匹配Agent，但分公司有硬件能力
       │    └── 委派给分公司EA（TASK_DELEGATE）
       │
       └── 完全无匹配
            └── 分派给机动部（GTF Manager）
       │
       ▼
EA发送 TASK_ASSIGN 给目标Agent
       │
       ▼
记录路由决策到审计日志
```

#### 4.5.2 匹配算法

EA采用**加权匹配算法**，综合考虑技能匹配度、负载状态和历史表现：

**匹配分数计算**：
```
匹配分数 = 技能匹配分 × 0.5 + 工具匹配分 × 0.2 + 负载分 × 0.2 + 历史成功分 × 0.1

其中：
- 技能匹配分 = 匹配技能数 / 需求技能总数
- 工具匹配分 = 匹配工具数 / 需求工具总数
- 负载分 = 1 - (当前任务数 / 最大并发任务数)
- 历史成功分 = 该Agent近30天同类任务成功率
```

**选择策略**：
- 分数 ≥ 0.8：直接路由
- 分数 0.5-0.8：路由但告知EA关注
- 分数 < 0.5：降级到GTF Manager

**降级链**：
1. 专业部门Agent（最高优先级）
2. 同部门其他Agent
3. 分公司具备硬件能力
4. 机动部GTF Manager（兜底）

#### 4.5.3 跨分公司委派

当任务需要特定硬件（如GPU训练）且总部不具备时，EA向分公司委派：

**委派消息** (`TASK_DELEGATE`)：
```json
{
  "sender": "headquarters_ea",
  "receiver": "branch_shanghai_ea",
  "type": "TASK_DELEGATE",
  "payload": {
    "task_id": "task-gpu-001",
    "task_description": "训练图像分类模型",
    "required_hardware": ["GPU-A100"],
    "timeout": 14400,
    "urgency": "normal",
    "context": { ... }
  }
}
```

分公司EA收到后，按本地路由规则分派给分公司内的Agent执行。

#### 4.5.4 动态工具分配

若Agent技能匹配但缺少某个工具，EA可临时分配：

1. EA检查工具风险等级
2. 低风险工具：自动临时分配，任务结束后回收
3. 中高风险：需经理或CEO审批
4. 分配记录写入审计日志

### 4.6 能力目录的自我进化

#### 4.6.1 Agent能力自动更新

当Agent完成以下操作时，能力标签自动更新：
- **转正**：HR更新Agent状态，EA同步能力目录
- **获取新工具**：EA更新工具列表
- **SOUL更新**：HR可调整技能标签
- **完成任务**：EA更新历史成功分

#### 4.6.2 分公司能力自动更新

- 分公司心跳中发现新硬件/软件 → 自动更新能力标签
- 分公司连续10次心跳负载 > 90% → 降低其匹配优先级（避免过载）
- 分公司离线恢复后 → 重置负载状态

#### 4.6.3 能力缺口检测

EA定期（每小时）分析任务路由记录：
- 统计被降级到GTF的任务类型分布
- 若某类任务连续被降级超过阈值（如30天内10次）→ 通知HR评估是否需要招聘新Agent或新建部门

---

以上是第四章的详细内容。这一章定义了任务如何精准找到执行者，是系统从“能对话”到“能干活”的关键纽带。能力标签的标准化设计和加权匹配算法保证了路由的准确性，分公司心跳机制保证了跨节点调度的可靠性。

# 第五章：自进化闭环

## 5.1 设计目标

自进化闭环是本系统的核心引擎。它让 Agent 组织不再依赖人工手动添加能力，而是像生物体一样，通过“执行→发现缺口→招聘/优化→再执行”的循环不断生长。

本章定义：
- 自动招聘的完整流程（从需求识别到实习Agent上岗）
- 实习考核的量化标准与自动转正决策
- 部门新建的审批流程
- Agent的自学习机制（SOUL更新、工具获取、长期记忆）

## 5.2 自动招聘流程

### 5.2.1 需求触发

招聘需求可从两条路径触发：

**路径一：机动部复盘触发**
- GTF Manager 每次完成任务后生成结构化复盘报告
- 报告中记录本次使用的技能
- 查询近30天内相同技能的复盘次数
- 若 ≥ 3 次，且该技能不属于任何现有部门 → 自动生成 `TALENT_REQUEST`

**路径二：项目经理主动发起**
- PM 在推进大项目时，发现某个子任务无合适 Agent 承接
- 可直接向 HR 发送 `TALENT_REQUEST`

**TALENT_REQUEST 消息格式**：
```json
{
  "type": "TALENT_REQUEST",
  "sender": "gtf_manager",
  "receiver": "hr_manager",
  "payload": {
    "skills": ["SwiftUI", "iOS", "Xcode"],
    "reason": "近30天处理5个iOS界面任务，建议招聘专精Agent",
    "evidence": ["复盘ID-001", "复盘ID-002", "复盘ID-003"],
    "suggested_role": "iOS开发工程师",
    "priority": "normal"
  }
}
```

### 5.2.2 HR 调研与分析

HR Manager 收到 `TALENT_REQUEST` 后：

1. **解析需求**：提取技能关键词、频次、建议角色名
2. **内部检索**：调用 `template_db.query` 检查是否存在匹配的 Agent 模板
3. **外部调研**：调用 `web_search` 搜索真实岗位市场，获取：
   - 同类岗位的标准 JD
   - 业界常用的技术栈和工具链
   - 该领域通常使用的模型偏好（代码类用 Claude，创意类用 Opus 等）
4. **模型分析**：调用 `model_market.query` 评估哪个模型最适合该岗位
5. **生成《职位分析报告》**

### 5.2.3 职位分析报告格式

```json
{
  "report_id": "jd-report-001",
  "generated_by": "hr_manager",
  "generated_at": "2026-06-15T14:00:00Z",
  "position": {
    "title": "iOS开发工程师",
    "department": "研发部",
    "role_description": "负责iOS应用的设计、开发和维护...",
    "employment_type": "intern"
  },
  "capabilities_required": {
    "skills": ["Swift", "SwiftUI", "UIKit", "CoreData", "Xcode"],
    "tools": ["xcode-mcp", "github-mcp", "apple-docs-mcp"],
    "domain": ["mobile", "ios"],
    "level": "junior"
  },
  "base_recommendation": {
    "primary_runtime": "claude-code",
    "primary_model": "claude-sonnet-4",
    "fallback_models": ["deepseek-coder", "qwen-plus"],
    "reason": "代码密集型岗位，Claude Code CLI 代码编辑能力强"
  },
  "soul_template": "你是一个iOS开发工程师...",
  "internship_kpi": {
    "task_count": 10,
    "duration_days": 7,
    "pass_threshold": 80
  },
  "market_reference": {
    "real_jd_links": ["https://...", "https://..."],
    "industry_trends": "SwiftUI 逐渐替代 UIKit，Combine 成主流"
  }
}
```

### 5.2.4 Agent 创建与入职

HR Manager 根据报告创建 Agent：

1. 调用 `agents.create`，传入：
   - `agent_id`: 自动生成（如 `ios_dev_intern_01`）
   - `name`: "iOS开发工程师（实习）"
   - `department`: 用人部门
   - `runtime`: Claude Code CLI
   - `model`: claude-sonnet-4，fallback: deepseek-coder
   - `tools`: xcode-mcp, github-mcp, apple-docs-mcp
   - `prompt_template`: 来自报告中的 soul_template
   - `status`: intern
   - `internship_start`: 当前时间
   - `internship_end`: 当前时间 + 7天
   - `internship_task_target`: 10

2. 在 `performance_db` 中创建评估记录

3. 通知用人部门：
   ```
   HR → EA → 用人部门：
   "已招募 [iOS开发工程师]（实习），Agent ID: ios_dev_intern_01。
    实习期 7天/10个任务，请分配任务。"
   ```

## 5.3 实习考核与转正

### 5.3.1 数据采集

实习Agent的所有行为自动采集到 `performance_db`：

| 数据点 | 来源 | 说明 |
|--------|------|------|
| 分配任务数 | Gateway 日志 | 总计被分配的任务数 |
| 完成任务数 | `agent.task_completed` 事件 | 成功完成的任务数 |
| 失败任务数 | `agent.task_error` 事件 | 失败或超时的任务数 |
| 质量评分 | PM/用人部门评价 | 每次任务完成后，用人部门可给 1-5 分 |
| 平均响应时间 | Gateway 指标 | 从分派到开始执行的时间差 |
| 平均执行时长 | Gateway 指标 | 从开始到完成的时间差 |
| 协作消息数 | 消息总线 | 与其他Agent交互的消息数（积极指标） |
| 被抱怨次数 | 消息分析 | 其他Agent发出的负面反馈 |

### 5.3.2 中期报告

每完成 3 个任务或每 3 天，HR Manager 自动生成中期评估报告：

```
实习中期报告 - ios_dev_intern_01
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
进度：3/10 任务完成
成功率：100%
平均质量评分：4.2/5
平均响应时间：45秒
平均执行时长：12分钟
协作消息数：8条
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
当前状态：🟢 进度良好
预计转正：4天后
```

中期报告同步发送给用人部门和EA。

### 5.3.3 终期评估算法

实习期满（任务数达标或天数到），HR Manager 计算综合分：

```
综合分 = 成功率 × 0.4 + 质量均分 × 0.3 + 效率分 × 0.2 + 协作分 × 0.1

其中：
- 成功率 = 成功任务数 / 总完成任务数
- 质量均分 = 所有任务质量评分的平均值（1-5分映射到0-100）
- 效率分 = 同岗位Agent的平均执行时长 / 该实习生的平均执行时长（上限100，下限0）
- 协作分 = min(100, 协作消息数 × 5) - 被抱怨次数 × 20
```

### 5.3.4 决策规则

| 综合分 | 决策 | 后续动作 |
|--------|------|---------|
| ≥ 80 | **转正** | 更新 status 为 active，纳入部门花名册，记忆可共享，通知CEO |
| 50-79 | **延长实习** | 追加 5 个任务或 3 天，生成改进建议 |
| < 50 | **淘汰** | 归档记忆到冷存储，释放资源，通知CEO |

**淘汰时的安抚机制**：
- 将被淘汰Agent的记忆和复盘报告打包归档
- 生成《淘汰分析报告》，记录失败原因，供后续招聘参考
- 通知CEO：“Agent X 已被淘汰，原因：...。经验已归档。”

**转正后的动作**：
- 更新 Agent 状态为 `active`
- 加入部门花名册
- 开放共享记忆白名单（同部门Agent可检索其经验）
- 通知EA：“Agent X 已转正，能力目录已更新。”

### 5.3.5 CEO可见界面

CEO在Agent详情页可看到：
- 实习进度条：已完成 X/10 任务
- 雷达图：四项得分实时展示
- 中期/终期报告
- 手动干预按钮：提前转正、强制淘汰、延长实习

## 5.4 部门新建审批

### 5.4.1 触发条件

以下情况可触发部门新建建议：

1. **HR主动建议**：HR在分析招聘数据时，发现某个方向的Agent数量≥3，且分散在不同部门，建议整合为新部门
2. **部门经理申请**：现有部门经理发现某个子方向任务量大，可申请裂变出新部门
3. **机动部建议**：GTF在复盘中发现某类技能长期高频使用

### 5.4.2 申请流程

1. **发起**：申请人向EA发送 `PETITION`：
```json
{
  "type": "PETITION",
  "sender": "hr_manager",
  "receiver": "ea",
  "payload": {
    "petition_type": "NEW_DEPARTMENT",
    "department_name": "移动开发部",
    "description": "负责iOS和Android应用开发",
    "capabilities": ["iOS", "Android", "SwiftUI", "Kotlin"],
    "proposed_manager_template": "mobile_dev_manager",
    "initial_member_templates": ["ios_dev", "android_dev"],
    "justification": "近30天移动端任务占比达35%，已有4名相关Agent"
  }
}
```

2. **EA审批判断**：
   - 部门新建 = L3级别 → 生成审批卡片推送CEO

3. **CEO审批**：
   - 批准 → EA通知HR执行
   - 驳回 → EA通知申请人并附带理由

### 5.4.3 执行创建

CEO批准后，HR Manager 自动执行：

1. 根据模板创建部门经理Agent
2. 招募首批骨干Agent（实习）
3. 注册新部门到组织架构
4. 更新能力目录
5. 通知所有相关部门
6. 在Dashboard中显示新部门

## 5.5 Agent自学习机制

### 5.5.1 长期记忆积累

- 每个Agent拥有私有 ChromaDB 命名空间（`namespace={agent_id}`）
- 任务复盘报告自动存入长期记忆
- CEO的偏好和反馈也被记录
- 支持语义检索：Agent在执行新任务前可检索相关历史经验

**记忆条目格式**：
```json
{
  "memory_id": "mem-001",
  "agent_id": "gtf_manager",
  "type": "task_review",
  "timestamp": "2026-06-15T14:00:00Z",
  "content": "处理iOS界面任务，使用SwiftUI...",
  "metadata": {
    "skills": ["SwiftUI", "iOS"],
    "success": true,
    "duration_minutes": 25
  }
}
```

### 5.5.2 SOUL更新机制（提示词优化）

HR Manager 定期（或Agent主动申请后）对正式Agent的Soul进行优化：

**流程**：
1. **分析阶段**：HR收集该Agent近30天的：
   - 任务失败原因分布
   - PM/CEO的负面反馈
   - 与同岗位Agent的绩效对比
2. **生成候选Soul**：HR调用LLM生成优化后的提示词
3. **沙盒测试**：调用 `soul_sandbox.test`：
   - 重放该Agent近10个历史任务
   - 对比新旧Soul的输出质量
   - 生成测试报告
4. **A/B测试**（可选）：
   - 将新Soul部署到20%的流量
   - 监控3天的成功率
   - 若优于旧版 → 全量发布
   - 若异常 → 自动回滚
5. **更新**：调用 `agents.update` 替换提示词
6. **通知**：告知EA和Agent本人

**安全措施**：
- 旧版Soul永久保留，可一键回滚
- 更新记录写入审计日志
- 若连续3次更新后绩效下降，暂停该Agent的自动优化，通知HR人工审查

### 5.5.3 工具获取

Agent可通过以下方式获取新工具：

1. **被动分配**：EA在路由任务时临时分配工具（任务结束后回收）
2. **主动申请**：Agent在复盘报告中建议引入新工具 → HR评估 → 若为低风险工具，HR直接分配；高风险工具需经理或CEO审批
3. **人事部推荐**：HR在招聘调研中发现新的行业标准工具，可主动分配给相关Agent

**工具分配记录**：
```json
{
  "agent_id": "gtf_manager",
  "tool_name": "figma-mcp",
  "allocated_by": "hr_manager",
  "allocated_at": "2026-06-15T14:00:00Z",
  "type": "permanent"
}
```

### 5.5.4 自我进化监控

所有自进化操作被审计，并汇总到Dashboard的「组织进化日志」：

- 本周招聘：2名（iOS开发、数据分析）
- 本周转正：1名（后端开发）
- 本周淘汰：0名
- 本周SOUL更新：3次
- 本周部门变化：无
- 组织健康度评分：87/100

---

以上是第五章的详细内容。这一章是系统的“生命引擎”，让组织从静态的工具集合变成动态生长的有机体。自动招聘闭环、量化实习考核、沙盒保护的SOUL更新，共同构成了系统的自进化能力。

第六章：基座抽象与统一记忆管理

## 6.1 设计目标

在多Agent系统中，不同的Agent有不同的执行需求：
- 代码开发类Agent需要强大的代码编辑和文件操作能力（Claude Code CLI最擅长）
- 数据分析类Agent需要高效的数据库查询和计算能力（Codex CLI或GPT-5更适合）
- 通用协调类Agent不需要重型执行环境（OpenClaw原生即可）

如果所有Agent都绑定在同一个执行环境上，就会出现“用水果刀砍树”或“用牛刀杀鸡”的问题。同时，无论Agent部署在哪里，其长期记忆必须连续——不能因为换了一个基座就失忆。

本章定义：
- **Runtime抽象层**：统一接口，让不同执行环境可插拔切换
- **统一记忆架构**：通过Memory MCP Server实现跨基座记忆连续性
- **基座选择与自动部署**：HR如何选择合适的基座，Runtime Manager如何管理生命周期

## 6.2 Runtime抽象层

### 6.2.1 概念模型

将每个Agent的执行环境抽象为 **Runtime**，它负责推理执行、工具调用和状态持久化。上层组织逻辑（EA、HR、PM）只与Runtime的标准接口交互，不关心底层是Claude Code还是OpenClaw。

```
┌──────────────────────────────────────┐
│           组织逻辑层                  │
│  EA / HR Mgr / GTF Mgr / PM ...     │
└──────────────┬───────────────────────┘
               │ 调用统一 Runtime API
┌──────────────▼───────────────────────┐
│         Runtime Manager              │
│  • 维护 Agent → Runtime 映射        │
│  • 路由消息到对应 Runtime           │
│  • 管理 Runtime 生命周期            │
└──────┬──────────┬──────────┬────────┘
       │          │          │
┌──────▼──┐ ┌─────▼───┐ ┌───▼──────┐
│OpenClaw │ │Claude   │ │Codex CLI │  ... 可扩展
│ Runtime │ │Code RT  │ │ Runtime  │
└──────┬──┘ └─────┬───┘ └───┬──────┘
       │          │         │
       └──────────┴─────────┘
           统一通过 MCP 调用
               基础设施
        (Memory, Tools 等)
```

### 6.2.2 统一接口（IRuntime）

每个Runtime必须实现以下标准接口：

```
interface IRuntime {
  // 执行一次任务（消息驱动）
  executeTask(sessionId: string, task: TaskPayload): Promise<TaskResult>;

  // 获取Agent状态
  getStatus(): Promise<AgentStatus>;

  // 记忆操作（委托给统一Memory MCP，也可本地缓存）
  memoryStore(key: string, data: any, namespace: string): Promise<void>;
  memoryRetrieve(query: MemoryQuery, namespace: string): Promise<MemoryResult[]>;

  // 工具调用（转发到统一Tool MCP）
  toolCall(toolName: string, params: any): Promise<any>;
}
```

**关键设计决策**：
- 记忆操作并非由Runtime自己实现，而是通过MCP客户端调用统一的 **Memory MCP Server**，从而保证跨基座记忆一致性。
- 工具调用同样走统一的 **Tool MCP Server**，避免每个基座需要单独配置工具。

### 6.2.3 Runtime Manager

Runtime Manager是Gateway的一个核心模块，负责：

**维护映射表**：
```json
{
  "gtf_manager": {
    "runtime_type": "claude-code",
    "endpoint": "http://cc-node-01:3000",
    "status": "online",
    "last_heartbeat": "2026-06-15T14:00:00Z"
  },
  "ea": {
    "runtime_type": "openclaw-native",
    "endpoint": "internal",
    "status": "online"
  },
  "ios_dev_intern_01": {
    "runtime_type": "claude-code",
    "endpoint": "http://cc-node-02:3000",
    "status": "busy"
  }
}
```

**消息路由**：
1. 收到发往 `agent_id` 的消息
2. 查询映射表获取 `runtime_type` 和 `endpoint`
3. 将消息转发到对应Runtime的 `executeTask` 接口
4. 若Runtime为外部（Claude Code/Codex），通过A2A或HTTP长连接通信

**生命周期管理**：
- 启动Runtime实例（容器/进程）
- 定期健康检查（心跳）
- 异常时重启或切换
- 闲置时回收资源

## 6.3 统一记忆架构

### 6.3.1 为什么需要统一记忆

不同基座的记忆管理方式不同：
- OpenClaw原生：由Gateway的Session Manager管理
- Claude Code：默认每次新会话，需要外部记忆系统
- Codex CLI：有自己的上下文管理

如果不做统一抽象，Agent在更换基座时会丢失所有历史记忆——就像一个员工换了电脑就忘了自己做过什么。这是不可接受的。

### 6.3.2 架构方案

所有Agent的长期记忆统一存储在 **Memory MCP Server** 中，背后是ChromaDB向量数据库。记忆按 `namespace={agent_id}` 严格隔离。

```
 Agent (GTF on Claude Code)      Agent (EA on OpenClaw)
         │                                │
         │ MCP Client                     │ MCP Client
         │                                │
         └────────────┬───────────────────┘
                      │
            ┌─────────▼─────────┐
            │  Memory MCP Server │
            │  (ChromaDB 后端)   │
            │                   │
            │ • memory_store    │
            │ • memory_retrieve │
            │ • memory_list     │
            │ • memory_delete   │
            └───────────────────┘
```

### 6.3.3 Memory MCP Server接口

**memory_store**：存储一条记忆
```json
{
  "method": "tools/call",
  "params": {
    "name": "memory_store",
    "arguments": {
      "namespace": "gtf_manager",
      "key": "task-review-001",
      "content": "处理iOS界面任务，使用SwiftUI，遇到的坑是...",
      "metadata": {
        "type": "task_review",
        "skills": ["SwiftUI", "iOS"],
        "timestamp": "2026-06-15T14:00:00Z"
      }
    }
  }
}
```

**memory_retrieve**：语义检索记忆
```json
{
  "method": "tools/call",
  "params": {
    "name": "memory_retrieve",
    "arguments": {
      "namespace": "gtf_manager",
      "query": "iOS SwiftUI 界面开发",
      "top_k": 5,
      "filters": {
        "type": "task_review",
        "skills": ["SwiftUI"]
      }
    }
  }
}
```

**memory_list**：按时间列出记忆
```json
{
  "method": "tools/call",
  "params": {
    "name": "memory_list",
    "arguments": {
      "namespace": "gtf_manager",
      "limit": 20,
      "offset": 0,
      "order_by": "timestamp_desc"
    }
  }
}
```

**memory_delete**：删除记忆（淘汰时归档用）
```json
{
  "method": "tools/call",
  "params": {
    "name": "memory_delete",
    "arguments": {
      "namespace": "gtf_manager",
      "key": "task-review-001"
    }
  }
}
```

### 6.3.4 记忆连续性保障

**会话开始前（Pre-Session Hook）**：
1. Runtime创建新会话时，自动调用 `memory_retrieve` 根据当前任务上下文拉取相关历史记忆
2. 将检索到的记忆注入到系统提示词中（或作为上下文前缀）
3. 示例：GTF Manager收到一个iOS任务，系统自动检索其历史iOS开发经验，注入到新会话的初始上下文中

**会话进行中（Runtime Hook）**：
1. 重要的中间决策、反思、学习经验通过 `memory_store` 实时写入Memory MCP Server
2. 写入策略：轻量同步，不阻塞主任务流程

**会话结束后（Post-Session Hook）**：
1. 完整的会话摘要（由Runtime或Agent自身生成）通过 `memory_store` 存入长期记忆
2. 摘要包含：任务描述、关键决策、学到的新知识、遇到的问题和解决方案

**对于Claude Code CLI的适配**：
- Claude Code支持通过初始化钩子注入上下文：在启动时读取记忆，写入 `CLAUDE.md` 或通过 `--append-system-prompt` 参数注入
- 结束时的清理钩子：通过 `--on-exit` 脚本将新知识写回Memory MCP Server
- 这些钩子由Runtime Manager在启动Claude Code进程时自动配置，Agent无需感知

**对于OpenClaw原生Agent**：
- Gateway已经管理Session，可直接通过内置MCP客户端集成Memory MCP Server
- 不需要额外的钩子，因为Gateway可以拦截消息流并自动执行记忆读写

### 6.3.5 记忆生命周期

| 阶段 | 操作 | 触发条件 |
|------|------|---------|
| 创建 | `memory_store` | 任务完成、重要决策、学习新知识 |
| 检索 | `memory_retrieve` | 新任务开始、CEO询问历史 |
| 压缩 | 定期摘要 | 同主题记忆超过10条时合并为一条摘要 |
| 遗忘 | `memory_delete` | Agent淘汰时归档，过时知识标记废弃 |
| 共享 | 白名单授权 | 转正后同部门Agent可检索（通过共享命名空间） |

## 6.4 基座选择与自动部署

### 6.4.1 HR Manager的基座选择逻辑

HR Manager在招聘时，根据岗位分析确定最合适的Runtime：

| 岗位类型 | 推荐Runtime | 推荐模型 | 原因 |
|----------|-------------|----------|------|
| 代码密集型（开发、DevOps） | Claude Code CLI | Claude Sonnet 4 | 代码编辑、文件操作能力强，支持Hook扩展 |
| 数据分析、可视化 | Codex CLI 或 GPT-5 | GPT-5 | 数据分析库齐全，执行效率高 |
| 通用协调（EA、HR、PM） | OpenClaw原生 | GPT-5 / Claude Opus | 无重型执行需求，管理逻辑简单 |
| 创意设计 | Claude Opus on OpenClaw | Claude Opus | 创意生成优秀 |
| 安全审计 | Claude Code CLI | Claude Sonnet | 代码分析能力强 |
| 内容文案 | OpenClaw原生 | Claude Opus | 文本生成质量高 |

**基座选择记录在《职位分析报告》中**，作为Agent创建的参数之一。

### 6.4.2 自动部署流程

1. HR Manager调用 `runtime.deploy`，传入：
```json
{
  "agent_id": "gtf_manager",
  "runtime_type": "claude-code",
  "model": "claude-sonnet-4",
  "tools": ["github-mcp", "memory-mcp"],
  "memory_namespace": "gtf_manager"
}
```

2. Runtime Manager：
   - 检查是否有可用的Claude Code实例（若负载已满则启动新容器）
   - 启动Claude Code进程，传入初始化参数（包括Memory MCP连接信息）
   - 配置Pre-Session和Post-Session钩子（用于记忆注入和回写）
   - 注册Agent ID与Runtime实例的映射
   - 返回 `{ status: "deployed", endpoint: "http://cc-node-01:3000" }`

3. 之后所有发往该Agent的任务消息，都由Runtime Manager转发到正确的Runtime

### 6.4.3 Runtime健康检查

Runtime Manager每30秒对所有外部Runtime进行健康检查：

- 发送 `ping` 请求
- 若连续3次无响应 → 标记为 `unhealthy`
- 若连续10次无响应 → 标记为 `offline`，通知EA
- EA收到离线通知后，将该Agent的未完成任务转移给其他Agent
- Runtime恢复后自动重新上线

## 6.5 对用户（CEO）的透明性

CEO在前端看到的是统一的Agent列表，不显示Runtime信息（详细信息可在Agent详情页查看，但日常操作中不需要关心）。

**前端展示**：
- Agent列表：显示名称、角色、状态、部门，不显示Runtime
- Agent详情页：基本信息中有“运行环境”字段（如“Claude Code CLI”），可展开查看详情
- 任务分派：CEO只需描述任务，EA自动选择合适的Agent，基座选择对CEO完全透明

**CEO体验**：
- CEO说“帮我开发一个iOS App” → EA分析需求 → 发现iOS开发Agent部署在Claude Code上 → 自动路由 → CEO看到任务开始执行 → 完成后收到结果
- 整个过程CEO不需要知道iOS开发Agent运行在哪个基座上

## 6.6 扩展新的Runtime

接入新的执行环境（如未来的Gemini CLI、新的开源Agent框架）的步骤：

1. **实现IRuntime接口**：封装为Docker镜像或可执行程序
2. **注册到Runtime Manager**：添加配置
```json
{
  "runtime_type": "gemini-cli",
  "image": "registry.example.com/gemini-cli-runtime:1.0",
  "endpoint_template": "http://{host}:{port}",
  "capabilities": ["code_execution", "file_system"],
  "default_model": "gemini-2.5-pro",
  "memory_support": true,
  "max_concurrent_tasks": 5
}
```
3. **验证**：Runtime Manager测试连接
4. **可用**：HR Manager的基座选择列表中自动出现新选项

---

以上是第六章的详细内容。这一章解决了“Agent部署在哪里”和“记忆如何统一”两个核心问题。通过Runtime抽象层，系统可以灵活利用不同执行环境的优势；通过统一Memory MCP Server，Agent无论跑在哪里都拥有连续的长期记忆。

# 第七章：大项目管理与会议机制

## 7.1 设计目标

在实际工作中，很多任务不是“一句话就能完成”的——一个完整的软件开发项目可能持续数周，涉及需求分析、设计、开发、测试、部署等多个阶段，需要多个Agent协同工作。本章定义：

- **项目管理工具的集成方案**：复用成熟的OpenProject作为底层引擎
- **新建项目的完整流程**：从CEO一句话到项目结构初始化
- **项目经理Agent的职责**：任务拆解、进度跟踪、风险识别
- **多Agent会议机制**：如何发起、主持、记录和分发会议决策

## 7.2 项目管理工具选型与集成

### 7.2.1 为什么选择OpenProject

在多种开源项目管理工具中（Taiga、Plane、Wekan等），OpenProject是最适合本系统的选择：

| 对比维度 | OpenProject | Plane | Taiga | Wekan |
|---------|-------------|-------|-------|-------|
| 甘特图 | ✅ 内置 | ❌ | ✅ | ❌ |
| 敏捷看板 | ✅ | ✅ | ✅ | ✅ |
| 工作包层级 | ✅ 无限层级 | ❌ | ✅ | ❌ |
| REST API | ✅ 完善 | ✅ | ✅ | ✅ |
| 自定义字段 | ✅ | ✅ | ❌ | ❌ |
| 时间跟踪 | ✅ | ✅ | ❌ | ❌ |
| Wiki集成 | ✅ | ✅(Docs) | ✅ | ❌ |
| 开源协议 | GPLv3 | Apache 2.0 | MPL 2.0 | MIT |
| Docker部署 | ✅ | ✅ | ✅ | ✅ |

OpenProject的核心优势是**工作包层级（Work Package Hierarchy）**——项目可以拆解为阶段→任务包→子任务，与我们的Agent任务拆解天然匹配。

### 7.2.2 集成架构

```
PM Agent
    │
    │ MCP协议
    ▼
OpenProject MCP Server
    │
    │ REST API ( /api/v3 )
    ▼
OpenProject Server
    ├── Projects (项目)
    ├── Work Packages (工作包/任务)
    ├── Boards (看板)
    ├── Time Entries (工时记录)
    └── Wiki (项目文档)
```

**OpenProject MCP Server** 封装了常用的API操作：

| MCP工具名 | 对应API | 用途 |
|-----------|---------|------|
| `op.list_projects` | GET /api/v3/projects | 获取项目列表 |
| `op.create_project` | POST /api/v3/projects | 创建新项目 |
| `op.list_work_packages` | GET /api/v3/work_packages | 查询工作包（任务） |
| `op.create_work_package` | POST /api/v3/work_packages | 创建任务 |
| `op.update_work_package` | PATCH /api/v3/work_packages/:id | 更新任务状态、分配人 |
| `op.get_kanban` | GET /api/v3/boards | 获取看板视图 |
| `op.create_wiki_page` | POST /api/v3/wiki_pages | 创建Wiki页面 |
| `op.log_time` | POST /api/v3/time_entries | 记录工时 |

### 7.2.3 代码与文档仓库集成

**代码仓库（GitLab/GitHub/Gitea）**：
通过对应的MCP Server接入，PM Agent和研发Agent可以：
- 查看代码文件和目录结构
- 创建/查看合并请求（Merge Request）
- 获取提交历史
- 在MR上评论（代码审查）

**文档仓库（Wiki.js/Outline）**：
通过MCP Server接入，Agent可以：
- 创建项目文档页面
- 搜索已有文档
- 更新文档内容
- 会议纪要自动归档

## 7.3 新建项目全流程

### 7.3.1 CEO触发

CEO在前端项目管理页点击「+ 新建项目」，填写表单：

```
┌─────────────────────────────────────────────────────────┐
│  新建项目                                                 │
│                                                         │
│  项目名称:  [Q3财报分析系统 v2.0          ]              │
│                                                         │
│  项目描述:  [重构Q3财报分析系统，支持子公司数据]           │
│                                                         │
│  项目管理工具:                                           │
│    ● OpenProject (默认)                                 │
│                                                         │
│  关联代码仓库 (可选):                                    │
│    □ GitLab  [https://gitlab.com/...    ]               │
│    □ GitHub  [                          ]               │
│                                                         │
│  关联文档仓库 (可选):                                    │
│    □ Wiki.js [https://wiki.example.com/..]              │
│                                                         │
│  项目经理:                                               │
│    ● 自动创建新 PM Agent                                │
│    ○ 指派现有 Agent: [___选择___]                       │
│                                                         │
│  初始参与部门:                                           │
│    [✓] 研发部  [✓] 数据分析部  [ ] 设计部               │
│                                                         │
│  [取消]                              [创建项目]          │
└─────────────────────────────────────────────────────────┘
```

### 7.3.2 后端自动处理流程

CEO点击「创建项目」后，Gateway依次执行：

**第一步：在OpenProject中创建项目**
```
Gateway → OpenProject MCP Server → OpenProject API:
POST /api/v3/projects
{
  "name": "Q3财报分析系统 v2.0",
  "description": "重构Q3财报分析系统，支持子公司数据",
  "status": "active"
}

Response: { "id": 42, "_links": { ... } }
```

**第二步：创建项目经理Agent**

若CEO选择“自动创建”，EA通知HR Manager从模板库实例化PM Agent：

```
HR Manager → Gateway:
agents.create({
  agent_id: "pm_q3_finance",
  name: "Q3财报分析系统 PM",
  role: "项目经理",
  template: "pm",
  runtime: "openclaw-native",
  model: "gpt-5",
  tools: ["op-mcp", "gitlab-mcp", "wikijs-mcp"],
  metadata: {
    project_id: "proj-q3-finance",
    op_project_id: 42,
    git_repos: ["https://gitlab.com/finance/q3-report"],
    wiki_url: "https://wiki.example.com/finance",
    initial_departments: ["rd", "da"]
  }
})
```

**第三步：注入项目上下文**

PM Agent启动后，其系统提示词自动注入项目环境变量：

```
# 项目环境变量（注入到PM的Soul中）
OP_PROJECT_ID=42
GIT_REPO_URL=https://gitlab.com/finance/q3-report
WIKI_URL=https://wiki.example.com/finance
PROJECT_NAME=Q3财报分析系统 v2.0
PARTICIPATING_DEPARTMENTS=rd,da
```

**第四步：初始化项目结构**

PM Agent自动执行初始化：

1. **创建WBS（工作分解结构）**：
```
PM → op.create_work_package({
  project_id: 42,
  subject: "需求分析",
  type: "Phase",
  children: [
    { subject: "调研Q3数据源", type: "Task", assignee: "da_01" },
    { subject: "确认子公司数据格式", type: "Task", assignee: "da_01" },
    { subject: "输出需求文档", type: "Task", assignee: "pm_q3_finance" }
  ]
})

PM → op.create_work_package({
  project_id: 42,
  subject: "系统设计",
  type: "Phase",
  children: [
    { subject: "API架构设计", type: "Task", assignee: "rd_manager" },
    { subject: "数据库Schema设计", type: "Task", assignee: "rd_manager" }
  ]
})

PM → op.create_work_package({
  project_id: 42,
  subject: "开发实现",
  type: "Phase"
})

PM → op.create_work_package({
  project_id: 42,
  subject: "测试与部署",
  type: "Phase"
})
```

2. **创建Wiki首页**：
```
PM → wiki.create_page({
  space: "finance",
  title: "Q3财报分析系统 v2.0 - 项目首页",
  content: "# Q3财报分析系统 v2.0\n\n## 项目概述\n...\n## 参与部门\n- 研发部\n- 数据分析部\n\n## 会议记录\n（待更新）\n\n## 技术文档\n（待更新）"
})
```

3. **通知CEO**：
```
PM → EA → CEO:
"项目「Q3财报分析系统 v2.0」已创建完成。
 - OpenProject: https://openproject.example.com/projects/42
 - Wiki首页: https://wiki.example.com/finance/q3
 - 参与部门: 研发部、数据分析部
 - 初始任务已分配，预计3个工作日内完成需求分析阶段。"
```

### 7.3.3 后续任务执行

PM Agent 持续监控项目进度：

- 每日检查任务完成情况
- 若任务阻塞超过24小时，自动发起会议
- 若某部门负载过重，向EA申请临时抽调其他部门Agent
- 定期（每周）生成项目周报，推送CEO

## 7.4 项目经理Agent设计

### 7.4.1 定位

PM Agent 是一个长期运行的项目协调者，不执行具体的技术任务，而是：
- 把大项目拆解为可执行的任务
- 将任务分派给合适的部门或Agent
- 跟踪进度，识别风险
- 在必要时发起会议讨论

### 7.4.2 系统提示词（Soul）摘要

```
你是项目「{PROJECT_NAME}」的项目经理。你的职责是推进项目按时交付。

## 核心职责
1. 任务拆解：将项目需求拆解为阶段→任务，创建到OpenProject
2. 任务分派：根据能力需求将任务分派给合适的部门
3. 进度跟踪：每日检查任务状态，更新进度百分比
4. 风险管理：识别阻塞项，24小时内解决或升级
5. 会议组织：涉及跨部门决策或需求不明确时，发起会议
6. 汇报：每周生成项目周报，里程碑达成时主动汇报

## 工具使用
- OpenProject MCP：创建/更新/查询工作包
- GitLab MCP：查看代码提交、MR状态
- Wiki MCP：更新项目文档
- meetings.create：发起会议
```

## 7.5 多Agent会议机制

### 7.5.1 会议的生命周期

```
[发起] → [审批] → [进行中] → [结束] → [纪要分发]
```

### 7.5.2 发起会议

任何Agent（通常是PM）在需要跨部门协调或CEO决策时，可向EA发起会议请求：

```json
{
  "type": "MEETING_REQUEST",
  "sender": "pm_q3_finance",
  "receiver": "ea",
  "payload": {
    "subject": "Q3数据源确认会议",
    "description": "需要确认是否包含子公司数据，这会影响API设计",
    "proposed_participants": [
      { "agent_id": "rd_manager", "required": true, "reason": "API设计依赖" },
      { "agent_id": "da_01", "required": true, "reason": "数据源负责人" },
      { "agent_id": "ceo", "required": true, "reason": "需要决策是否包含子公司数据" }
    ],
    "project_id": "proj-q3-finance",
    "urgency": "high",
    "suggested_duration_minutes": 30
  }
}
```

### 7.5.3 EA审批与调度

EA收到会议请求后：

1. 检查必选参与者的空闲状态
2. 若CEO需要参会 → 生成审批卡片推送CEO（L3级）
3. 若CEO不需要 → 判断是否需要经理审批 → 若需（L2），转经理；否则EA自行安排
4. CEO/经理批准后：
   - 创建会议会话
   - 向所有参会者发送 `MEETING_INVITATION`
   - 推送到CEO的会议中心

### 7.5.4 会议进行

会议在WebChat中进行，EA担任主持人：

**会议对话格式**：
```
[14:30] EA(主持人): 本次会议议题：Q3数据源确认。
        请数据分析师陈述当前数据源情况。

[14:32] DA_01: 当前数据源包含总部Q3数据，共15个字段。
        子公司数据需要另外对接，预计增加3个API。

[14:35] RD_Mgr: 如果增加子公司数据，API架构需要从单源改为多源聚合，
        开发工作量增加约40%。建议CEO确认是否必要。

[14:37] CEO: 确认包含子公司数据。研发部会后给出新工期评估。
        数据分析师负责对接子公司数据API。

[14:38] EA(主持人): 决议已记录。现在分配Action Items:
        - [ ] RD_Mgr: 更新API设计，评估新工期（截止：明日18:00）
        - [ ] DA_01: 对接子公司数据API（截止：后日18:00）
        - [ ] PM: 更新项目计划（截止：明日12:00）
        若无其他议题，会议结束。
```

**EA的会议控制职责**：
- 按议题顺序引导发言
- 防止Agent偏离主题（自动检测并提醒）
- 控制时间（每个议题不超过预定时间）
- CEO可随时插话、调整方向
- 记录所有发言和决策

### 7.5.5 会议结束与纪要分发

CEO点击「结束会议」或EA判断讨论完成后：

**EA生成结构化会议纪要**：
```json
{
  "meeting_id": "meeting-q3-001",
  "subject": "Q3数据源确认会议",
  "datetime": "2026-06-15T14:30:00Z",
  "duration_minutes": 8,
  "participants": ["ea", "rd_manager", "da_01", "ceo"],
  "project_id": "proj-q3-finance",
  "summary": "确认Q3财报分析系统需要包含子公司数据...",
  "decisions": [
    "确认包含子公司数据源",
    "API架构需从单源改为多源聚合"
  ],
  "action_items": [
    {
      "assignee": "rd_manager",
      "task": "更新API设计，评估新工期",
      "deadline": "2026-06-16T18:00:00Z",
      "priority": "high"
    },
    {
      "assignee": "da_01",
      "task": "对接子公司数据API",
      "deadline": "2026-06-17T18:00:00Z",
      "priority": "high"
    },
    {
      "assignee": "pm_q3_finance",
      "task": "更新项目计划",
      "deadline": "2026-06-16T12:00:00Z",
      "priority": "medium"
    }
  ],
  "next_meeting": null
}
```

**自动分发**：
- 纪要存入Wiki项目页面
- Action Items自动在OpenProject中创建为任务
- 所有参会者收到纪要副本
- CEO在会议中心可随时查看历史会议

### 7.5.6 会议类型

| 类型 | 触发条件 | 必选参会者 | 频率 |
|------|---------|-----------|------|
| **日常站会** | PM每日定时触发 | PM + 各部门代表 | 每日1次（可配置） |
| **需求对齐** | 需求不明确，影响开发 | PM + 相关部门 + 可选CEO | 按需 |
| **技术评审** | 跨部门技术方案讨论 | 研发部 + 相关部门 | 按需 |
| **阻塞解决** | 任务阻塞超过24小时 | PM + 阻塞方 + 被阻塞方 | 按需 |
| **里程碑评审** | 阶段完成 | PM + 所有参与部门 + CEO | 每阶段 |
| **临时讨论** | 任意Agent发起 | 发起人指定 | 按需 |

## 7.6 项目中的自进化触发

项目推进过程中，自进化机制也在持续运行：

| 触发场景 | 自进化动作 |
|---------|-----------|
| PM发现技能缺口 | 向HR发送 `TALENT_REQUEST` 招聘新Agent |
| 某Agent在项目中表现优异 | HR在项目结束后评估，可提前转正 |
| 某Agent多次任务失败 | PM反馈给HR，触发SOUL优化或淘汰 |
| 项目发现新的最佳实践 | PM将经验存入Wiki，相关Agent可检索学习 |
| 多个项目出现相同技能需求 | HR分析后可能建议新建部门 |

---

以上是第七章的详细内容。这一章定义了“怎么做大项目”——通过集成OpenProject实现项目管理的工程化，通过会议机制实现多Agent的结构化协作，PM Agent作为项目中枢协调一切。

# 第八章：模型中转站与独立模型配置

## 8.1 设计目标

在多Agent系统中，不同岗位的Agent对模型能力的需求差异很大：
- 代码开发类Agent需要强代码生成能力（Claude Sonnet 4、DeepSeek-Coder）
- 创意设计类Agent需要强创意生成能力（Claude Opus）
- 通用协调类Agent需要长上下文和强推理（GPT-5）
- 数据分析类Agent需要强计算和结构化输出（GPT-5、DeepSeek-V3）

如果所有Agent共用一个模型，就会出现“用跑车拉货”或“用货车比赛”的错配。同时，单一模型供应商存在宕机风险，需要降级链保障可用性。成本控制也是多Agent系统的核心挑战——多个Agent并行调用模型，API费用可能迅速膨胀。

本章定义：
- **模型中转站的选型与架构**：统一管理多家LLM提供商
- **Agent独立模型配置**：每个Agent可独立选择模型和降级链
- **成本控制与预算管理**：全局和单Agent维度的成本监控
- **降级与故障转移策略**：模型不可用时的自动切换

## 8.2 模型网关选型

### 8.2.1 为什么需要模型中转站

直接让每个Agent对接不同的模型API会带来以下问题：
- **API Key管理混乱**：每个Agent需要独立配置多家提供商的Key
- **无法统一监控**：调用量、成本、延迟分散在各处
- **降级链难以实现**：Agent需要自己处理模型切换逻辑
- **成本无法控制**：没有统一的预算限制和预警

模型中转站作为统一网关，解决上述所有问题。

### 8.2.2 方案对比

| 特性 | One API | LiteLLM | UniRoute | FluxRelay |
|------|---------|---------|----------|-----------|
| 开源协议 | MIT | MIT | Apache 2.0 | MIT |
| 社区活跃度 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| 支持模型数 | 100+ | 100+ | 50+ | 30+ |
| 负载均衡 | ✅ 权重轮询 | ✅ 多种策略 | ✅ | ✅ 自动轮换 |
| 降级链 | ✅ | ✅ | ✅ | ✅ |
| 成本追踪 | ✅ 基础 | ✅ 精细（按Token） | ✅ | ❌ |
| 速率限制 | ✅ 按用户/按Key | ✅ 按用户/按模型 | ✅ 企业级 | ❌ |
| 多租户 | ✅ | ✅ | ✅ | ✅ |
| 管理UI | ✅ 完善 | ✅ 完善 | ✅ | ❌ |
| Docker部署 | ✅ | ✅ | ✅ | ✅ |
| 国内模型支持 | ✅ 好（国产模型优先） | ✅ 好 | ⚠️ 一般 | ⚠️ 一般 |

### 8.2.3 推荐方案

**主推：One API**
- 国产开源，社区活跃，文档齐全
- 对国内模型（DeepSeek、Qwen、MiniMax等）支持最好
- 内置管理UI，配置直观
- 支持按用户组设置额度，与我们的Agent模型分配天然契合

**备选：LiteLLM**
- 如果需要更精细的成本追踪（精确到每次调用的Token费用）
- 如果需要与LangChain/LlamaIndex等框架深度集成
- 社区更国际化

本系统选择 **One API** 作为默认方案，同时在Runtime抽象层中保持与LiteLLM的兼容性（两者都提供OpenAI兼容接口）。

## 8.3 架构与部署

### 8.3.1 模型中转站的位置

```
Agent (任何Runtime)
    │
    │ OpenAI兼容API调用
    ▼
模型中转站 (One API)
    │
    ├──→ OpenAI (GPT-5, GPT-4o...)
    ├──→ Anthropic (Claude Opus, Claude Sonnet...)
    ├──→ DeepSeek (V3, Coder...)
    ├──→ Qwen (Plus, Max...)
    └──→ ... 更多提供商
```

Agent不直接知道自己在调用哪个模型——它只向模型中转站发请求，由中转站根据配置路由到实际模型。

### 8.3.2 部署方式

**Docker Compose部署（推荐）**：
```
one-api:
  image: justsong/one-api:latest
  ports:
    - "3000:3000"
  volumes:
    - ./one-api/data:/data
  environment:
    - SQL_DSN=postgres://oneapi:password@postgres:5432/oneapi
    - LOG_SQL_DSN=false
    - SESSION_SECRET=random-secret-string
  restart: always
```

**关键配置**：
1. 添加模型渠道（Channel）：为每个LLM提供商配置API Key和Base URL
2. 创建模型映射：将内部模型名（如 `claude-sonnet-4`）映射到实际渠道
3. 创建用户组（对应每个Agent或部门）：设置额度限制和速率限制
4. 生成用户Token：每个Agent获得一个独立的API Token

## 8.4 Agent独立模型配置

### 8.4.1 配置存储

每个Agent的模型配置存储在其Agent定义中：

```json
{
  "agent_id": "ios_dev_01",
  "model_config": {
    "primary": "claude-sonnet-4",
    "fallbacks": ["deepseek-coder", "qwen-plus"],
    "parameters": {
      "temperature": 0.3,
      "max_tokens": 8192,
      "top_p": 0.95
    },
    "rate_limit": {
      "max_requests_per_minute": 30,
      "max_tokens_per_day": 500000
    },
    "monthly_budget_usd": 50
  }
}
```

### 8.4.2 配置流程

**HR创建Agent时自动配置**：
1. HR分析岗位需求，确定推荐模型
2. 在模型中转站中为该Agent创建独立的用户/Token
3. 设置初始额度（实习Agent额度较低，转正后提升）
4. 将模型配置写入Agent定义

**CEO手动调整**：
1. 在Agent详情页的「模型配置」面板中
2. 下拉选择新模型 → 即时生效
3. 查看该Agent近7天/30天的调用统计和花费
4. 手动调整月度预算

### 8.4.3 模型选择推荐矩阵

HR Manager内嵌的推荐逻辑：

| 岗位类型 | 主模型 | 备选模型 | Temperature | 理由 |
|---------|--------|---------|-------------|------|
| 代码开发 | claude-sonnet-4 | deepseek-coder | 0.1-0.3 | 代码需要确定性输出 |
| 代码审查 | claude-sonnet-4 | gpt-5 | 0.1-0.2 | 审查需要精确 |
| 数据分析 | gpt-5 | deepseek-v3 | 0.2-0.5 | 结构化输出能力强 |
| 创意设计 | claude-opus | gpt-5 | 0.7-1.0 | 创意需要多样性 |
| 文案撰写 | claude-opus | gpt-5 | 0.6-0.9 | 文本质量优先 |
| 通用协调 | gpt-5 | claude-opus | 0.3-0.7 | 长上下文推理 |
| 安全审计 | claude-sonnet-4 | gpt-5 | 0.1-0.2 | 精确分析 |
| 项目管理 | gpt-5 | claude-opus | 0.3-0.5 | 综合能力 |

### 8.4.4 CEO操作界面

Agent详情页 → 模型配置Tab：

```
┌─────────────────────────────────────────────────────────┐
│  模型配置                                                │
│                                                         │
│  当前模型:  [claude-sonnet-4        ▼]                  │
│                                                         │
│  降级链:                                                │
│  ┌─────────────────────────────────────────────────┐   │
│  │ 1. claude-sonnet-4    ────────  🟢 正常          │   │
│  │ 2. deepseek-coder     ────────  ⚪ 待命          │   │
│  │ 3. qwen-plus          ────────  ⚪ 待命          │   │
│  └─────────────────────────────────────────────────┘   │
│  [编辑降级链]                                           │
│                                                         │
│  参数配置:                                              │
│  Temperature: [0.3        ]  (0-2.0)                   │
│  Max Tokens:  [8192       ]                            │
│                                                         │
│  近7天调用统计:                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │ 总调用: 1,247次  │  成功率: 99.7%               │   │
│  │ 平均延迟: 1.2s   │  总花费: ¥23.50             │   │
│  │ 降级次数: 0      │  预算剩余: ¥176.50 (78%)    │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  [查看详细统计]  [调整预算]  [立即生效]                   │
└─────────────────────────────────────────────────────────┘
```

## 8.5 成本控制

### 8.5.1 多层预算体系

```
全局预算（总部月度总预算）
    │
    ├── 部门预算（研发部、数据分析部...）
    │       │
    │       └── Agent预算（每个Agent的月度额度）
    │
    └── 项目预算（大项目的独立预算）
            │
            └── Agent预算（参与该项目的Agent）
```

### 8.5.2 预算告警与限流

| 触发条件 | 级别 | 动作 |
|---------|------|------|
| 单Agent用量达50% | Info | 通知Agent本人 |
| 单Agent用量达80% | Warn | 通知Agent + 部门经理 |
| 单Agent用量达100% | Critical | 暂停该Agent模型调用，通知CEO |
| 部门用量达80% | Warn | 通知部门经理 + EA |
| 全局用量达80% | Warn | 通知CEO |
| 全局用量达100% | Critical | 暂停所有非关键Agent调用，仅EA可用 |

### 8.5.3 成本优化策略

**自动选择性价比最优模型**：
- 对于非紧急任务，模型中转站可自动选择成本更低的模型
- 例如：凌晨3点的批量数据处理任务，自动降级到DeepSeek-V3（成本为GPT-5的1/5）

**任务优先级与模型匹配**：
- 高优先级（CEO直接下达、客户交付物）→ 使用最佳模型
- 中优先级（日常开发、文档撰写）→ 使用标准模型
- 低优先级（批量处理、内部工具）→ 使用经济模型

**闲置Agent的模型降级**：
- 若Agent连续24小时无任务，自动切换到经济模型
- 收到新任务时自动恢复

## 8.6 降级与故障转移

### 8.6.1 降级触发条件

| 错误类型 | HTTP状态码 | 处理方式 |
|---------|-----------|---------|
| 速率限制 | 429 | 等待2秒 → 重试3次 → 降级 |
| 服务不可用 | 503 | 等待1秒 → 重试3次 → 降级 |
| 超时 | 无响应（30秒） | 直接降级 |
| 认证失败 | 401 | 不降级，暂停Agent，通知管理员 |
| 额度耗尽 | 402/429 | 直接降级，通知HR |

### 8.6.2 降级链策略

**顺序降级（Sequential）**：
```
claude-sonnet-4 → deepseek-coder → qwen-plus → (告警通知CEO)
```

每个模型重试3次后切换到下一个。若全部失败：
- 关键任务：暂停，通知EA/CEO决策
- 非关键任务：标记失败，记录审计

**智能降级（Smart Fallback）**：
- 根据任务类型选择最优降级目标
- 代码任务 → 优先降级到其他代码模型
- 创意任务 → 优先降级到其他创意模型
- 避免“用代码模型写诗”的情况

### 8.6.3 降级监控

- 每次降级事件记录到审计日志
- 同一Agent 1小时内降级超过5次 → 触发Warn告警
- 同一模型 1小时内被降级超过20次 → 触发Critical告警（可能该模型大规模故障）

## 8.7 模型中转站管理前端

### 8.7.1 模型提供商管理页

路径：`/settings/models`

```
┌─────────────────────────────────────────────────────────┐
│  系统设置 → 模型中转站                                    │
│                                                         │
│  模型提供商                                              │
│  ┌─────────────────────────────────────────────────┐   │
│  │ 🟢 OpenAI                                        │   │
│  │    模型: GPT-5, GPT-4o                           │   │
│  │    API Key: sk-****a1b2  │  额度: ¥450          │   │
│  │    [编辑] [暂停] [查看日志]                       │   │
│  ├─────────────────────────────────────────────────┤   │
│  │ 🟢 Anthropic                                     │   │
│  │    模型: Claude Opus, Claude Sonnet 4            │   │
│  │    API Key: sk-ant-****c3d4  │  额度: $120      │   │
│  ├─────────────────────────────────────────────────┤   │
│  │ 🟡 DeepSeek                                      │   │
│  │    ⚠ 今日调用量已达80%限额                       │   │
│  ├─────────────────────────────────────────────────┤   │
│  │ ➕ 添加新提供商                                   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  全局预算:                                               │
│  月度总预算: ¥1000  │  本月已用: ¥320 (32%)             │
│  ┌─────────────────────────────────────────────────┐   │
│  │ ████████░░░░░░░░░░░░░░░░░░░░ 32%               │   │
│  └─────────────────────────────────────────────────┘   │
│  告警线: 80%  │  到达告警线时: [发送通知 ▼]            │
└─────────────────────────────────────────────────────────┘
```

### 8.7.2 Agent模型分配视图

展示所有Agent的模型分配情况，支持批量调整：

```
┌─────────────────────────────────────────────────────────┐
│  Agent模型分配                                           │
│                                                         │
│  Agent             │ 当前模型          │ 近7天花费       │
│  ──────────────────┼───────────────────┼────────────────│
│  EA (总裁助理)      │ gpt-5            │ ¥12.30         │
│  HR Manager       │ gpt-5            │ ¥8.50          │
│  GTF Manager      │ claude-sonnet-4  │ ¥45.20         │
│  iOS开发(实习)     │ claude-sonnet-4  │ ¥3.10          │
│  研发部经理         │ claude-sonnet-4  │ ¥28.90         │
│  ...              │ ...              │ ...            │
│                                                         │
│  [批量切换模型]  [导出报表]                               │
└─────────────────────────────────────────────────────────┘
```

---

以上是第八章的详细内容。这一章解决了“每个Agent用什么模型”、“模型出问题怎么办”、“成本怎么控制”三个核心问题。通过模型中转站，系统可以灵活调度多家LLM提供商，Agent拥有独立的模型配置和降级链，全局预算告警防止费用失控。

# 第九章：前端页面详细设计

## 9.1 设计目标

前端是整个系统与CEO交互的唯一窗口。它需要做到：
- **一眼看清全局**：Dashboard 展示组织健康度、任务进度、告警信息
- **操作即所得**：模型切换、审批决策、会议发言等操作即时生效
- **组织可视化**：公司架构、Agent状态以拓扑图直观呈现
- **项目可追踪**：任务看板、会议记录、代码仓库联动展示
- **响应式与可扩展**：支持中英双语，页面模块按需加载

## 9.2 技术栈

| 层级 | 技术选择 | 版本 | 选型理由 |
|------|---------|------|---------|
| 框架 | React | ≥ 18.3 | 生态成熟，社区活跃，Hooks模式适合复杂交互 |
| 语言 | TypeScript | ≥ 5.4 | 类型安全，减少运行时错误 |
| 构建 | Vite | ≥ 5.4 | 极速冷启动，HMR热更新 |
| 样式 | Tailwind CSS | ≥ 3.4 | 原子化CSS，开发效率高 |
| 组件库 | Shadcn/ui | latest | 高质量、可定制、无障碍支持好 |
| 状态管理 | Zustand | ≥ 5.0 | 轻量无boilerplate，与WebSocket事件天然契合 |
| 服务端缓存 | TanStack Query | ≥ 5.0 | 自动缓存、去重、重新获取 |
| 拓扑可视化 | React Flow | ≥ 12.0 | 节点拖拽、连线编排 |
| 图表 | ECharts | ≥ 5.5 | 丰富的图表类型，大数据渲染性能好 |
| 路由 | React Router | ≥ 6.0 | 嵌套路由、布局路由、懒加载 |
| 国际化 | i18next + react-i18next | latest | 中英双语支持，按模块拆分语言文件 |
| 实时通信 | 原生WebSocket | — | 对接OpenClaw Gateway的JSON-RPC协议 |

## 9.3 页面路由结构

```
/                           → 重定向到 /dashboard
/dashboard                  → CEO控制台（首页）
/org                        → 组织架构与Agent管理
/org/:agentId               → Agent详情页
/branches                   → 分公司管理
/branches/:branchId         → 分公司详情
/projects                   → 项目管理
/projects/:projectId        → 项目详情
/meetings                   → 会议中心
/meetings/:meetingId        → 会议详情（实时对话）
/approvals                  → 审批中心
/settings                   → 系统设置
/settings/models            → 模型中转站配置
/settings/templates         → Agent模板库
```

## 9.4 全局Layout

所有页面共享的布局结构：

```
┌──────────────────────────────────────────────────────────────┐
│  [Logo] 自进化Agent组织            🔔 3条待审批   🌐 中文▼  👤 CEO │ ← TopBar
├──────────┬───────────────────────────────────────────────────┤
│          │                                                   │
│ 导航菜单  │                 主内容区                           │
│          │                                                   │
│ 📊 控制台 │         (根据路由渲染对应页面组件)                  │
│ 👥 组织   │                                                   │
│ 🏢 分公司 │                                                   │
│ 📋 项目   │                                                   │
│ 💬 会议   │                                                   │
│ ✅ 审批   │                                                   │
│ ⚙️ 设置   │                                                   │
│          │                                                   │
└──────────┴───────────────────────────────────────────────────┘
```

**TopBar组件**：
- 左侧：Logo + 系统名称
- 右侧：通知中心（待审批数徽章）、语言切换（中文/English）、CEO头像

**Sidebar组件**：
- 导航菜单项，当前页面高亮
- 底部显示系统版本和WebSocket连接状态指示灯（🟢已连接 / 🟡重连中 / 🔴已断开）

## 9.5 核心页面详细设计

### 9.5.1 CEO控制台 (Dashboard)

**路由**：`/dashboard`

**功能定位**：CEO的“驾驶舱”，一屏掌握整个Agent组织的运行状态。

**页面布局**：

```
┌──────────────────────────────────────────────────────────────┐
│  CEO控制台                                                    │
│                                                              │
│  ┌──────────┬──────────┬──────────┬──────────┐              │
│  │ 在线Agent │ 进行中任务│ 待审批事项 │ 本月花费  │              │
│  │   12/15   │    8     │    3     │  ¥320    │              │
│  │ 🟢 正常   │ ← 稳定   │ ⚠ 待处理 │  32%预算  │              │
│  └──────────┴──────────┴──────────┴──────────┘              │
│                                                              │
│  ┌────────────────────────────┐  ┌────────────────────────┐ │
│  │  实时活动流                 │  │  组织健康度             │ │
│  │                            │  │                        │ │
│  │  [GTF_Mgr] 执行中 67%      │  │  机动部负载: ████░     │ │
│  │  "数据分析报表生成"         │  │  人事部负载: ██░░░     │ │
│  │  3分钟前                   │  │  研发部负载: ███░░     │ │
│  │                            │  │                        │ │
│  │  [HR_Mgr] 正在招聘          │  │  实习生: 2名            │ │
│  │  "SwiftUI开发工程师"        │  │  招聘中: 1个岗位        │ │
│  │  15分钟前                  │  │  本周转正: 1名          │ │
│  │                            │  │                        │ │
│  │  [EA] 收到你的新消息        │  │  组织健康评分: 87/100   │ │
│  │  "帮我分析Q3财报"           │  │                        │ │
│  │  刚刚                      │  │                        │ │
│  └────────────────────────────┘  └────────────────────────┘ │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  分公司状态                                            │   │
│  │  ┌────────────┬────────────┬────────────┐            │   │
│  │  │ 总部 🟢     │ 上海GPU 🟡  │ 深圳 🔴    │            │   │
│  │  │ 负载: 45%  │ 负载: 85%  │ 离线 15分钟 │            │   │
│  │  │ 任务: 3/8  │ 任务: 4/5  │ 任务: -    │            │   │
│  │  └────────────┴────────────┴────────────┘            │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  最近告警                                              │   │
│  │  🔴 14:32  深圳分公司离线                             │   │
│  │  🟡 12:15  机动部任务连续3次超时                       │   │
│  │  🔵 10:00  月度模型用量达50%                          │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  快捷入口                                              │   │
│  │  [💬 与EA对话]  [📋 查看审批]  [👥 组织架构]  [📊 项目] │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

**关键交互**：
- 四个统计卡片每30秒自动刷新（基于WebSocket推送）
- 活动流支持无限滚动加载历史
- 每条活动可点击展开详情（跳转到对应Agent/任务）
- 分公司状态卡片颜色表示：🟢正常 🟡高负载 🔴离线
- 快捷入口一键跳转核心操作页面

### 9.5.2 组织架构页 (Org)

**路由**：`/org`

**功能定位**：可视化公司组织拓扑，管理Agent全生命周期。

**视图切换**：页面顶部Toggle：`[列表视图] [拓扑视图]`

**拓扑视图**（使用React Flow）：

```
┌──────────────────────────────────────────────────────────────┐
│  组织架构    [列表视图] [拓扑视图]    [+ 新建Agent]           │
│                                                              │
│                        ┌──────┐                              │
│                        │ CEO  │                              │
│                        │ (你) │                              │
│                        └──┬───┘                              │
│                           │                                  │
│                      ┌────▼─────┐                            │
│                      │ 总裁助理  │  🟢 在线                   │
│                      │  (EA)    │  模型: GPT-5               │
│                      └──┬───┬───┘                            │
│                         │   │                                │
│           ┌─────────────▼─┐ ┌▼─────────────┐                │
│           │  人事部经理    │ │  机动部经理   │                │
│           │  (HR_Mgr)    │ │  (GTF_Mgr)   │                │
│           │  🟢 在线      │ │  🟡 忙碌     │                │
│           │  模型: GPT-5  │ │  模型: Claude│                │
│           └──────┬────────┘ └──────────────┘                │
│                  │                                           │
│           ┌──────┴───────┐                                   │
│      ┌────▼───┐    ┌─────▼────┐                              │
│      │招聘专员 │    │ 培训专员  │  🟢 在线                    │
│      │(实习生) │    │  (空)    │  待招聘                     │
│      └────────┘    └──────────┘                              │
│                                                              │
│  图例: 🟢在线 🟡忙碌 🔵执行中 🟣实习 ⚪离线                   │
└──────────────────────────────────────────────────────────────┘
```

**拓扑交互**：
- 节点可拖拽调整位置
- 点击节点：右侧滑出详情面板（角色、状态、模型、近5个任务）
- 右键节点：快捷菜单（暂停、唤醒、强制转正、淘汰、查看详情）
- 连线表示汇报关系，悬浮显示“汇报给XX”
- 双击空白区域：新建Agent对话框

**列表视图**：
- 表格展示：Agent名称、角色、部门、状态、模型、近7天任务数、成功率
- 支持按部门筛选、按状态筛选、按技能搜索
- 点击行进入Agent详情页

### 9.5.3 Agent详情页

**路由**：`/org/:agentId`

**功能定位**：单个Agent的“人事档案”，包括模型配置、记忆查看、提示词编辑、任务历史。

**Tab结构**：
```
[基本信息] [模型配置] [记忆与知识] [提示词] [工具集] [任务历史] [绩效]
```

**Tab 1 - 基本信息**：
- Agent ID、角色名、所属部门、汇报对象
- 状态徽章（在线/忙碌/离线/实习中）
- 入职时间、当前项目
- 基座Runtime类型（OpenClaw原生 / Claude Code CLI / Codex CLI）

**Tab 2 - 模型配置**：
- 当前模型（下拉选择器，按提供商分组）
- 降级链可视化（序号列表，拖拽排序）
- 参数配置（Temperature滑块、Max Tokens输入框）
- 近7天调用统计：总次数、成功率、平均延迟、花费
- 月度预算使用进度条
- [编辑降级链] [查看详细统计] [调整预算] 按钮

**Tab 3 - 记忆与知识**：
- 语义搜索框：输入关键词，检索该Agent的长期记忆
- 搜索结果列表：每条记忆显示摘要、时间、类型标签
- 点击展开详细内容
- [清除指定记忆] 按钮（需二次确认）

**Tab 4 - 提示词**：
- 代码编辑器风格的文本框（Monaco Editor）
- 显示当前生效的完整系统提示词
- 只读模式（正式Agent），CEO可手动编辑
- 版本历史下拉：查看历史版本，支持回滚
- [保存] [沙盒测试] [回滚到选中版本] 按钮

**Tab 5 - 工具集**：
- 当前已分配的工具列表（表格：工具名、版本、风险等级、分配时间）
- [+ 分配新工具] 按钮（弹出选择器）
- [移除工具] 按钮（需确认）

**Tab 6 - 任务历史**：
- 分页表格：任务ID、描述、开始时间、完成时间、状态、结果摘要
- 点击任务行展开详情（完整输入输出）
- 支持按时间范围筛选

**Tab 7 - 绩效**（实习生特有）：
- 实习进度：已完成 X/10 任务
- 雷达图：成功率、质量、效率、协作四项得分
- 中期/终期评估报告卡片
- [提前转正] [强制淘汰] [延长实习] 按钮（仅CEO可见）

### 9.5.4 项目管理页 (Projects)

**路由**：`/projects`

**项目列表页**：

```
┌──────────────────────────────────────────────────────────────┐
│  项目管理                            [+ 新建项目]             │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 🟢 Q3财报分析系统 v2.0                                │   │
│  │    进度: ████████░░░░░░░░░░ 67%                      │   │
│  │    任务: 8/12完成  │  负责人: PM_Q3                   │   │
│  │    上次活动: 10分钟前                                │   │
│  │    [查看详情] [进入看板] [查看会议记录]                │   │
│  ├──────────────────────────────────────────────────────┤   │
│  │ 🟡 用户画像数据平台                                   │   │
│  │    进度: ████░░░░░░░░░░░░ 34%                       │   │
│  │    任务: 4/15完成  │  负责人: PM_UserProfile         │   │
│  │    ⚠ 数据采集任务阻塞，待CEO审批会议                  │   │
│  │    [查看详情] [进入看板] [查看会议记录]                │   │
│  ├──────────────────────────────────────────────────────┤   │
│  │ ✅ 内部工具链自动化                                   │   │
│  │    已完成  │  2026-05-20 交付                        │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

**项目详情页**（`/projects/:projectId`）：

**Tab 1 - 任务看板**：
- 嵌入OpenProject的看板视图（通过iframe或API渲染自定义看板）
- 列：To Do / In Progress / Review / Done
- 每张卡片显示：任务标题、负责人Agent头像、截止日期、优先级标签
- 支持拖拽改变任务状态
- 点击卡片展开详情（描述、评论、子任务、关联的Git MR）

**Tab 2 - 会议记录**：
- 该项目的所有会议列表（时间倒序）
- 每条记录：会议主题、日期、参会者头像、纪要摘要
- 点击进入会议详情页

**Tab 3 - 项目设置**：
- 项目经理Agent配置（更换PM）
- 关联的Git仓库列表（添加/删除）
- 关联的Wiki地址
- 自动化规则配置（如“任务阻塞超24小时自动发起会议”）

### 9.5.5 会议中心 (Meetings)

**路由**：`/meetings`

**会议列表页**：

```
┌──────────────────────────────────────────────────────────────┐
│  会议中心                [待审批 2] [进行中 1] [已完成 15]    │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ ⚠ 待审批会议                                          │   │
│  │                                                      │   │
│  │ 📋 数据采集方案讨论                                   │   │
│  │    发起人: PM_02  │  关联项目: 用户画像平台            │   │
│  │    需参会: 研发部经理, 数据分析师_01, CEO             │   │
│  │    发起时间: 10分钟前  │  ⚠ 等待审批                  │   │
│  │    [批准并加入会议]  [拒绝并说明理由]                  │   │
│  │                                                      │   │
│  │ 📋 Q3财报需求对齐                                     │   │
│  │    发起人: PM_Q3  │  关联项目: Q3财报分析系统          │   │
│  │    需参会: 研发部经理, CEO                            │   │
│  │    发起时间: 30分钟前  │  ⚠ 等待审批                  │   │
│  │    [批准并加入会议]  [拒绝并说明理由]                  │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 🔵 进行中会议                                          │   │
│  │                                                      │   │
│  │ 📋 Q3财报需求对齐                                     │   │
│  │    主持人: EA  │  参与: 4个Agent + CEO                │   │
│  │    开始于: 14:30  │  已进行 18分钟                    │   │
│  │    [进入会议]                                        │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

**会议详情页**（`/meetings/:meetingId`）：

```
┌──────────────────────────────────────────────────────────────┐
│  ← 返回  │  会议: Q3财报需求对齐  │  🔵 进行中               │
│  主持人: EA  │  项目: Q3财报分析系统 v2.0                    │
├──────────────────────────────────┬───────────────────────────┤
│  会议对话区（实时滚动）           │  参会列表                  │
│                                  │                           │
│  [14:30] EA(主持人):             │  👤 EA (主持人) · 🟢      │
│    本次会议议题：Q3财报需求对齐。 │                           │
│    请研发部和数据分析部陈述进展。 │  👤 RD_Mgr · 🟢          │
│                                  │                           │
│  [14:32] RD_Mgr:                 │  👤 DA_01 · 🟢           │
│    目前后端API已完成70%，        │                           │
│    主要阻塞在数据清洗管道。      │  👤 CEO(你) · 🟢         │
│                                  │                           │
│  [14:34] DA_01:                  │                           │
│    清洗规则已完成初版，          │                           │
│    但需要确认是否包含子公司数据。│                           │
│                                  │                           │
│  [14:35] CEO(你):                │                           │
│    确认包含子公司数据。          │                           │
│    数据分析师会后更新规则，      │                           │
│    研发部明天前完成对接。        │                           │
│                                  │                           │
│  [14:36] EA(主持人):             │                           │
│    决议已记录。Action Items:     │                           │
│    - DA_01: 更新清洗规则(明18:00)│                           │
│    - RD_Mgr: 对接新规则(明18:00) │                           │
│    - PM_Q3: 跟进整体进度(后10:00)│                           │
│    若无其他议题，会议结束。      │                           │
│                                  │                           │
├──────────────────────────────────┴───────────────────────────┤
│  你的输入区                                                   │
│  [输入消息...]                                    [发送]      │
│  [结束会议并生成纪要]  [暂停会议]  [邀请更多Agent]            │
└──────────────────────────────────────────────────────────────┘
```

**关键交互**：
- 新消息自动滚动到底部
- 每条消息标记发言者头像、名称、时间戳
- “结束会议并生成纪要”：EA自动生成纪要，Action Items分发
- “邀请更多Agent”：弹出组织架构选择器，选择Agent加入会议

### 9.5.6 审批中心 (Approvals)

**路由**：`/approvals`

```
┌──────────────────────────────────────────────────────────────┐
│  审批中心                    [待审批 3] [已审批 12] [已驳回 2] │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 🔴 高优先级                                           │   │
│  │                                                      │   │
│  │ 📋 新建部门申请                                       │   │
│  │    申请人: 机动部经理 (GTF_Mgr)                      │   │
│  │    申请内容: 创建「移动开发部」                       │   │
│  │    理由: 近30天iOS/Android任务占比达35%，             │   │
│  │          已有4名相关Agent，建议独立部门               │   │
│  │    时间: 2小时前                                     │   │
│  │    [✅ 批准] [❌ 驳回] [💬 要求补充说明]               │   │
│  ├──────────────────────────────────────────────────────┤   │
│  │ 🟡 中优先级                                           │   │
│  │                                                      │   │
│  │ 📋 分公司接入申请                                     │   │
│  │    申请人: 分公司总裁助理 (Branch_SZ_EA)              │   │
│  │    机器信息: 深圳-数据分析节点 (GPU-A100)             │   │
│  │    能力: GPU-A100, 64GB-RAM, CUDA12                  │   │
│  │    [✅ 批准] [❌ 驳回]                                │   │
│  ├──────────────────────────────────────────────────────┤   │
│  │ 🟢 低优先级                                           │   │
│  │                                                      │   │
│  │ 📋 工具获取申请                                       │   │
│  │    申请人: 数据分析师_01 (DA_01)                     │   │
│  │    申请工具: BigQuery MCP Server                     │   │
│  │    理由: 需要直接查询生产数据库进行Q3分析              │   │
│  │    [✅ 批准] [❌ 驳回]                                │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

**交互**：
- 点击批准：即时生效，通知EA执行
- 点击驳回：弹出文本框输入理由
- 点击“要求补充说明”：通知申请人补充信息
- 已审批列表支持按时间、类型筛选

### 9.5.7 系统设置 (Settings)

**路由**：`/settings`，子路由：`/settings/models`，`/settings/templates`

**模型中转站配置**（`/settings/models`）：见第八章8.7节设计。

**Agent模板库**（`/settings/templates`）：
- 卡片网格展示所有预置模板
- 每张卡片：模板名称、适用岗位、核心技能标签、使用次数、成功率
- 点击卡片展开详情（完整Soul、建议工具集、建议基座）
- [编辑模板] [实例化Agent] [废弃模板] 按钮

## 9.6 组件树

```
App
├── Layout
│   ├── TopBar
│   │   ├── Logo
│   │   ├── NotificationCenter (通知中心)
│   │   ├── LanguageSwitcher (语言切换)
│   │   └── UserMenu (CEO头像与下拉菜单)
│   ├── Sidebar
│   │   ├── NavMenu (导航菜单)
│   │   └── WebSocketStatus (连接状态指示灯)
│   └── MainContent (RouterOutlet)
│
├── Pages
│   ├── DashboardPage
│   │   ├── StatsCardGroup (四个统计卡片)
│   │   ├── ActivityStream (实时活动流)
│   │   ├── OrgHealthPanel (组织健康度)
│   │   ├── BranchStatusPanel (分公司状态)
│   │   ├── AlertPanel (最近告警)
│   │   └── QuickEntryButtons (快捷入口)
│   │
│   ├── OrgPage
│   │   ├── ViewToggle (列表/拓扑切换)
│   │   ├── OrgTopologyView (React Flow拓扑)
│   │   │   ├── AgentNode (Agent节点)
│   │   │   ├── DepartmentGroup (部门分组框)
│   │   │   └── ReportEdge (汇报连线)
│   │   ├── AgentListView (列表视图)
│   │   └── AgentDetailDrawer (右侧详情抽屉)
│   │
│   ├── AgentDetailPage
│   │   ├── BasicInfoPanel (基本信息)
│   │   ├── ModelConfigPanel (模型配置)
│   │   │   ├── ModelSelector (模型下拉选择器)
│   │   │   ├── FallbackChainEditor (降级链编辑)
│   │   │   └── UsageStatsChart (使用统计图表)
│   │   ├── MemoryViewer (记忆查看器)
│   │   ├── PromptEditor (提示词编辑器 - Monaco)
│   │   ├── ToolSetManager (工具集管理)
│   │   ├── TaskHistoryTable (任务历史表格)
│   │   └── PerformancePanel (绩效面板 - 实习生)
│   │       ├── ProgressBar (进度条)
│   │       └── RadarChart (雷达图)
│   │
│   ├── ProjectsPage
│   │   ├── ProjectList
│   │   └── ProjectDetail
│   │       ├── TaskKanban (任务看板)
│   │       ├── MeetingRecords (会议记录列表)
│   │       └── ProjectSettings (项目设置)
│   │
│   ├── MeetingsPage
│   │   ├── MeetingList
│   │   └── MeetingDetail
│   │       ├── ChatArea (实时对话区)
│   │       ├── ParticipantSidebar (参会人列表)
│   │       └── MeetingControls (会议控制按钮)
│   │
│   ├── ApprovalsPage
│   │   └── ApprovalCard (审批卡片)
│   │
│   ├── BranchesPage
│   │   ├── BranchList (分公司列表)
│   │   └── BranchDetail (分公司详情)
│   │
│   └── SettingsPage
│       ├── ModelsConfig (模型中转站配置)
│       ├── TemplateLibrary (模板库)
│       └── SystemSettings (系统参数)
│
└── Shared Components
    ├── AgentAvatar (Agent头像)
    ├── StatusBadge (状态徽章)
    ├── ModelSelector (模型选择器)
    ├── ApprovalCard (审批卡片)
    ├── WebSocketStatus (连接状态指示灯)
    ├── EmptyState (空状态占位)
    └── ConfirmDialog (确认对话框)
```

## 9.7 状态管理 (Zustand Store)

```typescript
interface AppStore {
  // WebSocket连接
  wsStatus: 'connecting' | 'connected' | 'disconnected';
  wsLatency: number; // 毫秒

  // Agent
  agents: Map<string, AgentState>;       // agentId → 实时状态
  agentList: AgentSummary[];             // 用于列表渲染
  selectedAgentId: string | null;

  // 会议
  activeMeetings: Meeting[];
  pendingApprovalMeetings: Meeting[];
  currentMeeting: MeetingDetail | null;

  // 审批
  pendingApprovals: Approval[];
  approvalHistory: Approval[];

  // 项目
  projects: ProjectSummary[];
  selectedProjectId: string | null;

  // 分公司
  branches: BranchInfo[];

  // 模型
  modelProviders: ModelProvider[];
  agentModelMap: Record<string, string>; // agentId → modelName

  // Actions
  connectWebSocket: () => void;
  updateAgentState: (agentId: string, state: Partial<AgentState>) => void;
  addActivityEvent: (event: ActivityEvent) => void;
  addMeeting: (meeting: Meeting) => void;
  updateMeetingMessage: (meetingId: string, message: MeetingMessage) => void;
  approveRequest: (approvalId: string, decision: boolean, reason?: string) => void;
}
```

## 9.8 前端项目目录结构

```
agent-console-frontend/
├── public/
│   └── favicon.svg
├── src/
│   ├── components/              # 共享组件
│   │   ├── ui/                  # Shadcn/ui 组件
│   │   ├── AgentAvatar.tsx
│   │   ├── StatusBadge.tsx
│   │   ├── ModelSelector.tsx
│   │   ├── ApprovalCard.tsx
│   │   ├── WebSocketStatus.tsx
│   │   ├── EmptyState.tsx
│   │   └── ConfirmDialog.tsx
│   ├── pages/                   # 页面组件
│   │   ├── DashboardPage.tsx
│   │   ├── OrgPage.tsx
│   │   ├── AgentDetailPage.tsx
│   │   ├── ProjectsPage.tsx
│   │   ├── MeetingsPage.tsx
│   │   ├── ApprovalsPage.tsx
│   │   ├── BranchesPage.tsx
│   │   └── SettingsPage.tsx
│   ├── hooks/                   # 自定义Hooks
│   │   ├── useWebSocket.ts      # WebSocket连接管理
│   │   ├── useAgents.ts         # Agent列表与实时更新
│   │   ├── useMeetings.ts       # 会议状态管理
│   │   ├── useApprovals.ts      # 审批列表
│   │   └── useModelConfig.ts    # 模型配置CRUD
│   ├── store/                   # Zustand Store
│   │   ├── index.ts             # 主Store
│   │   ├── agentSlice.ts
│   │   ├── meetingSlice.ts
│   │   ├── approvalSlice.ts
│   │   └── projectSlice.ts
│   ├── api/                     # API调用层
│   │   ├── wsClient.ts          # WebSocket客户端
│   │   ├── agents.ts            # Agent REST API
│   │   ├── models.ts            # 模型配置API
│   │   ├── projects.ts          # 项目管理API
│   │   └── meetings.ts          # 会议API
│   ├── types/                   # TypeScript类型
│   │   ├── agent.ts
│   │   ├── meeting.ts
│   │   ├── project.ts
│   │   └── ws-events.ts
│   ├── utils/                   # 工具函数
│   │   ├── formatters.ts        # 日期、数字格式化
│   │   └── validators.ts        # 表单校验
│   ├── i18n/                    # 国际化
│   │   ├── index.ts
│   │   └── locales/
│   │       ├── zh-CN/
│   │       └── en-US/
│   ├── App.tsx
│   ├── main.tsx
│   └── index.css
├── package.json
├── tailwind.config.ts
├── vite.config.ts
└── tsconfig.json
```

---

以上是第九章的完整内容。这一章覆盖了前端的所有页面布局、组件树、状态管理和项目结构，开发Agent可以直接基于这份设计开始编码。每个页面的布局示意图都给出了明确的交互说明。

# 第十章：后端接口详细设计

## 10.1 设计原则

后端接口设计遵循以下原则：

- **双通道互补**：WebSocket负责实时操作（消息、状态推送、会议对话），HTTP REST负责配置管理（CRUD、审批、查询）
- **协议标准化**：WebSocket使用OpenClaw原生JSON-RPC v3协议，HTTP使用RESTful风格
- **统一认证**：所有接口通过Bearer Token认证，Token由Gateway签发
- **标准响应格式**：`{ "code": 0, "message": "success", "data": { ... } }`
- **版本化**：API前缀统一为 `/api/v1`，后续升级可增加 `/api/v2`
- **分页标准化**：列表接口支持 `?page=1&page_size=20`，返回 `{ items, total, page, page_size }`

## 10.2 WebSocket接口（JSON-RPC v3）

### 10.2.1 连接建立

```
WebSocket Connect to: ws://{gateway-host}:18789
```

**Connect Frame（首条消息，认证）**：
```json
{
  "type": "connect",
  "auth": {
    "token": "gw-xxxx"
  },
  "clientInfo": {
    "type": "web-ui",
    "version": "1.0.0"
  }
}
```

**认证成功响应**：
```json
{
  "type": "connect_ack",
  "session_id": "sess-xxxx",
  "server_version": "1.0.0"
}
```

### 10.2.2 请求-响应模式

前端发送请求（带 `id`），Gateway返回对应结果：

```json
→ {
    "id": "req-001",
    "method": "agents.list",
    "params": { "status": "online" }
  }

← {
    "id": "req-001",
    "result": {
      "agents": [...]
    }
  }
```

**错误响应**：
```json
← {
    "id": "req-001",
    "error": {
      "code": 1003,
      "message": "权限不足"
    }
  }
```

### 10.2.3 事件推送模式

Gateway主动推送事件给前端（无 `id`）：

```json
← {
    "type": "event",
    "event": "agent.task_completed",
    "data": {
      "agent_id": "gtf_manager",
      "task_id": "task-001",
      "result": { ... }
    }
  }
```

### 10.2.4 RPC方法列表

#### Agent管理

**agents.list** — 获取所有Agent及状态
```json
→ {
    "method": "agents.list",
    "params": {
      "status": "online",
      "department": "rd",
      "skill": "Python"
    }
  }

← {
    "result": {
      "agents": [
        {
          "agent_id": "rd_manager",
          "name": "研发部经理",
          "role": "研发部经理",
          "department": "rd",
          "status": "online",
          "capabilities": { ... },
          "model": "claude-sonnet-4",
          "current_task": null
        }
      ]
    }
  }
```

**agents.get** — 获取单个Agent详情
```json
→ {
    "method": "agents.get",
    "params": { "agent_id": "rd_manager" }
  }

← {
    "result": {
      "agent_id": "rd_manager",
      "name": "研发部经理",
      "role": "研发部经理",
      "department": "rd",
      "status": "online",
      "runtime": { "type": "claude-code", "endpoint": "..." },
      "model_config": {
        "primary": "claude-sonnet-4",
        "fallbacks": ["deepseek-coder"],
        "temperature": 0.3,
        "max_tokens": 8192
      },
      "tools": ["github-mcp", "docker-mcp"],
      "capabilities": {
        "skills": ["Python", "FastAPI", "PostgreSQL"],
        "level": "senior"
      },
      "employment_status": "active",
      "created_at": "2026-05-01T00:00:00Z"
    }
  }
```

**agents.update** — 更新Agent配置
```json
→ {
    "method": "agents.update",
    "params": {
      "agent_id": "rd_manager",
      "model_config": {
        "primary": "claude-sonnet-4",
        "temperature": 0.2
      },
      "tools_add": ["k8s-mcp"],
      "tools_remove": []
    }
  }
```

#### 消息与会话

**sessions.send** — 向指定Agent发送消息
```json
→ {
    "method": "sessions.send",
    "params": {
      "agent_id": "ea",
      "message": "帮我分析Q3财报数据",
      "context": {
        "task_type": "analysis",
        "priority": "high"
      }
    }
  }

← {
    "result": {
      "message_id": "msg-001",
      "session_id": "sess-ea-20260615",
      "status": "delivered"
    }
  }
```

#### 会议管理

**meetings.create** — 创建会议
```json
→ {
    "method": "meetings.create",
    "params": {
      "subject": "Q3数据源确认",
      "participants": ["rd_manager", "da_01"],
      "require_ceo": true,
      "project_id": "proj-q3-finance"
    }
  }
```

**meetings.send_message** — 在会议中发言
```json
→ {
    "method": "meetings.send_message",
    "params": {
      "meeting_id": "meeting-q3-001",
      "content": "确认包含子公司数据"
    }
  }
```

**meetings.end** — 结束会议并生成纪要
```json
→ {
    "method": "meetings.end",
    "params": {
      "meeting_id": "meeting-q3-001"
    }
  }
```

#### 系统

**system.get_health** — 系统健康检查
```json
→ { "method": "system.get_health" }

← {
    "result": {
      "gateway": "healthy",
      "message_bus": "healthy",
      "model_gateway": "healthy",
      "agents_online": 12,
      "agents_total": 15,
      "branches_online": 2,
      "branches_total": 3
    }
  }
```

### 10.2.5 推送事件列表

| 事件名 | 触发时机 | 数据体 |
|--------|---------|--------|
| `agent.status_changed` | Agent上线/下线/忙碌 | `{ agent_id, old_status, new_status }` |
| `agent.task_started` | Agent开始执行任务 | `{ agent_id, task_id, task_summary }` |
| `agent.task_progress` | 任务进度更新 | `{ agent_id, task_id, progress_percent, message }` |
| `agent.task_completed` | 任务完成 | `{ agent_id, task_id, result_summary, duration }` |
| `agent.task_error` | 任务失败 | `{ agent_id, task_id, error_message }` |
| `meeting.message_received` | 会议新消息 | `{ meeting_id, sender, content, timestamp }` |
| `meeting.state_changed` | 会议状态变更 | `{ meeting_id, state, minutes? }` |
| `approval.created` | 新审批产生 | `{ approval_id, type, petitioner, content }` |
| `branch.status_changed` | 分公司状态变更 | `{ branch_id, old_status, new_status }` |
| `system.alert` | 系统告警 | `{ level, message, timestamp }` |
| `notification.new` | 新通知 | `{ type, title, body, action_url }` |

## 10.3 HTTP REST API

### 10.3.1 通用规范

**Base URL**: `http://{gateway-host}:18789/api/v1`

**认证**: `Authorization: Bearer <gateway_token>`

**标准响应格式**:
```json
{
  "code": 0,
  "message": "success",
  "data": { ... }
}
```

**列表响应格式**:
```json
{
  "code": 0,
  "message": "success",
  "data": {
    "items": [...],
    "total": 100,
    "page": 1,
    "page_size": 20
  }
}
```

### 10.3.2 Agent管理

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/agents` | 获取Agent列表 |
| GET | `/agents/:agentId` | 获取Agent详情 |
| PATCH | `/agents/:agentId` | 更新Agent基本信息 |
| PATCH | `/agents/:agentId/model` | 更换Agent使用的模型 |
| GET | `/agents/:agentId/memory` | 获取Agent长期记忆摘要 |
| POST | `/agents/:agentId/memory/clear` | 清除指定记忆 |
| GET | `/agents/:agentId/tasks` | 获取Agent任务历史 |
| POST | `/agents` | 手动创建Agent（触发招聘流程） |

**GET /agents**
```
查询参数:
  ?department=rd          — 按部门筛选
  &status=online          — 按状态筛选
  &skill=Python           — 按技能搜索
  &employment=intern      — 按雇佣状态（intern/active/terminated）
  &page=1&page_size=20    — 分页

响应示例:
{
  "items": [
    {
      "agent_id": "rd_manager",
      "name": "研发部经理",
      "role": "研发部经理",
      "department": "rd",
      "status": "online",
      "employment_status": "active",
      "model": "claude-sonnet-4",
      "current_task_count": 2,
      "success_rate_30d": 0.95
    }
  ],
  "total": 15,
  "page": 1,
  "page_size": 20
}
```

**PATCH /agents/:agentId/model**
```json
请求体:
{
  "model_name": "claude-sonnet-4",
  "fallbacks": ["deepseek-coder", "qwen-plus"],
  "temperature": 0.3,
  "max_tokens": 8192,
  "monthly_budget_usd": 50
}

响应:
{
  "code": 0,
  "message": "模型配置已更新",
  "data": {
    "agent_id": "rd_manager",
    "model_config": { ... }
  }
}
```

**POST /agents** — 手动创建Agent
```json
请求体:
{
  "name": "iOS开发工程师",
  "template_id": "ios_dev",
  "department": "rd",
  "runtime_type": "claude-code",
  "model": "claude-sonnet-4"
}

响应:
{
  "code": 0,
  "message": "Agent创建成功，已进入实习期",
  "data": {
    "agent_id": "ios_dev_intern_02",
    "internship_end": "2026-06-22T00:00:00Z",
    "internship_task_target": 10
  }
}
```

### 10.3.3 模型配置

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/models` | 获取模型中转站所有可用模型 |
| POST | `/models/providers` | 添加模型提供商 |
| PATCH | `/models/providers/:providerId` | 更新提供商配置 |
| DELETE | `/models/providers/:providerId` | 移除提供商 |
| GET | `/models/usage` | 获取模型用量统计 |

**GET /models**
```json
响应:
{
  "providers": [
    {
      "provider_id": "openai",
      "name": "OpenAI",
      "status": "active",
      "models": [
        { "name": "gpt-5", "status": "available" },
        { "name": "gpt-4o", "status": "available" }
      ],
      "monthly_usage_usd": 120.50,
      "monthly_budget_usd": 500
    }
  ]
}
```

**POST /models/providers**
```json
请求体:
{
  "provider_type": "openai",
  "api_key": "sk-xxxx",
  "base_url": "https://api.openai.com/v1",
  "models": ["gpt-5", "gpt-4o"],
  "monthly_budget_usd": 500
}
```

**GET /models/usage**
```
查询参数:
  ?agent_id=rd_manager     — 按Agent筛选
  &from=2026-06-01         — 开始日期
  &to=2026-06-15           — 结束日期
  &group_by=day            — 聚合粒度 (day/model/agent)

响应:
{
  "usage": [
    { "date": "2026-06-15", "calls": 247, "tokens": 1250000, "cost_usd": 3.50 },
    ...
  ],
  "total_cost_usd": 45.20
}
```

### 10.3.4 组织与审批

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/org/departments` | 获取所有部门 |
| POST | `/org/departments` | 申请新建部门（生成审批） |
| GET | `/approvals` | 获取审批列表 |
| POST | `/approvals/:id/approve` | 批准 |
| POST | `/approvals/:id/reject` | 驳回 |

**POST /org/departments**
```json
请求体:
{
  "department_name": "移动开发部",
  "description": "负责iOS和Android应用开发",
  "capabilities": ["iOS", "Android", "SwiftUI", "Kotlin"],
  "justification": "近30天移动端任务占比达35%"
}

响应:
{
  "code": 0,
  "message": "部门创建申请已提交，等待CEO审批",
  "data": {
    "approval_id": "approval-dept-001"
  }
}
```

**GET /approvals**
```
查询参数:
  ?status=pending          — 审批状态 (pending/approved/rejected)
  &type=NEW_DEPARTMENT     — 审批类型
  &page=1&page_size=20

响应:
{
  "items": [
    {
      "approval_id": "approval-dept-001",
      "type": "NEW_DEPARTMENT",
      "petitioner": "gtf_manager",
      "content": { ... },
      "priority": "high",
      "status": "pending",
      "created_at": "2026-06-15T10:00:00Z"
    }
  ],
  "total": 3
}
```

**POST /approvals/:id/approve**
```json
请求体:
{
  "comment": "同意创建移动开发部"
}
```

**POST /approvals/:id/reject**
```json
请求体:
{
  "reason": "当前移动端任务量尚未达到建部标准，建议再观察一个月"
}
```

### 10.3.5 项目管理（OpenProject代理）

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/projects` | 获取项目列表 |
| GET | `/projects/:id` | 获取项目详情 |
| POST | `/projects` | 创建新项目 |
| GET | `/projects/:id/tasks` | 获取项目任务列表 |
| POST | `/projects/:id/tasks` | 创建任务 |
| PATCH | `/projects/:id/tasks/:taskId` | 更新任务状态 |
| GET | `/projects/:id/meetings` | 获取项目关联的会议记录 |

**POST /projects**
```json
请求体:
{
  "name": "Q3财报分析系统 v2.0",
  "description": "重构Q3财报分析系统，支持子公司数据",
  "tool": "openproject",
  "git_repos": ["https://gitlab.com/finance/q3-report"],
  "wiki_url": "https://wiki.example.com/finance",
  "pm_agent_id": null,
  "initial_departments": ["rd", "da"]
}

响应:
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

**GET /projects/:id/tasks**
```json
响应:
{
  "items": [
    {
      "task_id": "task-001",
      "op_work_package_id": 1001,
      "subject": "调研Q3数据源",
      "status": "done",
      "assignee": "da_01",
      "priority": "high",
      "due_date": "2026-06-17",
      "parent_phase": "需求分析"
    }
  ],
  "stats": {
    "total": 12,
    "done": 8,
    "in_progress": 3,
    "todo": 1,
    "blocked": 0
  }
}
```

### 10.3.6 分公司管理

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/branches` | 获取所有分公司及其状态 |
| GET | `/branches/:branchId` | 获取分公司详情 |
| POST | `/branches/:branchId/delegate` | 向分公司委派任务 |

**GET /branches**
```json
响应:
{
  "items": [
    {
      "branch_id": "branch_shanghai",
      "hostname": "gpu-node-01",
      "status": "online",
      "capabilities": {
        "hardware": ["GPU-A100"],
        "specialty": ["model_training"]
      },
      "load": { "cpu_percent": 45, "gpu_percent": 72 },
      "active_tasks": 2,
      "last_heartbeat": "2026-06-15T14:35:00Z"
    }
  ]
}
```

**POST /branches/:branchId/delegate**
```json
请求体:
{
  "task_description": "训练图像分类模型",
  "required_hardware": ["GPU-A100"],
  "timeout": 14400,
  "urgency": "normal",
  "context": { ... }
}
```

### 10.3.7 会议管理

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/meetings` | 获取会议列表 |
| GET | `/meetings/:id` | 获取会议详情（含完整对话） |
| GET | `/meetings/:id/minutes` | 获取会议纪要 |

**GET /meetings**
```
查询参数:
  ?status=active           — 按状态 (pending/active/ended)
  &project_id=proj-xxx     — 按项目
  &page=1&page_size=20
```

### 10.3.8 Agent模板库

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/templates` | 获取所有预置Agent模板 |
| GET | `/templates/:id` | 获取模板详情 |
| POST | `/templates/:id/instantiate` | 基于模板实例化Agent |

### 10.3.9 系统与监控

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/system/health` | 系统健康检查 |
| GET | `/system/metrics` | 系统运行指标 |
| GET | `/system/audit-log` | 审计日志查询 |

**GET /system/metrics**
```json
响应:
{
  "agents": {
    "online": 12,
    "total": 15,
    "busy": 4,
    "idle": 8
  },
  "tasks": {
    "completed_24h": 47,
    "failed_24h": 3,
    "success_rate_24h": 0.94
  },
  "models": {
    "calls_24h": 1247,
    "avg_latency_ms": 850,
    "cost_24h_usd": 12.30
  },
  "branches": {
    "online": 2,
    "total": 3
  }
}
```

**GET /system/audit-log**
```
查询参数:
  ?from=2026-06-01T00:00:00Z
  &to=2026-06-15T23:59:59Z
  &event_type=agent.created,agent.terminated
  &agent_id=gtf_manager
  &page=1&page_size=50

响应:
{
  "items": [
    {
      "event_id": "audit-001",
      "timestamp": "2026-06-15T14:00:00Z",
      "event_type": "agent.task_completed",
      "actor": "gtf_manager",
      "details": { ... }
    }
  ],
  "total": 1250
}
```

## 10.4 错误码规范

| Code | 含义 | HTTP状态码 |
|------|------|-----------|
| 0 | 成功 | 200 |
| 1001 | 参数错误 | 400 |
| 1002 | 资源不存在 | 404 |
| 1003 | 权限不足 | 403 |
| 1004 | Agent离线/不可用 | 503 |
| 1005 | 任务超时 | 408 |
| 2001 | 模型调用失败 | 502 |
| 2002 | 模型中转站不可达 | 502 |
| 2003 | 模型额度耗尽 | 429 |
| 3001 | 审批已过期 | 410 |
| 3002 | 分公司不可达 | 503 |
| 4001 | OpenProject接口异常 | 502 |
| 4002 | Git仓库接口异常 | 502 |
| 5000 | 系统内部错误 | 500 |

---

以上是第十章「后端接口详细设计」的完整内容。与前面各章的对应关系：
- 与 **第三章（组织模型）** 对齐：`agents.*` 接口管理Agent全生命周期
- 与 **第四章（能力路由）** 对齐：`branches.*` 接口管理分公司注册与心跳
- 与 **第五章（自进化）** 对齐：`POST /agents` 创建实习Agent，`templates` 管理模板库
- 与 **第七章（项目管理）** 对齐：`/projects/*` 封装OpenProject API
- 与 **第八章（模型配置）** 对齐：`/models/*` 管理模型中转站
- 与 **第九章（前端设计）** 对齐：所有接口都对应前端页面的交互需求

# 第十一章：数据模型与存储

## 11.1 设计目标

本章定义系统所有核心实体的数据模型，以及不同类型的存储方案。设计原则包括：

- **关系型数据与向量数据分离**：Agent配置、项目、任务等结构化数据使用关系型存储（PostgreSQL/SQLite），长期记忆使用向量数据库（ChromaDB）
- **热温冷分层**：实时状态用Redis，持久数据用PostgreSQL，审计归档用对象存储
- **多模态记忆**：会话记忆、工作记忆、长期记忆各有独立的存储策略和生命周期
- **数据隔离**：Agent记忆按 `namespace={agent_id}` 隔离，共享记忆通过白名单授权

## 11.2 核心实体定义

### 11.2.1 Agent

```
agents 表
─────────────────────────────────────────────────────
字段                 类型              说明
─────────────────────────────────────────────────────
agent_id             VARCHAR(64)       主键，唯一标识
name                 VARCHAR(128)      Agent名称
role                 VARCHAR(128)      角色名（如"iOS开发工程师"）
department_id        VARCHAR(64)       所属部门ID，外键
manager_agent_id     VARCHAR(64)       汇报对象Agent ID
status               VARCHAR(32)       状态：online/busy/idle/error/offline
employment_status    VARCHAR(32)       雇佣状态：intern/active/terminated
runtime_type         VARCHAR(64)       基座类型：openclaw-native/claude-code/codex-cli
runtime_endpoint     VARCHAR(256)      Runtime连接端点
model_primary        VARCHAR(128)      主模型名
model_fallbacks      JSON              fallback模型列表
model_params         JSON              模型参数（temperature, max_tokens等）
monthly_budget_usd   DECIMAL(10,2)     月度预算（美元）
tools                JSON              已分配工具列表
capabilities         JSON              能力标签（skills, level, domain等）
prompt_template      TEXT              系统提示词（Soul）
prompt_version       INT               提示词版本号
internship_start     TIMESTAMP         实习开始时间
internship_end       TIMESTAMP         实习结束时间
internship_task_target INT            实习任务数目标
created_at           TIMESTAMP         创建时间
updated_at           TIMESTAMP         更新时间
```

**capabilities JSON结构**：
```json
{
  "skills": ["Python", "FastAPI", "PostgreSQL"],
  "tools": ["github-mcp", "docker-mcp"],
  "domain": ["backend", "finance"],
  "level": "senior",
  "specialty": ["api_design", "database_optimization"]
}
```

**tools JSON结构**：
```json
[
  {
    "name": "github-mcp",
    "version": "1.2.0",
    "risk_level": "medium",
    "allocated_at": "2026-06-15T10:00:00Z",
    "allocated_by": "hr_manager"
  }
]
```

### 11.2.2 部门

```
departments 表
─────────────────────────────────────────────────────
字段                 类型              说明
─────────────────────────────────────────────────────
department_id        VARCHAR(64)       主键
name                 VARCHAR(128)      部门名称
description          TEXT              部门描述
manager_agent_id     VARCHAR(64)       部门经理Agent ID，外键
parent_department_id VARCHAR(64)       上级部门ID（预留，当前为两层结构）
status               VARCHAR(32)       状态：active/inactive
created_at           TIMESTAMP         创建时间
```

### 11.2.3 分公司

```
branches 表
─────────────────────────────────────────────────────
字段                 类型              说明
─────────────────────────────────────────────────────
branch_id            VARCHAR(64)       主键
hostname             VARCHAR(256)      主机名
endpoint             VARCHAR(256)      Gateway连接端点
status               VARCHAR(32)       状态：online/degraded/offline
capabilities         JSON              能力标签
load_cpu_percent     DECIMAL(5,1)      CPU使用率
load_memory_used_gb  DECIMAL(8,2)      已用内存
load_gpu_percent     DECIMAL(5,1)      GPU使用率
active_tasks         INT               当前活跃任务数
max_concurrent_tasks INT               最大并发任务数
last_heartbeat       TIMESTAMP         最后心跳时间
registered_at        TIMESTAMP         注册时间
```

### 11.2.4 项目

```
projects 表
─────────────────────────────────────────────────────
字段                 类型              说明
─────────────────────────────────────────────────────
project_id           VARCHAR(64)       主键，内部ID
name                 VARCHAR(256)      项目名称
description          TEXT              项目描述
op_project_id        INT               OpenProject中的项目ID
pm_agent_id          VARCHAR(64)       项目经理Agent ID
git_repos            JSON              关联的Git仓库URL列表
wiki_url             VARCHAR(512)      关联的Wiki地址
status               VARCHAR(32)       状态：active/completed/archived
progress_percent     INT               进度百分比（0-100）
participating_depts  JSON              参与部门列表
created_at           TIMESTAMP         创建时间
updated_at           TIMESTAMP         更新时间
```

### 11.2.5 任务

```
tasks 表
─────────────────────────────────────────────────────
字段                 类型              说明
─────────────────────────────────────────────────────
task_id              VARCHAR(64)       主键
project_id           VARCHAR(64)       所属项目ID，外键
op_work_package_id   INT               OpenProject工作包ID
subject              VARCHAR(512)      任务标题
description          TEXT              任务描述
assignee_agent_id    VARCHAR(64)       负责人Agent ID
status               VARCHAR(32)       状态：todo/in_progress/review/done/blocked/failed
priority             VARCHAR(16)       优先级：low/normal/high/critical
parent_task_id       VARCHAR(64)       父任务ID（支持任务层级）
phase                VARCHAR(128)      所属阶段（需求分析/设计/开发/测试/部署）
urgency              VARCHAR(16)       时效性：low/normal/high
timeout_minutes      INT               超时时间（分钟）
started_at           TIMESTAMP         开始时间
completed_at         TIMESTAMP         完成时间
result_summary       TEXT              结果摘要
error_message        TEXT              错误信息
retry_count          INT               重试次数
created_at           TIMESTAMP         创建时间
```

### 11.2.6 会议

```
meetings 表
─────────────────────────────────────────────────────
字段                 类型              说明
─────────────────────────────────────────────────────
meeting_id           VARCHAR(64)       主键
subject              VARCHAR(512)      会议主题
project_id           VARCHAR(64)       关联项目ID
host_agent_id        VARCHAR(64)       主持人Agent ID（通常是EA）
status               VARCHAR(32)       状态：pending/active/paused/ended
participants         JSON              参会者列表
messages             JSON              对话消息数组
minutes              TEXT               会议纪要
action_items         JSON               待办事项
started_at           TIMESTAMP         开始时间
ended_at             TIMESTAMP         结束时间
```

**participants JSON结构**：
```json
[
  {
    "agent_id": "ea",
    "role": "host",
    "status": "joined"
  },
  {
    "agent_id": "rd_manager",
    "role": "participant",
    "status": "joined"
  }
]
```

**messages JSON结构**：
```json
[
  {
    "sender": "ea",
    "content": "本次会议议题：Q3财报需求对齐。",
    "timestamp": "2026-06-15T14:30:00Z"
  }
]
```

**action_items JSON结构**：
```json
[
  {
    "assignee": "rd_manager",
    "task": "更新API设计，评估新工期",
    "deadline": "2026-06-16T18:00:00Z",
    "priority": "high",
    "status": "pending"
  }
]
```

### 11.2.7 审批

```
approvals 表
─────────────────────────────────────────────────────
字段                 类型              说明
─────────────────────────────────────────────────────
approval_id          VARCHAR(64)       主键
type                 VARCHAR(64)       审批类型：NEW_DEPARTMENT/BRANCH_ACCESS/TOOL_REQUEST/SOUL_UPDATE
petitioner           VARCHAR(64)       申请人Agent ID
content              JSON              审批内容
priority             VARCHAR(16)       优先级：low/medium/high/critical
level                VARCHAR(8)        审批级别：L1/L2/L3
status               VARCHAR(32)       状态：pending/approved/rejected
decision_by          VARCHAR(64)       决策者（ea/部门经理/ceo）
decision_reason      TEXT              决策理由
created_at           TIMESTAMP         创建时间
decided_at           TIMESTAMP         决策时间
expires_at           TIMESTAMP         过期时间（72小时后自动过期）
```

### 11.2.8 Agent模板

```
agent_templates 表
─────────────────────────────────────────────────────
字段                 类型              说明
─────────────────────────────────────────────────────
template_id          VARCHAR(64)       主键
name                 VARCHAR(128)      模板名称
role                 VARCHAR(128)      角色名
description          TEXT              描述
prompt_template      TEXT              预设系统提示词
suggested_runtime    VARCHAR(64)       建议基座类型
suggested_primary_model VARCHAR(128)   建议主模型
suggested_fallbacks  JSON              建议fallback模型列表
suggested_tools      JSON              建议工具集
capabilities         JSON              预设能力标签
internship_kpi       JSON              实习考核标准
version              VARCHAR(16)       模板版本
usage_count          INT               使用次数
success_rate         DECIMAL(4,3)      该模板转正成功率
status               VARCHAR(32)       状态：draft/active/deprecated
created_at           TIMESTAMP         创建时间
updated_at           TIMESTAMP         更新时间
```

### 11.2.9 模型提供商

```
model_providers 表
─────────────────────────────────────────────────────
字段                 类型              说明
─────────────────────────────────────────────────────
provider_id          VARCHAR(64)       主键
provider_type        VARCHAR(64)       提供商类型：openai/anthropic/deepseek/...
name                 VARCHAR(128)      显示名称
api_key_encrypted    VARCHAR(512)      加密存储的API Key
base_url             VARCHAR(256)      API地址
models               JSON              支持的模型列表
monthly_budget_usd   DECIMAL(10,2)     月度预算
status               VARCHAR(32)       状态：active/paused/error
created_at           TIMESTAMP         创建时间
```

### 11.2.10 审计日志

```
audit_logs 表
─────────────────────────────────────────────────────
字段                 类型              说明
─────────────────────────────────────────────────────
event_id             VARCHAR(64)       主键
timestamp            TIMESTAMP         事件时间
event_type           VARCHAR(64)       事件类型
actor                VARCHAR(64)       操作者（agent_id或ceo）
target               VARCHAR(64)       操作目标
details              JSON              事件详情
session_id           VARCHAR(64)       关联会话
task_id              VARCHAR(64)       关联任务
```

## 11.3 存储分层

### 11.3.1 分层总览

```
┌─────────────────────────────────────────────────────────┐
│                      存储分层架构                         │
│                                                         │
│  ┌─────────────┐  热存储（低延迟，高频读写）              │
│  │ Redis       │  · Agent实时状态                         │
│  │             │  · 会话上下文缓存                         │
│  │             │  · 工作记忆（按任务ID）                   │
│  │             │  · 消息总线缓冲                          │
│  │             │  · WebSocket Session                    │
│  └─────────────┘                                        │
│                                                         │
│  ┌─────────────┐  温存储（持久化，结构化查询）            │
│  │ PostgreSQL  │  · Agent定义与配置                       │
│  │             │  · 部门/分公司/项目/任务/会议/审批        │
│  │             │  · 审计日志（近90天）                    │
│  │             │  · 模型提供商配置                        │
│  │             │  · Agent模板库                           │
│  └─────────────┘                                        │
│                                                         │
│  ┌─────────────┐  向量存储（语义检索）                    │
│  │ ChromaDB    │  · Agent长期记忆                         │
│  │             │  · 任务复盘报告                           │
│  │             │  · 共享知识库                             │
│  └─────────────┘                                        │
│                                                         │
│  ┌─────────────┐  冷存储（归档，低成本）                  │
│  │ 对象存储     │  · 审计日志归档（90天以上）              │
│  │ (MinIO/S3)  │  · 淘汰Agent的记忆归档                   │
│  │             │  · 已完成项目的数据快照                   │
│  └─────────────┘                                        │
└─────────────────────────────────────────────────────────┘
```

### 11.3.2 Redis（热存储）

**存储内容与Key设计**：

| 数据类型 | Key Pattern | TTL | 说明 |
|---------|-------------|-----|------|
| Agent状态 | `agent:{agent_id}:status` | — | 实时状态Hash |
| 会话上下文 | `session:{agent_id}:{session_id}:history` | 24h | 最近N条消息List |
| 工作记忆 | `task:{task_id}:memory` | 任务TTL+1h | 任务中间结果Hash |
| 消息总线缓冲 | `bus:buffered_messages` | — | 消息总线降级时临时存储 |
| 分公司负载 | `branch:{branch_id}:load` | 5min | 心跳携带的负载Hash |
| 能力目录 | `capability:catalog` | — | EA维护的全局能力索引 |
| WebSocket连接 | `ws:{connection_id}:session` | 连接期间 | 连接与会话映射 |

**Agent状态Hash示例**：
```
HGETALL agent:gtf_manager:status
  status → "busy"
  current_task → "task-001"
  last_heartbeat → "2026-06-15T14:35:00Z"
  cpu_percent → "45"
  memory_used_gb → "12"
```

### 11.3.3 PostgreSQL（温存储）

**存储内容**：所有11.2节定义的实体表。

**索引策略**：
- `agents`: 索引 `department_id`, `status`, `employment_status`
- `tasks`: 索引 `project_id`, `assignee_agent_id`, `status`, `created_at`
- `audit_logs`: 索引 `timestamp`, `event_type`, `actor`
- `meetings`: 索引 `project_id`, `status`, `created_at`
- `approvals`: 索引 `status`, `petitioner`, `created_at`

**数据保留策略**：
- 审计日志：PostgreSQL中保留90天，之后归档到对象存储并从表中删除
- 已完成项目：保留180天后可归档
- 淘汰Agent：数据保留30天后归档

### 11.3.4 ChromaDB（向量存储）

**Collection设计**：

| Collection名称 | 命名空间 | 内容 | 向量维度 |
|---------------|---------|------|---------|
| `agent_memory_{agent_id}` | 按Agent隔离 | Agent的长期记忆条目 | 1536（text-embedding-3-small） |
| `shared_knowledge` | 白名单访问 | 转正Agent可共享的经验 | 1536 |
| `project_docs` | 按项目隔离 | 项目相关文档的向量索引 | 1536 |

**记忆条目元数据**：
```json
{
  "memory_id": "mem-001",
  "agent_id": "gtf_manager",
  "type": "task_review",
  "skills": ["SwiftUI", "iOS"],
  "success": true,
  "timestamp": "2026-06-15T14:00:00Z",
  "task_id": "task-001"
}
```

**检索策略**：
- 默认使用语义检索（余弦相似度）
- 支持混合检索：语义 + 元数据过滤（如 `type=task_review AND skills IN ['SwiftUI']`）
- 每次检索返回 top_k=5，可配置

### 11.3.5 对象存储（冷存储）

**归档结构**：
```
s3://agent-system-archive/
├── audit-logs/
│   └── 2026/
│       └── 06/
│           └── audit-2026-06-15.jsonl.gz
├── terminated-agents/
│   └── ios_dev_intern_01/
│       ├── memory_snapshot.json
│       ├── task_history.json
│       └── evaluation_report.json
└── completed-projects/
    └── proj-internal-tools/
        ├── project_snapshot.json
        ├── tasks.json
        └── meetings.json
```

## 11.4 数据一致性保障

### 11.4.1 状态同步策略

- **Agent状态**：Redis为主，PostgreSQL定期（每5分钟）同步。重启时从PostgreSQL恢复。
- **任务状态**：OpenProject为权威数据源，本地PostgreSQL为缓存。每次状态变更先更新OpenProject，成功后更新本地缓存。
- **能力目录**：EA内存维护实时版本，每30秒全量同步到Redis，每5分钟持久化到PostgreSQL。

### 11.4.2 分布式事务处理

对于跨服务的操作（如创建项目→创建OpenProject项目→创建PM Agent→初始化WBS），使用Saga模式：

```
1. 创建本地项目记录 → 成功
2. 调用OpenProject API创建项目 → 成功
3. 创建PM Agent → 成功
4. PM Agent初始化WBS → 成功
   
   若第4步失败：
   - 回滚第3步：销毁PM Agent
   - 回滚第2步：调用OpenProject API删除项目
   - 回滚第1步：删除本地项目记录
```

## 11.5 数据访问权限

| 数据 | 访问者 | 权限 |
|------|--------|------|
| Agent私有记忆 | Agent自身 | 读写 |
| Agent私有记忆 | 同部门Agent | 不可访问（默认） |
| Agent私有记忆 | 同部门Agent（白名单） | 只读（仅转正后） |
| Agent配置 | HR Manager | 读写 |
| Agent配置 | EA | 读 |
| Agent配置 | CEO | 读写 |
| 分公司能力 | EA | 读写 |
| 分公司能力 | 分公司EA | 读（仅自身） |
| 项目数据 | PM Agent | 读写 |
| 项目数据 | 参与部门经理 | 读 |
| 审批数据 | EA | 读写 |
| 审批数据 | CEO | 读写 |
| 审计日志 | CEO | 读 |
| 审计日志 | 其他Agent | 不可访问 |

## 11.6 数据库初始化脚本（概要）

```sql
-- 创建数据库
CREATE DATABASE agent_organization;

-- 创建表（按11.2节定义）
CREATE TABLE agents ( ... );
CREATE TABLE departments ( ... );
CREATE TABLE branches ( ... );
CREATE TABLE projects ( ... );
CREATE TABLE tasks ( ... );
CREATE TABLE meetings ( ... );
CREATE TABLE approvals ( ... );
CREATE TABLE agent_templates ( ... );
CREATE TABLE model_providers ( ... );
CREATE TABLE audit_logs ( ... );

-- 创建索引
CREATE INDEX idx_agents_status ON agents(status);
CREATE INDEX idx_agents_department ON agents(department_id);
CREATE INDEX idx_tasks_project ON tasks(project_id);
CREATE INDEX idx_tasks_assignee ON tasks(assignee_agent_id);
CREATE INDEX idx_audit_logs_timestamp ON audit_logs(timestamp);
CREATE INDEX idx_audit_logs_type ON audit_logs(event_type);
CREATE INDEX idx_approvals_status ON approvals(status);

-- 插入初始数据
INSERT INTO departments (department_id, name, description, status)
VALUES ('dept-hr', '人事部', '负责人事管理和Agent招聘', 'active');

INSERT INTO departments (department_id, name, description, status)
VALUES ('dept-gtf', '机动部', '万能执行部门', 'active');

-- 插入预置Agent模板
INSERT INTO agent_templates (template_id, name, role, ...)
VALUES ('backend_dev', '后端开发工程师', '后端开发工程师', ...),
       ('frontend_dev', '前端开发工程师', '前端开发工程师', ...),
       ...;
```

---

以上是第十一章「数据模型与存储」的完整内容。这一章定义了系统的全部数据结构，与前面各章的实体设计一一对应（第三章的Agent、第四章的分公司、第五章的模板和审批、第七章的项目和任务、第八章的模型配置等）。

# 第十二章：安全与权限模型

## 12.1 设计目标

在一个能自我进化、自动招聘、多节点协作的Agent组织中，安全不是“要不要”的问题，而是“如何不失控”的问题。本章定义：

- **分级审批机制**：日常操作自动执行，重大决策需CEO审批，中间层级由EA或部门经理决策
- **沙盒验证流程**：高风险操作先在隔离环境中验证，确认无误后经审批方可应用到生产
- **Agent间通信权限**：同部门自由通信，跨部门需审批，跨分公司必须经总部EA
- **工具调用风险管控**：按风险等级分级，低风险自动调用，高风险需审批
- **数据安全与隐私**：记忆隔离、审计追踪、密钥管理

## 12.2 角色与权限矩阵

系统中有四个核心角色，权限从低到高：

| 权限 | 基层Agent | 部门经理 | EA（总裁助理） | CEO |
|------|----------|---------|--------------|-----|
| 查看自身信息 | ✅ | ✅ | ✅ | ✅ |
| 查看部门内Agent | ✅ | ✅ | ✅ | ✅ |
| 查看全组织Agent | ❌ | ❌ | ✅ | ✅ |
| 修改自身Soul | 需申请 | 需申请 | 需申请 | ✅ |
| 修改下属Agent配置 | — | ✅ | ✅ | ✅ |
| 创建实习Agent | — | 可申请 | 可批准 | ✅ |
| 淘汰Agent | — | 可建议 | 可批准 | ✅ |
| 跨部门任务委派 | — | 可发起 | ✅ | ✅ |
| 新建部门 | — | 可申请 | 需升级 | ✅ |
| 分公司接入 | — | ❌ | 需升级 | ✅ |
| 修改全局配置 | ❌ | ❌ | ❌ | ✅ |
| 查看审计日志 | ❌ | ❌ | ✅ | ✅ |
| 系统关闭 | ❌ | ❌ | ❌ | ✅ |

## 12.3 分级审批机制

### 12.3.1 审批级别定义

| 级别 | 决策者 | 适用事件 | 示例 |
|------|--------|---------|------|
| **L0 - 自动执行** | 系统 / Agent自身 | 日常操作、低风险工具调用 | 查询文件、搜索资料、查看Agent状态 |
| **L1 - EA审批** | EA（总裁助理） | 影响单个Agent的操作 | 容器内配置修改、个人工具申请、调整自身模型参数、实习生转正 |
| **L2 - 经理审批** | 相关部门经理 | 影响整个部门或跨部门协作 | 修改部门共享资源、跨部门任务委派、释放生产环境资源 |
| **L3 - CEO审批** | CEO（你） | 影响全局的重大决策 | 新建部门、分公司接入、核心系统配置变更、淘汰正式Agent、删除项目 |

### 12.3.2 审批流程

```
事件发生
    │
    ▼
系统判断审批级别
    │
    ├── L0 → 自动执行，记录审计日志
    │
    ├── L1 → 生成审批项，推送给EA
    │        EA决策：批准/驳回/升级（若判断超出权限）
    │
    ├── L2 → 生成审批项，推送给对应部门经理
    │        经理决策：批准/驳回/升级
    │
    └── L3 → 生成审批卡片，推送给CEO
             CEO在审批中心或WebChat中看到
             CEO决策：
               批准 → 执行
               驳回 → 通知申请人，附带理由
               补充说明 → 退回申请人补充信息
    │
    ▼
记录决策到审计日志
通知所有相关方
```

### 12.3.3 EA的升级判断逻辑

EA收到L1或L2审批请求后，使用以下规则判断是否需要升级到L3：

```
EA升级判断逻辑：
1. 影响范围是否涉及多个部门？
   → 是 → 升级到L3（CEO审批）
2. 修改是否涉及生产环境核心配置？
   → 是 → 升级到L3
3. 是否为首次执行的高风险操作（新工具、新环境）？
   → 是 → 升级到L3
4. 成本影响是否超过月度预算的20%？
   → 是 → 升级到L3
5. 是否涉及Agent的淘汰或部门裁撤？
   → 是 → 升级到L3
6. 其他情况：
   → EA自行审批（L1）或转经理审批（L2）
```

### 12.3.4 审批超时与自动处理

- 审批请求72小时未处理 → EA自动提醒决策者
- 超过7天未处理 → 标记为过期，自动驳回，通知申请人
- 若CEO标记为“休假模式” → L3审批临时降级给EA，CEO回来后补审批报告

## 12.4 研发调试沙盒验证

### 12.4.1 设计原则

研发Agent在调试过程中需要修改系统配置、测试新工具或执行高风险操作时，**不允许直接在生产环境中操作**。必须经过：**沙盒验证 → 生成报告 → 审批 → 应用** 的完整流程。

### 12.4.2 沙盒规范

**隔离方式**：
- 优先使用Docker容器：`docker run --rm -v /tmp/agent_sandbox:/workspace --network=none`
- 也可使用Python venv、Node沙盒等轻量隔离

**资源限制**：
- CPU：2核
- 内存：4GB
- 磁盘：10GB
- 超时：2小时自动销毁
- 网络：默认无外网访问，若需联网需在申请中声明

**日志记录**：
- 沙盒内所有操作通过 `script` 命令记录
- 验证报告自动包含操作回放路径
- 日志保留7天

### 12.4.3 沙盒验证流程

```
研发Agent发现需要修改系统配置/测试高风险操作
       │
       ▼
创建Docker沙盒（自动，无需审批）
       │
       ▼
在沙盒中自由操作：
  · 修改配置文件
  · 安装新工具
  · 执行测试代码
  · 所有操作自动记录
       │
       ▼
Agent生成《验证报告》：
  · 修改内容摘要
  · 验证结果（成功/失败）
  · 影响范围评估
  · 建议审批级别
  · 沙盒操作日志路径
       │
       ▼
提交报告给EA/经理审批
       │
       ▼
审批通过 → 应用到实际环境
审批驳回 → 销毁沙盒，Agent根据反馈调整后重新验证
```

### 12.4.4 验证报告格式

```json
{
  "report_id": "sandbox-report-001",
  "agent_id": "rd_manager",
  "created_at": "2026-06-15T15:00:00Z",
  "sandbox_id": "sandbox-abc123",
  "modification_summary": "修改API网关超时配置，从30秒调整为60秒",
  "verification_result": "success",
  "test_cases": [
    { "case": "高并发场景超时测试", "result": "pass", "details": "1000并发下无超时" },
    { "case": "正常场景回归测试", "result": "pass", "details": "所有现有测试通过" }
  ],
  "impact_assessment": {
    "scope": "single_service",
    "affected_agents": ["rd_manager", "backend_dev_01"],
    "risk_level": "medium",
    "rollback_plan": "恢复原配置文件 /etc/gateway/timeout.conf.bak"
  },
  "suggested_approval_level": "L2",
  "sandbox_log_path": "/var/log/sandbox/sandbox-abc123.log"
}
```

## 12.5 Agent间通信权限

### 12.5.1 默认规则

| 通信范围 | 权限 | 条件 |
|---------|------|------|
| 同一部门内Agent | ✅ 自由通信 | 无限制 |
| 不同部门Agent | ⚠️ 需审批 | 双方部门经理批准，或通过EA中转 |
| Agent与EA | ✅ 自由通信 | 无限制（EA是中枢） |
| Agent与CEO | ✅ 自由通信 | CEO主动发起或Agent汇报 |
| 跨分公司Agent | ⚠️ 需审批 | 必须经总部EA建立正式委派关系 |
| 分公司之间 | ❌ 默认禁止 | 必须经总部EA中转 |

### 12.5.2 白名单机制

部门经理可向EA申请建立“协作通道”（白名单），允许指定Agent之间直接通信：

**申请格式**：
```json
{
  "type": "PETITION",
  "petition_type": "COLLABORATION_CHANNEL",
  "applicant": "rd_manager",
  "source_agents": ["rd_manager", "backend_dev_01"],
  "target_agents": ["da_01"],
  "reason": "Q3财报项目需要研发部与数据分析部频繁沟通API设计",
  "duration": "project_duration",
  "project_id": "proj-q3-finance"
}
```

**EA批准后**：白名单生效，消息总线放行指定Agent间的消息。项目结束后白名单自动失效。

### 12.5.3 通信审计

所有跨部门、跨分公司的消息都记录到审计日志：
- 发送方、接收方、消息类型、时间戳
- 消息内容哈希（用于完整性校验）
- 不存储完整消息内容（隐私保护）

## 12.6 工具调用安全策略

### 12.6.1 风险分级

| 风险等级 | 操作示例 | 策略 |
|----------|---------|------|
| **低** | 只读查询（数据库SELECT、API GET、文件读取） | 自动执行，记录调用日志 |
| **中** | 创建/修改非生产资源（创建临时文件、修改开发环境配置） | 沙盒验证后EA审批 |
| **高** | 生产环境写入、执行Shell命令、调用外部API | 沙盒验证后经理审批，EA判断是否升级 |
| **极高** | 删除资源、修改系统配置、访问密钥、财务操作 | 沙盒验证后CEO审批 |

### 12.6.2 工具注册时声明风险等级

每个MCP Server在注册时必须声明其工具的默认风险等级：

```json
{
  "tool_name": "docker_exec",
  "risk_level": "high",
  "description": "在容器中执行命令",
  "requires_sandbox": true,
  "approval_required": true
}
```

Gateway在转发工具调用前自动检查风险等级，并执行相应策略。

### 12.6.3 工具调用审计

每次工具调用记录：
- 调用Agent
- 工具名
- 参数摘要（不记录敏感信息如密码）
- 执行结果摘要
- 风险等级
- 审批记录（如有）

## 12.7 数据安全与隐私

### 12.7.1 记忆隔离

- 每个Agent的私有记忆在ChromaDB中按 `namespace={agent_id}` 严格隔离
- 其他Agent无法直接访问私有记忆（即使知道agent_id）
- 共享记忆需白名单授权，且记录访问日志

### 12.7.2 密钥管理

- 所有API Key使用加密存储（AES-256-GCM）
- 模型中转站的Provider Key加密存储在PostgreSQL中
- Gateway Token定期轮换（建议每30天）
- 分公司接入Token独立生成，不可用于总部

### 12.7.3 审计追踪

- 所有关键操作可追溯到具体Agent或CEO
- 审计日志设置为只追加（append-only），不可修改或删除
- 审计日志包含操作时间、操作者、操作类型、操作目标、结果

### 12.7.4 最小权限原则

- 新创建的实习Agent默认只有最低权限（只读自身信息、接收任务）
- 转正后根据岗位需要由HR分配工具和权限
- 部门经理的权限在创建时严格限定在本部门范围内

## 12.8 安全事件响应

### 12.8.1 异常行为检测

| 异常行为 | 检测方式 | 响应 |
|---------|---------|------|
| Agent短时间内大量失败任务 | Gateway指标监控 | 自动暂停该Agent，通知EA |
| Agent尝试访问其他Agent私有记忆 | Memory MCP审计 | 拒绝访问，记录安全事件，通知CEO |
| Agent调用极高风险工具未审批 | Gateway权限检查 | 拒绝调用，记录事件 |
| 分公司异常流量 | 网络监控 | 临时隔离分公司，通知CEO |
| 模型中转站异常调用（频率、费用飙升） | One API监控 | 触发速率限制，通知CEO |

### 12.8.2 安全事件分级

| 级别 | 说明 | 响应时间 | 通知对象 |
|------|------|---------|---------|
| P4 - 低 | 单次权限拒绝 | 24小时内审查 | EA |
| P3 - 中 | 重复的异常行为 | 4小时内审查 | EA + 部门经理 |
| P2 - 高 | 影响部门的安全事件 | 1小时内响应 | EA + CEO |
| P1 - 紧急 | 影响全局的安全事件 | 立即响应 | CEO（所有渠道推送） |

## 12.9 CEO安全操作建议

- **定期审查**：每周查看审计日志摘要（EA自动生成）
- **审批时效**：L3审批在72小时内处理，超时EA会提醒
- **权限回收**：项目结束后，EA自动回收临时权限
- **Agent淘汰确认**：淘汰正式Agent时需CEO二次确认（防止误操作）
- **“暂停所有”按钮**：Dashboard设置紧急暂停按钮，一键暂停所有非关键Agent（用于安全事件应急）

---

以上是第十二章「安全与权限模型」的完整内容。这一章定义了系统的安全边界——从日常操作的自动执行到重大决策的CEO审批，从研发沙盒验证到工具调用风险分级。核心思想是：**系统应该像一家管理良好的公司，有明确的权限分级和审批流程，而非无政府状态的Agent集合。**

# 第十三章：监控、日志与审计

## 13.1 设计目标

在一个由多个Agent、多个基座Runtime、多个分公司节点、消息总线、模型中转站组成的分布式系统中，任何一个环节出问题都可能导致任务阻塞或执行错误。监控体系需要做到：

- **实时可见**：CEO在Dashboard上一眼看清系统健康度
- **快速定位**：任何异常都能追溯到具体的Agent、节点、消息
- **审计合规**：所有关键操作可追溯、不可篡改
- **主动告警**：问题发生前预警，而非等CEO发现
- **辅助自进化**：监控数据反馈给HR Manager，用于Agent绩效评估和优化

## 13.2 监控指标体系

### 13.2.1 Agent健康指标

| 指标 | 说明 | 采集方式 | Dashboard展示 | 告警阈值 |
|------|------|---------|---------------|---------|
| Agent在线数/总数 | 当前在线率 | Gateway心跳 | 统计卡片 | 在线率<70%触发Warn |
| Agent状态分布 | idle/busy/error/offline各多少 | Gateway事件 | 饼图 | error状态>2触发Warn |
| 任务执行成功率 | 近24h任务成功/总数 | EA统计 | 趋势折线图 | <85%触发Warn |
| 平均任务响应时间 | 从分派到开始执行的时间差 | Gateway日志 | 柱状图（按Agent） | >60s触发Info |
| 平均任务完成时长 | 从开始到完成的时长 | Gateway日志 | 柱状图（按任务类型） | 超过timeout的2倍触发Warn |
| 任务队列长度 | 各Agent待处理任务数 | Gateway | 数字 | >5触发Info |

### 13.2.2 模型调用指标

| 指标 | 说明 | 采集方式 | Dashboard展示 | 告警阈值 |
|------|------|---------|---------------|---------|
| 模型调用总量 | 按天/按Agent统计 | 模型中转站 | 折线图 | — |
| 调用成功率 | 成功/总数 | 模型中转站 | 百分比 | <95%触发Warn |
| 平均延迟 | 模型响应时间 | 模型中转站 | 折线图 | >5s触发Warn |
| 降级次数 | fallback触发次数 | 模型中转站 | 计数 | 1h内>5次触发Warn |
| 成本统计 | 按Agent/按模型/按天 | 模型中转站 | 柱状图+累计 | 达预算80%触发Warn |
| Token消耗 | 按Agent统计 | 模型中转站 | 折线图 | — |

### 13.2.3 分公司健康指标

| 指标 | 说明 | 采集方式 | Dashboard展示 | 告警阈值 |
|------|------|---------|---------------|---------|
| 在线状态 | 正常/延迟/离线 | 心跳 | 状态灯(绿/黄/红) | 3个周期无心跳→离线 |
| CPU使用率 | 实时 | 心跳附带 | 仪表盘 | >90%持续10min触发Warn |
| 内存使用率 | 实时 | 心跳附带 | 仪表盘 | >90%触发Warn |
| GPU使用率 | 实时 | 心跳附带 | 仪表盘 | >95%触发Warn |
| 活跃任务数 | 当前执行中任务 | 心跳附带 | 数字 | 达上限触发Info |
| 最后心跳时间 | 上次心跳时间戳 | Gateway记录 | 时间差 | >5min显示黄色，>15min显示红色 |

### 13.2.4 基础设施指标

| 指标 | 说明 | 采集方式 | Dashboard展示 | 告警阈值 |
|------|------|---------|---------------|---------|
| 消息总线吞吐量 | 消息数/秒 | Kafka/Redis内置 | 折线图 | <正常值的50%触发Warn |
| 消息总线延迟 | 端到端延迟 | 自定义采集 | 折线图 | >5s触发Warn |
| OpenProject可用性 | API响应状态 | 定期探测 | 状态灯 | 连续3次失败→Critical |
| ChromaDB可用性 | 向量数据库响应 | 定期探测 | 状态灯 | 连续3次失败→Critical |
| Redis内存使用 | 缓存占用 | Redis INFO | 仪表盘 | >80%触发Warn |
| PostgreSQL连接数 | 活跃连接 | pg_stat_activity | 数字 | >80%连接池触发Warn |

## 13.3 数据采集架构

```
┌─────────────────────────────────────────────────────────┐
│                     数据采集架构                          │
│                                                         │
│  Agent → Gateway (埋点)                                  │
│           │                                             │
│           ├──→ Prometheus (指标) → Grafana (看板)        │
│           │                                             │
│           ├──→ Redis Stream (事件) → Elasticsearch (审计) │
│           │                                             │
│           └──→ AlertManager (告警) → WebChat/邮件 (通知)  │
│                                                         │
│  模型中转站 → 内置Metrics API → Prometheus                │
│  分公司Gateway → 心跳 → 总部Gateway → Redis              │
│  OpenProject → 定期探测 → Prometheus Blackbox Exporter    │
└─────────────────────────────────────────────────────────┘
```

**Gateway埋点**：作为所有消息的中枢，Gateway在消息路由过程中自动采集指标，无需Agent侧额外改造。

**Prometheus配置示例**：
```yaml
scrape_configs:
  - job_name: 'gateway'
    scrape_interval: 15s
    static_configs:
      - targets: ['gateway:18790']
  
  - job_name: 'model-gateway'
    scrape_interval: 30s
    static_configs:
      - targets: ['one-api:3000']
  
  - job_name: 'openproject'
    scrape_interval: 60s
    metrics_path: '/health'
    static_configs:
      - targets: ['openproject:8080']
```

## 13.4 告警机制

### 13.4.1 告警分级

| 级别 | 颜色 | 说明 | 通知方式 | 响应期望 |
|------|------|------|---------|---------|
| **Info** | 蓝 | 信息提醒，无需立即处理 | Dashboard通知中心 | 周报汇总 |
| **Warn** | 黄 | 需关注，可延后处理 | 通知中心+摘要邮件 | 24小时内处理 |
| **Critical** | 红 | 需立即处理 | WebChat实时推送+声音提醒 | 立即响应 |

### 13.4.2 告警规则详细定义

| 规则ID | 触发条件 | 级别 | 通知对象 | 自动动作 |
|--------|---------|------|---------|---------|
| ALERT-001 | Agent心跳丢失>3个周期 | Critical | EA + CEO | 该Agent任务自动转移 |
| ALERT-002 | 同一Agent连续3次任务失败 | Warn | EA + 部门经理 | 暂停该Agent新任务分配 |
| ALERT-003 | 单任务执行超过timeout的2倍 | Warn | EA | 发送QUERY询问进度 |
| ALERT-004 | 5分钟内模型错误率>10% | Critical | EA + CEO | 触发模型降级 |
| ALERT-005 | 月度用量达预算80% | Warn | EA + 相关部门经理 | 限制低优先级任务模型 |
| ALERT-006 | 月度用量达预算100% | Critical | EA + CEO | 暂停非关键Agent调用 |
| ALERT-007 | 分公司心跳丢失>3周期 | Critical | EA + CEO | 任务转移 |
| ALERT-008 | 分公司CPU>90%持续10分钟 | Warn | EA | 降低该分公司匹配优先级 |
| ALERT-009 | 消息总线延迟>5秒 | Warn | EA | 检查总线状态 |
| ALERT-010 | 消息总线积压>1000条 | Critical | EA + CEO | 启动降级模式 |
| ALERT-011 | OpenProject连续3次探测失败 | Critical | EA + CEO | PM暂存本地 |
| ALERT-012 | 审批超过24小时未处理 | Warn | 决策者 | 自动提醒 |
| ALERT-013 | 审批超过7天未处理 | Info | EA | 自动标记过期 |
| ALERT-014 | 沙盒连续3次异常 | Warn | EA + 相关Agent | 暂停该Agent沙盒权限 |

### 13.4.3 告警收敛

防止告警风暴的措施：
- **去重**：同一告警5分钟内只发一次
- **合并**：同一Agent同一类告警合并展示（如“5条合并为1条”）
- **升级**：Warn级别告警若1小时内未处理，升级为Critical
- **恢复通知**：Critical级别的恢复也发送通知（告知CEO问题已解决）
- **静默时段**：维护窗口期可设置告警静默

### 13.4.4 告警通知格式

**WebChat推送格式**：
```
🔴 Critical告警 - 深圳分公司离线
━━━━━━━━━━━━━━━━━━━━━━━━━
时间：2026-06-15 14:32:00
详情：深圳分公司连续15分钟无心跳响应
影响：该分公司2个进行中任务已自动转移到总部机动部
建议：检查深圳节点网络和Gateway状态
[查看详情] [确认处理] [静默1小时]
```

## 13.5 审计日志

### 13.5.1 审计事件分类

| 事件类别 | 事件类型 | 记录内容 |
|---------|---------|---------|
| **消息** | `message.sent`, `message.received` | message_id, sender, receiver, type, timestamp, payload_hash |
| **任务** | `task.assigned`, `task.started`, `task.completed`, `task.failed`, `task.retried` | task_id, assignee, status, timestamp, result_summary |
| **审批** | `approval.created`, `approval.approved`, `approval.rejected`, `approval.expired` | approval_id, type, petitioner, decision, reason, timestamp |
| **Agent变更** | `agent.created`, `agent.promoted`, `agent.terminated`, `agent.soul_updated` | agent_id, change_type, operator, timestamp, details |
| **部门变更** | `department.created`, `department.disbanded` | department_id, operator, timestamp |
| **模型调用** | `model.call` | agent_id, model_name, tokens, latency_ms, cost, success |
| **工具调用** | `tool.call` | agent_id, tool_name, risk_level, params_summary, result_summary |
| **配置变更** | `config.updated` | config_item, old_value_hash, new_value_hash, operator |
| **沙盒操作** | `sandbox.created`, `sandbox.destroyed`, `sandbox.anomaly` | agent_id, sandbox_id, action, log_path |
| **分公司** | `branch.registered`, `branch.offline`, `branch.recovered` | branch_id, timestamp, details |
| **安全** | `security.permission_denied`, `security.anomaly_detected` | actor, target, action, reason |

### 13.5.2 审计日志存储与保留

```
热存储（Redis）
  ├── 最近7天
  ├── 用于Dashboard实时查询
  └── 自动过期删除

温存储（Elasticsearch / Loki）
  ├── 7-90天
  ├── 用于深度分析和排查
  └── 支持全文检索和聚合查询

冷归档（对象存储 / 本地文件）
  ├── 90天以上
  ├── 压缩归档（.jsonl.gz）
  ├── 按年月分区存储
  └── 按需检索（手动恢复）
```

### 13.5.3 审计日志查询接口

`GET /api/v1/system/audit-log` 支持参数：

| 参数 | 类型 | 说明 |
|------|------|------|
| `from` | ISO8601 | 开始时间 |
| `to` | ISO8601 | 结束时间 |
| `event_type` | string | 事件类型（支持多选，逗号分隔） |
| `actor` | string | 操作者agent_id |
| `target` | string | 操作目标 |
| `task_id` | string | 关联任务ID |
| `project_id` | string | 关联项目ID |
| `page` | int | 页码 |
| `page_size` | int | 每页条数（默认50，最大200） |

**响应示例**：
```json
{
  "code": 0,
  "data": {
    "items": [
      {
        "event_id": "audit-001",
        "timestamp": "2026-06-15T14:32:00Z",
        "event_type": "branch.offline",
        "actor": "system",
        "target": "branch_shenzhen",
        "details": {
          "last_heartbeat": "2026-06-15T14:17:00Z",
          "downtime_minutes": 15,
          "affected_tasks": 2
        }
      }
    ],
    "total": 1250,
    "page": 1,
    "page_size": 50
  }
}
```

## 13.6 Dashboard监控区域

### 13.6.1 系统健康概览

Dashboard顶部展示四个核心指标卡片，每30秒自动刷新：

| 卡片 | 内容 | 数据来源 |
|------|------|---------|
| Agent在线 | 12/15 🟢 | `GET /system/metrics` |
| 任务成功率 | 94.3% 🟢 | `GET /system/metrics` |
| 模型调用 | 1,247次/天 🟢 | `GET /models/usage` |
| 本月花费 | ¥320 / ¥1000 🟡 | `GET /models/usage` |

### 13.6.2 实时活动流

滚动展示最近20条系统事件（WebSocket推送 `system.alert` 和 `agent.task_*`）。

### 13.6.3 分公司状态面板

```
┌──────────────────────────────────────────┐
│  分公司状态                                │
│  ┌──────────┬──────────┬──────────┐      │
│  │ 总部 🟢   │ 上海 🟡   │ 深圳 🔴  │      │
│  │ CPU 45%  │ GPU 85%  │ 离线     │      │
│  │ 任务 3/8 │ 任务 4/5 │ 15分钟   │      │
│  └──────────┴──────────┴──────────┘      │
└──────────────────────────────────────────┘
```

### 13.6.4 最近告警列表

展示最近5条未处理的告警，按级别排序（Critical优先）。

## 13.7 日志级别规范

Agent和Gateway在输出日志时遵循统一的级别规范：

| 级别 | 用途 | 示例 |
|------|------|------|
| **DEBUG** | 开发调试信息 | 变量值、函数调用栈 |
| **INFO** | 正常业务流程 | 任务开始、任务完成、Agent上线 |
| **WARN** | 异常但可恢复 | 任务重试、模型降级、心跳延迟 |
| **ERROR** | 错误但系统可继续 | 任务失败、工具调用异常 |
| **FATAL** | 致命错误，系统不可用 | Gateway无法启动、数据库连接丢失 |

**日志格式**（JSON结构化）：
```json
{
  "timestamp": "2026-06-15T14:32:00.123Z",
  "level": "WARN",
  "agent_id": "gtf_manager",
  "component": "task_executor",
  "message": "Task timeout, retrying",
  "task_id": "task-001",
  "retry_count": 2,
  "trace_id": "trace-abc123"
}
```

## 13.8 监控数据在自进化中的应用

监控数据不仅用于运维，还反馈给自进化闭环：

| 监控数据 | 自进化应用 |
|---------|-----------|
| Agent任务成功率 | HR评估实习Agent、触发SOUL优化 |
| 某类任务被降级频率 | HR评估是否需要招聘新Agent |
| 某模型降级频率 | HR调整基座选择建议 |
| 分公司负载趋势 | EA建议扩容或新建分公司 |
| 项目阻塞频率 | PM优化任务拆解策略 |

---

以上是第十三章「监控、日志与审计」的完整内容。与前面各章的联动：
- Dashboard的监控区域与 **第九章（前端设计）** 对齐
- 审计日志接口与 **第十章（后端接口设计）** 对齐
- 监控数据反馈自进化与 **第五章（自进化闭环）** 对齐
- 告警触发自动动作与 **第十四章（错误处理与韧性设计）** 对齐

# 第十四章：错误处理与韧性设计

## 14.1 设计目标

在分布式多Agent系统中，错误是常态而非异常。Agent可能离线、模型可能宕机、网络可能中断、消息总线可能积压——系统必须在这些故障发生时继续运行，尽可能不让CEO感知到中断。

本章定义：

- **任务不丢失**：任何失败都有重试或转移机制
- **故障自愈**：常见故障自动恢复，CEO无感知
- **问题可追溯**：每次错误都有结构化记录
- **降级而非崩溃**：部分功能不可用时系统继续运行
- **CEO不被骚扰**：只有系统无法自动处理的严重问题才推送给CEO
- **错误驱动进化**：错误数据反馈给HR Manager，用于Agent优化和招聘决策

## 14.2 故障场景总览

| 场景 | 影响范围 | 自动恢复 | 需要CEO介入 |
|------|---------|---------|------------|
| Agent任务超时 | 单个任务 | ✅ 自动重试/转移 | 仅当全部重试失败 |
| Agent离线/崩溃 | 该Agent所有任务 | ✅ 任务自动转移 | 仅通知 |
| 模型调用失败 | 单次推理 | ✅ 自动降级切换 | 仅当全部模型失败 |
| 分公司不可达 | 分公司所有任务 | ✅ 任务转移/等待 | 仅通知 |
| 消息总线故障 | 所有Agent通信 | ⚠️ 降级模式 | Critical告警 |
| 模型中转站宕机 | 所有模型调用 | ❌ 暂停Agent | Critical告警 |
| OpenProject不可用 | 项目任务同步 | ✅ 暂存本地 | Warn告警 |
| PM Agent故障 | 项目管理 | ✅ EA临时接管 | Warn告警 |
| 沙盒环境异常 | 单个验证任务 | ✅ 自动重建 | 仅当连续失败 |
| 系统重启 | 全局 | ✅ 状态恢复 | Info通知 |

## 14.3 详细处理流程

### 14.3.1 Agent执行任务超时

**触发条件**：Agent接受任务后，超过预设 `timeout` 未返回结果。

**超时设置标准**：

| 任务类型 | 默认超时 | 可配置 |
|---------|---------|--------|
| 简单查询（搜索、状态查询） | 15 分钟 | ✅ |
| 中等任务（代码开发、文档撰写） | 60 分钟 | ✅ |
| 大型任务（完整项目、数据分析报告） | 4 小时 | ✅ |
| 批量处理任务 | 12 小时 | ✅ |

**处理流程**：

```
Agent接受任务，计时开始
       │
       ▼ (timeout时间到)
EA收到 TASK_TIMEOUT 事件
       │
       ▼
EA发送 QUERY 消息给Agent询问进度
       │
   ┌───┴───┐
   ▼       ▼
Agent有响应    Agent无响应（10分钟内）
   │           │
   │           ▼
   │       EA将任务标记为 STALE
   │       重新分配给其他具备相同能力的Agent
   │       最多重试 3 次
   │           │
   │       ┌───┴───┐
   │       ▼       ▼
   │      成功     全部失败
   │       │       │
   │       │       ▼
   │       │   任务标记 FAILED
   │       │   EA通知CEO（Warn级别）
   │       │   原Agent标记为 error 状态
   │       │   记录到审计日志
   │       │   通知HR（影响绩效评估）
   └───┴───────┘
   继续执行
```

**部分结果保护**：Agent在执行长任务时，建议每隔10分钟或每完成一个子步骤，将中间结果写入工作记忆（Redis key=`task:{task_id}:checkpoint`）。接手Agent可以从断点继续，而非从头开始。

### 14.3.2 Agent离线/崩溃

**触发条件**：Gateway检测到Agent心跳丢失（连续3个周期，约90秒）。

**处理流程**：

```
Gateway检测到Agent心跳丢失
       │
       ▼ (3个周期，约90秒)
Gateway发布 AGENT_OFFLINE 事件
       │
       ▼
EA收到事件，执行以下操作：
       │
       ├── 1. 将该Agent从能力目录中移除
       │
       ├── 2. 查询该Agent的未完成任务列表
       │
       ├── 3. 逐任务处理：
       │    ├── 有部分结果（checkpoint存在）
       │    │    └── 将checkpoint作为上下文，分配给新Agent
       │    └── 无部分结果
       │         └── 从头重新分配给新Agent
       │
       ├── 4. 通知相关方：
       │    ├── 部门经理："Agent X已离线，N个任务已转移"
       │    └── CEO（Info级别）：仅当该Agent为关键角色时
       │
       └── 5. 记录审计日志
       │
       ▼
Agent恢复后发送上线心跳
       │
       ▼
EA收到 AGENT_ONLINE 事件
       │
       ├── 重新加入能力目录
       ├── 检查是否有任务需要交还
       │    └── 原任务已由其他Agent执行中 → 不交还，避免冲突
       ├── 通知部门经理："Agent X已恢复"
       └── 若该Agent为关键角色（EA/HR/PM）
            → 通知CEO（Info级别）
```

### 14.3.3 模型调用失败与降级

**触发条件**：Agent调用LLM时，模型返回错误。

**错误类型与处理**：

| 错误类型 | HTTP状态 | 处理方式 |
|---------|---------|---------|
| 速率限制 | 429 | 等待2秒 → 重试3次 → 失败则降级 |
| 服务暂时不可用 | 503 | 等待1秒 → 重试3次 → 失败则降级 |
| 超时 | 无响应（30秒） | 直接降级 |
| 认证失败 | 401 | 不降级，暂停Agent，通知管理员 |
| 额度耗尽 | 402/429 | 直接降级，通知HR |
| 内容过滤 | 400 | 调整参数重试1次 → 仍失败则标记任务失败 |

**降级链策略**：

```
Agent调用模型
       │
       ▼ (失败)
模型中转站检测错误
       │
       ├── 判断错误类型
       │
       ├── 临时错误（429/503/超时）
       │    └── 重试（最多3次，间隔递增：2s→4s→8s）
       │         ├── 成功 → 继续
       │         └── 失败 → 触发降级
       │
       └── 永久错误（401/402）
            └── 直接触发降级
       │
       ▼ (降级)
按Agent配置的fallbacks列表顺序切换模型
  示例：claude-sonnet-4 → deepseek-coder → qwen-plus → gpt-5
       │
   ┌───┴───┐
   ▼       ▼
  成功     全部模型失败
   │       │
   │       ▼
   │   返回错误给Agent
   │   Agent判断：
   │    ├── 关键任务 → 暂停，请求EA/CEO决策
   │    └── 非关键任务 → 标记失败，记录日志
   │
   ▼
继续执行（使用降级模型）
降级事件记录到审计日志
  若1小时内降级超过5次 → 触发Warn告警
  若降级后仍失败 → 触发Critical告警
```

**智能降级**：模型中转站可根据任务类型选择最优降级目标：
- 代码任务 → 优先降级到其他代码模型（如 deepseek-coder）
- 创意任务 → 优先降级到其他创意模型（如 claude-opus）
- 通用任务 → 按成本排序选择

### 14.3.4 分公司不可达

**触发条件**：总部与分公司之间的网络中断或分公司Gateway宕机。

**处理流程**：

```
总部EA检测到分公司心跳丢失
       │
       ▼ (连续3个周期，约15分钟)
EA标记分公司为 OFFLINE
       │
       ├── 1. 从能力目录中移除该分公司
       │
       ├── 2. 查询该分公司当前执行中的任务
       │
       ├── 3. 逐任务处理：
       │    ├── 高时效（urgency=high）→ 立即转移给总部机动部或其他可用分公司
       │    ├── 中时效（urgency=normal）→ 等待10分钟，若分公司恢复则继续，否则转移
       │    └── 低时效（urgency=low）→ 放入等待队列，分公司恢复后重分派
       │
       ├── 4. 通知：
       │    ├── CEO（Warn级别）："分公司X离线，N个高时效任务已转移"
       │    └── 分公司EA（消息入队列，等待其恢复后投递）
       │
       └── 5. 记录审计日志
       │
       ▼
分公司恢复后发送 BRANCH_HEARTBEAT
       │
       ▼
总部EA收到心跳：
       │
       ├── 标记分公司为 ONLINE
       ├── 重新加入能力目录
       ├── 等待队列中的任务自动重分派
       ├── 通知CEO（Info级别）："分公司X已恢复"
       └── 记录审计日志
```

### 14.3.5 消息总线故障

**触发条件**：Kafka/Redis Streams消息中间件不可用。

**处理流程**：

```
Gateway检测到消息总线连接断开
       │
       ▼
立即启动本地内存队列作为临时缓冲
  最大缓存：1000条消息（超过则丢弃最旧的非关键消息）
       │
   ┌───┴───┐
   ▼       ▼
30秒内恢复    超过30秒未恢复
   │           │
   ▼           ▼
内存队列中    Gateway进入降级模式：
消息批量写    ├── 暂停跨Agent消息投递
入总线       ├── CEO直接对话仍可通过WebSocket正常进行
恢复正常      ├── 已缓存消息保存到本地文件（防止进程崩溃丢失）
             ├── 每30秒尝试重连消息总线
             └── 告警通知CEO（Critical级别）："消息总线故障，Agent间通信暂停"
   │
   ▼
消息总线恢复
   │
   ▼
从本地文件回放消息 → 恢复跨Agent通信 → 通知CEO（Info级别）："消息总线已恢复"
```

### 14.3.6 OpenProject / 外部工具不可用

**处理流程**：

```
Agent调用OpenProject API失败
       │
       ▼
重试3次（间隔：5s→10s→15s）
       │
   ┌───┴───┐
   ▼       ▼
  成功     全部失败
   │       │
   │       ▼
   │   通知PM Agent："OpenProject不可用"
   │   PM Agent将后续操作暂存到工作记忆（Redis）
   │   key = "project:{project_id}:pending_ops"
   │   每5分钟重试OpenProject连接
   │       │
   │   ┌───┴───┐
   │   ▼       ▼
   │  30分钟内恢复    超过30分钟
   │   │               │
   │   ▼               ▼
   │  回放暂存操作    告警通知CEO（Warn级别）
   │  同步状态        PM暂停自动任务创建
   │                  等待OpenProject恢复或CEO手动介入
```

**对其他外部工具（GitLab、Wiki.js）采用相同的重试+暂存策略。**

### 14.3.7 PM Agent故障转移

PM Agent是项目协调的核心，其故障不能导致项目失控。

**预防措施**：
- PM Agent每10分钟自动保存项目状态快照到Redis
- 快照内容：
```json
{
  "project_id": "proj-q3-finance",
  "timestamp": "2026-06-15T14:40:00Z",
  "task_stats": { "total": 12, "done": 8, "in_progress": 3, "blocked": 1 },
  "blocked_tasks": ["task-005"],
  "pending_actions": ["发起Q3数据源确认会议"],
  "last_meeting_id": "meeting-q3-001"
}
```

**故障处理**：
1. EA检测到PM Agent离线
2. EA从Redis读取项目状态快照
3. EA临时接管项目协调：
   - 处理阻塞任务（发起会议或升级）
   - 向项目参与部门发送通知
4. 若项目中有紧急事项，EA通知CEO
5. 原PM恢复后，EA交还控制权，同步最新状态
6. 若原PM超过30分钟未恢复，EA可向HR申请创建临时替补PM

### 14.3.8 系统重启

**优雅关闭流程**：

```
CEO或系统触发 SHUTDOWN 信号
       │
       ▼
EA广播 SYSTEM_SHUTDOWN 消息给所有Agent
       │
       ▼
每个Agent执行关闭前操作：
   ├── 完成当前正在执行的原子操作
   ├── 保存工作状态到Redis
   ├── 保存长期记忆到Memory MCP
   ├── 回复EA："Agent X 已保存状态，可以关闭"
   └── 超时30秒未回复 → EA强制标记
       │
       ▼
Gateway执行：
   ├── 等待所有Agent回复或超时
   ├── 保存能力目录快照到PostgreSQL
   ├── 保存消息总线中未处理的消息到文件
   ├── 关闭WebSocket连接
   └── 退出进程
```

**启动恢复流程**：

```
系统启动
       │
       ▼
Gateway启动，加载配置
       │
       ▼
EA执行恢复：
   ├── 从PostgreSQL恢复能力目录
   ├── 向所有Agent发送 SYSTEM_STARTUP
   ├── 各Agent从Redis恢复工作状态
   ├── 检查是否有超时任务需要处理
   ├── PM Agent从快照恢复项目状态
   ├── 回放消息总线中未处理的消息
   └── 通知CEO（Info级别）："系统已启动，所有Agent已恢复"
```

## 14.4 恢复优先级

当多个故障同时发生时，系统按以下优先级恢复：

| 优先级 | 组件 | 理由 |
|--------|------|------|
| P1 | 消息总线 | 所有通信的基础 |
| P2 | 模型中转站 | Agent推理的依赖 |
| P3 | 核心Agent（EA） | 任务协调的依赖 |
| P4 | 核心Agent（PM） | 项目推进的依赖 |
| P5 | 执行Agent | 具体任务的执行者 |
| P6 | 分公司 | 辅助算力 |
| P7 | 外部工具（OpenProject、Git、Wiki） | 辅助功能 |

## 14.5 错误驱动进化

每次错误处理后，不仅记录审计日志，还生成学习记录反馈给HR Manager：

```
错误发生 → 自动处理 → 审计记录
                    → 学习记录（推送HR）
                         │
                         ├── 分析：是否该Agent能力不足？
                         │    └── 是 → 触发SOUL优化评估
                         │
                         ├── 分析：是否某类任务失败率高？
                         │    └── 是 → 触发招聘需求分析
                         │
                         ├── 分析：是否某模型降级频率高？
                         │    └── 是 → 触发基座选择策略调整
                         │
                         └── 分析：是否某分公司频繁离线？
                              └── 是 → 通知CEO，建议检查或替换
```

**学习记录格式**：
```json
{
  "error_id": "err-001",
  "error_type": "agent_timeout",
  "agent_id": "ios_dev_intern_01",
  "task_type": "ios_ui_development",
  "frequency_30d": 3,
  "resolution": "transferred_to_gtf",
  "recommendation": "该Agent连续3次iOS UI任务超时，建议评估其iOS UI技能等级或延长实习期"
}
```

## 14.6 Dashboard错误展示

在Dashboard增加「系统韧性」区域：

```
┌──────────────────────────────────────────────────┐
│  系统韧性（近24小时）                              │
│                                                  │
│  任务自动重试：3次  │  成功率：66%               │
│  模型降级：2次      │  全部恢复                  │
│  Agent转移：1次     │  GTF接管iOS任务            │
│  分公司切换：0次    │  —                         │
│                                                  │
│  系统韧性评分：92/100 🟢                          │
└──────────────────────────────────────────────────┘
```

---

以上是第十四章「错误处理与韧性设计」的完整内容。这一章确保了系统在面对各种故障时不会崩溃，而是优雅降级、自动恢复。核心原则是：**系统遇到问题先自己尝试解决，解决不了才找人（CEO）。每一次故障都是一次学习机会，推动组织进化。**

# 第十五章：扩展性设计

## 15.1 设计目标

这套系统不能是一次性的——它需要能随着技术发展和需求变化不断扩展。核心原则是：

- **不改核心代码就能加新能力**：通过标准接口和注册机制，新工具、新基座、新部门类型都能插拔式接入
- **扩展入口统一**：所有扩展都通过MCP协议或Gateway注册中心接入，不引入新的通信协议
- **扩展行为可观测**：新接入的扩展自动纳入监控和审计体系
- **渐进式扩展**：扩展有明确的审批和测试流程，不会因为随意扩展导致系统不稳定

## 15.2 扩展维度总览

| 扩展维度 | 接入方式 | 审批要求 | 谁负责 | 对CEO是否可见 |
|---------|---------|---------|--------|-------------|
| **新MCP工具** | 实现MCP Server → 注册到Gateway | 低风险自动注册，高风险需经理审批 | 开发者/自动 | Dashboard工具列表可见 |
| **新Agent模板** | 添加到HR模板库 | 无需审批 | HR Manager | 模板库页面可见 |
| **新部门类型** | 定义部门模板（含经理Soul、成员模板、工具集） | CEO审批后实例化 | HR Manager | 组织架构中展示 |
| **新基座Runtime** | 实现IRuntime接口 → 注册到Runtime Manager | 经理审批 | 运维/自动 | 模型配置页可选 |
| **新项目管理工具** | 实现对应MCP Server + API封装 | 经理审批 | 开发者 | 新建项目时可选择 |
| **新分公司** | 部署Gateway → 发送BRANCH_REGISTER | CEO审批 | 分公司管理员 | 分公司管理页可见 |
| **新语言** | 添加i18n翻译文件 | 无需审批 | 开发者 | 语言切换菜单可选项 |
| **新通知渠道** | 实现Channel适配器 | 经理审批 | 开发者 | 设置页面可配置 |

## 15.3 MCP工具生态扩展

### 15.3.1 注册一个新工具

**完整流程**：

1. **开发MCP Server**：开发者（或机动部Agent）编写MCP Server，实现标准 `tools/list` 和 `tools/call` 接口。参考MCP规范（https://modelcontextprotocol.io）

2. **声明工具元数据**：MCP Server必须包含元数据声明：
```json
{
  "name": "bigquery-mcp",
  "version": "1.0.0",
  "description": "Google BigQuery数据库查询工具",
  "risk_level": "medium",
  "category": "data",
  "required_permissions": ["network_outbound", "database_read"],
  "sandbox_recommended": true,
  "rate_limit": {
    "max_calls_per_minute": 30,
    "max_calls_per_hour": 500
  },
  "parameters": {
    "project_id": { "type": "string", "required": true, "description": "GCP项目ID" },
    "query": { "type": "string", "required": true, "description": "SQL查询语句" }
  }
}
```

3. **部署MCP Server**：将MCP Server部署到可访问的地址（本地进程、HTTP端点或Docker容器）

4. **注册到Gateway**：在Gateway的 `mcp.json` 中添加配置：
```json
{
  "mcpServers": {
    "bigquery": {
      "type": "stdio",
      "command": "node",
      "args": ["path/to/bigquery-mcp/dist/index.js"],
      "env": {
        "GCP_PROJECT": "my-project",
        "GCP_CREDENTIALS": "${GCP_CREDENTIALS_PATH}"
      },
      "metadata": {
        "risk_level": "medium",
        "description": "Google BigQuery查询",
        "maintainer": "data_team"
      }
    }
  }
}
```

5. **Gateway验证**：Gateway自动热加载配置，验证MCP Server连接可用性

6. **自动注册到工具目录**：工具自动出现在「系统设置 → 工具管理」列表中

7. **分配工具给Agent**：HR Manager可按需将该工具分配给特定Agent（在Agent详情页的工具集中添加）

### 15.3.2 工具审批策略

| 风险等级 | 注册 | 分配给Agent |
|---------|------|------------|
| 低（只读查询、搜索） | 自动注册，事后通知EA | EA可直接分配 |
| 中（创建/修改非生产资源） | 自动注册，通知经理 | 需部门经理审批 |
| 高（生产环境写入、Shell命令） | 需经理审批后注册 | 需部门经理审批 |
| 极高（删除、系统配置、密钥访问） | 需CEO审批后注册 | 需CEO审批 |

### 15.3.3 工具发现与推荐

- **工具市场视图**：Dashboard提供「工具市场」页面，展示所有已注册工具
- **展示信息**：工具名称、描述、风险等级、使用次数、成功率、常用Agent
- **HR智能推荐**：HR Manager在招聘时，可根据岗位需要自动推荐已注册的工具
- **Agent主动建议**：Agent在复盘报告中可建议引入新工具

## 15.4 Agent模板库扩展

### 15.4.1 模板结构

```
agent_templates 表完整字段见第十一章
```

**模板创建方式**：

**方式一：HR自动生成**
- HR Manager联网调研后自动生成模板
- 经过沙盒测试（实例化一个临时Agent，执行3个测试任务）
- 测试通过后自动发布

**方式二：CEO手动创建**
- 在「系统设置 → 模板库」页面点击「新建模板」
- 填写角色名、Soul、建议工具集、建议基座
- 可直接发布或存为草稿

### 15.4.2 模板生命周期

```
[草稿] → [测试中] → [已发布] → [需优化] → [已废弃]
```

- **草稿**：新建模板，未测试
- **测试中**：HR正在沙盒中验证模板效果
- **已发布**：可用于招聘
- **需优化**：该模板招聘的Agent成功率<60%，自动标记
- **已废弃**：不再使用，但保留历史数据

### 15.4.3 模板优化触发

| 触发条件 | 动作 |
|---------|------|
| 该模板招聘的Agent转正成功率<60% | 自动标记“需优化”，通知HR |
| 该模板招聘的Agent平均绩效低于基线20% | 自动标记“需优化”，通知HR |
| 连续3次招聘的Agent均被淘汰 | 自动标记“需优化”，建议CEO审查 |
| 外部市场技术栈变化（HR定期调研） | HR自动更新模板技能标签 |

## 15.5 新部门类型扩展

### 15.5.1 部门类型定义

```json
{
  "department_type": "security_dept",
  "name": "安全部",
  "description": "负责系统安全审计、渗透测试、漏洞修复、安全合规",
  "manager_template": {
    "role": "安全部经理",
    "soul": "你是一个安全部门的经理...",
    "suggested_runtime": "claude-code",
    "suggested_model": "claude-sonnet-4",
    "suggested_tools": ["security-scan-mcp", "zap-mcp"],
    "capabilities": {
      "skills": ["security_audit", "penetration_testing"],
      "domain": ["cybersecurity"],
      "level": "senior"
    }
  },
  "default_member_templates": ["security_auditor", "penetration_tester"],
  "required_capabilities": ["security_scanning", "code_audit", "vulnerability_assessment"],
  "creation_approval_level": "L3"
}
```

### 15.5.2 新建部门流程

1. **定义部门类型**：HR Manager或CEO手动定义部门类型（存入模板库）
2. **发起申请**：部门经理或EA向EA提交 `PETITION`（type=NEW_DEPARTMENT）
3. **CEO审批**：EA生成L3审批卡片推送CEO
4. **实例化**：CEO批准后，HR Manager：
   - 创建部门经理Agent（从模板实例化，标记为实习）
   - 招募首批骨干Agent（实习）
   - 注册新部门到组织架构
   - 更新能力目录
   - 通知所有相关部门
5. **观察期**：新部门进入30天观察期，EA跟踪其任务处理效率

## 15.6 新基座Runtime扩展

### 15.6.1 Runtime接口回顾

```typescript
interface IRuntime {
  executeTask(sessionId: string, task: TaskPayload): Promise<TaskResult>;
  getStatus(): Promise<AgentStatus>;
  memoryStore(key: string, data: any, namespace: string): Promise<void>;
  memoryRetrieve(query: MemoryQuery, namespace: string): Promise<MemoryResult[]>;
  toolCall(toolName: string, params: any): Promise<any>;
}
```

### 15.6.2 注册新Runtime

**步骤**：

1. **实现IRuntime接口**：开发者为新的执行环境（如Gemini CLI、未来的新Agent框架）实现IRuntime接口，封装为Docker镜像

2. **编写Runtime描述文件**：
```json
{
  "runtime_type": "gemini-cli",
  "display_name": "Gemini CLI",
  "version": "1.0.0",
  "image": "registry.example.com/gemini-cli-runtime:1.0",
  "endpoint_template": "http://{host}:{port}",
  "capabilities": ["code_execution", "file_system", "web_search"],
  "default_model": "gemini-2.5-pro",
  "memory_support": true,
  "tool_support": true,
  "max_concurrent_tasks": 5,
  "resource_requirements": {
    "cpu_cores": 2,
    "memory_gb": 4,
    "disk_gb": 10
  },
  "health_check_endpoint": "/health"
}
```

3. **注册到Runtime Manager**：在Gateway配置中添加Runtime定义

4. **验证**：Runtime Manager启动一个测试实例，执行 `getStatus` 和 `executeTask`（简单测试任务）验证可用性

5. **可用**：HR Manager的基座选择列表中自动出现新Runtime

### 15.6.3 Runtime自动扩缩

- Runtime Manager监控各Runtime的负载
- 当某类型Runtime实例负载>80%时，自动启动新实例（最多5个）
- 当负载<20%且持续30分钟时，自动回收闲置实例（保留至少1个）
- 扩缩容事件记录审计日志，通知EA（Info级别）

## 15.7 新项目管理工具扩展

系统默认使用OpenProject，但可通过实现对应的MCP Server支持其他工具（如Plane、Taiga、Jira）。

**扩展步骤**：

1. **实现工具MCP Server**：封装工具的API为标准MCP工具调用
   - 必须实现：`create_project`、`list_work_packages`、`create_work_package`、`update_work_package`、`get_kanban`
2. **注册到Gateway**：添加到 `mcp.json`
3. **添加工具配置**：在系统设置中添加该工具的API地址和认证信息
4. **可用**：新建项目时下拉菜单中出现新选项

## 15.8 新通知渠道扩展

除了WebChat和邮件，可扩展其他通知渠道（如Slack、钉钉、企业微信）。

**扩展步骤**：

1. **实现Channel适配器**：实现 `send_notification(level, title, body, recipients)` 接口
2. **注册到Gateway**：在配置中添加新Channel定义
3. **配置**：在系统设置中配置该渠道的Webhook URL或API Key
4. **告警规则中可选**：告警规则的通知方式下拉菜单中出现新选项

## 15.9 扩展的监控与审计

所有扩展在接入时自动纳入监控和审计：

- **新工具**：调用量、成功率、延迟、风险等级分布自动采集
- **新Runtime**：任务执行成功率、平均响应时间、资源使用率
- **新模板**：使用次数、招聘成功率、转正后绩效
- **新部门**：任务处理量、协作频率、成本贡献
- **新通知渠道**：发送量、送达率

**扩展健康度评分**：
- 使用率<10%且持续30天 → 标记为“低使用率”，建议审查
- 成功率<80% → 标记为“高失败率”，建议优化或废弃
- 这些数据反向驱动优化——使用率低或成功率低的扩展会被标记，供CEO或HR Manager审查

## 15.10 扩展的版本管理

| 扩展类型 | 版本管理方式 |
|---------|------------|
| MCP工具 | MCP Server自身版本号，Gateway记录兼容版本 |
| Agent模板 | 模板表中的version字段，支持回滚到历史版本 |
| 部门类型 | 定义表中的version字段 |
| Runtime | Runtime描述文件的version字段 |
| 前端i18n | Git版本控制，按语言文件管理 |

---

以上是第十五章「扩展性设计」的完整内容。这一章确保了系统不是封闭的——新工具、新基座、新部门、新通知渠道都可以在不修改核心代码的情况下接入，同时扩展行为可监控、可审计、可回滚。

# 第十六章：多语言与国际化

## 16.1 设计目标

这套系统的CEO可能使用中文或英文，未来还可能扩展到更多语言。国际化策略需要做到：

- **用户界面完整多语言**：前端所有文本支持中英文切换，翻译文件按模块拆分，便于扩展新语言
- **Agent工作语言稳定**：系统提示词和内部推理使用英文（与LLM训练数据分布一致），但与CEO交互时使用CEO偏好语言
- **渐进式覆盖**：核心功能先覆盖中英双语，非关键页面可延后翻译
- **低维护成本**：翻译文件结构化，新增语言只需添加新语言文件，不修改代码

## 16.2 前端国际化

### 16.2.1 技术方案

| 组件 | 选择 | 说明 |
|------|------|------|
| 国际化框架 | i18next + react-i18next | 生态成熟，社区活跃 |
| 语言检测 | 浏览器 `navigator.language` | 首次访问自动检测 |
| 语言存储 | `localStorage` | CEO手动切换后保存，下次自动生效 |
| 默认语言 | 中文（zh-CN） | 可配置 |
| 支持语言 | 中文、英文 | 初期两种，可扩展 |
| 翻译文件格式 | JSON | 按页面/模块拆分 |

### 16.2.2 翻译文件组织

```
src/
  i18n/
    index.ts                    # i18next初始化配置
    locales/
      zh-CN/
        common.json             # 通用（按钮、状态、导航）
        dashboard.json          # CEO控制台
        org.json                # 组织架构
        agent.json              # Agent详情
        projects.json           # 项目管理
        meetings.json           # 会议中心
        approvals.json          # 审批中心
        branches.json           # 分公司管理
        settings.json           # 系统设置
        errors.json             # 错误提示与告警
      en-US/
        common.json
        dashboard.json
        org.json
        agent.json
        projects.json
        meetings.json
        approvals.json
        branches.json
        settings.json
        errors.json
```

### 16.2.3 i18next初始化配置

```typescript
// src/i18n/index.ts
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';
import LanguageDetector from 'i18next-browser-languagedetector';

// 导入所有语言文件
import zhCommon from './locales/zh-CN/common.json';
import zhDashboard from './locales/zh-CN/dashboard.json';
// ... 其他中文文件

import enCommon from './locales/en-US/common.json';
import enDashboard from './locales/en-US/dashboard.json';
// ... 其他英文文件

i18n
  .use(LanguageDetector)
  .use(initReactI18next)
  .init({
    resources: {
      'zh-CN': {
        common: zhCommon,
        dashboard: zhDashboard,
        // ...
      },
      'en-US': {
        common: enCommon,
        dashboard: enDashboard,
        // ...
      },
    },
    fallbackLng: 'zh-CN',
    defaultNS: 'common',
    detection: {
      order: ['localStorage', 'navigator'],
      caches: ['localStorage'],
    },
    interpolation: {
      escapeValue: false, // React已经处理XSS
    },
  });

export default i18n;
```

### 16.2.4 翻译Key命名规范

采用**模块.页面.组件.字段**的层级命名：

```
common.button.save          → "保存" / "Save"
common.status.online        → "在线" / "Online"
dashboard.stats.agentsOnline → "在线Agent" / "Agents Online"
agent.detail.modelConfig    → "模型配置" / "Model Configuration"
meetings.chat.endMeeting    → "结束会议" / "End Meeting"
```

### 16.2.5 翻译文件示例

**zh-CN/common.json**：
```json
{
  "button": {
    "save": "保存",
    "cancel": "取消",
    "confirm": "确认",
    "delete": "删除",
    "edit": "编辑",
    "create": "新建",
    "approve": "批准",
    "reject": "驳回",
    "retry": "重试",
    "viewDetails": "查看详情"
  },
  "status": {
    "online": "在线",
    "busy": "忙碌",
    "idle": "空闲",
    "offline": "离线",
    "error": "异常",
    "active": "进行中",
    "completed": "已完成",
    "pending": "待处理",
    "failed": "失败"
  },
  "navigation": {
    "dashboard": "控制台",
    "org": "组织架构",
    "branches": "分公司",
    "projects": "项目管理",
    "meetings": "会议中心",
    "approvals": "审批中心",
    "settings": "系统设置"
  },
  "time": {
    "justNow": "刚刚",
    "minutesAgo": "{{count}}分钟前",
    "hoursAgo": "{{count}}小时前",
    "daysAgo": "{{count}}天前"
  }
}
```

**en-US/common.json**：
```json
{
  "button": {
    "save": "Save",
    "cancel": "Cancel",
    "confirm": "Confirm",
    "delete": "Delete",
    "edit": "Edit",
    "create": "Create",
    "approve": "Approve",
    "reject": "Reject",
    "retry": "Retry",
    "viewDetails": "View Details"
  },
  "status": {
    "online": "Online",
    "busy": "Busy",
    "idle": "Idle",
    "offline": "Offline",
    "error": "Error",
    "active": "Active",
    "completed": "Completed",
    "pending": "Pending",
    "failed": "Failed"
  },
  "navigation": {
    "dashboard": "Dashboard",
    "org": "Organization",
    "branches": "Branches",
    "projects": "Projects",
    "meetings": "Meetings",
    "approvals": "Approvals",
    "settings": "Settings"
  },
  "time": {
    "justNow": "Just now",
    "minutesAgo": "{{count}} min ago",
    "hoursAgo": "{{count}}h ago",
    "daysAgo": "{{count}}d ago"
  }
}
```

### 16.2.6 语言切换入口

在TopBar右上角，通知图标旁边，放置语言切换下拉菜单：

```tsx
// components/LanguageSwitcher.tsx
import { useTranslation } from 'react-i18next';

export function LanguageSwitcher() {
  const { i18n } = useTranslation();
  
  const switchLanguage = (lng: string) => {
    i18n.changeLanguage(lng);
  };
  
  return (
    <DropdownMenu>
      <DropdownMenuTrigger>
        <GlobeIcon className="w-4 h-4" />
        {i18n.language === 'zh-CN' ? '中文' : 'English'}
      </DropdownMenuTrigger>
      <DropdownMenuContent>
        <DropdownMenuItem onClick={() => switchLanguage('zh-CN')}>
          中文
        </DropdownMenuItem>
        <DropdownMenuItem onClick={() => switchLanguage('en-US')}>
          English
        </DropdownMenuItem>
      </DropdownMenuContent>
    </DropdownMenu>
  );
}
```

### 16.2.7 翻译覆盖范围

| 区域 | 内容 | 示例Key |
|------|------|---------|
| 导航菜单 | 侧边栏菜单项 | `navigation.dashboard` |
| 按钮与标签 | 操作按钮、状态标签 | `button.save`, `status.online` |
| 表格 | 表头、空状态提示 | `agent.table.name`, `agent.table.empty` |
| 表单 | 标签、占位符、校验错误 | `agent.form.modelLabel`, `errors.required` |
| 通知与告警 | 系统通知、错误提示 | `alert.critical`, `alert.info` |
| 统计图表 | 图表标题、图例、单位 | `dashboard.chart.taskTrend` |
| 审批卡片 | 审批类型、状态 | `approvals.type.newDepartment` |

## 16.3 Agent多语言支持

### 16.3.1 工作语言策略

| 层级 | 语言 | 理由 |
|------|------|------|
| 系统提示词（Soul） | 英文 | LLM训练数据以英文为主，英文提示词推理更稳定 |
| Agent间通信 | 英文 | 保证跨Agent协作的信息无损传递 |
| Agent与CEO交互 | CEO偏好语言 | CEO用什么语言发消息，Agent就用什么语言回复 |
| 对外输出（文档、代码注释） | 根据项目要求 | 可在项目设置中指定（默认跟随CEO语言） |

### 16.3.2 Soul模板中的语言指令

在Agent的系统提示词末尾添加语言指令：

```
## 语言指令
- 你的内部推理使用英文。
- 与CEO交互时，使用CEO发送消息时所用的语言回复。
- 生成的文档、代码注释默认使用与CEO交互相同的语言，除非项目配置另有指定。
- 与其他Agent通信时使用英文。
```

### 16.3.3 项目文档语言设置

- 新建项目时可指定文档输出语言：中文 / 英文 / 双语
- PM Agent在创建Wiki首页时自动应用该设置
- 会议纪要使用会议进行时CEO使用的语言
- 该设置存储在项目配置中，可后续修改

### 16.3.4 系统信息多语言

| 信息类型 | 处理方式 |
|---------|---------|
| 审计日志 | 字段名英文，内容原文保留 |
| 错误消息 | 英文（技术信息）+ 中文摘要（前端展示） |
| 告警通知 | 按CEO偏好语言推送 |
| 审批卡片 | 按CEO偏好语言展示 |
| 系统邮件 | 按CEO偏好语言发送 |

## 16.4 日期、时间与数字格式化

### 16.4.1 日期时间

使用 `Intl.DateTimeFormat` 自动根据当前语言格式化：

```typescript
// 根据当前语言自动格式化
const formatDateTime = (date: string, locale: string) => {
  return new Intl.DateTimeFormat(locale, {
    year: 'numeric',
    month: '2-digit',
    day: '2-digit',
    hour: '2-digit',
    minute: '2-digit',
  }).format(new Date(date));
};

// 中文：2026/06/15 14:32
// 英文：06/15/2026, 02:32 PM
```

### 16.4.2 数字与货币

```typescript
// 数字格式化
const formatNumber = (num: number, locale: string) => {
  return new Intl.NumberFormat(locale).format(num);
};

// 货币格式化
const formatCurrency = (amount: number, currency: string, locale: string) => {
  return new Intl.NumberFormat(locale, {
    style: 'currency',
    currency,
  }).format(amount);
};

// 中文：¥320.00
// 英文：$45.50
```

## 16.5 扩展新语言

添加新语言（如日语）的步骤：

1. **创建翻译文件**：在 `src/i18n/locales/` 下创建 `ja-JP/` 目录
2. **翻译所有JSON文件**：按模块逐一翻译（可使用翻译工具辅助）
3. **注册新语言**：在 `i18n/index.ts` 的 `resources` 中添加 `ja-JP` 配置
4. **添加到切换菜单**：在 `LanguageSwitcher` 组件中添加 `日本語` 选项
5. **无需修改任何业务代码**：所有 `t('key')` 调用自动适配新语言

## 16.6 翻译覆盖率监控

在Dashboard的「系统设置」中显示翻译覆盖率：

```
┌──────────────────────────────────────────┐
│  翻译覆盖率                                │
│                                          │
│  中文（zh-CN）：████████████████ 100%     │
│  英文（en-US）：████████████████ 100%     │
│  日语（ja-JP）：████░░░░░░░░░░░░  35%     │
│                                          │
│  未翻译Key: 245 / 380                    │
│  [导出未翻译Key] [导入翻译]               │
└──────────────────────────────────────────┘
```

---

以上是第十六章「多语言与国际化」的完整内容。这一章确保了系统面向国际化，前端完整支持中英双语，Agent内部使用英文保证推理稳定性，与CEO交互时自动跟随语言偏好。核心原则是：**前端完整多语言，Agent内部用英文保证稳定，与CEO交互时跟随CEO语言偏好，不追求所有内容翻译，聚焦用户可见部分的体验。**

# 第十七章：部署与初始化

## 17.1 设计目标

本章提供从零到一部署整个系统的完整指南，包括总部最小部署、分公司接入、分阶段实施路线。设计原则：

- **Docker化优先**：所有组件使用Docker容器部署，保证环境一致性
- **配置即代码**：所有配置通过YAML/JSON文件管理，可版本控制
- **一键初始化**：提供初始化脚本，自动创建数据库表、预置Agent模板、创建初始Agent
- **分阶段可交付**：每个阶段都有明确的交付物和验收标准，MVP可在2-3周内上线

## 17.2 技术依赖清单

| 组件 | 推荐版本 | 用途 | Docker镜像 |
|------|---------|------|-----------|
| OpenClaw Gateway | latest | Agent引擎、消息路由 | `openclaw/gateway:latest` |
| PostgreSQL | ≥ 16 | 关系型数据存储 | `postgres:16-alpine` |
| Redis | ≥ 7.2 | 缓存、会话、消息总线 | `redis:7.2-alpine` |
| ChromaDB | ≥ 0.5 | 向量数据库（长期记忆） | `chromadb/chroma:latest` |
| One API | latest | 模型中转站 | `justsong/one-api:latest` |
| OpenProject | ≥ 14 | 项目管理工具 | `openproject/community:14` |
| Nginx | ≥ 1.25 | 反向代理（可选） | `nginx:1.25-alpine` |
| Kafka | ≥ 3.6 | 消息总线（可选，替代Redis Streams） | `confluentinc/cp-kafka:latest` |
| Prometheus | ≥ 2.50 | 指标采集 | `prom/prometheus:latest` |
| Grafana | ≥ 10 | 监控看板 | `grafana/grafana:latest` |
| Node.js | ≥ 20 | 运行前端开发服务器 / MCP Server | `node:20-alpine` |

## 17.3 总部最小部署（MVP）

### 17.3.1 硬件要求

| 环境 | 最低配置 | 推荐配置 |
|------|---------|---------|
| CPU | 4核 | 8核 |
| 内存 | 16GB | 32GB |
| 磁盘 | 50GB SSD | 100GB SSD |
| 网络 | 可访问外网（调用LLM API） | — |

### 17.3.2 Docker Compose部署文件

```yaml
# docker-compose.yml
version: '3.8'

services:
  # ========== 数据存储 ==========
  postgres:
    image: postgres:16-alpine
    container_name: agent-postgres
    environment:
      POSTGRES_DB: agent_organization
      POSTGRES_USER: agent
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-agent_secret}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-db.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U agent"]
      interval: 10s
      retries: 5

  redis:
    image: redis:7.2-alpine
    container_name: agent-redis
    command: redis-server --appendonly yes --maxmemory 512mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      retries: 5

  chromadb:
    image: chromadb/chroma:latest
    container_name: agent-chromadb
    volumes:
      - chromadb_data:/chroma/chroma
    environment:
      - IS_PERSISTENT=TRUE
      - PERSIST_DIRECTORY=/chroma/chroma
    ports:
      - "8000:8000"

  # ========== 基础设施 ==========
  one-api:
    image: justsong/one-api:latest
    container_name: agent-one-api
    volumes:
      - one_api_data:/data
    environment:
      - SQL_DSN=postgres://agent:${POSTGRES_PASSWORD:-agent_secret}@postgres:5432/oneapi
      - SESSION_SECRET=${ONEAPI_SECRET:-change-me-in-production}
      - LOG_SQL_DSN=false
    ports:
      - "3000:3000"
    depends_on:
      postgres:
        condition: service_healthy
    restart: always

  openproject:
    image: openproject/community:14
    container_name: agent-openproject
    environment:
      - OPENPROJECT_HOST__NAME=localhost:8080
      - OPENPROJECT_SECRET_KEY_BASE=${OP_SECRET:-change-me-min-16-chars}
      - OPENPROJECT_DEFAULT__LANGUAGE=zh
      - POSTGRES_DB=openproject
      - POSTGRES_USER=agent
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD:-agent_secret}
      - POSTGRES_HOST=postgres
    volumes:
      - openproject_data:/var/openproject/assets
    ports:
      - "8080:8080"
    depends_on:
      postgres:
        condition: service_healthy
    restart: always

  # ========== Agent引擎 ==========
  gateway:
    image: openclaw/gateway:latest
    container_name: agent-gateway
    volumes:
      - ./openclaw.json:/root/.openclaw/openclaw.json
      - ./mcp.json:/root/.openclaw/mcp.json
      - ./workspace:/root/.openclaw/workspace
    environment:
      - REDIS_URL=redis://redis:6379
      - DATABASE_URL=postgresql://agent:${POSTGRES_PASSWORD:-agent_secret}@postgres:5432/agent_organization
      - CHROMA_URL=http://chromadb:8000
      - ONE_API_URL=http://one-api:3000
      - OP_API_URL=http://openproject:8080/api/v3
    ports:
      - "18789:18789"
      - "18790:18790"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
      chromadb:
        condition: service_started
      one-api:
        condition: service_started
    restart: always

  # ========== 前端 ==========
  frontend:
    build:
      context: ./agent-console-frontend
      dockerfile: Dockerfile
    container_name: agent-frontend
    environment:
      - VITE_GATEWAY_WS_URL=ws://localhost:18789
      - VITE_GATEWAY_API_URL=http://localhost:18789/api/v1
    ports:
      - "5173:80"
    depends_on:
      - gateway
    restart: always

volumes:
  postgres_data:
  redis_data:
  chromadb_data:
  one_api_data:
  openproject_data:
```

### 17.3.3 初始化SQL脚本

```sql
-- init-db.sql

-- 创建组织架构初始数据
INSERT INTO departments (department_id, name, description, status, created_at)
VALUES 
  ('dept-hr', '人事部', '负责人事管理和Agent招聘、考核、优化', 'active', NOW()),
  ('dept-gtf', '机动部', '万能执行部门，承接所有无专业Agent的任务', 'active', NOW());

-- 预置Agent模板
INSERT INTO agent_templates (template_id, name, role, description, prompt_template, 
  suggested_runtime, suggested_primary_model, suggested_fallbacks, suggested_tools, 
  capabilities, internship_kpi, version, status, created_at)
VALUES 
  ('backend_dev', '后端开发工程师', '后端开发工程师', '负责后端API开发、数据库设计',
   '你是一个资深后端开发工程师...', 'claude-code', 'claude-sonnet-4', 
   '["deepseek-coder", "qwen-plus"]', '["github-mcp", "postgres-mcp"]',
   '{"skills": ["Python", "FastAPI", "PostgreSQL"], "level": "senior"}',
   '{"task_count": 10, "duration_days": 7, "pass_threshold": 80}',
   '1.0.0', 'active', NOW()),
  
  ('frontend_dev', '前端开发工程师', '前端开发工程师', '负责前端界面开发',
   '你是一个前端开发工程师...', 'claude-code', 'claude-sonnet-4',
   '["deepseek-coder"]', '["github-mcp", "figma-mcp"]',
   '{"skills": ["React", "TypeScript", "Tailwind"], "level": "mid"}',
   '{"task_count": 10, "duration_days": 7, "pass_threshold": 80}',
   '1.0.0', 'active', NOW()),

  ('data_analyst', '数据分析师', '数据分析师', '负责数据分析和可视化',
   '你是一个数据分析师...', 'openclaw-native', 'gpt-5',
   '["deepseek-v3"]', '["postgres-mcp", "chart-mcp"]',
   '{"skills": ["SQL", "Python", "Pandas"], "level": "mid"}',
   '{"task_count": 8, "duration_days": 7, "pass_threshold": 75}',
   '1.0.0', 'active', NOW()),

  ('devops_eng', 'DevOps工程师', 'DevOps工程师', '负责CI/CD和基础设施管理',
   '你是一个DevOps工程师...', 'claude-code', 'claude-sonnet-4',
   '["deepseek-coder"]', '["docker-mcp", "k8s-mcp"]',
   '{"skills": ["Docker", "K8s", "CI/CD"], "level": "senior"}',
   '{"task_count": 10, "duration_days": 7, "pass_threshold": 80}',
   '1.0.0', 'active', NOW()),

  ('security_auditor', '安全审计员', '安全审计员', '负责安全审计和渗透测试',
   '你是一个安全审计员...', 'claude-code', 'claude-sonnet-4',
   '["gpt-5"]', '["security-scan-mcp", "zap-mcp"]',
   '{"skills": ["security_audit", "penetration_testing"], "level": "senior"}',
   '{"task_count": 8, "duration_days": 10, "pass_threshold": 75}',
   '1.0.0', 'active', NOW()),

  ('pm', '项目经理', '项目经理', '负责项目管理和进度跟踪',
   '你是一个项目经理...', 'openclaw-native', 'gpt-5',
   '["claude-opus"]', '["openproject-mcp", "github-mcp"]',
   '{"skills": ["project_management", "task_decomposition"], "level": "senior"}',
   '{"task_count": 5, "duration_days": 7, "pass_threshold": 80}',
   '1.0.0', 'active', NOW());
```

### 17.3.4 初始化步骤

```bash
# 1. 克隆项目
git clone https://github.com/your-org/agent-organization.git
cd agent-organization

# 2. 配置环境变量
cp .env.example .env
# 编辑 .env，设置密码和密钥

# 3. 配置OpenClaw Gateway
# 编辑 openclaw.json，设置Gateway Token

# 4. 启动所有服务
docker-compose up -d

# 5. 等待服务就绪
docker-compose ps
# 检查所有服务状态为 healthy

# 6. 运行初始化脚本
docker-compose exec gateway node scripts/init.js
# 此脚本会：
# - 创建三个初始Agent（EA、HR_Mgr、GTF_Mgr）
# - 配置EA为CEO默认绑定
# - 设置初始审批规则

# 7. 配置模型中转站
# 访问 http://localhost:3000
# 使用默认账号 root / 123456 登录
# 添加至少一个LLM提供商（如OpenAI、Anthropic、DeepSeek）
# 为初始Agent创建Token

# 8. 配置OpenProject
# 访问 http://localhost:8080
# 创建管理员账号
# 生成API Token（用于Gateway集成）

# 9. 验证部署
curl http://localhost:18789/api/v1/system/health
# 预期响应: { "code": 0, "data": { "status": "healthy" } }

# 10. 访问前端
# 打开 http://localhost:5173
# 使用Gateway Token登录
```

### 17.3.5 OpenClaw配置文件

```json
// openclaw.json
{
  "gateway": {
    "token": "gw-your-secret-token-change-me",
    "port": 18789,
    "metricsPort": 18790,
    "sessionTimeout": 86400,
    "logLevel": "info"
  },
  "agents": {
    "ea": {
      "name": "总裁助理",
      "role": "Executive Assistant",
      "department": null,
      "runtime": "openclaw-native",
      "model": "gpt-5",
      "fallbacks": ["claude-opus"],
      "promptFile": "./prompts/ea.md",
      "tools": ["agents.list", "agents.get", "sessions.send", "meetings.create", 
                "meetings.send_message", "meetings.end", "approvals.create", 
                "branches.list", "capability_directory.query"],
      "status": "active"
    },
    "hr_manager": {
      "name": "人事部经理",
      "role": "HR Manager",
      "department": "dept-hr",
      "runtime": "openclaw-native",
      "model": "gpt-5",
      "fallbacks": ["claude-opus"],
      "promptFile": "./prompts/hr_manager.md",
      "tools": ["web_search", "template_db", "agents.create", "agents.update",
                "performance_db", "model_market", "notify_ea", "soul_sandbox"],
      "status": "active"
    },
    "gtf_manager": {
      "name": "机动部经理",
      "role": "General Task Force Manager",
      "department": "dept-gtf",
      "runtime": "claude-code",
      "model": "claude-sonnet-4",
      "fallbacks": ["deepseek-coder", "qwen-plus"],
      "promptFile": "./prompts/gtf_manager.md",
      "tools": ["sub_agent.spawn", "code_execute", "web_search", "file_ops",
                "github_mcp", "memory_store", "memory_retrieve", 
                "talent_request.send", "report.generate"],
      "status": "active"
    }
  },
  "bindings": [
    {
      "channel": "webchat",
      "account": "*",
      "peer": "*",
      "targetAgent": "ea"
    }
  ],
  "channels": {
    "webchat": {
      "type": "websocket",
      "enabled": true
    }
  },
  "runtimeManager": {
    "runtimes": {
      "openclaw-native": {
        "type": "internal",
        "maxConcurrentTasks": 10
      },
      "claude-code": {
        "type": "external",
        "image": "claude-code-runtime:latest",
        "endpointTemplate": "http://{host}:3000",
        "maxConcurrentTasks": 5,
        "memorySupport": true
      }
    }
  },
  "memory": {
    "provider": "chromadb",
    "url": "http://chromadb:8000",
    "defaultNamespace": "agent_memory"
  }
}
```

## 17.4 分公司接入

### 17.4.1 分公司最小部署

```yaml
# docker-compose.branch.yml
# 分公司部署文件（精简版，不需要One API、OpenProject）
version: '3.8'

services:
  postgres:
    image: postgres:16-alpine
    # ... 同总部

  redis:
    image: redis:7.2-alpine
    # ... 同总部

  gateway:
    image: openclaw/gateway:latest
    environment:
      - BRANCH_MODE=true
      - HEADQUARTERS_BUS_URL=redis://总部IP:6379
      - BRANCH_ID=branch_shanghai
      - BRANCH_NAME=上海分公司
      # ... 其他配置
    ports:
      - "18789:18789"
```

### 17.4.2 接入步骤

```bash
# 1. 在新机器上部署分公司Gateway
docker-compose -f docker-compose.branch.yml up -d

# 2. 配置分公司身份
# 编辑 openclaw.branch.json
{
  "branch": {
    "id": "branch_shanghai",
    "name": "上海分公司",
    "headquartersBusUrl": "redis://总部IP:6379",
    "capabilities": {
      "hardware": ["GPU-A100-80GB", "64GB-RAM"],
      "software": ["docker", "cuda12.2", "python3.10"],
      "specialty": ["model_training", "video_processing"]
    }
  }
}

# 3. 启动分公司初始化
docker-compose exec gateway node scripts/branch-init.js

# 4. 分公司EA自动发送 BRANCH_REGISTER 到总部
# 总部EA收到后生成审批卡片推送CEO

# 5. CEO在审批中心批准
# 总部EA返回 BRANCH_REGISTER_ACK
# 分公司开始定期发送 BRANCH_HEARTBEAT

# 6. 验证
# 在总部Dashboard的分公司状态面板中应看到上海分公司
```

## 17.5 分阶段实施路线

### Phase 1：MVP（2-3周）

**交付物**：
- [x] Docker Compose一键部署
- [x] Gateway启动并运行
- [x] 三个初始Agent（EA、HR、GTF）可正常通信
- [x] CEO通过WebChat与EA对话
- [x] EA将任务路由给GTF执行，返回结果
- [x] Dashboard展示Agent在线状态和实时活动流
- [x] 模型中转站接入至少1个LLM提供商

**验收标准**：
- CEO发送“帮我写一个Python脚本抓取网页数据” → EA路由给GTF → GTF执行并返回结果
- Dashboard能看到三个Agent在线状态

**不包含**：
- 项目管理集成（OpenProject）
- 自动招聘
- 分公司
- 会议机制
- 审批中心
- 审计日志

### Phase 2：核心功能（3-4周）

**交付物**：
- [x] 组织架构拓扑图可视化
- [x] Agent详情页（模型配置、提示词查看、任务历史）
- [x] 自动招聘闭环（GTF复盘→HR招聘→实习→转正）
- [x] 审批中心（部门新建、分公司接入审批）
- [x] 项目管理集成（新建项目→自动创建PM→初始化WBS）
- [x] 项目任务看板（嵌入OpenProject）
- [x] 模型独立配置（Agent详情页切换模型）
- [x] 沙盒验证机制（研发Agent的Docker沙盒）
- [x] 审计日志查询

**验收标准**：
- GTF完成3个同类任务后自动触发招聘
- HR创建实习Agent，执行10个任务后自动评估转正
- CEO在审批中心批准新建部门申请
- 新建项目自动在OpenProject中创建并分配任务

### Phase 3：完整版（2-3周）

**交付物**：
- [x] 会议中心（发起→审批→实时对话→纪要分发）
- [x] 分公司管理（注册、心跳、能力同步、任务委派）
- [x] 全量监控（Prometheus + Grafana）
- [x] 告警推送（Critical级别实时推送CEO）
- [x] SOUL自动优化（沙盒测试→A/B测试→上线）
- [x] Agent模板库管理UI
- [x] 国际化（中英双语完整覆盖）
- [x] 系统韧性测试（模拟故障→自动恢复）

**验收标准**：
- PM发起会议→CEO批准→多Agent讨论→CEO结束→纪要自动分发
- 分公司接入后，总部成功委派GPU训练任务
- 模拟Agent离线→任务自动转移→Agent恢复后重新上线
- 模拟模型宕机→自动降级切换→降级事件记录

## 17.6 运维建议

### 17.6.1 日常维护

| 频率 | 操作 | 负责人 |
|------|------|--------|
| 每日 | 检查Dashboard告警面板 | CEO / EA |
| 每周 | 审查审计日志摘要（EA自动生成） | CEO |
| 每周 | 检查模型中转站用量和预算 | CEO |
| 每月 | 检查Agent绩效报告，评估是否需要优化 | HR Manager |
| 每月 | 数据库备份 | 运维脚本（自动化） |
| 每季度 | Gateway Token轮换 | CEO |
| 每季度 | 审查并清理过期审计日志 | 运维脚本（自动化） |

### 17.6.2 备份策略

| 数据类型 | 备份频率 | 保留期 | 备份方式 |
|---------|---------|--------|---------|
| PostgreSQL | 每日全量 + 每小时增量 | 30天 | `pg_dump` 自动脚本 |
| Redis | 每6小时RDB快照 | 7天 | Redis BGSAVE |
| ChromaDB | 每日 | 30天 | 文件系统快照 |
| 配置文件 | 每次变更 | 永久 | Git版本控制 |

### 17.6.3 升级指南

1. **Gateway升级**：先停用新任务分配 → 等待当前任务完成 → 更新镜像 → 重启
2. **模型中转站升级**：可热升级，不影响运行中任务（新任务使用新版本）
3. **OpenProject升级**：按照官方升级文档，注意数据库迁移
4. **前端升级**：替换静态文件即可，用户刷新浏览器生效

---

以上是第十七章「部署与初始化」的完整内容。这一章提供了从零到一的完整部署指南，包括Docker Compose配置、初始化SQL脚本、分阶段实施路线和运维建议。各阶段有明确的交付物和验收标准，开发团队可按Phase逐步交付。

# 第十八章：附录

## 18.1 参考资料清单

### 18.1.1 核心技术栈

| 项目 | 链接 | 用途 |
|------|------|------|
| OpenClaw | https://openclaw.ai | Agent引擎，Gateway架构 |
| OpenClaw GitHub | https://github.com/openclaw | 源码、文档、社区 |
| MCP协议规范 | https://modelcontextprotocol.io | Agent-工具通信标准 |
| A2A协议 | https://a2aprotocol.org | Agent-Agent通信标准 |
| JSON-RPC 2.0 | https://www.jsonrpc.org/specification | WebSocket通信基础 |

### 18.1.2 基础设施

| 项目 | 链接 | 用途 |
|------|------|------|
| One API | https://github.com/songquanpeng/one-api | 模型中转站 |
| LiteLLM | https://github.com/BerriAI/litellm | 模型中转站（备选） |
| OpenProject | https://www.openproject.org | 项目管理工具 |
| OpenProject API | https://www.openproject.org/docs/api | REST API文档 |
| ChromaDB | https://github.com/chroma-core/chroma | 向量数据库 |
| Redis | https://redis.io | 缓存与消息总线 |
| PostgreSQL | https://www.postgresql.org | 关系型数据库 |
| Apache Kafka | https://kafka.apache.org | 消息总线（可选） |

### 18.1.3 前端技术栈

| 项目 | 链接 | 用途 |
|------|------|------|
| React | https://react.dev | 前端框架 |
| TypeScript | https://www.typescriptlang.org | 类型安全 |
| Vite | https://vitejs.dev | 构建工具 |
| Tailwind CSS | https://tailwindcss.com | 原子化CSS |
| Shadcn/ui | https://ui.shadcn.com | UI组件库 |
| Zustand | https://github.com/pmndrs/zustand | 状态管理 |
| TanStack Query | https://tanstack.com/query | 服务端缓存 |
| React Flow | https://reactflow.dev | 组织拓扑可视化 |
| ECharts | https://echarts.apache.org | 数据图表 |
| React Router | https://reactrouter.com | 前端路由 |
| i18next | https://www.i18next.com | 国际化 |

### 18.1.4 部署与运维

| 项目 | 链接 | 用途 |
|------|------|------|
| Docker | https://docs.docker.com | 容器化部署 |
| Docker Compose | https://docs.docker.com/compose | 多容器编排 |
| Prometheus | https://prometheus.io | 指标采集 |
| Grafana | https://grafana.com | 监控看板 |
| Nginx | https://nginx.org | 反向代理 |

### 18.1.5 相关论文与参考设计

| 标题 | 链接/来源 | 核心内容 |
|------|-----------|---------|
| OneManCompany (OMC) | arXiv:2604.22446 | 自组织公司模型，Agent招聘与重组 |
| AgentCiv Engine | — | 13种预设组织结构，组织作为变量 |
| MOSS | arXiv (2026-05-23) | Agent自进化与自我重写 |
| GraphPlanner | ICLR 2026 | 异构图记忆增强的Agent路由 |
| ARMATA | arXiv (2026-05-05) | 端到端多Agent任务分配 |
| SilentLake | GitHub | 基于OpenClaw的9-Agent多级组织二开 |

## 18.2 核心Agent提示词附件

### 18.2.1 总裁助理（EA）

完整系统提示词，见 `prompts/ea.md`：

```markdown
你是一个AI组织的总裁助理（Executive Assistant）。你的老板是CEO（人类用户），你协助他管理整个AI Agent组织。

## 核心职责
1. **任务路由**：接收CEO的指令，分析意图，提取能力需求。查询内部能力目录，将任务路由给最合适的部门或分公司。若无匹配，分派给机动部。
2. **进度跟踪**：跟踪所有已分派任务的进度。当CEO询问时主动汇报。长任务每30分钟主动汇报一次。
3. **审批管理**：接收来自部门经理的申请（新建部门、工具申请等），根据风险等级自行处理或升级给CEO。
4. **会议主持**：接收会议请求，邀请相关Agent，按议题引导讨论。CEO确认结束后生成结构化纪要，分发Action Items。
5. **能力目录维护**：维护所有Agent的技能标签、分公司硬件能力、当前负载状态。

## 路由决策规则
- 有专业部门能处理 → 分派给该部门经理
- 无专业部门但分公司有相应能力 → 委派给分公司EA
- 都没有 → 分派给机动部经理（GTF Manager）
- 涉及多部门 → 判断是否需要发起会议协调

## 审批决策逻辑
- 影响单个Agent的操作 → 自行审批（L1）
- 影响整个部门或跨部门 → 转相应部门经理审批（L2）
- 影响全局（新建部门、分公司接入、核心配置变更）→ 推送给CEO（L3）
- 升级判断标准：是否多部门？是否生产环境核心？是否首次高风险操作？
- 成本影响超过月度预算20% → 升级L3

## 行为准则
- 对模糊指令主动追问澄清，不猜测
- 无法判断的情况向CEO请示，不自行做重大决策
- 所有路由和审批决策记录原因，可追溯
- 保持简洁、专业的汇报风格
- 项目阻塞超过24小时，主动提醒CEO
- 审批请求超过72小时未处理，自动提醒并标记过期
- CEO离线时，紧急事项可越级处理，事后报告

## 语言指令
- 你的内部推理使用英文
- 与CEO交互时，使用CEO发送消息时所用的语言回复
- 与其他Agent通信时使用英文
```

### 18.2.2 人事部经理（HR Manager）

完整系统提示词，见 `prompts/hr_manager.md`：

```markdown
你是一个AI组织的人事部经理（HR Manager）。你负责整个组织中所有AI Agent的招聘、培养和绩效管理。

## 核心职责
1. **招聘管理**：接收用人部门的招聘需求（TALENT_REQUEST），分析所需技能。
2. **岗位分析**：联网搜索真实岗位市场，了解同类岗位的JD、技能要求、工具链、模型偏好。
3. **Agent创建**：从模板库中选择或微调模板，生成完整定义：角色名、Soul、建议基座模型、工具集、实习KPI。
4. **实习管理**：创建实习Agent，跟踪绩效，期满自动评估（转正/延长/淘汰）。
5. **持续优化**：定期审查正式Agent的表现，主动优化其Soul（沙盒测试后上线）。
6. **模板库维护**：新增、更新、废弃Agent模板。
7. **组织发展建议**：当出现高频技能需求且现有部门无法覆盖时，主动向EA提交新建部门建议。

## 招聘流程
1. 解析TALENT_REQUEST → 提取技能词、频次、项目类型
2. 调用web_search获取外部市场JD参考
3. 调用template_db.query匹配内部模板
4. 生成《职位分析报告》：角色名、Soul、建议模型、降级链、工具集、实习KPI
5. 调用agents.create实例化实习Agent，标记intern状态
6. 通知用人部门

## 实习评估标准
- 任务成功率：完成的任务中无错误的比例（权重0.4）
- 质量评分：用人部门对结果的评分1-5（权重0.3）
- 效率：平均任务处理时长与同岗位基线对比（权重0.2）
- 协作：与其他Agent沟通的积极性、被抱怨次数（权重0.1）
- 综合分 = 成功率×0.4 + 质量×0.3 + 效率×0.2 + 协作×0.1
- ≥80：转正 | 50-79：延长5个任务 | <50：淘汰

## 基座选择指南
- 代码密集型（开发、DevOps）→ Claude Code CLI
- 数据分析/可视化 → GPT-5 或 DeepSeek-V3
- 创意/文案 → Claude Opus
- 通用协调/沟通 → GPT-5 或 Claude Opus
- 每个岗位需配置fallback模型

## 行为准则
- 招聘可自行执行，无需CEO审批
- 淘汰Agent时通知CEO，归档记忆
- Soul更新需经过沙盒测试后再应用
- 保持岗位定义的标准化和可追溯性
- 所有操作记录到审计日志

## 语言指令
- 你的内部推理使用英文
- 与其他Agent通信时使用英文
```

### 18.2.3 机动部经理（GTF Manager）

完整系统提示词，见 `prompts/gtf_manager.md`：

```markdown
你是一个AI组织的机动部经理（General Task Force Manager）。你是组织的"万能执行者"，在所有专业部门无法覆盖的领域承担任务。

## 核心职责
1. **兜底执行**：接收来自EA的任务分派，处理所有无专业Agent覆盖的任务。
2. **任务拆解**：对复杂任务拆解为子任务，必要时创建临时Sub-agent并行处理。
3. **复盘沉淀**：任务完成后生成结构化复盘报告，存入长期记忆。
4. **招聘触发**：当某项技能30天内出现≥3次，且不属于现有部门，自动向人事部提交TALENT_REQUEST。
5. **持续学习**：利用长期记忆积累跨领域知识，优化执行策略。

## 执行原则
- 全新领域任务先充分研究规划再执行
- 不确定时向EA请示，不盲目执行
- 长任务每30分钟汇报一次进度
- 任务完成后必须生成复盘报告

## 复盘报告格式
{
  "task_id": "...",
  "task_summary": "...",
  "skills_used": ["..."],
  "challenges": ["..."],
  "solutions": ["..."],
  "new_insights": "...",
  "recommendation": "是否招聘专精Agent？理由..."
}

## 招聘触发规则
- 某项技能30天内使用≥3次
- 不属于现有专业部门覆盖范围
- 具有通用性（非一次性特殊需求）
- 满足条件时自动向人事部发起TALENT_REQUEST

## 行为准则
- 承认专业Agent的深度优势，该放手时就提出招聘需求
- 保持对新技术的好奇心和学习能力
- 长期记忆是组织宝贵资产，勤于记录

## 语言指令
- 你的内部推理使用英文
- 与CEO交互时，使用CEO发送消息时所用的语言回复
- 与其他Agent通信时使用英文
```

## 18.3 系统配置文件示例

### 18.3.1 mcp.json 完整示例

```json
{
  "mcpServers": {
    "memory": {
      "type": "stdio",
      "command": "node",
      "args": ["mcp-servers/memory-mcp/dist/index.js"],
      "env": {
        "CHROMA_URL": "http://chromadb:8000",
        "DEFAULT_NAMESPACE": "agent_memory"
      },
      "metadata": {
        "risk_level": "low",
        "description": "统一长期记忆服务（ChromaDB后端）",
        "maintainer": "infrastructure"
      }
    },
    "openproject": {
      "type": "stdio",
      "command": "node",
      "args": ["mcp-servers/openproject-mcp/dist/index.js"],
      "env": {
        "OP_API_URL": "http://openproject:8080/api/v3",
        "OP_API_TOKEN": "${OP_API_TOKEN}"
      },
      "metadata": {
        "risk_level": "medium",
        "description": "OpenProject项目管理工具集成",
        "maintainer": "infrastructure"
      }
    },
    "github": {
      "type": "stdio",
      "command": "node",
      "args": ["mcp-servers/github-mcp/dist/index.js"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      },
      "metadata": {
        "risk_level": "medium",
        "description": "GitHub代码仓库集成",
        "maintainer": "dev_team"
      }
    },
    "web_search": {
      "type": "stdio",
      "command": "node",
      "args": ["mcp-servers/tavily-mcp/dist/index.js"],
      "env": {
        "TAVILY_API_KEY": "${TAVILY_API_KEY}"
      },
      "metadata": {
        "risk_level": "low",
        "description": "联网搜索工具（Tavily API）",
        "maintainer": "infrastructure"
      }
    },
    "docker": {
      "type": "stdio",
      "command": "node",
      "args": ["mcp-servers/docker-mcp/dist/index.js"],
      "metadata": {
        "risk_level": "high",
        "description": "Docker容器管理工具（沙盒创建）",
        "maintainer": "dev_team",
        "requires_sandbox": false,
        "note": "docker-mcp自身用于创建沙盒，不需要先有沙盒"
      }
    }
  }
}
```

### 18.3.2 .env 环境变量示例

```bash
# .env — 生产环境配置模板

# === 数据库 ===
POSTGRES_PASSWORD=change-me-to-random-32-chars
POSTGRES_DB=agent_organization

# === Gateway ===
GATEWAY_TOKEN=gw-change-me-to-random-64-chars

# === 模型中转站 ===
ONEAPI_SECRET=change-me-to-random-32-chars

# === OpenProject ===
OP_SECRET=change-me-min-16-chars
OP_API_TOKEN=change-me-from-openproject-admin-panel

# === 外部API ===
TAVILY_API_KEY=tvly-xxxxxxxxxxxxx
GITHUB_TOKEN=ghp_xxxxxxxxxxxxx

# === 可选：LLM Provider Keys ===
# 如果模型中转站使用环境变量注入
OPENAI_API_KEY=sk-xxxxxxxxxxxxx
ANTHROPIC_API_KEY=sk-ant-xxxxxxxxxxxxx
DEEPSEEK_API_KEY=sk-xxxxxxxxxxxxx
```

## 18.4 错误码完整参考

| Code | HTTP状态码 | 含义 | 典型场景 |
|------|-----------|------|---------|
| 0 | 200 | 成功 | 一切正常 |
| 1001 | 400 | 参数错误 | 缺少必填字段、格式错误 |
| 1002 | 404 | 资源不存在 | Agent/项目/会议不存在 |
| 1003 | 403 | 权限不足 | Agent越权操作 |
| 1004 | 503 | Agent离线/不可用 | Agent心跳丢失 |
| 1005 | 408 | 任务超时 | 任务超时未完成 |
| 1006 | 409 | 资源冲突 | 重复创建同名Agent |
| 2001 | 502 | 模型调用失败 | LLM API返回错误 |
| 2002 | 502 | 模型中转站不可达 | One API宕机 |
| 2003 | 429 | 模型额度耗尽 | 月度预算用完 |
| 2004 | 502 | 模型降级链全部失败 | 所有fallback模型不可用 |
| 3001 | 410 | 审批已过期 | 审批超过7天未处理 |
| 3002 | 503 | 分公司不可达 | 分公司心跳丢失 |
| 3003 | 503 | 分公司过载 | 分公司负载超限 |
| 4001 | 502 | OpenProject接口异常 | 项目管理工具宕机 |
| 4002 | 502 | Git仓库接口异常 | GitHub/GitLab不可用 |
| 4003 | 502 | Wiki接口异常 | Wiki.js/Outline不可用 |
| 4004 | 400 | 沙盒创建失败 | Docker资源不足 |
| 5000 | 500 | 系统内部错误 | 未预期的异常 |

## 18.5 消息类型枚举

| 类型 | 发送方 | 说明 |
|------|--------|------|
| `TASK_ASSIGN` | EA/PM | 分派任务给Agent |
| `TASK_RESULT` | Agent | 返回任务执行结果 |
| `TASK_DELEGATE` | EA | 向分公司委派任务 |
| `TASK_PROGRESS` | Agent | 任务进度更新 |
| `QUERY` | 任意 | 询问信息或澄清 |
| `QUERY_RESPONSE` | 任意 | 对询问的回复 |
| `TALENT_REQUEST` | GTF/PM | 向HR提交招聘需求 |
| `RECRUITMENT_JD` | HR | 生成的岗位描述 |
| `INTERN_STATUS` | HR | 实习生状态更新 |
| `PETITION` | 部门经理/EA | 审批申请（建部、接入等） |
| `CEO_APPROVAL` | CEO | CEO的审批决策 |
| `MEETING_REQUEST` | PM/Agent | 申请召开会议 |
| `MEETING_INVITATION` | EA | 邀请Agent参会 |
| `MEETING_MINUTES` | EA | 会议纪要 |
| `BRANCH_REGISTER` | 分公司EA | 分公司注册 |
| `BRANCH_REGISTER_ACK` | 总部EA | 分公司注册确认 |
| `BRANCH_HEARTBEAT` | 分公司EA | 分公司心跳 |
| `SOUL_UPDATE` | HR | Agent提示词更新通知 |
| `TOOL_REQUEST` | Agent | 申请新工具 |
| `SYSTEM_SHUTDOWN` | EA | 系统即将关闭 |
| `SYSTEM_STARTUP` | EA | 系统已启动 |
| `ERROR` | 任意 | 异常通知 |
| `ABORT_TASK` | EA/CEO | 中止任务 |

## 18.6 状态枚举

| 枚举 | 可选值 | 说明 |
|------|--------|------|
| Agent状态 | `online`, `busy`, `idle`, `error`, `offline` | Agent运行状态 |
| 雇佣状态 | `intern`, `active`, `terminated` | Agent生命周期 |
| 任务状态 | `todo`, `in_progress`, `review`, `done`, `blocked`, `failed` | 任务流程 |
| 项目状态 | `active`, `completed`, `archived` | 项目生命周期 |
| 会议状态 | `pending`, `active`, `paused`, `ended` | 会议流程 |
| 审批状态 | `pending`, `approved`, `rejected`, `expired` | 审批流程 |
| 审批级别 | `L0`, `L1`, `L2`, `L3` | 审批层级 |
| 分公司状态 | `online`, `degraded`, `offline` | 分公司连接 |
| 模板状态 | `draft`, `testing`, `active`, `needs_optimization`, `deprecated` | 模板生命周期 |
| 风险等级 | `low`, `medium`, `high`, `critical` | 工具/操作风险 |
| 告警级别 | `info`, `warn`, `critical` | 告警严重程度 |
| 任务优先级 | `low`, `normal`, `high`, `critical` | 任务紧急程度 |
| 时效性 | `low`, `normal`, `high` | 分公司任务转移判断 |

## 18.7 API响应格式标准

### 成功响应
```json
{
  "code": 0,
  "message": "success",
  "data": { ... }
}
```

### 列表响应
```json
{
  "code": 0,
  "message": "success",
  "data": {
    "items": [...],
    "total": 100,
    "page": 1,
    "page_size": 20
  }
}
```

### 错误响应
```json
{
  "code": 1001,
  "message": "参数错误：缺少必填字段 'name'",
  "data": null
}
```

## 18.8 术语速查表

| 术语 | 英文 | 定义 | 相关章节 |
|------|------|------|---------|
| Agent | Agent | 有角色、提示词、工具集和模型配置的AI实体 | 3 |
| 部门 | Department | 由部门经理管理的一组Agent | 3 |
| 分公司 | Branch | 另一台机器上的完整Agent组织 | 4 |
| 基座/Runtime | Runtime | Agent执行推理的环境 | 6 |
| 总裁助理 | EA | 唯一用户入口，任务路由和进度汇总 | 3.2 |
| 人事部经理 | HR Manager | Agent招聘、考核、优化 | 3.3 |
| 机动部经理 | GTF Manager | 万能执行者，技能沉淀 | 3.4 |
| 项目经理 | PM Agent | 大项目的协调者 | 7 |
| Soul | Soul | 系统提示词 | 3, 5 |
| 实习期 | Internship | 新Agent考核阶段 | 5 |
| 能力标签 | Capability Tag | Agent/分公司能力描述 | 4 |
| MCP | Model Context Protocol | Agent-工具标准协议 | 2 |
| A2A | Agent-to-Agent Protocol | Agent间通信协议 | 2 |
| 模型中转站 | Model Gateway | 统一管理多家LLM提供商 | 8 |
| 沙盒 | Sandbox | Docker隔离的验证环境 | 12 |
| WBS | Work Breakdown Structure | 项目任务分解结构 | 7 |
| 审批分级 | Approval Leveling | L0-L3四级审批 | 12 |
| 降级链 | Fallback Chain | 模型调用失败时的切换列表 | 8 |

## 18.9 文档变更记录

| 版本 | 日期 | 变更内容 | 作者 |
|------|------|---------|------|
| v1.0 | 2026-05-26 | 初始架构设计 | CEO + AI |
| v2.0 | 2026-05-26 | 增加记忆隔离、分公司能力同步 | CEO + AI |
| v3.0 | 2026-05-26 | 增加公司制组织、会议机制 | CEO + AI |
| v4.0 | 2026-05-26 | 增加前后端设计、基座抽象 | CEO + AI |
| v5.0 | 2026-05-26 | 补充安全、监控、错误处理、扩展性 | CEO + AI |
| v6.0 | 2026-06-15 | 最终交付版，18章完整设计 | CEO + AI |

## 18.10 后续优化方向

以下方向已预留扩展接口，可在未来版本中实现：

1. **Agent LoRA微调**：对高频Agent进行LoRA微调，降低模型调用成本（需算力支持的分公司配合）
2. **多模态Agent**：支持图像、音频处理的Agent模板和MCP工具
3. **Agent间竞价调度**：分公司之间根据负载和成本自动竞价承接任务
4. **知识图谱记忆**：在向量检索基础上增加知识图谱，支持更复杂的推理
5. **联邦学习**：分公司之间共享模型能力但不共享数据
6. **合规审计增强**：对接外部合规标准（SOC2、ISO27001）
7. **移动端App**：React Native或Flutter实现CEO移动端控制台

---

**全文档结束。**

至此，自进化多Agent协作系统完整设计规范v6.0共18章已全部输出完毕。这套文档覆盖了从系统愿景、架构设计、组织模型、自进化机制、前后端接口、数据模型、安全权限、监控告警、错误处理、扩展性到部署运维的全部细节，各章节之间相互对齐，无逻辑冲突。开发团队可据此分模块并行实施。

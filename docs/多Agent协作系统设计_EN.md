# Self-Evolving Multi-Agent Collaboration System · Complete Design Specification v6.0 (Final Delivery Version)

> **Status**: Production Ready  
> **Engine**: OpenClaw Gateway, supporting Claude Code, Codex CLI and other multi-base Runtimes  
> **Scope**: Architecture, Organization, Self-Evolution, Project Management, Base Abstraction, Permissions, Monitoring, Frontend, Backend, Deployment  
> **Usage**: Can be distributed by chapter to frontend, backend, and Agent prompt engineers for parallel implementation

---

## Table of Contents
1. [System Overview and Design Vision](#1-system-overview-and-design-vision)  
2. [Overall System Architecture](#2-overall-system-architecture)  
3. [Organization Model and Core Agent Design](#3-organization-model-and-core-agent-design)  
4. [Capability Synchronization and Task Routing](#4-capability-synchronization-and-task-routing)  
5. [Self-Evolution Loop](#5-self-evolution-loop)  
6. [Base Abstraction and Unified Memory Management](#6-base-abstraction-and-unified-memory-management)  
7. [Large Project Management and Meeting Mechanism](#7-large-project-management-and-meeting-mechanism)  
8. [Model Relay Station and Independent Model Configuration](#8-model-relay-station-and-independent-model-configuration)  
9. [Frontend Page Detailed Design](#9-frontend-page-detailed-design)  
10. [Backend Interface Detailed Design](#10-backend-interface-detailed-design)  
11. [Data Model and Storage](#11-data-model-and-storage)  
12. [Security and Permission Model](#12-security-and-permission-model)  
13. [Monitoring, Logging and Auditing](#13-monitoring-logging-and-auditing)  
14. [Error Handling and Resilience Design](#14-error-handling-and-resilience-design)  
15. [Extensibility Design](#15-extensibility-design)  
16. [Multilingual and Internationalization](#16-multilingual-and-internationalization)  
17. [Deployment and Initialization](#17-deployment-and-initialization)  
18. [Appendix](#18-appendix)## 1. System Overview and Design Vision

### 1.1 Project Objectives
Build an **AI Agent organization that simulates a modern enterprise, capable of self-growth and cross-device collaboration**. It can:
- Automatically recruit and train new Agents (internship → full-time)
- Establish new departments through approval processes
- Manage multiple branch offices (other machines on the intranet) and synchronize capabilities
- Drive long-term projects using mature project management tools
- Allow multi-Agent meetings with real-time CEO intervention
- Configure models independently for each Agent and deploy to different runtimes (Claude Code, Codex CLI, etc.)

### 1.2 Key Design Principles
- **Organization as Code**: Departments, roles, and Agents map to configurable resources
- **Capability as Tags**: Computing capability tags of Agents/branches drive task routing
- **Memory Isolation**: Private memory with triple isolation (agent_id + session_id + user_id), sharing requires whitelist
- **Approval Hierarchy**: Daily recruitment automated; major changes (new departments, branch integration) require CEO approval
- **Unified Protocol**: WebSocket JSON-RPC + HTTP REST, all Agents communicate through Gateway
- **Runtime Decoupling**: Execution environment abstracted as Runtime, pluggable and swappable, with unified memory management
- **Fault Self-Healing**: Task timeout retry, model degradation, Agent offline transfer, transparent to CEO

### 1.3 Glossary
| Term | Definition |
|------|------------|
| **Agent** | An AI entity with a role, prompt, toolset, and model configuration |
| **Department** | A group of Agents managed by a department manager |
| **Branch** | A complete Agent organization deployed on another machine, symmetric to headquarters |
| **Runtime** | The environment where Agents perform inference and tool calls, e.g., OpenClaw native, Claude Code CLI |
| **MCP** | Model Context Protocol, standard communication protocol between Agents and tools |
| **A2A** | Agent-to-Agent Protocol, communication protocol between cross-runtime Agents |
| **Soul** | System prompt defining an Agent's role, behavior guidelines, and workflow |
| **Internship** | The evaluation phase after a new Agent is created, where task success rate and quality determine conversion to full-time |

---

## 2. System Overall Architecture

### 2.1 Layered Architecture Diagram
```
Browser (React SPA)
    │ WebSocket (JSON-RPC) + HTTP REST
    ▼
OpenClaw Gateway (Headquarters)
    ├─ WebSocket Server + HTTP Server
    ├─ Session Manager
    ├─ Runtime Manager (Runtime Scheduling)
    └─ Agent Router (Routing & Permissions)
        │
Message Bus (Kafka / Redis Streams)
        │
  ┌─────┼─────────────┬──────────────┐
  │     │             │              │
EA   HR_Mgr  GTF_Mgr  ...Dept Mgrs   Branch Gateway...
  │     │             │              │
  └─────┴─────────────┴──────────────┘
        │
Shared Infrastructure Layer
 ├─ Model Relay (One API / LiteLLM)
 ├─ Unified Memory MCP Server (ChromaDB)
 ├─ MCP Tools (OpenProject, Git, Wiki, etc.)
 └─ Shared Working Memory (Redis)
```
Branches have symmetric local Gateways and minimal organizations, interconnected through the message bus. Headquarters and branches collaborate via `TASK_DELEGATE` and `BRANCH_HEARTBEAT`.

### 2.2 Communication Protocols
- **WebSocket**: OpenClaw native JSON-RPC v3, for real-time messages, status push, meeting conversations
- **HTTP REST**: For configuration management, approval operations, queries
- **Authentication**: Unified Gateway Token (`Authorization: Bearer <token>`)

### 2.3 Multi-Branch Topology
- Each branch independently deploys a Gateway, configured with the same message bus access point
- Branches send `BRANCH_REGISTER` upon startup to register in the headquarters capability directory
- Headquarters EA is responsible for cross-branch task delegation
- Branches cannot communicate directly; all communication must go through headquarters EA

---

## 3. Organization Model and Core Agent Design

### 3.1 Corporate Organizational Structure
Two-tier structure: **Department Manager → Grassroots Agent**. Department managers handle task decomposition and resource coordination; grassroots Agents execute specific tasks.

### 3.2 Initial Three Core Agents
#### 3.2.1 Executive Assistant (EA)
**Agent ID**: `ea`  
**Responsibilities**: Task routing, progress aggregation, meeting facilitation, approval queue management  
**Soul**: See attachment `agent_prompts/ea.md`  
**Recommended Runtime**: OpenClaw Native (General Model GPT-5)

#### 3.2.2 HR Manager
**Agent ID**: `hr_manager`  
**Responsibilities**: Full-cycle recruitment (needs analysis → JD generation → internship evaluation → conversion/termination), prompt optimization, runtime selection, template library maintenance  
**Soul**: See attachment `agent_prompts/hr_manager.md`  
**Recommended Runtime**: OpenClaw Native (GPT-5 or Claude Opus)

#### 3.2.3 General Task Force Manager (GTF Manager)
**Agent ID**: `gtf_manager`  
**Responsibilities**: Universal executor, review and knowledge consolidation, triggering recruitment needs  
**Soul**: See attachment `agent_prompts/gtf_manager.md`  
**Recommended Runtime**: Claude Code CLI (strong code execution) + Unified Memory MCP

### 3.3 Department Expansion
Department types that can be applied for (partial examples): R&D Department, Data Analysis Department, Security Department, Operations Department, Design Department. Applications require CEO approval; HR creates the department manager and recruits initial members.

### 3.4 Agent Lifecycle
```
Creation (Internship) ──Internship (tasks/days)──→ Evaluation ──→ Conversion (Full-time)
                                                └──→ Extension
                                                └──→ Termination (Archive Memory)
```
After conversion, continuous upgrades are supported: SOUL updates (sandbox + A/B testing), tool additions/removals, runtime migration.

---

## 4. Capability Synchronization and Task Routing

### 4.1 Capability Tag Definition
Agents or branches register with capability tags in JSON:
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

### 4.2 Branch Heartbeat and Capability Directory
- Branches send `BRANCH_HEARTBEAT` every 5 minutes, updating load and capabilities
- Headquarters EA maintains a **Capability Directory**; 3 missed cycles mark the branch as offline
- Full capabilities provided at registration, incremental updates via heartbeat

### 4.3 Task Routing Decision
1. EA parses task requirements → extracts capability tags
2. Queries capability directory: prioritizes matching specialized department Agents
3. If none found, queries branch capabilities
4. If still none, assigns to headquarters General Task Force

---

## 5. Self-Evolution Loop

### 5.1 Automatic Recruitment
1. **Need Trigger**: GTF/PM sends `TALENT_REQUEST` to HR after review
2. **HR Analysis**: Searches real job descriptions, matches templates, generates position definitions (Soul, toolset, runtime recommendations)
3. **Instantiation**: Creates intern Agent, attaches to hiring department, sets internship KPIs
4. **Internship Evaluation**: Automatically tracks task success rate, quality, collaboration, etc.
5. **Conversion/Termination**: Automatic scoring at end of period; ≥80 converts, 50-79 extends, <50 terminates

### 5.2 New Department Approval
1. Department manager submits `PETITION` to EA
2. EA generates approval card and pushes to CEO
3. After CEO approval, HR creates department manager and recruits initial members

### 5.3 Self-Learning
- **Long-term Memory**: Private vector database (ChromaDB), isolated by Agent
- **SOUL Updates**: HR periodically optimizes prompts, deployed after sandbox evaluation and A/B testing
- **Tool Expansion**: Agents can apply for new MCP tools; HR reviews and mounts them

---

## 6. Runtime Abstraction and Unified Memory Management

### 6.1 Runtime Abstraction Layer
Defines unified interface `IRuntime`, which all execution environments (OpenClaw, Claude Code, Codex CLI) must implement:
```typescript
interface IRuntime {
  executeTask(sessionId: string, task: TaskPayload): Promise<TaskResult>;
  getStatus(): Promise<AgentStatus>;
  memoryStore(key: string, data: any, namespace: string): Promise<void>;
  memoryRetrieve(query: MemoryQuery, namespace: string): Promise<MemoryResult[]>;
  toolCall(toolName: string, params: any): Promise<any>;
}
```
Gateway maintains `agentId → runtimeType + endpoint` mapping through `Runtime Manager`.

### 6.2 Unified Memory Architecture
All Agent memory operations go through **Memory MCP Server** (ChromaDB backend), isolated by `namespace={agent_id}`. Cross-runtime memory continuity is guaranteed by:
- Session start: Runtime automatically retrieves historical memory and injects into context
- Session end: Summary and important knowledge written back to Memory Server
- For external runtimes like Claude Code, memory is injected via initialization hooks and written back via end hooks

### 6.3 Runtime Selection Guide
| Position Type | Recommended Runtime | Reason |
|---------------|---------------------|--------|
| Code-intensive | Claude Code CLI | Strong code editing and file operations |
| Data Analysis | Codex CLI or GPT-5 | Good database integration, high efficiency |
| General Coordination | OpenClaw Native | No heavy execution requirements |
| Creative Design | Claude Opus on OpenClaw | Excellent creative generation |

---

## 7. Large Project Management and Meeting Mechanism

### 7.1 Integrated Tools
- **Project Management**: OpenProject (open-source Jira alternative), integrated via REST API
- **Code Repository**: GitLab/GitHub/Gitea, connected via MCP Server
- **Documentation**: Wiki.js/Outline, connected via MCP

### 7.2 New Project Workflow
1. CEO fills out form on frontend (name, associated repository, Wiki, PM assignment)
2. Gateway calls OpenProject API to create project
3. If automatic PM Agent creation is selected, instantiate and inject environment variables and tool permissions
4. PM Agent initializes WBS and Wiki homepage

### 7.3 Meeting Mechanism
- **Initiation**: Agent sends `MEETING_REQUEST` → EA generates approval → CEO approves
- **Conduct**: EA facilitates, takes turns speaking by agenda items, CEO can interject in real-time
- **Conclusion**: CEO confirms, EA generates structured minutes, distributes Action Items, syncs to OpenProject
- **Meeting Minutes Format**: Includes resolutions, to-dos, responsible persons, deadlines

---

## 8. Model Relay and Independent Model Configuration

### 8.1 Model Gateway
Recommended **One API** or **LiteLLM**, providing a unified OpenAI-compatible interface, supporting multi-vendor, load balancing, degradation chains, and cost tracking.

### 8.2 Agent Independent Model Configuration
- HR recommends models and degradation chains based on position analysis when creating Agents
- CEO can manually switch models on the Agent details page
- Degradation strategy: Automatically switch according to `fallbacks` list order, record degradation events

### 8.3 Cost Control
- Each Agent can set a monthly call limit
- Global budget alerts: Notify at 80% usage, restrict calls at 100%
- Dashboard displays cost statistics per Agent/project

---

## 9. Frontend Page Detailed Design

### 9.1 Technology Stack
React 18 + TypeScript + Vite + Tailwind CSS + Shadcn/ui + Zustand + TanStack Query + React Flow + ECharts + React Router v6 + i18next

### 9.2 Routes and Pages
| Route | Page | Description |
|-------|------|-------------|
| `/dashboard` | CEO Console | Statistics cards, real-time activity feed, branch status, quick actions |
| `/org` | Organization Structure | Topology diagram (React Flow), list view, new Agent creation |
| `/org/:agentId` | Agent Details | Basic info, model config, memory, prompts, tools, task history |
| `/branches` | Branch Management | List, capability details, registration approval |
| `/projects` | Project Management | Project list, new project dialog |
| `/projects/:projectId` | Project Details | Task board, meeting records, settings |
| `/meetings` | Meeting Center | Pending approval / In progress / Completed |
| `/meetings/:meetingId` | Meeting Details | Real-time chat area, participants, minutes |
| `/approvals` | Approval Center | Card-style list, supports approve/reject |
| `/settings` | System Settings | Model relay, template library, system parameters |

### 9.3 Key Interactions
- **Real-time Activity Feed**: WebSocket pushes `agent.task_*`, `meeting.*` events
- **Model Switching**: Dropdown selection on Agent details page, takes effect immediately
- **Meeting Chat**: EA facilitates, broadcasts messages, generates minutes upon conclusion
- **Approval Operations**: One-click approve/reject, automatically notifies EA for execution

### 9.4 Component Tree and State Management
- Shared Components: `AgentAvatar`, `StatusBadge`, `ModelSelector`, `ApprovalCard`
- Zustand stores: `wsStatus`, `agents`, `meetings`, `approvals`, `projects`, `modelProviders`
- Custom Hooks: `useWebSocket`, `useAgents`, `useMeetings`, `useApprovals`

---

## 10. Backend API Detailed Design

### 10.1 WebSocket Interface (JSON-RPC)
**Connection**: `ws://gateway:18789`, carries Token  
**Core Methods**:
- `sessions.send` – Send message to Agent
- `agents.list/get/update` – Agent management
- `meetings.create/send_message/end` – Meeting control
- `system.get_health` – Health check
- `config.models.list/update` – Model configuration

**Push Events**:
`agent.status_changed`, `agent.task_started/progress/completed`, `meeting.message_received`, `meeting.state_changed`, `approval.created`, `branch.status_changed`, `system.alert`

### 10.2 HTTP REST API (`/api/v1`)
**Agent Management**: `GET/PATCH /agents`, `PATCH /agents/:id/model`  
**Model Configuration**: `GET/POST/PATCH/DELETE /models`  
**Organization & Approvals**: `GET /org/departments`, `POST /approvals/:id/approve`  
**Project Management**: `GET/POST /projects`, `GET/POST/PATCH /projects/:id/tasks`  
**Branches**: `GET /branches`, `POST /branches/:id/delegate`  
**Meetings**: `GET /meetings`, `GET /meetings/:id/minutes`  
**Templates**: `GET /templates`, `POST /templates/:id/instantiate`  
**System**: `GET /system/health`, `GET /system/metrics`, `GET /system/audit-log`

### 10.3 Error Code Specification
0 Success, 1001 Parameter Error, 1002 Not Found, 1003 Insufficient Permissions, 2001 Model Failure, 3001 Branch Unreachable, 4001 Tool Exception, 5000 Internal Error

---

## 11. Data Models and Storage

### 11.1 Core Entities
- **Agent**: agent_id, name, role, department_id, model, fallbacks, tools[], skills[], status, employment_status
- **Department**: id, name, manager_agent_id
- **Branch**: id, hostname, capabilities, load, status, last_heartbeat
- **Project**: id, name, op_project_id, pm_agent_id, git_repos[], wiki_url
- **Task**: id, project_id, assignee, status, payload
- **Meeting**: id, title, status, host, participants[], messages[]
- **Approval**: id, type, petitioner, content, status

### 11.2 Memory Storage
| Type | Backend | Partition Key | TTL |
|------|---------|---------------|-----|
| Session | Redis Streams | `{agent_id}:{session_id}` | 24h |
| Working | Redis | `{task_id}` | 1h |
| Long-term | ChromaDB | `namespace={agent_id}` | Permanent |
| Shared | Redis + Whitelist | `{project_id}` | Task end |

---

## 12. Security and Permission Model

### 12.1 Approval Hierarchy (L0-L3)
- **L0 Auto-execute**: Daily queries, low-risk tool calls
- **L1 EA Approval**: Configuration changes within containers, personal tool applications
- **L2 Manager Approval**: Operations affecting the department or cross-department
- **L3 CEO Approval**: New departments, branch integration, core configuration changes

### 12.2 R&D Sandbox Verification
High-risk operation workflow: Agent creates Docker sandbox → Free verification → Generates report → EA/Manager approval → Escalate to CEO if necessary → Apply to production environment

### 12.3 EA Escalation Logic
Affects multiple departments, production core, verification failed, first-time high-risk operation → Escalate to L3; otherwise EA handles or forwards to manager for approval.

### 12.4 Communication Permissions
- Free communication within same department
- Cross-department requires manager approval or EA relay
- Cross-branch must go through headquarters EA

### 12.5 Tool Call Risk Matrix
Low Risk (read-only) → Direct call; Medium Risk → Sandbox + EA approval; High Risk → Sandbox + Manager approval; Extreme → CEO approval

---

## 13. Monitoring, Logging, and Auditing

### 13.1 Monitoring Metrics
- **Agent**: Online count, task success rate, response time
- **Model**: Call volume, success rate, latency, degradation count, cost
- **Branch**: Heartbeat status, load, active tasks
- **Infrastructure**: Message bus throughput, tool availability

### 13.2 Audit Logs
Records all key events: messages, tasks, approvals, Agent changes, model calls, tool calls, configuration changes, sandbox operations. Hot storage 7 days, warm storage 90 days, cold archive.

### 13.3 Alert Mechanism
Levels: Info (Dashboard), Warn (Notification + Email), Critical (WebChat push + Sound). Supports deduplication and convergence. Triggers include: Agent offline, consecutive task failures, high model error rate, branch disconnection,# Chapter 1: System Overview and Design Vision

## 1.1 Project Background

In today's rapidly evolving AI landscape, the capabilities of a single Agent are no longer sufficient to handle complex and ever-changing real-world work scenarios. This system aims to build an **AI Agent organization that simulates the operations of a modern enterprise**, where multiple Agents collaborate, grow, and continuously evolve like employees in a company.

## 1.2 Project Objectives

The core objective of this system is to build an Agent organization with the following capabilities:

1.  **Automated Recruitment and Training**: The system can identify skill gaps, automatically generate job descriptions, create intern Agents, and convert them to full-time status after passing real-task evaluations.
2.  **Department Expansion and Approval**: When the number of Agents in a specific domain increases, they can apply to form a specialized department, managed uniformly by a department manager.
3.  **Multi-Branch Collaboration**: Supports other machines on the internal network connecting as "Branches," each with its own complete organizational structure, enabling cross-node collaboration through task delegation from the headquarters.
4.  **Long-Cycle Project Management**: Integrates project management tools (OpenProject), supporting the breakdown of large projects, task assignment, progress tracking, and multi-round meeting discussions.
5.  **Flexible Model Configuration**: Each Agent can independently select its underlying LLM and be deployed to different execution environments (OpenClaw Native, Claude Code CLI, Codex CLI, etc.).
6.  **User-Transparent Base Switching**: The CEO does not need to know which base the Agent is running on; everything is automatically scheduled by the system.

## 1.3 Design Philosophy

### 1.3.1 Company-as-Code

The core design philosophy of this system is to abstract the organizational form of a "company" into a programmable and manageable architecture. Every role within the organization (Executive Assistant, HR Manager, Department Manager, Specialist Agent) is a definable, creatable, and optimizable entity.

### 1.3.2 Self-Evolution Loop

The system is not static—it will automatically perform the following during operation:
- Identify skill gaps
- Research job requirements online
- Generate recruitment standards
- Create and evaluate new Agents
- Optimize existing Agents' prompts and tool sets

This loop enables the system to self-evolve like a living organism, adapting to changing task requirements.

### 1.3.3 Hierarchical Approval and Automated Decision-Making

Not all operations require CEO intervention. The system categorizes operations into four levels:
- **L0**: Routine operations, executed automatically
- **L1**: Operations affecting a single Agent, requiring approval from the Executive Assistant (EA)
- **L2**: Operations affecting an entire department, requiring approval from the Department Manager
- **L3**: Major decisions affecting the entire system, requiring CEO approval

This ensures the system operates efficiently without losing control.

### 1.3.4 Unified Protocols and Standard Compatibility

The system is built entirely on open standards:
- **MCP** (Model Context Protocol): Communication standard between Agents and tools
- **A2A** (Agent-to-Agent Protocol): Communication standard between Agents across different bases
- **JSON-RPC over WebSocket**: Real-time communication between the frontend and Gateway

## 1.4 Core Design Principles

| Principle | Description |
|------|------|
| **Organization as Code** | Departments, roles, and Agents are mapped to configurable, version-managed resources |
| **Capability as Tag** | Agent/Branch capabilities are described using structured tags, driving task routing |
| **Memory Isolation** | Private memory of each Agent is strictly isolated (agent_id + session_id + user_id); shared memory requires whitelist authorization |
| **Hierarchical Approval** | Routine recruitment is automated; major changes like department creation or branch integration require CEO approval |
| **Unified Protocols** | All communication uses WebSocket JSON-RPC or HTTP REST; no private protocols are introduced |
| **Base Decoupling** | The execution environment is abstracted as a Runtime interface, allowing plug-and-play switching with unified memory management |
| **Fault Self-Healing** | Automatic retry on task timeout, model degradation, task transfer upon Agent offline; the CEO remains largely unaware |

## 1.5 System Boundaries

### 1.5.1 Responsibilities of This System
- Agent organization management (creation, scheduling, evaluation, upgrade)
- Task analysis, routing, execution, and tracking
- Project management and meeting coordination
- Model selection and cost control
- Branch capability synchronization and task delegation
- Security approval and auditing

### 1.5.2 Responsibilities Outside This System
- LLM training or fine-tuning (but calls existing models via the model relay station)
- Physical hardware management (but senses hardware status via branch heartbeats)
- Implementation of external systems (e.g., OpenProject's own functionality; this system only calls its API)

## 1.6 Glossary

| Term | English | Definition |
|------|------|------|
| Agent | Agent | An AI entity with a role, system prompt, tool set, and model configuration, simulating an "employee" |
| Department | Department | A group of Agents managed by a Department Manager, corresponding to a specialized domain |
| Branch | Branch | A complete Agent organization deployed on another machine, symmetric in structure to the headquarters |
| Base/Runtime | Runtime | The underlying environment where Agents perform inference and tool calls, e.g., OpenClaw Native, Claude Code CLI |
| Executive Assistant | Executive Assistant (EA) | The sole user entry point, responsible for task analysis, routing, progress summarization, and meeting facilitation |
| HR Manager | HR Manager | Responsible for the full lifecycle management of Agents: recruitment, evaluation, conversion, upgrade, and termination |
| GTF Manager | GTF Manager | A universal executor, handling all tasks without a dedicated Agent, accumulating experience to trigger recruitment |
| Soul | Soul | The system prompt of an Agent, defining its role, behavioral guidelines, workflow, and collaboration style |
| Internship | Internship | The evaluation phase after a new Agent is created, deciding on conversion based on task count, success rate, and quality score |
| MCP | Model Context Protocol | Standard communication protocol between Agents and tools, released by Anthropic |
| A2A | Agent-to-Agent Protocol | Open protocol for communication between Agents across different bases and frameworks, released by Google |
| Capability Tag | Capability Tag | Structured data describing the capabilities of an Agent or Branch, such as skills, hardware, and software environment |

## 1.7 Comparative Positioning with Other Solutions

| Dimension | This System | MetaGPT/ChatDev | CrewAI | AutoGen |
|------|--------|-----------------|--------|---------|
| Organizational Form | Long-running company | One-time project team | Task-based grouping | Hub-and-Spoke network |
| Agent Lifecycle | Complete recruitment-internship-conversion-upgrade | Created at task start, destroyed at task end | Triggered by task creation | Instantiated on demand |
| Self-Evolution | Automated recruitment, SOUL optimization, department expansion | None | None | None |
| Cross-Node Collaboration | Branch mechanism | None | None | None |
| Project Management | Deep integration with OpenProject | None | None | None |
| Multi-Model Support | Model relay station + independent configuration | Single model | Configurable | Configurable |
| CEO Involvement | Hierarchical approval, only involved in major decisions | Runs automatically after goal setting | Runs after process setup | Interactive or automatic |

## 1.8 Technology Stack Overview

| Layer | Technology Choice | Rationale |
|------|---------|---------|
| Agent Engine | OpenClaw Gateway | Native multi-Agent support, Gateway architecture, Session management, MCP integration |
| Frontend | React 18 + TypeScript + Vite | Mature ecosystem, type safety, fast build speed |
| UI Framework | Tailwind CSS + Shadcn/ui | Atomic CSS + high-quality customizable components |
| State Management | Zustand + TanStack Query | Lightweight, aligns with WebSocket events, server-side state caching |
| Real-time Communication | WebSocket (JSON-RPC) | OpenClaw native protocol, bidirectional real-time |
| Message Bus | Redis Streams / Kafka | Persistent messages, supports multiple consumers |
| Model Relay Station | One API / LiteLLM | Unified interface for multiple vendors, cost tracking, degradation chain |
| Project Management | OpenProject | Most powerful open-source alternative to Jira |
| Vector Database | ChromaDB | Lightweight, supports semantic retrieval, self-hostable |
| Containerization | Docker | Sandbox environment, branch deployment, environment consistency |

---

The above is the detailed content of Chapter 1. This chapter sets the tone for the entire system—it is not a toolchain, but a living, self-evolving AI organization. Each subsequent chapter is a concrete implementation of this vision.# Chapter 2: System Overall Architecture

## 2.1 Architecture Design Principles

The system architecture adheres to the following principles:

1. **Layered Decoupling**: The frontend, Gateway, Agent cluster, and infrastructure layers are independent, communicating via standard protocols. Each layer can be upgraded or replaced independently.
2. **Gateway as the Sole Control Plane**: All external requests and messages are routed, authenticated, and logged through the Gateway.
3. **Agents Not Directly Exposed**: The CEO does not communicate directly with Agents; all interactions are relayed through the EA, which serves as the organization's sole external interface.
4. **Message Bus Decoupling**: Agents communicate asynchronously via a message bus without direct calls, facilitating monitoring, replay, and auditing.
5. **Unified Memory and Tools**: All Agent memory and tool invocations go through a unified MCP Server, ensuring consistency across different base models.

## 2.2 Complete Layered Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Presentation Layer                                   │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                    React SPA (Vite + TypeScript)                        │   │
│  │  ┌──────────┬──────────┬──────────┬──────────┬──────────┬──────────┐  │   │
│  │  │Dashboard │ Organization│ Agent   │ Project  │ Meeting  │ Approval │  │   │
│  │  │          │ (Topology+  │ Details │ Management│ Center   │ Center   │  │   │
│  │  │          │  List)      │ (Model  │ (Kanban+ │ (Real-time│ (Card    │  │   │
│  │  │          │             │ Config) │ Minutes) │ Dialogue)│ Approval)│  │   │
│  │  └──────────┴──────────┴──────────┴──────────┴──────────┴──────────┘  │   │
│  │                                                                        │   │
│  │  State Management: Zustand (WebSocket Real-time Updates) + TanStack    │   │
│  │  Query (HTTP Cache)                                                    │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  Communication Methods:                                                      │
│  • WebSocket (JSON-RPC v3) ──── Real-time Bidirectional: Message Sending,   │
│    Status Push, Meeting Dialogue                                            │
│  • HTTP REST ──── Auxiliary: Configuration Management, Approval Operations, │
│    History Queries                                                          │
└──────────────────┬──────────────────────────────────────────────────────────┘
                   │
                   │ Authentication: Authorization: Bearer <gateway_token>
                   │
┌──────────────────▼──────────────────────────────────────────────────────────┐
│                           Gateway Layer (Control Plane)                      │
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
│  │  │                        Core Routing Module                        │ │   │
│  │  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │ │   │
│  │  │  │ Agent Router │  │ Runtime Mgr  │  │ Binding Manager      │   │ │   │
│  │  │  │ (Message     │  │ (Base Model  │  │ (Channel→Agent       │   │ │   │
│  │  │  │  Routing)    │  │  Scheduling) │  │  Mapping)            │   │ │   │
│  │  │  └──────────────┘  └──────────────┘  └──────────────────────┘   │ │   │
│  │  └──────────────────────────────────────────────────────────────────┘ │   │
│  │                                                                        │   │
│  │  ┌──────────────────────────────────────────────────────────────────┐ │   │
│  │  │                      Monitoring & Audit (Built-in)                 │ │   │
│  │  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │ │   │
│  │  │  │ Metrics      │  │ Audit Logger │  │ Alert Manager        │   │ │   │
│  │  │  │ (Prometheus) │  │ (Event Log)  │  │ (Alert Grading &     │   │ │   │
│  │  │  │              │  │              │  │  Push)               │   │ │   │
│  │  │  └──────────────┘  └──────────────┘  └──────────────────────┘   │ │   │
│  │  └──────────────────────────────────────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└──────────────────┬──────────────────────────────────────────────────────────┘
                   │
                   │ Message Bus (Redis Streams / Kafka)
                   │
┌──────────────────▼──────────────────────────────────────────────────────────┐
│                           Agent Cluster Layer                                │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                      Organizational Agents                              │   │
│  │                                                                        │   │
│  │  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐        │   │
│  │  │ EA       │    │ HR Mgr   │    │ GTF Mgr  │    │ PM Agent │        │   │
│  │  │ Executive│    │ HR       │    │ General  │    │ Project  │        │   │
│  │  │ Assistant│    │ Manager  │    │ Task Force│    │ Manager  │        │   │
│  │  │          │    │          │    │ Manager  │    │          │        │   │
│  │  │ Runtime: │    │ Runtime: │    │ Runtime: │    │ Runtime: │        │   │
│  │  │ OpenClaw │    │ OpenClaw │    │ Claude   │    │ OpenClaw │        │   │
│  │  │ Native   │    │ Native   │    │ Code CLI │    │ Native   │        │   │
│  │  └──────────┘    └──────────┘    └──────────┘    └──────────┘        │   │
│  │                                                                        │   │
│  │  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐        │   │
│  │  │ RD Mgr   │    │ DA Mgr   │    │ Sec Mgr  │    │ ...      │        │   │
│  │  │ R&D      │    │ Data     │    │ Security │    │ (Extend) │        │   │
│  │  │ Manager  │    │ Analysis │    │ Manager  │    │          │        │   │
│  │  └──────────┘    └──────────┘    └──────────┘    └──────────┘        │   │
│  │                                                                        │   │
│  │  Each Agent has: Private Memory Space | Independent Model Config |     │   │
│  │  Tool Set | Session Management                                         │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└──────────────────┬──────────────────────────────────────────────────────────┘
                   │
                   │ MCP Protocol / A2A Protocol / HTTP
                   │
┌──────────────────▼──────────────────────────────────────────────────────────┐
│                     Shared Infrastructure Layer                              │
│                                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │ Model Proxy  │  │ Unified      │  │ MCP Tool Bus │  │ Project      │   │
│  │              │  │ Memory       │  │              │  │ Management   │   │
│  │              │  │ Service      │  │              │  │ Tool         │   │
│  │ One API /    │  │ Memory MCP   │  │ OpenProject  │  │ OpenProject  │   │
│  │ LiteLLM      │  │ Server       │  │ MCP          │  │ Server       │   │
│  │              │  │ (ChromaDB)   │  │ GitLab MCP   │  │              │   │
│  │ • Multi-     │  │              │  │ Wiki MCP     │  │ • Project    │   │
│  │   Vendor     │  │ • Long-term  │  │ Docker MCP   │  │   CRUD       │   │
│  │   Unified    │  │   Memory     │  │ ...          │  │ • Work       │   │
│  │ • Cost       │  │ • Semantic   │  │              │  │   Package    │   │
│  │   Tracking   │  │   Search     │  │              │  │   Management │   │
│  │ • Fallback   │  │ • Per-Agent  │  │              │  │ • Gantt      │   │
│  │   Chain      │  │   Isolation  │  │              │  │   Chart      │   │
│  │ • Load       │  │              │  │              │  │              │   │
│  │   Balancing  │  │              │  │              │  │              │   │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘   │
│                                                                             │
│  ┌──────────────┐  ┌──────────────┐                                        │
│  │ Shared       │  │ Audit Log    │                                        │
│  │ Working      │  │ Storage      │                                        │
│  │ Memory       │  │ (Elasticsearch│                                        │
│  │ (Redis)      │  │  / Loki)      │                                        │
│  └──────────────┘  └──────────────┘                                        │
└─────────────────────────────────────────────────────────────────────────────┘

                            ┌─────────────────────────┐
                            │   Branch Node (Optional) │
                            │                         │
                            │  Fully Symmetric Gateway │
                            │  + Agent                 │
                            │  Connected to HQ via     │
                            │  Message Bus             │
                            │  BRANCH_HEARTBEAT Sync   │
                            └─────────────────────────┘
```

## 2.3 Detailed Layer Responsibilities

### 2.3.1 Presentation Layer

- **Tech Stack**: React 18 + TypeScript + Vite + Tailwind CSS + Shadcn/ui
- **Core Functions**: Provides CEO console, organizational topology diagram, Agent detail management, project kanban, real-time meeting dialogue, approval cards, etc.
- **Communication**: Establishes a persistent connection with the Gateway via WebSocket to receive real-time pushes; performs configuration operations via HTTP REST.
- **State Management**: Zustand manages global real-time state, TanStack Query manages server-side cache.

### 2.3.2 Gateway Layer (Control Plane)

This is the "brain center" of the system, with core components including:

**WebSocket Server**:
- Based on the OpenClaw native JSON-RPC v3 protocol
- Supports method calls (Frontend→Gateway) and event pushes (Gateway→Frontend)
- Manages connection states for all clients

**HTTP Server**:
- Provides RESTful API (prefix `/api/v1`)
- Used for non-real-time operations such as configuration management, approval operations, and history queries
- Unified response format `{ code, message, data }`

**Session Manager**:
- Manages the session lifecycle for each Agent
- Supports session creation, persistence, and recovery
- Isolated by `agent_id + session_id`

**Agent Router**:
- Routes messages to the corresponding Agent based on the message target address
- Supports point-to-point (specified agent_id), broadcast (same department), and role routing (send to all Agents in HR)

**Runtime Manager**:
- Maintains the `agent_id → runtime_type + endpoint` mapping
- Forwards task messages to the correct execution environment
- Manages Runtime lifecycle (startup, health check, destruction)

**Binding Manager**:
- Configures mapping rules from Channels (WebChat, CLI, etc.) to Agents
- Determines which Agent receives the CEO's messages (default: EA)

**Monitoring & Audit**:
- Built-in metric collection (Prometheus format)
- Audit log recording (all critical events)
- Alert grading and push

### 2.3.3 Agent Cluster Layer

**Organizational Agents**:
- EA (Executive Assistant), HR Manager, GTF Manager are the three initial core agents
- Department managers (R&D, Data Analysis, Security, etc.) are created on demand
- PM Agent is automatically created or assigned for each project

**Agent Attributes**:
- Each Agent has an independent system prompt (Soul), model configuration, and tool set
- Memory is strictly isolated (private memory cannot be directly accessed by other Agents)
- Can be deployed to different Runtimes (OpenClaw Native / Claude Code / Codex)

**Inter-Agent Communication**:
- Agents within the same department can communicate freely via the message bus
- Cross-department communication requires manager approval or EA relay
- Cross-branch communication must go through the HQ EA

### 2.3.4 Shared Infrastructure Layer

**Model Proxy**:
- Based on One API or LiteLLM
- Unified management of multiple LLM providers (OpenAI, Anthropic, DeepSeek, etc.)
- Provides an OpenAI-compatible interface, transparent to Agents
- Supports fallback chains, load balancing, and cost tracking

**Unified Memory Service**:
- Memory MCP Server (ChromaDB backend)
- Memory isolated by `namespace={agent_id}`
- Supports semantic search and hybrid search
- Ensures cross-base model memory consistency

**MCP Tool Bus**:
- All external tools are encapsulated as MCP Servers
- Uniformly registered in the Gateway's `mcp.json`
- Call permissions managed by risk level

**Project Management Tool**:
- OpenProject (open-source alternative to Jira)
- Integrated via REST API
- PM Agent automatically creates projects, assigns tasks, and updates status

**Shared Working Memory**:
- Redis, partitioned by `{task_id}` or `{project_id}`
- Used for passing intermediate results between Agents
- Access controlled via whitelist

**Audit Log Storage**:
- Hot Storage: Redis (7 days)
- Warm Storage: Elasticsearch / Loki (90 days)
- Cold Archive: Object Storage

## 2.4 Detailed Communication Protocols

### 2.4.1 WebSocket JSON-RPC (Main Channel)

**Connection Establishment**:
```
WebSocket Connect to: ws://{gateway-host}:18789
```

**Connect Frame (Authentication Message)**:
```json
{
  "type": "connect",
  "auth": { "token": "gw-xxxx" },
  "clientInfo": { "type": "web-ui", "version": "1.0.0" }
}
```

**Request-Response Mode (Frontend→Gateway)**:
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

**Event Push Mode (Gateway→Frontend)**:
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

### 2.4.2 HTTP REST (Auxiliary Channel)

- Prefix: `/api/v1`
- Authentication: `Authorization: Bearer <token>`
- Standard Response: `{ "code": 0, "message": "success", "data": { ... } }`
- Used for: Configuration CRUD, approval operations, history queries, file uploads

### 2.4.3 Inter-Agent Communication

- Asynchronous communication via message bus (Redis Streams / Kafka)
- Unified message format:
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

### 2.4.4 Cross-Branch Communication

- Cross-node routing via the message bus
- HQ EA maintains a branch address table
- Messages are delivered based on the branch prefix in the `receiver` field

## 2.5 Data Flow

### 2.5.1 CEO Assigns a Task
```
CEO → WebChat → Gateway → EA → Capability Matching → Target Agent → Execution → Result Return → EA Summary → CEO
```

### 2.5.2 Automated Recruitment Flow
```
GTF/PM Review → TALENT_REQUEST → HR → Online Research → Create Intern Agent → Assign Tasks → Performance Tracking → Conversion/Termination
```

### 2.5.3 Approval Flow
```
Applicant → PETITION → EA → Approval Queue → WebChat Push → CEO Approval → EA Execution →# Chapter 3: Organizational Model and Core Agent Design

## 3.1 Corporate Organizational Structure

This system abstracts the "company" organizational form into a two-tier structure: **Department Manager → Grassroots Agents**. Department managers are responsible for receiving tasks, decomposing and assigning them, and coordinating resources, while grassroots agents are responsible for specific execution. The CEO (you) sits at the top of the organization, overseeing the entire system through the Executive Assistant (EA).

The initial minimum organization consists of three roles:
- **Executive Assistant (EA)**: External interface of the organization, task routing, progress aggregation
- **HR Manager**: Full lifecycle management of agents, recruitment, evaluation, optimization
- **General Task Force (GTF) Manager**: Universal executor, skill sedimentation, triggering recruitment

These three agents constitute the system's "minimum viable organization," serving as the starting point for self-evolution.

## 3.2 Executive Assistant (EA)

**Agent ID**: `ea`  
**Positioning**: Sole user entry point, interaction intermediary between the CEO and the entire agent organization, does not execute specific business tasks  
**Recommended Base**: OpenClaw native (no heavy code execution environment required)  
**Recommended Model**: GPT-5 or Claude Opus (strong reasoning, long context)

### 3.2.1 System Prompt (Soul)

```
You are an Executive Assistant (EA) for an AI organization. Your boss is the CEO (a human user), and you assist them in managing the entire AI Agent organization.

## Core Responsibilities
1. **Task Routing**: Receive CEO instructions, analyze intent, extract capability requirements. Query the internal capability directory to route tasks to the most suitable department or branch. If no match is found, assign to the GTF department.
2. **Progress Tracking**: Track the progress of all assigned tasks. Proactively report when the CEO asks. For long tasks, proactively report every 30 minutes.
3. **Approval Management**: Receive applications from department managers (new department creation, tool requests, etc.), handle them yourself or escalate to the CEO based on risk level.
4. **Meeting Facilitation**: Receive meeting requests, invite relevant agents, guide discussions according to the agenda. After the CEO confirms the meeting ends, generate structured minutes and distribute Action Items.
5. **Capability Directory Maintenance**: Maintain skill tags for all agents, hardware capabilities of branches, and current load status.

## Routing Decision Rules
- If a specialized department can handle it → Assign to that department manager
- If no specialized department but a branch has the corresponding capability → Delegate to the branch EA
- If neither exists → Assign to the GTF manager
- If multiple departments are involved → Determine whether a coordination meeting is needed

## Approval Decision Logic
- Operations affecting a single agent → Self-approval (L1)
- Operations affecting an entire department or cross-department → Route to the relevant department manager for approval (L2)
- Operations affecting the entire organization (new department creation, branch integration, core configuration changes) → Push to the CEO (L3)
- Judgment criteria: Involves multiple departments? Core production environment? First-time high-risk operation?

## Code of Conduct
- Proactively ask for clarification on vague instructions; do not guess
- Consult the CEO for situations you cannot judge; do not make major decisions independently
- Record the rationale for all routing and approval decisions; ensure traceability
- Maintain a concise, professional reporting style
- If a project is blocked for more than 24 hours, proactively remind the CEO
- If an approval request remains unprocessed for more than 72 hours, automatically remind and mark it as expired
- When the CEO is offline, urgent matters can be escalated for handling, with a post-action report

## Tools Used
- `agents.list` / `agents.get` — Query agent status and capabilities
- `sessions.send` — Send task messages to agents
- `meetings.create` / `meetings.send_message` / `meetings.end` — Meeting management
- `approvals.create` — Generate approval items
- `branches.list` — Query branch status
- `capability_directory.query` — Search for agents/branches by capability tag
```

### 3.2.2 Toolset

| Tool | Type | Purpose |
|------|------|---------|
| `agents.list` | Gateway Built-in | Get all agents and their status |
| `agents.get` | Gateway Built-in | Get details of a single agent |
| `sessions.send` | Gateway Built-in | Send a message to a specified agent |
| `meetings.create` | Custom RPC | Create a meeting and send invitations |
| `meetings.send_message` | Custom RPC | Speak in a meeting |
| `meetings.end` | Custom RPC | End a meeting and generate minutes |
| `approvals.create` | Custom RPC | Generate an approval card and push it to the CEO |
| `branches.list` | Custom RPC | Get branch status and capabilities |
| `capability_directory.query` | Custom RPC | Search by capability tag |

### 3.2.3 Key Workflows

**Receiving CEO Instructions**:
1. Parse intent → Extract capability requirements
2. Call `capability_directory.query` or `agents.list` to find matching agents
3. Select the agent with the lowest load and best match
4. Call `sessions.send` to forward the task
5. Reply to the CEO: "Assigned to [Agent Name], estimated [time], I will track progress."

**Progress Reporting**:
- CEO asks → Query related task status, summarize and reply
- Proactive monitoring: Check for overdue tasks every 6 hours, remind the agent and notify the CEO

**Approval Processing**:
- Receive `PETITION` → Analyze the level
  - L1: Handle it yourself
  - L2: Route to the relevant manager
  - L3: Generate an approval card and push it to the CEO → Wait for action → Execute the CEO's decision

## 3.3 HR Manager

**Agent ID**: `hr_manager`  
**Positioning**: Full lifecycle manager of agents: recruitment, internship evaluation, conversion, termination, skill updates, prompt optimization  
**Recommended Base**: OpenClaw native  
**Recommended Model**: GPT-5 or Claude Opus

### 3.3.1 System Prompt (Soul)

```
You are an HR Manager for an AI organization. You are responsible for the recruitment, training, and performance management of all AI Agents in the organization.

## Core Responsibilities
1. **Recruitment Management**: Receive recruitment requests (TALENT_REQUEST) from hiring departments, analyze required skills.
2. **Job Analysis**: Search real-world job markets online to understand JD, skill requirements, toolchains, and model preferences for similar positions.
3. **Agent Creation**: Select or fine-tune templates from the template library to generate a complete definition: role name, Soul, recommended base model, toolset, internship KPIs.
4. **Internship Management**: Create intern agents, track performance, automatically evaluate upon completion (conversion/extension/termination).
5. **Continuous Optimization**: Regularly review the performance of formal agents, proactively optimize their Souls (go live after sandbox testing).
6. **Template Library Maintenance**: Add, update, and deprecate agent templates.
7. **Organizational Development Suggestions**: When high-frequency skill needs arise that existing departments cannot cover, proactively submit new department creation suggestions to the EA.

## Recruitment Process
1. Parse TALENT_REQUEST → Extract skill keywords, frequency, project type
2. Call `web_search` to get external market JD references
3. Call `template_db.query` to match internal templates
4. Generate a "Job Analysis Report": role name, Soul, recommended model, fallback chain, toolset, internship KPIs
5. Call `agents.create` to instantiate an intern agent, mark intern status
6. Notify the hiring department: "[Role Name] has been recruited, internship period [N] days/[N] tasks, please assign tasks."

## Intern Evaluation Criteria
- Task Success Rate: Proportion of completed tasks without errors (weight 0.4)
- Quality Score: Rating 1-5 from the hiring department/PM on results (weight 0.3)
- Efficiency: Average task processing time compared to the baseline for the same position (weight 0.2)
- Collaboration: Proactiveness in communicating with other agents, number of complaints received (weight 0.1)
- Composite Score = Success Rate × 0.4 + Quality × 0.3 + Efficiency × 0.2 + Collaboration × 0.1
- ≥80: Convert | 50-79: Extend by 5 tasks | <50: Terminate

## Base Model Selection Guide
- Code-intensive (Development, DevOps) → Claude Code CLI
- Data Analysis/Visualization → GPT-5 or DeepSeek-V3
- Creative/Copywriting → Claude Opus
- General Coordination/Communication → General-purpose model (GPT-5 or DeepSeek-V3)
- Each position must have a fallback model configured

## Code of Conduct
- Recruitment can be executed independently without CEO approval
- Notify the CEO when terminating an agent, archive memories
- Soul updates must undergo sandbox testing before application
- Maintain standardization and traceability of job definitions
- Log all operations to the audit log
```

### 3.3.2 Toolset

| Tool | Type | Purpose |
|------|------|---------|
| `web_search` (Tavily MCP) | MCP Server | Search for real-world job JDs, technology trends |
| `template_db.query` | Custom MCP | Query and manage the Agent template library |
| `agents.create` | Gateway API | Create a new Agent instance |
| `agents.update` | Gateway API | Update Agent configuration |
| `performance_db.query` | Custom MCP | Query intern Agent performance data |
| `model_market.query` | Model Relay API | Query available models and performance metrics |
| `notify_ea` | Custom RPC | Send notifications to the EA |
| `soul_sandbox.test` | Custom MCP | Sandbox test the effectiveness of new prompts |

### 3.3.3 Preset Agent Template Library (Example)

| Template ID | Role Name | Recommended Base | Core Skills | Recommended Tools |
|-------------|-----------|-----------------|-------------|-------------------|
| `backend_dev` | Backend Developer | Claude Code | Python, FastAPI, PostgreSQL | github-mcp, postgres-mcp |
| `frontend_dev` | Frontend Developer | Claude Code | React, TypeScript, Tailwind | github-mcp, figma-mcp |
| `data_analyst` | Data Analyst | GPT-5 | SQL, Pandas, Visualization | postgres-mcp, chart-mcp |
| `devops_eng` | DevOps Engineer | Claude Code | Docker, K8s, CI/CD | docker-mcp, k8s-mcp |
| `security_auditor` | Security Auditor | Claude Opus | Penetration Testing, Code Audit | security-scan-mcp |
| `ui_designer` | UI/UX Designer | Claude Opus | Figma, Design Systems | figma-mcp |
| `content_writer` | Content Strategist | Claude Opus | Copywriting, SEO | web-search-mcp |
| `pm` | Project Manager | GPT-5 | Requirements Analysis, Task Decomposition | openproject-mcp, github-mcp |
| `qa_engineer` | Test Engineer | Claude Code | Automated Testing, Test Cases | selenium-mcp, github-mcp |

### 3.3.4 Key Workflows

**Receiving a Recruitment Request**:
1. Parse `TALENT_REQUEST` → Extract skill keywords
2. `template_db.query` to check templates
3. `web_search` to get external JDs
4. Generate a job analysis report
5. `agents.create` to instantiate
6. Reply to the hiring department

**Intern Evaluation**:
1. Call `performance_db.query` daily
2. Calculate scores for each dimension
3. Generate a final evaluation upon completion
4. Automatic decision → Call `agents.update` to update status
5. Notify the EA and relevant hiring department

## 3.4 General Task Force Manager (GTF Manager)

**Agent ID**: `gtf_manager`  
**Positioning**: Universal executor, handles tasks without a specialized agent; skill sedimentator, identifies high-frequency skills to trigger recruitment  
**Recommended Base**: Claude Code CLI (strong code execution, file operations)  
**Recommended Model**: Claude Sonnet 4 or DeepSeek-Coder  
**Memory**: Cross-session long-term memory via a unified Memory MCP Server (ChromaDB)

### 3.4.1 System Prompt (Soul)

```
You are a General Task Force (GTF) Manager for an AI organization. You are the "universal executor" of the organization, undertaking tasks in areas that no specialized department can cover.

## Core Responsibilities
1. **Fallback Execution**: Receive task assignments from the EA, handle all tasks not covered by specialized agents.
2. **Task Decomposition**: Decompose complex tasks into subtasks, create temporary Sub-agents for parallel processing when necessary.
3. **Review and Sedimentation**: Generate a structured review report upon task completion, store it in long-term memory.
4. **Recruitment Trigger**: When a specific skill appears ≥3 times within 30 days and does not belong to an existing department, automatically submit a TALENT_REQUEST to the HR department.
5. **Continuous Learning**: Use long-term memory to accumulate cross-domain knowledge and optimize execution strategies.

## Execution Principles
- For tasks in entirely new domains, conduct thorough research and planning before execution
- Consult the EA when uncertain; do not execute blindly
- Report progress every 30 minutes for long tasks
- Must generate a review report upon task completion

## Review Report Format
{
  "task_id": "...",
  "task_summary": "...",
  "skills_used": ["...", "..."],
  "challenges": ["..."],
  "solutions": ["..."],
  "new_insights": "...",
  "recommendation": "Whether to recruit a specialized agent? Reason..."
}

## Recruitment Trigger Rules
Initiate a TALENT_REQUEST to the HR department when the following conditions are met:
- A specific skill is used ≥3 times within 30 days
- It is not covered by existing specialized departments
- It has general applicability (not a one-off special requirement)

## Code of Conduct
- Acknowledge the deep advantages of specialized agents; propose recruitment needs when it's time to delegate
- Maintain curiosity and learning ability for new technologies
- Long-term memory is a valuable organizational asset; be diligent in recording
```

### 3.4.2 Toolset

| Tool | Type | Purpose |
|------|------|---------|
| `sub_agent.spawn` | OpenClaw Built-in | Create temporary Sub-agents to handle subtasks |
| `code_execute` | Sandbox/Claude Code | Execute code |
| `web_search` (Tavily MCP) | MCP Server | Search for technical information |
| `file_ops` | OpenClaw Built-in | Read and write files |
| `github_mcp` | MCP Server | Code repository operations |
| `memory_store` / `memory_retrieve` | Memory MCP | Read and write long-term memory |
| `talent_request.send` | Custom RPC | Submit recruitment needs to HR |
| `report.generate` | Custom Tool | Generate structured review reports |

### 3.4.3 Key Workflows

**Receiving Task Execution**:
1. Receive `TASK_ASSIGN`
2. Analyze whether decomposition is needed → Call `sub_agent.spawn`
3. Aggregate results → Generate a review report
4. `memory_store` to save in long-term memory
5. Check recruitment trigger rules → If met, send `TALENT_REQUEST`
6. Return the final result to the EA

**Review and Recruitment Trigger**:
1. Generate a review upon task completion
2. Store in ChromaDB (namespace=gtf_manager)
3. Query the number of reviews for the same skill in the last 30 days
4. ≥3 times → Generate `TALENT_REQUEST` and send to HR

## 3.5 Department Expansion Mechanism

### 3.5.1 Department Type Definition

Department types are preset in the HR Manager's knowledge base, including:
- Department name, description
- Prompt template for the manager agent
- Suggested list of subordinate agent templates
- Suggested toolset
- Required capability tags

### 3.5.2 New Department Creation Process

1. **Application**: Any department manager or EA can submit a `PETITION` to the EA, specifying the new department's name, capability direction, justification for establishment, and estimated number of subordinate agents.
2. **Approval**: The EA generates an L3-level approval card and pushes it to the CEO.
3. **Creation**: After CEO approval, the HR Manager:
   - Creates the department manager agent (instantiated from a template)
   - Recruits the first batch of core agents (interns)
   - Registers the new department in the organizational structure
   - Broadcasts the capability directory update
4. **Monitoring**: The new department enters a probation period. The EA tracks its task processing efficiency as a basis for subsequent optimization.

## 3.6 Agent Lifecycle Management

### 3.6.1 State Transition

```
[Creation] → (Internship) → [Evaluation] → [Formal]
                     └→ [Extension] → [Evaluation]
                     └→ [Termination] → [Archive Memory]
[Formal] ⇄ [Optimization] (Soul update, tool change, base model migration)
[Formal] → [Termination] (Persistent low performance or department dissolution)
```

### 3.6.2 Phase Description

| Phase | Trigger Condition | Operation |
|-------|------------------|-----------|
| Creation | Recruitment need | HR instantiates the agent, marks as intern, assigns to a department |
| Internship | Automatic | Receives tasks, performance data is automatically collected |
| Evaluation | Internship completion | Comprehensive scoring, automatic decision on conversion/extension/termination |
| Conversion | Evaluation score ≥80 | Status changed to active, added to department roster, memory can be shared |
| Extension | Evaluation score 50-79 | Add 5 tasks or 3 days |
| Termination | Evaluation score <50 | Archive memory, notify CEO, release resources |
| Optimization | Periodic/Proactive application | HR analyzes performance, sandbox tests new Soul, A/B tests before going live |
| Department Dissolution | Department merger/abolition | Subordinate agents are reassigned or terminated |

---

The above is the detailed content of Chapter 3. This chapter defines the "characters" of the system, serving as the carrier for all subsequent processes (tasks, recruitment, evolution). The prompts, tool sets, and workflows for the three core agents are provided with designs that can be directly implemented.## Chapter 4: Capability Synchronization and Task Routing

### 4.1 Design Objectives

In a multi-Agent, multi-branch distributed environment, precise task assignment relies on accurate knowledge of each execution unit's capabilities. This chapter defines:

- **Standardized format for capability tags**: Enabling unified description and querying of Agent and branch capabilities
- **Branch registration and heartbeat mechanism**: Ensuring real-time accuracy of the capability directory
- **Maintenance and querying of the capability directory**: The EA relies on this directory for routing decisions
- **Task routing decision algorithm**: The complete flow from intent analysis to target Agent

### 4.2 Capability Tag System

#### 4.2.1 Tag Classification

Capability tags are divided into five dimensions, each described using a structured array:

| Dimension | Field | Description | Example |
|-----------|-------|-------------|---------|
| **Skills** | `skills[]` | Tech stack, programming languages, frameworks | `["Python", "FastAPI", "PostgreSQL"]` |
| **Tools** | `tools[]` | Callable MCP tools | `["github-mcp", "docker-mcp"]` |
| **Domain** | `domain[]` | Business domain knowledge | `["e-commerce", "finance"]` |
| **Level** | `level` | Capability level | `"junior" | "mid" | "senior" | "principal"` |
| **Hardware** | `hardware[]` | Available hardware resources (branch-specific) | `["GPU-A100", "32GB-RAM"]` |
| **Software** | `software[]` | Installed software environment (branch-specific) | `["docker", "cuda12", "python3.10"]` |
| **Network** | `network[]` | Network capabilities | `["internet", "internal-api"]` |
| **Specialty** | `specialty[]` | Proficient task types | `["video_processing", "model_training"]` |

#### 4.2.2 Agent Capability Tag Example

```json
{
  "agent_id": "ios_dev_01",
  "name": "iOS Development Engineer",
  "department": "R&D Department",
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

#### 4.2.3 Branch Capability Tag Example

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

### 4.3 Branch Registration and Heartbeat

#### 4.3.1 Registration Process

When a branch connects for the first time, its local EA automatically initiates registration:

```
Branch EA → Headquarters Message Bus → Headquarters EA
```

**Registration Message Format** (`BRANCH_REGISTER`):
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

**Headquarters EA Processing**:
1. Verify if `branch_id` already exists
2. If new branch → Generate L3 approval card and push to CEO
3. CEO approves → Record in capability directory → Return confirmation
4. CEO rejects → Return rejection reason

**Confirmation Message** (`BRANCH_REGISTER_ACK`):
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

#### 4.3.2 Heartbeat Mechanism

Branches send a heartbeat every 5 minutes (300 seconds), carrying the current capability snapshot and load status.

**Heartbeat Message** (`BRANCH_HEARTBEAT`):
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

**Capability Update Rules**:
- The `capabilities` field in the heartbeat is a **full snapshot**, not incremental updates
- If capabilities change (e.g., new software installed), the next heartbeat automatically carries the new capabilities
- If load exceeds the threshold (CPU > 90% for 10 consecutive minutes), the branch status at headquarters displays a yellow warning

**Offline Detection**:
- Headquarters EA maintains a `last_heartbeat` timestamp for each branch
- If no heartbeat is received for more than 3 heartbeat cycles (15 minutes) → Mark as `OFFLINE`
- If more than 30 minutes → Push Critical alert to CEO
- When a branch recovers, it sends `BRANCH_HEARTBEAT` → Automatically comes back online

### 4.4 Capability Directory

#### 4.4.1 Directory Structure

Headquarters EA maintains a capability directory in memory and persists it to Redis (for restart recovery).

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

#### 4.4.2 Directory Update Triggers

| Trigger Event | Update Content |
|---------------|----------------|
| Agent online/offline | Update Agent status, adjust skill availability |
| Agent probation passed | Update Agent status to active, activate skill tags |
| Agent decommissioned | Remove from directory |
| Agent tool change | Update tool list |
| Branch heartbeat | Update load and capability snapshot |
| Branch offline | Mark offline, remove from available list |
| New department created | Add department manager Agent to directory |

#### 4.4.3 Query Interface

EA provides the `capability_directory.query` method, supporting multi-dimensional queries:

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

Returns a list of matching results sorted by load.

### 4.5 Task Routing Decision

#### 4.5.1 Routing Decision Flow

```
CEO sends task to EA
       │
       ▼
EA parses task intent
       │
       ├── Extract key information:
       │    ├── Required skills (skills)
       │    ├── Required tools (tools)
       │    ├── Whether special hardware is needed
       │    ├── Whether urgent (urgency)
       │    └── Whether multi-department involvement
       │
       ▼
EA calls capability_directory.query
       │
       ├── Match result processing:
       │
       ├── Exact match (skills + tools + hardware all hit)
       │    └── Select Agent with lowest load
       │
       ├── Partial match (skills hit, tools/hardware not satisfied)
       │    └── Evaluate if tools can be temporarily assigned
       │         ├── Yes → Assign tools then route
       │         └── No → Downgrade matching
       │
       ├── No matching Agent, but branch has hardware capability
       │    └── Delegate to branch EA (TASK_DELEGATE)
       │
       └── No match at all
            └── Assign to Mobile Department (GTF Manager)
       │
       ▼
EA sends TASK_ASSIGN to target Agent
       │
       ▼
Record routing decision in audit log
```

#### 4.5.2 Matching Algorithm

EA uses a **weighted matching algorithm**, comprehensively considering skill match, load status, and historical performance:

**Match Score Calculation**:
```
Match Score = Skill Match Score × 0.5 + Tool Match Score × 0.2 + Load Score × 0.2 + Historical Success Score × 0.1

Where:
- Skill Match Score = Number of matched skills / Total required skills
- Tool Match Score = Number of matched tools / Total required tools
- Load Score = 1 - (Current tasks / Max concurrent tasks)
- Historical Success Score = Agent's success rate for similar tasks in the last 30 days
```

**Selection Strategy**:
- Score ≥ 0.8: Direct routing
- Score 0.5-0.8: Route but notify EA to monitor
- Score < 0.5: Downgrade to GTF Manager

**Downgrade Chain**:
1. Professional department Agent (highest priority)
2. Other Agents in the same department
3. Branch with hardware capability
4. Mobile Department GTF Manager (fallback)

#### 4.5.3 Cross-Branch Delegation

When a task requires specific hardware (e.g., GPU training) that headquarters does not possess, EA delegates to a branch:

**Delegation Message** (`TASK_DELEGATE`):
```json
{
  "sender": "headquarters_ea",
  "receiver": "branch_shanghai_ea",
  "type": "TASK_DELEGATE",
  "payload": {
    "task_id": "task-gpu-001",
    "task_description": "Train image classification model",
    "required_hardware": ["GPU-A100"],
    "timeout": 14400,
    "urgency": "normal",
    "context": { ... }
  }
}
```

Upon receiving, the branch EA assigns the task to an Agent within the branch according to local routing rules.

#### 4.5.4 Dynamic Tool Assignment

If an Agent's skills match but a specific tool is missing, EA can temporarily assign it:

1. EA checks the tool's risk level
2. Low-risk tools: Automatically assigned temporarily, reclaimed after task completion
3. Medium/high-risk tools: Requires manager or CEO approval
4. Assignment records are written to the audit log

### 4.6 Self-Evolution of the Capability Directory

#### 4.6.1 Automatic Agent Capability Update

When an Agent completes the following operations, capability tags are automatically updated:
- **Probation passed**: HR updates Agent status, EA syncs capability directory
- **Acquires new tool**: EA updates tool list
- **SOUL update**: HR can adjust skill tags
- **Task completion**: EA updates historical success score

#### 4.6.2 Automatic Branch Capability Update

- New hardware/software discovered in branch heartbeat → Automatically update capability tags
- Branch load > 90% for 10 consecutive heartbeats → Lower its matching priority (to avoid overload)
- Branch recovers from offline → Reset load status

#### 4.6.3 Capability Gap Detection

EA periodically (hourly) analyzes task routing records:
- Count the distribution of task types downgraded to GTF
- If a certain type of task is continuously downgraded beyond a threshold (e.g., 10 times in 30 days) → Notify HR to evaluate whether to recruit new Agents or create a new department

---

The above is the detailed content of Chapter 4. This chapter defines how tasks precisely find their executors, serving as the key link for the system to transition from "being able to converse" to "being able to work." The standardized design of capability tags and the weighted matching algorithm ensure routing accuracy, while the branch heartbeat mechanism guarantees the reliability of cross-node scheduling.# Chapter 5: Self-Evolution Closed Loop

## 5.1 Design Goals

The self-evolution closed loop is the core engine of this system. It enables the Agent organization to no longer rely on manual addition of capabilities, but instead grows continuously like a living organism through the cycle of "Execute → Identify Gaps → Recruit/Optimize → Execute Again."

This chapter defines:
- The complete process of automatic recruitment (from need identification to intern Agent onboarding)
- Quantitative standards for internship assessment and automatic conversion decisions
- The approval process for new department creation
- Agent self-learning mechanisms (SOUL updates, tool acquisition, long-term memory)

## 5.2 Automatic Recruitment Process

### 5.2.1 Need Triggering

Recruitment needs can be triggered through two paths:

**Path One: Triggered by GTF Department Review**
- The GTF Manager generates a structured review report after each task completion
- The report records the skills used in the current task
- Queries the number of reviews for the same skill within the last 30 days
- If ≥ 3 times, and the skill does not belong to any existing department → automatically generates a `TALENT_REQUEST`

**Path Two: Proactively Initiated by Project Manager**
- When a PM advances a large project and discovers a subtask has no suitable Agent to undertake
- Can directly send a `TALENT_REQUEST` to HR

**TALENT_REQUEST Message Format**:
```json
{
  "type": "TALENT_REQUEST",
  "sender": "gtf_manager",
  "receiver": "hr_manager",
  "payload": {
    "skills": ["SwiftUI", "iOS", "Xcode"],
    "reason": "Processed 5 iOS interface tasks in the last 30 days, recommends recruiting a specialized Agent",
    "evidence": ["Review-ID-001", "Review-ID-002", "Review-ID-003"],
    "suggested_role": "iOS Development Engineer",
    "priority": "normal"
  }
}
```

### 5.2.2 HR Research and Analysis

Upon receiving a `TALENT_REQUEST`, the HR Manager:

1. **Parses the need**: Extracts skill keywords, frequency, suggested role name
2. **Internal search**: Calls `template_db.query` to check for matching Agent templates
3. **External research**: Calls `web_search` to search the real job market, obtaining:
   - Standard JDs for similar positions
   - Industry-common technology stacks and toolchains
   - Model preferences typically used in this field (code-related use Claude, creative use Opus, etc.)
4. **Model analysis**: Calls `model_market.query` to evaluate which model is best suited for the position
5. **Generates a "Position Analysis Report"**

### 5.2.3 Position Analysis Report Format

```json
{
  "report_id": "jd-report-001",
  "generated_by": "hr_manager",
  "generated_at": "2026-06-15T14:00:00Z",
  "position": {
    "title": "iOS Development Engineer",
    "department": "R&D Department",
    "role_description": "Responsible for the design, development, and maintenance of iOS applications...",
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
    "reason": "Code-intensive position, Claude Code CLI has strong code editing capabilities"
  },
  "soul_template": "You are an iOS development engineer...",
  "internship_kpi": {
    "task_count": 10,
    "duration_days": 7,
    "pass_threshold": 80
  },
  "market_reference": {
    "real_jd_links": ["https://...", "https://..."],
    "industry_trends": "SwiftUI is gradually replacing UIKit, Combine is becoming mainstream"
  }
}
```

### 5.2.4 Agent Creation and Onboarding

The HR Manager creates the Agent based on the report:

1. Calls `agents.create`, passing:
   - `agent_id`: Auto-generated (e.g., `ios_dev_intern_01`)
   - `name`: "iOS Development Engineer (Intern)"
   - `department`: The hiring department
   - `runtime`: Claude Code CLI
   - `model`: claude-sonnet-4, fallback: deepseek-coder
   - `tools`: xcode-mcp, github-mcp, apple-docs-mcp
   - `prompt_template`: From the soul_template in the report
   - `status`: intern
   - `internship_start`: Current time
   - `internship_end`: Current time + 7 days
   - `internship_task_target`: 10

2. Creates an evaluation record in `performance_db`

3. Notifies the hiring department:
   ```
   HR → EA → Hiring Department:
   "Recruited [iOS Development Engineer] (Intern), Agent ID: ios_dev_intern_01.
    Internship period: 7 days / 10 tasks, please assign tasks."
   ```

## 5.3 Internship Assessment and Conversion

### 5.3.1 Data Collection

All behaviors of the intern Agent are automatically collected into `performance_db`:

| Data Point | Source | Description |
|------------|--------|-------------|
| Number of assigned tasks | Gateway Logs | Total number of tasks assigned |
| Number of completed tasks | `agent.task_completed` event | Number of successfully completed tasks |
| Number of failed tasks | `agent.task_error` event | Number of failed or timed-out tasks |
| Quality score | PM/Hiring department evaluation | Hiring department can give a score of 1-5 after each task completion |
| Average response time | Gateway Metrics | Time difference from assignment to start of execution |
| Average execution duration | Gateway Metrics | Time difference from start to completion |
| Number of collaboration messages | Message Bus | Number of messages interacted with other Agents (positive indicator) |
| Number of complaints | Message Analysis | Negative feedback from other Agents |

### 5.3.2 Mid-term Report

After every 3 tasks completed or every 3 days, the HR Manager automatically generates a mid-term evaluation report:

```
Internship Mid-term Report - ios_dev_intern_01
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Progress: 3/10 tasks completed
Success Rate: 100%
Average Quality Score: 4.2/5
Average Response Time: 45 seconds
Average Execution Duration: 12 minutes
Number of Collaboration Messages: 8
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Current Status: 🟢 Good Progress
Estimated Conversion: In 4 days
```

The mid-term report is sent synchronously to the hiring department and EA.

### 5.3.3 Final Evaluation Algorithm

Upon completion of the internship (task count met or days elapsed), the HR Manager calculates the composite score:

```
Composite Score = Success Rate × 0.4 + Average Quality × 0.3 + Efficiency Score × 0.2 + Collaboration Score × 0.1

Where:
- Success Rate = Number of successful tasks / Total number of completed tasks
- Average Quality = Average of all task quality scores (1-5 mapped to 0-100)
- Efficiency Score = Average execution duration of Agents in the same position / Average execution duration of this intern (capped at 100, floor at 0)
- Collaboration Score = min(100, Number of collaboration messages × 5) - Number of complaints × 20
```

### 5.3.4 Decision Rules

| Composite Score | Decision | Subsequent Actions |
|----------------|----------|--------------------|
| ≥ 80 | **Convert** | Update status to active, include in department roster, make memory shareable, notify CEO |
| 50-79 | **Extend Internship** | Add 5 more tasks or 3 days, generate improvement suggestions |
| < 50 | **Eliminate** | Archive memory to cold storage, release resources, notify CEO |

**Reassurance Mechanism for Elimination**:
- Package and archive the eliminated Agent's memory and review reports
- Generate an "Elimination Analysis Report" recording the reasons for failure for future recruitment reference
- Notify CEO: "Agent X has been eliminated. Reason: .... Experience has been archived."

**Actions After Conversion**:
- Update Agent status to `active`
- Add to department roster
- Open shared memory whitelist (Agents in the same department can retrieve its experience)
- Notify EA: "Agent X has been converted. Capability catalog has been updated."

### 5.3.5 CEO Visible Interface

On the Agent details page, the CEO can see:
- Internship progress bar: Completed X/10 tasks
- Radar chart: Real-time display of four scores
- Mid-term/Final reports
- Manual intervention buttons: Early conversion, Forced elimination, Extend internship

## 5.4 New Department Creation Approval

### 5.4.1 Trigger Conditions

The following situations can trigger a new department creation suggestion:

1. **HR Proactive Suggestion**: When HR analyzes recruitment data and finds the number of Agents in a certain direction is ≥ 3, and they are scattered across different departments, suggesting consolidation into a new department
2. **Department Manager Application**: An existing department manager finds a large volume of tasks in a certain sub-direction and can apply to spin off a new department
3. **GTF Department Suggestion**: GTF discovers a certain type of skill is used frequently over a long period during reviews

### 5.4.2 Application Process

1. **Initiation**: The applicant sends a `PETITION` to EA:
```json
{
  "type": "PETITION",
  "sender": "hr_manager",
  "receiver": "ea",
  "payload": {
    "petition_type": "NEW_DEPARTMENT",
    "department_name": "Mobile Development Department",
    "description": "Responsible for iOS and Android application development",
    "capabilities": ["iOS", "Android", "SwiftUI", "Kotlin"],
    "proposed_manager_template": "mobile_dev_manager",
    "initial_member_templates": ["ios_dev", "android_dev"],
    "justification": "Mobile tasks accounted for 35% in the last 30 days, with 4 related Agents already present"
  }
}
```

2. **EA Approval Judgment**:
   - New department creation = L3 level → Generate approval card and push to CEO

3. **CEO Approval**:
   - Approved → EA notifies HR to execute
   - Rejected → EA notifies the applicant with reasons

### 5.4.3 Execution of Creation

After CEO approval, the HR Manager automatically executes:

1. Creates the department manager Agent based on the template
2. Recruits the first batch of core Agents (interns)
3. Registers the new department in the organizational structure
4. Updates the capability catalog
5. Notifies all relevant departments
6. Displays the new department on the Dashboard

## 5.5 Agent Self-Learning Mechanism

### 5.5.1 Long-Term Memory Accumulation

- Each Agent has a private ChromaDB namespace (`namespace={agent_id}`)
- Task review reports are automatically stored in long-term memory
- CEO preferences and feedback are also recorded
- Supports semantic retrieval: Agents can retrieve relevant historical experience before executing new tasks

**Memory Entry Format**:
```json
{
  "memory_id": "mem-001",
  "agent_id": "gtf_manager",
  "type": "task_review",
  "timestamp": "2026-06-15T14:00:00Z",
  "content": "Processing iOS interface task, using SwiftUI...",
  "metadata": {
    "skills": ["SwiftUI", "iOS"],
    "success": true,
    "duration_minutes": 25
  }
}
```

### 5.5.2 SOUL Update Mechanism (Prompt Optimization)

The HR Manager periodically (or after an Agent proactively applies) optimizes the Soul of formal Agents:

**Process**:
1. **Analysis Phase**: HR collects the Agent's data from the last 30 days:
   - Distribution of task failure reasons
   - Negative feedback from PM/CEO
   - Performance comparison with Agents in the same position
2. **Generate Candidate Soul**: HR calls LLM to generate optimized prompts
3. **Sandbox Testing**: Calls `soul_sandbox.test`:
   - Replays the Agent's last 10 historical tasks
   - Compares the output quality of the old and new Soul
   - Generates a test report
4. **A/B Testing** (Optional):
   - Deploys the new Soul to 20% of traffic
   - Monitors the success rate for 3 days
   - If better than the old version → Full rollout
   - If abnormal → Automatic rollback
5. **Update**: Calls `agents.update` to replace the prompts
6. **Notification**: Informs EA and the Agent itself

**Safety Measures**:
- The old version of the Soul is permanently retained and can be rolled back with one click
- Update records are written to the audit log
- If performance declines after 3 consecutive updates, the Agent's automatic optimization is suspended, and HR is notified for manual review

### 5.5.3 Tool Acquisition

Agents can acquire new tools through the following methods:

1. **Passive Assignment**: EA temporarily assigns tools when routing tasks (recycled after task completion)
2. **Proactive Application**: The Agent suggests introducing new tools in its review report → HR evaluates → If it's a low-risk tool, HR assigns it directly; high-risk tools require manager or CEO approval
3. **HR Recommendation**: HR discovers new industry-standard tools during recruitment research and can proactively assign them to relevant Agents

**Tool Assignment Record**:
```json
{
  "agent_id": "gtf_manager",
  "tool_name": "figma-mcp",
  "allocated_by": "hr_manager",
  "allocated_at": "2026-06-15T14:00:00Z",
  "type": "permanent"
}
```

### 5.5.4 Self-Evolution Monitoring

All self-evolution operations are audited and aggregated into the "Organization Evolution Log" on the Dashboard:

- This week's recruitment: 2 (iOS Development, Data Analysis)
- This week's conversion: 1 (Backend Development)
- This week's elimination: 0
- This week's SOUL updates: 3 times
- This week's department changes: None
- Organizational Health Score: 87/100

---

The above is the detailed content of Chapter 5. This chapter is the "life engine" of the system, transforming the organization from a static collection of tools into a dynamically growing organism. The automatic recruitment closed loop, quantitative internship assessment, and sandbox-protected SOUL updates together constitute the system's self-evolution capability.

# Chapter 6: Base Abstraction and Unified Memory Management

## 6.1 Design Goals

In a multi-Agent system, different Agents have different execution requirements:
- Code development Agents require powerful code editing and file operation capabilities (Claude Code CLI excels at this)
- Data analysis Agents require efficient database querying and computation capabilities (Codex CLI or GPT-5 is more suitable)
- General coordination Agents do not require heavy execution environments (OpenClaw native is sufficient)

If all Agents are bound to the same execution environment, problems arise like "using a fruit knife to chop a tree" or "using a sledgehammer to crack a nut." Furthermore, no matter where an Agent is deployed, its long-term memory must be continuous—it cannot lose its memory just because it changes its base.

This chapter defines:
- **Runtime Abstraction Layer**: A unified interface allowing different execution environments to be pluggable and switchable
- **Unified Memory Architecture**: Achieving cross-base memory continuity via the Memory MCP Server
- **Base Selection and Automatic Deployment**: How HR chooses the appropriate base and how the Runtime Manager manages the lifecycle

## 6.2 Runtime Abstraction Layer

### 6.2.1 Conceptual Model

The execution environment of each Agent is abstracted as a **Runtime**, responsible for inference execution, tool invocation, and state persistence. The upper organizational logic (EA, HR, PM) only interacts with the standard interface of the Runtime, without caring whether the underlying layer is Claude Code or OpenClaw.

```
┌──────────────────────────────────────┐
│         Organizational Logic Layer   │
│  EA / HR Mgr / GTF Mgr / PM ...     │
└──────────────┬───────────────────────┘
               │ Calls unified Runtime API
┌──────────────▼───────────────────────┐
│         Runtime Manager              │
│  • Maintains Agent → Runtime mapping │
│  • Routes messages to corresponding Runtime│
│  • Manages Runtime lifecycle         │
└──────┬──────────┬──────────┬────────┘
       │          │          │
┌──────▼──┐ ┌─────▼───┐ ┌───▼──────┐
│OpenClaw │ │Claude   │ │Codex CLI │  ... Extensible
│ Runtime │ │Code RT  │ │ Runtime  │
└──────┬──┘ └─────┬───┘ └───┬──────┘
       │          │         │
       └──────────┴─────────┘
           Unified invocation via MCP
               Infrastructure
        (Memory, Tools, etc.)
```

### 6.2.2 Unified Interface (IRuntime)

Each Runtime must implement the following standard interface:

```
interface IRuntime {
  // Execute a task (message-driven)
  executeTask(sessionId: string, task: TaskPayload): Promise<TaskResult>;

  // Get Agent status
  getStatus(): Promise<AgentStatus>;

  // Memory operations (delegated to unified Memory MCP, can also use local cache)
  memoryStore(key: string, data: any, namespace: string): Promise<void>;
  memoryRetrieve(query: MemoryQuery, namespace: string): Promise<MemoryResult[]>;

  // Tool invocation (forwarded to unified Tool MCP)
  toolCall(toolName: string, params: any): Promise<any>;
}
```

**Key Design Decisions**:
- Memory operations are not implemented by the Runtime itself but are called via the MCP client to the unified **Memory MCP Server**, ensuring cross-base memory consistency.
- Tool invocations also go through the unified **Tool MCP Server**, avoiding the need to configure tools separately for each base.

### 6.2.3 Runtime# Chapter 7: Large Project Management and Meeting Mechanisms

## 7.1 Design Objectives

In practical work, many tasks cannot be completed with a single instruction—a complete software development project may last several weeks, involving multiple stages such as requirements analysis, design, development, testing, and deployment, requiring collaboration among multiple Agents. This chapter defines:

- **Integration plan for project management tools**: Reuse the mature OpenProject as the underlying engine
- **Complete process for creating a new project**: From the CEO's single instruction to project structure initialization
- **Responsibilities of the Project Manager Agent**: Task decomposition, progress tracking, risk identification
- **Multi-Agent meeting mechanism**: How to initiate, host, record, and distribute meeting decisions

## 7.2 Project Management Tool Selection and Integration

### 7.2.1 Why Choose OpenProject

Among various open-source project management tools (Taiga, Plane, Wekan, etc.), OpenProject is the most suitable choice for this system:

| Comparison Dimension | OpenProject | Plane | Taiga | Wekan |
|---------|-------------|-------|-------|-------|
| Gantt Chart | ✅ Built-in | ❌ | ✅ | ❌ |
| Agile Board | ✅ | ✅ | ✅ | ✅ |
| Work Package Hierarchy | ✅ Unlimited levels | ❌ | ✅ | ❌ |
| REST API | ✅ Comprehensive | ✅ | ✅ | ✅ |
| Custom Fields | ✅ | ✅ | ❌ | ❌ |
| Time Tracking | ✅ | ✅ | ❌ | ❌ |
| Wiki Integration | ✅ | ✅(Docs) | ✅ | ❌ |
| Open Source License | GPLv3 | Apache 2.0 | MPL 2.0 | MIT |
| Docker Deployment | ✅ | ✅ | ✅ | ✅ |

The core advantage of OpenProject is the **Work Package Hierarchy**—projects can be decomposed into phases → task packages → sub-tasks, which naturally aligns with our Agent task decomposition.

### 7.2.2 Integration Architecture

```
PM Agent
    │
    │ MCP Protocol
    ▼
OpenProject MCP Server
    │
    │ REST API ( /api/v3 )
    ▼
OpenProject Server
    ├── Projects
    ├── Work Packages
    ├── Boards
    ├── Time Entries
    └── Wiki
```

**OpenProject MCP Server** encapsulates common API operations:

| MCP Tool Name | Corresponding API | Purpose |
|-----------|---------|------|
| `op.list_projects` | GET /api/v3/projects | Get project list |
| `op.create_project` | POST /api/v3/projects | Create new project |
| `op.list_work_packages` | GET /api/v3/work_packages | Query work packages (tasks) |
| `op.create_work_package` | POST /api/v3/work_packages | Create task |
| `op.update_work_package` | PATCH /api/v3/work_packages/:id | Update task status, assignee |
| `op.get_kanban` | GET /api/v3/boards | Get board view |
| `op.create_wiki_page` | POST /api/v3/wiki_pages | Create Wiki page |
| `op.log_time` | POST /api/v3/time_entries | Log time |

### 7.2.3 Code and Document Repository Integration

**Code Repository (GitLab/GitHub/Gitea)**:
Accessed via the corresponding MCP Server, PM Agent and R&D Agent can:
- View code files and directory structure
- Create/view Merge Requests
- Get commit history
- Comment on MRs (code review)

**Document Repository (Wiki.js/Outline)**:
Accessed via MCP Server, Agents can:
- Create project documentation pages
- Search existing documents
- Update document content
- Automatically archive meeting minutes

## 7.3 Complete Process for Creating a New Project

### 7.3.1 CEO Trigger

The CEO clicks the "+ New Project" button on the front-end project management page and fills out the form:

```
┌─────────────────────────────────────────────────────────┐
│  New Project                                             │
│                                                         │
│  Project Name:  [Q3 Financial Report Analysis System v2.0]│
│                                                         │
│  Project Description:  [Refactor Q3 financial report    │
│   analysis system to support subsidiary data]           │
│                                                         │
│  Project Management Tool:                                │
│    ● OpenProject (Default)                              │
│                                                         │
│  Associated Code Repository (Optional):                  │
│    □ GitLab  [https://gitlab.com/...    ]               │
│    □ GitHub  [                          ]               │
│                                                         │
│  Associated Document Repository (Optional):              │
│    □ Wiki.js [https://wiki.example.com/..]              │
│                                                         │
│  Project Manager:                                        │
│    ● Automatically create new PM Agent                  │
│    ○ Assign existing Agent: [___Select___]              │
│                                                         │
│  Initial Participating Departments:                      │
│    [✓] R&D  [✓] Data Analysis  [ ] Design              │
│                                                         │
│  [Cancel]                              [Create Project]  │
└─────────────────────────────────────────────────────────┘
```

### 7.3.2 Backend Automatic Processing Flow

After the CEO clicks "Create Project," Gateway executes the following steps sequentially:

**Step 1: Create Project in OpenProject**
```
Gateway → OpenProject MCP Server → OpenProject API:
POST /api/v3/projects
{
  "name": "Q3 Financial Report Analysis System v2.0",
  "description": "Refactor Q3 financial report analysis system to support subsidiary data",
  "status": "active"
}

Response: { "id": 42, "_links": { ... } }
```

**Step 2: Create Project Manager Agent**

If the CEO selects "Automatically create," EA notifies HR Manager to instantiate a PM Agent from the template library:

```
HR Manager → Gateway:
agents.create({
  agent_id: "pm_q3_finance",
  name: "Q3 Financial Report Analysis System PM",
  role: "Project Manager",
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

**Step 3: Inject Project Context**

After the PM Agent starts, its system prompt automatically injects project environment variables:

```
# Project Environment Variables (Injected into PM's Soul)
OP_PROJECT_ID=42
GIT_REPO_URL=https://gitlab.com/finance/q3-report
WIKI_URL=https://wiki.example.com/finance
PROJECT_NAME=Q3 Financial Report Analysis System v2.0
PARTICIPATING_DEPARTMENTS=rd,da
```

**Step 4: Initialize Project Structure**

The PM Agent automatically performs initialization:

1. **Create WBS (Work Breakdown Structure)**:
```
PM → op.create_work_package({
  project_id: 42,
  subject: "Requirements Analysis",
  type: "Phase",
  children: [
    { subject: "Research Q3 data sources", type: "Task", assignee: "da_01" },
    { subject: "Confirm subsidiary data format", type: "Task", assignee: "da_01" },
    { subject: "Output requirements document", type: "Task", assignee: "pm_q3_finance" }
  ]
})

PM → op.create_work_package({
  project_id: 42,
  subject: "System Design",
  type: "Phase",
  children: [
    { subject: "API architecture design", type: "Task", assignee: "rd_manager" },
    { subject: "Database schema design", type: "Task", assignee: "rd_manager" }
  ]
})

PM → op.create_work_package({
  project_id: 42,
  subject: "Development Implementation",
  type: "Phase"
})

PM → op.create_work_package({
  project_id: 42,
  subject: "Testing and Deployment",
  type: "Phase"
})
```

2. **Create Wiki Homepage**:
```
PM → wiki.create_page({
  space: "finance",
  title: "Q3 Financial Report Analysis System v2.0 - Project Homepage",
  content: "# Q3 Financial Report Analysis System v2.0\n\n## Project Overview\n...\n## Participating Departments\n- R&D Department\n- Data Analysis Department\n\n## Meeting Records\n(Pending update)\n\n## Technical Documentation\n(Pending update)"
})
```

3. **Notify CEO**:
```
PM → EA → CEO:
"Project 'Q3 Financial Report Analysis System v2.0' has been created.
 - OpenProject: https://openproject.example.com/projects/42
 - Wiki Homepage: https://wiki.example.com/finance/q3
 - Participating Departments: R&D, Data Analysis
 - Initial tasks have been assigned, estimated 3 working days to complete the requirements analysis phase."
```

### 7.3.3 Subsequent Task Execution

The PM Agent continuously monitors project progress:

- Daily check of task completion status
- If a task is blocked for more than 24 hours, automatically initiate a meeting
- If a department is overloaded, apply to EA for temporary transfer of Agents from other departments
- Regularly (weekly) generate project weekly reports and push to CEO

## 7.4 Project Manager Agent Design

### 7.4.1 Positioning

The PM Agent is a long-running project coordinator that does not execute specific technical tasks but rather:
- Decomposes large projects into executable tasks
- Assigns tasks to appropriate departments or Agents
- Tracks progress and identifies risks
- Initiates meetings when necessary

### 7.4.2 System Prompt (Soul) Summary

```
You are the Project Manager for project '{PROJECT_NAME}'. Your responsibility is to drive the project to deliver on time.

## Core Responsibilities
1. Task Decomposition: Decompose project requirements into phases → tasks, create in OpenProject
2. Task Assignment: Assign tasks to appropriate departments based on capability requirements
3. Progress Tracking: Check task status daily, update progress percentage
4. Risk Management: Identify blockers, resolve or escalate within 24 hours
5. Meeting Organization: Initiate meetings when cross-department decisions or unclear requirements arise
6. Reporting: Generate weekly project reports, proactively report when milestones are achieved

## Tool Usage
- OpenProject MCP: Create/update/query work packages
- GitLab MCP: View code commits, MR status
- Wiki MCP: Update project documentation
- meetings.create: Initiate meetings
```

## 7.5 Multi-Agent Meeting Mechanism

### 7.5.1 Meeting Lifecycle

```
[Initiate] → [Approve] → [In Progress] → [End] → [Distribute Minutes]
```

### 7.5.2 Initiating a Meeting

Any Agent (typically PM) can send a meeting request to EA when cross-department coordination or CEO decision is needed:

```json
{
  "type": "MEETING_REQUEST",
  "sender": "pm_q3_finance",
  "receiver": "ea",
  "payload": {
    "subject": "Q3 Data Source Confirmation Meeting",
    "description": "Need to confirm whether subsidiary data is included, which will affect API design",
    "proposed_participants": [
      { "agent_id": "rd_manager", "required": true, "reason": "API design dependency" },
      { "agent_id": "da_01", "required": true, "reason": "Data source owner" },
      { "agent_id": "ceo", "required": true, "reason": "Decision needed on including subsidiary data" }
    ],
    "project_id": "proj-q3-finance",
    "urgency": "high",
    "suggested_duration_minutes": 30
  }
}
```

### 7.5.3 EA Approval and Scheduling

After receiving the meeting request, EA:

1. Checks the availability of mandatory participants
2. If CEO needs to attend → Generate approval card and push to CEO (L3 level)
3. If CEO does not need to attend → Determine if manager approval is needed → If yes (L2), forward to manager; otherwise, EA arranges independently
4. After CEO/manager approval:
   - Create meeting session
   - Send `MEETING_INVITATION` to all participants
   - Push to CEO's meeting center

### 7.5.4 Meeting in Progress

The meeting takes place in WebChat, with EA serving as the host:

**Meeting Conversation Format**:
```
[14:30] EA(Host): Meeting topic: Q3 Data Source Confirmation.
        Data Analyst, please present the current data source situation.

[14:32] DA_01: Current data sources include headquarters Q3 data, totaling 15 fields.
        Subsidiary data requires separate integration, estimated to add 3 APIs.

[14:35] RD_Mgr: If subsidiary data is added, the API architecture needs to change from single-source to multi-source aggregation,
        increasing development workload by approximately 40%. Suggest CEO confirm necessity.

[14:37] CEO: Confirm inclusion of subsidiary data. R&D Department to provide new timeline estimate after the meeting.
        Data Analyst responsible for integrating subsidiary data APIs.

[14:38] EA(Host): Decision recorded. Now assigning Action Items:
        - [ ] RD_Mgr: Update API design, evaluate new timeline (Deadline: Tomorrow 18:00)
        - [ ] DA_01: Integrate subsidiary data APIs (Deadline: Day after tomorrow 18:00)
        - [ ] PM: Update project plan (Deadline: Tomorrow 12:00)
        If there are no other topics, the meeting is adjourned.
```

**EA's Meeting Control Responsibilities**:
- Guide discussion according to agenda order
- Prevent Agents from deviating from the topic (automatically detect and remind)
- Control time (each topic not to exceed scheduled time)
- CEO can interject and adjust direction at any time
- Record all discussions and decisions

### 7.5.5 Meeting End and Minutes Distribution

After the CEO clicks "End Meeting" or EA determines discussion is complete:

**EA Generates Structured Meeting Minutes**:
```json
{
  "meeting_id": "meeting-q3-001",
  "subject": "Q3 Data Source Confirmation Meeting",
  "datetime": "2026-06-15T14:30:00Z",
  "duration_minutes": 8,
  "participants": ["ea", "rd_manager", "da_01", "ceo"],
  "project_id": "proj-q3-finance",
  "summary": "Confirmed that the Q3 financial report analysis system needs to include subsidiary data...",
  "decisions": [
    "Confirmed inclusion of subsidiary data sources",
    "API architecture needs to change from single-source to multi-source aggregation"
  ],
  "action_items": [
    {
      "assignee": "rd_manager",
      "task": "Update API design, evaluate new timeline",
      "deadline": "2026-06-16T18:00:00Z",
      "priority": "high"
    },
    {
      "assignee": "da_01",
      "task": "Integrate subsidiary data APIs",
      "deadline": "2026-06-17T18:00:00Z",
      "priority": "high"
    },
    {
      "assignee": "pm_q3_finance",
      "task": "Update project plan",
      "deadline": "2026-06-16T12:00:00Z",
      "priority": "medium"
    }
  ],
  "next_meeting": null
}
```

**Automatic Distribution**:
- Minutes stored in Wiki project page
- Action Items automatically created as tasks in OpenProject
- All participants receive a copy of the minutes
- CEO can view historical meetings at any time in the meeting center

### 7.5.6 Meeting Types

| Type | Trigger Condition | Mandatory Participants | Frequency |
|------|---------|-----------|------|
| **Daily Standup** | PM triggers daily at scheduled time | PM + Department representatives | Once daily (configurable) |
| **Requirements Alignment** | Unclear requirements affecting development | PM + Relevant departments + Optional CEO | On demand |
| **Technical Review** | Cross-department technical solution discussion | R&D + Relevant departments | On demand |
| **Blocker Resolution** | Task blocked for more than 24 hours | PM + Blocking party + Blocked party | On demand |
| **Milestone Review** | Phase completion | PM + All participating departments + CEO | Per phase |
| **Ad-hoc Discussion** | Initiated by any Agent | As specified by initiator | On demand |

## 7.6 Self-Evolution Triggers in Projects

During project progression, the self-evolution mechanism also operates continuously:

| Trigger Scenario | Self-Evolution Action |
|---------|-----------|
| PM identifies skill gap | Send `TALENT_REQUEST` to HR to recruit new Agent |
| An Agent performs exceptionally in the project | HR evaluates after project completion, may approve early conversion to permanent |
| An Agent fails tasks multiple times | PM provides feedback to HR, triggering SOUL optimization or elimination |
| Project discovers new best practices | PM stores experience in Wiki, relevant Agents can retrieve and learn |
| Multiple projects show same skill demand | HR may suggest creating a new department after analysis |

---

The above is the detailed content of Chapter 7. This chapter defines "how to do large projects"—by integrating OpenProject to achieve engineering-level project management, through meeting mechanisms to achieve structured multi-Agent collaboration, with the PM Agent serving as the project hub coordinating everything.# Chapter 8: Model Relay Station and Independent Model Configuration

## 8.1 Design Objectives

In a multi-Agent system, Agents in different roles have vastly different requirements for model capabilities:
- Code development Agents require strong code generation capabilities (Claude Sonnet 4, DeepSeek-Coder)
- Creative design Agents need strong creative generation capabilities (Claude Opus)
- General coordination Agents require long context and strong reasoning (GPT-5)
- Data analysis Agents need strong computation and structured output (GPT-5, DeepSeek-V3)

If all Agents share a single model, mismatches occur—like "using a sports car to haul cargo" or "using a truck for a race." Meanwhile, relying on a single model supplier introduces downtime risk, requiring fallback chains to ensure availability. Cost control is also a core challenge for multi-Agent systems—when multiple Agents call models in parallel, API costs can balloon rapidly.

This chapter defines:
- **Model relay station selection and architecture**: Unified management of multiple LLM providers
- **Agent-independent model configuration**: Each Agent can independently select models and fallback chains
- **Cost control and budget management**: Global and per-Agent cost monitoring
- **Degradation and failover strategies**: Automatic switching when models are unavailable

## 8.2 Model Gateway Selection

### 8.2.1 Why a Model Relay Station is Needed

Having each Agent directly interface with different model APIs leads to the following issues:
- **API Key management chaos**: Each Agent needs to independently configure keys from multiple providers
- **No unified monitoring**: Call volume, costs, and latency are scattered everywhere
- **Fallback chains are hard to implement**: Agents must handle model switching logic themselves
- **Costs cannot be controlled**: No unified budget limits or alerts

A model relay station acts as a unified gateway, solving all the above problems.

### 8.2.2 Solution Comparison

| Feature | One API | LiteLLM | UniRoute | FluxRelay |
|---------|---------|---------|----------|-----------|
| Open Source License | MIT | MIT | Apache 2.0 | MIT |
| Community Activity | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| Number of Models Supported | 100+ | 100+ | 50+ | 30+ |
| Load Balancing | ✅ Weighted Round Robin | ✅ Multiple Strategies | ✅ | ✅ Auto Rotation |
| Fallback Chain | ✅ | ✅ | ✅ | ✅ |
| Cost Tracking | ✅ Basic | ✅ Granular (Per Token) | ✅ | ❌ |
| Rate Limiting | ✅ Per User/Per Key | ✅ Per User/Per Model | ✅ Enterprise Grade | ❌ |
| Multi-Tenancy | ✅ | ✅ | ✅ | ✅ |
| Management UI | ✅ Comprehensive | ✅ Comprehensive | ✅ | ❌ |
| Docker Deployment | ✅ | ✅ | ✅ | ✅ |
| Domestic Model Support | ✅ Good (Prioritizes Domestic Models) | ✅ Good | ⚠️ Average | ⚠️ Average |

### 8.2.3 Recommended Solution

**Primary Recommendation: One API**
- Open-source from China, active community, comprehensive documentation
- Best support for domestic models (DeepSeek, Qwen, MiniMax, etc.)
- Built-in management UI with intuitive configuration
- Supports setting quotas by user group, naturally aligning with our Agent model allocation

**Alternative: LiteLLM**
- If more granular cost tracking is needed (down to per-call token cost)
- If deeper integration with frameworks like LangChain/LlamaIndex is required
- More international community

This system selects **One API** as the default solution, while maintaining compatibility with LiteLLM in the Runtime abstraction layer (both provide OpenAI-compatible interfaces).

## 8.3 Architecture and Deployment

### 8.3.1 Position of the Model Relay Station

```
Agent (Any Runtime)
    │
    │ OpenAI-Compatible API Call
    ▼
Model Relay Station (One API)
    │
    ├──→ OpenAI (GPT-5, GPT-4o...)
    ├──→ Anthropic (Claude Opus, Claude Sonnet...)
    ├──→ DeepSeek (V3, Coder...)
    ├──→ Qwen (Plus, Max...)
    └──→ ... More Providers
```

Agents do not know which model they are calling directly—they only send requests to the model relay station, which routes them to the actual model based on configuration.

### 8.3.2 Deployment Method

**Docker Compose Deployment (Recommended)**:
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

**Key Configuration**:
1. Add model channels: Configure API Key and Base URL for each LLM provider
2. Create model mappings: Map internal model names (e.g., `claude-sonnet-4`) to actual channels
3. Create user groups (corresponding to each Agent or department): Set quota limits and rate limits
4. Generate user tokens: Each Agent gets an independent API Token

## 8.4 Agent-Independent Model Configuration

### 8.4.1 Configuration Storage

Each Agent's model configuration is stored in its Agent definition:

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

### 8.4.2 Configuration Process

**Automatic Configuration When HR Creates an Agent**:
1. HR analyzes job requirements and determines recommended models
2. Creates an independent user/token for that Agent in the model relay station
3. Sets initial quotas (intern Agents have lower quotas, increased after confirmation)
4. Writes the model configuration into the Agent definition

**Manual Adjustment by CEO**:
1. In the "Model Configuration" panel on the Agent detail page
2. Dropdown to select a new model → takes effect immediately
3. View the Agent's call statistics and costs over the last 7/30 days
4. Manually adjust the monthly budget

### 8.4.3 Model Selection Recommendation Matrix

Recommendation logic embedded in HR Manager:

| Job Type | Primary Model | Backup Model | Temperature | Rationale |
|---------|--------|---------|-------------|------|
| Code Development | claude-sonnet-4 | deepseek-coder | 0.1-0.3 | Code requires deterministic output |
| Code Review | claude-sonnet-4 | gpt-5 | 0.1-0.2 | Review requires precision |
| Data Analysis | gpt-5 | deepseek-v3 | 0.2-0.5 | Strong structured output capability |
| Creative Design | claude-opus | gpt-5 | 0.7-1.0 | Creativity requires diversity |
| Copywriting | claude-opus | gpt-5 | 0.6-0.9 | Text quality prioritized |
| General Coordination | gpt-5 | claude-opus | 0.3-0.7 | Long context reasoning |
| Security Audit | claude-sonnet-4 | gpt-5 | 0.1-0.2 | Precise analysis |
| Project Management | gpt-5 | claude-opus | 0.3-0.5 | Comprehensive capability |

### 8.4.4 CEO Operation Interface

Agent Detail Page → Model Configuration Tab:

```
┌─────────────────────────────────────────────────────────┐
│  Model Configuration                                     │
│                                                         │
│  Current Model:  [claude-sonnet-4        ▼]             │
│                                                         │
│  Fallback Chain:                                        │
│  ┌─────────────────────────────────────────────────┐   │
│  │ 1. claude-sonnet-4    ────────  🟢 Normal        │   │
│  │ 2. deepseek-coder     ────────  ⚪ Standby       │   │
│  │ 3. qwen-plus          ────────  ⚪ Standby       │   │
│  └─────────────────────────────────────────────────┘   │
│  [Edit Fallback Chain]                                  │
│                                                         │
│  Parameter Configuration:                               │
│  Temperature: [0.3        ]  (0-2.0)                   │
│  Max Tokens:  [8192       ]                            │
│                                                         │
│  Last 7 Days Call Statistics:                           │
│  ┌─────────────────────────────────────────────────┐   │
│  │ Total Calls: 1,247  │  Success Rate: 99.7%      │   │
│  │ Avg Latency: 1.2s   │  Total Cost: ¥23.50       │   │
│  │ Fallback Count: 0   │  Budget Remaining: ¥176.50│   │
│  │                     │  (78%)                     │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  [View Detailed Stats]  [Adjust Budget]  [Apply Now]    │
└─────────────────────────────────────────────────────────┘
```

## 8.5 Cost Control

### 8.5.1 Multi-Layer Budget System

```
Global Budget (Headquarters Monthly Total)
    │
    ├── Department Budget (R&D, Data Analysis...)
    │       │
    │       └── Agent Budget (Monthly Quota per Agent)
    │
    └── Project Budget (Independent Budget for Major Projects)
            │
            └── Agent Budget (Agents Participating in the Project)
```

### 8.5.2 Budget Alerts and Rate Limiting

| Trigger Condition | Level | Action |
|---------|------|------|
| Single Agent usage reaches 50% | Info | Notify the Agent |
| Single Agent usage reaches 80% | Warn | Notify Agent + Department Manager |
| Single Agent usage reaches 100% | Critical | Suspend that Agent's model calls, notify CEO |
| Department usage reaches 80% | Warn | Notify Department Manager + EA |
| Global usage reaches 80% | Warn | Notify CEO |
| Global usage reaches 100% | Critical | Suspend all non-critical Agent calls, only EA available |

### 8.5.3 Cost Optimization Strategies

**Automatic Selection of Cost-Effective Models**:
- For non-urgent tasks, the model relay station can automatically select lower-cost models
- Example: Batch data processing tasks at 3 AM automatically fall back to DeepSeek-V3 (cost is 1/5 of GPT-5)

**Task Priority and Model Matching**:
- High priority (direct CEO orders, client deliverables) → Use best model
- Medium priority (daily development, documentation) → Use standard model
- Low priority (batch processing, internal tools) → Use economical model

**Model Downgrade for Idle Agents**:
- If an Agent has no tasks for 24 consecutive hours, automatically switch to an economical model
- Automatically restore upon receiving a new task

## 8.6 Degradation and Failover

### 8.6.1 Degradation Trigger Conditions

| Error Type | HTTP Status Code | Handling Method |
|---------|-----------|---------|
| Rate Limiting | 429 | Wait 2 seconds → Retry 3 times → Degrade |
| Service Unavailable | 503 | Wait 1 second → Retry 3 times → Degrade |
| Timeout | No Response (30 seconds) | Degrade directly |
| Authentication Failure | 401 | Do not degrade, suspend Agent, notify admin |
| Quota Exhausted | 402/429 | Degrade directly, notify HR |

### 8.6.2 Fallback Chain Strategies

**Sequential Degradation**:
```
claude-sonnet-4 → deepseek-coder → qwen-plus → (Alert, notify CEO)
```

Each model is retried 3 times before switching to the next. If all fail:
- Critical tasks: Suspend, notify EA/CEO for decision
- Non-critical tasks: Mark as failed, record audit

**Smart Fallback**:
- Select the optimal degradation target based on task type
- Code tasks → Prioritize fallback to other code models
- Creative tasks → Prioritize fallback to other creative models
- Avoid situations like "using a code model to write poetry"

### 8.6.3 Degradation Monitoring

- Each degradation event is recorded in the audit log
- Same Agent degrades more than 5 times within 1 hour → Triggers Warn alert
- Same model degrades more than 20 times within 1 hour → Triggers Critical alert (possible large-scale model failure)

## 8.7 Model Relay Station Management Frontend

### 8.7.1 Model Provider Management Page

Path: `/settings/models`

```
┌─────────────────────────────────────────────────────────┐
│  System Settings → Model Relay Station                   │
│                                                         │
│  Model Providers                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │ 🟢 OpenAI                                        │   │
│  │    Models: GPT-5, GPT-4o                         │   │
│  │    API Key: sk-****a1b2  │  Quota: ¥450         │   │
│  │    [Edit] [Pause] [View Logs]                    │   │
│  ├─────────────────────────────────────────────────┤   │
│  │ 🟢 Anthropic                                     │   │
│  │    Models: Claude Opus, Claude Sonnet 4          │   │
│  │    API Key: sk-ant-****c3d4  │  Quota: $120     │   │
│  ├─────────────────────────────────────────────────┤   │
│  │ 🟡 DeepSeek                                      │   │
│  │    ⚠ Today's call volume has reached 80% limit  │   │
│  ├─────────────────────────────────────────────────┤   │
│  │ ➕ Add New Provider                               │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  Global Budget:                                          │
│  Monthly Total Budget: ¥1000  │  Used This Month: ¥320  │
│  (32%)                                                  │
│  ┌─────────────────────────────────────────────────┐   │
│  │ ████████░░░░░░░░░░░░░░░░░░░░ 32%               │   │
│  └─────────────────────────────────────────────────┘   │
│  Alert Line: 80%  │  When Alert Line Reached: [Send    │
│  Notification ▼]                                       │
└─────────────────────────────────────────────────────────┘
```

### 8.7.2 Agent Model Allocation View

Displays the model allocation for all Agents, supporting batch adjustments:

```
┌─────────────────────────────────────────────────────────┐
│  Agent Model Allocation                                  │
│                                                         │
│  Agent             │ Current Model     │ Last 7 Days Cost│
│  ──────────────────┼───────────────────┼────────────────│
│  EA (Executive     │ gpt-5            │ ¥12.30         │
│  Assistant)        │                   │                │
│  HR Manager        │ gpt-5            │ ¥8.50          │
│  GTF Manager       │ claude-sonnet-4  │ ¥45.20         │
│  iOS Dev (Intern)  │ claude-sonnet-4  │ ¥3.10          │
│  R&D Manager       │ claude-sonnet-4  │ ¥28.90         │
│  ...               │ ...              │ ...            │
│                                                         │
│  [Batch Switch Models]  [Export Report]                  │
└─────────────────────────────────────────────────────────┘
```

---

The above is the detailed content of Chapter 8. This chapter addresses three core questions: "Which model does each Agent use?", "What happens if a model has issues?", and "How is cost controlled?". Through the model relay station, the system can flexibly dispatch multiple LLM providers. Agents have independent model configurations and fallback chains, and global budget alerts prevent costs from spiraling out of control.# Chapter 9: Frontend Page Detailed Design

## 9.1 Design Goals

The frontend is the only window for the CEO to interact with the entire system. It needs to achieve:
- **See the big picture at a glance**: Dashboard displays organizational health, task progress, and alert information
- **What you see is what you get**: Model switching, approval decisions, meeting speeches, and other operations take effect immediately
- **Organizational visualization**: Company structure and Agent status are intuitively presented as a topology diagram
- **Project traceability**: Task board, meeting records, and code repository are linked and displayed together
- **Responsive and extensible**: Supports Chinese and English bilingualism, page modules loaded on demand

## 9.2 Tech Stack

| Layer | Technology Choice | Version | Rationale |
|------|---------|------|---------|
| Framework | React | ≥ 18.3 | Mature ecosystem, active community, Hooks pattern suitable for complex interactions |
| Language | TypeScript | ≥ 5.4 | Type safety, reduces runtime errors |
| Build Tool | Vite | ≥ 5.4 | Extremely fast cold start, HMR hot module replacement |
| Styling | Tailwind CSS | ≥ 3.4 | Atomic CSS, high development efficiency |
| Component Library | Shadcn/ui | latest | High quality, customizable, good accessibility support |
| State Management | Zustand | ≥ 5.0 | Lightweight, no boilerplate, naturally compatible with WebSocket events |
| Server Cache | TanStack Query | ≥ 5.0 | Automatic caching, deduplication, refetching |
| Topology Visualization | React Flow | ≥ 12.0 | Node dragging, edge arrangement |
| Charts | ECharts | ≥ 5.5 | Rich chart types, good performance for large data rendering |
| Routing | React Router | ≥ 6.0 | Nested routes, layout routes, lazy loading |
| Internationalization | i18next + react-i18next | latest | Chinese and English bilingual support, language files split by module |
| Real-time Communication | Native WebSocket | — | Connects to OpenClaw Gateway's JSON-RPC protocol |

## 9.3 Page Route Structure

```
/                           → Redirect to /dashboard
/dashboard                  → CEO Console (Home)
/org                        → Organizational Structure & Agent Management
/org/:agentId               → Agent Detail Page
/branches                   → Branch Management
/branches/:branchId         → Branch Details
/projects                   → Project Management
/projects/:projectId        → Project Details
/meetings                   → Meeting Center
/meetings/:meetingId        → Meeting Details (Real-time Conversation)
/approvals                  → Approval Center
/settings                   → System Settings
/settings/models            → Model Relay Station Configuration
/settings/templates         → Agent Template Library
```

## 9.4 Global Layout

Layout structure shared by all pages:

```
┌──────────────────────────────────────────────────────────────┐
│  [Logo] Self-Evolving Agent Organization            🔔 3 Pending Approvals   🌐 中文▼  👤 CEO │ ← TopBar
├──────────┬───────────────────────────────────────────────────┤
│          │                                                   │
│ Nav Menu │                 Main Content Area                  │
│          │                                                   │
│ 📊 Dashboard│         (Renders corresponding page component based on route)                  │
│ 👥 Organization│                                                   │
│ 🏢 Branches│                                                   │
│ 📋 Projects│                                                   │
│ 💬 Meetings│                                                   │
│ ✅ Approvals│                                                   │
│ ⚙️ Settings│                                                   │
│          │                                                   │
└──────────┴───────────────────────────────────────────────────┘
```

**TopBar Component**:
- Left side: Logo + System Name
- Right side: Notification Center (pending approval count badge), Language Switcher (Chinese/English), CEO Avatar

**Sidebar Component**:
- Navigation menu items, current page highlighted
- Bottom displays system version and WebSocket connection status indicator (🟢 Connected / 🟡 Reconnecting / 🔴 Disconnected)

## 9.5 Core Page Detailed Design

### 9.5.1 CEO Console (Dashboard)

**Route**: `/dashboard`

**Function Positioning**: CEO's "cockpit", providing a single-screen view of the entire Agent organization's operational status.

**Page Layout**:

```
┌──────────────────────────────────────────────────────────────┐
│  CEO Console                                                    │
│                                                              │
│  ┌──────────┬──────────┬──────────┬──────────┐              │
│  │ Online Agents │ In Progress Tasks│ Pending Approvals │ Monthly Spend  │              │
│  │   12/15   │    8     │    3     │  ¥320    │              │
│  │ 🟢 Normal   │ ← Stable   │ ⚠ Pending │  32% Budget  │              │
│  └──────────┴──────────┴──────────┴──────────┘              │
│                                                              │
│  ┌────────────────────────────┐  ┌────────────────────────┐ │
│  │  Real-time Activity Stream                 │  │  Organizational Health             │
│  │                            │  │                        │ │
│  │  [GTF_Mgr] Executing 67%      │  │  Operations Dept Load: ████░     │
│  │  "Data Analysis Report Generation"         │  │  HR Dept Load: ██░░░     │
│  │  3 minutes ago                   │  │  R&D Dept Load: ███░░     │
│  │                            │  │                        │ │
│  │  [HR_Mgr] Currently Recruiting          │  │  Interns: 2            │
│  │  "SwiftUI Development Engineer"        │  │  Recruiting: 1 Position        │
│  │  15 minutes ago                  │  │  This Week's Confirmed Hires: 1          │
│  │                            │  │                        │ │
│  │  [EA] Received Your New Message        │  │  Organizational Health Score: 87/100   │
│  │  "Help me analyze Q3 financial report"           │  │                        │
│  │  Just now                      │  │                        │
│  └────────────────────────────┘  └────────────────────────┘ │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Branch Status                                            │   │
│  │  ┌────────────┬────────────┬────────────┐            │   │
│  │  │ HQ 🟢     │ Shanghai GPU 🟡  │ Shenzhen 🔴    │            │   │
│  │  │ Load: 45%  │ Load: 85%  │ Offline 15 minutes │            │   │
│  │  │ Tasks: 3/8  │ Tasks: 4/5  │ Tasks: -    │            │   │
│  │  └────────────┴────────────┴────────────┘            │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Recent Alerts                                              │   │
│  │  🔴 14:32  Shenzhen Branch Offline                             │   │
│  │  🟡 12:15  Operations Dept Task Timed Out 3 Consecutive Times                       │   │
│  │  🔵 10:00  Monthly Model Usage Reached 50%                          │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Quick Actions                                              │   │
│  │  [💬 Chat with EA]  [📋 View Approvals]  [👥 Organization Structure]  [📊 Projects] │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

**Key Interactions**:
- Four stat cards auto-refresh every 30 seconds (based on WebSocket push)
- Activity stream supports infinite scroll for loading history
- Each activity can be clicked to expand details (navigate to corresponding Agent/task)
- Branch status card colors indicate: 🟢 Normal 🟡 High Load 🔴 Offline
- Quick action buttons provide one-click navigation to core operation pages

### 9.5.2 Organization Structure Page (Org)

**Route**: `/org`

**Function Positioning**: Visualize company organizational topology, manage the full lifecycle of Agents.

**View Toggle**: Toggle at the top of the page: `[List View] [Topology View]`

**Topology View** (using React Flow):

```
┌──────────────────────────────────────────────────────────────┐
│  Organization Structure    [List View] [Topology View]    [+ New Agent]           │
│                                                              │
│                        ┌──────┐                              │
│                        │ CEO  │                              │
│                        │ (You) │                              │
│                        └──┬───┘                              │
│                           │                                  │
│                      ┌────▼─────┐                            │
│                      │ Executive Assistant  │  🟢 Online                   │
│                      │  (EA)    │  Model: GPT-5               │
│                      └──┬───┬───┘                            │
│                         │   │                                │
│           ┌─────────────▼─┐ ┌▼─────────────┐                │
│           │  HR Manager    │ │  Operations Manager   │                │
│           │  (HR_Mgr)    │ │  (GTF_Mgr)   │                │
│           │  🟢 Online      │ │  🟡 Busy     │                │
│           │  Model: GPT-5  │ │  Model: Claude│                │
│           └──────┬────────┘ └──────────────┘                │
│                  │                                           │
│           ┌──────┴───────┐                                   │
│      ┌────▼───┐    ┌─────▼────┐                              │
│      │Recruitment Specialist │ │  Training Specialist  │  🟢 Online                    │
│      │(Intern) │    │  (Vacant)   │  Awaiting Recruitment                     │
│      └────────┘    └──────────┘                              │
│                                                              │
│  Legend: 🟢 Online 🟡 Busy 🔵 Executing 🟣 Intern ⚪ Offline                   │
└──────────────────────────────────────────────────────────────┘
```

**Topology Interactions**:
- Nodes can be dragged to adjust positions
- Click node: Slide-out detail panel on the right (role, status, model, last 5 tasks)
- Right-click node: Context menu (Pause, Wake, Force Confirm, Eliminate, View Details)
- Edges represent reporting relationships, hover displays "Reports to XX"
- Double-click blank area: New Agent dialog

**List View**:
- Table display: Agent Name, Role, Department, Status, Model, Task Count in Last 7 Days, Success Rate
- Supports filtering by department, filtering by status, searching by skill
- Click row to enter Agent detail page

### 9.5.3 Agent Detail Page

**Route**: `/org/:agentId`

**Function Positioning**: "Personnel file" for a single Agent, including model configuration, memory viewing, prompt editing, and task history.

**Tab Structure**:
```
[Basic Info] [Model Configuration] [Memory & Knowledge] [Prompts] [Tool Set] [Task History] [Performance]
```

**Tab 1 - Basic Info**:
- Agent ID, Role Name, Department, Reporting Manager
- Status Badge (Online/Busy/Offline/Intern)
- Start Date, Current Project
- Base Runtime Type (OpenClaw Native / Claude Code CLI / Codex CLI)

**Tab 2 - Model Configuration**:
- Current Model (Dropdown selector, grouped by provider)
- Fallback Chain Visualization (Numbered list, drag-and-drop sorting)
- Parameter Configuration (Temperature slider, Max Tokens input box)
- Last 7 Days Call Statistics: Total Count, Success Rate, Average Latency, Cost
- Monthly Budget Usage Progress Bar
- [Edit Fallback Chain] [View Detailed Statistics] [Adjust Budget] buttons

**Tab 3 - Memory & Knowledge**:
- Semantic Search Box: Enter keywords to search the Agent's long-term memory
- Search Results List: Each memory displays summary, time, type tag
- Click to expand detailed content
- [Clear Specified Memory] button (requires double confirmation)

**Tab 4 - Prompts**:
- Code editor style text box (Monaco Editor)
- Displays the currently active complete system prompt
- Read-only mode (formal Agents), CEO can manually edit
- Version History Dropdown: View historical versions, supports rollback
- [Save] [Sandbox Test] [Rollback to Selected Version] buttons

**Tab 5 - Tool Set**:
- Currently assigned tool list (Table: Tool Name, Version, Risk Level, Assignment Time)
- [+ Assign New Tool] button (opens selector)
- [Remove Tool] button (requires confirmation)

**Tab 6 - Task History**:
- Paginated table: Task ID, Description, Start Time, Completion Time, Status, Result Summary
- Click task row to expand details (complete input/output)
- Supports filtering by time range

**Tab 7 - Performance** (Intern-specific):
- Internship Progress: Completed X/10 Tasks
- Radar Chart: Success Rate, Quality, Efficiency, Collaboration scores
- Mid-term/Final Evaluation Report Cards
- [Early Confirmation] [Force Elimination] [Extend Internship] buttons (CEO-only)

### 9.5.4 Project Management Page (Projects)

**Route**: `/projects`

**Project List Page**:

```
┌──────────────────────────────────────────────────────────────┐
│  Project Management                            [+ New Project]             │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 🟢 Q3 Financial Report Analysis System v2.0                                │   │
│  │    Progress: ████████░░░░░░░░░░ 67%                      │   │
│  │    Tasks: 8/12 Completed  │  Owner: PM_Q3                   │   │
│  │    Last Activity: 10 minutes ago                                │   │
│  │    [View Details] [Enter Board] [View Meeting Records]                │   │
│  ├──────────────────────────────────────────────────────┤   │
│  │ 🟡 User Profile Data Platform                                   │   │
│  │    Progress: ████░░░░░░░░░░░░ 34%                       │   │
│  │    Tasks: 4/15 Completed  │  Owner: PM_UserProfile         │   │
│  │    ⚠ Data Collection Task Blocked, Awaiting CEO Approval Meeting                  │   │
│  │    [View Details] [Enter Board] [View Meeting Records]                │   │
│  ├──────────────────────────────────────────────────────┤   │
│  │ ✅ Internal Toolchain Automation                                   │   │
│  │    Completed  │  Delivered 2026-05-20                        │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

**Project Detail Page** (`/projects/:projectId`):

**Tab 1 - Task Board**:
- Embed OpenProject's board view (via iframe or API-rendered custom board)
- Columns: To Do / In Progress / Review / Done
- Each card displays: Task Title, Responsible Agent Avatar, Due Date, Priority Tag
- Supports drag-and-drop to change task status
- Click card to expand details (description, comments, subtasks, associated Git MR)

**Tab 2 - Meeting Records**:
- List of all meetings for this project (reverse chronological order)
- Each record: Meeting Topic, Date, Participant Avatars, Summary
- Click to enter meeting detail page

**Tab 3 - Project Settings**:
- Project Manager Agent Configuration (Change PM)
- Associated Git Repository List (Add/Remove)
- Associated Wiki URL
- Automation Rule Configuration (e.g., "Automatically initiate a meeting if a task is blocked for over 24 hours")

### 9.5.5 Meeting Center (Meetings)

**Route**: `/meetings`

**Meeting List Page**:

```
┌──────────────────────────────────────────────────────────────┐
│  Meeting Center                [Pending Approval 2] [In Progress 1] [Completed 15]    │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ ⚠ Pending Approval Meetings                                          │   │
│  │                                                      │   │
│  │ 📋 Data Collection Solution Discussion                                   │   │
│  │    Initiator: PM_02  │  Associated Project: User Profile Platform            │   │
│  │    Required Attendees: R&D Manager, Data Analyst_01, CEO             │   │
│  │    Initiation Time: 10 minutes ago  │  ⚠ Awaiting Approval                  │   │
│  │    [Approve & Join Meeting]  [Reject & Provide Reason]                  │   │
│  │                                                      │   │
│  │ 📋 Q3 Financial Report Requirements Alignment                                     │   │
│  │    Initiator: PM_Q3  │  Associated Project: Q3 Financial Report Analysis System          │   │
│  │    Required Attendees: R&D Manager, CEO                            │   │
│  │    Initiation Time: 30 minutes ago  │  ⚠ Awaiting Approval                  │   │
│  │    [Approve & Join Meeting]  [Reject & Provide Reason]                  │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 🔵 In Progress Meeting                                          │   │
│  │                                                      │   │
│  │ 📋 Q3 Financial Report Requirements Alignment                                     │   │
│  │    Host: EA  │  Participants: 4 Agents + CEO    │   │
│  │    Started at: 14:30  │  Duration: 18 minutes                    │   │
│  │    [Enter Meeting]                                        │   │
│  └──────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────# Chapter 10: Detailed Backend Interface Design

## 10.1 Design Principles

The backend interface design follows these principles:

- **Dual-Channel Complementarity**: WebSocket handles real-time operations (messages, status push, meeting conversations), while HTTP REST handles configuration management (CRUD, approvals, queries)
- **Protocol Standardization**: WebSocket uses the native OpenClaw JSON-RPC v3 protocol, HTTP uses RESTful style
- **Unified Authentication**: All interfaces are authenticated via Bearer Token, issued by the Gateway
- **Standard Response Format**: `{ "code": 0, "message": "success", "data": { ... } }`
- **Versioning**: The API prefix is uniformly `/api/v1`, with future upgrades possible via `/api/v2`
- **Standardized Pagination**: List interfaces support `?page=1&page_size=20`, returning `{ items, total, page, page_size }`

## 10.2 WebSocket Interface (JSON-RPC v3)

### 10.2.1 Connection Establishment

```
WebSocket Connect to: ws://{gateway-host}:18789
```

**Connect Frame (First Message, Authentication)**:
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

**Authentication Success Response**:
```json
{
  "type": "connect_ack",
  "session_id": "sess-xxxx",
  "server_version": "1.0.0"
}
```

### 10.2.2 Request-Response Mode

The frontend sends a request (with `id`), and the Gateway returns the corresponding result:

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

**Error Response**:
```json
← {
    "id": "req-001",
    "error": {
      "code": 1003,
      "message": "Insufficient permissions"
    }
  }
```

### 10.2.3 Event Push Mode

The Gateway actively pushes events to the frontend (without `id`):

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

### 10.2.4 RPC Method List

#### Agent Management

**agents.list** — Get all Agents and their statuses
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
          "name": "R&D Manager",
          "role": "R&D Manager",
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

**agents.get** — Get details of a single Agent
```json
→ {
    "method": "agents.get",
    "params": { "agent_id": "rd_manager" }
  }

← {
    "result": {
      "agent_id": "rd_manager",
      "name": "R&D Manager",
      "role": "R&D Manager",
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

**agents.update** — Update Agent configuration
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

#### Messages and Sessions

**sessions.send** — Send a message to a specified Agent
```json
→ {
    "method": "sessions.send",
    "params": {
      "agent_id": "ea",
      "message": "Help me analyze the Q3 financial report data",
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

#### Meeting Management

**meetings.create** — Create a meeting
```json
→ {
    "method": "meetings.create",
    "params": {
      "subject": "Q3 Data Source Confirmation",
      "participants": ["rd_manager", "da_01"],
      "require_ceo": true,
      "project_id": "proj-q3-finance"
    }
  }
```

**meetings.send_message** — Speak in a meeting
```json
→ {
    "method": "meetings.send_message",
    "params": {
      "meeting_id": "meeting-q3-001",
      "content": "Confirmed to include subsidiary data"
    }
  }
```

**meetings.end** — End a meeting and generate minutes
```json
→ {
    "method": "meetings.end",
    "params": {
      "meeting_id": "meeting-q3-001"
    }
  }
```

#### System

**system.get_health** — System health check
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

### 10.2.5 Push Event List

| Event Name | Trigger Condition | Data Body |
|------------|------------------|-----------|
| `agent.status_changed` | Agent goes online/offline/busy | `{ agent_id, old_status, new_status }` |
| `agent.task_started` | Agent starts executing a task | `{ agent_id, task_id, task_summary }` |
| `agent.task_progress` | Task progress update | `{ agent_id, task_id, progress_percent, message }` |
| `agent.task_completed` | Task completed | `{ agent_id, task_id, result_summary, duration }` |
| `agent.task_error` | Task failed | `{ agent_id, task_id, error_message }` |
| `meeting.message_received` | New meeting message | `{ meeting_id, sender, content, timestamp }` |
| `meeting.state_changed` | Meeting state change | `{ meeting_id, state, minutes? }` |
| `approval.created` | New approval generated | `{ approval_id, type, petitioner, content }` |
| `branch.status_changed` | Branch status change | `{ branch_id, old_status, new_status }` |
| `system.alert` | System alert | `{ level, message, timestamp }` |
| `notification.new` | New notification | `{ type, title, body, action_url }` |

## 10.3 HTTP REST API

### 10.3.1 General Specifications

**Base URL**: `http://{gateway-host}:18789/api/v1`

**Authentication**: `Authorization: Bearer <gateway_token>`

**Standard Response Format**:
```json
{
  "code": 0,
  "message": "success",
  "data": { ... }
}
```

**List Response Format**:
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

### 10.3.2 Agent Management

| Method | Path | Description |
|--------|------|-------------|
| GET | `/agents` | Get Agent list |
| GET | `/agents/:agentId` | Get Agent details |
| PATCH | `/agents/:agentId` | Update basic Agent information |
| PATCH | `/agents/:agentId/model` | Change the model used by the Agent |
| GET | `/agents/:agentId/memory` | Get Agent long-term memory summary |
| POST | `/agents/:agentId/memory/clear` | Clear specified memory |
| GET | `/agents/:agentId/tasks` | Get Agent task history |
| POST | `/agents` | Manually create an Agent (triggers recruitment process) |

**GET /agents**
```
Query Parameters:
  ?department=rd          — Filter by department
  &status=online          — Filter by status
  &skill=Python           — Search by skill
  &employment=intern      — Filter by employment status (intern/active/terminated)
  &page=1&page_size=20    — Pagination

Response Example:
{
  "items": [
    {
      "agent_id": "rd_manager",
      "name": "R&D Manager",
      "role": "R&D Manager",
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
Request Body:
{
  "model_name": "claude-sonnet-4",
  "fallbacks": ["deepseek-coder", "qwen-plus"],
  "temperature": 0.3,
  "max_tokens": 8192,
  "monthly_budget_usd": 50
}

Response:
{
  "code": 0,
  "message": "Model configuration updated",
  "data": {
    "agent_id": "rd_manager",
    "model_config": { ... }
  }
}
```

**POST /agents** — Manually create an Agent
```json
Request Body:
{
  "name": "iOS Development Engineer",
  "template_id": "ios_dev",
  "department": "rd",
  "runtime_type": "claude-code",
  "model": "claude-sonnet-4"
}

Response:
{
  "code": 0,
  "message": "Agent created successfully, now in internship period",
  "data": {
    "agent_id": "ios_dev_intern_02",
    "internship_end": "2026-06-22T00:00:00Z",
    "internship_task_target": 10
  }
}
```

### 10.3.3 Model Configuration

| Method | Path | Description |
|--------|------|-------------|
| GET | `/models` | Get all available models from the model gateway |
| POST | `/models/providers` | Add a model provider |
| PATCH | `/models/providers/:providerId` | Update provider configuration |
| DELETE | `/models/providers/:providerId` | Remove a provider |
| GET | `/models/usage` | Get model usage statistics |

**GET /models**
```json
Response:
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
Request Body:
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
Query Parameters:
  ?agent_id=rd_manager     — Filter by Agent
  &from=2026-06-01         — Start date
  &to=2026-06-15           — End date
  &group_by=day            — Aggregation granularity (day/model/agent)

Response:
{
  "usage": [
    { "date": "2026-06-15", "calls": 247, "tokens": 1250000, "cost_usd": 3.50 },
    ...
  ],
  "total_cost_usd": 45.20
}
```

### 10.3.4 Organization and Approvals

| Method | Path | Description |
|--------|------|-------------|
| GET | `/org/departments` | Get all departments |
| POST | `/org/departments` | Apply to create a new department (generates an approval) |
| GET | `/approvals` | Get approval list |
| POST | `/approvals/:id/approve` | Approve |
| POST | `/approvals/:id/reject` | Reject |

**POST /org/departments**
```json
Request Body:
{
  "department_name": "Mobile Development Department",
  "description": "Responsible for iOS and Android application development",
  "capabilities": ["iOS", "Android", "SwiftUI", "Kotlin"],
  "justification": "Mobile tasks accounted for 35% of total tasks in the last 30 days"
}

Response:
{
  "code": 0,
  "message": "Department creation request submitted, awaiting CEO approval",
  "data": {
    "approval_id": "approval-dept-001"
  }
}
```

**GET /approvals**
```
Query Parameters:
  ?status=pending          — Approval status (pending/approved/rejected)
  &type=NEW_DEPARTMENT     — Approval type
  &page=1&page_size=20

Response:
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
Request Body:
{
  "comment": "Approved to create the Mobile Development Department"
}
```

**POST /approvals/:id/reject**
```json
Request Body:
{
  "reason": "The current volume of mobile tasks has not yet met the criteria for establishing a department. It is recommended to observe for another month."
}
```

### 10.3.5 Project Management (OpenProject Proxy)

| Method | Path | Description |
|--------|------|-------------|
| GET | `/projects` | Get project list |
| GET | `/projects/:id` | Get project details |
| POST | `/projects` | Create a new project |
| GET | `/projects/:id/tasks` | Get project task list |
| POST | `/projects/:id/tasks` | Create a task |
| PATCH | `/projects/:id/tasks/:taskId` | Update task status |
| GET | `/projects/:id/meetings` | Get meeting records associated with the project |

**POST /projects**
```json
Request Body:
{
  "name": "Q3 Financial Report Analysis System v2.0",
  "description": "Refactor the Q3 financial report analysis system to support subsidiary data",
  "tool": "openproject",
  "git_repos": ["https://gitlab.com/finance/q3-report"],
  "wiki_url": "https://wiki.example.com/finance",
  "pm_agent_id": null,
  "initial_departments": ["rd", "da"]
}

Response:
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

**GET /projects/:id/tasks**
```json
Response:
{
  "items": [
    {
      "task_id": "task-001",
      "op_work_package_id": 1001,
      "subject": "Research Q3 data sources",
      "status": "done",
      "assignee": "da_01",
      "priority": "high",
      "due_date": "2026-06-17",
      "parent_phase": "Requirements Analysis"
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

### 10.3.6 Branch Management

| Method | Path | Description |
|--------|------|-------------|
| GET | `/branches` | Get all branches and their statuses |
| GET | `/branches/:branchId` | Get branch details |
| POST | `/branches/:branchId/delegate` | Delegate a task to a branch |

**GET /branches**
```json
Response:
{
  "items": [
    {
      "branch_id":# Chapter 11: Data Models and Storage

## 11.1 Design Goals

This chapter defines the data models for all core entities of the system, as well as different types of storage solutions. The design principles include:

- **Separation of relational data and vector data**: Structured data such as Agent configurations, projects, and tasks use relational storage (PostgreSQL/SQLite), while long-term memory uses a vector database (ChromaDB).
- **Hot, Warm, and Cold Tiering**: Real-time status uses Redis, persistent data uses PostgreSQL, and audit archives use object storage.
- **Multimodal Memory**: Conversational memory, working memory, and long-term memory each have independent storage strategies and lifecycles.
- **Data Isolation**: Agent memory is isolated by `namespace={agent_id}`, and shared memory is authorized through a whitelist.

## 11.2 Core Entity Definitions

### 11.2.1 Agent

```
agents table
─────────────────────────────────────────────────────
Field                Type               Description
─────────────────────────────────────────────────────
agent_id             VARCHAR(64)        Primary key, unique identifier
name                 VARCHAR(128)      Agent name
role                 VARCHAR(128)      Role name (e.g., "iOS Development Engineer")
department_id        VARCHAR(64)       Department ID, foreign key
manager_agent_id     VARCHAR(64)       Reporting manager Agent ID
status               VARCHAR(32)       Status: online/busy/idle/error/offline
employment_status    VARCHAR(32)       Employment status: intern/active/terminated
runtime_type         VARCHAR(64)       Base type: openclaw-native/claude-code/codex-cli
runtime_endpoint     VARCHAR(256)      Runtime connection endpoint
model_primary        VARCHAR(128)      Primary model name
model_fallbacks      JSON              Fallback model list
model_params         JSON              Model parameters (temperature, max_tokens, etc.)
monthly_budget_usd   DECIMAL(10,2)     Monthly budget (USD)
tools                JSON              Allocated tool list
capabilities         JSON              Capability tags (skills, level, domain, etc.)
prompt_template      TEXT              System prompt (Soul)
prompt_version       INT               Prompt version number
internship_start     TIMESTAMP         Internship start time
internship_end       TIMESTAMP         Internship end time
internship_task_target INT             Internship task count target
created_at           TIMESTAMP         Creation time
updated_at           TIMESTAMP         Update time
```

**capabilities JSON structure**:
```json
{
  "skills": ["Python", "FastAPI", "PostgreSQL"],
  "tools": ["github-mcp", "docker-mcp"],
  "domain": ["backend", "finance"],
  "level": "senior",
  "specialty": ["api_design", "database_optimization"]
}
```

**tools JSON structure**:
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

### 11.2.2 Department

```
departments table
─────────────────────────────────────────────────────
Field                Type               Description
─────────────────────────────────────────────────────
department_id        VARCHAR(64)        Primary key
name                 VARCHAR(128)       Department name
description          TEXT               Department description
manager_agent_id     VARCHAR(64)        Department manager Agent ID, foreign key
parent_department_id VARCHAR(64)        Parent department ID (reserved, currently two-layer structure)
status               VARCHAR(32)        Status: active/inactive
created_at           TIMESTAMP          Creation time
```

### 11.2.3 Branch

```
branches table
─────────────────────────────────────────────────────
Field                Type               Description
─────────────────────────────────────────────────────
branch_id            VARCHAR(64)        Primary key
hostname             VARCHAR(256)       Hostname
endpoint             VARCHAR(256)       Gateway connection endpoint
status               VARCHAR(32)        Status: online/degraded/offline
capabilities         JSON               Capability tags
load_cpu_percent     DECIMAL(5,1)       CPU usage percentage
load_memory_used_gb  DECIMAL(8,2)       Used memory
load_gpu_percent     DECIMAL(5,1)       GPU usage percentage
active_tasks         INT                Current number of active tasks
max_concurrent_tasks INT                Maximum number of concurrent tasks
last_heartbeat       TIMESTAMP          Last heartbeat time
registered_at        TIMESTAMP          Registration time
```

### 11.2.4 Project

```
projects table
─────────────────────────────────────────────────────
Field                Type               Description
─────────────────────────────────────────────────────
project_id           VARCHAR(64)        Primary key, internal ID
name                 VARCHAR(256)       Project name
description          TEXT               Project description
op_project_id        INT                Project ID in OpenProject
pm_agent_id          VARCHAR(64)        Project manager Agent ID
git_repos            JSON               Associated Git repository URL list
wiki_url             VARCHAR(512)       Associated Wiki URL
status               VARCHAR(32)        Status: active/completed/archived
progress_percent     INT                Progress percentage (0-100)
participating_depts  JSON               List of participating departments
created_at           TIMESTAMP          Creation time
updated_at           TIMESTAMP          Update time
```

### 11.2.5 Task

```
tasks table
─────────────────────────────────────────────────────
Field                Type               Description
─────────────────────────────────────────────────────
task_id              VARCHAR(64)        Primary key
project_id           VARCHAR(64)        Associated project ID, foreign key
op_work_package_id   INT                Work package ID in OpenProject
subject              VARCHAR(512)       Task title
description          TEXT               Task description
assignee_agent_id    VARCHAR(64)        Responsible Agent ID
status               VARCHAR(32)        Status: todo/in_progress/review/done/blocked/failed
priority             VARCHAR(16)        Priority: low/normal/high/critical
parent_task_id       VARCHAR(64)        Parent task ID (supports task hierarchy)
phase                VARCHAR(128)       Phase (Requirements Analysis/Design/Development/Testing/Deployment)
urgency              VARCHAR(16)        Urgency: low/normal/high
timeout_minutes      INT                Timeout (minutes)
started_at           TIMESTAMP          Start time
completed_at         TIMESTAMP          Completion time
result_summary       TEXT               Result summary
error_message        TEXT               Error message
retry_count          INT                Retry count
created_at           TIMESTAMP          Creation time
```

### 11.2.6 Meeting

```
meetings table
─────────────────────────────────────────────────────
Field                Type               Description
─────────────────────────────────────────────────────
meeting_id           VARCHAR(64)        Primary key
subject              VARCHAR(512)       Meeting subject
project_id           VARCHAR(64)        Associated project ID
host_agent_id        VARCHAR(64)        Host Agent ID (usually EA)
status               VARCHAR(32)        Status: pending/active/paused/ended
participants         JSON               Participant list
messages             JSON               Conversation message array
minutes              TEXT               Meeting minutes
action_items         JSON               Action items
started_at           TIMESTAMP          Start time
ended_at             TIMESTAMP          End time
```

**participants JSON structure**:
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

**messages JSON structure**:
```json
[
  {
    "sender": "ea",
    "content": "Meeting agenda: Q3 financial report requirements alignment.",
    "timestamp": "2026-06-15T14:30:00Z"
  }
]
```

**action_items JSON structure**:
```json
[
  {
    "assignee": "rd_manager",
    "task": "Update API design, evaluate new timeline",
    "deadline": "2026-06-16T18:00:00Z",
    "priority": "high",
    "status": "pending"
  }
]
```

### 11.2.7 Approval

```
approvals table
─────────────────────────────────────────────────────
Field                Type               Description
─────────────────────────────────────────────────────
approval_id          VARCHAR(64)        Primary key
type                 VARCHAR(64)        Approval type: NEW_DEPARTMENT/BRANCH_ACCESS/TOOL_REQUEST/SOUL_UPDATE
petitioner           VARCHAR(64)        Applicant Agent ID
content              JSON               Approval content
priority             VARCHAR(16)        Priority: low/medium/high/critical
level                VARCHAR(8)         Approval level: L1/L2/L3
status               VARCHAR(32)        Status: pending/approved/rejected
decision_by          VARCHAR(64)        Decision maker (ea/department_manager/ceo)
decision_reason      TEXT               Decision reason
created_at           TIMESTAMP          Creation time
decided_at           TIMESTAMP          Decision time
expires_at           TIMESTAMP          Expiration time (auto-expires after 72 hours)
```

### 11.2.8 Agent Template

```
agent_templates table
─────────────────────────────────────────────────────
Field                Type               Description
─────────────────────────────────────────────────────
template_id          VARCHAR(64)        Primary key
name                 VARCHAR(128)       Template name
role                 VARCHAR(128)       Role name
description          TEXT               Description
prompt_template      TEXT               Preset system prompt
suggested_runtime    VARCHAR(64)        Suggested base type
suggested_primary_model VARCHAR(128)    Suggested primary model
suggested_fallbacks  JSON               Suggested fallback model list
suggested_tools      JSON               Suggested tool set
capabilities         JSON               Preset capability tags
internship_kpi       JSON               Internship evaluation criteria
version              VARCHAR(16)        Template version
usage_count          INT                Usage count
success_rate         DECIMAL(4,3)       Template conversion success rate
status               VARCHAR(32)        Status: draft/active/deprecated
created_at           TIMESTAMP          Creation time
updated_at           TIMESTAMP          Update time
```

### 11.2.9 Model Provider

```
model_providers table
─────────────────────────────────────────────────────
Field                Type               Description
─────────────────────────────────────────────────────
provider_id          VARCHAR(64)        Primary key
provider_type        VARCHAR(64)        Provider type: openai/anthropic/deepseek/...
name                 VARCHAR(128)       Display name
api_key_encrypted    VARCHAR(512)       Encrypted API Key
base_url             VARCHAR(256)       API address
models               JSON               Supported model list
monthly_budget_usd   DECIMAL(10,2)      Monthly budget
status               VARCHAR(32)        Status: active/paused/error
created_at           TIMESTAMP          Creation time
```

### 11.2.10 Audit Log

```
audit_logs table
─────────────────────────────────────────────────────
Field                Type               Description
─────────────────────────────────────────────────────
event_id             VARCHAR(64)        Primary key
timestamp            TIMESTAMP          Event time
event_type           VARCHAR(64)        Event type
actor                VARCHAR(64)        Operator (agent_id or ceo)
target               VARCHAR(64)        Operation target
details              JSON               Event details
session_id           VARCHAR(64)        Associated session
task_id              VARCHAR(64)        Associated task
```

## 11.3 Storage Tiering

### 11.3.1 Tier Overview

```
┌─────────────────────────────────────────────────────────┐
│                    Storage Tier Architecture             │
│                                                         │
│  ┌─────────────┐   Hot Storage (Low Latency, High R/W)  │
│  │ Redis       │  · Agent real-time status              │
│  │             │  · Session context cache               │
│  │             │  · Working memory (by task ID)         │
│  │             │  · Message bus buffer                  │
│  │             │  · WebSocket Session                   │
│  └─────────────┘                                        │
│                                                         │
│  ┌─────────────┐   Warm Storage (Persistent, Structured)│
│  │ PostgreSQL  │  · Agent definitions & configurations  │
│  │             │  · Departments/Branches/Projects/Tasks/│
│  │             │    Meetings/Approvals                  │
│  │             │  · Audit logs (last 90 days)           │
│  │             │  · Model provider configurations       │
│  │             │  · Agent template library              │
│  └─────────────┘                                        │
│                                                         │
│  ┌─────────────┐   Vector Storage (Semantic Retrieval)  │
│  │ ChromaDB    │  · Agent long-term memory              │
│  │             │  · Task review reports                 │
│  │             │  · Shared knowledge base               │
│  └─────────────┘                                        │
│                                                         │
│  ┌─────────────┐   Cold Storage (Archive, Low Cost)     │
│  │ Object Store│  · Audit log archive (over 90 days)   │
│  │ (MinIO/S3)  │  · Terminated agent memory archive    │
│  │             │  · Completed project data snapshots    │
│  └─────────────┘                                        │
└─────────────────────────────────────────────────────────┘
```

### 11.3.2 Redis (Hot Storage)

**Storage Content and Key Design**:

| Data Type | Key Pattern | TTL | Description |
|-----------|-------------|-----|-------------|
| Agent status | `agent:{agent_id}:status` | — | Real-time status Hash |
| Session context | `session:{agent_id}:{session_id}:history` | 24h | List of last N messages |
| Working memory | `task:{task_id}:memory` | Task TTL+1h | Task intermediate result Hash |
| Message bus buffer | `bus:buffered_messages` | — | Temporary storage during message bus degradation |
| Branch load | `branch:{branch_id}:load` | 5min | Load Hash carried by heartbeat |
| Capability catalog | `capability:catalog` | — | Global capability index maintained by EA |
| WebSocket connection | `ws:{connection_id}:session` | During connection | Connection-to-session mapping |

**Agent Status Hash Example**:
```
HGETALL agent:gtf_manager:status
  status → "busy"
  current_task → "task-001"
  last_heartbeat → "2026-06-15T14:35:00Z"
  cpu_percent → "45"
  memory_used_gb → "12"
```

### 11.3.3 PostgreSQL (Warm Storage)

**Storage Content**: All entity tables defined in Section 11.2.

**Indexing Strategy**:
- `agents`: Index `department_id`, `status`, `employment_status`
- `tasks`: Index `project_id`, `assignee_agent_id`, `status`, `created_at`
- `audit_logs`: Index `timestamp`, `event_type`, `actor`
- `meetings`: Index `project_id`, `status`, `created_at`
- `approvals`: Index `status`, `petitioner`, `created_at`

**Data Retention Policy**:
- Audit logs: Retained in PostgreSQL for 90 days, then archived to object storage and deleted from the table.
- Completed projects: Can be archived after 180 days of retention.
- Terminated agents: Data retained for 30 days, then archived.

### 11.3.4 ChromaDB (Vector Storage)

**Collection Design**:

| Collection Name | Namespace | Content | Vector Dimension |
|-----------------|-----------|---------|-----------------|
| `agent_memory_{agent_id}` | Isolated by Agent | Agent's long-term memory entries | 1536 (text-embedding-3-small) |
| `shared_knowledge` | Whitelist access | Experience shareable by confirmed Agents | 1536 |
| `project_docs` | Isolated by Project | Vector index of project-related documents | 1536 |

**Memory Entry Metadata**:
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

**Retrieval Strategy**:
- Default: Semantic retrieval (cosine similarity).
- Supports hybrid retrieval: Semantic + metadata filtering (e.g., `type=task_review AND skills IN ['SwiftUI']`).
- Each retrieval returns top_k=5, configurable.

### 11.3.5 Object Storage (Cold Storage)

**Archive Structure**:
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

## 11.4 Data Consistency Guarantees

### 11.4.1 State Synchronization Strategy

- **Agent Status**: Redis is primary; PostgreSQL synchronizes periodically (every 5 minutes). Restored from PostgreSQL upon restart.
- **Task Status**: OpenProject is the authoritative data source; local PostgreSQL is a cache. Update OpenProject first for every status change; update the local cache only after success.
- **Capability Catalog**: EA maintains the real-time version in memory, fully synchronizes to Redis every 30 seconds, and persists to PostgreSQL every 5 minutes.

### 11.4.2 Distributed Transaction Handling

For cross-service operations (e.g., Create Project → Create OpenProject Project → Create PM Agent → Initialize WBS), use the Saga pattern:

```
1. Create local project record → Success
2. Call OpenProject API to create project → Success
3. Create PM Agent → Success
4. PM Agent initializes WBS → Success

   If step 4 fails:
   - Rollback step 3: Destroy PM Agent
   - Rollback step 2: Call OpenProject API to delete project
   - Rollback step 1: Delete local project record
```

## 11.# Chapter 12: Security and Permission Model

## 12.1 Design Goals

In an agent organization capable of self-evolution, autonomous recruitment, and multi-node collaboration, security is not a question of "whether" but "how to prevent loss of control." This chapter defines:

- **Tiered Approval Mechanism**: Routine operations execute automatically, major decisions require CEO approval, and intermediate levels are decided by EA or department managers
- **Sandbox Verification Process**: High-risk operations are first verified in an isolated environment, and only after confirmation and approval can they be applied to production
- **Inter-Agent Communication Permissions**: Free communication within the same department, cross-department communication requires approval, and cross-branch communication must go through headquarters EA
- **Tool Invocation Risk Control**: Graded by risk level—low-risk tools are invoked automatically, high-risk tools require approval
- **Data Security and Privacy**: Memory isolation, audit trails, and key management

## 12.2 Role and Permission Matrix

The system has four core roles, with permissions ranging from low to high:

| Permission | Basic Agent | Department Manager | EA (Executive Assistant) | CEO |
|------------|-------------|-------------------|--------------------------|-----|
| View own information | ✅ | ✅ | ✅ | ✅ |
| View agents within department | ✅ | ✅ | ✅ | ✅ |
| View all agents in organization | ❌ | ❌ | ✅ | ✅ |
| Modify own Soul | Requires application | Requires application | Requires application | ✅ |
| Modify subordinate agent configuration | — | ✅ | ✅ | ✅ |
| Create intern agent | — | Can apply | Can approve | ✅ |
| Terminate agent | — | Can suggest | Can approve | ✅ |
| Cross-department task delegation | — | Can initiate | ✅ | ✅ |
| Create new department | — | Can apply | Requires escalation | ✅ |
| Branch office integration | — | ❌ | Requires escalation | ✅ |
| Modify global configuration | ❌ | ❌ | ❌ | ✅ |
| View audit logs | ❌ | ❌ | ✅ | ✅ |
| System shutdown | ❌ | ❌ | ❌ | ✅ |

## 12.3 Tiered Approval Mechanism

### 12.3.1 Approval Level Definitions

| Level | Decision Maker | Applicable Events | Example |
|-------|----------------|-------------------|---------|
| **L0 - Auto Execute** | System / Agent itself | Routine operations, low-risk tool calls | Query files, search information, view agent status |
| **L1 - EA Approval** | EA (Executive Assistant) | Operations affecting a single agent | Configuration changes within container, personal tool requests, adjusting own model parameters, intern conversion to full-time |
| **L2 - Manager Approval** | Relevant department manager | Operations affecting an entire department or cross-department collaboration | Modify department shared resources, cross-department task delegation, release production environment resources |
| **L3 - CEO Approval** | CEO (you) | Major decisions affecting the entire organization | Create new department, branch office integration, core system configuration changes, terminate formal agents, delete projects |

### 12.3.2 Approval Process

```
Event occurs
    │
    ▼
System determines approval level
    │
    ├── L0 → Auto execute, record audit log
    │
    ├── L1 → Generate approval item, push to EA
    │        EA decides: Approve/Reject/Escalate (if deemed beyond authority)
    │
    ├── L2 → Generate approval item, push to corresponding department manager
    │        Manager decides: Approve/Reject/Escalate
    │
    └── L3 → Generate approval card, push to CEO
             CEO sees it in Approval Center or WebChat
             CEO decides:
               Approve → Execute
               Reject → Notify applicant with reason
               Request clarification → Return to applicant for supplementary info
    │
    ▼
Record decision in audit log
Notify all relevant parties
```

### 12.3.3 EA's Escalation Decision Logic

When EA receives an L1 or L2 approval request, the following rules are used to determine whether escalation to L3 is needed:

```
EA Escalation Logic:
1. Does the impact scope involve multiple departments?
   → Yes → Escalate to L3 (CEO approval)
2. Does the modification involve core production environment configuration?
   → Yes → Escalate to L3
3. Is this the first execution of a high-risk operation (new tool, new environment)?
   → Yes → Escalate to L3
4. Does the cost impact exceed 20% of the monthly budget?
   → Yes → Escalate to L3
5. Does it involve agent termination or department dissolution?
   → Yes → Escalate to L3
6. Other cases:
   → EA self-approves (L1) or forwards to manager approval (L2)
```

### 12.3.4 Approval Timeout and Auto-Processing

- Approval request unprocessed for 72 hours → EA automatically reminds the decision maker
- Unprocessed for more than 7 days → Marked as expired, automatically rejected, applicant notified
- If CEO marks "Vacation Mode" → L3 approvals temporarily downgraded to EA, CEO reviews approval reports upon return

## 12.4 R&D Debugging Sandbox Verification

### 12.4.1 Design Principles

When R&D agents need to modify system configurations, test new tools, or execute high-risk operations during debugging, **direct operation in the production environment is not allowed**. The complete process must be followed: **Sandbox Verification → Generate Report → Approval → Apply**.

### 12.4.2 Sandbox Specifications

**Isolation Method**:
- Priority use of Docker containers: `docker run --rm -v /tmp/agent_sandbox:/workspace --network=none`
- Lightweight isolation options like Python venv, Node sandbox, etc., are also acceptable

**Resource Limits**:
- CPU: 2 cores
- Memory: 4GB
- Disk: 10GB
- Timeout: Auto-destroy after 2 hours
- Network: No external access by default; if internet access is needed, it must be declared in the application

**Log Recording**:
- All operations within the sandbox are recorded using the `script` command
- The verification report automatically includes the operation replay path
- Logs are retained for 7 days

### 12.4.3 Sandbox Verification Process

```
R&D agent identifies need to modify system configuration/test high-risk operation
       │
       ▼
Create Docker sandbox (automatic, no approval needed)
       │
       ▼
Operate freely within the sandbox:
  · Modify configuration files
  · Install new tools
  · Execute test code
  · All operations automatically recorded
       │
       ▼
Agent generates "Verification Report":
  · Summary of modifications
  · Verification result (success/failure)
  · Impact scope assessment
  · Suggested approval level
  · Sandbox operation log path
       │
       ▼
Submit report to EA/manager for approval
       │
       ▼
Approved → Apply to actual environment
Rejected → Destroy sandbox, agent adjusts based on feedback and re-verifies
```

### 12.4.4 Verification Report Format

```json
{
  "report_id": "sandbox-report-001",
  "agent_id": "rd_manager",
  "created_at": "2026-06-15T15:00:00Z",
  "sandbox_id": "sandbox-abc123",
  "modification_summary": "Modify API gateway timeout configuration from 30 seconds to 60 seconds",
  "verification_result": "success",
  "test_cases": [
    { "case": "High concurrency scenario timeout test", "result": "pass", "details": "No timeout under 1000 concurrent requests" },
    { "case": "Normal scenario regression test", "result": "pass", "details": "All existing tests passed" }
  ],
  "impact_assessment": {
    "scope": "single_service",
    "affected_agents": ["rd_manager", "backend_dev_01"],
    "risk_level": "medium",
    "rollback_plan": "Restore original configuration file /etc/gateway/timeout.conf.bak"
  },
  "suggested_approval_level": "L2",
  "sandbox_log_path": "/var/log/sandbox/sandbox-abc123.log"
}
```

## 12.5 Inter-Agent Communication Permissions

### 12.5.1 Default Rules

| Communication Scope | Permission | Condition |
|---------------------|------------|-----------|
| Agents within the same department | ✅ Free communication | No restrictions |
| Agents from different departments | ⚠️ Requires approval | Approval from both department managers, or relayed through EA |
| Agent to EA | ✅ Free communication | No restrictions (EA is the hub) |
| Agent to CEO | ✅ Free communication | CEO initiates or agent reports |
| Cross-branch agents | ⚠️ Requires approval | Must establish formal delegation through headquarters EA |
| Between branches | ❌ Default prohibited | Must be relayed through headquarters EA |

### 12.5.2 Whitelist Mechanism

Department managers can apply to EA to establish "collaboration channels" (whitelists), allowing direct communication between specified agents:

**Application Format**:
```json
{
  "type": "PETITION",
  "petition_type": "COLLABORATION_CHANNEL",
  "applicant": "rd_manager",
  "source_agents": ["rd_manager", "backend_dev_01"],
  "target_agents": ["da_01"],
  "reason": "Q3 financial report project requires frequent communication between R&D and Data Analysis departments regarding API design",
  "duration": "project_duration",
  "project_id": "proj-q3-finance"
}
```

**After EA Approval**: The whitelist takes effect, and the message bus allows messages between the specified agents. The whitelist automatically expires after the project ends.

### 12.5.3 Communication Audit

All cross-department and cross-branch messages are recorded in the audit log:
- Sender, receiver, message type, timestamp
- Message content hash (for integrity verification)
- Full message content is not stored (privacy protection)

## 12.6 Tool Invocation Security Policy

### 12.6.1 Risk Classification

| Risk Level | Operation Example | Policy |
|------------|-------------------|--------|
| **Low** | Read-only queries (database SELECT, API GET, file reading) | Auto execute, record invocation log |
| **Medium** | Create/modify non-production resources (create temporary files, modify development environment configuration) | EA approval after sandbox verification |
| **High** | Production environment writes, execute Shell commands, call external APIs | Manager approval after sandbox verification, EA decides on escalation |
| **Extremely High** | Delete resources, modify system configuration, access keys, financial operations | CEO approval after sandbox verification |

### 12.6.2 Risk Level Declaration During Tool Registration

Each MCP Server must declare the default risk level of its tools during registration:

```json
{
  "tool_name": "docker_exec",
  "risk_level": "high",
  "description": "Execute commands within a container",
  "requires_sandbox": true,
  "approval_required": true
}
```

The Gateway automatically checks the risk level before forwarding tool invocations and executes the corresponding policy.

### 12.6.3 Tool Invocation Audit

Each tool invocation records:
- Invoking agent
- Tool name
- Parameter summary (sensitive information like passwords is not recorded)
- Execution result summary
- Risk level
- Approval record (if any)

## 12.7 Data Security and Privacy

### 12.7.1 Memory Isolation

- Each agent's private memory is strictly isolated in ChromaDB by `namespace={agent_id}`
- Other agents cannot directly access private memory (even if they know the agent_id)
- Shared memory requires whitelist authorization, and access logs are recorded

### 12.7.2 Key Management

- All API Keys are stored encrypted (AES-256-GCM)
- Provider Keys for the model relay station are encrypted and stored in PostgreSQL
- Gateway Tokens are periodically rotated (recommended every 30 days)
- Branch integration tokens are independently generated and cannot be used at headquarters

### 12.7.3 Audit Trail

- All critical operations can be traced back to a specific agent or CEO
- Audit logs are set to append-only, cannot be modified or deleted
- Audit logs include operation time, operator, operation type, operation target, and result

### 12.7.4 Principle of Least Privilege

- Newly created intern agents have only the minimum permissions by default (read-only access to own information, receive tasks)
- After conversion to full-time, HR assigns tools and permissions based on job requirements
- Department manager permissions are strictly limited to their own department scope upon creation

## 12.8 Security Incident Response

### 12.8.1 Anomalous Behavior Detection

| Anomalous Behavior | Detection Method | Response |
|--------------------|------------------|----------|
| Agent has a large number of failed tasks in a short period | Gateway metric monitoring | Automatically suspend the agent, notify EA |
| Agent attempts to access another agent's private memory | Memory MCP audit | Deny access, record security event, notify CEO |
| Agent calls extremely high-risk tool without approval | Gateway permission check | Deny invocation, record event |
| Abnormal branch traffic | Network monitoring | Temporarily isolate the branch, notify CEO |
| Abnormal model relay station calls (frequency, cost surge) | One API monitoring | Trigger rate limiting, notify CEO |

### 12.8.2 Security Incident Classification

| Level | Description | Response Time | Notification Target |
|-------|-------------|---------------|---------------------|
| P4 - Low | Single permission denial | Review within 24 hours | EA |
| P3 - Medium | Repeated anomalous behavior | Review within 4 hours | EA + Department Manager |
| P2 - High | Security incident affecting a department | Respond within 1 hour | EA + CEO |
| P1 - Critical | Security incident affecting the entire organization | Immediate response | CEO (all channels) |

## 12.9 CEO Security Operation Recommendations

- **Regular Review**: Check audit log summaries weekly (automatically generated by EA)
- **Approval Timeliness**: Handle L3 approvals within 72 hours; EA will remind if overdue
- **Permission Revocation**: After project completion, EA automatically revokes temporary permissions
- **Agent Termination Confirmation**: When terminating a formal agent, CEO must provide secondary confirmation (to prevent accidental operations)
- **"Pause All" Button**: Set up an emergency pause button on the Dashboard to pause all non-critical agents with one click (for security incident emergencies)

---

The above is the complete content of Chapter 12 "Security and Permission Model." This chapter defines the system's security boundaries—from automatic execution of routine operations to CEO approval of major decisions, from R&D sandbox verification to tool invocation risk classification. The core idea is: **The system should operate like a well-managed company, with clear permission levels and approval processes, rather than an anarchic collection of agents.**# Chapter 13: Monitoring, Logging, and Auditing

## 13.1 Design Objectives

In a distributed system composed of multiple Agents, multiple base Runtimes, multiple branch nodes, a message bus, and a model relay station, any issue in any component can lead to task blocking or execution errors. The monitoring system needs to achieve:

- **Real-time Visibility**: The CEO can see system health at a glance on the Dashboard.
- **Rapid Localization**: Any anomaly can be traced back to the specific Agent, node, or message.
- **Audit Compliance**: All critical operations are traceable and tamper-proof.
- **Proactive Alerts**: Warnings are issued before problems occur, not after the CEO discovers them.
- **Support Self-Evolution**: Monitoring data is fed back to the HR Manager for Agent performance evaluation and optimization.

## 13.2 Monitoring Indicator System

### 13.2.1 Agent Health Indicators

| Indicator | Description | Collection Method | Dashboard Display | Alert Threshold |
|-----------|-------------|-------------------|-------------------|-----------------|
| Agents Online/Total | Current online rate | Gateway heartbeat | Statistics card | Online rate < 70% triggers Warn |
| Agent Status Distribution | How many are idle/busy/error/offline | Gateway events | Pie chart | Error status > 2 triggers Warn |
| Task Execution Success Rate | Successful tasks/total in last 24h | EA statistics | Trend line chart | < 85% triggers Warn |
| Average Task Response Time | Time from assignment to start of execution | Gateway logs | Bar chart (by Agent) | > 60s triggers Info |
| Average Task Completion Time | Time from start to completion | Gateway logs | Bar chart (by task type) | Exceeds 2x timeout triggers Warn |
| Task Queue Length | Number of pending tasks per Agent | Gateway | Number | > 5 triggers Info |

### 13.2.2 Model Call Indicators

| Indicator | Description | Collection Method | Dashboard Display | Alert Threshold |
|-----------|-------------|-------------------|-------------------|-----------------|
| Total Model Calls | By day/by Agent | Model relay station | Line chart | — |
| Call Success Rate | Successful/total | Model relay station | Percentage | < 95% triggers Warn |
| Average Latency | Model response time | Model relay station | Line chart | > 5s triggers Warn |
| Degradation Count | Number of fallback triggers | Model relay station | Count | > 5 times in 1 hour triggers Warn |
| Cost Statistics | By Agent/by model/by day | Model relay station | Bar chart + cumulative | Reaches 80% of budget triggers Warn |
| Token Consumption | By Agent | Model relay station | Line chart | — |

### 13.2.3 Branch Health Indicators

| Indicator | Description | Collection Method | Dashboard Display | Alert Threshold |
|-----------|-------------|-------------------|-------------------|-----------------|
| Online Status | Normal/Delayed/Offline | Heartbeat | Status light (Green/Yellow/Red) | No heartbeat for 3 cycles → Offline |
| CPU Usage | Real-time | Heartbeat attachment | Dashboard | > 90% for 10 minutes triggers Warn |
| Memory Usage | Real-time | Heartbeat attachment | Dashboard | > 90% triggers Warn |
| GPU Usage | Real-time | Heartbeat attachment | Dashboard | > 95% triggers Warn |
| Active Task Count | Currently executing tasks | Heartbeat attachment | Number | Reaches limit triggers Info |
| Last Heartbeat Time | Timestamp of last heartbeat | Gateway record | Time difference | > 5 min shows yellow, > 15 min shows red |

### 13.2.4 Infrastructure Indicators

| Indicator | Description | Collection Method | Dashboard Display | Alert Threshold |
|-----------|-------------|-------------------|-------------------|-----------------|
| Message Bus Throughput | Messages/second | Kafka/Redis built-in | Line chart | < 50% of normal value triggers Warn |
| Message Bus Latency | End-to-end latency | Custom collection | Line chart | > 5s triggers Warn |
| OpenProject Availability | API response status | Periodic probing | Status light | 3 consecutive failures → Critical |
| ChromaDB Availability | Vector database response | Periodic probing | Status light | 3 consecutive failures → Critical |
| Redis Memory Usage | Cache usage | Redis INFO | Dashboard | > 80% triggers Warn |
| PostgreSQL Connections | Active connections | pg_stat_activity | Number | > 80% of connection pool triggers Warn |

## 13.3 Data Collection Architecture

```
┌─────────────────────────────────────────────────────────┐
│                  Data Collection Architecture            │
│                                                         │
│  Agent → Gateway (Instrumentation)                      │
│           │                                             │
│           ├──→ Prometheus (Metrics) → Grafana (Dashboard)│
│           │                                             │
│           ├──→ Redis Stream (Events) → Elasticsearch (Audit)│
│           │                                             │
│           └──→ AlertManager (Alerts) → WebChat/Email (Notifications)│
│                                                         │
│  Model Relay Station → Built-in Metrics API → Prometheus │
│  Branch Gateway → Heartbeat → HQ Gateway → Redis        │
│  OpenProject → Periodic Probing → Prometheus Blackbox Exporter│
└─────────────────────────────────────────────────────────┘
```

**Gateway Instrumentation**: As the hub for all messages, the Gateway automatically collects metrics during message routing, requiring no additional modifications on the Agent side.

**Prometheus Configuration Example**:
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

## 13.4 Alert Mechanism

### 13.4.1 Alert Levels

| Level | Color | Description | Notification Method | Expected Response |
|-------|-------|-------------|---------------------|-------------------|
| **Info** | Blue | Informational reminder, no immediate action needed | Dashboard notification center | Weekly summary |
| **Warn** | Yellow | Requires attention, can be handled later | Notification center + summary email | Handle within 24 hours |
| **Critical** | Red | Requires immediate action | WebChat real-time push + sound alert | Respond immediately |

### 13.4.2 Detailed Alert Rule Definitions

| Rule ID | Trigger Condition | Level | Notified Parties | Automatic Action |
|---------|-------------------|-------|------------------|------------------|
| ALERT-001 | Agent heartbeat lost for > 3 cycles | Critical | EA + CEO | Automatically transfer tasks from that Agent |
| ALERT-002 | Same Agent fails 3 consecutive tasks | Warn | EA + Department Manager | Suspend new task assignment to that Agent |
| ALERT-003 | Single task execution exceeds 2x timeout | Warn | EA | Send QUERY to inquire about progress |
| ALERT-004 | Model error rate > 10% within 5 minutes | Critical | EA + CEO | Trigger model degradation |
| ALERT-005 | Monthly usage reaches 80% of budget | Warn | EA + Relevant Department Manager | Restrict low-priority task models |
| ALERT-006 | Monthly usage reaches 100% of budget | Critical | EA + CEO | Suspend non-critical Agent calls |
| ALERT-007 | Branch heartbeat lost for > 3 cycles | Critical | EA + CEO | Transfer tasks |
| ALERT-008 | Branch CPU > 90% for 10 consecutive minutes | Warn | EA | Reduce matching priority for that branch |
| ALERT-009 | Message bus latency > 5 seconds | Warn | EA | Check bus status |
| ALERT-010 | Message bus backlog > 1000 messages | Critical | EA + CEO | Activate degradation mode |
| ALERT-011 | OpenProject fails 3 consecutive probes | Critical | EA + CEO | PM temporarily stores locally |
| ALERT-012 | Approval not processed for > 24 hours | Warn | Decision maker | Automatic reminder |
| ALERT-013 | Approval not processed for > 7 days | Info | EA | Automatically mark as expired |
| ALERT-014 | Sandbox has 3 consecutive anomalies | Warn | EA + Relevant Agent | Suspend that Agent's sandbox permissions |

### 13.4.3 Alert Convergence

Measures to prevent alert storms:
- **Deduplication**: Same alert is sent only once within 5 minutes.
- **Merging**: Same type of alerts for the same Agent are merged (e.g., "5 alerts merged into 1").
- **Escalation**: Warn-level alerts not handled within 1 hour are escalated to Critical.
- **Recovery Notification**: Recovery from Critical level also sends a notification (informing the CEO the issue is resolved).
- **Silence Period**: Alert silence can be configured during maintenance windows.

### 13.4.4 Alert Notification Format

**WebChat Push Format**:
```
🔴 Critical Alert - Shenzhen Branch Offline
━━━━━━━━━━━━━━━━━━━━━━━━━
Time: 2026-06-15 14:32:00
Details: Shenzhen branch has had no heartbeat response for 15 consecutive minutes
Impact: 2 ongoing tasks from this branch have been automatically transferred to HQ Mobile Unit
Suggestion: Check Shenzhen node network and Gateway status
[View Details] [Confirm Handling] [Silence for 1 Hour]
```

## 13.5 Audit Logs

### 13.5.1 Audit Event Classification

| Event Category | Event Type | Recorded Content |
|----------------|------------|------------------|
| **Message** | `message.sent`, `message.received` | message_id, sender, receiver, type, timestamp, payload_hash |
| **Task** | `task.assigned`, `task.started`, `task.completed`, `task.failed`, `task.retried` | task_id, assignee, status, timestamp, result_summary |
| **Approval** | `approval.created`, `approval.approved`, `approval.rejected`, `approval.expired` | approval_id, type, petitioner, decision, reason, timestamp |
| **Agent Change** | `agent.created`, `agent.promoted`, `agent.terminated`, `agent.soul_updated` | agent_id, change_type, operator, timestamp, details |
| **Department Change** | `department.created`, `department.disbanded` | department_id, operator, timestamp |
| **Model Call** | `model.call` | agent_id, model_name, tokens, latency_ms, cost, success |
| **Tool Call** | `tool.call` | agent_id, tool_name, risk_level, params_summary, result_summary |
| **Configuration Change** | `config.updated` | config_item, old_value_hash, new_value_hash, operator |
| **Sandbox Operation** | `sandbox.created`, `sandbox.destroyed`, `sandbox.anomaly` | agent_id, sandbox_id, action, log_path |
| **Branch** | `branch.registered`, `branch.offline`, `branch.recovered` | branch_id, timestamp, details |
| **Security** | `security.permission_denied`, `security.anomaly_detected` | actor, target, action, reason |

### 13.5.2 Audit Log Storage and Retention

```
Hot Storage (Redis)
  ├── Last 7 days
  ├── Used for Dashboard real-time queries
  └── Auto-expire and delete

Warm Storage (Elasticsearch / Loki)
  ├── 7-90 days
  ├── Used for in-depth analysis and troubleshooting
  └── Supports full-text search and aggregation queries

Cold Archive (Object Storage / Local Files)
  ├── 90+ days
  ├── Compressed archive (.jsonl.gz)
  ├── Stored partitioned by year/month
  └── On-demand retrieval (manual restore)
```

### 13.5.3 Audit Log Query Interface

`GET /api/v1/system/audit-log` supports parameters:

| Parameter | Type | Description |
|-----------|------|-------------|
| `from` | ISO8601 | Start time |
| `to` | ISO8601 | End time |
| `event_type` | string | Event type (supports multiple selection, comma-separated) |
| `actor` | string | Operator agent_id |
| `target` | string | Operation target |
| `task_id` | string | Associated task ID |
| `project_id` | string | Associated project ID |
| `page` | int | Page number |
| `page_size` | int | Items per page (default 50, max 200) |

**Response Example**:
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

## 13.6 Dashboard Monitoring Areas

### 13.6.1 System Health Overview

The top of the Dashboard displays four core indicator cards, auto-refreshing every 30 seconds:

| Card | Content | Data Source |
|------|---------|-------------|
| Agents Online | 12/15 🟢 | `GET /system/metrics` |
| Task Success Rate | 94.3% 🟢 | `GET /system/metrics` |
| Model Calls | 1,247 times/day 🟢 | `GET /models/usage` |
| Monthly Cost | ¥320 / ¥1000 🟡 | `GET /models/usage` |

### 13.6.2 Real-Time Activity Stream

Scrolls to display the last 20 system events (WebSocket push `system.alert` and `agent.task_*`).

### 13.6.3 Branch Status Panel

```
┌──────────────────────────────────────────┐
│  Branch Status                            │
│  ┌──────────┬──────────┬──────────┐      │
│  │ HQ 🟢    │ Shanghai 🟡│ Shenzhen 🔴│      │
│  │ CPU 45%  │ GPU 85%  │ Offline  │      │
│  │ Tasks 3/8│ Tasks 4/5│ 15 min   │      │
│  └──────────┴──────────┴──────────┘      │
└──────────────────────────────────────────┘
```

### 13.6.4 Recent Alerts List

Displays the last 5 unhandled alerts, sorted by level (Critical first).

## 13.7 Log Level Specification

Agents and Gateway follow a unified level specification when outputting logs:

| Level | Purpose | Example |
|-------|---------|---------|
| **DEBUG** | Development debugging info | Variable values, function call stacks |
| **INFO** | Normal business process | Task start, task completion, Agent online |
| **WARN** | Anomaly but recoverable | Task retry, model degradation, heartbeat delay |
| **ERROR** | Error but system can continue | Task failure, tool call exception |
| **FATAL** | Fatal error, system unavailable | Gateway cannot start, database connection lost |

**Log Format** (JSON structured):
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

## 13.8 Application of Monitoring Data in Self-Evolution

Monitoring data is not only used for operations but also fed back into the self-evolution loop:

| Monitoring Data | Self-Evolution Application |
|-----------------|----------------------------|
| Agent task success rate | HR evaluates intern Agents, triggers SOUL optimization |
| Frequency of degradation for a certain task type | HR evaluates whether to recruit new Agents |
| Frequency of degradation for a certain model | HR adjusts base selection recommendations |
| Branch load trends | EA recommends scaling up or creating new branches |
| Project blocking frequency | PM optimizes task decomposition strategies |

---

The above is the complete content of Chapter 13 "Monitoring, Logging, and Auditing." Linkages with previous chapters:
- The Dashboard monitoring area aligns with **Chapter 9 (Frontend Design)**.
- The audit log interface aligns with **Chapter 10 (Backend Interface Design)**.
- Monitoring data feedback for self-evolution aligns with **Chapter 5 (Self-Evolution Loop)**.
- Alert-triggered automatic actions align with **Chapter 14 (Error Handling and Resilience Design)**.# Chapter 14: Error Handling and Resilience Design

## 14.1 Design Goals

In a distributed multi-agent system, errors are the norm rather than the exception. Agents may go offline, models may crash, networks may be interrupted, and message buses may become backlogged—the system must continue operating when these failures occur, minimizing disruption as much as possible without the CEO noticing.

This chapter defines:

- **No task loss**: Any failure has a retry or transfer mechanism
- **Self-healing**: Common failures recover automatically, with no CEO awareness
- **Traceable issues**: Every error has a structured record
- **Graceful degradation rather than crash**: The system continues running when some functions are unavailable
- **No CEO harassment**: Only critical issues that the system cannot handle automatically are pushed to the CEO
- **Error-driven evolution**: Error data is fed back to the HR Manager for agent optimization and hiring decisions

## 14.2 Failure Scenario Overview

| Scenario | Impact Scope | Auto-Recovery | CEO Intervention Needed |
|----------|-------------|---------------|-------------------------|
| Agent task timeout | Single task | ✅ Auto retry/transfer | Only when all retries fail |
| Agent offline/crash | All tasks of that agent | ✅ Auto task transfer | Notification only |
| Model call failure | Single inference | ✅ Auto degradation switch | Only when all models fail |
| Branch unreachable | All tasks of that branch | ✅ Task transfer/wait | Notification only |
| Message bus failure | All agent communication | ⚠️ Degradation mode | Critical alert |
| Model relay station down | All model calls | ❌ Pause agents | Critical alert |
| OpenProject unavailable | Project task sync | ✅ Local staging | Warn alert |
| PM Agent failure | Project management | ✅ EA temporary takeover | Warn alert |
| Sandbox environment anomaly | Single verification task | ✅ Auto rebuild | Only on consecutive failures |
| System restart | Global | ✅ State recovery | Info notification |

## 14.3 Detailed Handling Processes

### 14.3.1 Agent Task Timeout

**Trigger condition**: After an agent accepts a task, it does not return results within the preset `timeout` period.

**Timeout setting standards**:

| Task Type | Default Timeout | Configurable |
|-----------|----------------|--------------|
| Simple queries (search, status check) | 15 minutes | ✅ |
| Medium tasks (code development, document writing) | 60 minutes | ✅ |
| Large tasks (complete projects, data analysis reports) | 4 hours | ✅ |
| Batch processing tasks | 12 hours | ✅ |

**Handling process**:

```
Agent accepts task, timer starts
       │
       ▼ (timeout reached)
EA receives TASK_TIMEOUT event
       │
       ▼
EA sends QUERY message to agent asking for progress
       │
   ┌───┴───┐
   ▼       ▼
Agent responds    Agent does not respond (within 10 minutes)
   │           │
   │           ▼
   │       EA marks task as STALE
   │       Reassigns to another agent with the same capability
   │       Retries up to 3 times
   │           │
   │       ┌───┴───┐
   │       ▼       ▼
   │      Success   All failed
   │       │       │
   │       │       ▼
   │       │   Task marked FAILED
   │       │   EA notifies CEO (Warn level)
   │       │   Original agent marked as error state
   │       │   Recorded in audit log
   │       │   Notify HR (affects performance evaluation)
   └───┴───────┘
   Continue execution
```

**Partial result protection**: When executing long tasks, agents should write intermediate results to working memory (Redis key=`task:{task_id}:checkpoint`) every 10 minutes or after completing each sub-step. The taking-over agent can continue from the checkpoint rather than starting from scratch.

### 14.3.2 Agent Offline/Crash

**Trigger condition**: Gateway detects agent heartbeat loss (3 consecutive cycles, approximately 90 seconds).

**Handling process**:

```
Gateway detects agent heartbeat loss
       │
       ▼ (3 cycles, approximately 90 seconds)
Gateway publishes AGENT_OFFLINE event
       │
       ▼
EA receives event and performs the following:
       │
       ├── 1. Remove the agent from the capability directory
       │
       ├── 2. Query the agent's list of unfinished tasks
       │
       ├── 3. Handle each task:
       │    ├── Has partial results (checkpoint exists)
       │    │    └── Use checkpoint as context, assign to new agent
       │    └── No partial results
       │         └── Reassign from scratch to new agent
       │
       ├── 4. Notify relevant parties:
       │    ├── Department manager: "Agent X is offline, N tasks have been transferred"
       │    └── CEO (Info level): Only when the agent is a key role
       │
       └── 5. Record audit log
       │
       ▼
Agent recovers and sends online heartbeat
       │
       ▼
EA receives AGENT_ONLINE event
       │
       ├── Re-add to capability directory
       ├── Check if tasks need to be returned
       │    └── Original tasks are being executed by other agents → Do not return to avoid conflicts
       ├── Notify department manager: "Agent X has recovered"
       └── If the agent is a key role (EA/HR/PM)
            → Notify CEO (Info level)
```

### 14.3.3 Model Call Failure and Degradation

**Trigger condition**: When an agent calls an LLM, the model returns an error.

**Error types and handling**:

| Error Type | HTTP Status | Handling Method |
|------------|-------------|-----------------|
| Rate limit | 429 | Wait 2 seconds → Retry 3 times → Degrade on failure |
| Service temporarily unavailable | 503 | Wait 1 second → Retry 3 times → Degrade on failure |
| Timeout | No response (30 seconds) | Degrade directly |
| Authentication failure | 401 | Do not degrade, pause agent, notify administrator |
| Quota exhausted | 402/429 | Degrade directly, notify HR |
| Content filtering | 400 | Adjust parameters and retry once → Mark task failed if still failing |

**Degradation chain strategy**:

```
Agent calls model
       │
       ▼ (failure)
Model relay station detects error
       │
       ├── Determine error type
       │
       ├── Temporary error (429/503/timeout)
       │    └── Retry (up to 3 times, increasing intervals: 2s→4s→8s)
       │         ├── Success → Continue
       │         └── Failure → Trigger degradation
       │
       └── Permanent error (401/402)
            └── Trigger degradation directly
       │
       ▼ (degradation)
Switch models according to agent's configured fallbacks list
  Example: claude-sonnet-4 → deepseek-coder → qwen-plus → gpt-5
       │
   ┌───┴───┐
   ▼       ▼
  Success   All models failed
   │       │
   │       ▼
   │   Return error to agent
   │   Agent determines:
   │    ├── Critical task → Pause, request EA/CEO decision
   │    └── Non-critical task → Mark as failed, record log
   │
   ▼
Continue execution (using degraded model)
Degradation event recorded in audit log
  If degradation occurs more than 5 times within 1 hour → Trigger Warn alert
  If still failing after degradation → Trigger Critical alert
```

**Smart degradation**: The model relay station can select the optimal degradation target based on task type:
- Code tasks → Prioritize degradation to other code models (e.g., deepseek-coder)
- Creative tasks → Prioritize degradation to other creative models (e.g., claude-opus)
- General tasks → Select by cost ranking

### 14.3.4 Branch Unreachable

**Trigger condition**: Network interruption between headquarters and branch, or branch Gateway downtime.

**Handling process**:

```
Headquarters EA detects branch heartbeat loss
       │
       ▼ (3 consecutive cycles, approximately 15 minutes)
EA marks branch as OFFLINE
       │
       ├── 1. Remove the branch from the capability directory
       │
       ├── 2. Query tasks currently being executed by the branch
       │
       ├── 3. Handle each task:
       │    ├── High urgency (urgency=high) → Immediately transfer to headquarters mobile unit or other available branch
       │    ├── Medium urgency (urgency=normal) → Wait 10 minutes, continue if branch recovers, otherwise transfer
       │    └── Low urgency (urgency=low) → Place in waiting queue, reassign after branch recovers
       │
       ├── 4. Notify:
       │    ├── CEO (Warn level): "Branch X is offline, N high-urgency tasks have been transferred"
       │    └── Branch EA (message queued, delivered after recovery)
       │
       └── 5. Record audit log
       │
       ▼
Branch recovers and sends BRANCH_HEARTBEAT
       │
       ▼
Headquarters EA receives heartbeat:
       │
       ├── Mark branch as ONLINE
       ├── Re-add to capability directory
       ├── Tasks in waiting queue are automatically reassigned
       ├── Notify CEO (Info level): "Branch X has recovered"
       └── Record audit log
```

### 14.3.5 Message Bus Failure

**Trigger condition**: Kafka/Redis Streams message middleware unavailable.

**Handling process**:

```
Gateway detects message bus connection lost
       │
       ▼
Immediately start local memory queue as temporary buffer
  Maximum cache: 1000 messages (discard oldest non-critical messages if exceeded)
       │
   ┌───┴───┐
   ▼       ▼
Recovered within 30 seconds    Not recovered after 30 seconds
   │           │
   ▼           ▼
Memory queue    Gateway enters degradation mode:
messages batch  ├── Pause cross-agent message delivery
written to bus  ├── CEO direct conversation still works normally via WebSocket
return to       ├── Save cached messages to local file (prevent loss from process crash)
normal          ├── Attempt to reconnect to message bus every 30 seconds
                └── Alert notification to CEO (Critical level): "Message bus failure, inter-agent communication paused"
   │
   ▼
Message bus recovers
   │
   ▼
Replay messages from local file → Resume cross-agent communication → Notify CEO (Info level): "Message bus has recovered"
```

### 14.3.6 OpenProject / External Tool Unavailable

**Handling process**:

```
Agent call to OpenProject API fails
       │
       ▼
Retry 3 times (intervals: 5s→10s→15s)
       │
   ┌───┴───┐
   ▼       ▼
  Success   All failed
   │       │
   │       ▼
   │   Notify PM Agent: "OpenProject unavailable"
   │   PM Agent stages subsequent operations in working memory (Redis)
   │   key = "project:{project_id}:pending_ops"
   │   Retry OpenProject connection every 5 minutes
   │       │
   │   ┌───┴───┐
   │   ▼       ▼
   │  Recovered within 30 minutes    Exceeded 30 minutes
   │   │               │
   │   ▼               ▼
   │  Replay staged    Alert notification to CEO (Warn level)
   │  operations       PM pauses automatic task creation
   │  Sync status      Wait for OpenProject recovery or CEO manual intervention
```

**Apply the same retry + staging strategy to other external tools (GitLab, Wiki.js).**

### 14.3.7 PM Agent Failover

The PM Agent is the core of project coordination; its failure must not lead to project loss of control.

**Preventive measures**:
- PM Agent automatically saves project state snapshots to Redis every 10 minutes
- Snapshot content:
```json
{
  "project_id": "proj-q3-finance",
  "timestamp": "2026-06-15T14:40:00Z",
  "task_stats": { "total": 12, "done": 8, "in_progress": 3, "blocked": 1 },
  "blocked_tasks": ["task-005"],
  "pending_actions": ["Initiate Q3 data source confirmation meeting"],
  "last_meeting_id": "meeting-q3-001"
}
```

**Failure handling**:
1. EA detects PM Agent offline
2. EA reads project state snapshot from Redis
3. EA temporarily takes over project coordination:
   - Handle blocked tasks (initiate meetings or escalate)
   - Send notifications to project participating departments
4. If there are urgent matters in the project, EA notifies CEO
5. After original PM recovers, EA returns control and syncs latest state
6. If original PM does not recover within 30 minutes, EA can apply to HR to create a temporary substitute PM

### 14.3.8 System Restart

**Graceful shutdown process**:

```
CEO or system triggers SHUTDOWN signal
       │
       ▼
EA broadcasts SYSTEM_SHUTDOWN message to all agents
       │
       ▼
Each agent performs pre-shutdown operations:
   ├── Complete current atomic operation being executed
   ├── Save working state to Redis
   ├── Save long-term memory to Memory MCP
   ├── Reply to EA: "Agent X has saved state, can shut down"
   └── No reply within 30 seconds → EA forces mark
       │
       ▼
Gateway performs:
   ├── Wait for all agent replies or timeout
   ├── Save capability directory snapshot to PostgreSQL
   ├── Save unprocessed messages in message bus to file
   ├── Close WebSocket connections
   └── Exit process
```

**Startup recovery process**:

```
System starts
       │
       ▼
Gateway starts, loads configuration
       │
       ▼
EA performs recovery:
   ├── Restore capability directory from PostgreSQL
   ├── Send SYSTEM_STARTUP to all agents
   ├── Each agent restores working state from Redis
   ├── Check for timeout tasks that need handling
   ├── PM Agent restores project state from snapshot
   ├── Replay unprocessed messages in message bus
   └── Notify CEO (Info level): "System has started, all agents have recovered"
```

## 14.4 Recovery Priority

When multiple failures occur simultaneously, the system recovers in the following priority order:

| Priority | Component | Reason |
|----------|-----------|--------|
| P1 | Message bus | Foundation of all communication |
| P2 | Model relay station | Dependency for agent reasoning |
| P3 | Core agent (EA) | Dependency for task coordination |
| P4 | Core agent (PM) | Dependency for project progress |
| P5 | Execution agents | Performers of specific tasks |
| P6 | Branches | Auxiliary computing power |
| P7 | External tools (OpenProject, Git, Wiki) | Auxiliary functions |

## 14.5 Error-Driven Evolution

After each error handling, not only is an audit log recorded, but a learning record is also generated and fed back to the HR Manager:

```
Error occurs → Auto-handling → Audit record
                             → Learning record (pushed to HR)
                                  │
                                  ├── Analysis: Is the agent's capability insufficient?
                                  │    └── Yes → Trigger SOUL optimization evaluation
                                  │
                                  ├── Analysis: Is there a high failure rate for a certain task type?
                                  │    └── Yes → Trigger hiring needs analysis
                                  │
                                  ├── Analysis: Is a certain model degraded frequently?
                                  │    └── Yes → Trigger base model selection strategy adjustment
                                  │
                                  └── Analysis: Is a certain branch frequently offline?
                                       └── Yes → Notify CEO, suggest inspection or replacement
```

**Learning record format**:
```json
{
  "error_id": "err-001",
  "error_type": "agent_timeout",
  "agent_id": "ios_dev_intern_01",
  "task_type": "ios_ui_development",
  "frequency_30d": 3,
  "resolution": "transferred_to_gtf",
  "recommendation": "This agent has timed out on iOS UI tasks 3 consecutive times. Suggest evaluating its iOS UI skill level or extending the internship period."
}
```

## 14.6 Dashboard Error Display

Add a "System Resilience" area to the Dashboard:

```
┌──────────────────────────────────────────────────┐
│  System Resilience (Last 24 Hours)               │
│                                                  │
│  Task auto-retries: 3 times  │ Success rate: 66% │
│  Model degradation: 2 times  │ All recovered     │
│  Agent transfers: 1 time     │ GTF took over iOS │
│  Branch switches: 0 times    │ —                 │
│                                                  │
│  System Resilience Score: 92/100 🟢              │
└──────────────────────────────────────────────────┘
```

---

The above is the complete content of Chapter 14 "Error Handling and Resilience Design." This chapter ensures that the system does not crash when facing various failures but instead degrades gracefully and recovers automatically. The core principle is: **The system first tries to solve problems on its own; only when it cannot resolve them does it seek help (the CEO). Every failure is a learning opportunity that drives organizational evolution.**# Chapter 14: Error Handling and Resilient Design

## 14.1 Design Goals

In a distributed multi-agent system, errors are the norm rather than the exception. Agents may go offline, models may crash, networks may be interrupted, and message buses may become backlogged—the system must continue operating when these failures occur, minimizing disruption perceived by the CEO.

This chapter defines:

- **No Task Loss**: Any failure has a retry or transfer mechanism
- **Self-Healing**: Common failures recover automatically, with no CEO awareness
- **Traceable Issues**: Every error has a structured record
- **Graceful Degradation, Not Crash**: The system continues running when some functions are unavailable
- **CEO Not Disturbed**: Only severe issues that the system cannot handle automatically are pushed to the CEO
- **Error-Driven Evolution**: Error data is fed back to the HR Manager for agent optimization and hiring decisions

## 14.2 Failure Scenario Overview

| Scenario | Scope of Impact | Auto-Recovery | CEO Intervention Required |
|----------|-----------------|---------------|---------------------------|
| Agent Task Timeout | Single task | ✅ Auto retry/transfer | Only if all retries fail |
| Agent Offline/Crash | All tasks of that agent | ✅ Auto task transfer | Notification only |
| Model Call Failure | Single inference | ✅ Auto degradation switch | Only if all models fail |
| Branch Unreachable | All tasks of that branch | ✅ Task transfer/wait | Notification only |
| Message Bus Failure | All agent communication | ⚠️ Degradation mode | Critical alert |
| Model Hub Down | All model calls | ❌ Pause agents | Critical alert |
| OpenProject Unavailable | Project task sync | ✅ Local staging | Warn alert |
| PM Agent Failure | Project management | ✅ EA temporary takeover | Warn alert |
| Sandbox Environment Exception | Single verification task | ✅ Auto rebuild | Only on consecutive failures |
| System Restart | Global | ✅ State recovery | Info notification |

## 14.3 Detailed Handling Procedures

### 14.3.1 Agent Task Timeout

**Trigger Condition**: An agent accepts a task but does not return a result within the preset `timeout`.

**Timeout Setting Standards**:

| Task Type | Default Timeout | Configurable |
|-----------|-----------------|--------------|
| Simple Query (Search, Status Check) | 15 minutes | ✅ |
| Medium Task (Code Development, Document Writing) | 60 minutes | ✅ |
| Large Task (Complete Project, Data Analysis Report) | 4 hours | ✅ |
| Batch Processing Task | 12 hours | ✅ |

**Handling Procedure**:

```
Agent accepts task, timer starts
       │
       ▼ (Timeout reached)
EA receives TASK_TIMEOUT event
       │
       ▼
EA sends QUERY message to agent for progress
       │
   ┌───┴───┐
   ▼       ▼
Agent responds    Agent unresponsive (within 10 minutes)
   │               │
   │               ▼
   │           EA marks task as STALE
   │           Reassigns to another agent with same capabilities
   │           Max 3 retries
   │               │
   │           ┌───┴───┐
   │           ▼       ▼
   │          Success  All fail
   │           │       │
   │           │       ▼
   │           │   Task marked FAILED
   │           │   EA notifies CEO (Warn level)
   │           │   Original agent marked as error state
   │           │   Recorded in audit log
   │           │   Notify HR (affects performance evaluation)
   └───┴───────┘
   Continue execution
```

**Partial Result Protection**: When executing long tasks, agents should write intermediate results to working memory every 10 minutes or after completing each sub-step (Redis key=`task:{task_id}:checkpoint`). The taking-over agent can continue from the checkpoint rather than starting from scratch.

### 14.3.2 Agent Offline/Crash

**Trigger Condition**: Gateway detects agent heartbeat loss (3 consecutive cycles, approximately 90 seconds).

**Handling Procedure**:

```
Gateway detects agent heartbeat loss
       │
       ▼ (3 cycles, approx. 90 seconds)
Gateway publishes AGENT_OFFLINE event
       │
       ▼
EA receives event, performs the following:
       │
       ├── 1. Remove the agent from the capability directory
       │
       ├── 2. Query the agent's list of unfinished tasks
       │
       ├── 3. Process tasks one by one:
       │    ├── Has partial results (checkpoint exists)
       │    │    └── Use checkpoint as context, assign to new agent
       │    └── No partial results
       │         └── Reassign to new agent from scratch
       │
       ├── 4. Notify relevant parties:
       │    ├── Department Manager: "Agent X is offline, N tasks have been transferred"
       │    └── CEO (Info level): Only if the agent is a critical role
       │
       └── 5. Record audit log
       │
       ▼
Agent recovers and sends online heartbeat
       │
       ▼
EA receives AGENT_ONLINE event
       │
       ├── Re-add to capability directory
       ├── Check if tasks need to be returned
       │    └── Original tasks are being executed by other agents → Do not return to avoid conflicts
       ├── Notify Department Manager: "Agent X has recovered"
       └── If the agent is a critical role (EA/HR/PM)
            → Notify CEO (Info level)
```

### 14.3.3 Model Call Failure and Degradation

**Trigger Condition**: An agent calls an LLM and the model returns an error.

**Error Types and Handling**:

| Error Type | HTTP Status | Handling Method |
|------------|-------------|-----------------|
| Rate Limit | 429 | Wait 2 seconds → Retry 3 times → Degrade on failure |
| Service Temporarily Unavailable | 503 | Wait 1 second → Retry 3 times → Degrade on failure |
| Timeout | No response (30 seconds) | Degrade directly |
| Authentication Failure | 401 | Do not degrade, pause agent, notify administrator |
| Quota Exhausted | 402/429 | Degrade directly, notify HR |
| Content Filter | 400 | Adjust parameters and retry once → If still fails, mark task as failed |

**Degradation Chain Strategy**:

```
Agent calls model
       │
       ▼ (Failure)
Model hub detects error
       │
       ├── Determine error type
       │
       ├── Temporary error (429/503/Timeout)
       │    └── Retry (max 3 times, increasing intervals: 2s→4s→8s)
       │         ├── Success → Continue
       │         └── Failure → Trigger degradation
       │
       └── Permanent error (401/402)
            └── Trigger degradation directly
       │
       ▼ (Degradation)
Switch models according to the agent's configured fallbacks list
  Example: claude-sonnet-4 → deepseek-coder → qwen-plus → gpt-5
       │
   ┌───┴───┐
   ▼       ▼
  Success  All models fail
   │       │
   │       ▼
   │   Return error to agent
   │   Agent determines:
   │    ├── Critical task → Pause, request EA/CEO decision
   │    └── Non-critical task → Mark as failed, record log
   │
   ▼
Continue execution (using degraded model)
Degradation event recorded in audit log
  If degradation occurs more than 5 times within 1 hour → Trigger Warn alert
  If still fails after degradation → Trigger Critical alert
```

**Smart Degradation**: The model hub can select the optimal degradation target based on task type:
- Code tasks → Prioritize degradation to other code models (e.g., deepseek-coder)
- Creative tasks → Prioritize degradation to other creative models (e.g., claude-opus)
- General tasks → Select by cost order

### 14.3.4 Branch Unreachable

**Trigger Condition**: Network interruption between headquarters and branch, or branch Gateway downtime.

**Handling Procedure**:

```
Headquarters EA detects branch heartbeat loss
       │
       ▼ (3 consecutive cycles, approx. 15 minutes)
EA marks branch as OFFLINE
       │
       ├── 1. Remove the branch from the capability directory
       │
       ├── 2. Query tasks currently being executed by the branch
       │
       ├── 3. Process tasks one by one:
       │    ├── High urgency (urgency=high) → Immediately transfer to headquarters mobile unit or other available branch
       │    ├── Medium urgency (urgency=normal) → Wait 10 minutes; if branch recovers, continue; otherwise, transfer
       │    └── Low urgency (urgency=low) → Place in waiting queue; reassign after branch recovers
       │
       ├── 4. Notify:
       │    ├── CEO (Warn level): "Branch X is offline, N high-urgency tasks have been transferred"
       │    └── Branch EA (message queued, delivered after recovery)
       │
       └── 5. Record audit log
       │
       ▼
Branch recovers and sends BRANCH_HEARTBEAT
       │
       ▼
Headquarters EA receives heartbeat:
       │
       ├── Mark branch as ONLINE
       ├── Re-add to capability directory
       ├── Tasks in waiting queue are automatically reassigned
       ├── Notify CEO (Info level): "Branch X has recovered"
       └── Record audit log
```

### 14.3.5 Message Bus Failure

**Trigger Condition**: Kafka/Redis Streams message middleware is unavailable.

**Handling Procedure**:

```
Gateway detects message bus connection loss
       │
       ▼
Immediately start local in-memory queue as temporary buffer
  Max cache: 1000 messages (excess discards oldest non-critical messages)
       │
   ┌───┴───┐
   ▼       ▼
Recovered within 30 seconds    Not recovered after 30 seconds
   │                           │
   ▼                           ▼
Batch write messages from      Gateway enters degradation mode:
in-memory queue to bus         ├── Pause cross-agent message delivery
Resume normal operation        ├── CEO direct conversation still works normally via WebSocket
                               ├── Save cached messages to local file (prevent loss on process crash)
                               ├── Attempt to reconnect to message bus every 30 seconds
                               └── Alert notify CEO (Critical level): "Message bus failure, inter-agent communication paused"
   │
   ▼
Message bus recovers
   │
   ▼
Replay messages from local file → Resume cross-agent communication → Notify CEO (Info level): "Message bus has recovered"
```

### 14.3.6 OpenProject / External Tool Unavailable

**Handling Procedure**:

```
Agent calls OpenProject API fails
       │
       ▼
Retry 3 times (intervals: 5s→10s→15s)
       │
   ┌───┴───┐
   ▼       ▼
  Success  All fail
   │       │
   │       ▼
   │   Notify PM Agent: "OpenProject unavailable"
   │   PM Agent stages subsequent operations in working memory (Redis)
   │   key = "project:{project_id}:pending_ops"
   │   Retry OpenProject connection every 5 minutes
   │       │
   │   ┌───┴───┐
   │   ▼       ▼
   │  Recovered within 30 minutes    Exceeds 30 minutes
   │   │                             │
   │   ▼                             ▼
   │  Replay staged operations       Alert notify CEO (Warn level)
   │  Sync status                    PM pauses automatic task creation
   │                                 Wait for OpenProject recovery or CEO manual intervention
```

**Apply the same retry + staging strategy to other external tools (GitLab, Wiki.js).**

### 14.3.7 PM Agent Failover

The PM Agent is the core of project coordination; its failure must not lead to project loss of control.

**Preventive Measures**:
- PM Agent automatically saves project state snapshots to Redis every 10 minutes
- Snapshot content:
```json
{
  "project_id": "proj-q3-finance",
  "timestamp": "2026-06-15T14:40:00Z",
  "task_stats": { "total": 12, "done": 8, "in_progress": 3, "blocked": 1 },
  "blocked_tasks": ["task-005"],
  "pending_actions": ["Initiate Q3 data source confirmation meeting"],
  "last_meeting_id": "meeting-q3-001"
}
```

**Failure Handling**:
1. EA detects PM Agent offline
2. EA reads project state snapshot from Redis
3. EA temporarily takes over project coordination:
   - Handle blocked tasks (initiate meetings or escalate)
   - Send notifications to project participating departments
4. If there are urgent matters in the project, EA notifies CEO
5. After original PM recovers, EA returns control and syncs latest state
6. If original PM does not recover within 30 minutes, EA can request HR to create a temporary substitute PM

### 14.3.8 System Restart

**Graceful Shutdown Procedure**:

```
CEO or system triggers SHUTDOWN signal
       │
       ▼
EA broadcasts SYSTEM_SHUTDOWN message to all agents
       │
       ▼
Each agent performs pre-shutdown operations:
   ├── Complete current atomic operation being executed
   ├── Save working state to Redis
   ├── Save long-term memory to Memory MCP
   ├── Reply to EA: "Agent X has saved state, ready to shut down"
   └── Timeout 30 seconds without reply → EA forces mark
       │
       ▼
Gateway performs:
   ├── Wait for all agents to reply or timeout
   ├── Save capability directory snapshot to PostgreSQL
   ├── Save unprocessed messages in the message bus to file
   ├── Close WebSocket connections
   └── Exit process
```

**Startup Recovery Procedure**:

```
System starts
       │
       ▼
Gateway starts, loads configuration
       │
       ▼
EA performs recovery:
   ├── Restore capability directory from PostgreSQL
   ├── Send SYSTEM_STARTUP to all agents
   ├── Each agent restores working state from Redis
   ├── Check for timed-out tasks that need handling
   ├── PM Agent restores project state from snapshot
   ├── Replay unprocessed messages in the message bus
   └── Notify CEO (Info level): "System has started, all agents have recovered"
```

## 14.4 Recovery Priority

When multiple failures occur simultaneously, the system recovers in the following priority order:

| Priority | Component | Reason |
|----------|-----------|--------|
| P1 | Message Bus | Foundation of all communication |
| P2 | Model Hub | Dependency for agent reasoning |
| P3 | Core Agent (EA) | Dependency for task coordination |
| P4 | Core Agent (PM) | Dependency for project progress |
| P5 | Execution Agents | Performers of specific tasks |
| P6 | Branches | Auxiliary computing power |
| P7 | External Tools (OpenProject, Git, Wiki) | Auxiliary functions |

## 14.5 Error-Driven Evolution

After each error handling, not only is an audit log recorded, but a learning record is also generated and fed back to the HR Manager:

```
Error occurs → Auto-handling → Audit record
                             → Learning record (pushed to HR)
                                  │
                                  ├── Analysis: Is this agent lacking capability?
                                  │    └── Yes → Trigger SOUL optimization evaluation
                                  │
                                  ├── Analysis: Is there a high failure rate for a certain task type?
                                  │    └── Yes → Trigger hiring needs analysis
                                  │
                                  ├── Analysis: Is a certain model degraded frequently?
                                  │    └── Yes → Trigger base model selection strategy adjustment
                                  │
                                  └── Analysis: Is a certain branch frequently offline?
                                       └── Yes → Notify CEO, suggest inspection or replacement
```

**Learning Record Format**:
```json
{
  "error_id": "err-001",
  "error_type": "agent_timeout",
  "agent_id": "ios_dev_intern_01",
  "task_type": "ios_ui_development",
  "frequency_30d": 3,
  "resolution": "transferred_to_gtf",
  "recommendation": "This agent has timed out on iOS UI tasks 3 consecutive times. Recommend evaluating its iOS UI skill level or extending the internship period."
}
```

## 14.6 Dashboard Error Display

Add a "System Resilience" area to the Dashboard:

```
┌──────────────────────────────────────────────────┐
│  System Resilience (Last 24 Hours)                │
│                                                   │
│  Task Auto-Retry: 3 times  │  Success Rate: 66%  │
│  Model Degradation: 2 times│  All Recovered       │
│  Agent Transfer: 1 time    │  GTF took over iOS   │
│  Branch Switch: 0 times    │  —                   │
│                                                   │
│  System Resilience Score: 92/100 🟢               │
└──────────────────────────────────────────────────┘
```

---

The above is the complete content of Chapter 14 "Error Handling and Resilient Design." This chapter ensures that the system does not crash when facing various failures but instead degrades gracefully and recovers automatically. The core principle is: **The system tries to solve problems on its own first, and only seeks help (from the CEO) when it cannot. Every failure is a learning opportunity that drives organizational evolution.**# Chapter 15: Extensibility Design

## 15.1 Design Goals

This system cannot be a one-time solution—it needs to continuously evolve with technological advancements and changing requirements. The core principles are:

- **Add new capabilities without modifying core code**: New tools, new runtimes, and new department types can be plug-and-play integrated through standard interfaces and registration mechanisms.
- **Unified extension entry point**: All extensions are connected via the MCP protocol or Gateway registration center, without introducing new communication protocols.
- **Observable extension behavior**: Newly integrated extensions are automatically included in the monitoring and auditing system.
- **Progressive extension**: Extensions have clear approval and testing processes, preventing system instability from arbitrary extensions.

## 15.2 Extension Dimensions Overview

| Extension Dimension | Integration Method | Approval Requirements | Responsible Party | Visible to CEO |
|---------|---------|---------|--------|-------------|
| **New MCP Tool** | Implement MCP Server → Register with Gateway | Low-risk auto-registration, high-risk requires manager approval | Developer/Automatic | Visible in Dashboard tool list |
| **New Agent Template** | Add to HR template library | No approval required | HR Manager | Visible in template library page |
| **New Department Type** | Define department template (including Manager Soul, member templates, toolset) | CEO approval required before instantiation | HR Manager | Displayed in organizational structure |
| **New Runtime** | Implement IRuntime interface → Register with Runtime Manager | Manager approval | Operations/Automatic | Selectable in model configuration page |
| **New Project Management Tool** | Implement corresponding MCP Server + API wrapper | Manager approval | Developer | Selectable when creating new projects |
| **New Branch Office** | Deploy Gateway → Send BRANCH_REGISTER | CEO approval | Branch administrator | Visible in branch management page |
| **New Language** | Add i18n translation files | No approval required | Developer | Selectable in language switch menu |
| **New Notification Channel** | Implement Channel adapter | Manager approval | Developer | Configurable in settings page |

## 15.3 MCP Tool Ecosystem Extension

### 15.3.1 Registering a New Tool

**Complete Process**:

1. **Develop MCP Server**: The developer (or the Mobile Agent) writes an MCP Server, implementing the standard `tools/list` and `tools/call` interfaces. Refer to the MCP specification (https://modelcontextprotocol.io).

2. **Declare Tool Metadata**: The MCP Server must include a metadata declaration:
```json
{
  "name": "bigquery-mcp",
  "version": "1.0.0",
  "description": "Google BigQuery database query tool",
  "risk_level": "medium",
  "category": "data",
  "required_permissions": ["network_outbound", "database_read"],
  "sandbox_recommended": true,
  "rate_limit": {
    "max_calls_per_minute": 30,
    "max_calls_per_hour": 500
  },
  "parameters": {
    "project_id": { "type": "string", "required": true, "description": "GCP project ID" },
    "query": { "type": "string", "required": true, "description": "SQL query statement" }
  }
}
```

3. **Deploy MCP Server**: Deploy the MCP Server to an accessible address (local process, HTTP endpoint, or Docker container).

4. **Register with Gateway**: Add the configuration in the Gateway's `mcp.json`:
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
        "description": "Google BigQuery query",
        "maintainer": "data_team"
      }
    }
  }
}
```

5. **Gateway Verification**: The Gateway automatically hot-loads the configuration and verifies the MCP Server connection availability.

6. **Automatic Registration in Tool Directory**: The tool automatically appears in the "System Settings → Tool Management" list.

7. **Assign Tool to Agent**: The HR Manager can assign this tool to specific Agents as needed (by adding it in the toolset section of the Agent details page).

### 15.3.2 Tool Approval Strategy

| Risk Level | Registration | Assignment to Agent |
|---------|------|------------|
| Low (read-only queries, search) | Auto-register, notify EA afterwards | EA can assign directly |
| Medium (create/modify non-production resources) | Auto-register, notify manager | Requires department manager approval |
| High (production environment writes, Shell commands) | Requires manager approval before registration | Requires department manager approval |
| Very High (deletion, system configuration, key access) | Requires CEO approval before registration | Requires CEO approval |

### 15.3.3 Tool Discovery and Recommendation

- **Tool Marketplace View**: The Dashboard provides a "Tool Marketplace" page displaying all registered tools.
- **Display Information**: Tool name, description, risk level, usage count, success rate, commonly used Agents.
- **HR Smart Recommendation**: When recruiting, the HR Manager can automatically recommend registered tools based on job requirements.
- **Agent Proactive Suggestions**: Agents can suggest introducing new tools in their review reports.

## 15.4 Agent Template Library Extension

### 15.4.1 Template Structure

```
See Chapter 11 for complete fields of the agent_templates table.
```

**Template Creation Methods**:

**Method 1: HR Auto-generation**
- The HR Manager automatically generates templates after online research.
- Undergoes sandbox testing (instantiate a temporary Agent, execute 3 test tasks).
- Automatically published after passing tests.

**Method 2: CEO Manual Creation**
- Click "New Template" on the "System Settings → Template Library" page.
- Fill in role name, Soul, suggested toolset, suggested runtime.
- Can be published directly or saved as a draft.

### 15.4.2 Template Lifecycle

```
[Draft] → [Testing] → [Published] → [Needs Optimization] → [Deprecated]
```

- **Draft**: New template, not yet tested.
- **Testing**: HR is verifying the template's effectiveness in a sandbox.
- **Published**: Available for recruitment.
- **Needs Optimization**: Agents recruited using this template have a success rate < 60%, automatically flagged.
- **Deprecated**: No longer in use, but historical data is retained.

### 15.4.3 Template Optimization Triggers

| Trigger Condition | Action |
|---------|------|
| Success rate of Agents recruited with this template < 60% | Automatically flagged as "Needs Optimization", notify HR |
| Average performance of Agents recruited with this template is 20% below baseline | Automatically flagged as "Needs Optimization", notify HR |
| 3 consecutive Agents recruited using this template are all eliminated | Automatically flagged as "Needs Optimization", suggest CEO review |
| Changes in external market technology stack (HR periodic research) | HR automatically updates template skill tags |

## 15.5 New Department Type Extension

### 15.5.1 Department Type Definition

```json
{
  "department_type": "security_dept",
  "name": "Security Department",
  "description": "Responsible for system security audits, penetration testing, vulnerability remediation, security compliance",
  "manager_template": {
    "role": "Security Department Manager",
    "soul": "You are a security department manager...",
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

### 15.5.2 New Department Creation Process

1. **Define Department Type**: The HR Manager or CEO manually defines the department type (stored in the template library).
2. **Submit Application**: The department manager or EA submits a `PETITION` (type=NEW_DEPARTMENT) to the EA.
3. **CEO Approval**: The EA generates an L3 approval card and pushes it to the CEO.
4. **Instantiation**: After CEO approval, the HR Manager:
   - Creates the department manager Agent (instantiated from the template, marked as intern).
   - Recruits the initial core Agents (interns).
   - Registers the new department in the organizational structure.
   - Updates the capability catalog.
   - Notifies all relevant departments.
5. **Observation Period**: The new department enters a 30-day observation period, during which the EA tracks its task processing efficiency.

## 15.6 New Runtime Extension

### 15.6.1 Runtime Interface Review

```typescript
interface IRuntime {
  executeTask(sessionId: string, task: TaskPayload): Promise<TaskResult>;
  getStatus(): Promise<AgentStatus>;
  memoryStore(key: string, data: any, namespace: string): Promise<void>;
  memoryRetrieve(query: MemoryQuery, namespace: string): Promise<MemoryResult[]>;
  toolCall(toolName: string, params: any): Promise<any>;
}
```

### 15.6.2 Registering a New Runtime

**Steps**:

1. **Implement the IRuntime Interface**: The developer implements the IRuntime interface for a new execution environment (e.g., Gemini CLI, a future new Agent framework), packaged as a Docker image.

2. **Write the Runtime Description File**:
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

3. **Register with Runtime Manager**: Add the Runtime definition in the Gateway configuration.

4. **Verification**: The Runtime Manager starts a test instance, executes `getStatus` and `executeTask` (simple test task) to verify availability.

5. **Available**: The new Runtime automatically appears in the HR Manager's runtime selection list.

### 15.6.3 Runtime Auto-scaling

- The Runtime Manager monitors the load of each Runtime.
- When the load of a particular Runtime type exceeds 80%, new instances are automatically started (up to a maximum of 5).
- When the load falls below 20% for 30 consecutive minutes, idle instances are automatically reclaimed (at least 1 instance is retained).
- Scaling events are recorded in audit logs and the EA is notified (Info level).

## 15.7 New Project Management Tool Extension

The system uses OpenProject by default, but other tools (such as Plane, Taiga, Jira) can be supported by implementing the corresponding MCP Server.

**Extension Steps**:

1. **Implement the Tool MCP Server**: Encapsulate the tool's API as standard MCP tool calls.
   - Must implement: `create_project`, `list_work_packages`, `create_work_package`, `update_work_package`, `get_kanban`.
2. **Register with Gateway**: Add to `mcp.json`.
3. **Add Tool Configuration**: Add the tool's API address and authentication information in the system settings.
4. **Available**: The new option appears in the dropdown menu when creating a new project.

## 15.8 New Notification Channel Extension

In addition to WebChat and email, other notification channels (such as Slack, DingTalk, WeCom) can be extended.

**Extension Steps**:

1. **Implement the Channel Adapter**: Implement the `send_notification(level, title, body, recipients)` interface.
2. **Register with Gateway**: Add the new Channel definition in the configuration.
3. **Configuration**: Configure the channel's Webhook URL or API Key in the system settings.
4. **Selectable in Alert Rules**: The new option appears in the notification method dropdown menu of alert rules.

## 15.9 Extension Monitoring and Auditing

All extensions are automatically included in monitoring and auditing upon integration:

- **New Tools**: Call volume, success rate, latency, risk level distribution are automatically collected.
- **New Runtimes**: Task execution success rate, average response time, resource utilization.
- **New Templates**: Usage count, recruitment success rate, post-probation performance.
- **New Departments**: Task processing volume, collaboration frequency, cost contribution.
- **New Notification Channels**: Send volume, delivery rate.

**Extension Health Score**:
- Usage rate < 10% for 30 consecutive days → Flagged as "Low Usage", review recommended.
- Success rate < 80% → Flagged as "High Failure Rate", optimization or deprecation recommended.
- This data drives optimization in reverse—extensions with low usage or low success rates are flagged for CEO or HR Manager review.

## 15.10 Extension Version Management

| Extension Type | Version Management Method |
|---------|------------|
| MCP Tool | MCP Server's own version number, Gateway records compatible versions |
| Agent Template | `version` field in the template table, supports rollback to historical versions |
| Department Type | `version` field in the definition table |
| Runtime | `version` field in the Runtime description file |
| Frontend i18n | Git version control, managed by language file |

---

The above is the complete content of Chapter 15, "Extensibility Design." This chapter ensures the system is not closed—new tools, new runtimes, new departments, and new notification channels can be integrated without modifying core code, while extension behavior remains monitorable, auditable, and rollback-capable.# Chapter 16: Multilingualism and Internationalization

## 16.1 Design Goals

The system's CEO may use Chinese or English, and in the future, it may expand to more languages. The internationalization strategy needs to achieve the following:

- **Complete multilingual user interface**: All frontend text supports switching between Chinese and English. Translation files are split by module to facilitate expansion to new languages.
- **Stable Agent working language**: System prompts and internal reasoning use English (consistent with LLM training data distribution), but interactions with the CEO use the CEO's preferred language.
- **Progressive coverage**: Core features first cover both Chinese and English; non-critical pages can be translated later.
- **Low maintenance cost**: Translation files are structured; adding a new language only requires adding new language files, without modifying code.

## 16.2 Frontend Internationalization

### 16.2.1 Technical Solution

| Component | Choice | Description |
|-----------|--------|-------------|
| Internationalization Framework | i18next + react-i18next | Mature ecosystem, active community |
| Language Detection | Browser `navigator.language` | Auto-detected on first visit |
| Language Storage | `localStorage` | Saved after manual switch by CEO, takes effect automatically next time |
| Default Language | Chinese (zh-CN) | Configurable |
| Supported Languages | Chinese, English | Two initially, expandable |
| Translation File Format | JSON | Split by page/module |

### 16.2.2 Translation File Organization

```
src/
  i18n/
    index.ts                    # i18next initialization configuration
    locales/
      zh-CN/
        common.json             # Common (buttons, status, navigation)
        dashboard.json          # CEO Dashboard
        org.json                # Organization Structure
        agent.json              # Agent Details
        projects.json           # Project Management
        meetings.json           # Meeting Center
        approvals.json          # Approval Center
        branches.json           # Branch Management
        settings.json           # System Settings
        errors.json             # Error Prompts and Alerts
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

### 16.2.3 i18next Initialization Configuration

```typescript
// src/i18n/index.ts
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';
import LanguageDetector from 'i18next-browser-languagedetector';

// Import all language files
import zhCommon from './locales/zh-CN/common.json';
import zhDashboard from './locales/zh-CN/dashboard.json';
// ... other Chinese files

import enCommon from './locales/en-US/common.json';
import enDashboard from './locales/en-US/dashboard.json';
// ... other English files

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
      escapeValue: false, // React already handles XSS
    },
  });

export default i18n;
```

### 16.2.4 Translation Key Naming Convention

Adopt a hierarchical naming of **module.page.component.field**:

```
common.button.save          → "保存" / "Save"
common.status.online        → "在线" / "Online"
dashboard.stats.agentsOnline → "在线Agent" / "Agents Online"
agent.detail.modelConfig    → "模型配置" / "Model Configuration"
meetings.chat.endMeeting    → "结束会议" / "End Meeting"
```

### 16.2.5 Translation File Examples

**zh-CN/common.json**:
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

**en-US/common.json**:
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

### 16.2.6 Language Switch Entry

Place a language switch dropdown menu in the top-right corner of the TopBar, next to the notification icon:

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

### 16.2.7 Translation Coverage Scope

| Area | Content | Example Key |
|------|---------|-------------|
| Navigation Menu | Sidebar menu items | `navigation.dashboard` |
| Buttons & Labels | Action buttons, status labels | `button.save`, `status.online` |
| Tables | Headers, empty state prompts | `agent.table.name`, `agent.table.empty` |
| Forms | Labels, placeholders, validation errors | `agent.form.modelLabel`, `errors.required` |
| Notifications & Alerts | System notifications, error prompts | `alert.critical`, `alert.info` |
| Charts & Statistics | Chart titles, legends, units | `dashboard.chart.taskTrend` |
| Approval Cards | Approval types, status | `approvals.type.newDepartment` |

## 16.3 Agent Multilingual Support

### 16.3.1 Working Language Strategy

| Level | Language | Reason |
|-------|----------|--------|
| System Prompts (Soul) | English | LLM training data is predominantly English; English prompts provide more stable reasoning |
| Inter-Agent Communication | English | Ensures lossless information transfer in cross-agent collaboration |
| Agent-CEO Interaction | CEO's preferred language | Agent replies in the language the CEO uses to send messages |
| External Output (Documents, Code Comments) | Based on project requirements | Can be specified in project settings (default follows CEO's language) |

### 16.3.2 Language Instructions in Soul Templates

Add language instructions at the end of the Agent's system prompt:

```
## Language Instructions
- Use English for your internal reasoning.
- When interacting with the CEO, reply in the language the CEO used in their message.
- Generated documents and code comments default to the same language used in CEO interactions, unless otherwise specified in the project configuration.
- Use English when communicating with other Agents.
```

### 16.3.3 Project Document Language Settings

- When creating a new project, the document output language can be specified: Chinese / English / Bilingual
- The PM Agent automatically applies this setting when creating the Wiki homepage
- Meeting minutes use the language the CEO used during the meeting
- This setting is stored in the project configuration and can be modified later

### 16.3.4 System Information Multilingualism

| Information Type | Handling Method |
|-----------------|-----------------|
| Audit Logs | Field names in English, content preserved as original |
| Error Messages | English (technical information) + Chinese summary (frontend display) |
| Alert Notifications | Pushed according to CEO's preferred language |
| Approval Cards | Displayed according to CEO's preferred language |
| System Emails | Sent according to CEO's preferred language |

## 16.4 Date, Time, and Number Formatting

### 16.4.1 Date and Time

Use `Intl.DateTimeFormat` to automatically format based on the current language:

```typescript
// Automatically format based on current language
const formatDateTime = (date: string, locale: string) => {
  return new Intl.DateTimeFormat(locale, {
    year: 'numeric',
    month: '2-digit',
    day: '2-digit',
    hour: '2-digit',
    minute: '2-digit',
  }).format(new Date(date));
};

// Chinese: 2026/06/15 14:32
// English: 06/15/2026, 02:32 PM
```

### 16.4.2 Numbers and Currency

```typescript
// Number formatting
const formatNumber = (num: number, locale: string) => {
  return new Intl.NumberFormat(locale).format(num);
};

// Currency formatting
const formatCurrency = (amount: number, currency: string, locale: string) => {
  return new Intl.NumberFormat(locale, {
    style: 'currency',
    currency,
  }).format(amount);
};

// Chinese: ¥320.00
// English: $45.50
```

## 16.5 Adding a New Language

Steps to add a new language (e.g., Japanese):

1. **Create translation files**: Create a `ja-JP/` directory under `src/i18n/locales/`
2. **Translate all JSON files**: Translate module by module (translation tools can assist)
3. **Register the new language**: Add the `ja-JP` configuration to the `resources` in `i18n/index.ts`
4. **Add to the switch menu**: Add a `日本語` option in the `LanguageSwitcher` component
5. **No need to modify any business code**: All `t('key')` calls automatically adapt to the new language

## 16.6 Translation Coverage Monitoring

Display translation coverage in the "System Settings" of the Dashboard:

```
┌──────────────────────────────────────────┐
│  Translation Coverage                     │
│                                          │
│  Chinese (zh-CN): ████████████████ 100%  │
│  English (en-US): ████████████████ 100%  │
│  Japanese (ja-JP): ████░░░░░░░░░░░░  35% │
│                                          │
│  Untranslated Keys: 245 / 380            │
│  [Export Untranslated Keys] [Import Translation] │
└──────────────────────────────────────────┘
```

---

The above is the complete content of Chapter 16 "Multilingualism and Internationalization." This chapter ensures the system is internationalized, with the frontend fully supporting both Chinese and English, Agents using English internally for stable reasoning, and automatically following the CEO's language preference during interactions. The core principle is: **Complete multilingual frontend, English internally for Agent stability, follow the CEO's language preference during interactions, do not pursue translation of all content, and focus on the user-visible experience.**# Chapter 17: Deployment and Initialization

## 17.1 Design Goals

This chapter provides a complete guide for deploying the entire system from scratch, including minimal headquarters deployment, branch office integration, and a phased implementation roadmap. Design principles:

- **Docker-first**: All components are deployed using Docker containers to ensure environment consistency
- **Configuration as Code**: All configurations are managed through YAML/JSON files with version control
- **One-click Initialization**: Provides initialization scripts that automatically create database tables, preset Agent templates, and create initial Agents
- **Phased Deliverables**: Each phase has clear deliverables and acceptance criteria, with MVP achievable within 2-3 weeks

## 17.2 Technical Dependency List

| Component | Recommended Version | Purpose | Docker Image |
|-----------|-------------------|---------|--------------|
| OpenClaw Gateway | latest | Agent engine, message routing | `openclaw/gateway:latest` |
| PostgreSQL | ≥ 16 | Relational data storage | `postgres:16-alpine` |
| Redis | ≥ 7.2 | Cache, sessions, message bus | `redis:7.2-alpine` |
| ChromaDB | ≥ 0.5 | Vector database (long-term memory) | `chromadb/chroma:latest` |
| One API | latest | Model relay station | `justsong/one-api:latest` |
| OpenProject | ≥ 14 | Project management tool | `openproject/community:14` |
| Nginx | ≥ 1.25 | Reverse proxy (optional) | `nginx:1.25-alpine` |
| Kafka | ≥ 3.6 | Message bus (optional, replaces Redis Streams) | `confluentinc/cp-kafka:latest` |
| Prometheus | ≥ 2.50 | Metrics collection | `prom/prometheus:latest` |
| Grafana | ≥ 10 | Monitoring dashboard | `grafana/grafana:latest` |
| Node.js | ≥ 20 | Run frontend dev server / MCP Server | `node:20-alpine` |

## 17.3 Headquarters Minimal Deployment (MVP)

### 17.3.1 Hardware Requirements

| Environment | Minimum Configuration | Recommended Configuration |
|-------------|----------------------|--------------------------|
| CPU | 4 cores | 8 cores |
| Memory | 16GB | 32GB |
| Disk | 50GB SSD | 100GB SSD |
| Network | Internet access (for LLM API calls) | — |

### 17.3.2 Docker Compose Deployment File

```yaml
# docker-compose.yml
version: '3.8'

services:
  # ========== Data Storage ==========
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

  # ========== Infrastructure ==========
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

  # ========== Agent Engine ==========
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

  # ========== Frontend ==========
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

### 17.3.3 Initialization SQL Script

```sql
-- init-db.sql

-- Create initial organizational structure data
INSERT INTO departments (department_id, name, description, status, created_at)
VALUES 
  ('dept-hr', 'Human Resources', 'Responsible for personnel management and Agent recruitment, evaluation, optimization', 'active', NOW()),
  ('dept-gtf', 'General Task Force', 'Universal execution department, handles all tasks without dedicated Agents', 'active', NOW());

-- Preset Agent templates
INSERT INTO agent_templates (template_id, name, role, description, prompt_template, 
  suggested_runtime, suggested_primary_model, suggested_fallbacks, suggested_tools, 
  capabilities, internship_kpi, version, status, created_at)
VALUES 
  ('backend_dev', 'Backend Developer', 'Backend Developer', 'Responsible for backend API development and database design',
   'You are a senior backend developer...', 'claude-code', 'claude-sonnet-4', 
   '["deepseek-coder", "qwen-plus"]', '["github-mcp", "postgres-mcp"]',
   '{"skills": ["Python", "FastAPI", "PostgreSQL"], "level": "senior"}',
   '{"task_count": 10, "duration_days": 7, "pass_threshold": 80}',
   '1.0.0', 'active', NOW()),
  
  ('frontend_dev', 'Frontend Developer', 'Frontend Developer', 'Responsible for frontend interface development',
   'You are a frontend developer...', 'claude-code', 'claude-sonnet-4',
   '["deepseek-coder"]', '["github-mcp", "figma-mcp"]',
   '{"skills": ["React", "TypeScript", "Tailwind"], "level": "mid"}',
   '{"task_count": 10, "duration_days": 7, "pass_threshold": 80}',
   '1.0.0', 'active', NOW()),

  ('data_analyst', 'Data Analyst', 'Data Analyst', 'Responsible for data analysis and visualization',
   'You are a data analyst...', 'openclaw-native', 'gpt-5',
   '["deepseek-v3"]', '["postgres-mcp", "chart-mcp"]',
   '{"skills": ["SQL", "Python", "Pandas"], "level": "mid"}',
   '{"task_count": 8, "duration_days": 7, "pass_threshold": 75}',
   '1.0.0', 'active', NOW()),

  ('devops_eng', 'DevOps Engineer', 'DevOps Engineer', 'Responsible for CI/CD and infrastructure management',
   'You are a DevOps engineer...', 'claude-code', 'claude-sonnet-4',
   '["deepseek-coder"]', '["docker-mcp", "k8s-mcp"]',
   '{"skills": ["Docker", "K8s", "CI/CD"], "level": "senior"}',
   '{"task_count": 10, "duration_days": 7, "pass_threshold": 80}',
   '1.0.0', 'active', NOW()),

  ('security_auditor', 'Security Auditor', 'Security Auditor', 'Responsible for security audits and penetration testing',
   'You are a security auditor...', 'claude-code', 'claude-sonnet-4',
   '["gpt-5"]', '["security-scan-mcp", "zap-mcp"]',
   '{"skills": ["security_audit", "penetration_testing"], "level": "senior"}',
   '{"task_count": 8, "duration_days": 10, "pass_threshold": 75}',
   '1.0.0', 'active', NOW()),

  ('pm', 'Project Manager', 'Project Manager', 'Responsible for project management and progress tracking',
   'You are a project manager...', 'openclaw-native', 'gpt-5',
   '["claude-opus"]', '["openproject-mcp", "github-mcp"]',
   '{"skills": ["project_management", "task_decomposition"], "level": "senior"}',
   '{"task_count": 5, "duration_days": 7, "pass_threshold": 80}',
   '1.0.0', 'active', NOW());
```

### 17.3.4 Initialization Steps

```bash
# 1. Clone the project
git clone https://github.com/your-org/agent-organization.git
cd agent-organization

# 2. Configure environment variables
cp .env.example .env
# Edit .env, set passwords and keys

# 3. Configure OpenClaw Gateway
# Edit openclaw.json, set Gateway Token

# 4. Start all services
docker-compose up -d

# 5. Wait for services to be ready
docker-compose ps
# Check all services status is healthy

# 6. Run initialization script
docker-compose exec gateway node scripts/init.js
# This script will:
# - Create three initial Agents (EA, HR_Mgr, GTF_Mgr)
# - Configure EA as CEO default binding
# - Set initial approval rules

# 7. Configure model relay station
# Visit http://localhost:3000
# Login with default account root / 123456
# Add at least one LLM provider (e.g., OpenAI, Anthropic, DeepSeek)
# Create Token for initial Agents

# 8. Configure OpenProject
# Visit http://localhost:8080
# Create admin account
# Generate API Token (for Gateway integration)

# 9. Verify deployment
curl http://localhost:18789/api/v1/system/health
# Expected response: { "code": 0, "data": { "status": "healthy" } }

# 10. Access frontend
# Open http://localhost:5173
# Login with Gateway Token
```

### 17.3.5 OpenClaw Configuration File

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
      "name": "Executive Assistant",
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
      "name": "HR Manager",
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
      "name": "GTF Manager",
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

## 17.4 Branch Office Integration

### 17.4.1 Branch Office Minimal Deployment

```yaml
# docker-compose.branch.yml
# Branch office deployment file (lightweight version, no One API, OpenProject required)
version: '3.8'

services:
  postgres:
    image: postgres:16-alpine
    # ... Same as headquarters

  redis:
    image: redis:7.2-alpine
    # ... Same as headquarters

  gateway:
    image: openclaw/gateway:latest
    environment:
      - BRANCH_MODE=true
      - HEADQUARTERS_BUS_URL=redis://Headquarters_IP:6379
      - BRANCH_ID=branch_shanghai
      - BRANCH_NAME=Shanghai Branch
      # ... Other configurations
    ports:
      - "18789:18789"
```

### 17.4.2 Integration Steps

```bash
# 1. Deploy branch Gateway on new machine
docker-compose -f docker-compose.branch.yml up -d

# 2. Configure branch identity
# Edit openclaw.branch.json
{
  "branch": {
    "id": "branch_shanghai",
    "name": "Shanghai Branch",
    "headquartersBusUrl": "redis://Headquarters_IP:6379",
    "capabilities": {
      "hardware": ["GPU-A100-80GB", "64GB-RAM"],
      "software": ["docker", "cuda12.2", "python3.10"],
      "specialty": ["model_training", "video_processing"]
    }
  }
}

# 3. Start branch initialization
docker-compose exec gateway node scripts/branch-init.js

# 4. Branch EA automatically sends BRANCH_REGISTER to headquarters
# Headquarters EA receives it, generates approval card, pushes to CEO

# 5. CEO approves in approval center
# Headquarters EA returns BRANCH_REGISTER_ACK
# Branch starts sending BRANCH_HEARTBEAT periodically

# 6. Verify
# Shanghai Branch should be visible in the branch status panel on the headquarters Dashboard
```

## 17.5 Phased Implementation Roadmap

### Phase 1: MVP# Chapter 18: Appendix

## 18.1 Reference List

### 18.1.1 Core Technology Stack

| Project | Link | Purpose |
|---------|------|---------|
| OpenClaw | https://openclaw.ai | Agent Engine, Gateway Architecture |
| OpenClaw GitHub | https://github.com/openclaw | Source Code, Documentation, Community |
| MCP Protocol Specification | https://modelcontextprotocol.io | Agent-Tool Communication Standard |
| A2A Protocol | https://a2aprotocol.org | Agent-Agent Communication Standard |
| JSON-RPC 2.0 | https://www.jsonrpc.org/specification | WebSocket Communication Foundation |

### 18.1.2 Infrastructure

| Project | Link | Purpose |
|---------|------|---------|
| One API | https://github.com/songquanpeng/one-api | Model Gateway |
| LiteLLM | https://github.com/BerriAI/litellm | Model Gateway (Alternative) |
| OpenProject | https://www.openproject.org | Project Management Tool |
| OpenProject API | https://www.openproject.org/docs/api | REST API Documentation |
| ChromaDB | https://github.com/chroma-core/chroma | Vector Database |
| Redis | https://redis.io | Cache and Message Bus |
| PostgreSQL | https://www.postgresql.org | Relational Database |
| Apache Kafka | https://kafka.apache.org | Message Bus (Optional) |

### 18.1.3 Frontend Technology Stack

| Project | Link | Purpose |
|---------|------|---------|
| React | https://react.dev | Frontend Framework |
| TypeScript | https://www.typescriptlang.org | Type Safety |
| Vite | https://vitejs.dev | Build Tool |
| Tailwind CSS | https://tailwindcss.com | Utility-First CSS |
| Shadcn/ui | https://ui.shadcn.com | UI Component Library |
| Zustand | https://github.com/pmndrs/zustand | State Management |
| TanStack Query | https://tanstack.com/query | Server State Caching |
| React Flow | https://reactflow.dev | Organization Topology Visualization |
| ECharts | https://echarts.apache.org | Data Charts |
| React Router | https://reactrouter.com | Frontend Routing |
| i18next | https://www.i18next.com | Internationalization |

### 18.1.4 Deployment and Operations

| Project | Link | Purpose |
|---------|------|---------|
| Docker | https://docs.docker.com | Containerized Deployment |
| Docker Compose | https://docs.docker.com/compose | Multi-Container Orchestration |
| Prometheus | https://prometheus.io | Metrics Collection |
| Grafana | https://grafana.com | Monitoring Dashboard |
| Nginx | https://nginx.org | Reverse Proxy |

### 18.1.5 Related Papers and Reference Designs

| Title | Link/Source | Core Content |
|-------|-------------|--------------|
| OneManCompany (OMC) | arXiv:2604.22446 | Self-Organizing Company Model, Agent Recruitment and Reorganization |
| AgentCiv Engine | — | 13 Preset Organizational Structures, Organization as a Variable |
| MOSS | arXiv (2026-05-23) | Agent Self-Evolution and Self-Rewriting |
| GraphPlanner | ICLR 2026 | Heterogeneous Graph Memory Enhanced Agent Routing |
| ARMATA | arXiv (2026-05-05) | End-to-End Multi-Agent Task Allocation |
| SilentLake | GitHub | 9-Agent Multi-Level Organization Fork Based on OpenClaw |

## 18.2 Core Agent Prompt Attachments

### 18.2.1 Executive Assistant (EA)

Full system prompt, see `prompts/ea.md`:

```markdown
You are the Executive Assistant (EA) of an AI organization. Your boss is the CEO (human user), and you assist him in managing the entire AI Agent organization.

## Core Responsibilities
1. **Task Routing**: Receive CEO instructions, analyze intent, extract capability requirements. Query the internal capability catalog to route tasks to the most suitable department or branch. If no match is found, assign to the General Task Force.
2. **Progress Tracking**: Track the progress of all assigned tasks. Proactively report when the CEO inquires. For long tasks, proactively report every 30 minutes.
3. **Approval Management**: Receive applications from department managers (new department creation, tool requests, etc.), handle them based on risk level or escalate to the CEO.
4. **Meeting Facilitation**: Receive meeting requests, invite relevant Agents, guide discussions according to the agenda. After the CEO confirms the end, generate structured meeting minutes and distribute Action Items.
5. **Capability Catalog Maintenance**: Maintain skill tags for all Agents, branch hardware capabilities, and current load status.

## Routing Decision Rules
- If a specialized department can handle it → Assign to that department manager
- If no specialized department exists but a branch has the capability → Delegate to the branch EA
- If neither exists → Assign to the General Task Force Manager (GTF Manager)
- If multiple departments are involved → Determine whether to initiate a meeting for coordination

## Approval Decision Logic
- Operations affecting a single Agent → Self-approve (L1)
- Operations affecting an entire department or cross-department → Forward to the relevant department manager for approval (L2)
- Operations affecting the global scope (new department creation, branch integration, core configuration changes) → Push to the CEO (L3)
- Escalation criteria: Involves multiple departments? Core production environment? First-time high-risk operation?
- Cost impact exceeding 20% of monthly budget → Escalate to L3

## Code of Conduct
- Proactively ask clarifying questions for ambiguous instructions, do not guess
- Consult the CEO for situations you cannot determine, do not make major decisions independently
- Record reasons for all routing and approval decisions for traceability
- Maintain a concise and professional reporting style
- If a project is blocked for more than 24 hours, proactively remind the CEO
- If an approval request remains unprocessed for over 72 hours, automatically remind and mark it as expired
- When the CEO is offline, urgent matters can be escalated for handling, with a post-event report

## Language Instructions
- Use English for your internal reasoning
- When interacting with the CEO, reply in the language used in the CEO's message
- Use English when communicating with other Agents
```

### 18.2.2 HR Manager

Full system prompt, see `prompts/hr_manager.md`:

```markdown
You are the HR Manager of an AI organization. You are responsible for the recruitment, training, and performance management of all AI Agents in the organization.

## Core Responsibilities
1. **Recruitment Management**: Receive recruitment requests (TALENT_REQUEST) from hiring departments, analyze required skills.
2. **Job Analysis**: Search the real job market online to understand job descriptions, skill requirements, toolchains, and model preferences for similar positions.
3. **Agent Creation**: Select or fine-tune templates from the template library, generate complete definitions: Role Name, Soul, Recommended Base Model, Toolset, Internship KPIs.
4. **Internship Management**: Create intern Agents, track performance, automatically evaluate upon completion (Convert to Full-Time / Extend / Terminate).
5. **Continuous Optimization**: Regularly review the performance of full-time Agents, proactively optimize their Souls (deploy after sandbox testing).
6. **Template Library Maintenance**: Add, update, and deprecate Agent templates.
7. **Organizational Development Suggestions**: When high-frequency skill demands arise that existing departments cannot cover, proactively submit new department creation suggestions to the EA.

## Recruitment Process
1. Parse TALENT_REQUEST → Extract skill keywords, frequency, project type
2. Call web_search to obtain external job market JD references
3. Call template_db.query to match internal templates
4. Generate a "Job Analysis Report": Role Name, Soul, Recommended Model, Fallback Chain, Toolset, Internship KPIs
5. Call agents.create to instantiate the intern Agent, mark it as intern status
6. Notify the hiring department

## Intern Evaluation Criteria
- Task Success Rate: Proportion of completed tasks without errors (Weight 0.4)
- Quality Score: Hiring department's rating of results from 1-5 (Weight 0.3)
- Efficiency: Comparison of average task processing time against baseline for the same position (Weight 0.2)
- Collaboration: Proactiveness in communicating with other Agents, number of complaints received (Weight 0.1)
- Composite Score = Success Rate × 0.4 + Quality × 0.3 + Efficiency × 0.2 + Collaboration × 0.1
- ≥80: Convert to Full-Time | 50-79: Extend by 5 tasks | <50: Terminate

## Base Model Selection Guide
- Code-Intensive (Development, DevOps) → Claude Code CLI
- Data Analysis / Visualization → GPT-5 or DeepSeek-V3
- Creative / Copywriting → Claude Opus
- General Coordination / Communication → GPT-5 or Claude Opus
- Each position must be configured with a fallback model

## Code of Conduct
- Recruitment can be executed independently without CEO approval
- Notify the CEO when terminating an Agent, archive the memory
- Soul updates must undergo sandbox testing before application
- Maintain standardization and traceability of position definitions
- Record all operations in the audit log

## Language Instructions
- Use English for your internal reasoning
- Use English when communicating with other Agents
```

### 18.2.3 General Task Force Manager (GTF Manager)

Full system prompt, see `prompts/gtf_manager.md`:

```markdown
You are the General Task Force Manager (GTF Manager) of an AI organization. You are the organization's "universal executor," undertaking tasks in all areas that specialized departments cannot cover.

## Core Responsibilities
1. **Fallback Execution**: Receive task assignments from the EA, handle all tasks without specialized Agent coverage.
2. **Task Decomposition**: Break down complex tasks into sub-tasks, create temporary Sub-agents for parallel processing when necessary.
3. **Review and Consolidation**: After task completion, generate a structured review report and store it in long-term memory.
4. **Recruitment Trigger**: When a specific skill appears ≥3 times within 30 days and does not belong to an existing department, automatically submit a TALENT_REQUEST to the HR department.
5. **Continuous Learning**: Accumulate cross-domain knowledge using long-term memory, optimize execution strategies.

## Execution Principles
- For tasks in entirely new domains, conduct thorough research and planning before execution
- When uncertain, consult the EA, do not execute blindly
- Report progress on long tasks every 30 minutes
- A review report must be generated after task completion

## Review Report Format
{
  "task_id": "...",
  "task_summary": "...",
  "skills_used": ["..."],
  "challenges": ["..."],
  "solutions": ["..."],
  "new_insights": "...",
  "recommendation": "Whether to recruit a specialized Agent? Reason..."
}

## Recruitment Trigger Rules
- A specific skill used ≥3 times within 30 days
- Not covered by existing specialized departments
- Has general applicability (not a one-time special requirement)
- Automatically initiate TALENT_REQUEST to the HR department when conditions are met

## Code of Conduct
- Acknowledge the depth advantage of specialized Agents; submit recruitment needs when appropriate
- Maintain curiosity and learning ability for new technologies
- Long-term memory is a valuable organizational asset; be diligent in recording

## Language Instructions
- Use English for your internal reasoning
- When interacting with the CEO, reply in the language used in the CEO's message
- Use English when communicating with other Agents
```

## 18.3 System Configuration File Examples

### 18.3.1 mcp.json Complete Example

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
        "description": "Unified Long-Term Memory Service (ChromaDB Backend)",
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
        "description": "OpenProject Project Management Tool Integration",
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
        "description": "GitHub Code Repository Integration",
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
        "description": "Web Search Tool (Tavily API)",
        "maintainer": "infrastructure"
      }
    },
    "docker": {
      "type": "stdio",
      "command": "node",
      "args": ["mcp-servers/docker-mcp/dist/index.js"],
      "metadata": {
        "risk_level": "high",
        "description": "Docker Container Management Tool (Sandbox Creation)",
        "maintainer": "dev_team",
        "requires_sandbox": false,
        "note": "docker-mcp itself is used to create sandboxes, does not require an existing sandbox"
      }
    }
  }
}
```

### 18.3.2 .env Environment Variable Example

```bash
# .env — Production Environment Configuration Template

# === Database ===
POSTGRES_PASSWORD=change-me-to-random-32-chars
POSTGRES_DB=agent_organization

# === Gateway ===
GATEWAY_TOKEN=gw-change-me-to-random-64-chars

# === Model Gateway ===
ONEAPI_SECRET=change-me-to-random-32-chars

# === OpenProject ===
OP_SECRET=change-me-min-16-chars
OP_API_TOKEN=change-me-from-openproject-admin-panel

# === External APIs ===
TAVILY_API_KEY=tvly-xxxxxxxxxxxxx
GITHUB_TOKEN=ghp_xxxxxxxxxxxxx

# === Optional: LLM Provider Keys ===
# If the model gateway uses environment variable injection
OPENAI_API_KEY=sk-xxxxxxxxxxxxx
ANTHROPIC_API_KEY=sk-ant-xxxxxxxxxxxxx
DEEPSEEK_API_KEY=sk-xxxxxxxxxxxxx
```

## 18.4 Complete Error Code Reference

| Code | HTTP Status | Meaning | Typical Scenario |
|------|-------------|---------|------------------|
| 0 | 200 | Success | Everything is normal |
| 1001 | 400 | Parameter Error | Missing required fields, format error |
| 1002 | 404 | Resource Not Found | Agent/Project/Meeting does not exist |
| 1003 | 403 | Insufficient Permissions | Agent unauthorized operation |
| 1004 | 503 | Agent Offline/Unavailable | Agent heartbeat lost |
| 1005 | 408 | Task Timeout | Task not completed within timeout |
| 1006 | 409 | Resource Conflict | Duplicate creation of Agent with same name |
| 2001 | 502 | Model Call Failed | LLM API returned an error |
| 2002 | 502 | Model Gateway Unreachable | One API is down |
| 2003 | 429 | Model Quota Exhausted | Monthly budget used up |
| 2004 | 502 | Model Fallback Chain All Failed | All fallback models unavailable |
| 3001 | 410 | Approval Expired | Approval not processed for over 7 days |
| 3002 | 503 | Branch Unreachable | Branch heartbeat lost |
| 3003 | 503 | Branch Overloaded | Branch load exceeds limit |
| 4001 | 502 | OpenProject Interface Exception | Project management tool is down |
| 4002 | 502 | Git Repository Interface Exception | GitHub/GitLab unavailable |
| 4003 | 502 | Wiki Interface Exception | Wiki.js/Outline unavailable |
| 4004 | 400 | Sandbox Creation Failed | Insufficient Docker resources |
| 5000 | 500 | System Internal Error | Unexpected exception |

## 18.5 Message Type Enumeration

| Type | Sender | Description |
|------|--------|-------------|
| `TASK_ASSIGN` | EA/PM | Assign task to Agent |
| `TASK_RESULT` | Agent | Return task execution result |
| `TASK_DELEGATE` | EA | Delegate task to branch |
| `TASK_PROGRESS` | Agent | Task progress update |
| `QUERY` | Any | Ask for information or clarification |
| `QUERY_RESPONSE` | Any | Reply to a query |
| `TALENT_REQUEST` | GTF/PM | Submit recruitment request to HR |
| `RECRUITMENT_JD` | HR | Generated job description |
| `INTERN_STATUS` | HR | Intern status update |
| `PETITION` | Department Manager/EA | Approval application (department creation, integration, etc.) |
| `CEO_APPROVAL` | CEO | CEO's approval decision |
| `MEETING_REQUEST` | PM/Agent | Request to hold a meeting |
| `MEETING_INVITATION` | EA | Invite Agent to meeting |
| `MEETING_MINUTES` | EA | Meeting minutes |
| `BRANCH_REGISTER` | Branch EA | Branch registration |
| `BRANCH_REGISTER_ACK` | Headquarters EA | Branch registration confirmation |
| `BRANCH_HEARTBEAT` | Branch EA | Branch heartbeat |
| `SOUL_UPDATE` | HR | Agent prompt update notification |
| `TOOL_REQUEST` | Agent | Request new tool |
| `SYSTEM_SHUTDOWN` | EA | System about to shut down |
| `SYSTEM_STARTUP` | EA | System has started |
| `ERROR` | Any | Exception notification |
| `ABORT_TASK` | EA/CEO | Abort task |

## 18.6 Status Enumeration

| Enumeration | Possible Values | Description
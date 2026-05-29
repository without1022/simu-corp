# SimuCorp — Self-Evolving Multi-Agent Collaboration System

## Project Overview
SimuCorp is a "corporate-style" multi-agent collaboration system built on OpenClaw Gateway.
It simulates enterprise organizational structure with capabilities for automatic recruitment, self-evolution, multi-branch collaboration, and large project management.

**Core Philosophy**: Company-as-Code — let AI Agents divide labor, collaborate, and grow themselves like real company employees.

**Design Documentation**: `docs/多Agent协作系统设计.md` (18-chapter complete design)  
**Whitepaper**: `docs/模拟公司白皮书.md` (design philosophy & vision)  
**Agent Prompts**: `prompts/ea.md` (Executive Assistant), `prompts/hr_manager.md` (HR Manager), `prompts/gtf_manager.md` (GTF Manager)

## Tech Stack
- Engine: OpenClaw Gateway (Node.js)
- Frontend: React 18 + TypeScript + Vite + Tailwind CSS + Shadcn/ui
- Backend: Node.js (OpenClaw Gateway built-in HTTP Server + WebSocket JSON-RPC)
- Database: PostgreSQL 16, Redis 7.2, ChromaDB
- Model Gateway: One API / LiteLLM
- Project Management: OpenProject 14
- Deployment: Docker Compose

## Phased Development Plan

### Phase 1: MVP (Current Phase)
Goal: CEO can chat with EA via WebChat, EA routes tasks to GTF for execution, results returned.

1. **Environment Setup & Gateway Configuration**
   - Create docker-compose.yml (PostgreSQL + Redis + ChromaDB + One API + OpenProject + Gateway)
   - Create openclaw.json, register three initial Agents (EA, HR_Mgr, GTF_Mgr)
   - Create init-db.sql (department tables, template initialization)
   - Start Gateway, verify WebSocket connection

2. **Initial Agent Soul (System Prompt) Writing**
   - Create `prompts/ea.md`
   - Create `prompts/hr_manager.md`
   - Create `prompts/gtf_manager.md`

3. **Basic Communication Integration**
   - Implement CEO → EA → GTF task routing pipeline
   - EA correctly analyzes intent and selects target Agent

4. **Frontend MVP Pages**
   - Initialize React + Vite + Tailwind + Shadcn/ui project
   - Implement Dashboard basic layout and stats cards
   - Implement WebChat interface (WebSocket connect to EA)

### Phase 2: Core Features
1. Organizational structure visualization (React Flow topology)
2. Automatic recruitment loop (GTF retro → HR recruit → Intern Agent → promotion evaluation)
3. Approval center (department creation approval)
4. Project management integration (OpenProject)

### Phase 3: Full Features
1. Meeting center (real-time chat + minutes generation)
2. Branch office management (heartbeat sync + task delegation)
3. Full monitoring and alerting
4. SOUL auto-optimization
5. Internationalization

## Development Standards
- **All interfaces** follow the WebSocket RPC methods and REST API definitions in Chapter 10 of the design doc
- **Data models** follow Chapter 11
- **Error codes** use the specification in Chapter 18 appendix
- **Frontend component structure** references Chapter 9's component tree
- **Security policies** follow Chapter 12's hierarchical approval mechanism

## Design Document Quick Reference
`多Agent协作系统设计.md` — 18 chapters:

| Ch | Content | Key Reference |
|----|---------|---------------|
| 1 | System Overview & Design Vision | Project goals |
| 2 | System Architecture | Layered architecture & communication protocols |
| 3 | Organization Model & Core Agent Design | EA/HR/GTF souls & toolkits |
| 4 | Capability Sync & Task Routing | EA routing decision logic |
| 5 | Self-Evolution Loop | Auto-recruitment & internship evaluation |
| 6 | Runtime Abstraction & Unified Memory | Runtime interface & memory architecture |
| 7 | Large Project Management & Meetings | OpenProject integration & meeting workflow |
| 8 | Model Relay & Independent Model Config | Model configuration & fallback chain |
| 9 | Frontend Detailed Design | Page layout, component tree, state management |
| 10 | Backend API Detailed Design | WebSocket methods, REST API |
| 11 | Data Model & Storage | Database table structures |
| 12 | Security & Permission Model | Approval hierarchy, sandbox verification |
| 13 | Monitoring, Logging & Audit | Metrics, alert rules |
| 14 | Error Handling & Resilience | Timeout, offline, degradation |
| 15 | Extensibility Design | New tools, new Runtime integration |
| 16 | i18n & Internationalization | i18n solution |
| 17 | Deployment & Initialization | docker-compose.yml |
| 18 | Appendix | Message type enums, state enums, error codes |

## Current Task
Phase 1 — MVP core development completed.

## Quick Start

### 1. Start Infrastructure (optional, Gateway can run standalone)
```bash
docker compose up -d postgres redis chromadb
```

### 2. Start Gateway
```bash
cd gateway && npm run dev
```
Gateway starts at `http://localhost:18789`, WebSocket at `ws://localhost:18789`.

### 3. Start Frontend
```bash
cd frontend && npm run dev
```
Frontend starts at `http://localhost:5173`, auto-proxies API requests to Gateway.

### 4. Verify
```bash
curl http://localhost:18789/api/v1/system/health
```

## Project Structure
```
simu-corp/
├── docker-compose.yml       # PostgreSQL, Redis, ChromaDB, One API
├── init-db.sql              # Database init (7 tables + 8 templates)
├── openclaw.json            # Gateway config (3 Agents)
├── .env.example             # Environment variable template
├── prompts/                 # Agent Soul prompts
│   ├── ea.md
│   ├── hr_manager.md
│   └── gtf_manager.md
├── gateway/                 # Node.js Gateway server
│   └── src/
│       ├── index.ts         # Entry point
│       ├── server/          # HTTP + WebSocket server
│       ├── core/            # Agent registration, message routing, session mgmt
│       ├── agents/          # EA, GTF, HR Agent implementations
│       ├── transport/       # Agent communication transport layer
│       ├── api/             # REST API routes
│       ├── db/              # PostgreSQL connection pool
│       ├── redis/           # Redis client
│       ├── llm/             # One API LLM client
│       └── types/           # TypeScript type definitions
└── frontend/                # React + Vite + Tailwind frontend
    └── src/
        ├── components/      # UI components
        │   ├── layout/      # TopBar, Sidebar, MainLayout
        │   ├── dashboard/   # StatsCards, AgentStatusList, ActivityStream
        │   ├── chat/        # ChatPanel, MessageBubble, MessageInput
        │   └── shared/      # StatusBadge, WsStatusIndicator
        ├── pages/           # DashboardPage
        ├── hooks/           # useWebSocket
        ├── store/           # Zustand global state
        └── api/             # wsClient (WebSocket JSON-RPC)
```

## Implemented Phase 1 Features
- CEO chats with EA via WebChat / REST API
- EA keyword intent analysis (greeting, status query, task request)
- EA auto-routes tasks to GTF Manager
- GTF generates mock responses (code/analysis/copywriting/generic)
- Capability catalog with matching scores for 3 Agents
- WebSocket JSON-RPC + HTTP REST dual channel
- Frontend Dashboard: stats cards, Agent status list, live activity stream, WebChat
- Mock mode (default), switch to real LLM by configuring ONE_API_KEY

## Development Commands
| Command | Description |
|---------|-------------|
| `cd gateway && npm run dev` | Start Gateway (hot reload) |
| `cd gateway && npx tsc --noEmit` | Gateway type checking |
| `cd frontend && npm run dev` | Start Frontend (hot reload) |
| `cd frontend && npx tsc --noEmit` | Frontend type checking |
| `docker compose up -d` | Start all infrastructure |

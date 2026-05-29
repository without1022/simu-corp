# Document 1: Development Agent Collaboration Specification

**File**: `DEVELOPMENT.md`
**Status**: Immediately required

---

# SimuCorp — Development Agent Collaboration Specification

## 1. Overview

This document defines the collaboration rules for multiple development agents (Frontend Agent, Backend Agent, Prompt Agent, Test Agent) when concurrently developing SimuCorp. All development agents must adhere to this specification.

## 2. Development Agent Role Division

| Agent Role          | Responsible Module                                                               | Dependencies                               |
|---------------------|----------------------------------------------------------------------------------|--------------------------------------------|
| **Infrastructure Agent** | `docker-compose.yml`, `openclaw.json`, `init-db.sql`, `mcp.json`                  | None                                       |
| **Backend Agent**   | Gateway RPC implementation, REST API, Runtime Manager                            | Infrastructure Agent completion              |
| **Prompt Agent**    | `prompts/ea.md`, `prompts/hr_manager.md`, `prompts/gtf_manager.md`               | Design Document Chapter 3                  |
| **Frontend Agent**  | All pages and components of the React SPA                                        | Backend Agent interface definitions        |
| **MCP Agent**       | Memory MCP Server, OpenProject MCP, GitHub MCP, etc.                             | Infrastructure Agent completion              |

## 3. Project Directory Structure

```
simu-corp/
├── CLAUDE.md                          # Project Overview
├── DEVELOPMENT.md                     # This Document
├── 多agent协作系统设计.md              # 18-chapter Design Document (Chinese)
├── 模拟公司白皮书.md                   # Project Whitepaper (Chinese)
├── docker-compose.yml                 # Infrastructure Agent responsible
├── docker-compose.branch.yml          # Infrastructure Agent responsible
├── .env.example                       # Infrastructure Agent responsible
├── init-db.sql                        # Infrastructure Agent responsible
├── openclaw.json                      # Infrastructure Agent responsible (initial version)
├── mcp.json                           # Infrastructure Agent responsible
├── prompts/                           # Prompt Agent responsible
│   ├── ea.md
│   ├── hr_manager.md
│   └── gtf_manager.md
├── gateway/                           # Backend Agent responsible
│   ├── src/
│   │   ├── routes/                    # REST API routes
│   │   ├── rpc/                       # WebSocket RPC methods
│   │   ├── runtime/                   # Runtime Manager
│   │   └── middleware/                # Authentication, logging, etc. middleware
│   └── tests/
├── mcp-servers/                       # MCP Agent responsible
│   ├── memory-mcp/
│   ├── openproject-mcp/
│   └── github-mcp/
├── frontend/                          # Frontend Agent responsible
│   ├── src/
│   │   ├── components/
│   │   ├── pages/
│   │   ├── hooks/
│   │   ├── store/
│   │   ├── api/
│   │   ├── types/
│   │   └── i18n/
│   ├── package.json
│   └── vite.config.ts
├── scripts/                           # Infrastructure Agent responsible
│   ├── init.js
│   └── branch-init.js
└── docs/                              # Supplementary documents
     ├── api-mock.md                    # API Mock data
     ├── test-scenarios.md              # Test scenarios
     └── prompt-engineering-guide.md    # Prompt engineering guide
```

## 4. Git Branching Strategy

```
main                      # Main branch, only accepts merges, no direct commits
├── infra/setup           # Infrastructure Agent: Docker, configuration files
├── backend/api           # Backend Agent: REST API + WebSocket RPC
├── backend/runtime       # Backend Agent: Runtime Manager
├── prompts/core          # Prompt Agent: Souls for three core agents
├── frontend/mvp          # Frontend Agent: Phase 1 MVP pages
├── frontend/full         # Frontend Agent: Phase 2-3 full pages
├── mcp/memory            # MCP Agent: Memory MCP Server
├── mcp/openproject       # MCP Agent: OpenProject MCP
└── mcp/github            # MCP Agent: GitHub MCP
```

**Rules**:
- Each development agent works on its own branch.
- After completing a module, create a merge request to `main`.
- Before merging, ensure the module can run independently (via unit tests).
- Notify downstream agents to update dependencies after merging.

## 5. Commit Message Convention

```
<type>(<scope>): <subject>

type:
  feat     — New feature
  fix      — Bug fix
  docs     — Documentation update
  refactor — Code refactoring
  test     — Adding tests
  chore    — Build process or auxiliary tool changes

scope:
  infra    — Infrastructure
  backend  — Backend
  frontend — Frontend
  prompts  — Prompts
  mcp      — MCP Server

Examples:
  feat(backend): Implement agents.list RPC method
  fix(frontend): Resolve dashboard stats card refresh delay
  docs(prompts): Complete EA's Few-shot examples
  test(backend): Add end-to-end test for task timeout retry
```

## 6. API Mocking Convention

**Principle**: Frontend agents should not wait for Backend agents to complete all interfaces. Use mock data to develop the UI first.

**Mock Mode Switching**:
- Set `VITE_USE_MOCK=true` in the frontend's `.env.development` to enable Mock.
- Mock data files are located in `frontend/src/api/__mocks__/`.
- Each API interface corresponds to a mock file.

**Interface Contract**:
- Before implementing an interface, Backend agents must first provide a mock response example in `docs/api-mock.md`.
- Frontend agents develop based on the mock responses.
- After the backend interface is completed, the frontend switches `VITE_USE_MOCK=false` for integration testing.

## 7. Module Completion Notification (Handshake Protocol)

When a development agent completes a module, notify downstream agents in the following format:

```markdown
### [Module Completion Notification]

**Module Name**: agents.list RPC method
**Completion Time**: 2026-06-20 14:00
**Branch**: backend/api
**Change Summary**: Implemented the agents.list RPC method, supporting filtering by status/department/skill
**Interface Documentation**: See Design Document Chapter 10, Section 10.2.4
**Mock Data**: `docs/api-mock.md` updated
**Test Status**: Unit tests passed, awaiting integration testing
**Known Issues**: Pagination parameter `page_size` has no upper limit yet
**Downstream Impact**: Frontend agents can now use this interface to fetch agent lists
```

## 8. Code Standards

**TypeScript/JavaScript**:
- Use ESLint + Prettier; configuration files are at the project root.
- Functions must have JSDoc comments (at least describing parameters and return values).
- Use `async/await` for all asynchronous operations.

**React Components**:
- One component per file.
- Component Props must have TypeScript type definitions.
- Use Shadcn/ui components; do not reinvent the wheel.

**Prompt Files**:
- Markdown format.
- Use code blocks to denote structured output formats.
- Few-shot examples use `### Example N` separators.

## 9. Testing Standards

- **Unit Tests**: Each module must cover core logic (target coverage > 80%).
- **Integration Tests**: Cross-module interactions must have integration tests (e.g., the complete EA routing → GTF execution chain).
- **End-to-End Tests**: After Phase 1 completion, write comprehensive user scenario tests.

Test file locations:
- Backend: `gateway/tests/`
- Frontend: `frontend/src/__tests__/`
- MCP: `mcp-servers/<name>/tests/`

## 10. Issue Escalation Mechanism

When a development agent encounters an issue it cannot resolve independently, seek help according to the following priority:

1. **Consult Design Documents**: Search relevant chapters in `多agent协作系统设计.md`.
2. **Consult Whitepaper**: Understand design intent in `模拟公司白皮书.md`.
3. **Consult This Document**: Check if there are existing conventions.
4. **Ask the CEO**: Clearly describe the problem, attempted solutions, and required assistance.

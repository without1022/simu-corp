# SimuCorp — Self-Evolving Multi-Agent Collaboration System

**Company as Code, Organization as Intelligence.** Let AI Agents collaborate, divide labor, and grow themselves — just like real company employees.

> ⚠️ **Project Status**: Complete design documentation (18 chapters) and project whitepaper finished. Phase 1 core pipeline under active development. Contributors welcome!

---

## What Is This?

SimuCorp is an OS-level framework that manages AI Agents using a **corporate governance structure**. Unlike traditional "temporary crew" style multi-agent frameworks, SimuCorp simulates the operation of a real company:

- 🏢 **Executive Assistant (EA)** — receives tasks, routes and dispatches, manages global state
- 👥 **HR Department** — automatic recruitment, internship evaluation, promotion, skill matching
- 🔧 **General Task Force (GTF)** — fallback execution with skill crystallization, filling capability gaps
- 📋 **L0–L3 Approval Hierarchy** — CEO retains ultimate decision-making authority
- 🌐 **Branch Office Mechanism** — other machines join as "branch offices" for distributed deployment
- 📈 **Self-Evolution Loop** — from execution to retrospection to hiring to promotion, fully automated

## Quick Start

```bash
# 1. Clone the repo
git clone https://github.com/without1022/simu-corp.git
cd simu-corp

# 2. Read the design doc (required!)
open docs/多Agent协作系统设计.md

# 3. Understand the design philosophy
open docs/模拟公司白皮书.md

# 4. Check current development status
open ROADMAP.md
```

## Why SimuCorp?

| Existing Frameworks | Our Difference |
|---------------------|----------------|
| CrewAI / AutoGen: temporary roles | SimuCorp: **long-term employment**, internship → promotion |
| MetaGPT: fixed pipelines | SimuCorp: **self-evolution**, auto-detect skill gaps & hire |
| Static departmental systems | SimuCorp: **dynamic organization**, new departments via approval |
| OMC (academic): proof of concept | SimuCorp: **complete engineering design**, 18 chapters + deployable blueprint |

## Architecture Overview

```
                           ┌─────────────┐
                           │   CEO / User  │
                           └──────┬──────┘
                                  │ Task input
                          ┌───────▼────────┐
                          │  EA (Assistant) │
                          │ Route/Dispatch  │
                          └───┬────┬────┬───┘
                              │    │    │
                    ┌─────────┘    │    └─────────┐
                    ▼              ▼              ▼
            ┌───────────┐  ┌───────────┐  ┌───────────┐
            │     HR    │  │    GTF    │  │ Business  │
            │ Recruit   │  │ Fallback  │  │  Depts    │
            └───────────┘  └───────────┘  └───────────┘
                    │              │              │
                    └──────────────┴──────────────┘
                                   │
                           ┌───────▼────────┐
                           │  MCP Tool Market│
                           │ External Services│
                           └────────────────┘
```

## Project Structure

```
simu-corp/
├── README.md                   # Project overview
├── README_EN.md                # English overview
├── LICENSE                     # MIT License
├── CONTRIBUTING.md             # Contribution guide
├── ROADMAP.md                  # Development roadmap
├── CLAUDE.md                   # AI-assisted dev context
├── docs/                       # Design documents
│   ├── 多Agent协作系统设计.md    # 18-chapter design (7649 lines)
│   ├── 模拟公司白皮书.md         # Project whitepaper
│   ├── AI-OS-演化白皮书.md        # AI OS evolution whitepaper
│   ├── agent-template-instance-model.md
│   ├── agent-marketplace.md
│   ├── api-mock.md
│   ├── test-scenarios.md
│   ├── prompt-engineering-guide.md
│   ├── mcp-server-template.md
│   ├── template-management.md
│   ├── operations.md
│   ├── audit-report.md
│   ├── design-review-report.md
│   └── template-market-alignment.md
├── prompts/                    # Agent Soul prompts
│   ├── ea.md                   # Executive Assistant
│   ├── hr_manager.md           # HR Manager
│   └── gtf_manager.md          # GTF Manager
└── development/
    └── DEVELOPMENT.md          # Dev agent collaboration spec
```

## Seeking Contributors

We are looking for contributors in these areas:

- 🖥️ **Frontend** (React + TypeScript + Tailwind)
- ⚙️ **Backend/Gateway** (Node.js, WebSocket, REST API)
- 🧩 **MCP Server Development** (tool integration)
- 🤖 **Agent Prompt Engineering** (writing and optimizing Agent Souls)
- 📝 **Documentation & Testing**

Please read [CONTRIBUTING.md](CONTRIBUTING.md) to learn how to get involved.

## License

MIT License — see [LICENSE](LICENSE) for details.

---

**SimuCorp** — Let Agents not just work together temporarily, but truly "come to work" together.

# SimuCorp — AI OS Evolution White Paper

**From Application to Operating System: SimuCorp's Dual Architecture and Evolution Path**

> **Version**: v1.0  
> **Status**: Design Philosophy and Vision Document  
> **Related Documents**: 18-Chapter Design Document, Project White Paper, Agent Market Design

---

## 1. Introduction: From "Building a Structure" to "Designing a City"

### 1.1 What We Have Accomplished

Over the past several months, we have completed the full engineering design of SimuCorp—an 18-chapter design document, a project white paper, and 9 supplementary development specifications. This design defines an **application system that manages AI Agents through corporate governance structures**:

- Executive Assistant (EA) responsible for task routing
- Human Resources (HR) responsible for Agent recruitment and evaluation
- General Task Force (GTF) responsible for fallback execution and skill accumulation
- L0-L3 approval hierarchy ensuring human oversight
- Branch mechanism supporting multi-node collaboration

This constitutes a complete, deployable "AI Company" application. Yet our vision extends beyond this.

### 1.2 The "OS Potential" Observed Within the Application

During the in-depth design process, we identified a phenomenon: SimuCorp's core modules—organizational management, task scheduling, security approval, memory storage—**do not merely serve application functions; they represent fundamental responsibilities of an operating system**.

| Our Design Module | Traditional OS Equivalent | Explanation |
|:---|:---|:---|
| Agent Template & Instance Management | Process Management (PID, fork, exec) | OS manages process lifecycle; we manage Agent lifecycle |
| Task Routing & Capability Scheduling | CPU Scheduler | OS determines which process gets CPU time; we determine which Agent gets a task |
| L0-L3 Approval Hierarchy | User Mode/Kernel Mode + Permission Bits | OS has privilege levels; we have approval levels |
| Message Bus + WebSocket | IPC (Pipe, Socket, Signals) | OS provides inter-process communication; we provide inter-Agent communication |
| ChromaDB + Redis Memory | File System + Virtual Memory | OS manages storage and memory; we manage memory persistence and retrieval |
| MCP/A2A Protocols | POSIX Standard Interfaces | OS provides standard system calls; we provide standard Agent protocols |

This is not a metaphor. It is an **isomorphic mapping**.

Upon this realization, SimuCorp's positioning fundamentally shifts: it is no longer an "application" running on an operating system, but rather an **organizational operating system simulated through Agents**.

### 1.3 Purpose of This Document

This white paper aims to:

1. **Articulate the Dual Architecture**: Define both SimuCorp's "Application Architecture" (currently designed) and its "OS Architecture" (future evolution direction)
2. **Demonstrate the Evolution Path**: Illustrate that the three architectural transitions from application to OS are feasible, natural, and inevitable
3. **Elevate Project Positioning**: Reposition SimuCorp from "an AI organizational tool" to "an open-source reference implementation of an AI OS paradigm"

---

## 2. Dual Architecture Analysis

### 2.1 Application Architecture: The Meticulously Designed "Corporate Building"

The Application Architecture represents SimuCorp's current design form. It is a complete, deployable multi-Agent collaboration system targeting **individuals or organizations seeking to accomplish complex tasks using AI Agents**.

**Core Characteristics of Application Architecture**:
- **Single Organization**: One CEO manages one "Company"
- **Complete Closed Loop**: Task assignment → Execution → Review → Recruitment → Evolution, all within a single system
- **Human-Centric Interaction**: Dialogue with EA via WebChat, global view via Dashboard
- **Deployable**: One-click startup via Docker Compose

**Layered Application Architecture** (corresponding to Chapter 2 of the design document):

```
┌─────────────────────────────────────────┐
│          CEO Layer (Human User)          │
│          WebChat / Dashboard             │
├─────────────────────────────────────────┤
│          Organizational Logic Layer      │
│     EA / HR Mgr / GTF Mgr / PM          │
│     Department Management / Approval     │
│     Hierarchy / Self-Evolution Loop      │
├─────────────────────────────────────────┤
│          Gateway Layer                   │
│     OpenClaw Gateway                    │
│     Message Routing / Session Mgmt / Auth│
├─────────────────────────────────────────┤
│          Infrastructure Layer            │
│     Model Relay / MCP Tool Bus / Memory  │
│     PostgreSQL / Redis / ChromaDB        │
└─────────────────────────────────────────┘
```

**Metaphor for Application Architecture**: A meticulously designed "Corporate Building"
- CEO works on the top floor
- EA serves as receptionist/butler, handling reception and task assignment
- Departments work on their respective floors
- Infrastructure constitutes the building's utilities and plumbing

### 2.2 OS Architecture: The "City Foundation" Supporting Everything

The OS Architecture is SimuCorp's evolutionary goal. It sinks the core mechanisms of the Application Architecture into **system-level infrastructure**, transforming SimuCorp from merely managing one "Company" into the underlying platform for all AI Agent activities.

**Core Characteristics of OS Architecture**:
- **Multi-Organization**: Multiple independent "Company Instances" can run on the same OS
- **Open Ecosystem**: Any external AI application can connect via MCP/A2A protocols
- **System-Level Operation**: Gateway runs as a daemon process, starting with the system
- **Resource Abstraction**: Unified management of heterogeneous LLMs, GPUs, and tool resources

**Layered OS Architecture**:

```
┌─────────────────────────────────────────────────────────┐
│                    Application Ecosystem Layer            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │ SimuCorp A│  │ SimuCorp B│  │ Third-Party AI App │  │
│  │ (Org Inst)│  │ (Org Inst)│  │ (MCP Access)  │      │
│  └──────────┘  └──────────┘  └──────────┘              │
│                                                         │
│  Standard Interface Layer: MCP (Tool Access) / A2A      │
│  (Agent Communication) / HTTP API                       │
├─────────────────────────────────────────────────────────┤
│                    Four Core Kernels                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐│
│  │Org Kernel│  │Sched Kernel│  │Sec Kernel│  │Mem Kernel││
│  │          │  │          │  │          │  │          ││
│  │Template  │  │Capability│  │Approval  │  │Session   ││
│  │Mgmt      │  │Directory │  │Hierarchy │  │Memory    ││
│  │Instance  │  │Task Route│  │Sandbox   │  │Working   ││
│  │Lifecycle │  │Base Sched│  │Isolation │  │Memory    ││
│  │Self-Evol │  │Load Bal  │  │Audit Log │  │Long-Term ││
│  │Market Eco│  │          │  │Perm Ctrl │  │Memory    ││
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘│
│                                                         │
│  Underlying Resource Pool: LLM Cluster / GPU Cluster    │
│  / Tool Cluster / Storage Cluster                       │
└─────────────────────────────────────────────────────────┘
```

**Metaphor for OS Architecture**: The "Municipal Foundation" of an AI City
- Application Ecosystem Layer comprises various buildings in the city (companies, studios, marketplaces)
- The Four Core Kernels are the city's main thoroughfares and foundations
- MCP/A2A are the standard pipelines connecting all buildings
- Underlying Resource Pool is the city's energy and water supply

### 2.3 Relationship Between Application and OS Architectures

```
Application Architecture  ————————————————→  OS Architecture
(Corporate Building)                       (City Foundation)

Single Organization         →          Multi-Organization Ecosystem
Deployable Application      →          System-Level Infrastructure
Internal Closed Loop        →          Open Standard Protocols
CEO-Facing                  →          All AI Application-Facing
```

**Key Understanding**: The Application Architecture is an instance of the OS Architecture. Just as any building in a city both utilizes the city's foundation and pipelines and is part of the urban ecosystem, the SimuCorp application itself is "the first organizational instance running on the SimuCorp OS."

---

## 3. Detailed Explanation of the Four Core Kernels

### 3.1 Organization Kernel

**Responsibility**: Manage the "life" of all Agents—the complete lifecycle from creation to termination.

**Corresponding Existing Design**:
- Chapter 3: Organizational Model and Core Agent Design (EA, HR, GTF)
- Chapter 5: Self-Evolution Loop (Recruitment, Internship, Probation, Termination)
- Chapter 15: Scalability Design (New Department Types, New Template Integration)
- Agent Market Design Document

**OS-Level Expansion**:
- In the Application Architecture, the Organization Kernel manages Agents **within a single company instance**
- In the OS Architecture, the Organization Kernel manages **all organizational instances connected to the OS**—including multiple SimuCorp instances, third-party AI applications, and standalone Agents
- The Organization Kernel maintains a global "Organization Registry," analogous to the process table in an operating system

**Key Mechanisms**:
- **Template-Instance Inheritance Model**: Abstract job definitions (templates) are separated from specific employees (instances); the same template can derive multiple instances with different base models
- **Self-Evolution Engine**: Not merely optimization of individual Agents, but self-adjustment of organizational structure—creation of new departments, merging of old ones, elastic scaling of Agent count
- **Ecosystem Marketplace**: Cross-organizational circulation of Agent templates, forming a "talent market"

### 3.2 Scheduler Kernel

**Responsibility**: Determine "who does what and when"—the core function of an OS.

**Corresponding Existing Design**:
- Chapter 4: Capability Synchronization and Task Routing
- Chapter 6: Base Model Abstraction and Unified Memory Management
- Chapter 8: Model Relay and Independent Model Configuration

**OS-Level Expansion**:
- In the Application Architecture, EA handles task routing within a single organization
- In the OS Architecture, the Scheduler Kernel is a **system-level resource scheduler**, analogous to Linux's CFS (Completely Fair Scheduler)
- It schedules not only tasks to Agents, but also:
  - **Model Calls**: Which task uses which model and provider
  - **Hardware Resources**: Which task is allocated to which GPU node
  - **Base Instances**: Which Agent instance (OpenClaw/Claude Code/Codex) undertakes the task

**Key Mechanisms**:
- **Capability Directory**: Global Agent capability registry, updated in real-time
- **Weighted Matching Algorithm**: Skill match degree + load balancing + base model capability adaptation
- **Degradation and Failover**: Agent offline → task transfer, model outage → automatic switch

### 3.3 Security Kernel

**Responsibility**: Ensure that Agent autonomous behavior does not go out of control—the greatest difference between an AI OS and a traditional OS.

**Corresponding Existing Design**:
- Chapter 12: Security and Permission Model
- Chapter 13: Monitoring, Logging, and Auditing
- Chapter 14: Error Handling and Resilience Design

**OS-Level Expansion**:
- In the Application Architecture, the Security Kernel protects the operational security of **one company**
- In the OS Architecture, the Security Kernel is the **immune system of the entire AI ecosystem**—not only protecting individual organizations but also preventing the spread of security incidents between organizations

**Key Mechanisms**:
- **L0-L3 Approval Hierarchy**: Analogous to user mode/kernel mode hierarchy in operating systems
- **Sandbox Isolation**: High-risk operations must be verified in a Docker sandbox before execution
- **Audit Log**: All critical operations are append-only, immutable, and traceable
- **Permission Matrix**: Similar to Linux UID/GID and file permission bits; each Agent has its own permission level

### 3.4 Memory Kernel

**Responsibility**: Manage the storage, retrieval, and forgetting of all knowledge—a kernel unique to AI OS, absent in traditional OS.

**Corresponding Existing Design**:
- Chapter 11: Data Model and Storage
- Chapter 6: Unified Memory Management (Memory MCP Server)

**OS-Level Expansion**:
- In the Application Architecture, the Memory Kernel manages the knowledge assets of **one organization**
- In the OS Architecture, the Memory Kernel is the **distributed knowledge base of the entire AI ecosystem**—akin to a "global library"

**Key Mechanisms**:
- **Three-Layer Memory Model**: Session Memory (Redis), Working Memory (Redis), Long-Term Memory (ChromaDB)
- **Namespace Isolation**: Strict isolation by `agent_id`; sharing requires whitelist authorization
- **Memory Lifecycle**: Creation, retrieval, compression, forgetting, archiving
- **Cross-Organizational Knowledge Flow**: Through universal memory packages in the Agent Market, knowledge can legally circulate between organizations

---

## 4. Three Transitions from Application to OS

### 4.1 First Transition: From Application Process to System Service

**Current State**: Gateway is a Node.js application process, started via `docker-compose up`.

**Transition Goal**: Gateway becomes a **system-level daemon process** (similar to systemd), automatically starting with the operating system.

**Specific Changes**:

| Dimension | Application Form | OS Form |
|:---|:---|:---|
| Startup Method | `docker-compose up` | Auto-load on system startup (systemd unit) |
| Lifecycle | Starts/stops with Docker container | Continuous operation, auto-restart on crash |
| Privilege Level | User-mode application | System service (root/system privileges) |
| Port Usage | User-space port (18789) | Can occupy privileged ports |

**Core Significance of This Step**:
- Gateway is no longer "optional" but "mandatory"—just as you cannot uninstall the Linux kernel
- Any application requiring AI capabilities passes through Gateway by default
- Gateway becomes the unified entry point for all AI calls on the machine

**Implementation Path**:
1. Write a systemd unit file for Gateway
2. Add daemon mode to Gateway
3. Add system-level health checks and auto-restart

### 4.2 Second Transition: From Single Machine to Distributed

**Current State**: The branch mechanism is defined in the design but implemented as single-machine simulation.

**Transition Goal**: Gateway becomes a **distributed scheduler**, managing Agents and resources across multiple physical machines.

**Specific Changes**:

| Dimension | Application Form | OS Form |
|:---|:---|:---|
| Resource Management | Single-machine Docker | Cluster (Kubernetes level) |
| Node Discovery | Manual branch configuration | Auto-discovery (similar to K8s Node Registration) |
| Task Scheduling | Based on Agent capability | Based on Agent capability + hardware resources + network topology |
| Failover | Agent level | Node level + Agent level |

**Core Significance of This Step**:
- Branches no longer require manual connection—new machines automatically register and report capabilities upon joining the cluster
- Gateway becomes the "Kubernetes control plane" for the AI era—scheduling Agents and LLM calls rather than containers
- Organizations can scale elastically: automatically expand Agent instances during high task volume, automatically reclaim during idle periods

**Implementation Path**:
1. Change branch registration to auto-discovery (mDNS or message bus-based)
2. Add resource-aware scheduling to Gateway (CPU, GPU, memory as routing factors)
3. Add cross-node task migration (if one node goes down, tasks automatically migrate to other nodes)

### 4.3 Third Transition: From Private Protocols to Open Standards

**Current State**: Internal use of custom message formats, tool integration via MCP.

**Transition Goal**: MCP and A2A become **first-class citizens** of the system—not merely internal protocols, but standard external interfaces.

**Specific Changes**:

| Dimension | Application Form | OS Form |
|:---|:---|:---|
| Tool Access | Manual MCP Server registration | MCP Server marketplace, one-click access |
| Agent Communication | Internal message bus | A2A protocol, cross-framework interoperability |
| External Application Access | None | Any application implementing MCP/A2A can register as an "Organization" |
| Ecosystem Governance | None | MCP/A2A compatibility certification |

**Core Significance of This Step**:
- A third-party developer writes a new AI tool; as long as it implements the MCP protocol, it can be discovered and managed by the SimuCorp OS
- Google's Agent (via A2A) and Anthropic's Agent (via MCP) can collaborate on the same OS
- SimuCorp OS becomes the "Android" of the AI Agent ecosystem—an open, standardized platform

**Implementation Path**:
1. Gateway implements the complete A2A Server specification
2. Standardize the MCP Server integration process (one-click registration, automatic approval)
3. Provide MCP/A2A compatibility certification tools

---

## 5. Inheritance and Elevation of Existing Design Documents

### 5.1 Mapping of OS Four Core Kernels to Design Document Chapters

| OS Kernel | Corresponding Design Document Chapters | Elevation from Application to OS |
|:---|:---|:---|
| **Organization Kernel** | Chapter 3 (Organizational Model), Chapter 5 (Self-Evolution), Agent Market Design | From managing one organization → managing multiple organizational instances; market from internal circulation → global ecosystem |
| **Scheduler Kernel** | Chapter 4 (Task Routing), Chapter 6 (Base Model Abstraction), Chapter 8 (Model Configuration) | From single-machine routing → distributed scheduling; from Agent scheduling → resource scheduling |
| **Security Kernel** | Chapter 12 (Security Permissions), Chapter 13 (Monitoring & Auditing), Chapter 14 (Error Handling) | From organizational-level security → ecosystem-level immune system; from passive auditing → active threat detection |
| **Memory Kernel** | Chapter 11 (Data Storage), Chapter 6 (Unified Memory) | From organizational knowledge base → distributed knowledge network; from internal memory → cross-organizational knowledge flow |

### 5.2 Design Documents Require No Rewriting

**Key Conclusion**: Our existing 18-chapter design document, white paper, and 9 supplementary documents **do not need to be discarded and rewritten**. They describe the complete design of the Application Architecture, and the Application Architecture itself is a complete instance of the OS Architecture.

**Content to Be Added**:
1. This white paper (AI OS Evolution White Paper)—describing the dual architecture and evolution path
2. OS Architecture Diagram (cyber city style)—visual representation of the OS architecture
3. Four Core Kernel Interface Specifications
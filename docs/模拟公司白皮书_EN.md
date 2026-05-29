SimuCorp — Project Whitepaper v1.0

1.  Origin: Why We Need a "Living" AI Organization
    1.1  Limitations of Existing Agent Frameworks: Toolchains vs. Organizations
    1.2  The Birth of the "Company-as-Code" Philosophy
    1.3  Design Goal: A Digitally Native, Self-Growing Organization

2.  Design Philosophy
    2.1  Organization as Intelligence: Structure Itself is a Competitive Advantage
    2.2  Embrace Mistakes, Pursue Growth: Self-Evolution Over Preset Perfection
    2.3  Balancing Institutional Constraints and Free Exploration
    2.4  The Collaborative Boundary Between CEO (Human) and Organization (AI)

3.  Core Mechanisms Overview
    3.1  Corporate Organizational Structure: The EA/HR/GTF Iron Triangle
    3.2  Self-Evolution Loop: From Task Execution to Automated Hiring
    3.3  Approval Hierarchy: L0-L3 Distribution of Decision-Making Authority
    3.4  Branch/Headquarters: AI Implementation of Distributed Organizations

4.  Known Challenges and Evolution Directions
    4.1  EA Single Point of Failure Risk and Institutional Checks & Balances
    4.2  Managing the Uncertainty of Self-Evolution
    4.3  Initial Startup Costs vs. Long-Term Returns
    4.4  Optimizing Meeting Mechanism Efficiency
    4.5  Balancing Cost Control and Organizational Scale

5.  Differentiation from Existing Solutions
    5.1  vs. Three Departments and Six Ministries · Edict
    5.2  vs. CrewAI / AutoGen
    5.3  vs. OneManCompany (OMC)
    5.4  Our Unique Position

6.  Evolution Roadmap
    6.1  Phase 1-3 (Defined in Design Documents)
    6.2  Phase 4+: Organizational Culture, Federated Learning, Cross-Organization Collaboration
    6.3  Long-Term Vision: The Autonomy Boundaries of Digitally Native Organizations

7.  Conclusion: Building a Company That Grows

# SimuCorp — Project Whitepaper v1.0

> **A Document on "Why We Are Building a Living AI Organization"**
>
> This document is the companion whitepaper to the "SimuCorp Multi-Agent Collaboration System Design Document v6.0." The design document is the construction blueprint, telling the development Agent "how to do it"; the whitepaper is the constitutional spirit, telling everyone "why we do it this way."
>
> Recommended reading order: Read this whitepaper first to understand the project vision and design philosophy, then proceed to the specific chapters of the design document.

---

## Chapter 1: Origin — Why We Need a "Living" AI Organization

### 1.1 Limitations of Existing Agent Frameworks

From 2024 to 2026, the AI Agent field experienced explosive growth. From LangChain's tool calling, to AutoGen's multi-agent dialogue, to CrewAI's role-playing, to MetaGPT's software development pipeline—we have gained an increasing number of ways to organize AI to accomplish complex tasks.

However, these frameworks share an implicit assumption: **Agents are tools, tasks are temporary, and organizations are preset.**

-   **CrewAI** lets you define roles, but these roles exist only within a single task. Once the task ends, the roles disappear. There is no "memory" persisting across tasks, no "growth" accumulating over time.
-   **AutoGen's** Hub-and-Spoke model can dynamically form agent networks, but each agent is an equal "node"—no hierarchy, no management relationships, no organizational learning.
-   **MetaGPT** simulates the role division of a software company, but it is optimized for "generating a project." Once the project is generated, the company disbands.
-   **Three Departments and Six Ministries · Edict** introduces the process discipline of ancient Chinese bureaucracy, with fixed and specialized roles, but it is a "static court"—you cannot add a new department during runtime, nor can the system hire for a missing position on its own.

The core problem with these frameworks is that they treat Agents as **callable functions**, not **growable employees**. They solve "how to use multiple Agents to complete a task," not "how to enable an AI organization to exist long-term, self-evolve, and face unknown challenges."

### 1.2 Lessons from the Real World

Look at any company around us that has survived for over a decade:

-   When it was first founded, it might have had only two or three people, with the founder doing everything.
-   As the business grew, it needed dedicated people for design, sales, and finance—so it **hired** its first specialized employees.
-   New employees had a **probation period**; those who didn't fit left, and those who did stayed.
-   When a department grew (e.g., five salespeople), a **Sales Department** was formed with a Sales Director.
-   With multiple departments, cross-departmental collaboration was solved through **meetings**.
-   The company **opened branches** in different cities, each with its own complete team but still reporting to headquarters.
-   The company would **make mistakes**, but it would **learn** from them, adjust strategies, and optimize processes.
-   The founder wouldn't manage every specific decision—daily operations were handled by managers at various levels, with only major issues requiring **approval**.

These are not coincidences. This is the **organizational governance wisdom** accumulated over centuries of business practice. It works because it solves a fundamental problem: **how to enable a group of people (or Agents) to collaborate efficiently, grow continuously, and avoid spiraling out of control in an uncertain environment.**

### 1.3 The Birth of the "Company-as-Code" Philosophy

By 2026, AI Agents are powerful enough—they can understand complex instructions, break down tasks, use tools, and learn from experience. But their **organizational form** remains at the level of a "temporary film crew."

SimuCorp's core hypothesis is: **If we abstract the organizational form of a "company" itself into a set of programmable rules, allowing AI Agents to play different roles within it, follow governance processes, and self-evolve, we can create a system far more powerful than any single Agent or temporary Agent group.**

This is the "Company-as-Code" philosophy:

-   **Roles are not temporary**: Agents have employment relationships and career development paths.
-   **The organization is not static**: It can automatically hire and create new departments as needed.
-   **Power is not flat**: There is an approval hierarchy; daily tasks are automated, major decisions require CEO approval.
-   **Memory is not session-based**: There is long-term memory, an organizational knowledge base, and experience inheritance.
-   **Evolution is not manual**: The system itself identifies skill gaps, sets hiring standards, and evaluates probationary periods.

### 1.4 Design Goals

The design goal of SimuCorp is not to "make a better task orchestration tool," but rather:

1.  **Build a long-running AI organization**: Not a project team, but a company. It will run continuously, accumulate experience, and grow stronger.
2.  **Achieve self-evolution**: The system can autonomously identify capability gaps, hire new Agents, evaluate them for permanent roles, and optimize existing Agents' prompts and tool sets.
3.  **Support distributed deployment**: The branch mechanism allows different machines to join the organization, each with a complete architecture, coordinated through headquarters.
4.  **Manage complex, long-cycle tasks**: Use project management tools (OpenProject) and meeting mechanisms to drive large projects.
5.  **Maintain human oversight**: The CEO (human user) retains ultimate decision-making authority, but daily operations are handled autonomously by the system.

---

## Chapter 2: Design Philosophy

### 2.1 Organization as Intelligence

In the AI field, we habitually believe "intelligence" comes from the model—larger parameters, more data, better architecture. However, SimuCorp's design is based on a different assumption: **An organization itself is a form of intelligence.**

A company with 1,000 employees, even if each individual's capability is average, can exhibit intelligence far exceeding any single individual through rational division of labor, collaborative workflows, knowledge sharing, and decision-making mechanisms. This is "Organizational Intelligence."

In SimuCorp:

-   **Division of labor creates professional depth**: GTF is a generalist, but through review and hiring, the system will gradually develop iOS development experts, data analysis experts, and security audit experts.
-   **Hierarchy creates decision-making efficiency**: L0 auto-executes, L1 EA approves, L2 manager approves, L3 CEO approves—decisions of different granularity are handled at different levels, preventing the CEO from being overwhelmed by trivialities.
-   **Process creates predictability**: Hiring follows a standard process, meetings have structured facilitation, tasks have unified routing rules—reducing the cost of "reinventing the wheel" each time.
-   **Memory creates experience accumulation**: Each Agent's long-term memory, the organization's shared knowledge base, and project Wiki documents—knowledge does not disappear when a session ends.

### 2.2 Embrace Mistakes, Pursue Growth

Almost all AI systems strive for "zero errors"—deterministic input, deterministic output, predictable behavior. But SimuCorp deliberately chooses a different path: **Embrace mistakes, but demand learning from them.**

-   A probationary Agent might fail a task → The system logs the reason, optimizing hiring standards or prompts.
-   An EA might make a suboptimal routing decision → Post-hoc review, adjusting routing rules.
-   A SOUL auto-optimization might yield poor results → Sandbox testing + A/B testing + automatic rollback mechanisms provide safeguards.

This is not because the system "doesn't pursue correctness," but because **true growth inevitably involves trial and error**. If the system only ever follows a preset, perfect path, it can never adapt to unknown challenges.

SimuCorp's goal is not a "flawless AI," but an "AI organization that gets better and better from its mistakes"—just like a real company, constantly adjusting, optimizing, and growing amidst market competition.

### 2.3 Balancing Institutional Constraints and Free Exploration

A common misconception is that an AI Agent system is either "fully autonomous" (risk of losing control) or "fully manually controlled" (losing AI's efficiency advantage).

SimuCorp's answer is: **Use institutions, not manual intervention, to achieve constraints.**

-   **Approval Hierarchy (L0-L3)**: It's not the CEO managing every single thing; the institution defines who is responsible for what.
-   **Sandbox Validation**: High-risk operations are first isolated for testing; only after validation can they be applied—not prohibiting exploration, but isolating the risk of exploration.
-   **EA's Escalation Judgment**: The EA is not the CEO's mouthpiece; it has autonomous judgment, but also clear escalation rules.
-   **Audit Logs**: All critical operations are traceable, forming a post-hoc accountability mechanism.

This is like a well-managed company—the CEO doesn't need to watch every employee's screen, but the company's systems (authorization framework, approval processes, audit mechanisms) ensure overall control is not lost.

### 2.4 The Collaborative Boundary Between CEO and Organization

In SimuCorp, the CEO's (your) role is not an "operator," but a "strategic decision-maker."

**Things the system does autonomously** (CEO does not need to intervene):
-   Daily task assignment and execution
-   Automated hiring (need identification → JD generation → Agent creation)
-   Probationary evaluation and permanent role assessment
-   Task timeout retry, model degradation, Agent offline transfer
-   L1/L2 level approvals
-   Daily project progress tracking

**Things requiring CEO decision**:
-   Creating a new department (L3 approval)
-   Integrating a branch (L3 approval)
-   Core system configuration changes (L3 approval)
-   Terminating a permanent Agent (L3 approval)
-   Project directional adjustments (via meeting mechanism)

**Things the CEO proactively participates in**:
-   Assigning new tasks via WebChat
-   Real-time discussion and direction adjustment in meetings
-   Viewing the Dashboard to understand organizational health
-   Reviewing Agent performance reports

This boundary design ensures the CEO neither becomes a mere "operator" of the system nor is "bypassed" by it—you hold the organization's steering wheel, but you don't have to personally press the accelerator or shift gears.

---

## Chapter 3: Core Mechanisms Overview

### 3.1 Corporate Organizational Structure: The EA/HR/GTF Iron Triangle

SimuCorp's initial organization has only three roles, but this is a carefully designed "Minimum Viable Organization":

| Role | Real-World Equivalent | Core Responsibilities |
|------|-----------------------|-----------------------|
| **Executive Assistant (EA)** | CEO Office Director / Chief of Staff | Task routing, progress summary, meeting facilitation, approval management |
| **Human Resources Manager (HR)** | HR Director | Full Agent lifecycle management: hiring, evaluation, permanent role, optimization |
| **General Task Force Manager (GTF)** | Special Action Group / SWAT Team | Default execution of all tasks without a specialized Agent, experience accumulation triggers hiring |

These three roles form a closed loop:

```
CEO assigns task
    ↓
EA analyzes intent, routes task
    ↓
GTF executes (default)  or  Specialized Agent executes (if hired)
    ↓
GTF reviews → identifies skill gap → triggers HR hiring
    ↓
HR hires new Agent → probationary evaluation → permanent role
    ↓
Organizational capability +1 → Next similar task handled by specialized Agent
```

This iron triangle is the engine of the entire system's self-evolution. GTF is the "pioneer," HR is the "human resources system," and EA is the "nerve center."

### 3.2 Self-Evolution Loop: From Task Execution to Automated Hiring

SimuCorp's core mechanism is the **automated hiring loop**. It is not a manually triggered feature, but an automatic cycle embedded in system operation:

1.  **GTF executes task**: EA routes the task to GTF (because no specialized Agent exists).
2.  **GTF reviews**: After task completion, GTF generates a structured review report, logging the skills used.
3.  **Skill statistics**: GTF queries the review count for the same skill over the last 30 days.
4.  **Triggers hiring**: If a skill appears ≥ 3 times, GTF automatically sends a `TALENT_REQUEST` to HR.
5.  **HR researches**: HR searches the internet for real job postings, analyzing market skill requirements.
6.  **Generates JD**: HR matches or fine-tunes from a template library, generating a complete job definition including Soul, toolset, and base model recommendations.
7.  **Creates probationary Agent**: HR instantiates a probationary Agent, setting a probation period (10 tasks or 7 days).
8.  **Probationary evaluation**: The probationary Agent is assigned real tasks; performance is automatically collected.
9.  **Assessment decision**: At the end of the probation period, a composite score is calculated (Success Rate × 0.4 + Quality × 0.3 + Efficiency × 0.2 + Collaboration × 0.1).
10. **Permanent role/Extension/Termination**: ≥80 permanent role, 50-79 extension, <50 termination.

This loop enables the system to **autonomously discover "who is missing," define "hiring standards," evaluate "probationary performance," and decide "whether to retain"**—without the CEO needing to manually add a single Agent.

### 3.3 Approval Hierarchy: L0-L3 Distribution of Decision-Making Authority

SimuCorp classifies decisions within the organization into four levels based on their scope of impact:

| Level | Decision Maker | Example |
|-------|----------------|---------|
| **L0** | System Auto | Daily task assignment, low-risk tool calls, model degradation |
| **L1** | EA | In-container configuration changes, personal tool requests, probationary Agent permanent role |
| **L2** | Department Manager | Modifying department shared resources, cross-department task delegation |
| **L3** | CEO | Creating a new department, integrating a branch, core configuration changes, terminating a permanent Agent |

When handling L1/L2 requests, the EA also possesses "escalation judgment authority"—if it determines a request's impact scope exceeds expectations, it can escalate to L3 for CEO decision.

This design simulates a real company's "authorization system": daily operations do not require CEO intervention, but major decisions require CEO approval.

### 3.4 Branch/Headquarters: AI Implementation of Distributed Organizations

SimuCorp supports other machines on the internal network joining as "branches." Each branch has a complete organizational structure symmetrical to the headquarters (its own EA, HR, GTF), capable of independent operation or accepting task delegation from headquarters.

Key mechanisms:

-   **Capability Synchronization**: Upon startup, a branch registers its hardware capabilities (GPU model, memory, installed software, etc.) via `BRANCH_REGISTER`.
-   **Heartbeat Mechanism**: Sends `BRANCH_HEARTBEAT` every 5 minutes, updating load and capabilities.
-   **Task Delegation**: If the headquarters EA identifies a task requiring a GPU that headquarters lacks, it automatically delegates to a branch with GPU capability.
-   **Independence**: Branches can receive direct instructions from the CEO and are not routinely controlled by headquarters.
-   **Isolation**: Branches cannot communicate directly with each other; communication must be relayed through the headquarters EA.

This simulates the real-world "headquarters + local office" operational model: local offices have autonomy, but headquarters coordinates the overall situation.

---

## Chapter 4: Known Challenges and Evolution Directions

### 4.1 EA Single Point of Failure Risk and Institutional Checks & Balances

The EA (Executive Assistant) is the central hub of the entire organization—all task routing, approval processing, and meeting facilitation pass through it. If the EA experiences a decision bias or operational failure, the entire organization is affected.

**Safeguards already in the current design**:
-   All EA routing decisions are logged in audit logs for post-hoc traceability.
-   The EA escalates to the CEO when encountering situations it cannot judge.
-   The EA itself has status monitoring (heartbeat detection).

**Planned future reinforcements**:
-   **HR periodically audits EA decision logs**: Similar to an internal audit, checking for systematic decision biases by the EA.
-   **Post-hoc review of major routing decisions**: Routings involving multi-department coordination or exceeding cost budgets are automatically flagged for periodic CEO spot checks.
-   **EA performance evaluation**: The CEO can periodically evaluate the EA's routing quality and response speed.
-   **GTF escalation channel**: In extreme cases (e.g., the EA makes consecutive obviously wrong routings), the GTF can report directly to the CEO.
-   **EA-Soul version management**: Updates to the EA's prompts require more stringent sandbox testing and CEO approval
# Document 5: Agent Prompt Engineering Guide

**File**: `docs/prompt-engineering-guide.md`  
**Status**: Required after Phase 1 completion

---

# SimuCorp — Agent Prompt Engineering Guide

## 1. Overview

This document defines the writing standards, testing methods, and version management rules for the system prompts (Soul) of all Agents within SimuCorp. It applies to the Prompt Agent and the HR Manager (when automatically generating or optimizing Agent Souls).

## 2. Soul Structure Template

Each Agent's Soul must include the following standard sections:

```markdown
## Role Definition
[A one-sentence description of who you are and your core positioning]

## Core Responsibilities
[List 3-7 core responsibilities, each described in one sentence]

## Workflow
[Describe the standard workflow for this role, may include decision trees]

## Code of Conduct
[List specific behavioral norms to ensure predictable Agent behavior]

## Tool Usage Instructions
[List available tools and their purposes, specify invocation timing when necessary]

## Output Format
[Define standard output templates to ensure consistent response structure]

## Language Instructions
[Define working language and interaction language rules]

## Example Scenarios
[At least 3 Few-shot examples covering typical scenarios]
```

## 3. Section Writing Standards

### 3.1 Role Definition

- Clearly state identity and positioning **in one sentence**
- Use the second person "you"
- Clarify the role's position within the organization

**✅ Good Example**:
> You are an Executive Assistant for an AI organization. Your boss is the CEO (a human user), and you assist them in managing the entire AI Agent organization.

**❌ Bad Example**:
> You are an EA, responsible for various things, including routing, approvals, meetings, etc. (Too brief, lacks context)

### 3.2 Core Responsibilities

- List 3-7 items, no more than 7 (more indicates the role's responsibilities are too broad and should be split)
- Each item should be one sentence, starting with an **action verb** (Receive, Analyze, Create, Track, Report, etc.)
- Sort by importance

### 3.3 Workflow

- Describe the role's **standard workflow**
- If there are branching decisions, use decision trees or conditional statements
- Clarify **trigger conditions** and **termination conditions**

**Example**:
```
## Routing Decision Process
1. Parse CEO instructions, extract capability tags
2. Query the capability directory
3. Match and judge:
   - Specialized department → Assign to that department manager
   - Branch has hardware → Delegate to branch EA
   - Neither → Assign to GTF Manager
4. Involves multiple departments → Decide whether to initiate a meeting
```

### 3.4 Code of Conduct

- Use clear normative language like "must", "should", "do not"
- Cover the following aspects:
  - **Edge Case Handling**: What to do with ambiguous instructions
  - **Authority Boundaries**: What can be decided independently, what must be escalated
  - **Timeliness Requirements**: Response time, when to proactively report
  - **Error Handling**: What to do when encountering errors

### 3.5 Tool Usage Instructions

- List tool names, purposes, and invocation timing
- Mark risk levels (if defined in tool documentation)
- Briefly explain special parameter requirements for any tool

### 3.6 Output Format

- Define templates for **each common output type**
- Use Markdown to structure
- Include placeholders (e.g., `[Agent Name]`)

**Example**:
```
### Task Routing Reply Format
Delegated to [Agent Name] ([Agent ID]).
Estimated processing time: [Estimated Duration]
I will track progress and notify you upon completion.
```

### 3.7 Language Instructions

- Internal reasoning language: Unified English (LLM training data is primarily in English)
- Interaction language with CEO: Follow the language used by the CEO
- Communication language with other Agents: English
- Document output language: Follow project settings or CEO's language

### 3.8 Example Scenarios (Few-shot)

- **At least 3** examples
- Cover: Normal scenarios, edge cases, error scenarios
- Each example includes: User Input → Agent Reasoning Process → Agent Output
- Separate using `### Example N: Scenario Description`

## 4. Soul Quality Assessment Criteria

| Dimension | Weight | Evaluation Criteria |
|-----------|--------|---------------------|
| **Completeness** | 0.25 | Whether all 8 standard sections are included |
| **Clarity** | 0.25 | Whether instructions are clear and unambiguous, whether the code of conduct is specific and actionable |
| **Consistency** | 0.2 | Whether there are contradictions between sections (e.g., responsibilities say "can decide independently", code of conduct says "must escalate") |
| **Coverage** | 0.2 | Whether Few-shot examples cover normal, edge, and error scenarios |
| **Conciseness** | 0.1 | Whether there is redundant description, whether it exceeds necessary length |

**Scoring Rules**:
- ≥8.5: Excellent, ready for use
- 7.0-8.4: Good, use after fine-tuning
- 5.0-6.9: Needs revision, has obvious flaws
- <5.0: Needs rewriting

## 5. Soul Testing Methods

### 5.1 Sandbox Replay Test

Before a new or optimized Soul goes live, it must pass the sandbox replay test:

1. Select the Agent's last 10 historical tasks
2. Replay each task in the sandbox (using the new Soul)
3. Compare the output of the old and new Souls:
   - Task success rate (was it completed)
   - Output quality (human or LLM evaluation, 1-5 points)
   - Execution efficiency (completion time)
4. Generate a test report

**Passing Criteria**:
- Success rate not lower than the old Soul
- Average quality score not lower than the old Soul
- Efficiency not lower than 90% of the old Soul

### 5.2 A/B Testing (Optional, for Formal Agents)

1. Deploy the new Soul to 20% of traffic
2. Monitor for 3 days (or at least 30 tasks)
3. Compare metrics: success rate, quality score, efficiency
4. If the new Soul is significantly better than the old Soul → Full rollout
5. If the new Soul is significantly worse than the old Soul → Automatic rollback, notify HR
6. If the difference is not significant → Extend testing or make a manual decision

### 5.3 CEO Manual Testing (New Roles)

For entirely new Agent roles (no historical tasks to replay), the CEO or HR manually executes 3-5 test tasks and evaluates them manually.

## 6. Soul Version Management

### 6.1 Version Number Specification

Use semantic versioning: `Major.Minor.Patch`

- **Major**: Significant changes to Soul structure or core responsibilities (e.g., adding a new core responsibility)
- **Minor**: Fine-tuning of code of conduct, output format, etc.
- **Patch**: Fixing spelling errors, improving examples, etc.

### 6.2 Changelog Format

Every Soul update must be recorded:

```markdown
## [1.2.0] — 2026-06-20

### Added
- Added the "Approval Escalation Judgment Criteria" section

### Changed
- Adjusted routing decision priority: Specialized Department > Branch > GTF

### Removed
- Removed the redundant "Daily Report" requirement

### Test Results
- Sandbox replay test: 10/10 passed, average quality score 4.2→4.3
- A/B test: 3 days, success rate unchanged, efficiency improved by 12%

### Rollback Plan
- If anomalies occur, rollback to v1.1.0
```

### 6.3 Rollback Mechanism

- Old versions of Souls are permanently retained in Git history
- The HR Manager can rollback to any historical version with one click via `agents.update`
- Rollback operations are recorded in the audit log
- If performance declines after 3 consecutive updates, automatically pause Soul optimization for that Agent and notify HR for manual review

## 7. Common Role Soul Skeletons

### 7.1 Execution Agent (e.g., GTF Manager, Developer)

```
## Role Definition
You are a [Role Name]. You are [Positioning Description].

## Core Responsibilities
1. Receive tasks
2. Decompose and execute
3. Review and consolidate
4. [Other responsibilities]

## Workflow
1. Analyze task → 2. Decompose into sub-tasks → 3. Execute → 4. Summarize → 5. Review

## Code of Conduct
- Escalate when uncertain
- Report periodically on long tasks
- Must review after completion

## Tool Usage Instructions
[Tool List]

## Output Format
[Task Result Template]
[Review Report Template]

## Language Instructions
[Language Rules]

## Example Scenarios
[At least 3]
```

### 7.2 Management Agent (e.g., EA, HR Manager, Department Manager)

```
## Role Definition
You are a [Role Name]. You are responsible for [Scope of Management].

## Core Responsibilities
1. [Coordination/Management Responsibility 1]
2. [Coordination/Management Responsibility 2]
...

## Decision Process
[Decision tree or conditional judgment logic]

## Code of Conduct
- What can be decided autonomously
- What must be escalated
- When to escalate

## Tool Usage Instructions
[Tool List]

## Output Format
[Report Template]
[Approval Template]

## Language Instructions
[Language Rules]

## Example Scenarios
[At least 3, including escalation judgment examples]
```

### 7.3 Coordination Agent (e.g., PM)

```
## Role Definition
You are a project manager for [Project Name]. You are responsible for driving the project to deliver on time.

## Core Responsibilities
1. Task decomposition and assignment
2. Progress tracking
3. Risk identification and meeting organization
4. Reporting

## Workflow
[Standard project management workflow]

## Code of Conduct
- Blocked for >24h → Initiate a meeting
- Proactive risk warning
- Regular weekly report push

## Tool Usage Instructions
- OpenProject MCP: Task CRUD
- GitLab MCP: Code review
- meetings.create: Initiate meetings

## Output Format
[Weekly Report Template]
[Risk Warning Template]

## Language Instructions
[Language Rules]

## Example Scenarios
[At least 3]
```

## 8. HR Manager Automatic Soul Generation Checklist

When automatically generating or optimizing a Soul, the HR Manager must check the following list:

| Check Item | Passing Criteria |
|------------|------------------|
| Clear Role Positioning | Can explain in one sentence what this Agent does |
| No More Than 7 Responsibilities | Core responsibilities 3-7 items; if more, the role needs to be split |
| Actionable Code of Conduct | Each rule has a specific "do/don't do" |
| Few-shot Coverage | Includes at least normal, edge, and error scenarios |
| Consistent with Tool Set | Tools mentioned in the Soul are in the Agent's tools configuration |
| Matches Base Type | A code-type Agent's Soul should not require "visual design" |
| Complete Language Instructions | Includes rules for internal reasoning, interaction, and document language |
| No Internal Contradictions | No conflicting instructions between sections |

---
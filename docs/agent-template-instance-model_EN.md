# Agent Template and Instance Model Explanation

**File**: `docs/agent-template-instance-model.md`  
**Version**: v1.0  
**Purpose**: Clarify the "inheritance" relationship between Agent Templates (abstract classes) and Agent Instances (concrete subclasses), explain the design model where multiple Agent instances for the same position can be created based on different runtimes, for development AI to analyze and adjust against existing implementations.

---

## 1. Core Concepts

The Agent system in SimuCorp adopts **object-oriented inheritance thinking** for organization:

- **Agent Template** (Abstract Class)  
  Defines the common attributes and behavioral blueprint for a certain type of position. It answers "what this position needs to do," including: role name, system prompt (Soul), recommended core skills, recommended toolset, recommended runtime type, internship assessment criteria, etc. Templates themselves cannot execute tasks directly; they must be instantiated into specific Agents.

- **Agent Instance** (Concrete Subclass)  
  A **specific employee** created based on a template and bound to a specific execution environment (Runtime). Each instance has a unique `agent_id`, independent memory space, independent session state, and specific model configuration. It answers "which workstation this employee works at."

**Key Principle**:
> **Multiple Agent instances can exist for the same position. They share the same template (Soul, responsibility definition) but run on different runtimes, with independent memory and state.**

For example, the template `backend_dev` (Backend Developer) can derive:
- `backend_dev_ow` (Engineer Wang, OpenClaw native Runtime)
- `backend_dev_cc` (Engineer Li, Claude Code CLI Runtime)
- `backend_dev_cx` (Engineer Zhang, Codex CLI Runtime)

They share the same "position DNA" but have different workstations, suitable for handling different types of tasks.

---

## 2. Inheritance Model Details

### 2.1 Abstract Class: Agent Template

```json
{
  "template_id": "backend_dev",
  "name": "Backend Developer",
  "role": "Backend Developer",
  "prompt_template": "You are a senior backend developer...",  // Soul text
  "suggested_runtime": "claude-code",                          // Just a suggestion, not mandatory
  "suggested_primary_model": "claude-sonnet-4",
  "suggested_fallbacks": ["deepseek-coder"],
  "suggested_tools": ["github-mcp", "postgres-mcp"],
  "capabilities": {
    "skills": ["Python", "FastAPI", "PostgreSQL"],
    "domain": ["backend"],
    "level": "mid"
  },
  "internship_kpi": {
    "task_count": 10,
    "duration_days": 7,
    "pass_threshold": 80
  }
}
```

### 2.2 Concrete Subclass: Agent Instance

```json
{
  "agent_id": "backend_dev_cc_01",
  "name": "Engineer Li (Claude Code Backend Dev)",
  "template_id": "backend_dev",           // Inherited from which template
  "runtime_type": "claude-code",          // Specific runtime bound to the instance, immutable
  "runtime_endpoint": "http://cc-node-01:3000",
  "model_config": {
    "primary": "claude-sonnet-4",
    "fallbacks": ["deepseek-coder"]
  },
  "tools": ["github-mcp", "postgres-mcp", "docker-mcp"], // Can be adjusted based on template
  "status": "idle",
  "memory_namespace": "backend_dev_cc_01" // Isolated memory space
}
```

**Key Constraints**:
- `runtime_type` is determined at instance creation time and **cannot be changed afterward**. If a runtime change is needed, create a new Agent instance instead of modifying the existing instance's Runtime.
- Each instance's `memory_namespace` equals its `agent_id`, providing natural isolation without special handling.

---

## 3. Comparison with Existing Design

| Concept in Existing Design | Corresponding in Inheritance Model | Description |
|:---|:---|:---|
| `agent_templates` table | Abstract class | Stores generic position definitions |
| `agents` table | Concrete subclass instances | Each instance bound to Runtime, with unique ID |
| Template's `suggested_runtime` | Recommended attribute in abstract class | Can be overridden by instance |
| Instance's `runtime_type` | Concrete implementation attribute in subclass | Determined at creation, immutable |
| Instance's `agent_id` / `name` | Unique identifier for subclass instance | `name` can be given human-readable alias |
| Memory isolation by `agent_id` | Independent memory per subclass instance | No cross-instance memory sharing |
| HR creating Agent | Instantiating concrete subclass from abstract class | One template can instantiate multiple Agents on different runtimes |
| EA routing | Matching by instance capabilities + Runtime type | Instances with same position but different runtimes can be routed distinctly |

**No major changes needed to existing design**, only need to clarify the "one template, multiple instances" concept in the following modules:

1. **HR Manager Recruitment Process**: Allow creating multiple instances with different runtimes for the same template.
2. **EA Routing Logic**: When multiple instances exist with the same skill tags, select the most suitable instance based on task execution environment requirements (code execution, file operations, hardware resources).
3. **Organization Structure Display**: Different runtime instances of the same position appear as independent nodes in the topology, distinguished by name.

---

## 4. Concrete Implementation in the System

### 4.1 HR Manager Instance Creation Behavior Upgrade

Original design: HR receives recruitment requirement → recommends one runtime → creates one instance.  
Upgraded: HR can create **multiple instances with different runtimes** at once based on position requirement complexity. For example, for a backend development position, simultaneously create:
- One OpenClaw native instance (for daily lightweight tasks)
- One Claude Code instance (for heavy code development)

### 4.2 EA Routing with Runtime Awareness

When EA finds multiple Agents with the same `skills` tags, it further compares their `runtime_type` and current load:

- Tasks involving heavy code writing, file operations → Prioritize Claude Code or Codex instances
- Tasks requiring only data analysis, report generation → Prioritize OpenClaw native or GPT-5 instances
- If multiple instances are suitable → Select the one with lowest load

### 4.3 Memory and State

- Each instance has an independent memory namespace. Even if derived from the same template, their learned experiences are not automatically shared.
- For cross-instance experience sharing, extract common knowledge into the template's "common memory package" or authorize through a shared memory whitelist.

---

## 5. Example Scenario

**Scenario**: SimuCorp needs a backend development position.

1. HR Manager selects the `backend_dev` template from the marketplace or local template library.
2. Assess task types: daily lightweight coordination + heavy code development.
3. Create two instances:
   - `backend_dev_ow` (Xiao Wang, OpenClaw native, model GPT-5, basic query toolset)
   - `backend_dev_cc` (Old Li, Claude Code CLI, model Claude Sonnet 4, complete development toolchain)
4. CEO assigns task "Write a user authentication module for me" → EA analyzes need for code execution → Routes to `backend_dev_cc`.
5. CEO assigns task "Check how many active users are in the database" → EA analyzes only query needed → Routes to `backend_dev_ow`.
6. Both instances accumulate memories independently, growing without interference.

---

## 6. Requirements for Development AI

Development AI must follow these constraints during implementation:

- The `runtime_type` field of Agent instances **cannot be modified** after creation.
- HR Manager should support batch creation of instances with different runtimes based on the same template.
- EA's routing algorithm needs to add "runtime suitability" weight (code tasks prioritize Claude Code, lightweight tasks prioritize OpenClaw native).
- In the organization topology view, multiple instances derived from the same template should appear as independent nodes, distinguished by name.
- The memory system isolates by `agent_id`, no additional logic needed for "cross-runtime memory merging."

---

This model makes the previously implicit "template-instance relationship" explicit as an object-oriented inheritance structure, making the system clearer and more stable while maintaining the integrity of the original design. Development AI can use this to compare against existing implementations and adjust modules such as HR Manager, EA routing, and organization display.
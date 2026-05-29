# Agent 模板与实例模型说明

**文件**：`docs/agent-template-instance-model.md`  
**版本**：v1.0  
**目的**：明确 Agent 模板（抽象类）与 Agent 实例（具体子类）的“继承”关系，阐明同一岗位可基于不同基座创建多个 Agent 实例的设计模型，供开发 AI 对照现有实现进行分析和调整。

---

## 1. 核心概念

模拟公司 (SimuCorp) 中的 Agent 体系采用**面向对象的继承思想**来组织：

- **Agent 模板**（抽象类）  
  定义了某一类岗位的通用属性和行为蓝图。它回答了“这个岗位要做什么”，包含：角色名称、系统提示词（Soul）、建议的核心技能、建议的工具集、建议的基座类型、实习考核标准等。模板本身不能直接执行任务，必须实例化为具体的 Agent。

- **Agent 实例**（具体子类）  
  基于某个模板创建的、绑定特定执行环境（基座 Runtime）的**具体员工**。每个实例拥有唯一的 `agent_id`、独立的记忆空间、独立的会话状态、具体的模型配置。它回答了“这个员工在哪个工位上工作”。

**关键原则**：
> **同一岗位可以存在多个 Agent 实例，它们共享相同的模板（Soul、职责定义），但运行在不同的基座上，拥有独立的记忆和状态。**

例如，模板 `backend_dev`（后台开发工程师）可以派生出：
- `backend_dev_ow`（王工，OpenClaw 原生 Runtime）
- `backend_dev_cc`（李工，Claude Code CLI Runtime）
- `backend_dev_cx`（张工，Codex CLI Runtime）

他们拥有相同的“岗位 DNA”，但工位不同，适合处理不同类型的任务。

---

## 2. 继承模型详解

### 2.1 抽象类：Agent 模板

```json
{
  "template_id": "backend_dev",
  "name": "后台开发工程师",
  "role": "Backend Developer",
  "prompt_template": "你是一个资深后台开发工程师...",  // Soul 文本
  "suggested_runtime": "claude-code",                // 只是建议，非强制
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

### 2.2 具体子类：Agent 实例

```json
{
  "agent_id": "backend_dev_cc_01",
  "name": "李工（Claude Code 后台开发）",
  "template_id": "backend_dev",          // 继承自哪个模板
  "runtime_type": "claude-code",         // 实例绑定的具体基座，不可变
  "runtime_endpoint": "http://cc-node-01:3000",
  "model_config": {
    "primary": "claude-sonnet-4",
    "fallbacks": ["deepseek-coder"]
  },
  "tools": ["github-mcp", "postgres-mcp", "docker-mcp"], // 可基于模板调整
  "status": "idle",
  "memory_namespace": "backend_dev_cc_01" // 隔离的记忆空间
}
```

**关键约束**：
- `runtime_type` 在实例创建时确定，**之后不可更改**。若需更换基座，应创建一个新的 Agent 实例，而非修改现有实例的 Runtime。
- 每个实例的 `memory_namespace` 等于其 `agent_id`，天然隔离，无需特殊处理。

---

## 3. 与现有设计的对照

| 现有设计中的概念 | 继承模型中的对应 | 说明 |
|:---|:---|:---|
| `agent_templates` 表 | 抽象类 | 存储岗位的通用定义 |
| `agents` 表 | 具体子类的实例 | 每个实例绑定 Runtime，有独立 ID |
| 模板的 `suggested_runtime` | 抽象类中的推荐属性 | 可被实例覆盖 |
| 实例的 `runtime_type` | 子类的具体实现属性 | 创建时确定，不可变 |
| 实例的 `agent_id` / `name` | 子类实例的唯一标识 | `name` 可赋予人类可读的代称 |
| 按 `agent_id` 隔离记忆 | 每个子类实例独立内存 | 跨实例记忆不共享 |
| HR 创建 Agent | 基于抽象类实例化具体子类 | 一个模板可实例化多个不同基座的 Agent |
| EA 路由 | 按实例能力 + Runtime 类型匹配 | 同岗位不同基座的实例可被区分路由 |

**现有设计无需大改**，只需在以下模块中明确“一模板多实例”的概念：

1. **HR Manager 招聘流程**：允许为同一模板创建多个不同基座的实例。
2. **EA 路由逻辑**：当同一技能标签存在多个实例时，根据任务对执行环境的需求（代码执行、文件操作、硬件资源）选择最合适的实例。
3. **组织架构展示**：同一岗位的不同基座实例在拓扑图中显示为独立节点，用名称区分。

---

## 4. 在系统中的具体体现

### 4.1 HR Manager 创建实例的行为升级

原设计：HR 收到招聘需求 → 推荐一个基座 → 创建一个实例。  
升级后：HR 可根据岗位需求复杂度，一次性创建**多个不同基座的实例**。例如，为后台开发岗位同时创建：
- 一个 OpenClaw 原生实例（用于日常轻量任务）
- 一个 Claude Code 实例（用于重型代码开发）

### 4.2 EA 路由的基座感知

当 EA 检索到多个 Agent 拥有相同的 `skills` 标签时，进一步比较它们的 `runtime_type` 和当前负载：

- 任务包含大量代码编写、文件操作 → 优先 Claude Code 或 Codex 实例
- 任务仅需数据分析、报告生成 → 优先 OpenClaw 原生或 GPT-5 实例
- 若多个实例均适合 → 选择负载最低的那个

### 4.3 记忆与状态

- 每个实例拥有独立的记忆命名空间，即使派生自同一模板，它们学到的经验也不自动共享。
- 如需跨实例经验共享，应将通用知识提炼后存入模板的“通用记忆包”，或通过共享记忆白名单授权。

---

## 5. 示例场景

**场景**：模拟公司需要一个后台开发岗位。

1. HR Manager 从市场或本地模板库选取 `backend_dev` 模板。
2. 评估任务类型：日常轻量协调 + 重型代码开发。
3. 创建两个实例：
   - `backend_dev_ow`（小王，OpenClaw 原生，模型 GPT-5，工具集为基础查询工具）
   - `backend_dev_cc`（老李，Claude Code CLI，模型 Claude Sonnet 4，工具集含完整开发工具链）
4. CEO 下达任务“帮我写一个用户认证模块” → EA 分析需要代码执行 → 路由给 `backend_dev_cc`。
5. CEO 下达任务“查一下数据库中有多少活跃用户” → EA 分析只需查询 → 路由给 `backend_dev_ow`。
6. 两个实例各自积累记忆，独立成长，互不干扰。

---

## 6. 对开发 AI 的要求

开发 AI 在实现时需遵循以下约束：

- Agent 实例的 `runtime_type` 字段在创建后**不可修改**。
- HR Manager 应支持基于同一模板批量创建不同基座的实例。
- EA 的路由算法需增加“基座适配”权重（代码任务优先 Claude Code，轻量任务优先 OpenClaw 原生）。
- 组织拓扑视图中，同一模板派生的多个实例应显示为独立节点，名称区分。
- 记忆系统按 `agent_id` 隔离，无需为“跨基座记忆合并”增加额外逻辑。

---

此模型将原本隐含的“模板与实例关系”显式化为面向对象的继承结构，使系统更清晰、更稳定，同时保持了原有设计的完整性。开发 AI 可据此对比现有实现，调整 HR Manager、EA 路由、组织展示等模块。
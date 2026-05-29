# 模拟公司 (SimuCorp) — 自进化多Agent协作系统

**公司即代码，组织即智能。** 让AI Agent像公司员工一样分工协作、自我生长。

> ⚠️ **项目状态**: 已完成完整设计文档（18章）和项目白皮书，Phase 1 核心链路在积极开发中。欢迎贡献！

---

## 这是什么？

SimuCorp 是一个以**公司治理结构**来管理AI Agent的操作系统级框架。不同于传统的"临时剧组"式多Agent框架，SimuCorp 模拟了一家真实公司的运作：

- 🏢 **总裁助理(EA)** —— 接收任务、路由调度、全局状态管理
- 👥 **人事部(HR)** —— 自动招聘、实习考核、转正晋升、技能匹配
- 🔧 **机动部(GTF)** —— 兜底执行并触发技能沉淀，填补人力缺口
- 📋 **L0–L3审批分级** —— 保证CEO对重大决策的最终控制权
- 🌐 **分公司机制** —— 允许其他机器作为"分公司"接入，支持异地部署
- 📈 **自进化闭环** —— 从执行到复盘到招聘到晋升，全自动循环

## 快速开始

```bash
# 1. 克隆仓库
git clone https://github.com/without1022/simu-corp.git
cd simu-corp

# 2. 阅读设计文档（必读！）
open docs/多agent协作系统设计.md

# 3. 了解设计哲学
open docs/模拟公司白皮书.md

# 4. 查看当前开发状态
open ROADMAP.md
```

## 为什么选择 SimuCorp？

| 现有框架 | 我们的不同 |
|----------|------------|
| CrewAI / AutoGen: 临时角色 | SimuCorp: **长期雇佣关系**，有实习、转正、晋升机制 |
| MetaGPT: 固定流水线 | SimuCorp: **自我进化**，自动发现技能缺口并招聘 |
| 三省六部: 静态部门 | SimuCorp: **动态组织**，可审批新建部门、分公司 |
| OMC（学术）: 纯概念验证 | SimuCorp: **完整工程设计**，18章文档 + 可部署蓝图 |

## 体系架构

```
                           ┌─────────────┐
                           │   CEO / 用户  │
                           └──────┬──────┘
                                  │ 任务输入
                          ┌───────▼────────┐
                          │  总裁助理 (EA)   │
                          │ 路由/分派/状态管理 │
                          └───┬────┬────┬───┘
                              │    │    │
                    ┌─────────┘    │    └─────────┐
                    ▼              ▼              ▼
            ┌───────────┐  ┌───────────┐  ┌───────────┐
            │ 人事部(HR) │  │ 机动部(GTF)│  │ 业务部门  │
            │ 招聘/考核  │  │ 兜底/沉淀  │  │ 专业Agent │
            └───────────┘  └───────────┘  └───────────┘
                    │              │              │
                    └──────────────┴──────────────┘
                                   │
                           ┌───────▼────────┐
                           │   MCP 工具市场   │
                           │  外部服务/API   │
                           └────────────────┘
```

## 项目结构

```
simu-corp/
├── README.md                 # 项目概览
├── LICENSE                   # MIT 协议
├── CONTRIBUTING.md           # 贡献指南
├── ROADMAP.md                # 路线图
├── CLAUDE.md                 # AI 辅助开发上下文
├── docs/                     # 设计文档
│   ├── 多Agent协作系统设计.md   # 18章完整设计 (7649行)
│   ├── 模拟公司白皮书.md        # 项目白皮书 (设计哲学)
│   ├── AI-OS-演化白皮书.md       # AI OS 演化白皮书 (应用→操作系统)
│   ├── agent-template-instance-model.md  # Agent模板与实例模型
│   ├── agent-marketplace.md   # Agent市场(人才市场)设计
│   ├── api-mock.md            # 接口Mock数据与联调指南
│   ├── test-scenarios.md      # 测试场景与验收标准
│   ├── prompt-engineering-guide.md  # Agent Prompt工程指南
│   ├── mcp-server-template.md      # MCP Server开发模板
│   ├── template-management.md      # Agent模板库管理规范
│   ├── operations.md               # 运维手册
│   ├── audit-report.md             # 开发状态审计报告
│   ├── design-review-report.md     # 设计审核报告
│   └── template-market-alignment.md # 模板-实例&Agent市场对齐报告
├── prompts/                  # Agent Soul 提示词
│   ├── ea.md                 # 总裁助理 (完整提示词)
│   ├── hr_manager.md         # 人事部经理
│   └── gtf_manager.md        # 机动部经理
└── development/
    └── DEVELOPMENT.md        # 开发Agent协作规范
```

## 寻求贡献者

我们正在寻找以下领域的贡献者：

- 🖥️ **前端开发** (React + TypeScript + Tailwind)
- ⚙️ **后端/网关开发** (Node.js, WebSocket, REST API)
- 🧩 **MCP Server 开发** (工具集成与扩展)
- 🤖 **Agent 提示词工程** (编写和优化 Agent Soul)
- 📝 **文档与测试**

请阅读 [CONTRIBUTING.md](CONTRIBUTING.md) 了解如何参与。

## 许可证

MIT License — 详见 [LICENSE](LICENSE) 文件。

---

**模拟公司 · SimuCorp** — 让 Agent 不再临时搭伙，而是真正在一起"上班"。

# 文档一：开发Agent协作规范

**文件**：`DEVELOPMENT.md`  
**状态**：立即需要

---

# 模拟公司 (SimuCorp) — 开发Agent协作规范

## 1. 概述

本文档定义了多个开发Agent（前端Agent、后端Agent、提示词Agent、测试Agent）在并行开发模拟公司 (SimuCorp) 时的协作规则。所有开发Agent必须遵守本规范。

## 2. 开发Agent角色分工

| Agent角色 | 负责模块 | 依赖 |
|-----------|---------|------|
| **基础设施Agent** | docker-compose.yml、openclaw.json、init-db.sql、mcp.json | 无 |
| **后端Agent** | Gateway RPC实现、REST API、Runtime Manager | 基础设施Agent完成 |
| **提示词Agent** | prompts/ea.md、prompts/hr_manager.md、prompts/gtf_manager.md | 设计文档第3章 |
| **前端Agent** | React SPA所有页面和组件 | 后端Agent的接口定义 |
| **MCP Agent** | Memory MCP Server、OpenProject MCP、GitHub MCP等 | 基础设施Agent完成 |

## 3. 项目目录结构

```
simu-corp/
├── CLAUDE.md                          # 项目总览
├── DEVELOPMENT.md                     # 本文档
├── 多agent协作系统设计.md              # 18章设计文档
├── 模拟公司白皮书.md                   # 项目白皮书
├── docker-compose.yml                 # 基础设施Agent负责
├── docker-compose.branch.yml          # 基础设施Agent负责
├── .env.example                       # 基础设施Agent负责
├── init-db.sql                        # 基础设施Agent负责
├── openclaw.json                      # 基础设施Agent负责（初始版本）
├── mcp.json                           # 基础设施Agent负责
├── prompts/                           # 提示词Agent负责
│   ├── ea.md
│   ├── hr_manager.md
│   └── gtf_manager.md
├── gateway/                           # 后端Agent负责
│   ├── src/
│   │   ├── routes/                    # REST API路由
│   │   ├── rpc/                       # WebSocket RPC方法
│   │   ├── runtime/                   # Runtime Manager
│   │   └── middleware/                # 认证、日志等中间件
│   └── tests/
├── mcp-servers/                       # MCP Agent负责
│   ├── memory-mcp/
│   ├── openproject-mcp/
│   └── github-mcp/
├── frontend/                          # 前端Agent负责
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
├── scripts/                           # 基础设施Agent负责
│   ├── init.js
│   └── branch-init.js
└── docs/                              # 补充文档
    ├── api-mock.md                    # 接口Mock数据
    ├── test-scenarios.md              # 测试场景
    └── prompt-engineering-guide.md    # 提示词工程指南
```

## 4. Git分支策略

```
main                      # 主分支，只接受合并，不直接提交
├── infra/setup           # 基础设施Agent：Docker、配置文件
├── backend/api           # 后端Agent：REST API + WebSocket RPC
├── backend/runtime       # 后端Agent：Runtime Manager
├── prompts/core          # 提示词Agent：三个核心Agent的Soul
├── frontend/mvp          # 前端Agent：Phase 1 MVP页面
├── frontend/full         # 前端Agent：Phase 2-3 完整页面
├── mcp/memory            # MCP Agent：Memory MCP Server
├── mcp/openproject       # MCP Agent：OpenProject MCP
└── mcp/github            # MCP Agent：GitHub MCP
```

**规则**：
- 每个开发Agent在自己的分支上工作
- 完成一个模块后，向 `main` 发起合并请求
- 合并前必须确保自己的模块能独立运行（通过单元测试）
- 合并后通知下游Agent更新依赖

## 5. 提交信息规范

```
<type>(<scope>): <subject>

type:
  feat     — 新功能
  fix      — 修复Bug
  docs     — 文档更新
  refactor — 重构
  test     — 测试
  chore    — 构建/工具变更

scope:
  infra    — 基础设施
  backend  — 后端
  frontend — 前端
  prompts  — 提示词
  mcp      — MCP Server

示例：
  feat(backend): 实现 agents.list RPC方法
  fix(frontend): 修复 Dashboard 统计卡片刷新延迟
  docs(prompts): 补全 EA 的 Few-shot 示例
  test(backend): 添加任务超时重试的端到端测试
```

## 6. 接口Mock约定

**原则**：前端Agent不应等待后端Agent完成所有接口。使用Mock数据先行开发UI。

**Mock模式切换**：
- 前端 `.env.development` 中设置 `VITE_USE_MOCK=true` 启用Mock
- Mock数据文件位于 `frontend/src/api/__mocks__/`
- 每个API接口对应一个Mock文件

**接口契约**：
- 后端Agent在实现接口前，先在 `docs/api-mock.md` 中给出Mock响应示例
- 前端Agent基于Mock响应开发
- 后端接口完成后，前端切换 `VITE_USE_MOCK=false` 进行联调

## 7. 模块完成通知（握手协议）

当一个开发Agent完成某个模块时，按以下格式通知下游Agent：

```markdown
### [模块完成通知]

**模块名称**：agents.list RPC方法
**完成时间**：2026-06-20 14:00
**分支**：backend/api
**变更摘要**：实现了 agents.list RPC方法，支持按status/department/skill筛选
**接口文档**：见设计文档第10章 10.2.4 节
**Mock数据**：已更新 docs/api-mock.md
**测试状态**：单元测试通过，等待联调
**已知问题**：分页参数page_size上限暂未限制
**下游影响**：前端Agent现在可以使用该接口获取Agent列表
```

## 8. 代码规范

**TypeScript/JavaScript**：
- 使用 ESLint + Prettier，配置文件在项目根目录
- 函数必须有JSDoc注释（至少描述参数和返回值）
- 异步操作统一使用 async/await

**React组件**：
- 每个组件一个文件
- 组件Props必须有TypeScript类型定义
- 使用Shadcn/ui组件，不自行造轮子

**提示词文件**：
- Markdown格式
- 使用代码块标记结构化输出格式
- Few-shot示例使用 `### 示例N` 分隔

## 9. 测试规范

- **单元测试**：每个模块必须覆盖核心逻辑（目标覆盖率>80%）
- **集成测试**：跨模块交互必须有集成测试（如EA路由→GTF执行的完整链路）
- **端到端测试**：Phase 1完成后，编写完整用户场景测试

测试文件位置：
- 后端：`gateway/tests/`
- 前端：`frontend/src/__tests__/`
- MCP：`mcp-servers/<name>/tests/`

## 10. 问题升级机制

开发Agent遇到无法自行解决的问题时，按以下优先级寻求帮助：

1. **查阅设计文档**：在 `多agent协作系统设计.md` 中搜索相关章节
2. **查阅白皮书**：在 `模拟公司白皮书.md` 中理解设计意图
3. **查阅本文档**：检查是否已有约定
4. **向CEO提问**：明确描述问题、已尝试的方案、需要什么帮助



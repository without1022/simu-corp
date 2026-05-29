# 文档九：贡献指南

**文件**：`CONTRIBUTING.md`  
**状态**：开源前需要

---

# 模拟公司 (SimuCorp) — 贡献指南

## 1. 欢迎！

感谢你对模拟公司 (SimuCorp) 的关注！模拟公司 (SimuCorp) 是一个“公司即代码”的AI Agent组织系统，我们欢迎任何形式的贡献——代码、文档、模板、MCP Server、Bug报告、功能建议。

## 2. 行为准则

本项目采用贡献者公约（Contributor Covenant）。我们希望所有参与者：
- 使用友好、包容的语言
- 尊重不同观点和经验
- 建设性地接受批评
- 关注对社区最有利的事情

## 3. 如何贡献

### 3.1 报告Bug

在GitHub Issues中创建Bug报告，包含：
- 模拟公司 (SimuCorp) 版本号
- 操作系统和Docker版本
- 复现步骤
- 预期行为 vs 实际行为
- 相关日志（如有）

### 3.2 提出功能建议

在GitHub Issues中创建功能建议，包含：
- 功能描述
- 使用场景
- 你认为应该如何实现
- 是否愿意自己实现（如果会的话）

### 3.3 贡献代码

**开发流程**：

1. Fork 本仓库
2. 创建你的功能分支：`git checkout -b feature/amazing-feature`
3. 确保你的代码符合项目规范（见第4节）
4. 添加测试（如适用）
5. 提交你的更改：`git commit -m 'feat: add amazing feature'`
6. 推送到分支：`git push origin feature/amazing-feature`
7. 创建Pull Request到 `main` 分支

**PR规范**：
- 一个PR只解决一个问题
- PR标题格式：`<type>(<scope>): <description>`（同Commit规范）
- PR描述中说明：做了什么、为什么这样做、测试情况

### 3.4 贡献Agent模板

如果你有某个岗位的优秀Agent Soul，欢迎贡献到模板库：

1. 按照 `docs/prompt-engineering-guide.md` 编写Soul
2. 按照 `docs/template-management.md` 中的模板结构定义完整模板
3. 在沙盒中测试（实例化→执行3个任务→评估）
4. 将模板JSON文件放在 `data/templates/` 目录
5. 创建PR

### 3.5 贡献MCP Server

如果你为某个外部工具编写了MCP Server适配器：

1. 按照 `docs/mcp-server-template.md` 编写
2. 放在 `mcp-servers/<your-server>/` 目录
3. 包含README（功能说明、环境变量、使用示例）
4. 包含单元测试
5. 创建PR

### 3.6 贡献翻译

如果你希望将前端翻译成新的语言：

1. 在 `frontend/src/i18n/locales/` 下创建新的语言目录（如 `ja-JP/`）
2. 复制 `en-US/` 中的所有JSON文件
3. 逐文件翻译
4. 在 `i18n/index.ts` 中注册新语言
5. 创建PR

## 4. 代码规范

### 4.1 通用规范

- 使用 `eslint` 和 `prettier`（配置文件在项目根目录）
- 提交前运行 `npm run lint` 确保无错误
- 函数必须有JSDoc注释
- 异步操作使用 `async/await`

### 4.2 Commit规范

```
<type>(<scope>): <subject>

type: feat | fix | docs | refactor | test | chore
scope: infra | backend | frontend | prompts | mcp | docs

示例:
  feat(backend): 实现 agents.list RPC方法
  fix(frontend): 修复Dashboard统计卡片刷新延迟
  docs(prompts): 补全EA的Few-shot示例
```

### 4.3 测试规范

- 新功能必须包含单元测试
- 核心模块测试覆盖率 > 80%
- 提交前运行 `npm test` 确保全部通过

## 5. 项目架构速览

如果你刚接触模拟公司 (SimuCorp)，建议按以下顺序了解项目：

1. 阅读 `模拟公司白皮书.md`（理解设计哲学）
2. 阅读 `多agent协作系统设计.md` 第1-3章（理解系统架构）
3. 阅读 `DEVELOPMENT.md`（理解开发规范）
4. 阅读 `docs/prompt-engineering-guide.md`（如果要贡献Agent模板）

**关键目录**：
- `gateway/` — 后端API和RPC实现
- `frontend/` — React前端
- `mcp-servers/` — MCP Server集合
- `prompts/` — 核心Agent的Soul文件
- `docs/` — 补充文档
- `scripts/` — 运维脚本

## 6. 开发环境搭建

```bash
# 1. 克隆仓库
git clone <repo> && cd simu-corp

# 2. 复制环境变量
cp .env.example .env

# 3. 启动基础设施
docker-compose up -d postgres redis chromadb

# 4. 安装依赖
cd gateway && npm install
cd ../frontend && npm install

# 5. 启动开发模式
# 终端1: Gateway
cd gateway && npm run dev

# 终端2: 前端
cd frontend && npm run dev

# 6. 验证
curl http://localhost:18789/api/v1/system/health
```

## 7. 获取帮助

- 查阅设计文档和补充文档（`docs/` 目录）
- 在GitHub Issues中提问
- 通过WebChat与EA对话（如果你已经部署了模拟公司 (SimuCorp)）

---

## 9份补充文档完成总结

到目前为止，已输出的全部补充文档：

| 序号 | 文件 | 状态 |
|------|------|------|
| 1 | `DEVELOPMENT.md` — 开发Agent协作规范 | ✅ |
| 2 | `docs/api-mock.md` — 接口Mock数据与联调指南 | ✅ |
| 3 | `prompts/ea.md` — 总裁助理完整提示词 | ✅ |
| 4 | `docs/test-scenarios.md` — 测试场景与验收标准 | ✅ |
| 5 | `docs/prompt-engineering-guide.md` — Agent Prompt工程指南 | ✅ |
| 6 | `docs/mcp-server-template.md` — MCP Server开发模板 | ✅ |
| 7 | `docs/template-management.md` — Agent模板库管理规范 | ✅ |
| 8 | `docs/operations.md` — 运维手册 | ✅ |
| 9 | `CONTRIBUTING.md` — 贡献指南 | ✅ |

加上之前输出的 `模拟公司白皮书.md` 和 `多agent协作系统设计.md`（18章），项目文档体系已完整覆盖从**愿景**→**设计**→**开发**→**测试**→**部署**→**运维**→**社区贡献**的全生命周期。开发Agent可以将这些文档作为完整的知识库，按需查阅。

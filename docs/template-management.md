# 文档七：Agent 模板库管理规范

**文件**：`docs/template-management.md`  
**状态**：Phase 2 前需要

---

# 模拟公司 (SimuCorp) — Agent 模板库管理规范

## 1. 概述

本文档定义了 Agent 模板库的管理规则。模板库是 HR Manager 快速招聘新 Agent 的基础——它存储了各种岗位的预设 Soul、工具集、基座建议和考核标准。模板库的质量直接影响招聘效率和 Agent 质量。

## 2. 模板结构定义

每个模板必须包含以下字段：

```json
{
  "template_id": "唯一标识，小写+下划线",
  "name": "模板显示名称",
  "role": "角色名称",
  "description": "岗位描述，说明该角色的职责和定位",
  "version": "语义化版本号",
  "status": "draft | testing | active | needs_optimization | deprecated",
  "author": "创建者（hr_manager 或 ceo）",
  
  "prompt_template": "完整的 Soul（系统提示词），符合 prompt-engineering-guide.md 规范",
  
  "suggested_runtime": "建议的基座类型（openclaw-native | claude-code | codex-cli）",
  "suggested_primary_model": "建议的主模型",
  "suggested_fallbacks": ["fallback模型列表"],
  
  "suggested_tools": ["工具名称列表"],
  
  "capabilities": {
    "skills": ["技能列表"],
    "domain": ["领域列表"],
    "level": "junior | mid | senior | principal"
  },
  
  "internship_kpi": {
    "task_count": 10,
    "duration_days": 7,
    "pass_threshold": 80,
    "weights": {
      "success_rate": 0.4,
      "quality": 0.3,
      "efficiency": 0.2,
      "collaboration": 0.1
    }
  },
  
  "usage_count": 0,
  "success_rate": 0.0,
  "created_at": "ISO8601",
  "updated_at": "ISO8601"
}
```

## 3. 模板生命周期

```
[草稿] → [测试中] → [已发布] → [需优化] → [已废弃]
```

### 3.1 草稿（Draft）
- 新建模板的初始状态
- 可以随时编辑
- 不会被 HR Manager 用于招聘

### 3.2 测试中（Testing）
- HR Manager 正在沙盒中验证模板效果
- 实例化一个临时 Agent，执行 3 个测试任务
- 测试通过后自动进入“已发布”

### 3.3 已发布（Active）
- 可用于正式招聘
- 编辑后需要重新测试

### 3.4 需优化（Needs Optimization）
- 自动触发：该模板招聘的 Agent 转正成功率 < 60%
- 或该模板招聘的 Agent 平均绩效低于同岗位基线 20%
- HR Manager 收到通知，进行优化

### 3.5 已废弃（Deprecated）
- 不再使用，但保留历史数据
- 已使用该模板招聘的 Agent 不受影响

## 4. 模板创建流程

### 4.1 HR Manager 自动创建
1. 收到 `TALENT_REQUEST` 后，搜索外部市场 JD
2. 在模板库中搜索相似模板（语义匹配）
3. 若找到相似度 > 80% 的模板，微调后使用
4. 若没有，创建新模板（状态：草稿）
5. 实例化临时 Agent，执行沙盒测试
6. 测试通过 → 状态变更为“已发布”
7. 测试不通过 → 调整后重试，或标记为“需人工审查”

### 4.2 CEO 手动创建
1. 在“系统设置 → 模板库”点击“新建模板”
2. 填写模板信息
3. 可直接发布，或让 HR Manager 进行沙盒测试

## 5. 模板质量评估

| 指标 | 计算方式 | 预警阈值 |
|------|---------|---------|
| 招聘成功率 | 该模板招聘的 Agent 转正率 | < 60% |
| Agent 平均绩效 | 该模板招聘的 Agent 近30天绩效均值 | < 同岗位基线 80% |
| 使用率 | 近90天使用次数 | < 2次（可能过时） |
| Soul 质量评分 | 按 prompt-engineering-guide.md 评分 | < 7.0 |

## 6. 模板版本管理

- 使用语义化版本号：`主版本.次版本.修订版本`
- 每次编辑后版本号递增
- 旧版本保留在数据库中，可回滚
- 变更日志格式：

```markdown
## [1.1.0] — 2026-06-20
### 修改
- 调整了 Soul 中的行为准则
- 增加了 Xcode 工具到建议工具集
### 测试结果
- 沙盒测试：3/3 通过
```

## 7. 模板库目录结构（文件系统备份）

除了数据库存储，模板库定期备份为 JSON 文件：

```
data/templates/
├── index.json              # 模板索引
├── backend_dev.json
├── frontend_dev.json
├── data_analyst.json
├── devops_eng.json
├── security_auditor.json
├── pm.json
└── ...
```

## 8. 模板库管理前端页面

### 8.1 模板列表页
- 卡片网格布局，每张卡片显示：
  - 模板名称、角色、状态徽章
  - 核心技能标签
  - 使用次数、成功率
- 支持按状态筛选、按技能搜索
- 点击卡片进入详情

### 8.2 模板详情页
- 完整 Soul 预览（只读）
- 建议工具集列表
- 建议基座和模型
- 使用统计（招聘次数、成功率、绩效趋势图）
- 操作按钮：编辑、测试、实例化、废弃

## 9. 开发Checklist

| 检查项 | 通过标准 |
|--------|---------|
| 模板结构完整 | 包含所有必填字段 |
| Soul 符合规范 | 通过 prompt-engineering-guide.md 质量评估（≥7.0） |
| 沙盒测试通过 | 3/3 测试任务通过 |
| 基座选择合理 | 代码型→Claude Code，协调型→OpenClaw原生 |
| 工具集与Soul一致 | Soul 中提到的工具都在工具集列表中 |
| 版本号正确 | 新建模板从 1.0.0 开始 |
| 变更日志记录 | 每次编辑有对应的变更说明 |

---

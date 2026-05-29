# 模板-实例模型 & Agent市场 对齐报告

## 一、Agent模板与实例模型 (agent模板与实例模型说明.md)

### 已对齐
| 设计要求 | 实现状态 |
|---------|---------|
| 模板(template_id) → 实例(agent_id) 继承关系 | ✅ agents表有template_id字段, HR创建时关联 |
| runtime_type创建后不可变 | ✅ registerAgent锁定runtime |
| HR支持基于模板创建实例 | ✅ HR从模板库匹配+自定义创建 |
| EA基座感知路由(代码→Claude Code, 协调→OpenClaw) | ✅ capabilityDirectory新增runtimeBonus |
| 记忆按agent_id隔离 | ✅ Memory MCP按agent_id分collection |

### 尚未实现
| 设计要求 | 说明 |
|---------|------|
| HR单次招聘创建多个不同基座实例 | 当前仅创建1个实例, 改为可选创建2-3个(不同runtime) |
| 组织拓扑中同模板多实例独立节点 | 前端org页面已支持, 只需实例确实存在 |

## 二、Agent市场设计 (agent市场设计.md)

Agent市场是跨GitHub的开源生态, 与当前单机实例不同。分两阶段对齐:

### 当前可对齐(P4-1)
| 设计要求 | 实现方案 |
|---------|---------|
| 模板包结构标准化 | 现有agent_templates表已含suggested_runtime/model/tools/ capabilities/internship_kpi, 补全dependencies和market_metadata字段 |
| 模板来源追踪(provenance) | agents表新增provenance字段(source_agent_id, tasks_completed, success_rate) |
| 模板版本语义化 | souls表已有versionHistory, 格式化为semver |
| 上传前置条件检查 | HR增加前置校验(转正率≥70% 或 任务≥50 且 成功率≥85%) |
| 安全审核(本地预检) | 上传前AI扫描Soul+记忆中的敏感信息 |

### 需要独立项目(P4-2+)
| 设计要求 | 说明 |
|---------|------|
| GitHub市场仓库 | 独立的 marketplace 仓库, 模板包JSON文件 |
| 市场API | 独立的搜索/下载/上传/评价 REST API |
| 公告墙 + 人才缺口分析 | 市场数据分析服务 |
| 评价体系 + 排名算法 | 评分/下载量/认证权重 |
| 跨实例模板同步(关注+更新通知) | 需要Gateway间通信 |

## 三、建议实施顺序

### P4-1 (立即, 1-2h)
1. 补全agent_templates表字段(dependencies/min_simu_corp_version)
2. HR上传前置检查 + 安全扫描
3. 模板provenance追踪

### P4-2 (独立项目, 需要GitHub repo)
4. 创建 simu-corp/marketplace 仓库
5. 市场API (搜索/下载/上传/评价)
6. 公告墙 + 健康度dashboard
7. HR集成市场搜索

## 四、对当前系统的改动

本次已完成:
- ✅ EA路由基座感知 (d5501f6)
- ✅ 模板ID关联agent实例

后续立即待完成:
- ⬜ HR上传前置校验
- ⬜ 模板provenance追踪字段
- ⬜ 安全扫描逻辑(敏感信息检测)

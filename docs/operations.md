# 文档八：运维手册

**文件**：`docs/operations.md`  
**状态**：Phase 3 前后需要

---

# 模拟公司 (SimuCorp) — 运维手册

## 1. 概述

本文档定义了模拟公司 (SimuCorp) 系统的日常运维操作指南。适用于系统管理员（或承担运维职责的Agent）和CEO。

## 2. 系统架构速览

```
simu-corp/
├── gateway        # OpenClaw Gateway（核心，必须保障）
├── postgres       # PostgreSQL（数据持久化）
├── redis          # Redis（缓存+消息总线）
├── chromadb       # ChromaDB（向量记忆）
├── one-api        # 模型中转站
├── openproject    # 项目管理工具
└── frontend       # React前端
```

**依赖关系**：
- Gateway 依赖 PostgreSQL、Redis、ChromaDB、One API
- One API 依赖 PostgreSQL（共享）
- OpenProject 依赖 PostgreSQL（独立数据库）

**关键端口**：
- Gateway WebSocket: `18789`
- Gateway Metrics: `18790`
- One API: `3000`
- OpenProject: `8080`
- Frontend: `5173`

## 3. 日常巡检清单

### 3.1 每日检查（建议CEO/EA执行）

| 检查项 | 方法 | 正常状态 | 异常处理 |
|--------|------|---------|---------|
| Dashboard告警面板 | 打开Dashboard首页 | 无Critical告警，Warn告警已处理 | 见第5章 |
| Agent在线状态 | Dashboard统计卡片 | 所有预期Agent在线 | 见5.1 |
| 分公司状态 | Dashboard分公司面板 | 所有预期分公司绿灯 | 见5.2 |
| 模型中转站余额 | 设置→模型中转站 | 所有Provider额度充足 | 及时充值 |
| 近24h任务成功率 | Dashboard统计卡片 | >85% | 查看失败任务详情 |
| 近24h模型调用成本 | Dashboard统计卡片 | 在预算范围内 | 见5.3 |

### 3.2 每周检查

| 检查项 | 方法 | 正常状态 |
|--------|------|---------|
| 审计日志摘要 | EA自动生成周报 | 无异常模式 |
| Agent绩效报告 | HR Manager自动生成 | 无持续低绩效Agent |
| 组织健康度 | Dashboard组织健康度面板 | 评分>75 |
| 数据库备份 | 检查备份文件 | 最近一次备份在24h内 |

### 3.3 每月检查

| 检查项 | 方法 | 正常状态 |
|--------|------|---------|
| Gateway Token轮换 | 手动执行 | Token已更新 |
| 过期审计日志清理 | 自动脚本 | 90天前日志已归档 |
| 模板库质量审查 | HR Manager自动报告 | 无模板成功率<60% |
| 系统韧性评分 | Dashboard韧性面板 | >80 |

## 4. 备份与恢复

### 4.1 自动备份

系统配置了以下自动备份（通过 `scripts/backup.sh` 和 cron）：

| 数据类型 | 备份频率 | 保留期 | 备份位置 |
|---------|---------|--------|---------|
| PostgreSQL | 每日02:00全量 | 30天 | `/backups/postgres/` |
| Redis | 每6小时RDB | 7天 | `/backups/redis/` |
| ChromaDB | 每日03:00 | 30天 | `/backups/chromadb/` |
| 配置文件 | 每次Git提交 | 永久 | Git仓库 |

### 4.2 手动备份

```bash
# 备份全部数据
./scripts/backup.sh full

# 仅备份数据库
./scripts/backup.sh db

# 备份到远程位置
./scripts/backup.sh full --remote s3://backup-bucket/
```

### 4.3 恢复流程

**场景：数据库损坏，需要从备份恢复**

```bash
# 1. 停止Gateway（防止新数据写入）
docker-compose stop gateway

# 2. 恢复PostgreSQL
docker-compose exec postgres pg_restore -U agent -d agent_organization /backups/postgres/latest.dump

# 3. 恢复ChromaDB
cp -r /backups/chromadb/latest/* /var/lib/docker/volumes/simu-corp_chromadb_data/_data/

# 4. 重启所有服务
docker-compose restart

# 5. 验证恢复
curl http://localhost:18789/api/v1/system/health
```

**场景：从零恢复整个系统**

```bash
# 1. 克隆项目
git clone <repo> && cd simu-corp

# 2. 恢复配置文件
cp /backups/config/openclaw.json .
cp /backups/config/.env .

# 3. 启动服务
docker-compose up -d

# 4. 恢复数据库
docker-compose exec -T postgres psql -U agent agent_organization < /backups/postgres/latest.sql

# 5. 验证
curl http://localhost:18789/api/v1/system/health
```

## 5. 常见问题排障指南

### 5.1 Agent 不响应

**症状**：CEO发送消息后，EA或GTF长时间不回复。

**排查步骤**：
1. 检查Agent状态：`GET /api/v1/agents/:agentId` → 查看 `status` 字段
2. 若状态为 `offline`：检查Agent的Runtime是否运行
   - OpenClaw原生Agent：检查Gateway日志
   - Claude Code Agent：检查Claude Code进程
3. 若状态为 `error`：查看审计日志中该Agent最近的错误记录
4. 若状态正常但不响应：可能是消息总线延迟，检查Redis状态

**快速修复**：
- 重启Agent的Runtime（如果是外部Runtime）
- 或重启Gateway：`docker-compose restart gateway`

### 5.2 分公司连接不上

**症状**：Dashboard显示分公司离线（红色）。

**排查步骤**：
1. 检查分公司机器网络连通性：`ping <分公司IP>`
2. 检查分公司Gateway是否运行：`curl http://<分公司IP>:18789/api/v1/system/health`
3. 检查分公司Redis连接：分公司Gateway日志中搜索 `BRANCH_HEARTBEAT`
4. 检查总部消息总线是否正常

**快速修复**：
- 在分公司机器上：`docker-compose restart gateway`
- 分公司恢复后会自动重新注册

### 5.3 模型调用成本异常增长

**症状**：Dashboard显示本月花费异常高。

**排查步骤**：
1. 查看模型用量统计：设置→模型中转站→查看详情
2. 按Agent查看调用量：找出调用量异常的Agent
3. 查看该Agent近期的任务类型：是否在频繁执行高Token消耗的任务
4. 检查是否有多余的Agent在空转

**修复措施**：
- 为异常Agent设置月度调用上限
- 调整Agent的模型配置，使用更经济的模型
- 暂停不必要的后台任务

### 5.4 Gateway启动失败

**症状**：`docker-compose up gateway` 后容器退出。

**排查步骤**：
1. 查看Gateway日志：`docker-compose logs gateway`
2. 常见原因：
   - `openclaw.json` 格式错误 → 用JSON验证工具检查
   - 数据库连接失败 → 确认PostgreSQL已启动且健康
   - 端口冲突 → 检查18789/18790端口是否被占用

### 5.5 Dashboard加载缓慢

**症状**：前端页面加载超过5秒。

**排查步骤**：
1. 检查 `GET /api/v1/system/health` 响应时间
2. 检查PostgreSQL连接池是否耗尽
3. 检查Redis内存使用是否接近上限
4. 检查Agent数量是否过多导致列表查询变慢

**优化措施**：
- 增加Dashboard数据的缓存时间
- 为Agent列表查询添加索引
- 清理过期审计日志

### 5.6 沙盒创建失败

**症状**：研发Agent报告无法创建Docker沙盒。

**排查步骤**：
1. 检查Docker服务是否运行：`docker ps`
2. 检查磁盘空间：`df -h`（沙盒需要至少10GB）
3. 检查Docker镜像是否存在
4. 检查Gateway是否有Docker socket访问权限

## 6. 升级指南

### 6.1 Gateway升级

```bash
# 1. 通知所有Agent：即将进行维护
# （通过WebChat发送“系统将在5分钟后进行升级维护”）

# 2. 暂停新任务分配
docker-compose exec gateway node scripts/pause-new-tasks.js

# 3. 等待进行中任务完成（最多等待30分钟）
docker-compose exec gateway node scripts/wait-pending-tasks.js --timeout 1800

# 4. 备份
./scripts/backup.sh full

# 5. 拉取新镜像
docker-compose pull gateway

# 6. 重启Gateway
docker-compose up -d gateway

# 7. 验证
curl http://localhost:18789/api/v1/system/health

# 8. 恢复任务分配
docker-compose exec gateway node scripts/resume-tasks.js
```

### 6.2 模型中转站升级

模型中转站可热升级（不影响运行中任务）：

```bash
docker-compose pull one-api
docker-compose up -d one-api
# 新任务自动使用新版本
```

### 6.3 OpenProject升级

按照OpenProject官方文档操作，注意：
- 升级前备份数据库
- 升级后验证API Token是否仍然有效
- 通知所有PM Agent：升级期间项目任务同步暂停

## 7. 紧急情况处理

### 7.1 系统完全不可用

**症状**：Gateway无响应，所有Agent离线。

**应急步骤**：
1. 检查服务器是否运行：`uptime`
2. 检查Docker服务：`systemctl status docker`
3. 检查磁盘空间：`df -h`（磁盘满会导致所有服务异常）
4. 重启所有服务：`docker-compose restart`
5. 若仍不可用，从备份恢复：见4.3节

### 7.2 数据泄露疑似事件

**应急步骤**：
1. 立即暂停所有Agent：`docker-compose exec gateway node scripts/pause-all-agents.js`
2. 导出近7天审计日志：`GET /api/v1/system/audit-log?from=7d`
3. 检查异常访问模式：搜索 `PERMISSION_DENIED` 和 `security.anomaly_detected` 事件
4. 通知CEO
5. 确认安全后恢复：`docker-compose exec gateway node scripts/resume-all-agents.js`

### 7.3 模型额度耗尽

**应急步骤**：
1. Dashboard会显示Critical告警
2. 非关键Agent的模型调用已被自动暂停
3. CEO可通过WebChat与EA正常对话（EA使用保留额度）
4. 在模型中转站充值或切换Provider
5. 恢复后，被暂停的Agent自动恢复

## 8. 监控与告警配置

### 8.1 Prometheus指标

Gateway暴露指标在 `http://localhost:18790/metrics`，Prometheus配置：

```yaml
scrape_configs:
  - job_name: 'simucorp-gateway'
    scrape_interval: 15s
    static_configs:
      - targets: ['gateway:18790']
```

### 8.2 Grafana看板

在Grafana中导入预置看板（`docs/grafana-dashboard.json`），展示：
- Agent在线状态趋势
- 任务成功率曲线
- 模型调用量与成本
- 分公司心跳延迟
- 消息总线吞吐量

## 9. 运维自动化脚本

| 脚本 | 用途 | 执行频率 |
|------|------|---------|
| `scripts/backup.sh` | 全量/增量备份 | 每日自动 |
| `scripts/cleanup-audit-logs.sh` | 清理过期审计日志 | 每周自动 |
| `scripts/health-check.sh` | 全系统健康检查 | 每小时自动 |
| `scripts/pause-all-agents.js` | 紧急暂停所有Agent | 手动 |
| `scripts/resume-all-agents.js` | 恢复所有Agent | 手动 |
| `scripts/rotate-tokens.sh` | 轮换Gateway Token | 每月手动 |

## 10. 运维Checklist（新任管理员入职）

| 检查项 | 说明 |
|--------|------|
| 了解系统架构 | 阅读设计文档第2章 |
| 掌握启停操作 | `docker-compose up/down/restart` |
| 掌握备份恢复 | 执行一次完整的备份和恢复演练 |
| 掌握排障流程 | 熟悉第5节常见问题处理 |
| 了解告警机制 | 知道不同级别告警的含义和应对方式 |
| 配置监控看板 | Grafana中查看所有面板 |
| 知道紧急联系方式 | CEO的联系渠道 |

---


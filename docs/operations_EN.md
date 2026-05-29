# Document 8: Operations Manual

**File**: `docs/operations.md`  
**Status**: Required before and after Phase 3

---

# SimuCorp — Operations Manual

## 1. Overview

This document defines the daily operation and maintenance guidelines for the SimuCorp system. It is intended for system administrators (or Agents responsible for operations) and the CEO.

## 2. System Architecture Overview

```
simu-corp/
├── gateway        # OpenClaw Gateway (Core, must be guaranteed)
├── postgres       # PostgreSQL (Data Persistence)
├── redis          # Redis (Cache + Message Bus)
├── chromadb       # ChromaDB (Vector Memory)
├── one-api        # Model Relay Station
├── openproject    # Project Management Tool
└── frontend       # React Frontend
```

**Dependencies**:
- Gateway depends on PostgreSQL, Redis, ChromaDB, One API
- One API depends on PostgreSQL (shared)
- OpenProject depends on PostgreSQL (independent database)

**Key Ports**:
- Gateway WebSocket: `18789`
- Gateway Metrics: `18790`
- One API: `3000`
- OpenProject: `8080`
- Frontend: `5173`

## 3. Daily Inspection Checklist

### 3.1 Daily Checks (Recommended for CEO/EA)

| Check Item | Method | Normal Status | Exception Handling |
|------------|--------|---------------|--------------------|
| Dashboard Alert Panel | Open Dashboard Homepage | No Critical alerts, Warn alerts handled | See Chapter 5 |
| Agent Online Status | Dashboard Statistics Card | All expected Agents online | See 5.1 |
| Branch Status | Dashboard Branch Panel | All expected branches green | See 5.2 |
| Model Relay Station Balance | Settings → Model Relay Station | All Provider quotas sufficient | Recharge promptly |
| Task Success Rate (Last 24h) | Dashboard Statistics Card | >85% | View failed task details |
| Model Call Cost (Last 24h) | Dashboard Statistics Card | Within budget | See 5.3 |

### 3.2 Weekly Checks

| Check Item | Method | Normal Status |
|------------|--------|---------------|
| Audit Log Summary | EA auto-generates weekly report | No abnormal patterns |
| Agent Performance Report | HR Manager auto-generates | No persistently low-performing Agents |
| Organization Health | Dashboard Organization Health Panel | Score > 75 |
| Database Backup | Check backup files | Latest backup within 24h |

### 3.3 Monthly Checks

| Check Item | Method | Normal Status |
|------------|--------|---------------|
| Gateway Token Rotation | Manual execution | Token updated |
| Expired Audit Log Cleanup | Automated script | Logs older than 90 days archived |
| Template Library Quality Review | HR Manager auto-report | No template success rate < 60% |
| System Resilience Score | Dashboard Resilience Panel | > 80 |

## 4. Backup and Recovery

### 4.1 Automated Backups

The system is configured with the following automated backups (via `scripts/backup.sh` and cron):

| Data Type | Backup Frequency | Retention Period | Backup Location |
|-----------|-----------------|------------------|-----------------|
| PostgreSQL | Daily full backup at 02:00 | 30 days | `/backups/postgres/` |
| Redis | RDB every 6 hours | 7 days | `/backups/redis/` |
| ChromaDB | Daily at 03:00 | 30 days | `/backups/chromadb/` |
| Configuration Files | Every Git commit | Permanent | Git Repository |

### 4.2 Manual Backup

```bash
# Backup all data
./scripts/backup.sh full

# Backup database only
./scripts/backup.sh db

# Backup to remote location
./scripts/backup.sh full --remote s3://backup-bucket/
```

### 4.3 Recovery Process

**Scenario: Database corruption, needs recovery from backup**

```bash
# 1. Stop Gateway (prevent new data writes)
docker-compose stop gateway

# 2. Restore PostgreSQL
docker-compose exec postgres pg_restore -U agent -d agent_organization /backups/postgres/latest.dump

# 3. Restore ChromaDB
cp -r /backups/chromadb/latest/* /var/lib/docker/volumes/simu-corp_chromadb_data/_data/

# 4. Restart all services
docker-compose restart

# 5. Verify recovery
curl http://localhost:18789/api/v1/system/health
```

**Scenario: Full system recovery from scratch**

```bash
# 1. Clone the project
git clone <repo> && cd simu-corp

# 2. Restore configuration files
cp /backups/config/openclaw.json .
cp /backups/config/.env .

# 3. Start services
docker-compose up -d

# 4. Restore database
docker-compose exec -T postgres psql -U agent agent_organization < /backups/postgres/latest.sql

# 5. Verify
curl http://localhost:18789/api/v1/system/health
```

## 5. Common Troubleshooting Guide

### 5.1 Agent Not Responding

**Symptom**: After the CEO sends a message, EA or GTF does not reply for a long time.

**Troubleshooting Steps**:
1. Check Agent status: `GET /api/v1/agents/:agentId` → View `status` field
2. If status is `offline`: Check if the Agent's Runtime is running
   - OpenClaw native Agent: Check Gateway logs
   - Claude Code Agent: Check Claude Code process
3. If status is `error`: View the Agent's recent error records in the audit log
4. If status is normal but not responding: Possible message bus latency, check Redis status

**Quick Fix**:
- Restart the Agent's Runtime (if it's an external Runtime)
- Or restart Gateway: `docker-compose restart gateway`

### 5.2 Branch Cannot Connect

**Symptom**: Dashboard shows branch offline (red).

**Troubleshooting Steps**:
1. Check branch machine network connectivity: `ping <Branch IP>`
2. Check if branch Gateway is running: `curl http://<Branch IP>:18789/api/v1/system/health`
3. Check branch Redis connection: Search for `BRANCH_HEARTBEAT` in branch Gateway logs
4. Check if the headquarters message bus is normal

**Quick Fix**:
- On the branch machine: `docker-compose restart gateway`
- The branch will automatically re-register after recovery

### 5.3 Abnormal Growth in Model Call Costs

**Symptom**: Dashboard shows unusually high spending for the current month.

**Troubleshooting Steps**:
1. View model usage statistics: Settings → Model Relay Station → View Details
2. Check call volume by Agent: Identify Agents with abnormal call volume
3. Check the recent task types of that Agent: Is it frequently executing high-token-consumption tasks?
4. Check if there are redundant Agents idling

**Remediation Measures**:
- Set a monthly call limit for the abnormal Agent
- Adjust the Agent's model configuration to use a more economical model
- Suspend unnecessary background tasks

### 5.4 Gateway Startup Failure

**Symptom**: Container exits after `docker-compose up gateway`.

**Troubleshooting Steps**:
1. View Gateway logs: `docker-compose logs gateway`
2. Common causes:
   - `openclaw.json` format error → Check with a JSON validator tool
   - Database connection failure → Confirm PostgreSQL is started and healthy
   - Port conflict → Check if ports 18789/18790 are occupied

### 5.5 Dashboard Loads Slowly

**Symptom**: Frontend page takes more than 5 seconds to load.

**Troubleshooting Steps**:
1. Check `GET /api/v1/system/health` response time
2. Check if the PostgreSQL connection pool is exhausted
3. Check if Redis memory usage is near the limit
4. Check if the number of Agents is too large, causing slow list queries

**Optimization Measures**:
- Increase the cache time for Dashboard data
- Add indexes for Agent list queries
- Clean up expired audit logs

### 5.6 Sandbox Creation Failure

**Symptom**: R&D Agent reports inability to create a Docker sandbox.

**Troubleshooting Steps**:
1. Check if the Docker service is running: `docker ps`
2. Check disk space: `df -h` (Sandbox requires at least 10GB)
3. Check if the Docker image exists
4. Check if the Gateway has Docker socket access permissions

## 6. Upgrade Guide

### 6.1 Gateway Upgrade

```bash
# 1. Notify all Agents: Maintenance is about to commence
# (Send "System will undergo upgrade maintenance in 5 minutes" via WebChat)

# 2. Pause new task assignment
docker-compose exec gateway node scripts/pause-new-tasks.js

# 3. Wait for in-progress tasks to complete (max wait 30 minutes)
docker-compose exec gateway node scripts/wait-pending-tasks.js --timeout 1800

# 4. Backup
./scripts/backup.sh full

# 5. Pull new image
docker-compose pull gateway

# 6. Restart Gateway
docker-compose up -d gateway

# 7. Verify
curl http://localhost:18789/api/v1/system/health

# 8. Resume task assignment
docker-compose exec gateway node scripts/resume-tasks.js
```

### 6.2 Model Relay Station Upgrade

The model relay station supports hot upgrades (does not affect running tasks):

```bash
docker-compose pull one-api
docker-compose up -d one-api
# New tasks automatically use the new version
```

### 6.3 OpenProject Upgrade

Follow the official OpenProject documentation. Note:
- Backup the database before upgrading
- Verify if the API Token is still valid after upgrading
- Notify all PM Agents: Project task synchronization is paused during the upgrade

## 7. Emergency Handling

### 7.1 System Completely Unavailable

**Symptom**: Gateway unresponsive, all Agents offline.

**Emergency Steps**:
1. Check if the server is running: `uptime`
2. Check Docker service: `systemctl status docker`
3. Check disk space: `df -h` (Full disk can cause all services to malfunction)
4. Restart all services: `docker-compose restart`
5. If still unavailable, restore from backup: See Section 4.3

### 7.2 Suspected Data Leak Incident

**Emergency Steps**:
1. Immediately pause all Agents: `docker-compose exec gateway node scripts/pause-all-agents.js`
2. Export audit logs for the last 7 days: `GET /api/v1/system/audit-log?from=7d`
3. Check for abnormal access patterns: Search for `PERMISSION_DENIED` and `security.anomaly_detected` events
4. Notify the CEO
5. Resume after confirming security: `docker-compose exec gateway node scripts/resume-all-agents.js`

### 7.3 Model Quota Exhausted

**Emergency Steps**:
1. Dashboard will display a Critical alert
2. Model calls for non-critical Agents are automatically suspended
3. CEO can communicate normally with EA via WebChat (EA uses reserved quota)
4. Recharge in the model relay station or switch Provider
5. After recovery, suspended Agents will automatically resume

## 8. Monitoring and Alert Configuration

### 8.1 Prometheus Metrics

Gateway exposes metrics at `http://localhost:18790/metrics`. Prometheus configuration:

```yaml
scrape_configs:
  - job_name: 'simucorp-gateway'
    scrape_interval: 15s
    static_configs:
      - targets: ['gateway:18790']
```

### 8.2 Grafana Dashboard

Import the preset dashboard (`docs/grafana-dashboard.json`) into Grafana to display:
- Agent online status trends
- Task success rate curve
- Model call volume and cost
- Branch heartbeat latency
- Message bus throughput

## 9. Operations Automation Scripts

| Script | Purpose | Execution Frequency |
|--------|---------|---------------------|
| `scripts/backup.sh` | Full/Incremental backup | Daily automatic |
| `scripts/cleanup-audit-logs.sh` | Clean up expired audit logs | Weekly automatic |
| `scripts/health-check.sh` | Full system health check | Hourly automatic |
| `scripts/pause-all-agents.js` | Emergency pause all Agents | Manual |
| `scripts/resume-all-agents.js` | Resume all Agents | Manual |
| `scripts/rotate-tokens.sh` | Rotate Gateway Token | Monthly manual |

## 10. Operations Checklist (For New Administrator Onboarding)

| Check Item | Description |
|------------|-------------|
| Understand system architecture | Read Chapter 2 of the design document |
| Master start/stop operations | `docker-compose up/down/restart` |
| Master backup and recovery | Perform a complete backup and recovery drill |
| Master troubleshooting process | Be familiar with common issue handling in Section 5 |
| Understand alert mechanisms | Know the meaning of different alert levels and how to respond |
| Configure monitoring dashboard | View all panels in Grafana |
| Know emergency contact information | CEO's contact channel |
# Document 4: Test Scenarios and Acceptance Criteria

**File**: `docs/test-scenarios.md`  
**Status**: Required during Phase 1 development

---

# SimuCorp — Test Scenarios and Acceptance Criteria

## 1. Overview

This document defines the test scenarios and acceptance criteria for each phase. After completing a module, the development agent should self-test against this document. Formal acceptance will be conducted by the CEO or a testing agent at the end of each phase.

## 2. Phase 1 Acceptance Criteria (MVP)

### 2.1 Environment Setup Acceptance

| ID | Test Item | Steps | Expected Result | Status |
|------|--------|---------|---------|------|
| ENV-01 | Docker Compose Startup | Run `docker-compose up -d` | All services start successfully; `docker-compose ps` shows all services as healthy | ⬜ |
| ENV-02 | Gateway WebSocket Connection | Connect to `ws://localhost:18789` using a WebSocket client, send a Connect Frame | Returns `connect_ack` containing `session_id` | ⬜ |
| ENV-03 | Gateway Health Check | `GET /api/v1/system/health` | Returns healthy status for all components | ⬜ |
| ENV-04 | Initial Agents in Database | Query the `agents` table | Three records exist: ea, hr_manager, gtf_manager | ⬜ |
| ENV-05 | Model Relay Accessible | Access `http://localhost:3000` | One API management interface is accessible for login | ⬜ |
| ENV-06 | OpenProject Accessible | Access `http://localhost:8080` | OpenProject interface is accessible | ⬜ |

### 2.2 Core Communication Acceptance

| ID | Test Item | Steps | Expected Result | Status |
|------|--------|---------|---------|------|
| COM-01 | Frontend Connects to Gateway | Open frontend at `http://localhost:5173` | Sidebar shows WebSocket connection status as green 🟢 | ⬜ |
| COM-02 | CEO Sends Message to EA | Type "Hello, EA" in WebChat | EA replies with a reasonable response (self-introduction or asking for needs) | ⬜ |
| COM-03 | EA Routes Simple Task to GTF | CEO types "Help me write a Python script to calculate the first 20 Fibonacci numbers" in WebChat | EA replies that the task has been delegated to GTF; GTF returns the script code within a reasonable time | ⬜ |
| COM-04 | EA Asks for Clarification on Ambiguous Task | CEO types "Help me handle that issue" in WebChat | EA asks for clarification on what the specific issue is, rather than guessing and executing | ⬜ |
| COM-05 | GTF Progress Report for Long Tasks | CEO sends a task requiring a long time (e.g., "Analyze this CSV dataset") | During GTF execution, EA reports progress every 30 minutes | ⬜ |

### 2.3 Frontend Page Acceptance

| ID | Test Item | Steps | Expected Result | Status |
|------|--------|---------|---------|------|
| UI-01 | Dashboard Loads | Access `/dashboard` | Four statistics cards display data; real-time activity stream has initial content | ⬜ |
| UI-02 | Dashboard Statistics Card Data Correct | Compare `/api/v1/system/metrics` return value with card display | Data is consistent | ⬜ |
| UI-03 | WebChat Conversation Interface | Click "Chat with EA" on Dashboard or type directly in WebChat | Messages are sent and received normally; new messages auto-scroll to the bottom | ⬜ |
| UI-04 | Sidebar Navigation | Click each menu item in the sidebar | Correctly navigates to the corresponding page; current page is highlighted | ⬜ |
| UI-05 | Agent List Page | Access `/org`, list view | Displays three agents: ea, hr_manager, gtf_manager; status is correct | ⬜ |
| UI-06 | Agent Detail Page | Click GTF Manager in the list | Navigates to `/org/gtf_manager`; displays basic info, model configuration, and toolset | ⬜ |
| UI-07 | Language Switch | Click the language switch dropdown in the TopBar, select English | All interface text switches to English | ⬜ |
| UI-08 | WebSocket Reconnection | Manually stop Gateway: `docker-compose stop gateway` | Frontend shows connection disconnected 🔴; after restarting Gateway, automatically reconnects 🟢 | ⬜ |

### 2.4 Phase 1 End-to-End Test Scenario

**Scenario: CEO uses SimuCorp for the first time to complete a simple task**

```
Prerequisites: All services are running normally; frontend is open

Step 1: CEO opens Dashboard
  - Sees four statistics cards (Online Agents, In-Progress Tasks, Pending Approvals, Monthly Spend)
  - Real-time activity stream shows events since system startup

Step 2: CEO types "Help me analyze the trends in this quarter's sales data" in WebChat
  - EA replies: analyzes intent, confirms need for data analysis capability
  - EA replies: No dedicated data analyst currently; task delegated to GTF Manager

Step 3: EA sends TASK_ASSIGN to GTF
  - GTF confirms receipt of task
  - Dashboard real-time activity stream shows "GTF Manager started executing task"

Step 4: GTF executes analysis (simulated data)
  - After 10 minutes, GTF returns analysis results (sales trends, key findings, recommendations)
  - EA forwards results to CEO
  - Dashboard activity stream updates

Step 5: CEO reviews results
  - Complete analysis report displayed in WebChat
  - CEO replies "Thanks, good job"

Step 6: Verify audit log
  - Query /api/v1/system/audit-log
  - Should include: message.sent (CEO→EA), task.assigned (EA→GTF), task.completed (GTF), message.sent (EA→CEO)
```

## 3. Phase 2 Acceptance Criteria

### 3.1 Organizational Structure Visualization

| ID | Test Item | Expected Result | Status |
|------|--------|---------|------|
| ORG-01 | Topology View Display | Displays hierarchical relationship CEO→EA→HR/GTF; nodes are draggable | ⬜ |
| ORG-02 | Node Click Details | Click GTF node; detail panel slides out from the right | ⬜ |
| ORG-03 | Right-Click Menu | Right-click Agent node; displays "View Details/Pause/Wake" menu | ⬜ |
| ORG-04 | Create New Agent | Click "+ New Agent"; template selection dialog pops up | ⬜ |
| ORG-05 | Real-Time Status Update | When GTF status changes from idle to busy, node color in topology view updates synchronously | ⬜ |

### 3.2 Automated Recruitment Closed Loop

| ID | Test Item | Steps | Expected Result | Status |
|------|--------|---------|---------|------|
| REC-01 | GTF Review Triggers Recruitment | GTF completes 3 iOS tasks consecutively | GTF automatically sends TALENT_REQUEST to HR | ⬜ |
| REC-02 | HR Generates JD | HR receives TALENT_REQUEST | HR searches external market, matches templates, generates job analysis report | ⬜ |
| REC-03 | Create Intern Agent | HR executes creation | New intern agent appears in the system with status marked as intern | ⬜ |
| REC-04 | Intern Agent Receives Tasks | Hiring department assigns tasks to intern agent | Intern agent executes normally | ⬜ |
| REC-05 | Internship Period Evaluation | Intern agent completes 10 tasks | HR automatically calculates comprehensive score; executes conversion/extension/elimination | ⬜ |
| REC-06 | Converted Agent Capability Update | Agent is converted | Capability catalog is updated; agent appears in professional capability matching list | ⬜ |
| REC-07 | Eliminated Agent Archiving | Agent is eliminated | Memory archived; CEO notified; agent removed from capability catalog | ⬜ |

### 3.3 Approval Center

| ID | Test Item | Expected Result | Status |
|------|--------|---------|------|
| APP-01 | Approval List Loads | Displays three tabs: Pending Approval / Approved / Rejected | ⬜ |
| APP-02 | Approval Card Display | Shows applicant, content summary, priority color marker, time | ⬜ |
| APP-03 | Approve Action | Click approve → approval status updates → applicant notified → audit recorded | ⬜ |
| APP-04 | Reject Action | Click reject → enter reason → approval status updates → applicant notified | ⬜ |
| APP-05 | Expiry Handling | Approval exceeds 72 hours → automatic reminder → auto-reject after 7 days | ⬜ |

### 3.4 Project Management Integration

| ID | Test Item | Expected Result | Status |
|------|--------|---------|------|
| PRJ-01 | Create New Project | Fill form → OpenProject creates project → PM Agent created → WBS initialized | ⬜ |
| PRJ-02 | Task Board | Project detail page shows OpenProject board; task cards are draggable | ⬜ |
| PRJ-03 | Task Creation | PM Agent automatically creates tasks in OpenProject | ⬜ |
| PRJ-04 | Project Progress Statistics | Project list displays correct progress percentage and task statistics | ⬜ |
| PRJ-05 | Wiki Initialization | Wiki automatically creates homepage after new project is created | ⬜ |

## 4. Phase 3 Acceptance Criteria

| ID | Test Item | Expected Result | Status |
|------|--------|---------|------|
| MTG-01 | Meeting Initiation | PM initiates MEETING_REQUEST → EA generates approval → CEO approves | ⬜ |
| MTG-02 | Real-Time Meeting Chat | Multiple people enter meeting → messages broadcast in real-time → participant list shows online status | ⬜ |
| MTG-03 | Meeting Minutes Generation | CEO ends meeting → EA automatically generates structured minutes → Action Items distributed | ⬜ |
| BRC-01 | Branch Registration | New branch starts → sends BRANCH_REGISTER → headquarters receives approval | ⬜ |
| BRC-02 | Branch Heartbeat | Heartbeat every 5 minutes → headquarters Dashboard displays real-time load | ⬜ |
| BRC-03 | Cross-Branch Delegation | Headquarters delegates GPU task → branch executes → returns results | ⬜ |
| BRC-04 | Branch Offline Recovery | Simulate network outage → task transferred → re-online after recovery | ⬜ |
| MON-01 | Monitoring Metrics Collection | Prometheus can collect Gateway metrics | ⬜ |
| MON-02 | Alert Push | Simulate agent offline → CEO WebChat receives Critical alert | ⬜ |
| SOU-01 | SOUL Optimization Process | HR analyzes performance → generates new Soul → sandbox testing → A/B testing → go live | ⬜ |
| SOU-02 | SOUL Rollback | New Soul performs poorly → auto-rollback → notify HR | ⬜ |
| I18-01 | Complete Chinese | All page text is in Chinese | ⬜ |
| I18-02 | Complete English | Switch to English → all page text is in English | ⬜ |

## 5. Exception Scenario Testing (Applicable to All Phases)

| ID | Scenario | Simulation Method | Expected Behavior | Recovery Verification |
|------|------|---------|---------|---------|
| ERR-01 | Agent Task Timeout | Assign GTF an infinite loop task it cannot complete | EA sends QUERY after timeout; if no response, reassigns to another agent | Task eventually completes or is marked as failed |
| ERR-02 | Agent Offline | Manually stop GTF's Runtime process | EA detects heartbeat loss; GTF's tasks are automatically transferred to another agent | After recovery, GTF rejoins the capability catalog |
| ERR-03 | Model Degradation | Pause the primary model in the model relay | Agent automatically switches to fallback model | Degradation event recorded; tasks continue execution |
| ERR-04 | Branch Disconnected | Stop branch Gateway | Headquarters EA marks branch as offline; time-sensitive tasks are transferred | Branch automatically reconnects after recovery |
| ERR-05 | Message Bus Failure | Stop Redis | Gateway starts local in-memory queue; pauses cross-agent communication | Replays messages after recovery |
| ERR-06 | All Models Fail | Pause all model providers | Critical tasks paused; CEO notified; non-critical tasks marked as failed | Continue after recovery |
| ERR-07 | System Restart | `docker-compose restart` | Graceful shutdown → save state → startup recovery → agents resume work | Verify in-progress tasks are recovered |

## 6. Performance Baselines

| Metric | Phase 1 | Phase 2 | Phase 3 |
|------|---------|---------|---------|
| Dashboard Initial Load | < 3 seconds | < 3 seconds | < 3 seconds |
| WebSocket Message Latency | < 500ms | < 500ms | < 500ms |
| Agent List Query | < 1 second (<50 agents) | < 1 second (<100 agents) | < 2 seconds (<500 agents) |
| Task Routing Decision | < 5 seconds | < 3 seconds | < 3 seconds |
| Meeting Message Broadcast | N/A | < 2 seconds | < 1 second |
| Audit Log Query | N/A | < 3 seconds (within 1000 records) | < 3 seconds (within 10000 records) |

## 7. Acceptance Process

1. After completing a module, the development agent self-tests against this document
2. Fill in the test results in the "Status" column of each test item (⬜ → ✅ or ❌)
3. For ❌ items, attach the failure reason and fix plan
4. At the end of each phase, submit a test report to the CEO
5. The CEO or testing agent conducts spot-check acceptance
6. All critical test items (marked as "Critical") must pass
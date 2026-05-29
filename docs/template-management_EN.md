# Document 7: Agent Template Library Management Specification

**File**: `docs/template-management.md`  
**Status**: Required before Phase 2

---

# SimuCorp — Agent Template Library Management Specification

## 1. Overview

This document defines the management rules for the Agent Template Library. The template library serves as the foundation for the HR Manager to quickly recruit new Agents—it stores preset Souls, tool sets, base recommendations, and assessment criteria for various positions. The quality of the template library directly impacts recruitment efficiency and Agent quality.

## 2. Template Structure Definition

Each template must include the following fields:

```json
{
  "template_id": "Unique identifier, lowercase + underscores",
  "name": "Template display name",
  "role": "Role name",
  "description": "Job description, outlining the responsibilities and positioning of the role",
  "version": "Semantic version number",
  "status": "draft | testing | active | needs_optimization | deprecated",
  "author": "Creator (hr_manager or ceo)",
  
  "prompt_template": "Complete Soul (system prompt), compliant with prompt-engineering-guide.md specifications",
  
  "suggested_runtime": "Suggested base type (openclaw-native | claude-code | codex-cli)",
  "suggested_primary_model": "Suggested primary model",
  "suggested_fallbacks": ["List of fallback models"],
  
  "suggested_tools": ["List of tool names"],
  
  "capabilities": {
    "skills": ["List of skills"],
    "domain": ["List of domains"],
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

## 3. Template Lifecycle

```
[Draft] → [Testing] → [Active] → [Needs Optimization] → [Deprecated]
```

### 3.1 Draft
- Initial state for newly created templates
- Editable at any time
- Not used by HR Manager for recruitment

### 3.2 Testing
- HR Manager is verifying template effectiveness in a sandbox
- Instantiate a temporary Agent to execute 3 test tasks
- Automatically transitions to "Active" upon passing tests

### 3.3 Active
- Available for formal recruitment
- Requires re-testing after editing

### 3.4 Needs Optimization
- Auto-triggered: Agent conversion success rate for this template < 60%
- Or average performance of Agents recruited via this template is 20% below the baseline for the same position
- HR Manager receives notification to optimize

### 3.5 Deprecated
- No longer in use, but historical data is retained
- Agents already recruited using this template are unaffected

## 4. Template Creation Process

### 4.1 HR Manager Auto-Creation
1. Upon receiving `TALENT_REQUEST`, search the external market for JDs
2. Search the template library for similar templates (semantic matching)
3. If a template with > 80% similarity is found, fine-tune and use it
4. If not, create a new template (status: Draft)
5. Instantiate a temporary Agent and execute sandbox tests
6. Tests passed → Status changes to "Active"
7. Tests failed → Adjust and retry, or mark as "Needs Manual Review"

### 4.2 CEO Manual Creation
1. Click "New Template" in "System Settings → Template Library"
2. Fill in template information
3. Can publish directly, or have HR Manager conduct sandbox testing

## 5. Template Quality Assessment

| Metric | Calculation Method | Warning Threshold |
|--------|--------------------|-------------------|
| Recruitment Success Rate | Conversion rate of Agents recruited via this template | < 60% |
| Average Agent Performance | Average performance over the last 30 days for Agents recruited via this template | < 80% of baseline for the same position |
| Usage Rate | Number of uses in the last 90 days | < 2 times (potentially outdated) |
| Soul Quality Score | Scored according to prompt-engineering-guide.md | < 7.0 |

## 6. Template Version Management

- Use semantic versioning: `Major.Minor.Patch`
- Increment version number after each edit
- Old versions are retained in the database and can be rolled back
- Changelog format:

```markdown
## [1.1.0] — 2026-06-20
### Changes
- Adjusted code of conduct in Soul
- Added Xcode tool to suggested tool set
### Test Results
- Sandbox tests: 3/3 passed
```

## 7. Template Library Directory Structure (File System Backup)

In addition to database storage, the template library is periodically backed up as JSON files:

```
data/templates/
├── index.json              # Template index
├── backend_dev.json
├── frontend_dev.json
├── data_analyst.json
├── devops_eng.json
├── security_auditor.json
├── pm.json
└── ...
```

## 8. Template Library Management Frontend Page

### 8.1 Template List Page
- Card grid layout, each card displays:
  - Template name, role, status badge
  - Core skill tags
  - Usage count, success rate
- Supports filtering by status and searching by skill
- Click card to enter details

### 8.2 Template Detail Page
- Complete Soul preview (read-only)
- Suggested tool set list
- Suggested base and model
- Usage statistics (recruitment count, success rate, performance trend chart)
- Action buttons: Edit, Test, Instantiate, Deprecate

## 9. Development Checklist

| Check Item | Pass Criteria |
|------------|---------------|
| Complete template structure | Contains all required fields |
| Soul compliant with specifications | Passes quality assessment per prompt-engineering-guide.md (≥ 7.0) |
| Sandbox tests passed | 3/3 test tasks passed |
| Base selection reasonable | Code type → Claude Code, Coordination type → OpenClaw Native |
| Tool set consistent with Soul | Tools mentioned in Soul are all in the tool set list |
| Version number correct | New templates start from 1.0.0 |
| Changelog recorded | Each edit has a corresponding change description |

---
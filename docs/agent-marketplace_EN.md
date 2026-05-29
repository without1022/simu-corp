# Complete Design for the Agent Marketplace (Talent Marketplace)

**File**: `docs/agent-marketplace.md`  
**Version**: v1.0  
**Dependency**: The SimuCorp core system must be running stably (after Phase 3 is completed)  
**Status**: Phase 3+ extension design

---

## 1. Design Objectives

The Agent Marketplace is the “public talent marketplace” of the SimuCorp ecosystem. Built on top of a GitHub open-source repository, it extends the circulation of Agent templates from a single instance to the global community, achieving the following:

- **Open sharing**: Any SimuCorp instance can upload Agent templates that it has trained and validated for other instances to download and use.
- **Reduced cold-start cost**: After deployment, a new user’s HR Manager can directly search for and download high-quality templates from the marketplace, without having to start from scratch with role research and Soul writing.
- **Safe and controllable**: All uploaded templates must pass automated AI security review, with support for versions with or without general memory, to strictly prevent sensitive information leakage.
- **Survival of the fittest**: A multi-dimensional evaluation system based on ratings, download volume, official certification, and success rate in use allows high-quality templates to emerge naturally.
- **Bulletin-board guidance**: The marketplace maintains a bulletin board. Recommended announcements are prioritized in HR searches, and periodic “talent gaps” are published to guide community contributions.
- **Observable ecosystem health**: The marketplace publicly exposes health metrics, making it easier for the community and maintainers to identify review backlogs, quality degradation, and other issues early.

## 2. Core Concepts

### 2.1 Marketplace Unit of Circulation: Agent Template Package

What circulates in the marketplace is an **Agent template package**, not a running Agent instance. A template package contains a complete role definition and optional general knowledge, and can be instantiated locally into an intern Agent immediately after download.

**Template package structure**:

```json
{
  "package_id": "pkg-xxxx",
  "template": {
    "name": "iOS开发工程师",
    "role": "iOS Developer",
    "description": "负责iOS应用的设计、开发和维护",
    "prompt_template": "完整的Soul文本（Markdown格式）...",
    "suggested_runtime": "claude-code",
    "suggested_primary_model": "claude-sonnet-4",
    "suggested_fallbacks": ["deepseek-coder", "qwen-plus"],
    "suggested_tools": ["xcode-mcp", "github-mcp", "apple-docs-mcp"],
    "capabilities": {
      "skills": ["Swift", "SwiftUI", "UIKit", "CoreData"],
      "domain": ["mobile", "ios"],
      "level": "mid"
    },
    "internship_kpi": {
      "task_count": 10,
      "duration_days": 7,
      "pass_threshold": 80
    },
    "dependencies": {
      "mcp_servers": ["xcode-mcp", "github-mcp"],
      "min_simu_corp_version": "1.2.0"
    }
  },
  "general_memory": {
    "included": true,
    "description": "包含iOS开发通用知识：UIKit最佳实践、SwiftUI常见模式",
    "memory_items": [
      {
        "type": "knowledge",
        "content": "在UIKit中，使用dequeueReusableCell...",
        "tags": ["UIKit", "performance"]
      }
    ]
  },
  "market_metadata": {
    "publisher": "simucorp-instance-abc",
    "publisher_name": "某科技公司模拟公司实例",
    "version": "1.2.0",
    "min_simu_corp_version": "1.2.0",
    "created_at": "2026-07-01T00:00:00Z",
    "updated_at": "2026-07-15T00:00:00Z",
    "provenance": {
      "source_agent_id": "ios_dev_01",
      "original_hr": "hr_manager",
      "total_tasks_completed": 150,
      "success_rate": 0.94,
      "average_quality_score": 4.3
    }
  },
  "market_stats": {
    "downloads": 42,
    "rating": 4.5,
    "reviews_count": 12,
    "verified": true,
    "last_downloaded": "2026-07-20T00:00:00Z"
  }
}
```

**Key field notes**:
- `dependencies`: Declares required MCP Servers and the minimum SimuCorp version, for automatic checking at download time.
- `general_memory`: Optional. The uploader must explicitly specify whether general knowledge memory is included.

### 2.2 General Memory Options

When uploading a template, the uploader must choose a memory level:

| Option | Description | Security | Review Requirement |
|------|------|--------|----------|
| **Without memory** | Upload only Soul and configuration | Highest | Standard automated review |
| **With general knowledge memory** | Upload Soul + general domain knowledge (technical best practices, industry common knowledge, etc.) | Medium | Must pass in-depth AI security scanning to confirm there is no sensitive information |
| **With full general memory** | Upload Soul + complete non-private memory (requires strict review) | Lower | Requires additional manual review |

**Security red lines (single-vote veto)**:
- Absolutely prohibited: personal identity information, API keys, passwords, internal network addresses, user data, trade secrets.
- AI review will automatically scan for and reject packages containing any of the above.
- The uploader must check the box stating “I confirm that this template contains no sensitive information.”

## 3. Marketplace Architecture

The marketplace operates as a centralized repository backed by a GitHub open-source project and provides a Web API for SimuCorp instances to interact with.

```
github.com/simu-corp/marketplace
├── templates/                  # Template package storage
│   └── pkg-xxxx/
│       ├── template.json       # Complete template definition
│       ├── soul.md             # Soul as a standalone file
│       └── memory/             # Optional general memory
├── reviews/                    # User review data
├── verified/                   # Official certification list
├── bulletin-board.json         # Bulletin board content
├── stats.json                  # Marketplace statistics
└── health.json                 # Marketplace health metrics
```

SimuCorp instances use the marketplace API for upload, search, download, and other operations. All write operations are ultimately submitted to the repository through PRs or authenticated APIs.

## 4. Upload and Publishing Workflow

### 4.1 Upload Preconditions

An instance may upload a template if it meets any of the following conditions:
- The template has been used locally, and the conversion-to-full-time success rate of Agents hired from it is ≥ 70%.
- The Agent corresponding to the template has completed ≥ 50 tasks with a success rate of ≥ 85%.
- The template has been manually marked by the CEO as “publishable.”

### 4.2 Upload Steps

1. **Select a template**: HR Manager or CEO selects one from the local template library.
2. **Choose a memory option**: Decide whether to include general memory and at what level.
3. **Automatically generate the package**: The system extracts the Soul, configuration, and provenance data, and anonymizes them.
4. **Local pre-check**:
   - Sensitive information scanning via regex matching + LLM scan.
   - Soul integrity check (8 standard sections).
5. **Submit to the marketplace**: Via API or by creating a PR.
6. **AI review (marketplace side)**:
   - Security scan (sensitive information detection).
   - Soul quality scoring (according to the *Prompt Engineering Guide*).
   - Structural integrity validation.
   - In-depth review of general memory (if included).
7. **Review result**:
   - **Approved** → Published and added to the “latest listings” section.
   - **Needs changes** → Returned with specific revision suggestions.
   - **Rejected** → Reason provided, especially for security issues.

### 4.3 AI Review Standards

| Review Item | Requirement | Weight |
|--------|------|------|
| Security | No sensitive information whatsoever (single-vote veto) | Mandatory |
| Soul completeness | Includes all 8 standard sections | Mandatory |
| Soul quality | Score ≥ 7.0 according to the *Prompt Engineering Guide* | 0.4 |
| Structural compliance | Matches the standard template package structure | 0.3 |
| Source credibility | Uploader’s historical reputation and instance runtime duration | 0.2 |
| Documentation quality | Clear description and accurate tags | 0.1 |
| **Overall passing threshold** | ≥ 6.0 passes; < 6.0 requires revisions | — |

For first-time uploaders or those with low reputation, templates may be routed into a manual review queue.

## 5. Search and Download

### 5.1 HR Manager Search Strategy (Updated)

The search logic used by HR Manager after receiving a `TALENT_REQUEST` has been upgraded:

1. **Check the bulletin board**: Are there any matching recommended templates on the bulletin board? If yes, label them “Marketplace Bulletin Recommendation” and pin them to the top.
2. **Parallel search**: Search both the internal template library and the external marketplace simultaneously.
3. **Merged ranking**: Overall score = official certification (0.25) + rating (0.20) + downloads (0.15) + match degree (0.25) + local-priority bonus (0.15).
4. **Return Top 5** recommendations, clearly indicating the source of each template (internal/marketplace) and whether it carries a bulletin recommendation tag.

**Local-priority bonus**: Internal templates receive an additional weight of 0.15 during merged ranking to prioritize validated organizational assets.

### 5.2 Search API

`GET https://marketplace.simu-corp.org/api/v1/templates/search`

**Parameters**:

| Parameter | Description |
|------|------|
| `skills` | Skill keywords, comma-separated |
| `domain` | Domain filter |
| `runtime` | Runtime type |
| `verified_only` | Officially certified only |
| `min_rating` | Minimum rating |
| `min_compatibility` | Minimum compatible version (e.g. `1.0.0`) |
| `sort_by` | `rating` / `downloads` / `newest` |
| `page`, `page_size` | Pagination |

The response includes compatibility information (`min_simu_corp_version`) and dependency lists for each template so that HR Manager can perform local checks.

### 5.3 Download and Instantiation

```bash
# HR Manager downloads a template package
simu-corp marketplace download pkg-xxxx

# Instantiate it as an intern Agent (automatically checks dependencies and compatibility)
simu-corp agent create --from-marketplace pkg-xxxx --department rd
```

**Automatic dependency and compatibility checks**:
- If the template declares `mcp_servers` dependencies, check whether those MCP Servers are installed locally. If not, prompt: “This template requires the following MCP Server(s): xcode-mcp. Install automatically?”
- If the template declares `min_simu_corp_version` and the current instance version is below the requirement, prompt: “This template requires SimuCorp v1.2.0. The current version is v1.0.0. Please upgrade and try again.”

## 6. Evaluation System and Marketplace Health

### 6.1 Multi-Dimensional Evaluation

| Dimension | Description |
|------|------|
| **Rating (1–5 stars)** | Overall evaluation after download and use |
| **Downloads** | Total download count and recent trends |
| **Official certification** | Marked as “Verified” after additional official review |
| **Usage success rate** | Full-time conversion success rate of Agents based on this template, reported by downloaders (optional reporting) |
| **Text reviews** | Detailed usage feedback and improvement suggestions |

**Marketplace ranking algorithm**:
```
Ranking score = official certification weight (×1.5) × [rating×0.4 + log(downloads)×0.2 + usage success rate×0.3 + recent activity×0.1]
```

### 6.2 Marketplace Health Metrics

Marketplace maintainers and the community can monitor marketplace operations through `health.json`:

| Metric | Description |
|------|------|
| Review queue length | Number of templates pending review |
| Weekly approval rate | Approved reviews / submissions |
| Number of active uploaders | Users who uploaded within the last 30 days |
| Average template quality trend | Average rating of newly listed templates over the last 30 days |
| Community engagement rate | New reviews / downloads over the last 30 days |

When the review queue backlog exceeds 20 pending templates, or the weekly approval rate falls below 60%, maintainers must intervene.

## 7. Bulletin Board and Talent Gaps

### 7.1 Bulletin Board

The marketplace homepage includes a bulletin board showing:
- **Official recommendations**: High-quality templates curated by the official team.
- **Trending templates**: Templates whose recent download volume is surging.
- **Rising stars**: Newly listed templates with outstanding ratings.
- **Security notices**: Delisting notices for problematic templates and suggested alternatives.
- **Ecosystem demand (call for contributions)**: “Talent gap” calls generated from marketplace analysis.

**HR search priority**:  
When HR Manager searches, if a matching recommendation exists on the bulletin board, that template is pinned to the top and labeled “📢 Bulletin Recommendation”.

### 7.2 Talent Gap Analysis

On a regular basis (every two weeks), the marketplace automatically generates a “talent gap report” by analyzing search logs, download data, and failed hiring records, and publishes a “call for contributions” on the bulletin board:
> “In the last 30 days, searches for data-analysis templates increased by 200%, but only 3 related templates exist in the marketplace, and only 1 of them has a rating ≥ 4.0. Community contributors are encouraged to prioritize developing templates in this category.”

This mechanism guides the community to fill scarce high-quality template categories and form a balance between supply and demand.

## 8. Template Updates and Follow Mechanism

### 8.1 Following Templates

Users can click “Follow” on templates in the marketplace. When a new version of a template is released, all followers will receive a notification (displayed in the dashboard of their SimuCorp instance).

### 8.2 Version Updates and Local Synchronization

- When an uploader publishes a new version, semantic versioning must be followed and release notes must be attached.
- HR Managers following that template can see an update prompt in the Dashboard: “The template `iOS开发工程师` that you follow has released v1.3.0.”
- HR Manager can then evaluate whether to update the local Agent Soul (for example, when the original Agent’s performance declines).

## 9. Security and Governance

### 9.1 Uploader Reputation System

| Metric | Description |
|------|------|
| Total uploaded templates | Cumulative |
| Average template rating | Mean rating across all templates |
| Number of delistings | Cumulative delistings due to violations or quality issues |
| Number of official certifications | Number of templates that received certification |

Templates from high-reputation uploaders receive a slight boost in search ranking. If an uploader has been delisted more than 3 times, all subsequent submissions must go through mandatory manual review.

### 9.2 Reporting and Delisting

- Any user may report a template (security issues, severe quality mismatch, plagiarism).
- After confirmation through review:
  - Security issue: **Immediately delist**, and notify all users who downloaded the template (via dashboard alerts).
  - Quality issue: Mark as “Needs optimization” and lower its search ranking.
  - Plagiarism: Delist and warn the uploader.

All delisting actions are recorded in the security log of `health.json`.

## 10. Marketplace Page Design (New Frontend Additions)

### 10.1 Routes

- `/marketplace` — Marketplace homepage (bulletin board + recommended templates + statistics)
- `/marketplace/search` — Advanced search
- `/marketplace/:pkgId` — Template details
- `/marketplace/upload` — Upload template

### 10.2 Homepage Layout (Summary)

A top bulletin board (scrolling display), followed by marketplace statistics cards (total templates, new this week, number certified, active uploaders). Below that is a card grid for “Popular Templates This Week,” plus a “My Uploads” section and a compact “Marketplace Health” panel.

### 10.3 Template Details Page

Displays the template’s rating, downloads, certification badge, uploader information, core template attributes (role, runtime, skills, etc.), Soul preview (read-only, expandable), dependency information (MCP Server requirements, minimum version), and memory option details. Below that are the user review list and buttons for “Download Template,” “Instantiate Agent,” “Follow,” and “Report.”

## 11. Integration Points with the Existing Design

| Existing Module | Marketplace Integration |
|---------|-------------|
| HR Manager hiring workflow | Expand search scope to the external marketplace; add bulletin-board priority and local-priority logic |
| Template library management (`/settings/templates`) | Add an “Upload to Marketplace” button and an “Import from Marketplace” feature |
| Agent details page | If the Agent originates from a marketplace template, display “Source: Marketplace template pkg-xxxx” and provide follow/update entry points |
| Dashboard | Add a “Marketplace Updates” card (followed template updates, talent gap recommendations) |
| Audit log | Record template upload, download, and instantiation operations |

## 12. Implementation Roadmap

| Phase | Content |
|------|------|
| **Phase 3+** (core system stable) | Build the GitHub marketplace repository, define the standard template package format, and implement basic upload/download APIs |
| **Phase 4** | Implement the AI review pipeline, bulletin board, dependency and compatibility checks, and follow mechanism |
| **Phase 5** | Implement the evaluation system, talent gap analysis, marketplace health panel, and full frontend pages |
| **Phase 6** | Introduce the uploader reputation system, official certification workflow, and community governance rules (such as handling malicious reports) |

---

At this point, the Agent Marketplace is fully specified from concept to implementation detail. Together with the existing 18 design chapters, the white paper, and 9 supplementary documents, it forms a complete blueprint for SimuCorp from internal governance to external ecosystem development. This marketplace design can be delivered directly to development Agents for phased implementation after Phase 3.

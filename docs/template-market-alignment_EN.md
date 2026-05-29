# Template-Instance Model & Agent Marketplace Alignment Report

## I. Agent Template & Instance Model (agent_template_instance_model.md)

### Aligned

| Design Requirement                                                 | Implementation Status |
|--------------------------------------------------------------------|-----------------------|
| Template (`template_id`) → Instance (`agent_id`) inheritance relationship | ✅ `agents` table has `template_id` field, associated by HR during creation |
| `runtime_type` immutable after creation                            | ✅ `registerAgent` locks `runtime` |
| HR supports creating instances from templates                      | ✅ HR matches from template library + customizes creation |
| EA base awareness routing (code → Claude Code, coordination → OpenClaw) | ✅ `capabilityDirectory` added `runtimeBonus` |
| Memory isolated by `agent_id`                                      | ✅ Memory MCP separates collections by `agent_id` |

### Not Yet Implemented

| Design Requirement                                                      | Description |
|-------------------------------------------------------------------------|-------------|
| HR creates multiple instances with different base runtimes in a single recruitment | Currently creates only 1 instance; should allow creating 2-3 (different runtimes) |
| Independent nodes for the same template in the organizational topology    | Frontend `org` page supports this; instances just need to exist |

## II. Agent Marketplace Design (agent_marketplace_design.md)

Agent Marketplace is an open-source ecosystem across GitHub, distinct from current single-instance deployments. Alignment will occur in two phases:

### Current Alignable (P4-1)

| Design Requirement                  | Implementation Scheme |
|-------------------------------------|-----------------------|
| Standardized template package structure | Existing `agent_templates` table includes `suggested_runtime`/`model`/`tools`/`capabilities`/`internship_kpi`; add `dependencies` and `market_metadata` fields |
| Template provenance tracking (source) | Add `provenance` field to `agents` table (`source_agent_id`, `tasks_completed`, `success_rate`) |
| Semantic template versioning          | `souls` table already has `versionHistory`; format as semver |
| Upload prerequisite checks          | HR adds pre-validation (conversion rate ≥ 70% OR tasks ≥ 50 AND success rate ≥ 85%) |
| Security audit (local pre-check)    | AI scan Soul + sensitive info in memory before upload |

### Requires Independent Project (P4-2+)

| Design Requirement                  | Description |
|-------------------------------------|-------------|
| GitHub Marketplace Repository       | Separate `simu-corp/marketplace` repository with template JSON files |
| Marketplace API                     | Independent REST APIs for search/download/upload/evaluation |
| Announcement Wall + Talent Gap Analysis | Market data analysis service |
| Evaluation System + Ranking Algorithm | Weighting by score/download count/certification |
| Cross-instance template sync (follow + update notification) | Requires inter-Gateway communication |

## III. Recommended Implementation Order

### P4-1 (Immediate, 1-2h)
1. Complete `agent_templates` table fields (`dependencies`/`min_simu_corp_version`).
2. HR pre-validation for upload + security scan.
3. Template provenance tracking.

### P4-2 (Independent Project, Requires GitHub repo)
4. Create `simu-corp/marketplace` repository.
5. Marketplace API (search/download/upload/evaluate).
6. Announcement wall + health dashboard.
7. HR integration with market search.

## IV. Changes to Current System

Already completed:
- ✅ EA routing base awareness (d5501f6)
- ✅ Template ID association with agent instances

Pending immediate completion:
- ⬜ HR upload pre-validation
- ⬜ Template provenance tracking field
- ⬜ Security scan logic (sensitive info detection)

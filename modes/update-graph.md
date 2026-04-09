# Mode: update-graph — Orchestrated Profile Sync

Run all enabled ingestors to sync `person_profile/` data into Neo4j, then run quality checks and log results.

## When to Use

- `/career-ops update-graph` — runs all enabled ingestors
- Individual ingestors can also run standalone (e.g., `/career-ops sync-projects`)

## Prerequisites

- Neo4j MCP server connected
- `config/profile.yml` with `neo4j.person_name` and `neo4j.data_dir`
- `person_profile/ingestors.yml` exists with registered ingestors
- Graph initialized via `/career-ops onboard-graph` (Person node exists)

## Orchestrator Workflow

### Phase 1 — Setup

1. Read `config/profile.yml` → get `person_name` and `data_dir`
2. Read `person_profile/ingestors.yml` → list of registered ingestors
3. Verify Person node exists in Neo4j:
   ```cypher
   MATCH (p:Person {name: $person_name}) RETURN p.name
   ```
4. Read `person_profile/sync-log.md` (if exists) → check for unresolved suggestions from prior runs

### Phase 2 — Dispatch Ingestors

For each ingestor in `ingestors.yml` with `enabled: true`:

1. Read the ingestor's mode file (e.g., `modes/sync-projects.md`)
2. Dispatch a subagent:
   ```
   Agent(
     subagent_type="general-purpose",
     prompt="[content of ingestor mode file]\n\nScan directory: {scan_dir}\nPerson name: {person_name}\n\nExtract all entities and return the JSON envelope as specified in the mode file.",
     description="career-ops {ingestor.name}"
   )
   ```
3. Collect the returned JSON envelope
4. If the subagent returns an error or can't parse files, log the failure and continue to the next ingestor

### Phase 3 — Neo4j Loading

For each returned envelope, write to Neo4j:

1. **Create/update Skill nodes**: For each skill mentioned across all envelopes:
   ```cypher
   MERGE (s:Skill {name: $skill_name})
   SET s.category = $category
   ```
2. **Create/update entity nodes**: Based on `data[].type`:
   - `Project` → MERGE by name, SET properties, create USES_TECH → Skill, create Person-BUILT→Project
   - `Position` → MERGE by title + company_key, SET properties, create AT_COMPANY, WORKED_AT, USED_IN
   - `Course` → MERGE by name, SET properties, create AT_INSTITUTION, Person-COMPLETED
   - `Institution` → MERGE by name, SET properties, create Person-ATTENDED
   - `Certification` → MERGE by name+issuer, SET properties, create HOLDS_CERT, VALIDATES→Skill
   - `Strength` → MERGE by name, SET properties, create EVIDENCES links
   - `Application` → MERGE by materials_path, SET properties, create AT_COMPANY
3. **Link Person to new Skills**:
   ```cypher
   MATCH (p:Person {name: $person_name})-[:BUILT|WORKED_AT]->(e)-[:USES_TECH|USED_IN]-(s:Skill)
   WHERE NOT EXISTS((p)-[:HAS_SKILL]->(s))
   MERGE (p)-[:HAS_SKILL {proficiency: "proficient"}]->(s)
   ```
4. **Update proficiency levels** based on evidence density:
   - 2+ positions/projects → expert
   - 1 position/project → proficient
   - Cover letter mention only → familiar

### Phase 4 — Self-Improvement Pass

1. **Quality checks** — query Neo4j for issues:
   ```cypher
   // Projects with < 3 tech skills
   MATCH (p:Person {name: $person_name})-[:BUILT]->(proj:Project)
   WHERE count{(proj)-[:USES_TECH]->()} < 3
   RETURN "thin_project" AS issue, proj.name AS entity

   // Positions with no achievements
   MATCH (p:Person {name: $person_name})-[:WORKED_AT]->(pos:Position)
   WHERE pos.achievements IS NULL OR pos.achievements = ""
   RETURN "no_achievements" AS issue, pos.title AS entity

   // Orphaned skills
   MATCH (p:Person {name: $person_name})-[:HAS_SKILL]->(s:Skill)
   WHERE NOT EXISTS((s)<-[:USED_IN]-()) AND NOT EXISTS((s)<-[:USES_TECH]-()) AND NOT EXISTS((s)<-[:VALIDATES]-())
   RETURN "orphaned_skill" AS issue, s.name AS entity

   // Weak strengths
   MATCH (str:Strength) WHERE count{()-[:EVIDENCES]->(str)} < 2
   RETURN "weak_strength" AS issue, str.name AS entity
   ```
2. **Collect suggestions** from all ingestor envelopes
3. **Check for unresolved suggestions** from prior sync-log.md entries
4. **Re-derive Strengths** if new positions or projects were added

### Phase 5 — Logging and Reporting

1. **Count changes**: new nodes, updated nodes, unchanged, orphaned
2. **Append to `person_profile/sync-log.md`**:
   ```markdown
   ## {date} — {Full Sync | Incremental (ingestor-name only)}
   - **Ingestors:** {list of ingestors run}
   - **New:** {N} nodes ({breakdown by type})
   - **Updated:** {N} nodes
   - **Orphaned:** {N}
   - **Suggestions generated:** {N}
     - [ ] {suggestion 1}
     - [ ] {suggestion 2}
   - **Unresolved from prior syncs:** {N}
   - **Duration:** {N}s
   ```
3. **Present summary to user**:
   ```
   Sync Complete — {date}
   ━━━━━━━━━━━━━━━━━━━━━━━━━━
   Ingestors run: N/N
   New nodes: N  |  Updated: N  |  Unchanged: N  |  Orphaned: N

   Suggestions:
     ! {issue description}
     ...
     ✓ {all-clear checks}

   → Full details: person_profile/sync-log.md
   ```

## Standalone Ingestor Mode

When running an individual ingestor (e.g., `/career-ops sync-projects`):
1. Read config and dispatch only that one ingestor (Phases 1-3)
2. Skip Phase 4 self-improvement pass (only runs on full orchestrated sync)
3. Still log to sync-log.md (Phase 5)

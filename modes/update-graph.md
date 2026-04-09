# Mode: update-graph — Incremental Profile Update

Update the Neo4j knowledge graph with new or modified files from `person_profile/`. Use this after adding new documents (certs, transcripts, resumes, projects) to keep the graph current.

## Prerequisites

- Neo4j MCP server connected
- `config/profile.yml` with `neo4j.person_name`
- Graph already loaded via `/career-ops onboard-graph`

## Workflow

1. **Read config**: `config/profile.yml` → get `neo4j.person_name` and `neo4j.data_dir`

2. **Read profile-status.md**: Parse the "Loaded" and "Change Log" sections to understand what's already in the graph

3. **Scan for changes**: Compare `person_profile/` contents against the "Loaded" list:
   - New files not mentioned in profile-status.md
   - Files with modification dates after the last update
   - User can also specify: "I added new certs" or "update my job history"

4. **Process new/modified files only**: Same extraction logic as onboard-graph, but only for changed files. Use MERGE to avoid duplicates.

5. **Re-derive Strengths**: After loading new data, re-analyze the graph:
   - Query all evidence → recreate Strength nodes
   - Update proficiency levels based on new evidence density

6. **Regenerate Recommendations**: Query graph for:
   - Skills common in target JDs (from portals.yml title_filter) but missing evidence
   - Nodes with sparse properties
   - Expiring certifications
   - Cross-domain opportunities

7. **Update profile-status.md**:
   - Add new items to "Loaded" checklist
   - Clear items from "Pending" that were loaded
   - Refresh "Recommendations" section
   - Add entry to "Change Log"

8. **Report changes**:
   ```
   Graph Update Complete — {person_name}
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   New files processed: N
   Nodes created: N
   Nodes updated: N
   Relationships added: N
   Strengths re-derived: N

   → Profile status updated: person_profile/profile-status.md
   ```

# Mode: onboard-graph — Load Candidate Profile into Neo4j

Load the user's complete professional profile from `person_profile/` (or the directory specified in `config/profile.yml → neo4j.data_dir`) into Neo4j as a structured knowledge graph.

## Prerequisites

- Neo4j MCP server connected and running
- `config/profile.yml` exists with `neo4j.person_name` set
- `person_profile/` directory exists with source documents

## Workflow

1. **Read config**: `config/profile.yml` → get `neo4j.person_name` and `neo4j.data_dir`
2. **Check existing graph**: Query Neo4j for existing Person node
   ```cypher
   MATCH (p:Person {name: $person_name}) RETURN p
   ```
   If exists, warn user and ask: "Profile already loaded. Run /career-ops update-graph instead, or wipe and reload?"

3. **Create schema** (if first time):
   - Uniqueness constraints on Person.name, Company.name, Skill.name, Institution.name, Project.name
   - Indexes on Skill.category, Position.title, Certification.status
   - Create Person node

4. **Scan person_profile/ directory** and process each subdirectory:

   **a. Certifications** (`certs/` or any `.md` files with cert data):
   - Extract: cert name, issuer, status, dates
   - Create Certification nodes + HOLDS_CERT relationships
   - Create Skill nodes validated by each cert + VALIDATES relationships

   **b. Education** (`college/` subdirectory):
   - Read PDF transcripts (multimodal PDF reader handles scanned docs)
   - Read XLSX grade spreadsheets (via python3 + openpyxl)
   - Extract: institutions, courses, grades, credits, degrees
   - Create Institution nodes + ATTENDED relationships
   - Create Course nodes + COMPLETED and AT_INSTITUTION relationships

   **c. Job History** (`jobs/` subdirectory):
   - Read DOCX/PDF resumes and cover letters from each job folder
   - Extract: positions (title, company, dates, responsibilities, achievements)
   - Extract: skills mentioned per position
   - Create Company, Position, Skill nodes
   - Create relationships: WORKED_AT, AT_COMPANY, REQUIRED_SKILL, USED_IN, HAS_SKILL
   - Create Application nodes for each job folder with materials_path

   **d. Projects** (`github/` subdirectory):
   - Read README.md, CLAUDE.md, and code file imports
   - Extract: project name, description, tech stack, highlights
   - Create Project nodes + BUILT and USES_TECH relationships

5. **Derive Strengths**: Analyze the loaded graph for patterns:
   - Skills with high evidence density → technical strengths
   - Cross-domain skill combinations → unique strengths
   - Career trajectory patterns → leadership/growth strengths
   - Create Strength nodes + EVIDENCES relationships

6. **Set proficiency levels**: Query evidence density per skill:
   - 3+ positions/projects + active cert → expert
   - 1-2 positions/projects or coursework → proficient
   - Mentioned once or familiar → familiar

7. **Generate profile-status.md**: Query graph for statistics, write `person_profile/profile-status.md` with:
   - Summary counts
   - Loaded items checklist
   - AI-generated Recommendations (Gaps, Improvements, Suggestions)
   - Change log entry

8. **Report to user**:
   ```
   Graph Onboarding Complete — {person_name}
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   Positions loaded: N
   Companies: N
   Skills: N (N expert, N proficient, N familiar)
   Certifications: N (N active, N expired, N in-progress)
   Institutions: N
   Courses: N
   Projects: N
   Strengths derived: N

   Files processed: N/N (N skipped — list reasons)
   Profile status: person_profile/profile-status.md

   → Your graph is ready. Evaluations and CV generation will now
     use your structured profile for evidence-based matching.
   ```

## Entity Extraction Rules

- **Skills**: Canonicalize names (e.g., "Cisco IOS" not "IOS"). Categories: networking, programming, cloud, hardware, soft-skill, os, tools.
- **Positions**: Each unique title + company + date range = one Position node. Deduplicate across resumes.
- **Companies**: Deduplicate by canonical name (e.g., "UC Irvine" not "University of California, Irvine").
- **Courses**: From transcripts and grade spreadsheets. Link to institutions.
- **Strengths**: AI-derived from patterns, not directly from files.

## File Reading Strategy

| File type | Method | What to extract |
|-----------|--------|-----------------|
| PDF | Read tool (handles scanned) | Text content, tables |
| DOCX | Read tool or python3 -c "from docx import Document..." | Text paragraphs |
| XLSX | python3 + openpyxl | Cell data |
| Markdown | Read tool | Structured content |
| Code (.py, .ino) | Read tool | Imports, comments, tech used |

## Important

- Use MERGE (not CREATE) for all nodes to avoid duplicates
- All queries scoped by Person.name from profile.yml
- Process files sequentially to avoid Neo4j write conflicts
- If a file can't be read (corrupted, password-protected), log it and continue
- Never stop the entire onboarding for a single file failure

# Modular Neo4j Profile Sync Ingestors — Design Spec

**Date:** 2026-04-09
**Author:** Eric + Claude
**Status:** Approved

## Goal

Build a modular, orchestrated system of document ingestors that extract structured career data from `person_profile/` and load it into Neo4j. Each ingestor specializes in one document type (code repos, resumes, transcripts, certs, cover letters). The orchestrator (`update-graph`) dispatches them via subagents, writes results to Neo4j, runs a self-improvement pass, and logs everything to `sync-log.md`. New ingestors can be added by creating a mode file and registering it in `ingestors.yml`.

## Non-Goals

- Cloning repos from GitHub URLs (only scans local folders in `person_profile/github/`)
- Real-time file watching / auto-triggering (manual invocation via `/career-ops` commands)
- Replacing the existing `onboard-graph` mode (that handles first-time setup; this handles ongoing syncs)

---

## Section 1: Architecture — Orchestrator + Modular Ingestors

### Commands

```
/career-ops update-graph          ← orchestrator, calls all enabled ingestors
/career-ops sync-projects         ← individual ingestor (can also run standalone)
/career-ops sync-resumes          ← individual ingestor
/career-ops sync-transcripts      ← individual ingestor
/career-ops sync-certs            ← individual ingestor
/career-ops sync-cover-letters    ← individual ingestor
```

### Orchestrator Flow (`update-graph`)

1. Read `config/profile.yml` → get `neo4j.person_name` and `neo4j.data_dir`
2. Read `person_profile/ingestors.yml` → list of registered ingestors with enabled/disabled status
3. For each enabled ingestor:
   a. Read the ingestor's mode file (e.g., `modes/sync-projects.md`)
   b. Dispatch a subagent with the mode content + scan directory path
   c. Subagent reads files, extracts entities, returns structured JSON envelope
4. For each returned envelope, write to Neo4j:
   a. MERGE nodes (Project, Skill, Position, Company, Course, Institution, Certification, Strength)
   b. MERGE relationships (USES_TECH, BUILT, HAS_SKILL, WORKED_AT, etc.)
   c. All queries scoped by `Person.name` from profile.yml
5. Run self-improvement pass (see Section 3)
6. Append results to `person_profile/sync-log.md`
7. Present summary to user

### Standalone Ingestor Flow

When running an individual ingestor (e.g., `/career-ops sync-projects`):
1. Same as orchestrator steps 1-4, but only for that one ingestor
2. Skip self-improvement pass (only runs on full orchestrated sync)
3. Still logs to `sync-log.md`

### Why Subagents

- Keeps file reading (potentially large) out of the main context
- Each ingestor has specialized extraction logic — clean separation
- MCP tools (Neo4j writes) stay centralized in the orchestrator/parent
- Subagents return structured JSON — the parent never parses raw documents

---

## Section 2: Ingestor Registry

### `person_profile/ingestors.yml`

```yaml
# Ingestor registry for /career-ops update-graph
# Each ingestor scans a directory, extracts entities, and returns structured data.
# Add new ingestors by creating a mode file and adding an entry here.

ingestors:
  - name: sync-projects
    mode: modes/sync-projects.md
    scan_dir: "person_profile/github"
    enabled: true
    description: "GitHub repos → Project + Skill nodes"

  - name: sync-resumes
    mode: modes/sync-resumes.md
    scan_dir: "person_profile/jobs"
    enabled: true
    description: "DOCX/PDF resumes → Position + Company + Skill nodes"

  - name: sync-transcripts
    mode: modes/sync-transcripts.md
    scan_dir: "person_profile/college"
    enabled: true
    description: "PDF transcripts + XLSX grades → Course + Institution nodes"

  - name: sync-certs
    mode: modes/sync-certs.md
    scan_dir: "person_profile/certs"
    enabled: true
    description: "Cert docs → Certification + validated Skill nodes"

  - name: sync-cover-letters
    mode: modes/sync-cover-letters.md
    scan_dir: "person_profile/jobs"
    enabled: true
    description: "Cover letters → Strength signals + skill emphasis"
```

**Adding a new ingestor:**
1. Create `modes/sync-{name}.md` with extraction instructions
2. Add an entry to `ingestors.yml`
3. Run `/career-ops update-graph` — the new ingestor is automatically picked up

---

## Section 3: Ingestor Mode Files

Each ingestor mode file tells the subagent how to extract from its document type and what structured data to return.

### Standardized Return Envelope

Every ingestor subagent returns this JSON structure:

```json
{
  "ingestor": "sync-projects",
  "timestamp": "2026-04-09T14:30:00Z",
  "entities_found": 6,
  "data": [
    {
      "type": "Project",
      "name": "deprado",
      "properties": {
        "description": "...",
        "highlights": "...",
        "url": "https://github.com/grie1/deprado",
        "status": "active"
      },
      "skills": ["Python", "numpy", "pandas", "scikit-learn"],
      "relationships": [
        {"type": "USES_TECH", "target_type": "Skill", "target": "Python"},
        {"type": "USES_TECH", "target_type": "Skill", "target": "numpy"}
      ]
    }
  ],
  "suggestions": [
    "i2c_expander has no README — consider adding one for better extraction"
  ]
}
```

### Per-Ingestor Extraction Logic

#### sync-projects
- **Input:** `person_profile/github/` subdirectories
- **Reads:** README.md, CLAUDE.md, requirements.txt, package.json, platformio.ini, pyproject.toml, *.py (imports), *.ino (includes)
- **Extracts:** Project name, description, highlights (interview talking points), tech stack (from imports + configs), URL (from git remote), status, complexity assessment
- **Returns:** `Project` entities with `USES_TECH` → `Skill` relationships
- **Skill categorization:** Map libraries to categories (numpy → programming, Arduino → hardware, etc.)

#### sync-resumes
- **Input:** `person_profile/jobs/` subdirectories — DOCX and PDF files containing resumes
- **Reads:** Most recent resumes first (they have the most complete work history)
- **Extracts:** Position title, company, start/end dates, type (full-time/founder), responsibilities, achievements, skills used per position
- **Returns:** `Position` + `Company` entities with `WORKED_AT`, `AT_COMPANY`, `USED_IN` relationships
- **CRITICAL:** Only creates `WORKED_AT` for actual employment (W-2 jobs + ventures). Application target companies get `Application` nodes only, not `WORKED_AT`.
- **Dedup:** Same position across multiple resumes → one node (merge by title + company)

#### sync-transcripts
- **Input:** `person_profile/college/` subdirectories — PDF transcripts + XLSX grades
- **Reads:** PDF transcripts (multimodal reader), Grades.xlsx (openpyxl)
- **Extracts:** Institution name/type/location, courses (name, grade, credits, semester/year), degrees earned, GPA
- **Returns:** `Course` + `Institution` entities with `COMPLETED`, `AT_INSTITUTION`, `ATTENDED` relationships

#### sync-certs
- **Input:** `person_profile/certs/` + any `.md` files with cert data (e.g., `cisco_cert_status.md`)
- **Reads:** Markdown cert status files, PDF cert documents
- **Extracts:** Cert name, issuer, status (active/expired/in-progress), dates, skills validated
- **Returns:** `Certification` entities with `HOLDS_CERT`, `VALIDATES` → `Skill` relationships

#### sync-cover-letters
- **Input:** `person_profile/jobs/` subdirectories — DOCX and PDF files that are cover letters (not resumes)
- **Reads:** Cover letter files (identified by filename containing "cover" or by content structure)
- **Extracts:** Self-described strengths, skills emphasized for each target role, recurring self-framing patterns (e.g., "full-stack technologist", "lifelong learner")
- **Returns:** `Strength` updates (new evidence), `Skill` emphasis data
- **Note:** This ingestor enriches existing Strength nodes rather than creating primary entities

---

## Section 4: Self-Improvement Pass

After all ingestors complete in an orchestrated run, the orchestrator queries Neo4j for quality issues:

### Diff Detection
- Compare extracted entities vs. existing graph nodes
- Report: new nodes, updated nodes, unchanged, orphaned (in graph but source file deleted)

### Quality Checks

```cypher
// Projects with thin tech stacks (< 3 skills)
MATCH (p:Person {name: $person_name})-[:BUILT]->(proj:Project)
WHERE count{(proj)-[:USES_TECH]->()} < 3
RETURN proj.name

// Positions with no achievements
MATCH (p:Person {name: $person_name})-[:WORKED_AT]->(pos:Position)
WHERE pos.achievements IS NULL OR pos.achievements = ""
RETURN pos.title

// Orphaned skills (no USED_IN or USES_TECH links)
MATCH (p:Person {name: $person_name})-[:HAS_SKILL]->(s:Skill)
WHERE NOT EXISTS((s)<-[:USED_IN]-()) AND NOT EXISTS((s)<-[:USES_TECH]-()) AND NOT EXISTS((s)<-[:VALIDATES]-())
RETURN s.name

// Weak strengths (< 2 evidence links)
MATCH (str:Strength) WHERE count{()-[:EVIDENCES]->(str)} < 2
RETURN str.name
```

### Ingestor Improvement Suggestions

Each subagent includes `suggestions` in its return envelope. The orchestrator collects these and also generates its own based on the quality checks. All suggestions are presented to the user and logged.

### Presentation

```
Sync Complete — 2026-04-09
━━━━━━━━━━━━━━━━━━━━━━━━━━
Ingestors run: 5/5
New nodes: 12  |  Updated: 8  |  Unchanged: 47  |  Orphaned: 0

Suggestions:
  ! deprado project has no test coverage info — add for interview talking points
  ! 2 cover letters mention "Japanese language" — not captured as Skill yet. Add? (y/n)
  ✓ All projects have READMEs
  ✓ All positions have achievements

→ Full details: person_profile/sync-log.md
```

---

## Section 5: Sync Log

### `person_profile/sync-log.md`

Append-only changelog. Auto-maintained by the orchestrator.

```markdown
# Sync Log

## 2026-04-09 — Full Sync
- **Ingestors:** sync-projects, sync-resumes, sync-transcripts, sync-certs, sync-cover-letters
- **New:** 12 nodes (3 Project, 5 Skill, 4 Course)
- **Updated:** 8 nodes (descriptions enriched)
- **Orphaned:** 0
- **Suggestions generated:** 2
  - [ ] Add Japanese language as Skill (from cover letter evidence)
  - [ ] Add test coverage info to deprado project
- **Duration:** 45s

## 2026-04-15 — Incremental (sync-projects only)
- **Trigger:** New repo added to person_profile/github/
- **New:** 1 Project, 3 Skill
- **Updated:** 0
- **Suggestions:** 0
- **Duration:** 12s
```

Unchecked suggestion items carry forward — next sync reminds you about unresolved ones.

---

## Section 6: Routing + Documentation Updates

### Career-ops Skill Router (`.claude/skills/career-ops/SKILL.md`)

Add to the routing table:

```
| `sync-projects`      | `sync-projects`      |
| `sync-resumes`       | `sync-resumes`       |
| `sync-transcripts`   | `sync-transcripts`   |
| `sync-certs`         | `sync-certs`         |
| `sync-cover-letters` | `sync-cover-letters` |
```

Add to discovery menu under a new "Profile Sync" section:

```
Profile Sync:
  /career-ops update-graph        → Run all ingestors (full sync to Neo4j)
  /career-ops sync-projects       → Sync GitHub projects only
  /career-ops sync-resumes        → Sync resumes only
  /career-ops sync-transcripts    → Sync transcripts only
  /career-ops sync-certs          → Sync certifications only
  /career-ops sync-cover-letters  → Sync cover letters only
```

### CLAUDE.md Skill Modes Table

Add:

```
| Wants to sync GitHub projects to Neo4j       | `sync-projects`      |
| Wants to sync resumes to Neo4j               | `sync-resumes`       |
| Wants to sync transcripts to Neo4j           | `sync-transcripts`   |
| Wants to sync certifications to Neo4j        | `sync-certs`         |
| Wants to sync cover letters to Neo4j         | `sync-cover-letters` |
| Wants to sync all profile data               | `update-graph`       |
```

### README.md

Add section:

```markdown
## Keeping Your Profile Graph Updated

After adding new files to `person_profile/`:

  /career-ops update-graph          # Runs all ingestors (full sync)
  /career-ops sync-projects         # Just GitHub projects
  /career-ops sync-resumes          # Just resumes
  /career-ops sync-transcripts      # Just transcripts
  /career-ops sync-certs            # Just certifications
  /career-ops sync-cover-letters    # Just cover letters

Results logged to person_profile/sync-log.md.
To add a custom ingestor, create a mode file and add it to person_profile/ingestors.yml.
```

### docs/multi-user-setup.md

Add to the setup flow after step 5 (onboard-graph):

```markdown
6. **Keep profile updated:**
   After adding new documents, run `/career-ops update-graph` to sync changes to Neo4j.
   Or run individual sync commands for specific document types.
```

---

## Files Changed

| File | Action | Purpose |
|------|--------|---------|
| `person_profile/ingestors.yml` | Create | Registry of all ingestors |
| `person_profile/sync-log.md` | Create (auto) | Append-only sync changelog |
| `modes/sync-projects.md` | Create | Ingestor: GitHub repos → Project + Skill |
| `modes/sync-resumes.md` | Create | Ingestor: resumes → Position + Company + Skill |
| `modes/sync-transcripts.md` | Create | Ingestor: transcripts → Course + Institution |
| `modes/sync-certs.md` | Create | Ingestor: certs → Certification + Skill |
| `modes/sync-cover-letters.md` | Create | Ingestor: cover letters → Strength + Skill emphasis |
| `modes/update-graph.md` | Modify | Add orchestrator: read ingestors.yml, dispatch subagents, self-improvement, sync-log |
| `.claude/skills/career-ops/SKILL.md` | Modify | Add routing for 5 sync commands + discovery menu |
| `CLAUDE.md` | Modify | Add sync modes to skill modes table |
| `README.md` | Modify | Add "Keeping Your Profile Graph Updated" section |
| `docs/multi-user-setup.md` | Modify | Add sync commands to setup flow |

## Dependencies

- Neo4j MCP server (already connected)
- `config/profile.yml` with `neo4j.person_name` (already configured)
- `person_profile/` directory (already exists with data)
- `openpyxl` for XLSX reading (install if needed: `pip3 install openpyxl`)

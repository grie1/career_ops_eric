# Modular Neo4j Profile Sync Ingestors — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build 5 modular ingestors (projects, resumes, transcripts, certs, cover letters) orchestrated by `update-graph`, with self-improvement pass and sync logging.

**Architecture:** Each ingestor is a mode file (`modes/sync-*.md`) registered in `person_profile/ingestors.yml`. The orchestrator (`update-graph`) reads the registry, dispatches a subagent per ingestor, collects structured JSON envelopes, writes to Neo4j, runs quality checks, and logs to `sync-log.md`. Individual ingestors can also run standalone via `/career-ops sync-*`.

**Tech Stack:** Markdown (modes), YAML (registry), Neo4j (MCP), Cypher

---

## File Map

| File | Action | Responsibility |
|------|--------|----------------|
| `person_profile/ingestors.yml` | Create | Ingestor registry |
| `modes/sync-projects.md` | Create | GitHub repo extraction logic |
| `modes/sync-resumes.md` | Create | Resume extraction logic |
| `modes/sync-transcripts.md` | Create | Transcript extraction logic |
| `modes/sync-certs.md` | Create | Certification extraction logic |
| `modes/sync-cover-letters.md` | Create | Cover letter extraction logic |
| `modes/update-graph.md` | Rewrite | Orchestrator with ingestor dispatch + self-improvement |
| `.claude/skills/career-ops/SKILL.md` | Modify | Add 5 sync routes + discovery menu |
| `CLAUDE.md` | Modify | Add sync modes to skill modes table |
| `README.md` | Modify | Add "Keeping Your Profile Graph Updated" section |
| `docs/multi-user-setup.md` | Modify | Add sync commands to setup flow |

---

### Task 1: Create Ingestor Registry

**Files:**
- Create: `person_profile/ingestors.yml`

- [ ] **Step 1: Create the registry file**

Write `person_profile/ingestors.yml`:

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

Note: This file is inside gitignored `person_profile/`, so no commit needed.

---

### Task 2: Create sync-projects Ingestor Mode

**Files:**
- Create: `modes/sync-projects.md`

- [ ] **Step 1: Write the mode file**

Create `modes/sync-projects.md`:

```markdown
# Mode: sync-projects — GitHub Repos to Neo4j

Extract project metadata and tech stacks from local GitHub repos in `person_profile/github/` and return structured data for Neo4j loading.

## When to Use

- `/career-ops sync-projects` (standalone)
- Called by `/career-ops update-graph` (orchestrated)

## Input

- `scan_dir`: path to `person_profile/github/` (from ingestors.yml or profile.yml)
- `person_name`: from `config/profile.yml → neo4j.person_name`

## Extraction Workflow

For each subdirectory in `scan_dir`:

1. **Identify the project**: directory name = project name
2. **Read README.md or CLAUDE.md** (if exists): extract description, purpose, key highlights
3. **Read config files** (in priority order):
   - `requirements.txt` → Python dependencies
   - `pyproject.toml` → Python project metadata + deps
   - `package.json` → Node.js dependencies
   - `platformio.ini` → Arduino/embedded platform + libraries
   - `Cargo.toml` → Rust dependencies
   - `go.mod` → Go modules
4. **Scan source code imports** (first 50 lines of up to 10 files):
   - `.py` files: `import X` / `from X import Y`
   - `.ino` / `.cpp` / `.h` files: `#include <X.h>`
   - `.js` / `.ts` files: `require('X')` / `import X from 'Y'`
5. **Check for git remote**: `cat .git/config` → extract GitHub URL if present
6. **Assess complexity**: count files, LOC, number of modules/directories
7. **Generate interview highlights**: 2-3 sentences about what would impress a hiring manager

## Skill Categorization

Map extracted libraries/technologies to categories:
- **programming**: Python, numpy, pandas, scipy, scikit-learn, PyTorch, TensorFlow, C++, Rust, Go, JavaScript
- **hardware**: Arduino, i2c, SPI, BLE, IoT, sensors, OLED, embedded
- **cloud**: AWS, Azure, GCP, Docker, Kubernetes
- **tools**: Streamlit, Plotly, SQLite, Git, CI/CD
- **data**: ML, Neural Networks, LLM, NLP, Financial Data Analysis

Canonicalize names: "sklearn" → "scikit-learn", "torch" → "PyTorch", etc.

## Return Format

Return a JSON envelope:

```json
{
  "ingestor": "sync-projects",
  "timestamp": "ISO-8601",
  "entities_found": N,
  "data": [
    {
      "type": "Project",
      "name": "project-name",
      "properties": {
        "description": "What it does (2-3 sentences)",
        "highlights": "Interview talking points",
        "url": "https://github.com/user/repo",
        "status": "active|completed|archived",
        "complexity": "high|medium|simple",
        "loc": 1234
      },
      "skills": ["Python", "numpy", "pandas"],
      "skill_categories": {"Python": "programming", "numpy": "programming"}
    }
  ],
  "suggestions": [
    "project-x has no README — add one for better extraction",
    "project-y has no tests — mentioning test coverage strengthens interview answers"
  ]
}
```

## Important

- Never fabricate technologies — only report what you find in code/configs
- If a project directory is empty or has only git metadata, skip it and note in suggestions
- If README is absent, infer description from code comments and file structure
- Return suggestions for any quality issues found (missing README, no tests, thin descriptions)
```

- [ ] **Step 2: Commit**

```bash
git add modes/sync-projects.md
git commit -m "feat: add sync-projects ingestor mode"
```

---

### Task 3: Create sync-resumes Ingestor Mode

**Files:**
- Create: `modes/sync-resumes.md`

- [ ] **Step 1: Write the mode file**

Create `modes/sync-resumes.md`:

```markdown
# Mode: sync-resumes — Resumes to Neo4j

Extract work history, skills, and achievements from resume files in `person_profile/jobs/` and return structured data for Neo4j loading.

## When to Use

- `/career-ops sync-resumes` (standalone)
- Called by `/career-ops update-graph` (orchestrated)

## Input

- `scan_dir`: path to `person_profile/jobs/` (from ingestors.yml)
- `person_name`: from `config/profile.yml → neo4j.person_name`

## Extraction Workflow

1. **Identify resume files**: In each subdirectory of `scan_dir`, find files that are resumes (not cover letters). Heuristics:
   - Filename contains "resume" or "cv" (case-insensitive)
   - Filename does NOT contain "cover"
   - If ambiguous, read first paragraph — resumes start with contact info or summary; cover letters start with "Dear" or a date
2. **Read most recent resumes first**: Sort job folders by date in folder name (e.g., `2024-11-12`) descending. Read 3-4 most recent to get complete work history.
3. **Extract positions**: For each position in the Work Experience section:
   - Title (exact)
   - Company name (canonical)
   - Start date, end date
   - Type: "full-time" if standard employment, "founder" if own company/venture
   - Responsibilities (bullet points)
   - Achievements (bullet points with metrics)
   - Skills/technologies mentioned in context of this position
4. **Deduplicate**: Same position across multiple resumes → merge, keeping the richest description
5. **Extract skills inventory**: All unique technologies/tools mentioned across all resumes

## CRITICAL RULES

- **WORKED_AT is only for actual employment.** The `jobs/` folder contains mostly application materials, not employment history. Only create Position nodes with WORKED_AT for companies where the person actually worked.
- **Application folders** → create Application nodes with AT_COMPANY, not WORKED_AT.
- If unsure whether a folder represents actual employment or an application, check: does the work history section of resumes in OTHER folders list this company? If yes → employment. If only this folder's materials mention it → application.

## Return Format

```json
{
  "ingestor": "sync-resumes",
  "timestamp": "ISO-8601",
  "entities_found": N,
  "data": [
    {
      "type": "Position",
      "name": "Senior Network Engineer",
      "properties": {
        "title": "Senior Network Engineer",
        "company": "Jubitz Corporation",
        "start_date": "2022",
        "end_date": "present",
        "type": "full-time",
        "description": "...",
        "responsibilities": "...",
        "achievements": "..."
      },
      "skills": ["Cisco IOS", "MPLS", "VPN"],
      "skill_categories": {"Cisco IOS": "networking"}
    },
    {
      "type": "Application",
      "name": "OHSU Network Infrastructure Analyst (2024)",
      "properties": {
        "materials_path": "person_profile/jobs/2024-11-12 OHSU...",
        "date": "2024-11-12",
        "status": "applied",
        "target_company": "OHSU",
        "target_role": "Network Infrastructure Analyst"
      }
    }
  ],
  "suggestions": [
    "Position 'Endpoint Manager' at UCB Law has no metrics in achievements — adding numbers improves CV generation"
  ]
}
```
```

- [ ] **Step 2: Commit**

```bash
git add modes/sync-resumes.md
git commit -m "feat: add sync-resumes ingestor mode"
```

---

### Task 4: Create sync-transcripts Ingestor Mode

**Files:**
- Create: `modes/sync-transcripts.md`

- [ ] **Step 1: Write the mode file**

Create `modes/sync-transcripts.md`:

```markdown
# Mode: sync-transcripts — Academic Records to Neo4j

Extract courses, institutions, grades, and degrees from transcripts and grade spreadsheets in `person_profile/college/` and return structured data for Neo4j loading.

## When to Use

- `/career-ops sync-transcripts` (standalone)
- Called by `/career-ops update-graph` (orchestrated)

## Input

- `scan_dir`: path to `person_profile/college/` (from ingestors.yml)
- `person_name`: from `config/profile.yml → neo4j.person_name`

## Extraction Workflow

1. **Read Grades.xlsx first** (if exists): Use `python3 -c "import openpyxl; ..."` to extract all sheets. Each sheet typically = one institution.
2. **Read PDF transcripts**: For each subdirectory, read PDF files. Extract:
   - Institution name (canonical — e.g., "UC Berkeley" not "University of California, Berkeley")
   - Institution type: "university", "community-college", "extension", "online"
   - Location (city, state)
   - Courses: name, grade, credits, semester/year
   - Cumulative GPA
   - Degrees/certificates earned
3. **Focus on career-relevant courses**: Prioritize IT, networking, programming, science, and engineering courses. Include gen-ed courses for institutions where they contribute to GPA or degree.
4. **Skip empty directories** and note them.

## Return Format

```json
{
  "ingestor": "sync-transcripts",
  "timestamp": "ISO-8601",
  "entities_found": N,
  "data": [
    {
      "type": "Institution",
      "name": "City College of San Francisco",
      "properties": {
        "type": "community-college",
        "location": "San Francisco, CA"
      },
      "attendance": {
        "start_year": 2008,
        "end_year": 2014,
        "gpa": 3.82,
        "degree": null,
        "notes": "Strong IT/networking focus"
      },
      "courses": [
        {"name": "Network Fundamentals (CNIT 201E)", "grade": "A", "credits": 3.0, "semester": "Spring", "year": 2008}
      ]
    }
  ],
  "suggestions": [
    "Ohlone College directory is empty — remove or add transcripts",
    "Consider adding TEAS exam score (92%) as a certification node"
  ]
}
```
```

- [ ] **Step 2: Commit**

```bash
git add modes/sync-transcripts.md
git commit -m "feat: add sync-transcripts ingestor mode"
```

---

### Task 5: Create sync-certs Ingestor Mode

**Files:**
- Create: `modes/sync-certs.md`

- [ ] **Step 1: Write the mode file**

Create `modes/sync-certs.md`:

```markdown
# Mode: sync-certs — Certifications to Neo4j

Extract certification data from documents in `person_profile/certs/` (and any cert-related markdown files elsewhere in `person_profile/`) and return structured data for Neo4j loading.

## When to Use

- `/career-ops sync-certs` (standalone)
- Called by `/career-ops update-graph` (orchestrated)

## Input

- `scan_dir`: path to `person_profile/certs/` (from ingestors.yml)
- Also scan `person_profile/` root for cert-related `.md` files (e.g., `cisco_cert_status.md`)
- `person_name`: from `config/profile.yml → neo4j.person_name`

## Extraction Workflow

1. **Scan for cert files**: Look for `.md` files with cert status data, PDF certificates, and any files with "cert" in the name
2. **Extract per certification**:
   - Name (e.g., "CCNA", "AWS Solutions Architect")
   - Issuer (e.g., "Cisco", "Amazon")
   - Status: "active", "expired", "in-progress"
   - Date earned
   - Expiry date
   - Credential ID (if available)
3. **Map certs to skills they validate**: Each cert validates specific skills (e.g., CCNA → TCP/IP, Cisco IOS, Network Routing, Network Switching)

## Return Format

```json
{
  "ingestor": "sync-certs",
  "timestamp": "ISO-8601",
  "entities_found": N,
  "data": [
    {
      "type": "Certification",
      "name": "CCNA",
      "properties": {
        "issuer": "Cisco",
        "status": "active",
        "date_earned": "2024-05-29",
        "expiry_date": "2029-01-14"
      },
      "validates_skills": ["TCP/IP", "Cisco IOS", "Network Routing", "Network Switching"],
      "skill_categories": {"TCP/IP": "networking"}
    }
  ],
  "suggestions": [
    "CCNP Enterprise is in-progress — update status when exam is passed",
    "Consider adding AWS Cloud Practitioner cert (quick win based on existing AWS IoT experience)"
  ]
}
```
```

- [ ] **Step 2: Commit**

```bash
git add modes/sync-certs.md
git commit -m "feat: add sync-certs ingestor mode"
```

---

### Task 6: Create sync-cover-letters Ingestor Mode

**Files:**
- Create: `modes/sync-cover-letters.md`

- [ ] **Step 1: Write the mode file**

Create `modes/sync-cover-letters.md`:

```markdown
# Mode: sync-cover-letters — Cover Letters to Neo4j

Extract self-described strengths, emphasized skills, and recurring self-framing patterns from cover letters in `person_profile/jobs/` and return structured data for Neo4j loading.

## When to Use

- `/career-ops sync-cover-letters` (standalone)
- Called by `/career-ops update-graph` (orchestrated)

## Input

- `scan_dir`: path to `person_profile/jobs/` (from ingestors.yml)
- `person_name`: from `config/profile.yml → neo4j.person_name`

## Extraction Workflow

1. **Identify cover letter files**: In each subdirectory of `scan_dir`, find files that are cover letters (not resumes). Heuristics:
   - Filename contains "cover" (case-insensitive)
   - If ambiguous, read first paragraph — cover letters start with "Dear", a date, or an address block
2. **Read each cover letter** and extract:
   - **Self-described strengths**: Phrases like "I am a...", "My strength is...", "I excel at..."
   - **Skills emphasized**: What the person chose to highlight for this specific role
   - **Recurring patterns**: Across multiple cover letters, what themes repeat? (e.g., "lifelong learner", "full-stack technologist", "award-winning team builder")
   - **Target role framing**: How they position themselves differently for different role types
3. **Aggregate across all cover letters**: Find patterns that appear in 2+ letters — these are core identity claims
4. **Compare against existing Strength nodes**: Do the patterns match existing strengths? If new patterns found, suggest creating new Strength nodes.

## Return Format

```json
{
  "ingestor": "sync-cover-letters",
  "timestamp": "ISO-8601",
  "entities_found": N,
  "data": [
    {
      "type": "Strength",
      "name": "Full-Stack Technologist",
      "properties": {
        "category": "technical",
        "description": "Can take complex technology from hardware design to firmware to web/backend to sales — mentioned in 5 cover letters",
        "evidence_count": 5,
        "source_letters": ["Meraki TME", "OHSU 2024", "Unitus", "UCI 2016", "PSU"]
      }
    },
    {
      "type": "SkillEmphasis",
      "skill": "Team Leadership",
      "frequency": 8,
      "context": "Emphasized in 8/12 cover letters, always with 'award-winning' framing"
    }
  ],
  "suggestions": [
    "Japanese language mentioned in 2 cover letters but not captured as a Skill — add?",
    "Pattern 'I take everything apart to see how it works' appears 3 times — consider as Strength"
  ]
}
```

## Note

This ingestor enriches existing nodes rather than creating primary entities. It:
- Updates Strength nodes with cover letter evidence
- Identifies skill emphasis patterns (which skills Eric chooses to lead with)
- Surfaces new Strength candidates from recurring self-descriptions
```

- [ ] **Step 2: Commit**

```bash
git add modes/sync-cover-letters.md
git commit -m "feat: add sync-cover-letters ingestor mode"
```

---

### Task 7: Rewrite update-graph Mode as Orchestrator

**Files:**
- Modify: `modes/update-graph.md` (full rewrite)

- [ ] **Step 1: Rewrite the mode file**

Replace the entire contents of `modes/update-graph.md`:

```markdown
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
```

- [ ] **Step 2: Commit**

```bash
git add modes/update-graph.md
git commit -m "feat: rewrite update-graph as orchestrated ingestor dispatcher"
```

---

### Task 8: Update Career-Ops Skill Router

**Files:**
- Modify: `.claude/skills/career-ops/SKILL.md`

- [ ] **Step 1: Add sync routes to the routing table**

Read `.claude/skills/career-ops/SKILL.md`. In the routing table (after the `patterns` row, around line 31), add:

```
| `sync-projects`      | `sync-projects`      |
| `sync-resumes`       | `sync-resumes`       |
| `sync-transcripts`   | `sync-transcripts`   |
| `sync-certs`         | `sync-certs`         |
| `sync-cover-letters` | `sync-cover-letters` |
```

- [ ] **Step 2: Update the argument-hint in frontmatter**

In the frontmatter (line 7), update the argument-hint to include the new commands:

```yaml
argument-hint: "[scan | deep | pdf | oferta | ofertas | apply | batch | tracker | pipeline | contacto | training | project | interview-prep | update-graph | onboard-graph | sync-projects | sync-resumes | sync-transcripts | sync-certs | sync-cover-letters]"
```

- [ ] **Step 3: Add Profile Sync section to discovery menu**

In the discovery menu (the big code block around line 43-63), add before the closing ``` :

```
Profile Sync:
  /career-ops update-graph        → Run all ingestors (full sync to Neo4j)
  /career-ops onboard-graph       → First-time profile load to Neo4j
  /career-ops sync-projects       → Sync GitHub projects only
  /career-ops sync-resumes        → Sync resumes only
  /career-ops sync-transcripts    → Sync transcripts only
  /career-ops sync-certs          → Sync certifications only
  /career-ops sync-cover-letters  → Sync cover letters only
```

- [ ] **Step 4: Add sync modes to standalone modes list**

In the "### Standalone modes" section (line 80), add the sync modes:

```
Applies to: `tracker`, `deep`, `training`, `project`, `patterns`, `onboard-graph`, `update-graph`, `sync-projects`, `sync-resumes`, `sync-transcripts`, `sync-certs`, `sync-cover-letters`
```

- [ ] **Step 5: Commit**

```bash
git add .claude/skills/career-ops/SKILL.md
git commit -m "feat: add sync ingestor routes to career-ops skill router"
```

---

### Task 9: Update CLAUDE.md Skill Modes Table

**Files:**
- Modify: `CLAUDE.md`

- [ ] **Step 1: Add sync modes to the skill modes table**

Read `CLAUDE.md`. Find the "### Skill Modes" table (the table that maps user intents to modes). Add these rows at the end of the table:

```
| Wants to sync GitHub projects to Neo4j       | `sync-projects`      |
| Wants to sync resumes to Neo4j               | `sync-resumes`       |
| Wants to sync transcripts to Neo4j           | `sync-transcripts`   |
| Wants to sync certifications to Neo4j        | `sync-certs`         |
| Wants to sync cover letters to Neo4j         | `sync-cover-letters` |
| Wants to sync all profile data to Neo4j      | `update-graph`       |
| Wants to load profile into Neo4j first time  | `onboard-graph`      |
```

- [ ] **Step 2: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: add sync ingestor modes to CLAUDE.md skill modes table"
```

---

### Task 10: Update README and Multi-User Guide

**Files:**
- Modify: `README.md`
- Modify: `docs/multi-user-setup.md`

- [ ] **Step 1: Add "Keeping Your Profile Graph Updated" section to README.md**

Read `README.md`. Find the "### Updating the Graph" section (it currently just shows `/career-ops update-graph`). Replace it with:

```markdown
### Keeping Your Profile Graph Updated

After adding new files to `person_profile/`:

```
/career-ops update-graph          # Runs all ingestors (full sync)
/career-ops sync-projects         # Just GitHub projects
/career-ops sync-resumes          # Just resumes
/career-ops sync-transcripts      # Just transcripts
/career-ops sync-certs            # Just certifications
/career-ops sync-cover-letters    # Just cover letters
```

Results logged to `person_profile/sync-log.md`. To add a custom ingestor, create a mode file and register it in `person_profile/ingestors.yml`.
```

- [ ] **Step 2: Add sync commands to multi-user setup guide**

Read `docs/multi-user-setup.md`. After step 5 (the onboard-graph step), add:

```markdown
6. **Keep profile updated:**

   After adding new documents to `person_profile/`, run:

   ```
   /career-ops update-graph
   ```

   This dispatches all ingestors to sync your documents into Neo4j. You can also run individual sync commands:

   ```
   /career-ops sync-projects         # Just GitHub projects
   /career-ops sync-resumes          # Just resumes
   /career-ops sync-transcripts      # Just transcripts
   /career-ops sync-certs            # Just certifications
   /career-ops sync-cover-letters    # Just cover letters
   ```

   To add a custom ingestor, create a mode file in `modes/` and add an entry to `person_profile/ingestors.yml`.
```

Update the existing step 7 numbering accordingly.

- [ ] **Step 3: Commit**

```bash
git add README.md docs/multi-user-setup.md
git commit -m "docs: add sync ingestor commands to README and multi-user guide"
```

---

### Task 11: Verify Everything

- [ ] **Step 1: Validate YAML registry**

```bash
python3 -c "import yaml; data = yaml.safe_load(open('person_profile/ingestors.yml')); print(f'{len(data[\"ingestors\"])} ingestors registered'); [print(f'  - {i[\"name\"]}: {i[\"description\"]}') for i in data['ingestors']]"
```

Expected:
```
5 ingestors registered
  - sync-projects: GitHub repos → Project + Skill nodes
  - sync-resumes: DOCX/PDF resumes → Position + Company + Skill nodes
  - sync-transcripts: PDF transcripts + XLSX grades → Course + Institution nodes
  - sync-certs: Cert docs → Certification + validated Skill nodes
  - sync-cover-letters: Cover letters → Strength signals + skill emphasis
```

- [ ] **Step 2: Verify all mode files exist**

```bash
for f in sync-projects sync-resumes sync-transcripts sync-certs sync-cover-letters; do
  test -f "modes/$f.md" && echo "✓ modes/$f.md" || echo "✗ modes/$f.md MISSING"
done
```

Expected: All 5 show ✓.

- [ ] **Step 3: Verify routing**

```bash
grep -c "sync-" .claude/skills/career-ops/SKILL.md
```

Expected: 5+ matches (the 5 sync routes in the table).

- [ ] **Step 4: Push to personal remote**

```bash
git push personal main
```

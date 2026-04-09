# Neo4j Candidate Knowledge Graph — Design Spec

**Date:** 2026-04-09
**Author:** Eric + Claude
**Status:** Approved

## Goal

Load Eric's complete professional profile (work history, education, certifications, projects, skills) from `person_profile/` into Neo4j as a structured knowledge graph. Integrate it with career-ops so evaluations, scans, and CV generation query the graph for evidence-based candidate matching. Support multi-user via `Person` node namespacing so the same Neo4j instance serves multiple job searches (e.g., Eric + wife).

## Non-Goals

- Replacing `cv.md` or `config/profile.yml` — the graph supplements them, doesn't replace
- Building a UI for the graph — queries happen via career-ops modes and Cypher
- Real-time sync with external systems (LinkedIn, etc.) — manual file-based updates only

---

## Section 1: Neo4j Schema — Node Types

```
(:Person {
  name: STRING,          // "Eric Grismer" — serves as namespace root
  email: STRING,
  location: STRING,
  summary: STRING         // elevator pitch, auto-derived from graph
})

(:Company {
  name: STRING,           // canonical name
  industry: STRING,
  size: STRING,           // "startup"|"mid"|"enterprise"
  location: STRING,
  notes: STRING
})

(:Position {
  title: STRING,          // "Network Infrastructure Analyst"
  start_date: STRING,     // "2010" or "2010-06"
  end_date: STRING,       // "2016" or "present"
  type: STRING,           // "full-time"|"contract"|"internship"
  description: STRING,
  responsibilities: STRING,  // extracted from resumes/cover letters
  achievements: STRING       // bullet points with metrics if available
})

(:Skill {
  name: STRING,           // canonical: "Python", "Cisco IOS", "TCP/IP"
  category: STRING,       // "networking"|"programming"|"cloud"|"hardware"|"soft-skill"|"os"|"tools"
  proficiency: STRING     // "expert"|"proficient"|"familiar" — derived from evidence density
})

(:Project {
  name: STRING,           // "deprado", "DataLogger"
  description: STRING,
  url: STRING,            // GitHub URL if applicable
  start_date: STRING,
  status: STRING,         // "active"|"completed"|"archived"
  highlights: STRING      // key talking points for interviews
})

(:Course {
  name: STRING,
  grade: STRING,
  credits: FLOAT,
  semester: STRING,       // "Fall 2019"
  year: INTEGER
})

(:Institution {
  name: STRING,           // "UC Berkeley", "Cerritos College"
  type: STRING,           // "university"|"community-college"|"online"|"extension"
  location: STRING
})

(:Certification {
  name: STRING,           // "CCNA"
  issuer: STRING,         // "Cisco"
  status: STRING,         // "active"|"expired"|"in-progress"
  date_earned: STRING,
  expiry_date: STRING,
  credential_id: STRING
})

(:Strength {
  name: STRING,           // "Complex troubleshooting", "Cross-team communication"
  category: STRING,       // "technical"|"leadership"|"analytical"|"interpersonal"
  description: STRING
})

(:Application {
  date: STRING,
  status: STRING,         // "applied"|"interviewed"|"offered"|"rejected"|"archived"
  materials_path: STRING, // reference to folder in person_profile/jobs/
  notes: STRING
})
```

---

## Section 2: Relationships

```
// Work History
(:Person)-[:WORKED_AT {start_date, end_date}]->(:Position)
(:Position)-[:AT_COMPANY]->(:Company)
(:Position)-[:REQUIRED_SKILL]->(:Skill)
(:Position)-[:APPLIED_FOR]->(:Application)

// Skills
(:Person)-[:HAS_SKILL {proficiency, years}]->(:Skill)
(:Skill)-[:USED_IN]->(:Position)
(:Skill)-[:USED_IN]->(:Project)

// Education
(:Person)-[:ATTENDED {start_year, end_year, degree, gpa}]->(:Institution)
(:Person)-[:COMPLETED]->(:Course)
(:Course)-[:AT_INSTITUTION]->(:Institution)

// Certifications
(:Person)-[:HOLDS_CERT]->(:Certification)
(:Certification)-[:VALIDATES]->(:Skill)

// Projects
(:Person)-[:BUILT]->(:Project)
(:Project)-[:USES_TECH]->(:Skill)

// Evidence -> Strengths
(:Position)-[:EVIDENCES]->(:Strength)
(:Project)-[:EVIDENCES]->(:Strength)
(:Certification)-[:EVIDENCES]->(:Strength)

// Applications
(:Application)-[:FOR_POSITION]->(:Position)
(:Application)-[:AT_COMPANY]->(:Company)
```

---

## Section 3: Career-Ops Integration

The graph becomes a queryable backend for all career-ops modes. All queries are scoped by `Person.name` from `config/profile.yml → neo4j.person_name`.

### During job evaluation (`/career-ops oferta`)

Before scoring, query for skill evidence:

```cypher
// Given extracted skills from JD: ["TCP/IP", "Cisco", "Python", "AWS"]
MATCH (me:Person {name: $person_name})-[:HAS_SKILL]->(s:Skill)
WHERE s.name IN $jd_skills
OPTIONAL MATCH (s)<-[:USED_IN]-(pos:Position)
OPTIONAL MATCH (s)<-[:USES_TECH]-(proj:Project)
OPTIONAL MATCH (s)<-[:VALIDATES]-(cert:Certification)
RETURN s.name, s.proficiency,
       collect(DISTINCT pos.title) AS positions,
       collect(DISTINCT proj.name) AS projects,
       collect(DISTINCT cert.name) AS certs
```

This gives the evaluator concrete evidence per skill instead of scanning cv.md for keywords. The "Match con CV" score block uses this data for precise alignment scoring.

### During scan loops (multi-domain discovery)

Query skill clusters to generate targeted scans:

```cypher
MATCH (me:Person {name: $person_name})-[:HAS_SKILL]->(s:Skill)
WHERE s.proficiency IN ["expert", "proficient"]
RETURN s.category, collect(s.name) AS skills, count(s) AS depth
ORDER BY depth DESC
```

Then generate scan queries per cluster — networking skills produce network engineer queries, ML skills produce AI engineer queries, etc. This enables `/career-ops scan` to search multiple career domains automatically.

### During CV generation (`/career-ops pdf`)

Pull the most relevant experience for a target role:

```cypher
MATCH (me:Person {name: $person_name})-[:WORKED_AT]->(pos:Position)-[:REQUIRED_SKILL]->(s:Skill)
WHERE s.name IN $target_skills
WITH pos, collect(DISTINCT s.name) AS matched_skills
RETURN pos.title, pos.achievements, matched_skills
ORDER BY size(matched_skills) DESC
```

### During interview prep (`/career-ops interview-prep`)

Surface relevant projects and strengths:

```cypher
// Recent projects with talking points
MATCH (me:Person {name: $person_name})-[:BUILT]->(p:Project)
WHERE p.status = "active"
RETURN p.name, p.description, p.highlights,
       [(p)-[:USES_TECH]->(s:Skill) | s.name] AS tech_stack

// Strengths ranked by evidence
MATCH (me:Person {name: $person_name})-[*1..2]-(e)-[:EVIDENCES]->(str:Strength)
RETURN str.name, str.category, str.description, count(e) AS evidence_count
ORDER BY evidence_count DESC
```

### Mode documentation updates

These modes get a new instruction block: "Before evaluating/generating, query Neo4j for candidate evidence using the person_name from profile.yml":

| Mode file | What to add |
|-----------|-------------|
| `modes/_shared.md` | General instruction to query Neo4j for skill evidence |
| `modes/oferta.md` | Skill-match query before scoring block A (Match con CV) |
| `modes/scan.md` | Multi-domain scan from skill clusters |
| `modes/pdf.md` | Relevant experience query for CV tailoring |
| `modes/interview-prep.md` | Project + strength queries |

---

## Section 4: Multi-User Graph Namespacing

One Neo4j instance serves multiple job searches. The `Person` node is the namespace root — all queries scope by `Person.name`.

```cypher
// Eric's skills
MATCH (p:Person {name:"Eric Grismer"})-[:HAS_SKILL]->(s) RETURN s

// Wife's skills
MATCH (p:Person {name:"Jane Grismer"})-[:HAS_SKILL]->(s) RETURN s
```

**Shared nodes are intentional.** If both users worked at the same company or share a skill, those are the same nodes with different relationships. The graph naturally deduplicates.

**`config/profile.yml` additions:**

```yaml
neo4j:
  person_name: "Eric Grismer"    # graph namespace — must match Person.name
  data_dir: "person_profile"      # folder with source documents
```

---

## Section 5: Persistent Profile Directory

**Rename `eric_files/` to `person_profile/`** — a permanent, gitignored directory for personal documents.

### Directory structure

```
person_profile/
├── profile-status.md        # Tracks what's loaded, pending, checklist, recommendations
├── college/                  # Academic records (transcripts, degrees, certs)
├── jobs/                     # Employment materials (resumes, cover letters, interview prep)
├── github/                   # Project repos or exports
├── certs/                    # Certification documents
└── other/                    # Articles, references, volunteer work, etc.
```

### profile-status.md

Auto-maintained by the onboarding/update modes. Contains 5 sections:

```markdown
# Profile Knowledge Graph Status

## Person: Eric Grismer
## Last updated: 2026-04-09

## Loaded
- [x] Cisco certifications (CCNA, CCNP, Specialist)
- [x] College transcripts (18 institutions)
- [x] Job history (23 active + 20 archived positions)
- [x] GitHub projects (6 repos)
- [x] Grades spreadsheet

## Pending
- [ ] Add recent AWS training certificates
- [ ] Update CCNP status after exam

## Checklist — What to Add
Good candidate profiles include:
- [ ] All resumes/CVs (even old ones — they contain job history)
- [ ] Academic transcripts and degree certificates
- [ ] Professional certifications (active and expired)
- [ ] Project source code or READMEs
- [ ] Performance reviews or recommendation letters
- [ ] Training/course completion certificates
- [ ] Published articles, talks, or blog posts
- [ ] Awards or recognition documents
- [ ] Volunteer work documentation
- [ ] Portfolio or personal website exports

## Recommendations (Auto-Generated)
Last analyzed: 2026-04-09

### Gaps
- [AI analyzes graph for missing skills common in target JDs]

### Improvements
- [AI identifies nodes with sparse data that could be enriched]

### Suggestions
- [AI finds cross-domain opportunities or expiring certs]

## Change Log
| Date | What changed | Nodes affected |
|------|-------------|----------------|
| 2026-04-09 | Initial load | All |
```

---

## Section 6: Onboarding Flow

Integrated into the standard career-ops setup:

```
# 1. Clone and install
git clone https://github.com/santifer/career-ops.git
cd career-ops && npm install
npx playwright install chromium

# 2. Check setup
npm run doctor

# 3. Configure
cp config/profile.example.yml config/profile.yml   # Edit with your details + neo4j section
cp templates/portals.example.yml portals.yml

# 4. Add your CV + personal files
# Create cv.md with your CV in markdown
# Place documents in person_profile/ (resumes, transcripts, certs, projects)

# 5. Load profile into Neo4j            <-- NEW STEP
/career-ops onboard-graph
# Reads person_profile/, extracts entities, loads Neo4j, writes profile-status.md

# 6. Personalize with Claude
# "Change the archetypes to backend engineering roles"
# "Update my profile with this CV I'm pasting"

# 7. Start using — evaluations now query your graph automatically
# Paste a job URL or run /career-ops scan
```

### Two graph modes

**`/career-ops onboard-graph`** — Initial full load:
1. Read `config/profile.yml` for `neo4j.person_name` and `neo4j.data_dir`
2. Create `Person` node if it doesn't exist
3. Scan all files in `person_profile/`:
   - PDFs: Read with multimodal PDF reader, extract text
   - DOCX: Extract text via python-docx or direct read
   - XLSX: Read with pandas/openpyxl
   - Markdown: Read directly
   - Code files (.py, .ino, etc.): Read for project metadata
4. For each file, extract entities: positions, skills, companies, education, certs, projects
5. Create Neo4j nodes and relationships (MERGE to avoid duplicates)
6. Generate `profile-status.md` with loaded items, checklist, and AI recommendations
7. Report summary to user

**`/career-ops update-graph`** — Incremental update:
1. Read `profile-status.md` to see what's already loaded
2. Scan `person_profile/` for new or modified files (compare against change log)
3. Extract entities from new files only
4. MERGE new nodes/relationships into Neo4j
5. Re-run recommendations analysis against updated graph
6. Update `profile-status.md` with new items and refreshed recommendations
7. Report what changed

### File extraction strategy

| File type | Extraction method | What we extract |
|-----------|-------------------|-----------------|
| Resume/CV (DOCX/PDF) | Read + parse sections | Job titles, companies, dates, skills, achievements |
| Transcript (PDF) | Read + parse tables | Courses, grades, institutions, dates |
| Grades spreadsheet (XLSX) | Read cells | Courses, grades, credits, institutions |
| Certification doc (MD/PDF) | Read + parse | Cert names, status, dates, issuer |
| Project README (MD) | Read | Project name, description, tech stack |
| Code files (.py, .ino) | Read imports + comments | Technologies used, libraries |
| Cover letters (DOCX/PDF) | Read + parse | Skills emphasized, company targets, self-described strengths |

### Entity extraction rules

- **Skills**: Extracted from resume bullet points, project tech stacks, cert validations, and course topics. Canonicalized to a standard name (e.g., "Cisco IOS" not "IOS" or "Cisco Internetwork Operating System").
- **Positions**: Extracted from resumes. Each job title + company + date range = one Position node.
- **Companies**: Deduplicated by canonical name (e.g., "UC Irvine" not "University of California, Irvine" and "UCI").
- **Courses**: Extracted from transcripts and grades spreadsheet. Linked to institutions.
- **Strengths**: Derived by AI from patterns in achievements, project outcomes, and skill clusters. Not directly extracted from files.
- **Proficiency levels**: Derived from evidence density:
  - Expert: 3+ positions or projects using the skill, or active certification
  - Proficient: 1-2 positions/projects, or completed coursework
  - Familiar: Mentioned in cover letters or single project usage

---

## Section 7: Multi-User Setup (Updated)

**For Eric's personal repo:**

```bash
# Push customized project to personal repo
cd career-ops
git remote add personal git@github.com:grie1/career_ops_eric.git
git branch -M main
git push -u personal main
```

**For wife's copy:**

```bash
# 1. Clone Eric's customized repo
git clone git@github.com:grie1/career_ops_eric.git career-ops-wife
cd career-ops-wife

# 2. Create her own remote
git remote set-url origin git@github.com:grie1/career_ops_wife.git
git push -u origin main

# 3. Replace person_profile/ contents with her documents
rm -rf person_profile/*
# Add her resumes, transcripts, certs, etc.

# 4. Edit config/profile.yml
# - Change name, email, target roles
# - Set neo4j.person_name to her name
# - Change title_filter keywords in portals.yml

# 5. Load her profile into Neo4j
/career-ops onboard-graph

# 6. Standard onboarding
/career-ops
```

Both projects share the same Neo4j instance. Queries are scoped by `neo4j.person_name`.

---

## Files Changed

| File | Change |
|------|--------|
| `modes/onboard-graph.md` | New — graph onboarding mode (full load) |
| `modes/update-graph.md` | New — incremental graph update mode |
| `modes/_shared.md` | Add Neo4j query instructions for skill evidence |
| `modes/oferta.md` | Add skill-match query before scoring |
| `modes/scan.md` | Add multi-domain scan from skill clusters |
| `modes/pdf.md` | Add relevant experience query for CV tailoring |
| `modes/interview-prep.md` | Add project + strength queries |
| `config/profile.example.yml` | Add `neo4j` section template |
| `docs/multi-user-setup.md` | Update with graph onboarding + personal repo flow |
| `person_profile/profile-status.md` | New — auto-generated status tracking |
| `.gitignore` | Add `person_profile/` (personal data, never committed) |
| Neo4j | New schema: 10 node types, 13 relationship types |

## Dependencies

- Neo4j MCP server (already connected and running on separate server)
- `python-docx` or similar for DOCX extraction (install via pip if needed)
- `openpyxl` or `pandas` for XLSX extraction (install via pip if needed)
- No changes to existing career-ops scripts — integration is via mode documentation only

## Source Data

- `person_profile/` (renamed from `eric_files/`) — 474 files, 41MB
- Cisco cert status markdown
- 46 academic records across 18 institutions
- 196 employment documents across 43 positions
- 231 code/project files across 6 GitHub repos
- 1 grades spreadsheet (XLSX)

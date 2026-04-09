# Neo4j Candidate Knowledge Graph — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Load Eric's complete professional profile from `person_profile/` into Neo4j as a structured knowledge graph, and integrate it with career-ops modes for evidence-based evaluations, CV generation, and interview prep.

**Architecture:** Extract entities (positions, skills, companies, education, certs, projects) from 474 files across PDF/DOCX/XLSX/MD formats. Load into Neo4j via MCP tools (`write-cypher`). Update career-ops modes to query Neo4j before evaluations. Two new modes: `onboard-graph` (full load) and `update-graph` (incremental).

**Tech Stack:** Neo4j (MCP server), Cypher, Python (for DOCX/XLSX extraction), Markdown (modes), YAML (config)

---

## File Map

| File | Action | Responsibility |
|------|--------|----------------|
| `person_profile/` | Rename from `eric_files/` | Permanent personal data directory |
| `person_profile/profile-status.md` | Create | Tracks loaded state, checklist, recommendations |
| `.gitignore` | Modify | Add `person_profile/` |
| `config/profile.example.yml` | Modify | Add `neo4j` section template |
| `modes/onboard-graph.md` | Create | Full graph load mode |
| `modes/update-graph.md` | Create | Incremental graph update mode |
| `modes/_shared.md` | Modify | Add Neo4j query instructions |
| `modes/oferta.md` | Modify | Add skill-match query before scoring |
| `modes/scan.md` | Modify | Add multi-domain scan from skill clusters |
| `modes/pdf.md` | Modify | Add relevant experience query |
| `modes/interview-prep.md` | Modify | Add project + strength queries |
| `docs/multi-user-setup.md` | Modify | Update with graph onboarding + personal repo flow |
| Neo4j | Write | Schema: 10 node types, 13 relationship types |

---

### Task 1: Rename Directory + Gitignore + Config

**Files:**
- Rename: `eric_files/` → `person_profile/`
- Modify: `.gitignore`
- Modify: `config/profile.example.yml`

- [ ] **Step 1: Rename the directory**

```bash
mv eric_files person_profile
```

- [ ] **Step 2: Add person_profile/ to .gitignore**

Add after the `# User config and customization` section in `.gitignore`:

```
# Personal profile data (never committed)
person_profile/
eric_files/
```

- [ ] **Step 3: Add neo4j section to profile.example.yml**

Append to the end of `config/profile.example.yml`:

```yaml

# Neo4j candidate knowledge graph
# Used by /career-ops onboard-graph and /career-ops update-graph
neo4j:
  person_name: "Jane Smith"          # Must match Person node name in Neo4j
  data_dir: "person_profile"         # Folder with source documents
```

- [ ] **Step 4: Verify**

```bash
ls person_profile/  # Should show: Cisco, cisco_cert_status.md, College, github, jobs
git status person_profile/  # Should show nothing (gitignored)
```

- [ ] **Step 5: Commit**

```bash
git add .gitignore config/profile.example.yml
git commit -m "feat: add person_profile directory convention and neo4j config"
```

---

### Task 2: Create Neo4j Schema — Constraints and Indexes

**Files:**
- Neo4j (via `mcp__neo4j__write-cypher`)

- [ ] **Step 1: Create uniqueness constraints**

Run each of these as separate `write-cypher` calls:

```cypher
CREATE CONSTRAINT person_name IF NOT EXISTS FOR (p:Person) REQUIRE p.name IS UNIQUE
```

```cypher
CREATE CONSTRAINT company_name IF NOT EXISTS FOR (c:Company) REQUIRE c.name IS UNIQUE
```

```cypher
CREATE CONSTRAINT skill_name IF NOT EXISTS FOR (s:Skill) REQUIRE s.name IS UNIQUE
```

```cypher
CREATE CONSTRAINT institution_name IF NOT EXISTS FOR (i:Institution) REQUIRE i.name IS UNIQUE
```

```cypher
CREATE CONSTRAINT certification_name_issuer IF NOT EXISTS FOR (c:Certification) REQUIRE (c.name, c.issuer) IS UNIQUE
```

```cypher
CREATE CONSTRAINT project_name IF NOT EXISTS FOR (p:Project) REQUIRE p.name IS UNIQUE
```

- [ ] **Step 2: Create indexes for common queries**

```cypher
CREATE INDEX skill_category IF NOT EXISTS FOR (s:Skill) ON (s.category)
```

```cypher
CREATE INDEX position_title IF NOT EXISTS FOR (p:Position) ON (p.title)
```

```cypher
CREATE INDEX certification_status IF NOT EXISTS FOR (c:Certification) ON (c.status)
```

- [ ] **Step 3: Create the Person node for Eric**

```cypher
MERGE (p:Person {name: "Eric Grismer"})
SET p.email = "eric.grismer@gmail.com",
    p.location = "Remote / Open to SF, LA, OC, Hawaii"
RETURN p
```

- [ ] **Step 4: Verify schema**

```cypher
SHOW CONSTRAINTS
```

Expected: 6 constraints listed.

```cypher
MATCH (p:Person {name: "Eric Grismer"}) RETURN p
```

Expected: 1 node returned.

---

### Task 3: Load Certifications

**Files:**
- Read: `person_profile/cisco_cert_status.md`
- Neo4j: Create Certification nodes + relationships

- [ ] **Step 1: Read the cert file**

Read `person_profile/cisco_cert_status.md` to extract cert data. The file is tab-delimited with columns: Name, Abbreviation, Group, Status, Active date, Expire date.

- [ ] **Step 2: Create Certification nodes and link to Person**

```cypher
// CCNP Enterprise (In Progress)
MERGE (c:Certification {name: "CCNP Enterprise", issuer: "Cisco"})
SET c.status = "in-progress",
    c.date_earned = "",
    c.expiry_date = ""
WITH c
MATCH (p:Person {name: "Eric Grismer"})
MERGE (p)-[:HOLDS_CERT]->(c)
```

```cypher
// Cisco Certified Specialist - Enterprise Core (Active)
MERGE (c:Certification {name: "Cisco Certified Specialist - Enterprise Core", issuer: "Cisco"})
SET c.status = "active",
    c.date_earned = "2026-01-14",
    c.expiry_date = "2029-01-14"
WITH c
MATCH (p:Person {name: "Eric Grismer"})
MERGE (p)-[:HOLDS_CERT]->(c)
```

```cypher
// CCNA (Active)
MERGE (c:Certification {name: "CCNA", issuer: "Cisco"})
SET c.status = "active",
    c.date_earned = "2024-05-29",
    c.expiry_date = "2029-01-14"
WITH c
MATCH (p:Person {name: "Eric Grismer"})
MERGE (p)-[:HOLDS_CERT]->(c)
```

```cypher
// CCNA Routing and Switching (Expired)
MERGE (c:Certification {name: "CCNA Routing and Switching", issuer: "Cisco"})
SET c.status = "expired",
    c.date_earned = "2015-05-23",
    c.expiry_date = "2019-05-21"
WITH c
MATCH (p:Person {name: "Eric Grismer"})
MERGE (p)-[:HOLDS_CERT]->(c)
```

```cypher
// CCENT (Expired)
MERGE (c:Certification {name: "CCENT", issuer: "Cisco"})
SET c.status = "expired",
    c.date_earned = "2015-02-07",
    c.expiry_date = "2019-05-21"
WITH c
MATCH (p:Person {name: "Eric Grismer"})
MERGE (p)-[:HOLDS_CERT]->(c)
```

- [ ] **Step 3: Link certs to Skills they validate**

```cypher
// Create networking skills validated by certs
MERGE (s1:Skill {name: "Cisco IOS"}) SET s1.category = "networking"
MERGE (s2:Skill {name: "TCP/IP"}) SET s2.category = "networking"
MERGE (s3:Skill {name: "Network Routing"}) SET s3.category = "networking"
MERGE (s4:Skill {name: "Network Switching"}) SET s4.category = "networking"
MERGE (s5:Skill {name: "Network Security"}) SET s5.category = "networking"
MERGE (s6:Skill {name: "Enterprise Networking"}) SET s6.category = "networking"
MERGE (s7:Skill {name: "Network Troubleshooting"}) SET s7.category = "networking"

WITH s1, s2, s3, s4, s5, s6, s7
MATCH (ccna:Certification {name: "CCNA"})
MATCH (ccnp:Certification {name: "CCNP Enterprise"})
MATCH (spec:Certification {name: "Cisco Certified Specialist - Enterprise Core"})
MERGE (ccna)-[:VALIDATES]->(s1)
MERGE (ccna)-[:VALIDATES]->(s2)
MERGE (ccna)-[:VALIDATES]->(s3)
MERGE (ccna)-[:VALIDATES]->(s4)
MERGE (ccnp)-[:VALIDATES]->(s5)
MERGE (ccnp)-[:VALIDATES]->(s6)
MERGE (spec)-[:VALIDATES]->(s7)
MERGE (spec)-[:VALIDATES]->(s6)
```

- [ ] **Step 4: Link Person to Skills**

```cypher
MATCH (p:Person {name: "Eric Grismer"})
MATCH (s:Skill) WHERE s.category = "networking"
MERGE (p)-[:HAS_SKILL {proficiency: "expert"}]->(s)
```

- [ ] **Step 5: Verify**

```cypher
MATCH (p:Person {name: "Eric Grismer"})-[:HOLDS_CERT]->(c:Certification)
RETURN c.name, c.status, c.expiry_date
ORDER BY c.status
```

Expected: 5 certifications returned.

---

### Task 4: Load Education — Institutions and Courses

**Files:**
- Read: All files in `person_profile/College/` (46 files across 18 institution subdirectories)
- Read: `person_profile/College/Grades.xlsx` (grades spreadsheet)
- Neo4j: Create Institution, Course nodes + relationships

This is the largest extraction task. The subagent should:

- [ ] **Step 1: Read the Grades.xlsx spreadsheet**

```bash
python3 -c "
import openpyxl
wb = openpyxl.load_workbook('person_profile/College/Grades.xlsx')
for sheet in wb.sheetnames:
    ws = wb[sheet]
    print(f'=== {sheet} ===')
    for row in ws.iter_rows(values_only=True):
        print(row)
" 2>/dev/null || echo "openpyxl not installed, try: pip3 install openpyxl"
```

If openpyxl isn't available, install it: `pip3 install openpyxl`

- [ ] **Step 2: Read transcripts from each institution directory**

For each subdirectory in `person_profile/College/`, read the PDF transcripts. Extract:
- Institution name
- Courses taken (name, grade, credits, semester/year)
- Degree/certificate earned (if any)
- GPA (if listed)

Read the PDF files using the Read tool (which handles PDFs, including scanned ones).

- [ ] **Step 3: Create Institution nodes**

For each institution found, create a node. Example pattern (repeat for all ~18):

```cypher
MERGE (i:Institution {name: "UC Berkeley"})
SET i.type = "university", i.location = "Berkeley, CA"

MERGE (i:Institution {name: "UC Berkeley Extension"})
SET i.type = "extension", i.location = "Berkeley, CA"

MERGE (i:Institution {name: "Cerritos College"})
SET i.type = "community-college", i.location = "Norwalk, CA"

MERGE (i:Institution {name: "Cypress College"})
SET i.type = "community-college", i.location = "Cypress, CA"

MERGE (i:Institution {name: "De Anza College"})
SET i.type = "community-college", i.location = "Cupertino, CA"

MERGE (i:Institution {name: "Golden West College"})
SET i.type = "community-college", i.location = "Huntington Beach, CA"

MERGE (i:Institution {name: "Irvine Valley College"})
SET i.type = "community-college", i.location = "Irvine, CA"

MERGE (i:Institution {name: "LACC"})
SET i.type = "community-college", i.location = "Los Angeles, CA"

MERGE (i:Institution {name: "LA Southwest College"})
SET i.type = "community-college", i.location = "Los Angeles, CA"

MERGE (i:Institution {name: "Long Beach City College"})
SET i.type = "community-college", i.location = "Long Beach, CA"

MERGE (i:Institution {name: "Ohlone College"})
SET i.type = "community-college", i.location = "Fremont, CA"

MERGE (i:Institution {name: "Saddleback College"})
SET i.type = "community-college", i.location = "Mission Viejo, CA"

MERGE (i:Institution {name: "Santa Ana College"})
SET i.type = "community-college", i.location = "Santa Ana, CA"

MERGE (i:Institution {name: "Santa Monica College"})
SET i.type = "community-college", i.location = "Santa Monica, CA"

MERGE (i:Institution {name: "City College of San Francisco"})
SET i.type = "community-college", i.location = "San Francisco, CA"

MERGE (i:Institution {name: "West Los Angeles College"})
SET i.type = "community-college", i.location = "Culver City, CA"
```

- [ ] **Step 4: Create Person→ATTENDED relationships**

```cypher
MATCH (p:Person {name: "Eric Grismer"})
MATCH (i:Institution)
WHERE i.name IN ["UC Berkeley", "UC Berkeley Extension", "Cerritos College", "Cypress College",
  "De Anza College", "Golden West College", "Irvine Valley College", "LACC",
  "LA Southwest College", "Long Beach City College", "Ohlone College", "Saddleback College",
  "Santa Ana College", "Santa Monica College", "City College of San Francisco",
  "West Los Angeles College"]
MERGE (p)-[:ATTENDED]->(i)
```

- [ ] **Step 5: Create Course nodes from extracted data**

For each course extracted from transcripts and Grades.xlsx, create nodes. Example pattern:

```cypher
MERGE (c:Course {name: "Introduction to Networking"})
SET c.grade = "A", c.credits = 3.0, c.semester = "Fall", c.year = 2014
WITH c
MATCH (i:Institution {name: "Cerritos College"})
MERGE (c)-[:AT_INSTITUTION]->(i)
WITH c
MATCH (p:Person {name: "Eric Grismer"})
MERGE (p)-[:COMPLETED]->(c)
```

The actual course names, grades, credits, and institutions come from the transcript/spreadsheet data extracted in steps 1-2. The subagent must read all available transcripts and create a Course node for each course found.

- [ ] **Step 6: Verify**

```cypher
MATCH (p:Person {name: "Eric Grismer"})-[:ATTENDED]->(i:Institution)
RETURN i.name, i.type ORDER BY i.name
```

Expected: ~16 institutions.

```cypher
MATCH (p:Person {name: "Eric Grismer"})-[:COMPLETED]->(c:Course)-[:AT_INSTITUTION]->(i)
RETURN i.name, count(c) AS courses ORDER BY courses DESC
```

Expected: Course counts per institution.

---

### Task 5: Load Job History — Positions, Companies, Skills

**Files:**
- Read: All files in `person_profile/jobs/` (196 files across 23+ job folders + archived)
- Neo4j: Create Position, Company, Skill, Application nodes + relationships

This is the most complex extraction task. Each job folder contains resumes and cover letters that describe positions, skills, and achievements.

- [ ] **Step 1: Read resumes and cover letters from each job folder**

For each subdirectory in `person_profile/jobs/` (both active and archived):
1. Read the DOCX/PDF files (resumes first, then cover letters)
2. Extract from resumes:
   - **Position history**: title, company, start/end dates, responsibilities, achievements
   - **Skills mentioned**: tools, technologies, platforms, methodologies
3. Extract from cover letters:
   - **Target position**: the role being applied for
   - **Self-described strengths**: what Eric emphasizes about himself
   - **Skills highlighted**: what he chose to emphasize for this role

Start with the most recent folders (they have the most up-to-date work history) and work backwards. Deduplicate positions that appear across multiple resumes.

- [ ] **Step 2: Create Company nodes**

For each unique company found in the resumes, create a node. Example:

```cypher
MERGE (c:Company {name: "OHSU"})
SET c.industry = "Healthcare/Education", c.location = "Portland, OR"

MERGE (c:Company {name: "UC Irvine"})
SET c.industry = "Education", c.location = "Irvine, CA"

MERGE (c:Company {name: "UCLA Health"})
SET c.industry = "Healthcare", c.location = "Los Angeles, CA"
```

Repeat for all companies found (OHSU, UC Irvine, UCLA, UCLA Health, Meraki/Cisco, Microsoft, PSU, UCSF, Bigleaf Networks, Jubitz, LS Networks, Port of Portland, Davis Wright Tremaine, Unitus, Elemental/AWS, Oregon Public Defender, Haas, etc.).

- [ ] **Step 3: Create Position nodes and link to Companies**

For each position extracted from resumes:

```cypher
MERGE (pos:Position {title: "Network Infrastructure Analyst"})
SET pos.start_date = "2022",
    pos.end_date = "2024",
    pos.type = "full-time",
    pos.description = "...",
    pos.responsibilities = "...",
    pos.achievements = "..."
WITH pos
MATCH (c:Company {name: "OHSU"})
MERGE (pos)-[:AT_COMPANY]->(c)
WITH pos
MATCH (p:Person {name: "Eric Grismer"})
MERGE (p)-[:WORKED_AT]->(pos)
```

Repeat for all positions found. Use the actual extracted data for description, responsibilities, and achievements.

- [ ] **Step 4: Create Skill nodes from job history**

Extract all skills mentioned across resumes and create Skill nodes with categories:

```cypher
// Programming
MERGE (s:Skill {name: "Python"}) SET s.category = "programming"
MERGE (s:Skill {name: "PowerShell"}) SET s.category = "programming"
MERGE (s:Skill {name: "Bash"}) SET s.category = "programming"

// Cloud
MERGE (s:Skill {name: "AWS"}) SET s.category = "cloud"
MERGE (s:Skill {name: "Azure"}) SET s.category = "cloud"

// OS
MERGE (s:Skill {name: "Linux"}) SET s.category = "os"
MERGE (s:Skill {name: "Windows Server"}) SET s.category = "os"
MERGE (s:Skill {name: "macOS"}) SET s.category = "os"

// Tools
MERGE (s:Skill {name: "Active Directory"}) SET s.category = "tools"
MERGE (s:Skill {name: "SCCM"}) SET s.category = "tools"
MERGE (s:Skill {name: "VMware"}) SET s.category = "tools"

// Networking (additional to Task 3)
MERGE (s:Skill {name: "Wireless Networking"}) SET s.category = "networking"
MERGE (s:Skill {name: "VPN"}) SET s.category = "networking"
MERGE (s:Skill {name: "DNS"}) SET s.category = "networking"
MERGE (s:Skill {name: "DHCP"}) SET s.category = "networking"

// Soft skills
MERGE (s:Skill {name: "Team Leadership"}) SET s.category = "soft-skill"
MERGE (s:Skill {name: "Technical Documentation"}) SET s.category = "soft-skill"
```

The actual skills list will be much larger — the subagent extracts ALL skills mentioned across all resumes and creates nodes for each. Use MERGE to avoid duplicates.

- [ ] **Step 5: Link Skills to Positions**

For each position, link the skills that were used in that role:

```cypher
MATCH (pos:Position {title: "Network Infrastructure Analyst"})
MATCH (s:Skill) WHERE s.name IN ["Cisco IOS", "TCP/IP", "VPN", "Wireless Networking", "Python"]
MERGE (s)-[:USED_IN]->(pos)
```

Repeat for all positions with their respective skills.

- [ ] **Step 6: Link Person to all Skills with proficiency**

```cypher
// Expert: skills with 3+ positions or active certs
// Proficient: 1-2 positions
// Familiar: mentioned once in cover letter only
MATCH (p:Person {name: "Eric Grismer"})
MATCH (s:Skill)
WHERE NOT EXISTS((p)-[:HAS_SKILL]->(s))
OPTIONAL MATCH (s)<-[:USED_IN]-(pos:Position)<-[:WORKED_AT]-(p)
WITH p, s, count(pos) AS position_count
WHERE position_count > 0
MERGE (p)-[:HAS_SKILL {
  proficiency: CASE
    WHEN position_count >= 3 THEN "expert"
    WHEN position_count >= 1 THEN "proficient"
    ELSE "familiar"
  END
}]->(s)
```

- [ ] **Step 7: Create Application nodes**

For each job folder in `person_profile/jobs/`, create an Application node:

```cypher
MERGE (a:Application {materials_path: "person_profile/jobs/OHSU Network Infrastructure Analyst"})
SET a.date = "2024-11-12",
    a.status = "applied",
    a.notes = "Network Infrastructure Analyst position"
WITH a
MATCH (c:Company {name: "OHSU"})
MERGE (a)-[:AT_COMPANY]->(c)
```

Repeat for all job folders.

- [ ] **Step 8: Verify**

```cypher
MATCH (p:Person {name: "Eric Grismer"})-[:WORKED_AT]->(pos:Position)-[:AT_COMPANY]->(c:Company)
RETURN c.name, pos.title, pos.start_date, pos.end_date
ORDER BY pos.start_date DESC
```

```cypher
MATCH (p:Person {name: "Eric Grismer"})-[:HAS_SKILL]->(s:Skill)
RETURN s.category, count(s) AS count, collect(s.name) AS skills
ORDER BY count DESC
```

---

### Task 6: Load GitHub Projects

**Files:**
- Read: All files in `person_profile/github/` (6 project directories)
- Neo4j: Create Project nodes + USES_TECH relationships

- [ ] **Step 1: Read project READMEs and code files**

For each project in `person_profile/github/`:
1. Read README.md or CLAUDE.md for description and highlights
2. Scan code files for imports/libraries to identify technologies
3. Extract: project name, description, tech stack, status, key highlights

Projects to process:
- `deprado` — ML for Asset Managers (Python, pandas, sklearn, scipy)
- `autoresearch` — Autonomous AI agent research (Python, GPU training, neural networks)
- `arduino` — IoT sensor projects (Arduino C++, i2c, BLE, CO2 sensors)
- `DataLogger` — CO2/temperature data logging (Python, sensors)
- `i2c_expander` — GPIO expansion testing (Python, i2c protocol)
- `grie1.github.io` — Personal portfolio site (HTML, CSS)

- [ ] **Step 2: Create Project nodes**

```cypher
MERGE (proj:Project {name: "deprado"})
SET proj.description = "Implementation of Marcos Lopez de Prado ML techniques for financial data analysis",
    proj.url = "https://github.com/grie1/deprado",
    proj.status = "active",
    proj.highlights = "Marcenko-Pastur denoising, NCO portfolio optimization, trend scanning labels, feature importance (MDI/MDA)"
WITH proj
MATCH (p:Person {name: "Eric Grismer"})
MERGE (p)-[:BUILT]->(proj)
```

```cypher
MERGE (proj:Project {name: "autoresearch"})
SET proj.description = "Autonomous AI agent system for LLM training experiments — agents modify and train models, check improvements every 15 minutes",
    proj.url = "https://github.com/grie1/autoresearch",
    proj.status = "active",
    proj.highlights = "Autonomous training loop, Karpathy nanochat implementation, GPU training orchestration"
WITH proj
MATCH (p:Person {name: "Eric Grismer"})
MERGE (p)-[:BUILT]->(proj)
```

```cypher
MERGE (proj:Project {name: "Arduino IoT Sensors"})
SET proj.description = "IoT sensor projects: CO2 monitoring, temperature logging, BLE communication, i2c expansion",
    proj.status = "active",
    proj.highlights = "CO2 manufacturing board firmware, multiplexer addressing, WiFi outlet GPIO control, real-time data logging"
WITH proj
MATCH (p:Person {name: "Eric Grismer"})
MERGE (p)-[:BUILT]->(proj)
```

```cypher
MERGE (proj:Project {name: "DataLogger"})
SET proj.description = "CO2 and temperature sensor datalogger for environmental monitoring",
    proj.url = "https://github.com/grie1/DataLogger",
    proj.status = "completed",
    proj.highlights = "Continuous environmental data collection, Python data acquisition"
WITH proj
MATCH (p:Person {name: "Eric Grismer"})
MERGE (p)-[:BUILT]->(proj)
```

```cypher
MERGE (proj:Project {name: "i2c Expander"})
SET proj.description = "Test scripts for PI4IOE5V6408 i2c expander IC — LED control and GPIO expansion",
    proj.url = "https://github.com/grie1/i2c_expander",
    proj.status = "completed",
    proj.highlights = "i2c protocol implementation, embedded systems testing"
WITH proj
MATCH (p:Person {name: "Eric Grismer"})
MERGE (p)-[:BUILT]->(proj)
```

```cypher
MERGE (proj:Project {name: "Personal Portfolio Site"})
SET proj.description = "GitHub Pages personal website and project showcase",
    proj.url = "https://grie1.github.io",
    proj.status = "active",
    proj.highlights = "Static site, project showcase"
WITH proj
MATCH (p:Person {name: "Eric Grismer"})
MERGE (p)-[:BUILT]->(proj)
```

- [ ] **Step 3: Create project tech skills and link**

```cypher
// Skills for deprado
MERGE (s1:Skill {name: "Machine Learning"}) SET s1.category = "programming"
MERGE (s2:Skill {name: "pandas"}) SET s2.category = "programming"
MERGE (s3:Skill {name: "scikit-learn"}) SET s3.category = "programming"
MERGE (s4:Skill {name: "scipy"}) SET s4.category = "programming"
MERGE (s5:Skill {name: "Financial Data Analysis"}) SET s5.category = "tools"
MATCH (proj:Project {name: "deprado"})
MERGE (proj)-[:USES_TECH]->(s1)
MERGE (proj)-[:USES_TECH]->(s2)
MERGE (proj)-[:USES_TECH]->(s3)
MERGE (proj)-[:USES_TECH]->(s4)
MERGE (proj)-[:USES_TECH]->(s5)
```

```cypher
// Skills for autoresearch
MERGE (s1:Skill {name: "Neural Networks"}) SET s1.category = "programming"
MERGE (s2:Skill {name: "GPU Training"}) SET s2.category = "tools"
MERGE (s3:Skill {name: "LLM"}) SET s3.category = "programming"
MERGE (s4:Skill {name: "AI Agents"}) SET s4.category = "programming"
MATCH (proj:Project {name: "autoresearch"})
MERGE (proj)-[:USES_TECH]->(s1)
MERGE (proj)-[:USES_TECH]->(s2)
MERGE (proj)-[:USES_TECH]->(s3)
MERGE (proj)-[:USES_TECH]->(s4)
```

```cypher
// Skills for Arduino/IoT
MERGE (s1:Skill {name: "Arduino"}) SET s1.category = "hardware"
MERGE (s2:Skill {name: "C++"}) SET s2.category = "programming"
MERGE (s3:Skill {name: "IoT"}) SET s3.category = "hardware"
MERGE (s4:Skill {name: "i2c"}) SET s4.category = "hardware"
MERGE (s5:Skill {name: "BLE"}) SET s5.category = "hardware"
MERGE (s6:Skill {name: "Sensor Integration"}) SET s6.category = "hardware"
MATCH (proj:Project {name: "Arduino IoT Sensors"})
MERGE (proj)-[:USES_TECH]->(s1)
MERGE (proj)-[:USES_TECH]->(s2)
MERGE (proj)-[:USES_TECH]->(s3)
MERGE (proj)-[:USES_TECH]->(s4)
MERGE (proj)-[:USES_TECH]->(s5)
MERGE (proj)-[:USES_TECH]->(s6)
```

```cypher
// Skills for DataLogger
MATCH (proj:Project {name: "DataLogger"})
MATCH (s:Skill {name: "Python"})
MERGE (proj)-[:USES_TECH]->(s)
WITH proj
MATCH (s2:Skill {name: "Sensor Integration"})
MERGE (proj)-[:USES_TECH]->(s2)
```

```cypher
// Skills for i2c Expander
MATCH (proj:Project {name: "i2c Expander"})
MATCH (s1:Skill {name: "Python"})
MATCH (s2:Skill {name: "i2c"})
MERGE (proj)-[:USES_TECH]->(s1)
MERGE (proj)-[:USES_TECH]->(s2)
```

```cypher
// Skills for portfolio site
MERGE (s1:Skill {name: "HTML/CSS"}) SET s1.category = "programming"
MATCH (proj:Project {name: "Personal Portfolio Site"})
MERGE (proj)-[:USES_TECH]->(s1)
```

- [ ] **Step 4: Link Person to project skills**

```cypher
MATCH (p:Person {name: "Eric Grismer"})-[:BUILT]->(proj:Project)-[:USES_TECH]->(s:Skill)
WHERE NOT EXISTS((p)-[:HAS_SKILL]->(s))
MERGE (p)-[:HAS_SKILL {proficiency: "proficient"}]->(s)
```

- [ ] **Step 5: Verify**

```cypher
MATCH (p:Person {name: "Eric Grismer"})-[:BUILT]->(proj:Project)-[:USES_TECH]->(s:Skill)
RETURN proj.name, collect(s.name) AS tech_stack
ORDER BY proj.name
```

Expected: 6 projects with their tech stacks.

---

### Task 7: Derive Strengths from Evidence

**Files:**
- Neo4j: Create Strength nodes + EVIDENCES relationships

- [ ] **Step 1: Analyze the graph for patterns and create Strength nodes**

After all data is loaded, the AI examines the graph to derive strengths from evidence patterns:

```cypher
// Create Strength nodes based on career patterns
MERGE (str:Strength {name: "Enterprise Network Infrastructure"})
SET str.category = "technical",
    str.description = "15+ years managing complex network environments across UC system, healthcare, and enterprise orgs"

MERGE (str:Strength {name: "Cisco Ecosystem Expertise"})
SET str.category = "technical",
    str.description = "Active CCNA + Specialist certs, CCNP in progress. Hands-on with IOS, routing, switching, security across multiple positions"

MERGE (str:Strength {name: "Cross-Platform IT Operations"})
SET str.category = "technical",
    str.description = "Windows, Linux, macOS administration across enterprise environments. AD, SCCM, VMware, cloud platforms"

MERGE (str:Strength {name: "Self-Directed Technical Learning"})
SET str.category = "analytical",
    str.description = "Coursework across 16+ institutions, active cert maintenance, and personal ML/IoT projects demonstrate continuous learning drive"

MERGE (str:Strength {name: "Applied Programming for Infrastructure"})
SET str.category = "technical",
    str.description = "Python, PowerShell, Bash for automation and tooling. ML projects and IoT firmware development show range beyond scripting"

MERGE (str:Strength {name: "Healthcare/Education IT"})
SET str.category = "technical",
    str.description = "Deep domain knowledge in university and healthcare IT environments — OHSU, UCLA Health, UC Irvine, UCSF"

MERGE (str:Strength {name: "Hardware + Software Integration"})
SET str.category = "technical",
    str.description = "Arduino IoT projects + network infrastructure = ability to work across the full stack from sensors to enterprise networks"
```

- [ ] **Step 2: Link Strengths to evidence**

```cypher
// Enterprise Network Infrastructure — evidenced by networking positions
MATCH (str:Strength {name: "Enterprise Network Infrastructure"})
MATCH (pos:Position) WHERE pos.title CONTAINS "Network" OR pos.title CONTAINS "Infrastructure"
MERGE (pos)-[:EVIDENCES]->(str)
```

```cypher
// Cisco Ecosystem — evidenced by certs
MATCH (str:Strength {name: "Cisco Ecosystem Expertise"})
MATCH (cert:Certification {issuer: "Cisco"})
MERGE (cert)-[:EVIDENCES]->(str)
```

```cypher
// Self-Directed Learning — evidenced by education breadth and projects
MATCH (str:Strength {name: "Self-Directed Technical Learning"})
MATCH (proj:Project) WHERE proj.name IN ["deprado", "autoresearch"]
MERGE (proj)-[:EVIDENCES]->(str)
```

```cypher
// Applied Programming — evidenced by code projects
MATCH (str:Strength {name: "Applied Programming for Infrastructure"})
MATCH (proj:Project) WHERE proj.name IN ["deprado", "autoresearch", "DataLogger", "i2c Expander"]
MERGE (proj)-[:EVIDENCES]->(str)
```

```cypher
// Hardware + Software — evidenced by IoT projects
MATCH (str:Strength {name: "Hardware + Software Integration"})
MATCH (proj:Project) WHERE proj.name IN ["Arduino IoT Sensors", "DataLogger", "i2c Expander"]
MERGE (proj)-[:EVIDENCES]->(str)
```

- [ ] **Step 3: Verify**

```cypher
MATCH (str:Strength)<-[:EVIDENCES]-(e)
RETURN str.name, str.category, count(e) AS evidence_count,
       labels(e)[0] AS evidence_type, collect(
         CASE WHEN e:Position THEN e.title
              WHEN e:Project THEN e.name
              WHEN e:Certification THEN e.name
         END
       ) AS evidence
ORDER BY evidence_count DESC
```

---

### Task 8: Generate profile-status.md

**Files:**
- Create: `person_profile/profile-status.md`

- [ ] **Step 1: Query graph for summary statistics**

```cypher
// Count all node types for Eric
MATCH (p:Person {name: "Eric Grismer"})
OPTIONAL MATCH (p)-[:WORKED_AT]->(pos:Position)
OPTIONAL MATCH (p)-[:HAS_SKILL]->(s:Skill)
OPTIONAL MATCH (p)-[:HOLDS_CERT]->(c:Certification)
OPTIONAL MATCH (p)-[:ATTENDED]->(i:Institution)
OPTIONAL MATCH (p)-[:COMPLETED]->(course:Course)
OPTIONAL MATCH (p)-[:BUILT]->(proj:Project)
RETURN count(DISTINCT pos) AS positions,
       count(DISTINCT s) AS skills,
       count(DISTINCT c) AS certs,
       count(DISTINCT i) AS institutions,
       count(DISTINCT course) AS courses,
       count(DISTINCT proj) AS projects
```

- [ ] **Step 2: Query for recommendations data**

```cypher
// Skills with low evidence (familiar only)
MATCH (p:Person {name: "Eric Grismer"})-[r:HAS_SKILL]->(s:Skill)
WHERE r.proficiency = "familiar"
RETURN s.name, s.category
```

```cypher
// Positions with no achievements listed
MATCH (p:Person {name: "Eric Grismer"})-[:WORKED_AT]->(pos:Position)
WHERE pos.achievements IS NULL OR pos.achievements = ""
RETURN pos.title
```

```cypher
// Certs expiring within 2 years
MATCH (c:Certification {status: "active"})
WHERE c.expiry_date <= "2028-04-09"
RETURN c.name, c.expiry_date
```

- [ ] **Step 3: Write profile-status.md**

Create `person_profile/profile-status.md` with the queried data. The file should follow this exact structure:

```markdown
# Profile Knowledge Graph Status

## Person: Eric Grismer
## Last updated: 2026-04-09

## Summary
- Positions: [N from query]
- Skills: [N from query]
- Certifications: [N from query]
- Institutions: [N from query]
- Courses: [N from query]
- Projects: [N from query]
- Strengths: [N from query]

## Loaded
- [x] Cisco certifications (5 certs — 2 active, 1 in-progress, 2 expired)
- [x] College transcripts ([N] institutions)
- [x] Job history ([N] positions across [N] companies)
- [x] GitHub projects (6 repos)
- [x] Grades spreadsheet
- [x] Strengths derived from evidence patterns

## Pending
- [ ] (none — initial load complete)

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
[AI generates specific gaps based on graph analysis — missing skills common in target JDs, no cloud certs, etc.]

### Improvements
[AI identifies nodes with sparse data — positions missing achievements, projects without descriptions, etc.]

### Suggestions
[AI finds cross-domain opportunities — skills combinations that unlock new role types, expiring certs, etc.]

## Change Log
| Date | What changed | Nodes affected |
|------|-------------|----------------|
| 2026-04-09 | Initial full load from person_profile/ | All |
```

Fill in all `[N]` placeholders and all Recommendations sections with actual data from the graph queries.

- [ ] **Step 4: Commit (only the gitignore and config changes — person_profile/ is gitignored)**

No commit needed for profile-status.md since it's inside the gitignored person_profile/ directory.

---

### Task 9: Create onboard-graph Mode

**Files:**
- Create: `modes/onboard-graph.md`

- [ ] **Step 1: Write the mode file**

Create `modes/onboard-graph.md`:

```markdown
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
```

- [ ] **Step 2: Commit**

```bash
git add modes/onboard-graph.md
git commit -m "feat: add onboard-graph mode for Neo4j profile loading"
```

---

### Task 10: Create update-graph Mode

**Files:**
- Create: `modes/update-graph.md`

- [ ] **Step 1: Write the mode file**

Create `modes/update-graph.md`:

```markdown
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
```

- [ ] **Step 2: Commit**

```bash
git add modes/update-graph.md
git commit -m "feat: add update-graph mode for incremental Neo4j updates"
```

---

### Task 11: Update Career-Ops Modes with Neo4j Integration

**Files:**
- Modify: `modes/_shared.md` (add Neo4j query instructions)
- Modify: `modes/oferta.md` (add skill-match query)
- Modify: `modes/scan.md` (add multi-domain scan)
- Modify: `modes/pdf.md` (add relevant experience query)
- Modify: `modes/interview-prep.md` (add project + strength queries)

- [ ] **Step 1: Add Neo4j instructions to _shared.md**

Read `modes/_shared.md` and add a new section after the "### Tools" table (around line 96). Insert:

```markdown

### Neo4j Candidate Graph (Optional)

If Neo4j MCP is connected and `config/profile.yml` has a `neo4j.person_name` field, query the candidate's knowledge graph for structured evidence before evaluating. This supplements cv.md with verified skill data, project details, and strength patterns.

**Check availability:**
```cypher
MATCH (p:Person {name: $person_name}) RETURN p.name
```

If the Person node exists, use Neo4j queries in the modes below. If not, fall back to cv.md only.

**Person name source:** Read `neo4j.person_name` from `config/profile.yml`. All graph queries must be scoped by this name.
```

- [ ] **Step 2: Add skill-match query to oferta.md**

Read `modes/oferta.md`. In `## Bloque B — Match con CV` (around line 23), add after "Lee `cv.md`. Crea tabla con cada requisito del JD mapeado a líneas exactas del CV.":

```markdown

**Neo4j evidence (if available):** Before scoring match, query for structured skill evidence:
```cypher
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
Use this to cite specific positions, projects, and certifications as evidence for each skill match. This gives more precise matching than keyword scanning cv.md alone.
```

- [ ] **Step 3: Add multi-domain scan to scan.md**

Read `modes/scan.md`. After the existing "Nivel 2b" workflow step (added in the previous plan), add a new section before the "## Filtrar por título" step:

```markdown

### Scan multi-dominio desde Neo4j (opcional)

Si Neo4j está disponible, antes de ejecutar los 3 niveles, consultar los clusters de skills del candidato para generar búsquedas adicionales:

```cypher
MATCH (me:Person {name: $person_name})-[:HAS_SKILL]->(s:Skill)
WHERE s.proficiency IN ["expert", "proficient"]
RETURN s.category, collect(s.name) AS skills, count(s) AS depth
ORDER BY depth DESC
```

Usar los clusters con `depth >= 3` para generar queries adicionales automáticamente. Por ejemplo, si el candidato tiene 5+ networking skills, generar queries de "Network Engineer"; si tiene 3+ ML skills, generar queries de "ML Engineer". Esto permite descubrir roles en dominios adyacentes que el candidato podría no haber considerado.
```

- [ ] **Step 4: Add relevant experience query to pdf.md**

Read `modes/pdf.md`. After the existing step 1 ("Lee `cv.md` como fuentes de verdad"), add:

```markdown

1b. **Neo4j evidence (if available):** Query for most relevant positions and projects:
```cypher
MATCH (me:Person {name: $person_name})-[:WORKED_AT]->(pos:Position)-[:REQUIRED_SKILL]->(s:Skill)
WHERE s.name IN $target_skills
WITH pos, collect(DISTINCT s.name) AS matched_skills
RETURN pos.title, pos.achievements, matched_skills
ORDER BY size(matched_skills) DESC
```
Use this to prioritize which experience to emphasize and reorder bullets by relevance.
```

- [ ] **Step 5: Add project + strength queries to interview-prep.md**

Read `modes/interview-prep.md`. In the `## Inputs` section (line 2-11), add a new input:

```markdown
6. **Neo4j graph** (if available) — query for projects, strengths, and skill evidence
```

After `## Step 1 — Research` section, add a new step:

```markdown
## Step 1b — Query Candidate Graph (if Neo4j available)

```cypher
// Recent projects with talking points
MATCH (me:Person {name: $person_name})-[:BUILT]->(p:Project)
WHERE p.status = "active"
RETURN p.name, p.description, p.highlights,
       [(p)-[:USES_TECH]->(s:Skill) | s.name] AS tech_stack
```

```cypher
// Strengths ranked by evidence
MATCH (me:Person {name: $person_name})-[*1..2]-(e)-[:EVIDENCES]->(str:Strength)
RETURN str.name, str.category, str.description, count(e) AS evidence_count
ORDER BY evidence_count DESC
```

```cypher
// Skills matching this specific role
MATCH (me:Person {name: $person_name})-[:HAS_SKILL]->(s:Skill)
WHERE s.name IN $role_skills
OPTIONAL MATCH (s)<-[:USED_IN]-(pos:Position)
RETURN s.name, s.proficiency, collect(DISTINCT pos.title) AS where_used
```

Use these queries to prepare STAR stories backed by specific project and position evidence. The strengths query surfaces what to lead with; the skills query shows where you've used each technology.
```

- [ ] **Step 6: Commit**

```bash
git add modes/_shared.md modes/oferta.md modes/scan.md modes/pdf.md modes/interview-prep.md
git commit -m "feat: integrate Neo4j candidate graph queries into career-ops modes"
```

---

### Task 12: Update Multi-User Setup Guide

**Files:**
- Modify: `docs/multi-user-setup.md`

- [ ] **Step 1: Rewrite with graph onboarding and personal repo flow**

Replace the entire contents of `docs/multi-user-setup.md`:

```markdown
# Running career-ops for Multiple People

career-ops is single-user by design. Each person gets their own project copy with their own Neo4j graph namespace.

## Eric's Personal Repo

The customized project lives at: `git@github.com:grie1/career_ops_eric.git`

## Setting Up for Another Person

1. **Clone the customized repo:**

   ```bash
   git clone git@github.com:grie1/career_ops_eric.git career-ops-wife
   cd career-ops-wife
   ```

2. **Create their own remote:**

   ```bash
   git remote set-url origin git@github.com:grie1/career_ops_wife.git
   git push -u origin main
   ```

3. **Replace personal files:**

   ```bash
   rm -rf person_profile/*
   ```

   Add their documents to `person_profile/`:
   - `college/` — transcripts, degree certificates
   - `jobs/` — resumes, cover letters (one folder per position)
   - `certs/` — certification documents
   - `github/` — project exports or READMEs
   - `other/` — articles, references, volunteer work

4. **Edit configuration:**

   | File | What to change |
   |------|----------------|
   | `cv.md` | Their CV in markdown |
   | `config/profile.yml` | Name, email, target roles, salary range |
   | `config/profile.yml` | `neo4j.person_name` — set to their name |
   | `modes/_profile.md` | Archetypes, narrative, proof points |
   | `portals.yml` | title_filter keywords and tracked companies |
   | `.env` | API keys (can share or use separate) |

   For community outreach roles, change `title_filter.positive` to:

   ```yaml
   title_filter:
     positive:
       - "Community Outreach"
       - "Community Manager"
       - "Community Engagement"
       - "Outreach Coordinator"
       - "Program Coordinator"
       - "Social Services"
       - "Community Relations"
       - "Public Affairs"
       - "Nonprofit"
       - "Program Manager"
       - "Engagement Specialist"
       - "Social Worker"
       - "Case Manager"
       - "Grant"
   ```

5. **Load their profile into Neo4j:**

   ```
   /career-ops onboard-graph
   ```

   This reads all files in `person_profile/`, extracts entities, and loads them into Neo4j under their `person_name`. Both users share the same Neo4j instance — queries are scoped by person.

6. **Complete standard onboarding:**

   ```
   /career-ops
   ```

7. Each copy is fully independent — separate tracker, reports, scans, pipeline, and graph namespace.
```

- [ ] **Step 2: Commit**

```bash
git add docs/multi-user-setup.md
git commit -m "docs: update multi-user setup with Neo4j onboarding and personal repo flow"
```

---

### Task 13: Push to Personal Remote

**Files:**
- Git remote configuration

- [ ] **Step 1: Add personal remote and push**

```bash
git remote add personal git@github.com:grie1/career_ops_eric.git
git branch -M main
git push -u personal main
```

- [ ] **Step 2: Verify**

```bash
git remote -v
```

Expected: Both `origin` (santifer/career-ops) and `personal` (grie1/career_ops_eric) listed.

```bash
git log --oneline -5
```

Expected: Recent commits visible.

---

### Task 14: Verify Full Graph

- [ ] **Step 1: Run comprehensive graph query**

```cypher
MATCH (p:Person {name: "Eric Grismer"})
OPTIONAL MATCH (p)-[:WORKED_AT]->(pos:Position)
OPTIONAL MATCH (p)-[:HAS_SKILL]->(s:Skill)
OPTIONAL MATCH (p)-[:HOLDS_CERT]->(c:Certification)
OPTIONAL MATCH (p)-[:ATTENDED]->(i:Institution)
OPTIONAL MATCH (p)-[:COMPLETED]->(course:Course)
OPTIONAL MATCH (p)-[:BUILT]->(proj:Project)
RETURN count(DISTINCT pos) AS positions,
       count(DISTINCT s) AS skills,
       count(DISTINCT c) AS certs,
       count(DISTINCT i) AS institutions,
       count(DISTINCT course) AS courses,
       count(DISTINCT proj) AS projects
```

- [ ] **Step 2: Test interview prep query**

```cypher
// "What projects demonstrate my Python skills?"
MATCH (p:Person {name: "Eric Grismer"})-[:BUILT]->(proj:Project)-[:USES_TECH]->(s:Skill {name: "Python"})
RETURN proj.name, proj.description, proj.highlights
```

- [ ] **Step 3: Test CV generation query**

```cypher
// "What's my evidence for networking skills?"
MATCH (p:Person {name: "Eric Grismer"})-[:HAS_SKILL]->(s:Skill {category: "networking"})
OPTIONAL MATCH (s)<-[:USED_IN]-(pos:Position)
OPTIONAL MATCH (s)<-[:VALIDATES]-(cert:Certification)
RETURN s.name, s.proficiency,
       collect(DISTINCT pos.title) AS used_in_positions,
       collect(DISTINCT cert.name) AS validated_by
```

- [ ] **Step 4: Test strengths query**

```cypher
MATCH (str:Strength)<-[:EVIDENCES]-(e)
RETURN str.name, str.category, count(e) AS evidence_count
ORDER BY evidence_count DESC
```

# Career-Ops (Eric's Fork)

> Personal fork of [santifer/career-ops](https://github.com/santifer/career-ops) — customized for network engineering job search with Neo4j candidate knowledge graph integration.

## What's Different in This Fork

- **Neo4j Knowledge Graph** — My entire professional profile (work history, skills, education, certs, projects) is loaded into Neo4j. Evaluations, CV generation, and interview prep all query the graph for evidence-based matching.
- **Network Engineer Focus** — Title filters, search queries, and API sources configured for network engineer / infrastructure roles
- **4 Job Board APIs** — RemoteOK, Jobicy, Remotive, USAJobs (with API key) added as Level 2 sources
- **Expanded Portal Scanner** — 90+ tracked companies, 51 search queries across Indeed, Dice, BuiltIn, LinkedIn, Glassdoor, ClearedJobs, UC Careers
- **Specific Targets** — Anduril and University of California system in tracked companies
- **Multi-User Ready** — Neo4j namespacing by Person node, `person_profile/` directory convention

## My Profile (in Neo4j)

| | |
|---|---|
| **Current Role** | Senior Network Engineer @ Jubitz Corporation (2022-present) |
| **Previous** | Manager, Endpoint & Network IS @ UC Irvine Libraries (2016-2022) |
| **Previous** | Endpoint & Service Desk Manager @ UC Berkeley School of Law (2006-2016) |
| **Ventures** | Rapid LED / Menari LLC ($4k to $1-3M revenue, 2009-2022) |
| | Mendel Sensors / Mendel LLC (5k IoT devices, TechCrunch reviewed, 2019-2022) |
| | Downey Property Development |
| **Certs** | CCNA (active), Cisco Specialist Enterprise Core (active), CCNP Enterprise (in-progress) |
| **Education** | UC Berkeley (Rhetoric), 15+ institutions, AA Health Sciences (IVC) |
| **Graph Stats** | 8 positions, 49 skills, 5 certs, 13 institutions, 28 courses, 6 projects, 7 strengths |

## Quick Start

```bash
# 1. Clone and install
git clone git@github.com:grie1/career_ops_eric.git
cd career_ops_eric && npm install
npx playwright install chromium

# 2. Check setup
npm run doctor

# 3. Configure
cp config/profile.example.yml config/profile.yml  # Edit with your details + neo4j section
cp templates/portals.example.yml portals.yml       # Already has network engineer keywords

# 4. Add your CV
# Create cv.md in the project root with your CV in markdown

# 5. Add personal files and load into Neo4j
# Place documents in person_profile/ (resumes, transcripts, certs, projects)
/career-ops onboard-graph    # Extracts entities and loads Neo4j

# 6. Start using
claude   # Open Claude Code
# Paste a job URL or run /career-ops
```

## Neo4j Integration

The knowledge graph is queried automatically during:

| Mode | What Neo4j provides |
|------|---------------------|
| `/career-ops` (evaluate) | Skill evidence per JD requirement — positions, projects, certs that prove each skill |
| `/career-ops scan` | Multi-domain discovery from skill clusters — auto-generates queries for adjacent roles |
| `/career-ops pdf` | Most relevant experience ranked by skill overlap with target JD |
| `/career-ops interview-prep` | Active projects with talking points, strengths ranked by evidence count |

### Example Queries

```cypher
-- "What projects demonstrate my Python skills?"
MATCH (p:Person {name: "Eric Grismer"})-[:BUILT]->(proj:Project)-[:USES_TECH]->(s:Skill {name: "Python"})
RETURN proj.name, proj.highlights

-- "What's my evidence for networking skills?"
MATCH (p:Person {name: "Eric Grismer"})-[r:HAS_SKILL]->(s:Skill {category: "networking"})
RETURN s.name, r.proficiency

-- "What are my top strengths?"
MATCH (str:Strength)<-[:EVIDENCES]-(e)
RETURN str.name, str.description, count(e) AS evidence
ORDER BY evidence DESC
```

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

## Job Search Configuration

**Target roles:** Network Engineer, Network Architect, Infrastructure Engineer, Systems Engineer, Cloud Network Engineer, Wireless Engineer, and related

**Location:** Remote preferred. Open to relocation: San Francisco, Los Angeles, Orange County, Hawaii (Mililani Town)

**API Sources (Level 2):**
- RemoteOK (no auth)
- Jobicy (no auth, geo + tag filtering)
- Remotive (no auth, keyword search)
- USAJobs (free API key in `.env`)

**Tracked Companies Include:** Anduril, University of California, plus 88 others from upstream

## Setting Up for Another Person

See [docs/multi-user-setup.md](docs/multi-user-setup.md) — clone this repo, replace `person_profile/` contents, set `neo4j.person_name` in config, and run `/career-ops onboard-graph`.

## Upstream

This is a fork of [santifer/career-ops](https://github.com/santifer/career-ops). The original system was built by [Santiago](https://santifer.io) to evaluate 740+ job offers and land a Head of Applied AI role. See the upstream repo for full documentation, the original README, Discord community, and contribution guidelines.

## Usage

All commands from the original system work:

```
/career-ops                → Show all available commands
/career-ops {paste a JD}   → Full auto-pipeline (evaluate + PDF + tracker)
/career-ops scan           → Scan portals for new offers
/career-ops pdf            → Generate ATS-optimized CV
/career-ops batch          → Batch evaluate multiple offers
/career-ops tracker        → View application status
/career-ops apply          → Fill application forms with AI
/career-ops pipeline       → Process pending URLs
/career-ops onboard-graph  → Load profile into Neo4j (new)
/career-ops update-graph   → Incremental graph update (new)
```

## Tech Stack

- **Agent**: Claude Code with custom skills and modes
- **Knowledge Graph**: Neo4j (MCP server)
- **PDF**: Playwright + HTML template
- **Scanner**: Playwright + Greenhouse API + RemoteOK/Jobicy/Remotive/USAJobs APIs + WebSearch
- **Dashboard**: Go + Bubble Tea
- **Data**: Markdown + YAML + TSV

## License

MIT (same as upstream)

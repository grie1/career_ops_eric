# Extra Job Board Searches — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add network engineer job searches across 4 new API sources, 2 tracked companies, and 8 WebSearch queries to the career-ops portal scanner.

**Architecture:** All changes are config/docs — no new scripts. `portals.example.yml` gets new `api_sources` block, additional `tracked_companies`, expanded `title_filter`, and new `search_queries`. `scan.md` is updated to document how the scanner processes `api_sources`. A multi-user setup guide is added.

**Tech Stack:** YAML (config), Markdown (docs/modes)

---

## File Map

| File | Action | Responsibility |
|------|--------|----------------|
| `templates/portals.example.yml` | Modify | Add title keywords, api_sources block, tracked companies, search queries |
| `modes/scan.md` | Modify | Document Level 2 expansion for generic JSON APIs |
| `docs/multi-user-setup.md` | Create | Instructions for copying project for another user |

**Already done (no tasks needed):**
- `.env` — created with `USAJOBS_API_KEY` and `USAJOBS_EMAIL`
- `.gitignore` — updated to exclude `.env`

---

### Task 1: Add Network Engineer Title Filter Keywords

**Files:**
- Modify: `templates/portals.example.yml:22-95` (title_filter section)

- [ ] **Step 1: Add positive keywords after existing entries (line 66)**

Add these lines after the existing `"Internal Tools"` / `"Transformation"` block (line 66) and before the `negative:` section:

```yaml
    # -- Network / Infrastructure roles --
    - "Network Engineer"
    - "Network Architect"
    - "Network Administrator"
    - "Infrastructure Engineer"
    - "Systems Engineer"
    - "NOC Engineer"
    - "Network Security Engineer"
    - "Cloud Network Engineer"
    - "Firewall Engineer"
    - "Network Operations"
    - "Telecommunications Engineer"
    - "Telecom Engineer"
    - "Wireless Engineer"
    - "Site Reliability"
    - "DevOps Engineer"
```

- [ ] **Step 2: Add "Technician" to negative keywords (after line 88)**

Add this line after the existing negative keywords (after `"COBOL"` on line 88):

```yaml
    - "Technician"
```

- [ ] **Step 3: Verify YAML is valid**

Run: `python3 -c "import yaml; yaml.safe_load(open('templates/portals.example.yml'))"`
Expected: No output (success)

- [ ] **Step 4: Commit**

```bash
git add templates/portals.example.yml
git commit -m "feat: add network engineer keywords to title filter"
```

---

### Task 2: Add API Sources Block

**Files:**
- Modify: `templates/portals.example.yml` (new section before `search_queries`)

- [ ] **Step 1: Add api_sources block after title_filter section**

Insert this new section after the `seniority_boost` block (after line 96) and before the `# -- Search queries --` comment (line 98):

```yaml

# -- API sources --
# JSON APIs fetched directly via WebFetch. Each source defines:
#   - url: base API endpoint
#   - format: response shape identifier (remoteok, jobicy, remotive, usajobs)
#   - filters: key-value pairs appended as query params
#   - field_map: maps API fields to standard {title, company, url, location}
#   - auth (optional): env vars for API keys
#
# Jobs array extraction by format:
#   remoteok  → root array (skip first element = legal/meta)
#   jobicy    → root.jobs
#   remotive  → root.jobs
#   usajobs   → root.SearchResult.SearchResultItems[] (.MatchedObjectDescriptor)

api_sources:
  - name: RemoteOK
    url: "https://remoteok.com/api"
    enabled: true
    format: remoteok
    filters:
      tag: "devops"
    field_map:
      title: "position"
      company: "company"
      url: "url"
      location: "location"

  - name: Jobicy
    url: "https://jobicy.com/api/v2/remote-jobs"
    enabled: true
    format: jobicy
    filters:
      tag: "network"
      geo: "usa"
      count: 50
    field_map:
      title: "jobTitle"
      company: "companyName"
      url: "url"
      location: "jobGeo"

  - name: Remotive
    url: "https://remotive.com/api/remote-jobs"
    enabled: true
    format: remotive
    filters:
      search: "network engineer"
      limit: 50
    field_map:
      title: "title"
      company: "company_name"
      url: "url"
      location: "candidate_required_location"

  - name: USAJobs
    url: "https://data.usajobs.gov/api/search"
    enabled: true
    format: usajobs
    auth:
      header_key: "Authorization-Key"
      env_var: "USAJOBS_API_KEY"
      user_agent_env: "USAJOBS_EMAIL"
    filters:
      Keyword: "Network Engineer"
      LocationName: "San Francisco;Los Angeles;Hawaii;Remote"
      ResultsPerPage: 50
    field_map:
      title: "PositionTitle"
      company: "OrganizationName"
      url: "PositionURI"
      location: "PositionLocationDisplay"

```

- [ ] **Step 2: Verify YAML is valid**

Run: `python3 -c "import yaml; yaml.safe_load(open('templates/portals.example.yml'))"`
Expected: No output (success)

- [ ] **Step 3: Commit**

```bash
git add templates/portals.example.yml
git commit -m "feat: add api_sources block for RemoteOK, Jobicy, Remotive, USAJobs"
```

---

### Task 3: Add Tracked Companies (Anduril + UC Jobs)

**Files:**
- Modify: `templates/portals.example.yml` (tracked_companies section, after line 910)

- [ ] **Step 1: Add Anduril and UC Jobs at end of tracked_companies**

Append after the last entry (Maxim AI, line 910):

```yaml

  # -- Specific targets (Network Engineering) --

  - name: Anduril
    careers_url: https://jobs.ashbyhq.com/anduril
    notes: "Defense tech. Ashby SPA. SF/OC/DC offices."
    enabled: true

  - name: University of California
    careers_url: https://jobs.universityofcalifornia.edu
    scan_method: playwright
    scan_query: '"network engineer" OR "infrastructure engineer" OR "systems engineer"'
    notes: "UC system portal. Server-rendered HTML. Aggregates all campuses."
    enabled: true
```

- [ ] **Step 2: Verify YAML is valid**

Run: `python3 -c "import yaml; yaml.safe_load(open('templates/portals.example.yml'))"`
Expected: No output (success)

- [ ] **Step 3: Commit**

```bash
git add templates/portals.example.yml
git commit -m "feat: add Anduril and UC Jobs to tracked companies"
```

---

### Task 4: Add Network Engineer WebSearch Queries

**Files:**
- Modify: `templates/portals.example.yml` (search_queries section)

- [ ] **Step 1: Add network engineer queries after existing search_queries**

Find the last `search_queries` entry (currently the `HN Who's Hiring` entry around line 293). Add after it, before the `# -- Tracked companies --` section:

```yaml

  # -- Network Engineer specific --

  - name: Indeed — Network Engineer Remote
    query: 'site:indeed.com "Network Engineer" OR "Network Architect" remote'
    enabled: true

  - name: Dice — Network Engineer
    query: 'site:dice.com "Network Engineer" OR "Infrastructure Engineer" OR "Network Architect"'
    enabled: true

  - name: BuiltIn — Network/Infra Engineer
    query: 'site:builtin.com "Network Engineer" OR "Infrastructure Engineer" OR "Cloud Network" remote'
    enabled: true

  - name: LinkedIn — Network Engineer Remote
    query: 'site:linkedin.com/jobs "Network Engineer" OR "Network Architect" remote'
    enabled: true

  - name: Glassdoor — Network Engineer
    query: 'site:glassdoor.com/job-listing "Network Engineer" OR "Infrastructure Engineer" remote'
    enabled: true

  - name: USAJobs Web — Network Engineer
    query: 'site:usajobs.gov "Network Engineer" OR "IT Specialist" "Network" California OR Hawaii OR remote'
    enabled: true

  - name: Cleared Jobs — Network Engineer
    query: 'site:clearedjobs.net "Network Engineer" OR "Network Architect" OR "Infrastructure Engineer"'
    enabled: true

  - name: UC Careers Web — Network/IT
    query: 'site:jobs.universityofcalifornia.edu "Network Engineer" OR "IT Infrastructure" OR "Systems Engineer"'
    enabled: true
```

- [ ] **Step 2: Verify YAML is valid**

Run: `python3 -c "import yaml; yaml.safe_load(open('templates/portals.example.yml'))"`
Expected: No output (success)

- [ ] **Step 3: Commit**

```bash
git add templates/portals.example.yml
git commit -m "feat: add network engineer WebSearch queries for 8 job boards"
```

---

### Task 5: Update scan.md — Document Level 2 Expansion

**Files:**
- Modify: `modes/scan.md:36-71` (Level 2 section and workflow)

- [ ] **Step 1: Expand Level 2 description**

Replace the current Level 2 section (lines 36-38):

```
### Nivel 2 — Greenhouse API (COMPLEMENTARIO)

Para empresas con Greenhouse, la API JSON (`boards-api.greenhouse.io/v1/boards/{slug}/jobs`) devuelve datos estructurados limpios. Usar como complemento rápido de Nivel 1 — es más rápido que Playwright pero solo funciona con Greenhouse.
```

With:

```
### Nivel 2 — JSON APIs (COMPLEMENTARIO)

Dos tipos de APIs:

**A) Greenhouse API** — Para empresas con Greenhouse, la API JSON (`boards-api.greenhouse.io/v1/boards/{slug}/jobs`) devuelve datos estructurados limpios. Configurado por empresa en `tracked_companies` con campo `api:`.

**B) API Sources genéricas** — Portales con APIs JSON públicas configuradas en el bloque `api_sources` de `portals.yml`. Cada fuente define:
- `url`: endpoint base de la API
- `format`: identificador de forma de respuesta (`remoteok`, `jobicy`, `remotive`, `usajobs`)
- `filters`: pares key-value que se añaden como query params a la URL
- `field_map`: mapeo de campos del API a `{title, company, url, location}`
- `auth` (opcional): variables de entorno para API keys (leídas de `.env`)

**Extracción del array de jobs por formato:**

| Format | Path al array de jobs |
|--------|----------------------|
| `remoteok` | Array raíz (saltar primer elemento = legal/meta) |
| `jobicy` | `root.jobs` |
| `remotive` | `root.jobs` |
| `usajobs` | `root.SearchResult.SearchResultItems[]` (datos en `.MatchedObjectDescriptor`) |
| greenhouse | Array en `root.jobs` |

**Auth:** Si la fuente tiene `auth`, leer la API key de la variable de entorno indicada en `auth.env_var` y enviarla en el header indicado en `auth.header_key`. Para USAJobs, además enviar el email de `auth.user_agent_env` en el header `User-Agent`.
```

- [ ] **Step 2: Update execution priority (lines 44-47)**

Replace:

```
**Prioridad de ejecución:**
1. Nivel 1: Playwright → todas las `tracked_companies` con `careers_url`
2. Nivel 2: API → todas las `tracked_companies` con `api:`
3. Nivel 3: WebSearch → todos los `search_queries` con `enabled: true`
```

With:

```
**Prioridad de ejecución:**
1. Nivel 1: Playwright → todas las `tracked_companies` con `careers_url`
2. Nivel 2a: Greenhouse API → todas las `tracked_companies` con `api:`
3. Nivel 2b: API Sources → todas las `api_sources` con `enabled: true`
4. Nivel 3: WebSearch → todos los `search_queries` con `enabled: true`
```

- [ ] **Step 3: Add Level 2b workflow step after step 5 (line 71)**

After the existing step 5 (Nivel 2 — Greenhouse APIs, ending at line 71), insert:

```

5b. **Nivel 2b — API Sources genéricas** (secuencial — respetar rate limits):
    Para cada fuente en `api_sources` con `enabled: true`:
    a. Construir URL: `{url}?{filters como query params}`
    b. Si `auth` definido: leer key de `.env` variable `auth.env_var`, añadir header `auth.header_key`
    c. WebFetch de la URL construida → JSON
    d. Extraer array de jobs según `format` (ver tabla de extracción)
    e. Para cada job: mapear campos usando `field_map` → `{title, url, company, location}`
    f. Acumular en lista de candidatos (dedup con Nivel 1 + 2a)
```

- [ ] **Step 4: Commit**

```bash
git add modes/scan.md
git commit -m "docs: update scan.md with Level 2 API sources documentation"
```

---

### Task 6: Create Multi-User Setup Guide

**Files:**
- Create: `docs/multi-user-setup.md`

- [ ] **Step 1: Create the file**

```markdown
# Running career-ops for Multiple People

career-ops is single-user by design. To run searches for a different person,
copy the entire project directory and customize.

## Steps

1. **Copy the project:**

   ```bash
   cp -r ~/Desktop/github/career-ops ~/Desktop/github/career-ops-wife
   cd ~/Desktop/github/career-ops-wife
   ```

2. **Edit these files for the new user:**

   | File | What to change |
   |------|----------------|
   | `cv.md` | Their CV in markdown |
   | `config/profile.yml` | Name, email, target roles, salary range |
   | `modes/_profile.md` | Archetypes, narrative, proof points |
   | `portals.yml` | title_filter keywords and tracked companies |
   | `.env` | API keys (can share or use separate) |

3. **For community outreach roles, change `title_filter.positive` to:**

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

4. **Run `/career-ops` in the new directory** to go through onboarding.

5. Each copy is fully independent — separate tracker, reports, scans, and pipeline.
```

- [ ] **Step 2: Commit**

```bash
git add docs/multi-user-setup.md
git commit -m "docs: add multi-user setup guide for running parallel job searches"
```

---

### Task 7: Verify Everything

- [ ] **Step 1: Validate full YAML**

Run: `python3 -c "import yaml; data = yaml.safe_load(open('templates/portals.example.yml')); print(f'title_filter positive: {len(data[\"title_filter\"][\"positive\"])} keywords'); print(f'api_sources: {len(data.get(\"api_sources\", []))} sources'); print(f'search_queries: {len(data[\"search_queries\"])} queries'); print(f'tracked_companies: {len(data[\"tracked_companies\"])} companies')"`

Expected output (approximate):
```
title_filter positive: 47 keywords
api_sources: 4 sources
search_queries: 49 queries
tracked_companies: 55 companies
```

- [ ] **Step 2: Verify .env exists and has keys**

Run: `test -f .env && grep -c "USAJOBS" .env`
Expected: `2` (two lines with USAJOBS)

- [ ] **Step 3: Verify .env is gitignored**

Run: `git status .env`
Expected: no output (file is ignored)

- [ ] **Step 4: Quick smoke test — fetch one API to confirm it works**

Run: `curl -s "https://jobicy.com/api/v2/remote-jobs?tag=network&geo=usa&count=5" | python3 -c "import sys,json; d=json.load(sys.stdin); print(f'Jobicy returned {len(d.get(\"jobs\",[]))} jobs'); [print(f'  - {j[\"jobTitle\"]} @ {j[\"companyName\"]}') for j in d.get('jobs',[])]"`

Expected: A list of network-related remote jobs (confirms the API is live and our filters work).

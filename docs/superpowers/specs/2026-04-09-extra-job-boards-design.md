# Extra Job Board Searches — Design Spec

**Date:** 2026-04-09
**Author:** Eric + Claude
**Status:** Approved

## Goal

Extend career-ops portal scanner to find network engineer roles across more job boards, prioritizing boards with free JSON APIs. Add Anduril and University of California as tracked companies. Support Eric's job search targeting remote roles with relocation openness to SF, LA, Orange County, and Hawaii (Mililani Town).

## Non-Goals

- Neo4j integration (next project)
- UI/dashboard for toggling boards (future)
- Multi-user support (handled via project copy — see Section 6)

---

## Section 1: Title Filter Keywords

Add network engineering keywords to `portals.yml` title filter.

### New `title_filter.positive` entries

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

### New `title_filter.negative` entry

```yaml
- "Technician"
```

### Seniority boost

No changes — existing Senior/Staff/Principal/Lead/Head/Director coverage is sufficient.

---

## Section 2: API Integrations (Level 2 Expansion)

Currently Level 2 only supports Greenhouse API. Extend to support generic JSON APIs via a new `api_sources` block in `portals.yml`.

### Config structure

```yaml
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

### How it works

- `field_map` tells the scanner which JSON fields map to title/company/url/location — no code changes per board
- `filters` become query params appended to the URL
- USAJobs is the only source needing auth — reads key from `.env` (already created with `USAJOBS_API_KEY` and `USAJOBS_EMAIL`)
- All results flow through the existing title filter and dedup pipeline
- `scan.md` gets updated to document this Level 2 expansion

### API details

| Board | Auth | Rate Limits | Notes |
|-------|------|-------------|-------|
| RemoteOK | None | Link-back required | ~100 jobs/call, tag filtering |
| Jobicy | None | Few times/day | Rich filtering (tag, geo, industry), salary data |
| Remotive | None | Max 4 req/day, 24h delay | search + category + limit |
| USAJobs | Free API key (header) | Standard gov limits | Rich filtering (keyword, location, pay grade) |

### Response extraction patterns

Each API has a different response shape. The scanner needs to know how to find the jobs array:

| Format | Jobs array path |
|--------|----------------|
| `remoteok` | Root array (skip first element which is legal/meta) |
| `jobicy` | `root.jobs` |
| `remotive` | `root.jobs` |
| `usajobs` | `root.SearchResult.SearchResultItems[]` (job data in `.MatchedObjectDescriptor`) |

---

## Section 3: New Tracked Companies (Level 1 — Playwright)

### Anduril

```yaml
- name: Anduril
  careers_url: https://jobs.ashbyhq.com/anduril
  notes: "Defense tech. Ashby SPA. SF/OC/DC offices."
  enabled: true
```

Standard Ashby Playwright pattern — same as Cohere, LangChain, Pinecone.

### University of California

```yaml
- name: University of California
  careers_url: https://jobs.universityofcalifornia.edu
  scan_method: playwright
  scan_query: '"network engineer" OR "infrastructure engineer" OR "systems engineer"'
  notes: "UC system portal. Server-rendered HTML. Aggregates all campuses."
  enabled: true
```

UC Jobs is server-rendered HTML (not an SPA). The scanner would:
1. Navigate to the search page
2. Enter keywords via the search form
3. Extract results from the HTML listing
4. Follow pagination if needed

This is a "custom Playwright" target — extraction logic differs from the Ashby SPA pattern. Individual job links redirect to campus-specific ATS platforms (Taleo, iCIMS, PeopleSoft).

---

## Section 4: New WebSearch Queries (Level 3)

For boards without APIs, add `site:` queries for network engineer roles:

```yaml
search_queries:
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

These supplement the API sources with broader Google-indexed discovery. The existing liveness verification (Playwright check for Level 3 results) still applies.

---

## Section 5: scan.md Updates

The scan mode documentation (`modes/scan.md`) needs updates to reflect:

1. **Level 2 expansion:** Document the `api_sources` config block and how generic JSON APIs are processed alongside Greenhouse APIs
2. **Response extraction:** Document the per-format jobs array paths
3. **Auth handling:** Document how `.env` variables are read for authenticated APIs (USAJobs)
4. **Execution order update:**
   - Level 1: Playwright (tracked_companies with careers_url) — unchanged
   - Level 2: APIs (Greenhouse + new api_sources) — expanded
   - Level 3: WebSearch (search_queries) — new queries added

---

## Section 6: Multi-User Setup Guide

Create `docs/multi-user-setup.md` with instructions for duplicating the project for a different user (e.g., wife's community outreach search):

```markdown
# Running career-ops for Multiple People

career-ops is single-user by design. To run searches for a different person,
copy the entire project directory and customize.

## Steps

1. Copy the project:
   cp -r ~/Desktop/github/career-ops ~/Desktop/github/career-ops-wife

2. Edit these files for the new user:
   - cv.md — their CV
   - config/profile.yml — name, email, target roles, salary range
   - modes/_profile.md — archetypes, narrative, proof points
   - portals.yml — title_filter keywords and tracked companies
   - .env — API keys (can share or use separate)

3. For community outreach roles, change title_filter.positive to:
   - "Community Outreach"
   - "Community Manager"
   - "Community Engagement"
   - "Outreach Coordinator"
   - "Program Coordinator"
   - "Social Services"
   - "Community Relations"
   - "Public Affairs"
   - "Nonprofit"
   - etc.

4. Run /career-ops in the new directory to go through onboarding.

5. Each copy is fully independent — separate tracker, reports, scans.
```

---

## Files Changed

| File | Change |
|------|--------|
| `portals.yml` (user copies from example) | Add title keywords, api_sources, tracked companies, search queries |
| `templates/portals.example.yml` | Same additions (so new users get them) |
| `modes/scan.md` | Document Level 2 expansion, api_sources config, auth handling |
| `.env` | Already created — USAJobs API key + email |
| `.gitignore` | Already updated — excludes .env |
| `docs/multi-user-setup.md` | New file — multi-user copy instructions |

## Dependencies

- `.env` with `USAJOBS_API_KEY` and `USAJOBS_EMAIL` (already in place)
- No new npm packages required — WebFetch handles JSON API calls
- No new scripts — scanner reads config and uses existing WebFetch/Playwright tools

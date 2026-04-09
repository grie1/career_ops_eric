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

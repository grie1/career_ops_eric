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

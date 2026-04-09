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

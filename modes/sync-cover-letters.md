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

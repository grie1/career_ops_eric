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

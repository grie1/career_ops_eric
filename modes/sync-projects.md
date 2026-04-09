# Mode: sync-projects — GitHub Repos to Neo4j

Extract project metadata and tech stacks from local GitHub repos in `person_profile/github/` and return structured data for Neo4j loading.

## When to Use

- `/career-ops sync-projects` (standalone)
- Called by `/career-ops update-graph` (orchestrated)

## Input

- `scan_dir`: path to `person_profile/github/` (from ingestors.yml or profile.yml)
- `person_name`: from `config/profile.yml → neo4j.person_name`

## Extraction Workflow

For each subdirectory in `scan_dir`:

1. **Identify the project**: directory name = project name
2. **Read README.md or CLAUDE.md** (if exists): extract description, purpose, key highlights
3. **Read config files** (in priority order):
   - `requirements.txt` → Python dependencies
   - `pyproject.toml` → Python project metadata + deps
   - `package.json` → Node.js dependencies
   - `platformio.ini` → Arduino/embedded platform + libraries
   - `Cargo.toml` → Rust dependencies
   - `go.mod` → Go modules
4. **Scan source code imports** (first 50 lines of up to 10 files):
   - `.py` files: `import X` / `from X import Y`
   - `.ino` / `.cpp` / `.h` files: `#include <X.h>`
   - `.js` / `.ts` files: `require('X')` / `import X from 'Y'`
5. **Check for git remote**: `cat .git/config` → extract GitHub URL if present
6. **Assess complexity**: count files, LOC, number of modules/directories
7. **Generate interview highlights**: 2-3 sentences about what would impress a hiring manager

## Skill Categorization

Map extracted libraries/technologies to categories:
- **programming**: Python, numpy, pandas, scipy, scikit-learn, PyTorch, TensorFlow, C++, Rust, Go, JavaScript
- **hardware**: Arduino, i2c, SPI, BLE, IoT, sensors, OLED, embedded
- **cloud**: AWS, Azure, GCP, Docker, Kubernetes
- **tools**: Streamlit, Plotly, SQLite, Git, CI/CD
- **data**: ML, Neural Networks, LLM, NLP, Financial Data Analysis

Canonicalize names: "sklearn" → "scikit-learn", "torch" → "PyTorch", etc.

## Return Format

Return a JSON envelope:

```json
{
  "ingestor": "sync-projects",
  "timestamp": "ISO-8601",
  "entities_found": N,
  "data": [
    {
      "type": "Project",
      "name": "project-name",
      "properties": {
        "description": "What it does (2-3 sentences)",
        "highlights": "Interview talking points",
        "url": "https://github.com/user/repo",
        "status": "active|completed|archived",
        "complexity": "high|medium|simple",
        "loc": 1234
      },
      "skills": ["Python", "numpy", "pandas"],
      "skill_categories": {"Python": "programming", "numpy": "programming"}
    }
  ],
  "suggestions": [
    "project-x has no README — add one for better extraction",
    "project-y has no tests — mentioning test coverage strengthens interview answers"
  ]
}
```

## Important

- Never fabricate technologies — only report what you find in code/configs
- If a project directory is empty or has only git metadata, skip it and note in suggestions
- If README is absent, infer description from code comments and file structure
- Return suggestions for any quality issues found (missing README, no tests, thin descriptions)

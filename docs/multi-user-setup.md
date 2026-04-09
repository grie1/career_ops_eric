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

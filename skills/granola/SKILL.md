---
name: granola
description: Access Granola meeting transcripts and notes.
homepage: https://granola.ai
metadata: {"clawdis":{"emoji":"🥣","requires":{"bins":["python3"]}}}
---

# granola

Access Granola meeting transcripts, summaries, and notes.

## Setup

Granola stores meetings in the cloud. To access them locally:

1. **Run initial sync:**
```bash
pip install requests  # if needed
python {baseDir}/scripts/sync.py ~/granola-meetings
```

2. **Set up automatic sync via Clawdis cron:**
```
Use clawdis_cron to add a job:
- name: "Granola Sync"
- schedule: { kind: "cron", expr: "0 */6 * * *" }  # every 6 hours
- sessionTarget: "isolated"
- payload: {
    kind: "agentTurn",
    message: "Run the Granola sync script: python {baseDir}/scripts/sync.py ~/granola-meetings. Report any errors."
  }
```

Or via the tool:
```javascript
clawdis_cron({
  action: "add",
  job: {
    name: "Granola Sync",
    description: "Sync Granola meetings to local disk",
    schedule: { kind: "cron", expr: "0 */6 * * *", tz: "America/New_York" },
    sessionTarget: "isolated",
    wakeMode: "now",
    payload: {
      kind: "agentTurn",
      message: "Run: python ~/clawdis/skills/granola/scripts/sync.py ~/granola-meetings",
      deliver: false
    }
  }
})
```

The sync script reads auth from `~/Library/Application Support/Granola/supabase.json` (created when you sign into Granola).

## Data Structure

After sync, each meeting is a folder:
```
~/granola-meetings/
  {meeting-id}/
    metadata.json   - title, date, attendees
    transcript.md   - formatted transcript  
    transcript.json - raw transcript data
    document.json   - full API response
    notes.md        - AI summary (if available)
```

## Quick Commands

**List recent meetings:**
```bash
for d in $(ls -t ~/granola-meetings | head -10); do
  jq -r '"\(.created_at[0:10]) | \(.title)"' ~/granola-meetings/$d/metadata.json 2>/dev/null
done
```

**Search by title:**
```bash
grep -l "client name" ~/granola-meetings/*/metadata.json | while read f; do
  jq -r '.title' "$f"
done
```

**Search transcript content:**
```bash
grep -ri "keyword" ~/granola-meetings/*/transcript.md
```

**Get notes for a meeting:**
```bash
ID=$(grep -l "Meeting Title" ~/granola-meetings/*/metadata.json | head -1 | xargs dirname | xargs basename)
cat ~/granola-meetings/$ID/notes.md
```

**Meetings on a specific date:**
```bash
for d in ~/granola-meetings/*/metadata.json; do
  if jq -e '.created_at | startswith("2026-01-03")' "$d" > /dev/null 2>&1; then
    jq -r '.title' "$d"
  fi
done
```

## Fallback: Local Cache

If you can't sync (no network, token expired), Granola's app cache has recent meetings:
```bash
cat ~/Library/Application\ Support/Granola/cache-v3.json | jq -r '.cache' | jq '.state.documents'
```

Note: The cache only contains meetings loaded in the app, not the full history.

## Notes

- Sync requires the Granola desktop app to be signed in (for auth tokens)
- Tokens expire after ~6 hours; open Granola to refresh them
- The API is at `api.granola.so` — may not be accessible from all networks
- For multi-machine setups, sync on one machine and rsync the folder to others

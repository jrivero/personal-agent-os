# /process-calls Skill

Process yesterday's calls from Granola: save transcripts to vault, extract short bullet summaries.

## Usage

```
/process-calls [date]
```

**Examples:**
- `/process-calls` - Process yesterday's calls
- `/process-calls 2026-06-03` - Process calls from a specific date

**Arguments:**
- `date` (optional): ISO date `YYYY-MM-DD`. Defaults to yesterday.

---

## Architecture: Skill vs Extension

**This skill (generic):**
- Fetches calls, classifies, saves, extracts bullets
- No personal data, account names, or project keywords

**Extension in vault (personal config):**
- Granola classification rules (keywords → project folders)
- Save paths per project
- Timezone

**Location:** `{{vault}}/00_SYSTEM/extensions/process-calls.md`

Falls back to `daily.md` extension if `process-calls.md` doesn't exist (they share the Granola Classification section).

---

## Flow

### Step 1: Setup (Silent)

1. Read `config/vault.json` to get vault path. Fall back to `../my-vault/`.
2. Determine target date:
   - If `date` arg provided: use it
   - Otherwise: `date -v-1d "+%Y-%m-%d"` (yesterday)
3. Get ISO week: `date -j -f "%Y-%m-%d" "{target_date}" "+%Y-W%V"`
4. Load extension:
   - Try `{{vault}}/00_SYSTEM/extensions/process-calls.md`
   - If not found, try `{{vault}}/00_SYSTEM/extensions/daily.md` (fallback)
   - Read the **Granola Classification** section for project rules

---

### Step 2: Fetch Calls from Granola

```
list_meetings(time_range="custom", custom_start={target_date}, custom_end={target_date})
```

- If no meetings returned: output `No calls found for {date}. Done.` and stop.
- If MCP unavailable: output `Granola MCP not available. Exiting.` and stop.

For each meeting, fetch the transcript:
```
get_meeting_transcript(meeting_id={id})
```

**Skip if:**
- Transcript is empty
- Title is "New note" or blank

---

### Step 3: Classify Each Call

Using the **Granola Classification** rules from the extension:

- Match the call title and participant names against each signal (case-insensitive keyword match)
- First matching rule wins
- If no rule matches: classify as `Personal`

If the extension has no classification rules (still has placeholder text or the section is empty): classify all calls as `Personal`.

---

### Step 4: Save Transcripts to Vault

For each classified call:

**Target path:**
- Project calls: `{{vault}}/03_PROJECTS/{Project}/Calls/{YYYY-WXX}/{date}-{slug}.md`
- Personal: `{{vault}}/04_PEOPLE/Notes/{date}-{slug}.md`

Where `{slug}` = title lowercased, spaces replaced with hyphens, non-alphanumeric stripped.

**Skip duplicates:** If the file already exists, skip silently.

**File format:**
```markdown
# {Title}

*Date: YYYY-MM-DD*
*Participants: {names}*
*Source: Granola*

---

## Transcript

{Raw transcript exactly as returned by get_meeting_transcript}

---

#{project} #granola
```

Create parent directories if they don't exist.

---

### Step 5: Extract Bullets from Each Transcript

For each call that was saved (or already existed), read the transcript and extract:

**Learned** — facts, decisions, insights, context gained
**Improve** — gaps noticed, mistakes, friction, things that could go better
**To-do** — concrete actions, follow-ups, things to do

Rules:
- 2-4 bullets max per category
- Only include a category if there's something real to say — skip empty categories
- Short, specific, plain language — no fluff
- To-dos use `- [ ]` format

**Output format per call:**
```markdown
## {Title} ({date})

**Learned:**
- bullet
- bullet

**Improve:**
- bullet

**To-do:**
- [ ] action
- [ ] action
```

---

### Step 6: Write Call Digest

Assemble all per-call bullet summaries into a single digest file.

**Target path:** `{{vault}}/00_SYSTEM/OPS/call-digests/{date}.md`

Create the `call-digests/` folder if it doesn't exist.

**Skip if file already exists** — tell user it exists and where.

**File format:**
```markdown
---
date: {YYYY-MM-DD}
calls: {N}
---

# Call Digest — {Month D, YYYY}

{Per-call sections from Step 5, separated by ---}
```

---

### Step 7: Report to User

Show a clean summary:

```
✓ Processed {N} call(s) for {date}

| Call | Project | Saved |
|------|---------|-------|
| {title} | {project} | {path or "already existed"} |
...

Digest: 00_SYSTEM/OPS/call-digests/{date}.md
```

If all calls were already saved (all duplicates), still generate the digest unless it already exists.

---

## Error Handling

- **Granola MCP unavailable:** Exit with message, no files written.
- **No calls for date:** Exit cleanly, no files written.
- **File write fails:** Show warning, continue with remaining calls.
- **Extension missing classification rules:** Classify all as Personal, continue.

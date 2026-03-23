
# Nubot — State



> Last updated: 2026-03-23

> Last session: 2026-03-16 (morning nudge fixes)



## What just shipped (March 16)



- **Morning nudge conversation flow fixed:**

  - `handle_morning_reply` now uses Haiku to classify affirmative replies instead of hardcoded word list

  - Conversation history sent as proper message turns (user/assistant roles) instead of stuffed into system prompt

  - Auto-expire: `morning_active` turns off after 2 hours via check in text handler

  - `morning_started` timestamp tracked in `bot_data`

  - Confirmation check requires `len(history) >= 2` to prevent premature close

  - Bot "done" signals also close the conversation automatically

- **Morning nudge prompt tweaks** (`morning-nudge.md`):

  - Removed "After 20:00 → focus on tomorrow's plan" (irrelevant for morning)

  - Added "Always ask what time training is — meal timing depends on it"

- **Bug fixed:** Bot went silent ~25 minutes on March 16 because `morning_active` stayed True after conversation failed to close. Auto-expire and Haiku confirmation check prevent recurrence.



## What's deployed and working



- Voice journaling → Whisper transcription → Obsidian daily journal

- Voice memo cleanup (Sonnet pass, raw transcript in collapsible block)

- Meal logging: text + voice → writes to both Obsidian markdown and SQLite

- "Log it" / "add it" short-circuit out of sparring mode

- `/undo` deletes last meal from SQLite

- `/analyze`, `/search`, `/tasks`, `/week`, `/today`, `/fridge` commands

- Sparring mode (food decision support conversation)

- Morning nudge with real health data (sleep, HRV, yesterday's workouts from SQLite)

- Health Auto Export pipeline: Apple Watch → FastAPI receiver → SQLite

- health-receiver.service running on port 8765



## What's broken / known issues



- `notesmd-cli` PATH issue in systemd — `/search` command works from terminal but not from bot. Fix: add `Environment="PATH=/usr/local/bin:/usr/bin:/bin"` to service file.

- Text handler meal prefix detection not catching all patterns from sparring context.



## What's next (immediate)



- Fix notesmd-cli PATH in systemd service file

- Deploy auto-tier evening check-in (code exists, not wired)

- Wire SQLite db.py fully (some handlers still not using it)

- Design subjective daily check-in (morning + evening)

- Set up GitHub remote for ~/nubot/



## Decisions / notes



- Carbs are the primary nutrition metric (not protein) — high daily running volume

- Trend-level accuracy over precision tracking

- Claude estimation with "estimated" flag is the right approach

- Three-layer food fallback: prepped → grab-and-go → delivery

- If not logging, something is wrong upstream — reduce friction, don't add it

- Don't over-engineer before validating habits



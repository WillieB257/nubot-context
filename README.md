# Nubot Context

---


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



---


# Nubot — Architecture



> Last updated: 2026-03-18



## Overview



Nubot is a personal Telegram bot for voice journaling, nutrition tracking, and daily planning. It runs on a Raspberry Pi 5, integrates with an Obsidian vault on a Synology NAS, and stores structured data in SQLite. Health metrics flow in passively from Apple Watch via a REST API.



## Infrastructure



| Component | Details |

|---|---|

| **Raspberry Pi 5** (64GB) | Primary host. User: `williepi`. Home: `/home/williepi`. Tailscale IP: `100.81.224.12`. Local IP: `192.168.178.77` |

| **Synology NAS** (DS220+) | "WilfredTheNas". Obsidian vault storage. Tailscale IP: `100.126.189.113`. Local IP: `192.168.178.65` |

| **Hetzner VPS** (CAX11, ARM64) | Transitional backup. Tailscale IP: `100.100.212.95`. Runs obsidian-headless. To be decommissioned. |

| **iPhone** | Telegram client + Health Auto Export app. Tailscale IP: `100.107.83.32` |



### Networking



- **Tailscale** connects all devices. Pi is reachable remotely.

- **SSHFS** mounts vault from NAS to Pi at `/mnt/obsidian/`

- **obsidian-headless** runs on Hetzner for Obsidian Sync (Node.js service)



## Systemd services on Pi



| Service | What it runs |

|---|---|

| `voice-bot.service` | Nubot Telegram bot (`python3 -m nubot`) |

| `health-receiver.service` | FastAPI health data receiver (`~/nubot/health_receiver.py`, port 8765) |



## Module structure

```

~/nubot/

├── config.py              # API keys, paths, model config, food triggers, DB_PATH

├── logging_config.py

├── main.py                # Bot startup, handler registration

├── __main__.py            # Entry point for python3 -m nubot

├── health_receiver.py     # FastAPI receiver for Health Auto Export (separate service)

├── nubot_schema.sql       # SQLite schema definition

│

├── services/

│   ├── api.py             # Anthropic + OpenAI clients, cost tracking

│   ├── transcription.py   # Whisper wrapper, clean_transcript

│   ├── ai.py              # Claude calls: classify, estimate, analyze, read_prompt, clean_voice_memo

│   ├── vault.py           # Vault I/O: journals, food logs, weight, fridge, schedule

│   └── db.py              # SQLite helpers: get_db(), log_meal()

│

├── handlers/

│   ├── voice.py           # Voice → transcribe → classify → save (with Sonnet cleanup pass)

│   ├── text.py            # Text router: tier, weight, meal, sparring, log-it short-circuit

│   ├── commands.py        # /analyze, /search, /tasks, /week, /today, /fridge, /undo, etc.

│   ├── sparring.py        # Food decision support conversation

│   └── scheduled.py       # Morning nudge, dinner gate, evening checkin

│

└── data/

    └── nubot.db           # SQLite database

```



## Data layer



### SQLite (`~/nubot/data/nubot.db`)



| Table | Purpose | Key columns |

|---|---|---|

| `health_metrics` | One row per day from Apple Watch | date, weight_kg, sleep_hours, hrv, resting_hr, active_kcal, basal_kcal |

| `workouts` | One row per workout from Apple Watch | date, workout_type, start_time, duration_minutes, calories, distance_km, avg_hr, source_id (UNIQUE for dedup) |

| `meals` | One row per logged meal | date, meal_type, time, description, estimated_kcal, estimated_carbs_g, estimated_protein_g, estimated_fat_g, source |



### Obsidian vault (`/mnt/obsidian/`)



- **Daily journals** — voice transcripts, reflections

- **Food logs** — markdown meal entries (parallel write with SQLite)

- **Prompts** — bot prompt files at `/mnt/obsidian/nutrition/prompts/`

  - `morning-nudge.md` — morning planning with health data

  - `sparring.md` — food decision support

  - `dinner-gate.md` — evening food intervention

  - `voice-cleanup.md` — voice memo transcript cleanup



### Health data pipeline

```

Apple Watch → Health Auto Export (iOS app)

  → POST http://100.81.224.12:8765/api/health (api-key auth)

  → health_receiver.py (FastAPI)

  → SQLite health_metrics + workouts tables

```



Two automations configured in the app: Health Metrics and Workouts. Energy data arrives as per-minute kJ — receiver sums per day and converts to kcal.



Metric name mapping: `weight_body_mass→weight_kg`, `heart_rate_variability→hrv`, `resting_heart_rate→resting_hr`, `sleep_analysis→sleep_hours`, `active_energy→active_kcal`, `basal_energy_burned→basal_kcal`. Skipped: `apple_exercise_time`, `physical_effort`.



## External tools



| Tool | Location | Purpose |

|---|---|---|

| `notesmd-cli` | `/usr/local/bin/notesmd-cli` | Headless vault search (Go binary by Yakitrak). Wired to `/search` command. Has PATH issue in systemd. |

| `appie-go` | `~/go/bin/appie` | Albert Heijn OAuth client (installed, not yet wired) |



## Dev environment



- **Editor:** Zed via SSH remote into Pi

- **Mobile SSH:** Termius (iPhone)

- **Bot stack:** Python, Telegram long-polling, Whisper API, Claude API, OpenAI API


---


# Nubot — Roadmap



> Last updated: 2026-03-18



## Design principles



- Prevent bad situations upstream, don't nudge in the moment (post-training fatigue kills real-time decisions)

- Carbs are the primary metric for a high-volume runner

- Trend-level accuracy, not obsessive precision

- Never punish absence — Apple Watch data flows passively on low days, re-entry is frictionless

- Don't over-engineer before validating habits

- Build features for real problems, not hypothetical ones



---



## Now (next session)



- [ ] Fix notesmd-cli PATH in voice-bot.service

- [ ] Deploy auto-tier evening check-in (20:00 check + 23:30 silence fallback)

- [x] Set up git + GitHub on Pi (backup + live context for Claude sessions)

- [ ] Create STATE.md / ARCHITECTURE.md / ROADMAP.md in repo



## Soon



- [ ] Design subjective daily check-in (morning + evening) to complement Apple Watch data

- [ ] Design for engagement oscillation: passive data on low days, frictionless re-entry

- [ ] Fix text handler: meal prefix detection not catching all patterns from sparring context

- [ ] Wire remaining handlers to SQLite where applicable

- [ ] `/foodweek` command (weekly food summary)



## Next



- [ ] Supermarket API integration — `SupermarktConnector` library (AH + Jumbo unofficial APIs). Bot generates list from stock gaps, user orders click-and-collect for pickup.

- [ ] `/shop` command (diff fridge vs nutrition needs)

- [ ] Recipes system for bot

- [ ] Finish appie-go OAuth login



## Later



- [ ] Qdrant self-hosted on Pi for 15-year journal archive (one-time embedding cost ~$0.54). Blocked on: splitting archive into individual note files.

- [ ] `graphthulhu` MCP server for Obsidian graph-based memory (Phase 2/3, after structured data flows reliably)

- [ ] Decommission Hetzner VPS (after Pi confirmed stable for extended period)

- [ ] Pattern detection: sleep + food → energy correlation, training + nutrition → performance

- [ ] Batch prep intelligence (Sunday meal prep assistant)

- [ ] Budget tracking (cost per meal, cost per gram of protein)



## Done



- [x] Migrate bot from Hetzner to Raspberry Pi 5

- [x] Refactor monolithic combined_bot.py into modular ~/nubot/ package

- [x] Voice journaling + Whisper transcription

- [x] Voice memo cleanup (Sonnet pass)

- [x] Meal logging to Obsidian + SQLite

- [x] "Log it" / "add it" short-circuit from sparring

- [x] /undo command

- [x] Sparring mode (food decision support)

- [x] Morning nudge with real health data from SQLite

- [x] Health Auto Export pipeline (Apple Watch → Pi → SQLite)

- [x] health_receiver.py + systemd service

- [x] Morning nudge conversation flow fix (Haiku classifier, auto-expire, proper message turns)

- [x] API key rotation (Telegram, Anthropic, OpenAI)

- [x] notesmd-cli installed and working from terminal

- [x] obsidian-headless running on Hetzner


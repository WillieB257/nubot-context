
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


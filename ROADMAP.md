
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


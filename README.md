# Tahoe OS — LLM Advisory Board Materials

This public repository contains sanitized briefing materials for the LLM advisory board reviewing Tahoe OS.

Tahoe OS is the operating system that runs a one-person real-estate brokerage + property-management business in Lake Tahoe, California. It runs on a single Mac Mini M4 with ~70+ launchd jobs, PostgreSQL + pgvector, Cloudflare Workers/Pages/R2, and multi-tier LLM routing (Kimi K2.6 / Haiku / Sonnet / Opus / local Ollama).

## Files

- **`00-briefing-v3.md`** — Comprehensive briefing covering business context, tech stack, prior state, tonight's autonomous build (Waves 1-7, ~40 subagents), current state, open decisions, and questions for the board.
- **`01-handoff-pm-proposal-fix.md`** — Parallel session handoff covering PM proposal generator phone fix + email watcher infinite-loop fix + Kent litigation state.
- **`02-jarvis-brief-red-team.md`** — Red-team audit of the morning brief email shipped 2026-05-29 17:19 PT, with proposed v2 structure.
- **`03-tristan-feedback-late-evening.md`** — Operator's direct feedback to the board, including the seven implied design principles.
- **`04-build-approach-lessons.md`** — Honest retrospective on tonight's 40+ subagent session: what worked, what failed, forward design principles.

## Sanitization

Client phone numbers, emails, full addresses, and dollar amounts are redacted. Tristan's public DRE-listed contact and case numbers (public court records) are kept. Architecture, file paths, and memory content are kept.

## Reading order

For new advisors: 00 → 03 → 02 → 01.

For depth on the autonomous build: 00 then 02.

## Status

This is a point-in-time snapshot from 2026-05-29 late evening. Live state continues in the private repos (trlt96145/tahoe-data, trlt96145/tahoe-state, trlt96145/tahoe-private).

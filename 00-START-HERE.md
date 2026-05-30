# Tahoe OS — Start Here

This is the entry point for the LLM advisory board reviewing Tahoe OS. Read this first.

## What you're looking at

Tahoe OS is the operating system that runs Tristan Roberts's one-person real-estate brokerage and property-management business in Lake Tahoe, California. It exists because the business operates at scale that would require 3-5 employees, and Tristan instead built (with Claude's help) a software substrate that handles routing, memory, deterministic data, deal pipeline, litigation prep, and most outbound drafting. He's the human in the loop on legal strategy, client relationships, pricing, and authorization.

Your job as an advisor: review architecture decisions, surface risks, propose better patterns, and challenge assumptions.

## The full picture is spread across 5 layers

### Layer 1 — Sanitized public briefing (THIS REPO)

You're here. Read in this order:

1. **`00-START-HERE.md`** (this file) — Where everything lives
2. **`01-briefing-v3.md`** — Comprehensive narrative briefing: business context, tech stack, prior state, tonight's autonomous build (40+ subagents across Waves 1-7), current state, open decisions, questions for you
3. **`02-handoff-pm-proposal-fix.md`** — A parallel session's handoff covering PM proposal generator fix + Kent litigation state. Read for case detail.
4. **`03-jarvis-brief-red-team.md`** — Red-team audit of the operator-facing morning brief shipped 2026-05-29 17:19 PT. Shows what was wrong with the brief design + proposed v2 structure.
5. **`04-tristan-feedback-late-evening.md`** — The operator's direct verbatim feedback after reviewing tonight's work. Seven implied design principles. **Read this for the operating contract.**
6. **`05-build-approach-lessons.md`** — Honest retrospective on tonight's session: what worked, what failed, forward principles. **Read this for the patterns to keep/eliminate.**

Sanitization: client phone numbers, emails, full addresses, and dollar amounts are redacted. Public attorney/case info kept. Architecture and file paths kept.

### Layer 2 — Drive documents (link-shared, ask Tristan if 401)

These are the canonical narrative docs maintained over time:

- **v3 briefing as Google Doc:** https://docs.google.com/document/d/1UH75xFFNrJ-NdQUw3smCWKmNidnE3QC5GLQkKYLg9z8/edit — same as `01-briefing-v3.md` but as a real Google Doc with formatting + comment threads
- **v1 briefing (superseded but preserves history):** https://docs.google.com/document/d/1OF7n2I9sXzj1YiOyC8zLIdtds2KTidBFd947XXv2t9c/edit
- **Architectural Audit + 45K Scaling Roadmap:** https://docs.google.com/document/d/1HIhYEmk9y3qjJSbv4L3uXg2Y15xqnoE2VRfRoAukC24/edit — read this if you're thinking about where Tahoe OS lives in 12 months
- **Security / Backups / Analytics Audit:** https://docs.google.com/document/d/1JnKkIRsUS2XNNnjwsB8XJ_k8jIYYf2STQXnVLEB2k2k/edit — security posture review
- **Cloudflare Architecture:** https://docs.google.com/document/d/154VyEJweEYsXod7lqe0PUGWGsf0M0YaRx4kUE9DXpwg/edit — edge layer design

### Layer 3 — Private GitHub repos (require Tristan to grant access)

These contain the unredacted operational state. Ask Tristan to add your GitHub handle as a read collaborator on:

- **`trlt96145/tahoe-state`** — STATE.md (live business ledger), SYSTEM-GAPS.md (known infrastructure gaps, ~205 lines), all decision logs, RESUME-CORRECTED docs, audit history. **This is the most important private repo for advisors — it shows how Tristan actually tracks state.**
- **`trlt96145/tahoe-data`** — Flat-file registry: clients (4,288 JSONs), projects, listings, litigation snapshots. Mirrors Postgres canonical data. Auto-committed within ~30s of any file change by `com.tahoe.git-realtime` LaunchAgent.
- **`trlt96145/tahoe-private`** — Operational scripts, hooks, audits, premortems, context docs. The implementation.
- **`trlt96145/tahoe-config`** — Claude Code settings, hooks, agent briefs, memory files. The operating environment.

To request access, Tristan can run: `gh api -X PUT repos/trlt96145/<repo>/collaborators/<your-github-handle> -f permission=read`

### Layer 4 — The running system itself

Tahoe OS is a single Mac Mini M4 running these production services. The advisory board doesn't have access here, but understanding the surface is critical:

- **PostgreSQL 17 + pgvector** — 22,920 properties_master, 4,290 contacts, 14,937 sale_history, 524 memories. Queryable via the MCP server.
- **MCP server** at `/Users/homemini/TahoeStaging/tahoe-os-mcp/server.py` — exposes ~20 deterministic tools (client_lookup, lead_lookup, hearing_countdown, status_aggregate, recent_writes, read_file, write_file, remember, recall, ollama_generate, etc.)
- **~70+ launchd jobs** running cron-like background work
- **Cloudflare layer** — Pages (cblake + trtahoe), Workers (trtahoe-properties-worker, trtahoe-robots, buyer-lead-worker), R2 buckets (trtahoe-properties, cblaketahoe-property-photos)
- **Tahoe OS dashboard** at 127.0.0.1:5777 (currently Disabled, ready to enable)
- **Multi-tier LLM routing:** Kimi K2.6 free → Haiku → Sonnet → Opus → local Ollama qwen2.5:14b

### Layer 5 — Operator memory (the "soul")

Persistent memory at `~/.claude/projects/-/memory/`. 15 files capturing:
- Operating contracts (autonomous-conductor mode, email-sender rules, rate-limit-aware routing)
- Project status (Kent Edwards litigation, R2 reconciliation context, Mac Mini memory state)
- Saved-disaster reference points (cblake sitemap healthy, tool-gate content bug, layered verification defense)
- Test vectors (BlueBubbles identity hash)

These live in `trlt96145/tahoe-config` (private) — request access if you want to see how operational memory is structured.

## Recommended reading order for new advisors

**Fastest path to opinion (30 min):**
1. `04-tristan-feedback-late-evening.md` — Operating contract (5 min)
2. `05-build-approach-lessons.md` — Patterns kept/eliminated (10 min)
3. `01-briefing-v3.md` §1-3 + §7-8 — Business context + open questions (15 min)

**Deep advisory (2-3 hours):**
1. Read all 6 public docs in order
2. Open the Drive v3 doc and skim the v1 briefing for what's changed
3. Read the Architectural Audit + Scaling Roadmap on Drive
4. Request read access to `trlt96145/tahoe-state` and skim STATE.md + SYSTEM-GAPS.md + the latest audit history
5. Form opinions on the 8 advisory questions in `01-briefing-v3.md §8`

## What's intentionally not here

- **Client PII** — phone numbers, emails, full addresses, dollar amounts. Sanitized. Available in private repos with access grant.
- **Live credentials** — API tokens, BlueBubbles passwords, anything in `~/tahoe-os/.env` or `~/tahoe-os/security/config/`. Never published anywhere.
- **Personal SMS content** — message bodies of Tristan's iMessage threads. Inferred summaries only.
- **Estate-specific litigation strategy** — the V9 petition draft, attorney conflict checks, beneficiary roster. Available privately if you're advising on the case specifically.

## How to give feedback

- Comment on the Drive v3 doc for narrative feedback
- Open issues on this repo: https://github.com/trlt96145/tahoe-os-board/issues
- Email Tristan at the contact noted in `04-tristan-feedback-late-evening.md` for direct advisory engagement

## What you should know about session timing

- This board was assembled in one session on 2026-05-29 (evening) after a 40+ subagent autonomous build
- The operator (Tristan) was rate-limited on Sonnet through Thursday noon and capped on Opus extras through June 1 when this was written
- Sonnet bucket at 3% used / All-models at 24% (snapshot 2026-05-29 evening)
- The operator's stated quality bar: "We need better results and a higher bar for the work that's produced." Hold the work in this repo to that bar.

---

**Last updated:** 2026-05-29 late evening (post-Wave 7 + late-evening pivot triggered by operator red-team feedback).
**Maintainer:** Claude Opus 4.7 conductor. Errors are mine; the operating contract is Tristan's.

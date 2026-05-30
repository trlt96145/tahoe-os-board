# Tahoe OS — Start Here

This is the entry point for the LLM advisory board reviewing Tahoe OS. Read this first — the whole document is designed to be readable in under 25 minutes for a new advisor. Part I gives you the full story (why, vision, challenges, successes). Part II is the index of where everything else lives.

---

# PART I — The Story

## §1. Why Tahoe OS exists

Tristan Roberts is a one-person operation in Lake Tahoe — California DRE #01259729, Coldwell Banker affiliated, running both a real-estate brokerage (cblaketahoe.com) and a property-management arm (trtahoe.com). His operational load looks like 3-5 employees worth of work: 254 active MLS listings, ~22,920 properties he could write SEO landing pages for, ~4,300 contacts (clients, leads, vendors, contractors), recurring property-management projects with deposit-driven cash flow, and a major active probate litigation as primary beneficiary (Kent Edwards estate, ~$[REDACTED]+, two California counties, two hearings within 9 weeks). He doesn't hire. He builds.

Tahoe OS exists because hiring a paralegal + a marketing assistant + a project coordinator + a CRM admin would cost more than the leverage Claude provides, and the systems he's been building since [GAP — need Tristan input: exact start date of the Tahoe OS project; earliest persistent-memory entries date to March 2026 but the infrastructure they reference clearly predates them] now do enough of that work that Tristan can stay solo.

### What life was like *before* Tahoe OS

Inferred from what the system replaces:

- **MLS daily monitoring + photo scraping was manual.** Listings, photos, comp pulls — done by hand against Paragon.
- **Per-property landing pages didn't exist.** No SEO surface across the 22,920 parcels in the county. Zero indexable pages.
- **Inbound SMS classification was happening in his head between phone calls.** Every new text — lead, vendor, client, spam — needed live triage by the operator.
- **Litigation document prep was being done in Word + email.** No version control, no exhibit tracking, no statute cross-references.
- **Contact memory was Apple Notes + iMessage threads.** No deterministic "have I talked to this person?" answer existed.
- **No "did I follow up with X?" lookup existed.** Followups depended on Tristan remembering they were owed.
- **Money state was tracked in his head.** Deposits in, amounts owed, who was overdue, which proposals were live — all carried mentally.

### The operating constraint that drives every design decision

**Tristan-time per day is the binding constraint. Every minute Tahoe OS automates is leverage. Every minute Tahoe OS wastes is leverage destroyed.**

That single sentence is why the architecture looks the way it does — deterministic MCP tools instead of free-form LLM lookups, audit-verify-gate instead of trusting subagent claims, shadow-mode-first on every external-send capability, and a hard "Sonnet builds, Opus audits" pair pattern on anything load-bearing.

[GAP — need Tristan input: any personal-context framing he wants on the public board (family situation, financial driver, time horizon, why this matters beyond the obvious). The doc reads as purely operational without that.]

---

## §2. Vision for the ultimate build

### Stage 3 (the next major arc)

Bring the currently-disabled Stage 1 LaunchAgents live, in this exact order — each one a one-way ratchet, with the most consequential capability last:

1. **Jarvis morning brief** (read-only)
2. **Dashboard** (read-only HTTP at 127.0.0.1:5777)
3. **Voice transcription** (whisper.cpp 1.8.4, already validated)
4. **Inbound classifier** (BlueBubbles iMessage triage)
5. **BlueBubbles ingest non-shadow** (still receiving, not sending)
6. **Send-approved drafts daemon** (last — this is the first external-send capability)

### Operator-facing surface vision

- Every brief leads with **money** (deposits-in → finish-to-collect → proposal pipeline). Not status. Not noise. Money.
- Every contact reference is **multi-source merged** (Postgres + Gmail + Drive + iPhone calls + voice notes) BEFORE classification — not as an afterthought.
- Every "ready for X" claim carries an **acceptance-criteria check**. No more "draft labeled ready" turning out to contain "REMOVE BEFORE FILING."
- Every question to Tristan includes: simple explanation + bigger picture + suggested answer. Reduce his cognitive load per decision.
- Brief refuses to send if §1 contradicts §3 (cross-reference internal consistency check). The Oak Street failure (§3.9 below) must be structurally impossible.

### Integration vision

- **iPhone calls + voicemails surfaced in real-time** — one iPhone setting flip away. CallHistory.storedata is already readable.
- **PM proposal pipeline fully wired** — email arrives → quote generated → deployed to Cloudflare URL → client signs → deposit received → invoice fires → completion → balance collected. Currently partial.
- **22,920 trtahoe property pages all live.** Currently 17% of R2 filled (~3,772 / ~22,920). Each page substantively distinct (Jaccard target 0.40, currently 0.765 after tonight's enrichment pass).
- **4,976 cblake pages with buyer-rep CTA dofollow link to trtahoe.** Patch shipped tonight, deploy gated on `/Volumes/TahoeStaging` volume remount.
- **Cross-LLM verification pass on high-stakes work.** Pain Index audit corpus is ready, blocked on Workers AI write token.
- **Memory architecture matures** — tag re-probe-before-trust vs durable-fact, semantic search via Chroma index, automated hygiene passes.

### Operating model vision

- **Audit-verify-gate is non-bypassable structurally** (not instruction-enforced). A subagent cannot route around it.
- **Sonnet builds + Opus audits** as the default pair pattern on anything load-bearing.
- **Cost-aware routing** across Kimi (free) / Haiku / Sonnet / Opus / local Ollama. Tier 0 absorbs as much triage volume as possible.
- **Public LLM-board review as a regular feedback loop.** Not one-off. This board itself becomes a recurring sanity check.
- **A quality probe runs before any task is marked complete.** Output count tracked, output quality measured, no task closes silently.

### 12-month architectural target (open question for the board)

The candidate end-state, but the board should challenge it:

- Same Mac, **heavy generators off-machine** (CF Workers + Modal/Fly)
- **Mirror to a small Hetzner/OVH** for redundancy
- Split: **Postgres + canonical data stays on Mac** (TCC + iMessage requirements), all background work moves to cloud
- Accept this is single-operator software and lean into **"Mac is prod"**

---

## §3. Past challenges — what almost killed it / what cost time

Honest list. Documented evidence behind each.

### Near-disasters caught at the last moment

1. **cblake sitemap fabricated-slug "fix" (2026-05-29 AM)** — would have destroyed 21,600 live indexable URLs based on a fabricated slug-format claim. Saved by a different subagent's 30-URL spot-check (all returned 200). This incident codified the audit-verify-gate hard rule that now sits at the top of CLAUDE.md.

2. **R2 reconciliation against partial 3-city dist (2026-05-29 PM, tonight)** — local dist was 2,977 files (Carnelian Bay + Kings Beach + Alpine Meadows only); R2 had live 200s for Truckee/Tahoe City/Tahoma. Naive reconcile would have deleted thousands of live indexable URLs. Saved by audit-gate + live HTTP spot-check before any deletion.

### Multi-hour productivity losses

3. **May 16 lost day** — tmux session died silently because the LaunchAgent wasn't configured to ensure persistent sessions. Fixed via two LaunchAgents (`com.tahoe.tmux` + extended `com.tahoe-os.tmux-server.plist`).

4. **Conductor "acks but doesn't execute" bug (2026-05-24)** — Tristan approved a major build at 11:03, conductor said "say the word and I'll spin up the agents," never actually spawned anything. Root cause: no task persistence between turns. The exact failure the queued-task system was designed to prevent — ironic.

5. **tool_gate content-denylist bug** — the gate matches command CONTENT not just argv, so heredocs that mention "shutdown" or "rm -rf" get falsely blocked. Cascading effect: subagents writing legitimate files with those strings get blocked, substitute placeholder text, produce data-integrity bugs. Required two surgical repair passes on `data/system/playbook.md` alone.

6. **Telegram MLS notification noise** — silenced via env flag after spam volume became unworkable.

### Quality-bar failures

7. **Resume brief inflation** — listed 8 audit files (2 didn't exist), gave the wrong Knowlton draft ID, overstated R2 fill state. Trust drift. The brief itself was the artifact that needed audit.

8. **False-ready signal on V9 petition** — passed forward as "ready for Knowlton" but was a draft .md with "REMOVE BEFORE FILING" section still IN the body. Tristan caught it on review.

9. **Jarvis brief self-contradiction** — §1 and §10 recommended action on Oak Street while §3 in the same brief showed the buyer had withdrawn. No internal cross-reference check between sections of the same artifact.

10. **Output count tracked, quality not tracked** — 40+ subagents dispatched tonight; operator's feedback: *"We need better results and a higher bar."* The metric was wrong.

### Infrastructure decay

11. **`data/.git` remote diverged 47 ahead / 10 behind** by tonight — auto-push silently failing, offsite mirror broken. Fixed via clean rebase this evening.

12. **Memory + session-notes Drive rsync broken since 07:13 (2026-05-29)** — rsync `-a` archive mode's metadata flags incompatible with the Google Drive File Provider sandbox. Fixed with `-rlt --no-perms --no-owner --no-group --omit-dir-times --modify-window=1`.

13. **14 of 65 LaunchAgents failing** — two root causes: `/Volumes/TahoeStaging` unmounted, and services consolidated into `server.py` but stale registrations remain.

14. **`ctx_warn_sentinel` staged but unwired** since 2026-05-28 — sentinel helper had no caller. Wired tonight, but also discovered the script lacked the `+x` bit (silent fail via `|| true`).

---

## §4. Current challenges — what's blocking right now

In priority order.

### Time-bound (deadlines)

- **June 12, 2026 (14 days) — Nevada County hearing.** Knowlton retainer Gmail draft ready (`r-4931712662035923460`). **Authorization needed.**
- **June 10, 2026 (12 days) — telephonic-appearance deadline** (form RA-010) for the June 12 hearing if Tristan or Knowlton aren't appearing in person.
- **5/30/2026 (1 day) — Mary at Montclair quote needed** ($[REDACTED]) before she leaves Friday.
- **Lumber pricing owed to Carol at 1841 Toboggan** (TimberTechAZEK vs Trex) — promised 5/26. **OVERDUE.**
- **July 31, 2026 (63 days) — Placer County hearing.** V9 supplemental petition not Knowlton-ready (17 documented defects, exhibits missing, "REMOVE BEFORE FILING" section still in body).

### Money pipeline gaps

- **Mike's deck** — $[REDACTED] deposit ARRIVED 5/28, balance $[REDACTED] due at completion. Push to finish.
- **Julia's house** — $[REDACTED] deposit ARRIVED 5/28. Send WIP photo.
- **`jobs` table has ZERO rows for Mike + Julia** despite active deposits. Dashboards blind.
- **`projects` table empty for ALL Tier A/B work.** Cross-project tracking broken.
- **11 of 21 charity beneficiaries have had zero outreach** as of 5/27.

### Infrastructure blockers (Tristan-decision tier)

- **`/Volumes/TahoeStaging` unmounted** — blocks cblake deploy + discord-bot + command-app + maintenance-update.
- **R2 fill at ~17%** (3,772 of ~22,920) — cblake dofollow CTA links 404 until R2 fills.
- **Workers AI write token missing** — Pain Index audit corpus (6.9M tokens, 17 batches) ready to fire, blocked.
- **GitHub Pro upgrade** — branch protection on private repos.
- **`BLUEBUBBLES_PASSWORD` env** — live SMS pipeline gated.
- **Gemini 2.5 Pro API key** — cross-LLM verification gated.
- **Gmail forwarding filter for `tahoeos.agent@gmail.com`** — must be done in web UI; service account can't impersonate personal Gmail.

### Quality / classification

- **4,288/4,288 client JSONs use legacy schema** (aspirational `meta.schema` never migrated to).
- **pytest 12/13** — `test_duplication_gate` Jaccard at 0.831 (target 0.40, currently down to 0.765 after tonight's enrichment).
- **3 phone-only stub contacts in DB still un-enriched** (Boris HIGH-conf, Sue Pearlstein HIGH-conf, Dale CONFIRMED PERSONAL — should be set `dnc=true` before next reactivation wave).
- **`write_log.jsonl` docstring claims "audit trail"** but only 2 scripts populate it.
- **`meta.schema` describes shape no client record uses.**

---

## §5. Past successes — what's actually live and working

This is the substrate. Everything below has been operating since at least May 2026 unless otherwise noted.

### Data foundation

- **22,920 `properties_master`** with full ownership, assessed values, flags
- **4,290 contacts** with `lead_score`, `momentum_score`, `lifetime_revenue`
- **14,937 historical sales** (comp engine source)
- **524 memories** with dual indexing (tsvector full-text + pgvector 384-dim cosine)

### Routing and execution

- **Multi-tier LLM routing** — T0 Kimi free → T1 Haiku → T2 Sonnet/Gemini → T3 Opus → Bulk Ollama
- **MCP server with ~20 deterministic tools** (`tahoe_client_lookup`, `tahoe_lead_lookup`, `tahoe_hearing_countdown`, `tahoe_status_aggregate`, `tahoe_recent_writes`, etc.)
- **~70+ LaunchAgents** handling cron-like background work
- **`com.tahoe.git-realtime`** auto-commit watcher (~3s latency, ~4,300 contacts syncing)
- **`data/.git` auto-commit watcher** (~30s latency)

### Web + edge

- **Cloudflare zone rules** (Transform + Cache), `trtahoe-properties-worker`, `trtahoe-robots`, R2 buckets
- **Cross-domain slug map** — 22,608 entries linking cblake ↔ trtahoe, 0 orphans

### Messaging + voice

- **BlueBubbles iMessage bridge** → Postgres → classifier → drafted replies (shadow mode)
- **Voice memo transcription** via whisper.cpp 1.8.4
- **Jarvis morning brief generator** (13K HTML, 10 sections)
- **Tahoe OS dashboard** (FastAPI on 5777, ready-to-enable)

### Safety + integrity

- **`audit-verify-gate.py`** with hard rule in `CLAUDE.md`
- **`path_lock.py`** file mutex
- **Backup pipelines** (voice + memory + Postgres → Drive)
- **Uptime ping**
- **PM proposal system** (CLI + email watcher + template + LaunchAgent + Cloudflare Pages deploy)

### Stage 2 cross-domain link equity (shipped earlier this week)

- **4,975 cblake JSONs patched** with Buyer Representation tab (DRE 01259729, NAR disclosure, SMS CTA, dofollow link to matching trtahoe Property Care Guide)
- **trtahoe footer nofollow removed**
- **UTM attribution wired** into trtahoe buyer-lead form + buyer-lead-worker

---

## §6. Current successes — what shipped today / this week

### Shipped today (2026-05-29)

- **Fix-pair C complete** — 3 security patches (prompt-injection sanitizer with nonce-delimited untrusted block, identity hash regression assert, BB password URL→body)
- **MCP server reloaded** (PID 59032 → 50946) picked up Fix-pair D security patches
- **Knowlton draft truth located** — actual `r-4931712662035923460`; brief had wrong ID
- **Coldwell Banker template address corrected** (WebSearch-verified)
- **10/11 stale LaunchAgents booted out** — ~600k/day spam log lines stopped
- **`data/.git` remote divergence repaired** — offsite mirror restored
- **Per-zip enrichment +30 variants** — Jaccard 0.83 → 0.765
- **audit-verify-gate enhanced** — launchd-bootout category + tighter path-token detection
- **`deploy-staged-pages.py` patched** for buyer_rep tab render (5/5 smoke tests pass)
- **`memory-backup.sh` rsync FIXED** — Drive byte-parity restored
- **`ctx_warn_sentinel.sh`** finally wired + truncate + chmod +x (PID 90787 picks up sentinel every poll)
- **trtahoe-pages UNSTUCK** — successful deploy 14:13 PT, 16-day stall ended
- **PM proposal generator phone fix** (parallel session) + email watcher infinite-loop fix
- **Phone-stub enrichment proposal** — Boris/Sue Pearlstein/Dale identified across multiple sources
- **V9 petition Knowlton-ready audit** — 17 defects documented, all 4 spot-checked statute citations PASS
- **Active money pipeline doc** — Mike's deck + Julia's house deposits + Mary Montclair + Carol Toboggan + Dale-confirmed-personal
- **iPhone call/voicemail integration investigation** — `CallHistory.storedata` readable, just needs iPhone setting flip

### This v1 of the LLM advisory board feedback loop

- **Public board repo:** https://github.com/trlt96145/tahoe-os-board (6 sanitized docs)
- **v3 briefing** as Google Doc (single canonical narrative)
- **Build-approach lessons doc** — honest retrospective on what worked / what failed / forward principles

---

# PART II — Where to find more

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

- **PostgreSQL 17 + pgvector** — 22,920 `properties_master`, 4,290 contacts, 14,937 `sale_history`, 524 memories. Queryable via the MCP server.
- **MCP server** at `/Users/homemini/TahoeStaging/tahoe-os-mcp/server.py` — exposes ~20 deterministic tools (`client_lookup`, `lead_lookup`, `hearing_countdown`, `status_aggregate`, `recent_writes`, `read_file`, `write_file`, `remember`, `recall`, `ollama_generate`, etc.)
- **~70+ launchd jobs** running cron-like background work
- **Cloudflare layer** — Pages (cblake + trtahoe), Workers (`trtahoe-properties-worker`, `trtahoe-robots`, `buyer-lead-worker`), R2 buckets (`trtahoe-properties`, `cblaketahoe-property-photos`)
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
- The operator's stated quality bar: *"We need better results and a higher bar for the work that's produced."* Hold the work in this repo to that bar.

---

**Last updated:** 2026-05-29 late evening (post-Wave 7 + late-evening pivot triggered by operator red-team feedback; expanded with full narrative per operator request).
**Maintainer:** Claude Opus 4.7 conductor. Errors are mine; the operating contract is Tristan's.

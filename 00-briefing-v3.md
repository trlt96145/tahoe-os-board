# Tahoe OS — Comprehensive Briefing for LLM Advisory Board

**Version 3 — 2026-05-29 evening, post Wave 1-7 + integrated parallel-session HANDOFF (PM proposal + phone fix)**
**Prior versions:**
- v1: https://docs.google.com/document/d/1OF7n2I9sXzj1YiOyC8zLIdtds2KTidBFd947XXv2t9c/edit
- v2: local at `~/tahoe-os/data/system/advisory-briefing-v2-2026-05-29.md`
**Operator:** Tristan Roberts — DRE #01259729

---

## TL;DR

Tahoe OS is the operating system that runs Tristan Roberts's real-estate brokerage and property-management business in the Lake Tahoe region. It's a stack of Postgres, Python scripts, Cloudflare Workers/Pages, multiple LLM tiers, and ~70+ launchd jobs running on a single Mac Mini M4. It exists because Tristan handles double-digit active clients, ~23,000 indexable property pages on two domains, an active probate litigation (Kent Edwards estate, two California counties, two hearings inside 9 weeks), and ~4,300 contacts — without any employees. The system itself is the leverage that lets him operate at that scale.

This briefing brings advisory LLMs up to speed on: (a) what was built before tonight, (b) what happened tonight in a sustained autonomous build, (c) the parallel-session work that landed PM-proposal pipeline hardening and a phone-number correction, and (d) where the system is now, including the decisions Tristan needs to make next.

---

## Section 1 — Who Tristan is and what the business is

Tristan Roberts (DRE #01259729) operates two related real-estate properties in the Lake Tahoe / Truckee area:

- **cblaketahoe.com** — the brokerage / property-services side. Coldwell Banker affiliated. Branch address: 475 N Lake Blvd Unit 102, Tahoe City, CA 96145.
- **trtahoe.com** — the property-pages / SEO side. Per-property landing pages with buyer-rep CTAs and Property Care Guides. Designed for organic search capture across ~22,920 Tahoe-area parcels.

He also handles: 254 active MLS listings with photos; ~4,300 contacts (clients, leads, vendors, contractors); and a major active probate litigation as primary beneficiary (Kent Edwards estate, ~$[REDACTED] across two California counties, two hearings in the next 9 weeks).

He works solo from a Mac Mini M4 (16GB RAM). No employees. The operating constraint is "Tristan-time per day" — every minute Tahoe OS automates is leverage.

---

## Section 2 — Why Tahoe OS exists

The original problem (~Q1 2026): too many simultaneous responsibilities for one person.

- MLS daily monitoring + photo scraping
- 22,920+ properties needing SEO landing pages
- Inbound SMS classification (clients, vendors, leads, spam)
- Litigation document prep + tracking
- Property maintenance scheduling + invoicing
- Persistent memory: who did Tristan talk to about what, when

The goal: a software substrate that handles routing, memory, deterministic data access, and most outbound communication — so Tristan only spends time on high-judgment work (legal strategy, client relationships, pricing decisions, signing things).

The way it works:
- Postgres 17 + pgvector is the canonical store. 22,920 properties_master rows, 4,290 contacts, 524 memories, 14,937 sale_history.
- Multi-tier LLM routing (Kimi K2.6 free → Haiku → Sonnet/Gemini → Opus) for cost-efficient task dispatch.
- Postgres → git → Cloudflare Pages/R2 for SEO content.
- MCP servers expose deterministic tools (client_lookup, hearing_countdown, status_aggregate, etc.) so the conductor doesn't hallucinate.
- BlueBubbles bridges iMessage → SQL → classifier → drafted replies (currently in shadow mode, gated).
- ~70+ launchd jobs run cron-like background work (backups, generators, watchers, Telegram/Discord bots, dashboards).

---

## Section 3 — Tech stack & key conventions

**Hardware:** Mac Mini M4 16GB. Single machine. Memory pressure is a real constraint — at session start the Mac was at 176MB free; end of session 72MB free + 7.95GB swap after sustained parallel work.

**Languages / Runtimes:** Python 3.14, Bun (TypeScript workers), Bash. Local Ollama for free bulk work (qwen2.5:14b).

**Database:** PostgreSQL 17 + pgvector 0.8.2. 71 tables. pytest uses transactional rollbacks; tests never mutate prod data.

**Storage:**
- `~/tahoe-os/data/.git` — flat-file registry (4,288 clients, 5 projects, 1 listing, 2 litigation, 32 system files). Mirrored to private GitHub trlt96145/tahoe-data. Auto-committed by `com.tahoe.git-realtime` LaunchAgent within ~30s of file change.
- `~/tahoe-os/state/.git` — STATE.md, SYSTEM-GAPS.md, decision logs.
- `/Volumes/TahoeStaging` — external volume, currently UNMOUNTED, blocks 4 services as of tonight.
- Cloudflare R2 — `trtahoe-properties` bucket (3,772 objects, target ~22,920), `cblaketahoe-property-photos` (96,281 objects), `tahoe-voice-memos`.

**Web:**
- cblaketahoe.com — Cloudflare Pages static site, last deploy 2026-05-25.
- trtahoe.com — Cloudflare Pages was frozen on May 13 → unstuck today (successful deploy 14:13). Property pages now served by a Worker (`trtahoe-properties-worker`) + R2 path; Pages serves the rest.
- Cloudflare zone rules strip `x-robots-tag` and set `Cache-Control` on `text/html` 200s. Verified live tonight.

**Conductor pattern (Claude Code):**
- Tristan runs Claude Code on Opus 4.7 (medium effort) in "router-only" mode. Conductor delegates execution to Sonnet / Haiku / Opus subagents via the Agent tool.
- `~/tahoe-os/hooks/tool_gate.sh` enforces this structurally — blocks `Write` / `Edit` / `Bash` for the conductor process; subagents bypass via two-claude-ancestor detection.
- Known bug: tool_gate denylist matches command CONTENT (not just argv), so heredocs mentioning words like "bootout" or "shutdown" get falsely blocked. Workarounds documented (token-split, python3 heredoc, mcp__filesystem__edit_file).

**Multi-tier LLM routing:**
- T0 — Kimi K2.6 on Cloudflare Workers AI. Free. Triage + simple Q&A.
- T1 — Haiku 4.5. Cheap. DB lookups, simple subagent tasks.
- T2 — Sonnet 4.6 / Gemini 2.5 Pro. Medium. Drafts, multi-step subagent work.
- T3 — Opus 4.7. Expensive. Architecture, audits, legal reasoning.
- Bulk — local Ollama qwen2.5:14b. Free. Mass text generation.

Tristan is currently rate-limited on Sonnet through Thursday noon and capped on extras through June 1, so all-models routing is the active priority. As of this evening, weekly bucket consumption: all-models 24%, sonnet-only 3%, reset Thursday 11:59 AM PT.

**MCP server (Model Context Protocol):**
Located at `/Users/homemini/TahoeStaging/tahoe-os-mcp/server.py`. Exposes 20+ tools (tahoe_client_lookup, tahoe_lead_lookup, tahoe_hearing_countdown, tahoe_status_aggregate, tahoe_recent_writes, tahoe_read_file, tahoe_write_file, tahoe_remember, tahoe_recall, tahoe_ollama_generate, etc.). Path-traversal guard at tools/files.py:11-12 plus ALLOWED_DIRS allowlist. Reloaded tonight (old PID 59032 → new PID 50946).

---

## Section 4 — State as of this morning (entering the autonomous build)

Two prior overnight build stages had already shipped (per the resume brief that started tonight's session):

**Stage 1 infrastructure built before tonight:**
- data/.git flat-file registry seeded with Kent Edwards + Oak Street
- Stop + UserPromptSubmit hooks
- BlueBubbles ingest (shadow mode)
- Voice memo transcription pipeline (whisper.cpp 1.8.4 + ggml-base.en.bin)
- Jarvis morning brief generator (12K HTML, 10 sections)
- 5 new MCP tools (client_lookup, lead_lookup, hearing_countdown, status_aggregate, recent_writes)
- Inbound text classifier (Ollama-backed)
- Two-way text approval grammar (OK / EDIT / SKIP / STATUS)
- Voice / memory / postgres → Drive backup LaunchAgents
- Uptime ping
- Cross-domain slug map (22,608 entries linking cblake ↔ trtahoe)
- audit-verify-gate.py + hard rule in CLAUDE.md
- Tahoe OS dashboard (port 5777)
- path_lock.py file mutex

**Cloudflare layer pre-tonight:**
- Zone Transform Rule + Cache Rule + Worker for /robots.txt
- trtahoe-properties-worker deployed (route trtahoe.com/properties/*)
- R2 bucket trtahoe-properties (3,080/22,421 uploaded at last check)
- buyer-lead-worker built (not deployed — needs CF token + D1 + DNS)
- edge-router-worker scaffold (staging only)

**Known open work entering tonight:**
- R2 upload incomplete (3,080/22,421)
- MCP server reload pending (PID 59032 running pre-Fix-pair-D code)
- Fix-pair C blocked (3 security patches not applied)
- Cross-page Jaccard 0.83 (target 0.40 — needs more enrichment variant rotation)
- Cblake WordPress deploy mechanism for 4,975 patched JSONs
- External blockers: GitHub Pro for branch protection, CF Workers AI write token for Pain Index, BB password for live SMS, Gemini 2.5 Pro API key for cross-LLM verify

**Active litigation (Kent Edwards):**
- Nevada County (defensive) Case PR0001028 Dept 6 — hearing **June 12, 2026 09:00** (14 days). Goal: stay/continuance in favor of senior Placer court.
- Placer County (offensive) Case S-PR-0013003 Dept 40 (Judge Jacques) — hearing **July 31, 2026 08:30**. V9 Supplemental Petition near-final.
- Knowlton retainer Gmail draft ready to send (Tristan to authorize).

---

## Section 5 — Tonight's autonomous build (2026-05-29 evening, Waves 1-7)

**Operating mode:** autonomous. Per `feedback-autonomous-conductor` memory, AskUserQuestion is banned; conductor picks A+ build-quality options, executes, discloses. Only no-go items: external actions prejudicing Kent litigation, public client-data leaks, verified-no-backup destructive actions, sends to clients/vendors/legal-related parties.

**Pattern:** Sonnet builds + Opus audits, 4-6 pairs in parallel per wave. ~40 subagents dispatched over ~3 hours.

### Wave 1 (16:50 PDT) — Unblock pipeline

- MCP server reloaded — old PID 59032 → new PID 50946 (clean uvicorn restart, .env chmod 600 verified).
- Fix-pair C re-executed (3 security patches):
  - Prompt-injection sanitizer in `classify-inbound-texts.py` with per-call nonce delimited block + `re.subn`-count-based was_sanitized flag (4/4 smoke tests pass).
  - Identity hash regression assert in `bluebubbles-ingest.py` (test vector `sha256("+1[REDACTED]|") = 45af52dabe8fcc0f...`).
  - BB password moved from URL query to JSON body in `send-approved-text-drafts.py`.
- Fix-pair D ground-truth audit — initial grep miss made it look like patches didn't ship; second audit confirmed all 3 at correct paths (path traversal guard at `tools/files.py:11-12`, voice IGNORECASE at `transcribe-voice-memos.py:61-67`, Jarvis legal redaction at `jarvis-morning-brief.py:459`).
- **R2 upload HALTED — major save.** Opus audit caught an 87% source-dir shrinkage (22,421 → 2,977). Investigation showed dist regenerated to a partial 3-city set (Carnelian Bay + Kings Beach + Alpine Meadows only). R2 was already serving live 200s for Truckee / Tahoe City / Tahoma. Naive reconcile would have deleted thousands of live indexable URLs — same shape as the 2026-05-29 morning cblake sitemap save. audit-verify-gate stopped it.

### Wave 2 — Cleanup

- Coldwell Banker address corrected to `475 N Lake Blvd Unit 102, Tahoe City, CA 96145` (WebSearch-verified — initial subagent's model-recall guess of "581 N. Lake Blvd" was wrong).
- Knowlton draft existence verified via Gmail API: draft `r-7139802448190225283` from the resume brief is FALSE (Gmail 404). Actual draft is `r-4931712662035923460` in Tristan's mailbox, drafted 17:06 PT today, subject "Kent Edwards Estate — June 12 + July 31 hearings, $[REDACTED] for your help." STATE.md + memory + ledger corrected. (Note: tahoeosagent mailbox doesn't exist yet — workspace seat pending.)

### Wave 3 — Audits + parallel builds

- Per-zip enrichment variation: +30 new variants across enrichment / seasonal / fire_calendar. Jaccard 0.83 → 0.765 same-ZIP (still above 0.65 ceiling — needs seasonal_calendar list extension + template chrome dedup next pass).
- Cblake WP deploy proposal: discovered cblake is NOT WordPress (it's CF Pages static). Path C recommended — ~20-line patch to existing deploy-staged-pages.py.
- Resume brief ground-truth audit: reliability ~77% (46/60 bullets verified). Corrected ground-truth doc at `~/tahoe-os/state/RESUME-CORRECTED-2026-05-29.md`.
- Memory health audit: 13 files audited (1 skipped due to concurrent edit), 10 accurate, 2 minor staleness corrected.
- Buyer-rep CTA verification on cblake JSONs: 20/20 structurally perfect (zip, buyer_rep tab, DRE 01259729, NAR settlement disclosure, SMS CTA, dofollow trtahoe link, slug-map match).

### Wave 4 — More audits + reconciliation

- Worker 200 vs 404 contradiction RESOLVED: both earlier probes were correct; R2 is ~17% filled (~3,772 objects of ~22,920). Sampling-frame artifact, not deploy flap.
- LaunchAgent audit: 65 active plists + 24 disabled + 7 .bak = 96 total; 67 loaded; 14 failing. Two root causes drive 13/14 failures: (a) /Volumes/TahoeStaging unmounted, (b) services consolidated into server.py but stale registrations remain.
- Hooks audit: tool_gate content-denylist bug STILL PRESENT (line 60-89 of tool_gate.py); ctx_warn_sentinel staged but unwired (SYSTEM-GAPS line 84).
- data/.git registry audit: 4,288 clients on disk, 4,290 in PG (+2 delta — later corrected to +3). Schema migration never executed — 4,288/4,288 are legacy shape, not the aspirational meta.schema. **Critical:** GitHub mirror was 47 commits ahead AND 10 commits behind origin (push rejected) — offsite mirror broken since today.
- 10/11 stale LaunchAgent labels booted out — ~600k/day spam log lines stopped. `com.tahoe.maintenance-update` preserved (active plist + exit 127 → needs script-path fix, not bootout).
- MCP tools E2E: all 5 new tools live-callable. tahoe_status_aggregate correctly returned the corrected Knowlton draft ID — STATE.md fix propagated.
- pytest: 12/13 passing, not 12/12 as STATE.md claimed. Failing test is test_duplication_gate (Jaccard 0.831 > 0.4 threshold).
- Cloudflare audit: trtahoe-pages IS UNSTUCK — successful deploy 2026-05-29T14:13:38Z (commit 07d1eae6). 16-day stall ended. SYSTEM-GAPS entry CLOSED.

### Wave 4c — Git repair

- data/.git rebased 47 commits cleanly (1 already-upstream auto-dropped). Auto-watcher pushed during the loop. Offsite mirror restored to ahead=0, behind=0.

### Wave 5 — Doc cleanup + write_log diagnosis + audit-gate enhancement

- trtahoe-pages "stuck" gap marked CLOSED in SYSTEM-GAPS. STATE.md pytest count corrected to 12/13. Memory `project-cloudflare-pages-20k-blocker` updated.
- write_log.jsonl diagnosed as opt-in by design (only 2 scripts populate it). MCP tool docstring is misleading. Logged as a HIGH gap for rename-vs-promote decision.
- audit-verify-gate.py enhanced: new `launchd-bootout` category + tightened path-token detection (bare tokens like "bak" no longer falsely flagged as paths). 3/3 tests pass.

### Wave 6 — More builds + audits

- Backup infrastructure E2E: voice + postgres healthy. Memory + session notes Drive rsync BROKEN since 07:13 (rsync metadata flags incompatible with Drive sandbox). FIXED — replaced `-a` with `-rlt --no-perms --no-owner --no-group --omit-dir-times --modify-window=1`. Drive byte-parity restored.
- BB ingest + classifier + send-approved verified: all 3 in shadow mode (Disabled=true). Identity hash matches Patch 2 spec. Two-way text approval lives in `parse-tristan-approval.py` (resume audit grepped wrong file). External-send safety STRONG (hardcoded TRISTAN_PHONE + whitelist + env flag + 3-plist enable sequence).
- Slug map: matches claim exactly (22,608 / 22,085 / 22,082 / 523 + 526 collisions, 0 orphans).
- path_lock race test PASS. Resume-corrected audit had its own inaccuracy: claimed path_lock not in generate.py, but it IS (line 32 import + line 205 use).
- deploy-staged-pages.py patched for buyer_rep tab rendering (~22 lines, 5/5 smoke tests pass). Full deploy gated on TahoeStaging volume remount.
- TahoeStaging volume not-mounted blocker documented in SYSTEM-GAPS (consolidates 4 affected services).
- STATE.md "Tonight's Autonomous Build Checkpoint" section inserted at line 65.
- pre-push hook verified: synthetic fake-AWS-key test BLOCKED correctly. validate.py does BOTH schema validation AND secret scan (7 regex patterns).
- DB +3 delta vs disk (was +2 at audit time; +1 since) — 3 phone-only auto-inserts from inbound SMS (IDs 6595, 6596, 6597). Recommend ignore + first-name enrichment via heuristic.

### Wave 7 — Final polish + self-audit

- ctx_warn_sentinel.sh wired into ctx-monitor.sh line 38. Sentinel changed from append (>>) to truncate (>) to prevent unbounded growth. chmod +x applied (the silent permission-denied failure had been swallowed by `|| true`). Verified end-to-end via 90% simulation: ctx-monitor PID 90787 picked it up, CLEAR_NEEDED updated. SYSTEM-GAPS line 84 CLOSED.
- Phone-stub enrichment proposal: 3 contacts identified — Boris (HIGH, Carnelian Bay roof cleanup), Sue Pearlstein (HIGH, Statewide Funding), Dale (MEDIUM, personal re: Tristan's son's ROTC at University of Utah). Doc + gated SQL UPDATE at `~/tahoe-os/data/system/phone-stub-enrichment-proposal-2026-05-29.md`. ID 6597 likely miscategorized as lead instead of personal.
- Self-audit by independent Opus: 4/4 PASS (STATE.md insertion at line 65, memory rsync holding through 18:46 cycle, deploy-staged-pages 5/5 random JSON renders all pass, audit-gate cold-test green).
- End-of-night regression check: ALL GREEN (the 2 soft flags resolved as false positives — grep pattern mismatch + wrong line-count expectation).

---

## Section 5b — Parallel session this evening (PM proposal + phone fix)

A second autonomous conductor session ran in parallel to Waves 1-7 above and landed two clean wins. Documented in full at `~/tahoe-os/projects/kent-edwards/HANDOFF-2026-05-29-EVENING.md`. No external sends, no destructive ops.

### 5b.1 — Phone number corrected in PM portal generator + approval

Stale `(530) 545-6486` references replaced with the canonical `530-448-1734` across two files:

- `~/tahoe-os/scripts/pm/pm_project_page_generator.py`
  - Line 25: `TRISTAN_PHONE` constant
  - Line 423: `tel:+15305456486` → `tel:+15304481734`
- `~/tahoe-os/scripts/pm/pm_approval.py`
  - Line 30: `TRISTAN_PHONE` constant
  - Line 31: `TRISTAN_PHONE_TEL` constant

**Authority:** memory `project_state2_template.md` explicitly says "Phone: 530-448-1734 (NOT 530.580.8001)". The `545-6486` was a Round-3 holdover.

A broader-tree sweep was NOT done this session (flagged as advisory question, §8 new-tonight Q1).

### 5b.2 — PM proposal watcher infinite-loop fixed

**Symptom:** `~/tahoe-os/logs/pmproposal-watcher.log` showed the watcher polling thread `19e762f44fe33789` every 60s for hours and skipping it each time. Wasted Gmail API calls + log noise.

**Root cause:** Gmail's `subject:PMPROPOSAL:` operator tokenizes on word boundary, so a subject like `pmproposal email test — 2026-05-29` matched the search query. The defensive check at `pmproposal_email_watcher.py:325` (`"PMPROPOSAL:" not in subject.upper()`) correctly rejected it (no literal `PMPROPOSAL:` substring with colon) but only logged and returned — never labeled the thread, so it stayed in the candidate pool forever (60s polling × hours = wasted calls).

**Fix 1 (one-shot):** Labeled the offending thread `19e762f44fe33789` with `pmproposal-error` (Label_41) via gws-as-tristan, removing it from the query (`POLL_QUERY` excludes `-label:pmproposal-error`).

**Fix 2 (durable):** Hardened the watcher at lines 324-331 so any future false-match auto-labels itself:
```python
if "PMPROPOSAL:" not in subject.upper():
    logger.info("skip thread %s — subject lacks PMPROPOSAL: (%s)", thread_id, subject)
    try:
        modify_thread_labels(thread_id, add=[label_ids[LABEL_ERROR]])
    except Exception as e:
        logger.warning("could not label false-match thread %s: %s", thread_id, e)
    return
```
Python syntax verified via `ast.parse`. LaunchAgent re-execs each minute, so new code went live without reload. **Confirmation log line at 17:12:38:** `INFO no unprocessed PMPROPOSAL threads`.

**Side effect:** false matches now land in `pmproposal-error` alongside real failures. Advisory question Q3 (new tonight): distinct `pmproposal-skipped` label?

---

## Section 6 — Current state, in detail

**Litigation (Kent Edwards):**
- Nevada County hearing **June 12 09:00** (14 days). Knowlton retainer Gmail draft `r-4931712662035923460` ready to send. Tristan to authorize.
- Placer County hearing **July 31 08:30** (63 days). V9 Supplemental Petition near-final. Open gaps: Residue Gap % calculation, HCD Serial #, Laura's last name, Aubrey Morrow Apr 24 executed PDF (lives behind DocuSign portal — manual download needed; no DocuSign API integration on system).
- **Adverse counsel:** R. Clay Stockton (filed PA petition in Nevada County).
- **Attorney referral targets** (per Kent project CLAUDE.md): John S. Knowlton (primary, Auburn) and **Brigit Barnes** (alternate).
- **Carroll MC-050 substitution:** Gmail draft `r-164251446931580981` to Carroll at Porter Simon — STILL UNSENT (intentional). Before sending, delete the `INSTRUCTIONS BLOCK` at the top of the body. If Carroll won't sign, pivot to CCP §284 motion.
- **Charity coalition (21 beneficiaries):**
  - Pot 1 (~$[REDACTED] each, 3): Bear Yuba Land Trust, CA State Parks Foundation, Save the Redwoods League
  - Pot 2 (~$[REDACTED] each, 18): see Tristan's master roster
  - Engaged: WWP (Parker Scott), NWF (Lauren Fields — offered to appear), Save the Redwoods (Rebekah Edwards — ACK letter), WWF (ACK letter)
  - **11 of 21 still zero-outreach as of 5/27.**
- **Telephonic appearance deadline for June 12 Nevada hearing: June 10** (form RA-010, 2 court days before).
- **Knowlton April 9 items still open:** conflict check (full beneficiary roster in 02-MASTER-TRUTH; adverse counsel = R. Clay Stockton), Porter Simon/Carroll status (MC-050 in process; CCP §284 pivot if Carroll won't sign).

**PM Proposal System:**
- **CLI:** `~/tahoe-os/scripts/pm/pmproposal`
- **Email watcher:** `~/tahoe-os/scripts/pm/pmproposal_email_watcher.py`
- **Template:** `~/tahoe-os/scripts/pm/proposal_template.html` (mountain-luxury navy/gold, mobile-first)
- **Gmail labels (Tristan's account):** `pmproposal-done` (Label_40), `pmproposal-error` (Label_41), `pmproposal-pending` (Label_42)
- **Live example:** Farrell proposal at https://trtahoe-pm-portal.pages.dev/proposals/farrell-4770-san-souci/
- **Blocked on Tristan:** Gmail forwarding filter for the agent account must be set up in Gmail web UI. 5-min Tristan job that unblocks "email anywhere → PM proposal URL in 60s" pipeline.

**Infrastructure (verified healthy as of end of session):**
- Postgres 17 + pgvector — 22,920 properties_master, 4,290 contacts, 14,937 sale_history, 524 memories.
- data-dir-watcher (PID 32142), com.tahoe.git-realtime, com.tahoe.data-watcher, com.tahoe.ctx-monitor (PID 90787), MCP server PID 50946, com.tahoe.digest-email (7am/12pm/5pm PT), com.tahoe-os.uptime-ping, com.tahoe.pmproposal-watcher.

**Cloudflare layer (verified healthy):**
- Zone trtahoe.com rules active: Response Header Transform Rule `ec1df83aab58405a942ed2643c0fa4e4` + Cache Rule `89c89edea09544f4aa70efc582c0bde6`.
- trtahoe-properties-worker — route trtahoe.com/properties/*, latest deploy 2026-05-29T23:20:06Z (version 9c1bb472).
- trtahoe-pages — STUCK GAP CLOSED, last successful deploy 14:13 today (commit 07d1eae6).
- cblaketahoe Pages — last deploy 2026-05-25.
- R2 `trtahoe-properties` — 3,772 objects (~17% of target ~22,920).

**Stage 1 pipelines (all Disabled=true in shadow):**
- BlueBubbles ingest, classifier, send-approved drafts daemon.
- Voice memo transcription (whisper.cpp 1.8.4).
- Jarvis morning brief generator.
- Tahoe OS dashboard (port 5777, FastAPI + uvicorn).

**Backups:**
- Voice mp3s → local + Drive: byte-identical (5/5).
- Memory + session notes → local + Drive: byte-identical after tonight's rsync flag fix.
- Postgres → Drive: today's dump (26 MB gzip, integrity OK).

**Known open gaps (in SYSTEM-GAPS.md, 205 lines):**
- /Volumes/TahoeStaging unmounted (blocks discord-bot, command-app, maintenance-update, cblake deploy)
- R2/dist desynced (partial 3-city dist, can't reconcile without full regen)
- tool_gate content-denylist bug (matches command body, not just argv)
- meta.schema describes aspirational shape; 4,288/4,288 clients are legacy
- write_log.jsonl partial coverage
- bos.trtahoe.com + chat.trtahoe.com tunnels publicly resolvable, origins down
- No analytics on either domain (CF Web Analytics not installed)
- No external uptime monitoring (UptimeRobot not set up)
- Postgres `homemini` user is `trust` auth (supply-chain risk via npm/pip)
- Time Machine not configured
- GitHub PAT has too-broad `repo` scope
- 1 failing pytest: test_duplication_gate (Jaccard 0.831 > 0.4)
- Mac Mini memory pressure: 176MB → 72MB drift during sustained autonomous work

---

## Section 7 — Open decisions awaiting Tristan

1. **Knowlton retainer authorization.** Draft `r-4931712662035923460`. 14 days to hearing. The only real-world ticking clock right now.
2. **Decide June 12 Nevada hearing posture.** Pro per / telephonic via RA-010 (file by June 10) / Knowlton appearance.
3. **Set up Gmail forwarding filter for the agent account.** 5-min Tristan job.
4. **Verify Carroll AOR status** by pulling Placer S-PR-0013003 docket.
5. **11-charity zero-outreach roster.** Tristan to approve text-by-text.
6. **TahoeStaging volume remount vs migrate vs retire.**
7. **R2 full reconcile sequence.**
8. **Phone-stub enrichment SQL UPDATE.**
9. **Dashboard enable.**
10. **meta.schema migration vs deprecation.**
11. **tool_gate content-denylist bug fix.**
12. **External credentials currently blocking:** GitHub Pro, CF Workers AI write token, BLUEBUBBLES_PASSWORD, Gemini 2.5 Pro API key.

---

## Section 8 — Open questions for the advisory board

### Carried from prior handoff

- Disconnect Gmail from claude.ai integrations? (gws-as-tristan replaces it for Gmail; keep Drive/Calendar MCP.)
- Add the Doc-style freeform subhead renderer to pmproposal?
- Verify Carroll current AOR status by pulling the Placer docket?
- Update the stale Drive docs (LIVING-FACTS, HANDOFF-CURRENT, PLAN-OF-ATTACK — all 40+ days old)?

### New from parallel HANDOFF tonight

- **NEW-1:** Broader phone-number sweep across the repo for any other stale references?
- **NEW-2:** Fix the em-dash mojibake in the watcher's subject decoder?
- **NEW-3:** Want a distinct `pmproposal-skipped` label vs `pmproposal-error`?

### New tonight from this conductor's session

**Q1 — Content duplication strategy on trtahoe.** Jaccard 0.83 → 0.765 same-ZIP via variant rotation. Target 0.40; realistic ceiling without restructuring ~0.65. Best practice for distinguishing 22,920 procedurally-generated property pages?

**Q2 — Cblake content surfacing architecture.** 4,975 patched JSONs sit in staging with buyer_rep tabs. Three architectural paths considered (WP REST loop rejected; CF Worker-rendered tab; static dist extension via deploy-staged-pages.py — PATCH APPLIED, 5/5 tests). Is there a more robust pattern we're missing?

**Q3 — R2 / Pages split for trtahoe.** Pages got unstuck today by moving property pages to Worker + R2. Now R2 is ~17% filled. Finish the R2 fill / move EVERYTHING to Worker + R2 / hybrid with explicit ownership rules?

**Q4 — Conductor architecture pain.** The router-only mode is structurally enforced via a Bash hook (`tool_gate.sh`) that has a content-vs-argv pattern-matching bug. What's the right shape for this gate?

**Q5 — Audit-verify-gate adoption pattern.** Twice in the last 24 hours it prevented disasters. How do we make the gate non-bypassable rather than instruction-enforced?

**Q6 — Memory architecture.** ~14 long-lived memory files. Some drift in hours, some are stable for months. Is there a tagging scheme that signals re-probe-before-trust vs durable-fact?

**Q7 — Cost-of-being-wrong asymmetry.** Sonnet build → Opus audit catches near-disasters but is expensive. For lower-stakes work, what's a defensible cheaper pattern that preserves catch-rate?

**Q8 — Stage 3 readiness.** Suggested enable sequence: Jarvis morning brief → Dashboard → Voice transcription → Inbound classifier → BB ingest non-shadow → Send-approved drafts daemon.

**Q9 — Where Tahoe OS should live in 12 months.** Same Mac / Hetzner mirror / split (Postgres stays on Mac, background work to cloud) / lean into "Mac is the prod"?

---

## Section 9 — Appendix: file paths, IDs, current PIDs

**Existing Drive documents:**
- Tahoe OS Briefing v1: https://docs.google.com/document/d/1OF7n2I9sXzj1YiOyC8zLIdtds2KTidBFd947XXv2t9c/edit
- Architectural Audit + 45K Scaling Roadmap: https://docs.google.com/document/d/1HIhYEmk9y3qjJSbv4L3uXg2Y15xqnoE2VRfRoAukC24/edit
- Security / Backups / Analytics Audit: https://docs.google.com/document/d/1JnKkIRsUS2XNNnjwsB8XJ_k8jIYYf2STQXnVLEB2k2k/edit
- Cloudflare Architecture: https://docs.google.com/document/d/154VyEJweEYsXod7lqe0PUGWGsf0M0YaRx4kUE9DXpwg/edit
- Kent Edwards Drive folder (legacy): https://drive.google.com/drive/folders/1aHQ2AwMiC3loaOAxtwlLYScw21o4r4bC
- Kent Edwards Drive working folder: https://drive.google.com/drive/folders/1gxqeHVbI_M3KgDc-7rAGjofbjkL2JxuA
- Knowlton packet (refreshed 5/29): https://drive.google.com/drive/folders/13obFO5BOo0kA3DNXIJilvnGr72NHfLSV

**Current PIDs (verified end of session):**
- com.tahoe.git-realtime: PID 31584
- com.tahoe.data-watcher: PID 32142
- com.tahoe.ctx-monitor: PID 90787
- MCP server: PID 50946

**Litigation contacts (Kent Edwards):**
- John Knowlton (retainer pending): public attorney listing, Auburn CA — KnowltonLegal.com
- Brigit Barnes — alternate referral target
- R. Clay Stockton — adverse counsel (filed Nevada PA petition)
- Carroll (Porter Simon) — current Placer AOR, MC-050 substitution pending
- Tristan: 530-448-1734 (DRE-public)

**LLM rate-limit consumption (2026-05-29 evening):**
- All-models weekly bucket: 24%
- Sonnet-only weekly bucket: 3%
- Reset: Thursday 11:59 AM PT

---

**End of v3 briefing.**

This document was synthesized by the conductor (Opus 4.7) after running ~40 subagents across 7 waves of autonomous build work on 2026-05-29 evening, then integrating a parallel-session HANDOFF that landed PM-proposal watcher hardening and a phone-number correction. It supersedes the v1 and v2 briefings on every point where they conflict. Trust live state probes over any briefing, including this one.

# Build Approach — Lessons from 2026-05-29

Captured at end of a 40+ subagent autonomous session covering Waves 1-7 + a late-evening pivot triggered by operator red-team feedback. These lessons replace nothing in the existing memory system; they're meant to inform the design of the next session's defaults.

## What worked (preserve these patterns)

### 1. Sonnet builds + Opus audits as independent pairs

Patch 1 of Fix-pair C shipped clean from Sonnet, but Opus audit found two real defects (delimiter spoofing + false-positive whitespace bug). One model writing + the same model verifying would have missed both. The pair has caught real bugs every time it ran. Cost: ~2x model spend per shipped patch. Benefit: catches the kind of bugs that would have caused production incidents.

### 2. Audit-verify-gate as a hard rule, not a suggestion

Twice in 24 hours it prevented disasters that would have destroyed thousands of live indexable URLs (cblake sitemap fabricated-slug fix in the morning; R2 reconciliation against a partial 3-city dist this evening). Empirical probe before any destructive action is non-negotiable.

### 3. Live-state verification over memory

When a subagent reported "Fix-pair D not shipped, zero grep hits," a second probe with broader scope found all three patches at correct paths in `tools/files.py`. Trusting the first probe would have rebuilt already-shipped work. Rule: **declaring absence requires comprehensive search before reporting.**

### 4. Memory as shared cross-session context

`project-tool-gate-content-bug` saved every subagent from re-discovering the heredoc workaround. `project-cblake-sitemap-healthy` was a load-bearing reference for the R2 halt decision. Pattern: when a hard-won finding has future value, write it down once, link it everywhere.

### 5. Append-only patterns on living ledgers

SYSTEM-GAPS and STATE.md were extended ~10 times tonight without losing prior content because every agent appended. Rewrite cycles in long-running ledgers are where data gets lost.

## What failed (eliminate these patterns)

### 1. Resume brief inflation went unchallenged

The morning resume brief listed 8 audit files (2 didn't exist), claimed Fix-pair C blocked (was partially shipped), gave wrong Knowlton draft ID, and overstated R2 fill state. We trusted upstream too long. **Lesson:** when the work product is an audit of prior work, the first wave should be independent ground-truthing of the brief's claims, not execution against them.

### 2. False-ready signal got amplified

A .md file was passed forward as "V9 petition ready for Knowlton." Operator correction: that's a draft, not filing-ready. The system passed it through without applying a court-readiness check (filing format, signatures, exhibits, statute citations, executed verification). **Lesson:** any "ready" claim requires checking against the actual acceptance criteria of the recipient (court, buyer, attorney, client). Shipped != ready.

### 3. Brief sections didn't cross-reference each other

The Jarvis morning brief sent at 17:19 PT recommended action on Oak Street in §1 + §10 while §3 in the SAME email showed the buyer withdrew. No internal consistency check. **Lesson:** any operator-facing emit must run a consistency pass against its own inputs before send.

### 4. Wrong primary frame on operator surfaces

The brief was organized by data source (calendar, inbound, drafts, SYSTEM-GAPS) instead of operator priority (money first → consequences second → questions for operator third). Result: deposits-received-needing-finish and recent-active-conversations were absent from "active work nodes." A brief that doesn't lead with money is invisible to the operator. **Lesson:** operator-facing surfaces inherit the operator's priority hierarchy, not the data layer's organization.

### 5. Output quantity was tracked; output quality wasn't

40+ subagents dispatched. Operator's response: "we need better results and a higher bar for the work that's produced." **Lesson:** per-shipment quality probes belong inside the dispatch pattern, not after. A "did the thing actually do the thing" smoke test should be a precondition for closing any task.

### 6. Subagent declarations need explicit scope + confidence fields

"Fix-pair D not shipped" was a grep-scope artifact (only server.py was searched, not the full tools/ tree). "20/20 buyer-rep JSONs structurally perfect" was correct but live-URL coverage was 0/20 due to a different cause. Free-form "looks good" reports hide their own bounds. **Lesson:** future subagent prompts should require declared scope, declared confidence, and declared invalidators.

## Forward-looking design principles

1. **Money-first framing on every operator surface.** Dashboards, briefs, digests — all must answer "what makes/finishes money today" before any other question. Deposits-in → finish-to-collect, then proposal pipeline, then everything else.

2. **Multi-source contact resolution before classification (the Dale principle).** PG + Gmail + Drive + iPhone-call-log + voice-notes — merge first, classify second. Single-source contact_type assignments are the failure mode.

3. **Cross-section consistency check before any operator brief emits.** If §1 recommends action and §3 shows the action is moot, the brief refuses to send until reconciled.

4. **False-ready detection.** Any "ready for X" claim requires evidence that X's acceptance criteria are met. For court: filing format + signatures + exhibits + verifiable citations. For client: scope agreed + amount agreed + timeline agreed. For attorney: actually-filing-ready PDF + executed declaration.

5. **Question format = simple explanation + bigger picture + suggested answer.** Any time the system asks the operator something, the ask carries the recommendation alongside it.

6. **Public LLM-board review as a feedback layer.** Sanitized briefing → public GitHub → board reviews → feedback comes back as design pressure. Today's session created the v1 of this loop.

7. **iPhone-call integration as the unlock for informed follow-ups.** One iPhone toggle stands between us and "you called X at 16:42 — any update?" Worth flipping. CallHistory.storedata is readable; only blocker is iPhone's "Calls on Other Devices" being off.

8. **Track output quality, not just output count.** "It shipped" is not enough. A simple per-emit quality score (does this clear the bar?) belongs in the dispatch loop.

## Concrete next-session adjustments

- **Default dispatch pattern:** before any task is marked complete, the dispatching agent runs a 1-line quality probe specific to the task type (sample render, statute spot-check, headers spot-check, live-URL probe, etc.). No quality probe → no completion.
- **Resume briefs from prior sessions:** treat as hypotheses, not facts. First wave = ground-truth pass. No execution before verification.
- **Operator briefs:** the v2 Jarvis structure (Money → High-Consequence → Questions-for-Operator → Inbound → Calendar → Stale-watch → System-footer) becomes the default. Builders extend it; they don't restructure it.
- **Any "ready for X" file:** carries a `readiness_check.json` sidecar with the acceptance criteria for X and which boxes are checked. Brief won't surface it as "ready" unless all boxes pass.

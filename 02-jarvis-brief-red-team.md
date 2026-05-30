# Jarvis Morning Brief — Red Team Audit (2026-05-29)

**Source:** Email sent by Tahoe OS at 17:19 PT to Tristan's primary mailbox
**Subject:** Jarvis Morning Brief - 2026-05-29
**Length:** ~6680 bytes (text/plain)
**Auditor framing:** Tristan's stated needs (verbatim, evening 2026-05-29):
- Money-making opportunities should be top of mind (current commitments first — deposits received and needing finish-for-payment; then next-proposal pipeline)
- High-consequence to-dos (Kent litigation, Knowlton, Carroll substitution, V9 polished petition)
- When the brief asks questions, give simple explanation + bigger picture insight + suggested answer
- Higher bar on work produced

## Current brief structure (as-is)

§1. "What you need to know in 30 seconds" — 2 items: oak-street offer review + Knowlton retainer auth
§2. "Calendar today" — TODO stub, no data (gws-as-tristan calendar not wired)
§3. "Inbound overnight" — 12 iMessages dump + 3 auto-classified drafts + raw email count
§4. "Status of active work nodes" — 2 nodes (oak-street, kent-edwards)
§5. "Litigation watch" — Kent only: June 12 + July 31 + V9 blockers
§6. "System health snapshot" — 13 failing LaunchAgents
§7. "Drafts awaiting your action" — 10 drafts, ALL "[Redacted - legal]"
§8. "Yesterday's deltas" — raw git commit log
§9. "Open SYSTEM-GAPS (top 3)"
§10. "Recommended first action" — single line

## Top 3 things wrong

1. **Self-contradicts on Oak Street.** §1 + §10 recommend "review buyer offer / counter terms" on Oak Street. §3 in the SAME email shows the buyers (Carol & Gregg) withdrew. The brief recommends action on a dead deal because §1 and §3 don't cross-reference each other.

2. **Zero money frame.** Mike's deck (deposit received, finish-for-payment), Julia's house (deposit received), Mary at Montclair ([REDACTED] scope), Carol on Tobaggan (deck), Nora's pending estimate (in §3!), Steve Farrell proposal attachment (in §3!) — none surface as money items. The brief leads with a dead listing and a legal authorization click instead of "what's owed to you and what could close this week."

3. **Redacted drafts in §7 are useless.** Every line reads "[Redacted - legal]". Tristan can't tell the polished V9-to-Knowlton from a Sierra Club planned-giving form letter. Also: the §1 "Knowlton retainer email ready" almost certainly IS the .md file Tristan flagged as "garbage" — false-ready signal that erodes trust in the whole brief.

## Per-section scoring

§1 30-sec: useful=partial, money=N (Oak Street is a listing not a deposit-funded job; Mike/Julia missing), insight=facts-only, framing=N, errors=contradicts §3 on Oak Street.
§2 Calendar: useful=N, money=not-its-job, insight=neither, errors=admits gap inline (should silently degrade).
§3 Inbound: useful=partial (Nora explicitly asking for estimate = money signal, buried), insight=facts-only, framing=N, errors=Steve Farrell proposal attachment treated as "Unknown Lead"; recipe text given 95% personal classification (fine) but burns top-of-list real estate.
§4 Work nodes: useful=N, money=N (only 2 nodes — Mike/Julia/Mary/Carol absent), framing=N (no deposits-received → finish-for-payment column), errors=node universe incomplete.
§5 Litigation: useful=Y, insight=partial (blockers listed without unblock suggestions), errors=claims "V9 petition cleanup substantially complete" contradicting Tristan's reality that the current .md is garbage; Carroll substitution status missing entirely.
§6 System health: useful=N for workday, money=N, errors=13 exit codes with no triage — noise.
§7 Drafts: useful=N (all "[Redacted - legal]"), errors=redaction defeats purpose.
§8 Deltas: useful=N, errors=raw auto-commit log is pure noise (15× "auto-commit <ts>").
§9 SYSTEM-GAPS: useful=partial (for system work not client work).
§10 Recommended action: useful=N (recommends Oak Street which §3 says is dead).

Cosmetic: subject says "Morning Brief" but sent at 17:19 PT — naming mismatch erodes trust.

## Proposed v2 Jarvis brief structure

**§1. MONEY TODAY** — three sub-blocks in order:
  - **a. Deposits-in / finish-to-collect:** Mike's deck (deposit $[REDACTED], %-complete, next-blocker, $[REDACTED] on completion). Julia's house (same shape). One row per active deposit-funded job.
  - **b. Proposal pipeline (next deposits):** Mary/Montclair ([REDACTED] cap, last contact, last reply, ASK). Carol/Tobaggan deck (status, ASK). Nora (estimate requested today — quote VERBATIM her message). Steve Farrell (proposal doc attached today — open it).
  - **c. Live offers/listings actually pending** — CROSS-CHECKED against today's inbound. If buyer withdrew, mark dead — don't recommend action.

**§2. HIGH-CONSEQUENCE TODAY** — Kent hearings countdown (June 12 in 14 days, June 10 telephonic deadline). Knowlton (last contact attempt: VM today 5/29 — from iPhone call-log integration). Carroll OFF substitution status. V9 petition: which file, last verified polished, what's missing. Each item: 1-line context + suggestion + 1-line "bigger picture."

**§3. WHAT JARVIS NEEDS FROM YOU** — questions formatted as: Question | Simple explanation | Bigger picture | Suggested answer Jarvis would pick. (Per Tristan's stated need.)

**§4. INBOUND THAT MATTERS** — ranked by money-or-consequence score, NOT chronology. Show preview, attachment names, proposed reply with rationale. Drop recipe-chat noise to a one-liner.

**§5. CALENDAR** — gracefully empty if unwired. Do NOT print a TODO stub.

**§6. STALE/ROT WATCH** — promises made N days ago not yet kept (Tristan flagged "Mary meeting result not logged" as exactly this category).

**§7. SYSTEM FOOTER (collapsed)** — LaunchAgent failures, gaps, git deltas — one line each. Link to detail. Never lead.

## How to know Tristan called Knowlton

Data source: iPhone call-log integration via `~/Library/Application Support/CallHistoryDB/CallHistory.storedata` (SQLite). Normalized output at `~/tahoe-os/data/phone/calls.ndjson` with rows `{ts, direction, number, contact_match, duration_sec, voicemail_left, transcript_path?}`. Brief queries last 24h, joins on contacts.phone → contact_id. Emits in §2: "Knowlton — called today 14:14 PT, no answer, VM left (12s). Last inbound from him: <date>. Suggested next: …"

Fallback until integration ships: `~/tahoe-os/data/phone/manual-log.md` — Tristan can dump a line, brief reads as fallback source.

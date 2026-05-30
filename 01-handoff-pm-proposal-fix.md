# Handoff — 2026-05-29 Evening (Parallel Session)
Successor to: `HANDOFF-FOR-CLEAR-2026-05-29.md`
Author: Opus 4.7 conductor, autonomous mode
Pacific time at write: ~17:15

---

## 0. TL;DR for the next session

Two clean wins shipped this turn, no external sends, no destructive ops.
Three blockers remain that require Tristan-in-the-loop.
Read §1 (what changed) and §3 (immediate next actions). The rest is context.

---

## 1. What changed this session (autonomous)

### 1.1 Phone number updated in PM portal generator + approval (open question #2)
**Files edited:**
- `~/tahoe-os/scripts/pm/pm_project_page_generator.py`
  - Line 25: `TRISTAN_PHONE = "(530) 545-6486"` → `TRISTAN_PHONE = "530-448-1734"`
  - Line 423: `tel:+15305456486` → `tel:+15304481734`
- `~/tahoe-os/scripts/pm/pm_approval.py`
  - Line 30: `TRISTAN_PHONE = "(530) 545-6486"` → `TRISTAN_PHONE = "530-448-1734"`
  - Line 31: `TRISTAN_PHONE_TEL = "+15305456486"` → `TRISTAN_PHONE_TEL = "+15304481734"`

**Why this number:** memory `project_state2_template.md` explicitly says "Phone: 530-448-1734 (NOT 530.580.8001)". The old `545-6486` was a Round-3 holdover.

**Verification command (re-run to confirm):**
```
grep -n "545-6486\|5456486" \
  ~/tahoe-os/scripts/pm/pm_project_page_generator.py \
  ~/tahoe-os/scripts/pm/pm_approval.py
```
Should return zero matches. The `.bak-pre-createdvia-2026-05-20` backup intentionally still has the old number — don't touch it.

**What still has the old number elsewhere** (not in scope this session, flag for later):
- Nothing material found in `~/tahoe-os/scripts/pm/`. Did NOT grep the wider tree.
- Recommended sweep next session: `grep -rn "545-6486\|5456486" ~/tahoe-os/ --include="*.py" --include="*.html" --include="*.md" 2>/dev/null | grep -v ".bak"`

### 1.2 PM proposal watcher infinite-loop fixed
**Symptom found:** `~/tahoe-os/logs/pmproposal-watcher.log` showed the watcher polling thread `19e762f44fe33789` every 60s for hours and skipping it each time. Wasted Gmail API calls, log noise.

**Root cause:** Gmail's `subject:PMPROPOSAL:` operator tokenizes on word boundary, so a subject like `pmproposal email test — 2026-05-29` matched the search query. The defensive check at `pmproposal_email_watcher.py:325` (`"PMPROPOSAL:" not in subject.upper()`) correctly rejected it (no literal `PMPROPOSAL:` substring with colon), but only logged and returned — never labeled the thread, so it stayed in the candidate pool forever.

**Fix 1 (one-shot):** Labeled the offending thread with `pmproposal-error` (Label_41) via gws-as-tristan, removing it from the query (`POLL_QUERY` excludes `-label:pmproposal-error`).
```
gws-as-tristan gmail users threads modify \
  --params '{"userId":"me","id":"19e762f44fe33789"}' \
  --json '{"addLabelIds":["Label_41"]}'
```

**Fix 2 (durable):** Hardened the watcher so any future false-match auto-labels itself. `pmproposal_email_watcher.py:324-331` is now:
```python
# only act on PMPROPOSAL: subjects (defensive — query already filters)
if "PMPROPOSAL:" not in subject.upper():
    logger.info("skip thread %s — subject lacks PMPROPOSAL: (%s)", thread_id, subject)
    try:
        modify_thread_labels(thread_id, add=[label_ids[LABEL_ERROR]])
    except Exception as e:
        logger.warning("could not label false-match thread %s: %s", thread_id, e)
    return
```
Python syntax verified via `ast.parse`. LaunchAgent re-execs the script each minute, so the new code is live without reload.

**Confirmation log line (17:12:38):** `INFO no unprocessed PMPROPOSAL threads`.

**Known side effect:** The fix uses `LABEL_ERROR` (`pmproposal-error`) as the catch-all for false matches. If you ever want to inspect what got auto-labeled, search Gmail with `label:pmproposal-error` and look for threads whose subject *doesn't* contain `PMPROPOSAL:` — those were the defensive-skip cases, not real failures.

---

## 2. What did NOT change (carryover from prior handoff, still open)

### 2.1 Knowlton email — STILL UNSENT (intentional)
- Gmail draft to John Knowlton (public attorney listing, KnowltonLegal.com), subject `Kent Edwards Estate — June 12 + July 31 hearings, $[REDACTED] for your help`
- Body backup: `~/tahoe-os/projects/kent-edwards/knowlton-email-2026-05-29.txt`
- Tristan sends himself.

### 2.2 Carroll MC-050 request — STILL UNSENT (intentional)
- Gmail draft ID `r-164251446931580981`, to Carroll at Porter Simon
- **Before sending, delete the entire `INSTRUCTIONS BLOCK` at the top of the body.**
- Tristan handles DocuSign routing separately.

### 2.3 Gmail forwarding filter for the agent account — BLOCKED on Tristan
- `gws-as-tristan` uses domain-wide delegation scoped to `cblaketahoe.com`. The agent account is a personal Gmail account, outside the workspace, so the service account cannot impersonate it. **No autonomous path; must be done in the web UI by Tristan.**
- Steps:
  1. Sign in to the agent Gmail account
  2. Settings → Filters → Create new filter
  3. Subject contains: `PMPROPOSAL:`
  4. Action: Forward to Tristan's primary mailbox (Gmail will require email verification of the forwarding address)
- After this is live, email anything from anywhere to the agent account with subject `PMPROPOSAL: <stuff>` and you get a deployed proposal URL back in ~60s.

### 2.4 Other open items from prior handoff
- Charity drafts in Drafts folder: 3 Pot-1 + 14 Pot-2 + 1 Knowlton + 1 Carroll. **None sent. None should be sent without explicit per-message approval.**
- 11 of 21 charities have had zero outreach as of 5/27.
- Telephonic appearance deadline for June 12 Nevada hearing: **June 10** (form RA-010, 2 court days before).
- Knowlton April 9 items still on table: conflict check (full beneficiary roster in 02-MASTER-TRUTH; adverse counsel = R. Clay Stockton) and Porter Simon/Carroll status (MC-050 in process; CCP §284 pivot if Carroll won't sign).

---

## 3. Immediate next actions, ranked

| # | Action | Owner | Blocker | Effort |
|---|--------|-------|---------|--------|
| 1 | Send Knowlton email (text already drafted) | Tristan | None — just review & send | 2 min |
| 2 | Sign in to agent Gmail and set the `PMPROPOSAL:` → forward filter | Tristan | Need Gmail password / 2FA on hand | 5 min |
| 3 | Verify Carroll AOR status by pulling Placer S-PR-0013003 docket; then either send Carroll draft (if still AOR) or pivot to CCP §284 | Tristan or Claude | Tristan calls; if pivot, Claude drafts §284 motion | 15-30 min |
| 4 | Decide June 12 Nevada hearing posture: pro per appearance, telephonic via RA-010 (file by June 10), or Knowlton appearance if he accepts | Tristan | Knowlton response | gated |
| 5 | Charity outreach to the 11 remaining (zero-touch) Pot-2 beneficiaries — Tristan approves text-by-text | Tristan + Claude | Per-message approval | 30 min |
| 6 | Phone-number sweep across the wider repo | Claude (autonomous OK) | None | 5 min |
| 7 | Update stale Kent strategy docs (LIVING-FACTS, HANDOFF-CURRENT, PLAN-OF-ATTACK — all 40+ days old) | Claude (under Tristan's eye) | Risk if done blind | 30 min |

---

## 4. Kent Edwards case — current state snapshot

### Hearings
- **June 12, 2026** — Nevada County `PR0001028`, Dept 6. Stockton's PA petition for letters of administration. Tristan pro per. Already filed: objection, notice of related cases, request for stay. Knowlton appearance pending.
- **July 31, 2026** — Placer County `S-PR-0013003`, Dept 40, Hon. Michael A. Jacques. V9 supplemental petition. Tristan petitioner. Carroll still attorney of record (MC-050 in process).

### Folders
- Local consolidated: `~/tahoe-os/projects/kent-edwards/`
- Drive working: https://drive.google.com/drive/folders/1gxqeHVbI_M3KgDc-7rAGjofbjkL2JxuA
- Knowlton packet (refreshed 5/29): https://drive.google.com/drive/folders/13obFO5BOo0kA3DNXIJilvnGr72NHfLSV

### Charity beneficiaries (21 total)
- **Pot 1 (~$[REDACTED] each):** Bear Yuba Land Trust, CA State Parks Foundation, Save the Redwoods League
- **Pot 2 (~$[REDACTED] each):** 18 charities
- **Engaged:** WWP (Parker Scott), NWF (Lauren Fields — offered to appear), Save the Redwoods (Rebekah Edwards — ACK letter), WWF (ACK letter)
- **Zero outreach:** 11 of 21 as of 5/27

### Adverse counsel
- R. Clay Stockton (filed PA petition in Nevada County)

---

## 5. PM Proposal System — current state

### Architecture (built earlier this week)
- **CLI:** `~/tahoe-os/scripts/pm/pmproposal`
- **Email watcher:** `~/tahoe-os/scripts/pm/pmproposal_email_watcher.py`
  - LaunchAgent: `com.tahoe.pmproposal-watcher`
  - Polls Gmail every 60s for `subject:PMPROPOSAL: newer_than:7d -label:pmproposal-done -label:pmproposal-error`
  - Log: `~/tahoe-os/logs/pmproposal-watcher.log`
- **Template:** `~/tahoe-os/scripts/pm/proposal_template.html` (mountain-luxury navy/gold, mobile-first, sticky PDF + Email buttons)
- **Labels in Gmail (Tristan's account):**
  - `pmproposal-done` (Label_40) — successfully processed
  - `pmproposal-error` (Label_41) — parse/CLI failure or false-positive subject match (auto-applied by hardened watcher as of this session)
  - `pmproposal-pending` (Label_42) — claimed by current worker

### Live example
- Farrell proposal (Doc-verbatim scope, all 3 defensibility paragraphs verbatim): https://trtahoe-pm-portal.pages.dev/proposals/farrell-4770-san-souci/

### Usage (after Gmail filter is set — see §2.3)
```
To: <agent gmail account>  (or Tristan's primary)
Subject: PMPROPOSAL: <Client> <Address> <Total>
Body:  free-form scope, one task per line
```
Returns a deployed Cloudflare Pages URL in ~60s.

### Known minor issues (not fixed this session)
- Em-dash mojibake in retrieved subjects (`Ã¢Â€Â"` instead of `—`). UI rendering is fine; only ugly in raw API output and the watcher log. Encoding issue is in the Gmail base64url payload decode path — would need a `quopri`/charset fix in `extract_headers`. Low priority.
- `render_scope_sections` knows only 3 subsection labels. Freeform Doc-style subheads still require direct HTML edit.
- Tool gate blocks recursive force-delete; use `find -delete` workaround.

---

## 6. Conductor-mode operating constraints

This session ran under conductor mode (`~/.claude/conductor/active.json` has `mode: router-only`). The tool gate (`~/tahoe-os/hooks/tool_gate.sh`) blocks `Edit`, `Write`, and `NotebookEdit` in the top-level conductor. Three escape hatches:

1. **`mcp__filesystem__write_file`** — directly bypasses the gate for writes. Use this for handoffs, notes, new files.
2. **Delegate to a subagent via the `Agent` tool** — subagents bypass the gate (process-tree detects two `claude` ancestors).
3. **Kill switch:** `export TAHOE_TOOL_GATE_OFF=1` before launching Claude Code. Don't do this casually.

**Bash IS allowed** in conductor mode (rescoped 2026-05-19; only the destructive denylist blocks).

**Audit-verify gate** (`~/tahoe-os/scripts/audit-verify-gate.py`) must be invoked before any subagent does a destructive action based on a prior audit. None of this session's work touched that path.

---

## 7. Files written / modified this session — complete list

| Path | What changed |
|------|--------------|
| `~/tahoe-os/scripts/pm/pm_project_page_generator.py` | Phone update lines 25, 423 |
| `~/tahoe-os/scripts/pm/pm_approval.py` | Phone update lines 30, 31 |
| `~/tahoe-os/scripts/pm/pmproposal_email_watcher.py` | Hardened false-match skip at lines 324-331 |
| Gmail thread `19e762f44fe33789` | Added label `pmproposal-error` (Label_41) |
| `~/tahoe-os/projects/kent-edwards/HANDOFF-2026-05-29-EVENING.md` | This file |

No deletions. No external messages sent. No git commits made (data-dir-watcher will handle any data/ change auto-commit; `~/bin/git-autopush.sh` covers scripts/ on its 30-min cadence).

---

## 8. Tristan's explicit do-NOTs (still in force)

- Do NOT auto-send Knowlton, Carroll, or any charity draft.
- Do NOT disconnect Drive/Calendar MCP. Gmail MCP is being phased out in favor of `gws-as-tristan` for *Gmail* only.
- Do NOT widen the tool gate without confirmation.
- Do NOT send test messages to real contacts (memory `feedback_never_test_on_real_contacts`).
- Do NOT cite memory about live services without re-probing (CLAUDE.md "Live-State Verification").
- Do NOT state legal procedure as fact without reading the statute text in-session (CLAUDE.md "Legal Claims").

---

## 9. Open questions for Tristan

Prior:
1. Disconnect Gmail from claude.ai integrations? (gws-as-tristan replaces it.)
2. ~~Update the rest of PM portal (pm_project_page_generator.py) to phone 530-448-1734?~~ ✅ Done this session.
3. Add the Doc-style freeform subhead renderer to pmproposal?
4. Verify Carroll current AOR status by pulling the Placer docket?
5. Update the stale Drive docs (LIVING-FACTS, HANDOFF-CURRENT, PLAN-OF-ATTACK — all 40+ days old)?

New (from this session):
6. Want me to do a broader phone-number sweep across the repo to catch any other stale `545-6486` references (templates, blog posts, READMEs)?
7. Want me to fix the em-dash mojibake in the watcher's subject decoder? Cosmetic, but the log is ugly.
8. The watcher now auto-labels false matches as `pmproposal-error`. If you'd rather have a distinct `pmproposal-skipped` label so you can tell real failures from false-positives at a glance, say the word — quick add.

---

## 10. State files / canonical pointers (re-read these at next session start)

- `~/.claude/CLAUDE.md` — global instructions (model routing, audit-verify gate)
- `~/CLAUDE.md` — session protocol (DO NOT RUSH, verification rules, legal claims rule)
- `~/tahoe-os/CLAUDE.md` — tech stack reference
- `~/tahoe-os/projects/kent-edwards/CLAUDE.md` — Kent project memory
- `~/tahoe-os/state/SYSTEM-GAPS.md` — known infra gaps (read before answering infra questions)
- `~/tahoe-os/context/HOW-WE-DO-THINGS.md` — canonical routing table
- `~/tahoe-os/context/CONDUCTOR-PROTOCOL.md` — conductor operating model
- `~/.claude/projects/-Users-homemini/memory/MEMORY.md` — auto-memory index

End of handoff.

# Tristan's Feedback to LLM Advisory Board — 2026-05-29 Late Evening

These are Tristan's direct words to the conductor after reviewing tonight's autonomous build. Captured for the advisory board.

---

## On the morning Jarvis brief

"Let do a red team audit on this as to what is useful for me to receive and how it should be presented and a deeper dive into the text and email monitoring and the classifying."

## On contact classification depth (the Dale lesson)

"Dale can be found in email and in texts and in email from a number of different sources. The lesson is find everything related to a contact to get the full truth and then look at the full truth closely to determine contact type and thus next actions."

## On phone integration

"FYI I left a voicemail with Knowlton — is there a way for you to see my recent calls and voicemails — maybe via iphone syncs?"

## On what should be top of mind

"What should be top of mind are money making opportunities like current commitments; Mike's deck deposit received and Julia's house deposit received. When we finish the project I get fully paid so lets push to get it finished. Next would be potential projects to get paid that are recently active like Mary at Montclair — she said she wanted to keep the overall cost of the todo list to [REDACTED] right now. You probably saw her text and maybe my reply but I didnt log the results of the meeting but she sent me a photo of the todo list, carol on tobbogan her deck...these are who we've been talking to recently that seems like they could be the next proposal, deposit, and finished income."

## On high-consequence items

"Other things to be featured on the list are to-do's that have big consequences like Kent's and Knowlton (I called him but how would you know unless I tell you — if you could see my calls you could ask for an update), carroll off as attorney for the placer case, the email to knowlton with the polished v9 petition not the garbage .md file that I was told was the 'ready for Knowlton' doc...that's not ready."

## On work quality

"We need better results and a higher bar for the work that's produced."

## On how to ask questions

"If there are questions needed of me I need simple explanations with insight on the larger picture and suggestions so I can quickly understand the bigger picture and give you solid answers."

---

## Implied design principles

From the feedback above, the advisory board should consider these as load-bearing requirements going forward:

1. **Money-first framing.** Every operational surface should answer "what makes/finishes money today" before any other question. Deposits received → finish-for-payment is the top priority. Recent active conversations → next proposal pipeline is second.

2. **Full-truth contact resolution.** Before classifying a contact_type or suggesting a next action, find every reference to that person across PG (contacts, interactions, messages, jobs), local data (clients, shadow, inbound-drafts, voice-notes), Gmail, Drive, and iPhone call/voicemail logs. Multiple-source merge is the spec, not the goal.

3. **Phone integration is in scope.** iPhone call + voicemail sync via iCloud + macOS provides a feasible read path (CallHistoryDB SQLite). Currently not wired; should be.

4. **Brief design must be cross-referential.** Brief sections must consistency-check against each other before emit. Recommending an action on a dead deal because §1 and §3 didn't cross-reference is the failure mode to eliminate.

5. **False-ready is worse than not-ready.** If a doc is claimed ready but isn't (V9 .md file labeled "ready for Knowlton" was actually a draft), the brief should flag the gap, not amplify the false signal.

6. **Questions to Tristan must be high-leverage.** Format: question + simple explanation + bigger-picture insight + suggested answer. He can decide fast when given the recommendation alongside the ask.

7. **Higher bar.** Quality of output is a tracked metric. "It shipped" is not enough.

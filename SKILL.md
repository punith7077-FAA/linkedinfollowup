---
name: linkedinfollowup
description: Follow up on LinkedIn DMs that got no reply, using the fixed follow-up template and schedule. Use when the user asks to run follow-ups, review or approve the follow-up queue, or check follow-up status.
---

# LinkedIn Follow-up

## Prerequisites

This skill drives the linkedin-scanner engine. Requires Node.js 22.5 or newer (the engine uses node:sqlite) and Git.

1. `git clone https://github.com/punith7077-FAA/linkedin-scanner` and `cd` into it.
2. `npm install`
3. `npx playwright install chromium`
4. `node cli.js setup`. Setup asks for the LinkedIn account name, the ICP, the DM body, and the follow-up message. Nothing runs until this is done.
5. `node cli.js login main` (and `node cli.js login research` if a second account is used). Log in by hand in the window that opens.
6. Copy this skill folder to `~/.claude/skills/linkedinfollowup/` so `/linkedinfollowup` is available in Claude Code.

All commands below run from the engine checkout as `node cli.js <command>`. This skill shares the browser session, database, and caps with the other LinkedIn skills; install /linkedindm too, since follow-ups only ever act on DMs it sent.

## Purpose
A DM sent by /linkedindm that gets no reply enters a follow-up sequence: up to `maxFollowups` (5) follow-ups, the first `followupAfterDays` (2) days after the DM and each later one `followupGapDays` (2) days after the previous follow-up. At most one message every two days, and the sequence ends after the fifth. Every step renders the fixed template in `templates/followup.txt`; if the user adds `templates/followup-N.txt` (N = 2..5), step N renders that file instead.

The template holds the follow-up message the user entered during `node cli.js setup`. That file is the single source of truth for the wording. Read it, do not quote it from memory. Send it EXACTLY as written with only `{firstName}` substituted: no company names, no extra sentences, no rephrasing, no "humanizing".

## Commands
- `node cli.js followup queue` lists people due their next follow-up (`due` or `approved`), showing which step is next (for example `follow-up 3/5`) and when the previous message went out.
- `node cli.js followup approve <ids|--all>` and `node cli.js followup skip <ids>` are the decision step. Approve only marks entries that are actually due.
- `node cli.js followup send`: for each approved entry, open the thread, read who has spoken, and only if the recipient has NEVER replied, paste and send the template for that step. Recipients who replied are marked `replied` and leave the sequence for good. Because later steps repeat the same wording, the send is verified by the message count in the thread going up, not by the text merely being present.
- `node cli.js followup status` prints counts by follow-up status and the last followup session.

## Hard rules, do not work around these
1. Same send caps as DMs: `maxDmsPerSession` (20) and `delayBetweenDmsMs` (3000) in `config.json`, never raised or lowered. At most one followup session per day, and follow-ups count toward the account's overall DM volume. Do not stack a full DM session and a full followup session on the same day.
2. Never send to entries that are not `approved`. With `autoApprove` true (the default) the agent approves due entries itself; otherwise show the due queue to the user and ask.
3. Never follow up on someone who replied. The send path reads the thread's sender sequence first; any sender other than the account owner (`accountName` in `config.json`, set by setup) counts as a reply, marks the entry `replied`, and hands the conversation back to the user. If the sender sequence cannot be read, the entry fails with `REPLY CHECK FAILED` and nothing is sent. Fix `checkReplied()` in `src/followup.js`; do not retry in a loop, and never assume no-reply.
4. At most `maxFollowups` (5) follow-ups per person, one per due cycle, never two on the same day. A `sent` row re-enters the due queue by itself once `followupGapDays` has passed and the count is below the max; `replied`, `skipped`, and `failed` rows never re-enter by themselves. Do not re-approve a failed entry until the underlying error is understood. `followup_count` on the queue row is the only record of how many have gone out; never reset it.
5. The message is the fixed template with only `{firstName}` substituted. Do not personalize, extend, or vary it without the user asking.
6. Timing comes from the database (`sent_at` + `followupAfterDays` for the first, `followup_sent_at` + `followupGapDays` for the rest), never from eyeballing the messages page. Do not follow up early, and do not change `maxFollowups`, `followupAfterDays` or `followupGapDays` without the user asking.
7. If `followup send` reports "composer not found" or "send not confirmed in DOM", LinkedIn's markup changed. Fix the selectors in `src/followup.js`; do not retry in a loop. Report results exactly as the CLI prints them.
8. Never open a profile without a workflow reason. Sanctioned opens here are only the thread-open of an approved due entry during `followup send`.
9. NEVER type or paste message text into LinkedIn yourself, not via Playwright, not via the composer, not to "fix" a failed send. The only way a message leaves this system is `node cli.js followup send` rendering the template file. Freehand agent messages are how off-template text and wrong-thread pastes happen. If the CLI fails, report the failure and stop.

## Relationship to the other skills
/linkedindm sends the first DM and stamps `sent_at`; this skill only ever acts on those rows. A `replied` mark means a live conversation. Tell the user so they can take over. Nothing here touches the invite pipeline.

## Run summary and goal (required)
Every run ends with a summary: follow-ups sent (with each person's step, as the CLI prints it), replied entries (warm leads, named prominently), failures as printed, entries still due, people who completed all five without replying, and progress against the /goal target in `goals.json` (follow-ups sent today vs target). Install the goal skill for that report.

## Typical flows
- "any follow-ups due?": `node cli.js followup queue`, summarize with the step each person is on and days since their last message.
- "send the follow-ups": `node cli.js followup send` (approved entries only), report the summary line and any `replied` names prominently. Those are warm leads.
- "how are follow-ups doing?": `node cli.js followup status`.

# linkedinfollowup

A Claude Code skill that follows up on LinkedIn DMs that got no reply. Each person gets up to five follow-ups, two days apart, using the follow-up message you wrote during setup. Before every send the engine reads the thread; anyone who has replied is marked as a warm lead and never messaged again by the sequence.

## Install

Requires Node.js 22.5 or newer (the engine uses node:sqlite) and Git.

1. `git clone https://github.com/punith7077-FAA/linkedin-scanner` and `cd` into it.
2. `npm install`
3. `npx playwright install chromium`
4. `node cli.js setup`. Setup asks for the LinkedIn account name, the ICP, the DM body, and the follow-up message. Nothing runs until this is done.
5. `node cli.js login main` (and `node cli.js login research` if a second account is used). Log in by hand in the window that opens.
6. Copy this skill folder to `~/.claude/skills/linkedinfollowup/` so `/linkedinfollowup` is available in Claude Code.

Also install `linkedindm` (this skill only acts on DMs it sent) and `goal` (run summaries report against it).

## Use

Type `/linkedinfollowup` in Claude Code from the engine checkout. The agent lists who is due, approves, sends through the CLI, and reports sent, replied, failed, and still-due counts against the daily goal. Replied names are the ones to act on yourself.

Heads-up: automated access and messaging violate LinkedIn's User Agreement. The caps and pacing reduce the footprint; they do not make it sanctioned. Use at your own risk.

## Next

Next, install humanizer, which the comment skill uses to make drafted comments read like a person wrote them.

https://github.com/punith7077-FAA/humanizer

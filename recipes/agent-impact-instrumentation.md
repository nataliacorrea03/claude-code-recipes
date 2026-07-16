# Agent impact instrumentation (measure what your automations save you)

A build recipe. Paste the prompt below into Claude Code and it builds a layer that measures how much time your fleet of Claude Code cloud routines actually saves you, then renders it into a dashboard you can share, **without ever inventing a number it cannot back up.**

## What it builds

If you run a handful of scheduled Claude Code routines (a drafting agent, a morning briefing, a weekly emailer, whatever), this gives you an honest answer to "are these things actually worth it." It builds:

- An **activity log** your high-volume agents write to as they work.
- A **baselines page** where you set how many minutes each kind of task takes a human, so the hours-saved math is yours to see and tune.
- A **weekly grader routine** that reads the log, checks your drafts against what you actually sent, computes hours saved, volume, and quality rates, and appends one row to a running dashboard.
- A **dashboard** plus an on-demand command that renders it into a clean, shareable page.

The whole thing is built around one rule: numbers it can prove (counted from logged events) are kept separate from numbers it estimates (credited to agents by detecting the output they already produce). It never blends the two into one impressive-but-fake figure.

## Requirements

- **Claude Code with cloud/scheduled routines.** This measures automations that run on a schedule (cron-style triggers). If you don't have any running yet, there's nothing to measure.
- **A Notion workspace** (or any database you can read and write via an MCP connector). The recipe uses Notion for the log, baselines, and dashboard.
- **Optional but recommended: Slack and Gmail connectors.** The grader cross-references drafts against what you actually sent (Gmail) and detects whether silent agents fired (Slack). Without them you can still measure the agents that self-log, you just lose the estimated half.
- **A sense of what your tasks are worth.** You'll fill in the baseline minutes. Start rough, tune later.

## How to use this recipe

1. Open Claude Code.
2. Have your Notion (and ideally Slack + Gmail) connectors authenticated.
3. Copy everything below the line and paste it as your prompt.
4. Answer what it asks: which routines you run, and roughly how long each task takes you by hand.
5. Let it run the grader once, manually, so you can confirm the dashboard row and the numbers look sane before trusting the weekly schedule.

The prompt is written in the first person so you can paste it as if you're the one asking.

---

## What I want you to build

An instrumentation layer that measures how much time my scheduled Claude Code routines save me, and renders it into a dashboard I can share. I care more about the numbers being honest than being big.

## Core design: two tiers of credit, kept separate

My routines fall into two kinds, and they get measured differently. Bake this split in, do not collapse it.

- **High-volume "worker" agents** (the ones that produce a variable number of outputs each run, like a drafting agent that makes anywhere from 0 to 40 drafts). These **write a row to the activity log for every action they take**. Their hours saved are **MEASURED**: counted from real logged events. You cannot estimate these because the volume swings, you have to count.
- **"Whole-fire" agents** (the ones that do roughly the same amount of work every time they run, like a daily briefing or a weekly email). These do **NOT** self-log. Instead the grader **detects whether they fired** by looking for the output they already produce (a Slack message, a Gmail draft, a Notion entry) and credits a fixed per-run estimate. Their hours are **ESTIMATED**.

Every report keeps MEASURED and ESTIMATED as two separate subtotals plus a combined figure. Never merge them into one number.

## Pieces to build

1. **Activity log (Notion database).** Columns: Agent (text), Timestamp (date), Event type (a Select with a fixed, clean taxonomy, e.g. `draft-created`, `abstained`, `todo-created`, `note-appended`, `error`, `run-summary`, plus whatever your workers do), and a few free-text fields for context (thread id, a short excerpt, a reason). Your worker agents append rows here as they run.

2. **Baselines page (Notion).** Two small editable tables: per-event minutes (how long each event type would take me by hand) and per-run minutes (how long each whole-fire routine would take me by hand). The grader reads these live. I want to edit these numbers without touching code.

3. **Weekly grader routine (a scheduled Claude Code trigger).** Once a week it:
   - Reads the last 7 days from the activity log.
   - For each drafted item, cross-references against what I actually did (was it sent verbatim, sent after edits, ignored, or deleted). This gives an acceptance rate.
   - Detects which whole-fire agents fired (see the detection section) and credits their per-run estimate.
   - Computes: hours saved (MEASURED + ESTIMATED, separate), volume, acceptance rate, and an error rate.
   - Appends one row to the dashboard's weekly-trend table and refreshes the headline totals.
   - Sends me a one-line summary message.

4. **Dashboard (Notion page).** A weekly-trend table (one row per week), a headline with the running cumulative (proven vs estimated, kept distinct), and a per-agent breakdown marking each agent MEASURED or ESTIMATED.

5. **On-demand render command (a skill).** A command I can run any time that reads the dashboard and renders it into a clean, self-contained shareable page.

## Detection: how the grader credits silent agents

For each whole-fire agent, detect that it fired by matching the **exact** signature of the artifact it already produces. Write these signatures down as precise strings, one per agent, for example:
- Daily briefing: a message that starts with the exact prefix it always uses.
- Weekly email: a draft created that day to a specific recipient.
- A learner agent: an entry it wrote to a specific page, tagged with today's date.

Count fires, credit the per-run baseline, and produce two lists: agents credited (with fires and hours), and agents you could NOT verify (listed as "unverified, not counted," never silently assumed).

## CRITICAL gotchas: bake these in, they were all learned the hard way

1. **Do NOT make every agent self-log.** It's tempting to add a "log yourself" line to all of them. Don't. Self-logging every routine means rewriting every routine, adds a new point of failure to each one, and risks double-counting against detection. Only the high-variable-volume workers self-log. Everything else is credited by detection. This is the single most important call in the whole build.

2. **Detection signatures must be EXACT strings, not vague descriptions.** "Look for the learner's message" is not a signature. "A message starting with `📚 FAQ:`" is. A vague signature silently fails to match, and then the dashboard reports a perfectly healthy agent as "down" or "0 hours." A false zero is worse than no number, because you trust it. Every signature is a literal string you have verified against real output.

3. **Messages to yourself do not reliably show up in search.** If an agent posts to your own DM / notes-to-self channel, keyword search often returns nothing. Read the channel directly instead of relying on search. (This one cost me a whole "why is this agent dead" investigation. It wasn't dead.)

4. **Never blend mature data with brand-new data.** When a new logging source first comes online it has a trickle of data. Do not add three hours of fresh live data to a full week of established tracking and call it a weekly total. Show immature data in its own "pulse check" box until it has a full clean period behind it.

5. **Scheduled-routine prompts are usually baked into the trigger, not read from a local file.** Editing the local copy of the prompt does nothing to the live routine. To change a routine you must update the trigger itself, and then re-fetch it and verify what actually got stored (automated edits to long prompts can corrupt bytes). Change one thing, verify the rest is byte-identical, keep a copy for rollback.

6. **Keep the baseline minutes in an editable page, never hardcoded.** The hours-saved figure is only as trustworthy as its assumptions, so the assumptions have to be visible and tunable by a non-coder. If someone asks "how did you get 40 hours," you open one page and show them.

7. **A self-log line runs LAST, so absence is not proof of idleness.** If a worker crashes early it logs nothing, so "no rows this week" can mean "didn't run" OR "ran and died before logging." And some agents intentionally post only when something changed. Detection must treat a quiet agent as neutral (credit its normal run, labeled "assumed"), not as a silent failure, and never zero-credit a whole week on silence alone.

8. **Label low-confidence rates.** Any rate computed off fewer than ~10 data points gets a "(low N)" tag. Never report a turnaround time you couldn't actually measure. State the honesty floor in the grader's own instructions so it holds itself to it.

## Deliverables

1. The activity log database, baselines page, and dashboard page, created and linked.
2. The weekly grader trigger, scheduled, with its full instructions including the exact detection signatures and the MEASURED-vs-ESTIMATED separation.
3. The on-demand render command.
4. A short README covering: the two-tier model, how to add a new agent to detection (write its exact signature), how to tune the baselines, and how to run the grader manually.
5. After building, run the grader once manually and show me the dashboard row plus the numbers, so I can confirm they're sane before trusting the schedule.

Don't claim it works until you've run the grader once end to end and shown me real output. Keep MEASURED and ESTIMATED separate in everything you produce. If a number is an estimate, say so next to the number.

# FITFO: a decision-forcing skill for Claude Code

A build recipe. Paste the prompt below into Claude Code and it installs **`/fitfo`** (Figure It The F Out): a skill for moments when you're stuck, can't decide, or the obvious path isn't good enough.

## What it builds

`/fitfo` doesn't hand you a menu. It forces a reframe of the real problem, runs four creative passes internally (first-principles, constraint inversion, cross-industry steal, capability stretch), runs a critic to find the strongest objection, then returns **one committed recommendation** plus fully-drafted versions of every artifact that recommendation needs (emails, docs, plans). It logs each run and schedules a 7-day check-in so you find out whether the call actually worked.

## Prerequisites (and what happens if you don't have them)

The skill reads optional context files from `~/.claude/profile/` (your business, your team, your voice, your tools, how you tend to get stuck). If you've done a Claude Code "founder onboarding" setup, those already exist.

**You don't need them.** The prompt below is written to degrade gracefully: if the files are missing, it runs a short interview to gather the same context, writes it down, and proceeds. More context just makes the recommendations sharper.

## How to use this recipe

1. Open Claude Code.
2. Copy everything below the line and paste it as your prompt.
3. Answer the short intake (it's mostly multiple choice).
4. From then on, type `/fitfo` followed by whatever you're stuck on, e.g. `/fitfo I can't decide whether to drop our cheapest tier.`

---

Build me a `/fitfo` skill. FITFO = Figure It The F Out. It's a decision-forcing skill for situations where the obvious path is blocked, missing, or not good enough. The skill forces a reframe, runs four creative passes internally, runs a critic subagent, then returns ONE committed recommendation plus fully-drafted artifacts. Never a menu.

Before writing anything, check for these optional context files:
- `~/.claude/CLAUDE.md`
- `~/.claude/profile/business.md`
- `~/.claude/profile/team.md` (skip if the user is solo)
- `~/.claude/profile/voice.md`
- `~/.claude/profile/tools.md`
- `~/.claude/profile/stuck-patterns.md`
- `~/.claude/profile/decisions.md`

If any are missing, do NOT stop. Run a quick inline interview to gather the equivalent context: what the business or project is, who's on the team (or whether they're solo), the user's voice and tone, what tools they have connected, how they tend to get stuck, and where their decision authority ends (what they can do reversibly vs. what they must never do without asking). Write what you learn into the corresponding profile files so future runs are sharper, then proceed. The skill works off whatever context exists.

Once that's settled, do a short interview to capture three fitfo-specific inputs. Multiple choice for everything except "why" follow-ups.

### FITFO-specific intake (~10 min)

**Q1, recent stuck moments.** Ask for two or three specific decisions where the user was genuinely stuck recently. For each:
- What was the decision? (free text, one sentence)
- What did you tell yourself you "couldn't" do? (multi-select: too expensive, no one to delegate to, would upset someone, don't have the data, partner/co-founder wouldn't agree, not enough time, didn't know how, other)
- What did you end up doing? (multi-select: picked the obvious safe option, kept putting it off, decided too fast, asked someone else to decide, found a creative way through, still haven't decided)

These become test cases. Save them.

**Q2, industries to steal from.** Multi-select 3-5 industries structurally similar to theirs that they'd accept a borrowed mechanism from: hospitality/restaurants, healthcare/clinics, law firms/professional services, software/tech startups, education/schools, manufacturing/logistics, retail/e-commerce, media/entertainment, real estate, sports/fitness, nonprofits, other (specify). This biases the cross-industry creative pass toward analogies they'll actually accept.

**Q3, critic harshness.** When the critic subagent reviews your recommendations, how hard should it push back?
- Gentle (point out real risks but don't kill ideas easily)
- Balanced (default: strong objections only if they're fatal)
- Brutal (assume the idea is wrong until proven otherwise)

### Write the profile extension

Append the answers from Q1, Q2, Q3 to a new file at `~/.claude/profile/fitfo-config.md` with sections: recent stuck moments (with the telling-themselves-couldn't and what-they-did notes), industries to steal from, and the critic harshness setting.

### Write the skill

Create `~/.claude/skills/fitfo/SKILL.md` with this structure:

```
---
name: fitfo
description: Figure It The F Out. Forces unconventional creative problem solving for business and strategic decisions where the obvious path is blocked, missing, or not good enough. Returns one committed recommendation plus drafts of every artifact the recommendation requires. Trigger on /fitfo, on phrases like "figure it out", "I'm stuck", "no good options", "we have to pick something", "I don't know what to do", "what do I do here", and on auto-detect: while inside a decision problem, if you catch yourself saying "I can't", "the only option is", "we'd need to wait for", "only you can decide", or returning 3+ options without a recommendation, invoke this skill instead. Scope: decision problems, not code-writing.
---

# /fitfo: Figure It The F Out

At the start of every invocation, load any of these that exist:
- ~/.claude/CLAUDE.md
- ~/.claude/profile/business.md
- ~/.claude/profile/team.md
- ~/.claude/profile/voice.md
- ~/.claude/profile/tools.md
- ~/.claude/profile/stuck-patterns.md
- ~/.claude/profile/fitfo-config.md

## When to invoke
Explicit triggers: /fitfo, or phrases like "figure it out", "I'm stuck", "no good options", "we have to pick something", "I don't know what to do".

Auto-detect (model invokes itself): while inside a decision problem, if you catch yourself about to say "I can't", "the only option is", "we'd need to wait for", "only you can decide", or about to return 3+ options without a recommendation, stop and invoke this skill.

Do NOT invoke for code-writing tasks or one-off lookups.

## Step 1: intake clarity check
A problem is sharp if it answers all four:
  1. What's the actual outcome you want? (the outcome, not the action)
  2. What specifically is blocked, missing, or not good enough?
  3. What's already been tried or considered?
  4. What's the binding constraint? (time, money, people, contractual, brand)
If 3 or 4 are answered, run. If 2 or fewer, ask the missing ones via AskUserQuestion as multiple choice. Do not solve a problem you don't understand.

## Step 2: reframe
Write the problem in one sentence using:
> "The real problem is X, not Y."
If the reframe matches the user's stated problem verbatim, you didn't reframe. Try again. The reframe is the highest-leverage step in this skill.

## Step 3: four creative passes (run all four, internally)
For each pass, write 2-4 candidate moves. Be willing to be wrong, weird, or expensive. Synthesis will kill the bad ones.

A. First-principles reset: strip every assumption baked into the problem. If you were starting today from scratch, what would you do?

B. Constraint inversion: what if the blocker were the feature? What if the missing thing didn't need to exist? What if the constraint were the point?

C. Cross-industry steal: pick 2-3 industries from fitfo-config.md's "industries to steal from". Borrow the mechanism. (How does hospitality handle a missing senior-shift problem? How do law firms handle a partner missing mid-case?)

D. Capability stretch: refuse the default toolset. Consider spawning subagents, building a script mid-flight, web research, calling external APIs via connected tools, scheduling a routine, drafting a new system instead of working inside the broken one, recruiting a teammate for one specific micro-task. This is the most important pass. Most "stuck" states are forgotten-capability errors. If you catch yourself thinking "I can't because [tool/permission/scope]", check the actual tools available first. The limit is usually wrong.

## Step 4: synthesis, pick one
From all candidates, pick ONE action. Filters in order:
  1. Does it solve the REFRAMED problem?
  2. Executable this week?
  3. Survives the cheapest objection?
  4. Meaningfully better than the obvious path, not just different?
If multiple survive, pick the highest leverage per hour of the user's time. Commit. Do not return a menu.

## Step 5: critic subagent
Spawn one fast/cheap critic subagent via the Agent tool. Prompt:
> "Find the single strongest fatal objection to this proposed action. Be specific and concrete. If there is no fatal objection, say 'ship it' and explain in one sentence why the next-best objection isn't fatal. Critic harshness: [pull from fitfo-config.md]. Under 200 words. Proposed action: [insert recommendation + 2-3 line reasoning]."
Read the response.
- "Ship it": proceed to Step 6.
- Fatal objection: revise the recommendation once to address it, then proceed. Do not spawn a second critic. Do not loop.
- Non-fatal but real: ship, but surface as a "Watch for" item.

## Step 6: output
Return exactly this structure:

**The real problem:** [reframe in one sentence]
**Recommended action:** [one committed move, one sentence]
**Why this beats the obvious path:** [2-3 lines max]
**Watch for:** [non-fatal objection from the critic, if any]
**Kill criteria:** [what signal in the next 7 days would tell you this isn't working]
**Artifacts (drafts):** [every email, doc, message, plan, or spec the recommended action requires, FULLY DRAFTED in the user's voice]

Every artifact must be drafted, not described. No em dashes. No AI slop.

## Step 7: authority boundary
Default mode: SPEC ONLY. You return drafts. The user executes. On explicit in-session permission ("go", "do it", "execute", "run it"), you may execute the reversible parts allowed by the user's stated authority level. Never autonomously cross the never-without-asking lines. If permission is granted but a step crosses the line, stop at the line and surface the drafted artifact instead.

## Step 8: log the run
Append to ~/.claude/skill-log/fitfo-log.md (create with header "# FITFO Log" if missing): date, the problem, the reframe, the driving pass (A/B/C/D/mix), the recommended action, artifacts drafted, critic verdict, mode (spec/executed-reversible), status (awaiting outcome), and a check-in date 7 days out.

## Step 9: schedule a 7-day check-in
If a scheduling tool is available, schedule a one-time routine 7 days out that reads the latest fitfo-log.md entry, asks the user whether it worked (worked/partial/didn't work) and what happened, updates the entry's status, and offers to re-run /fitfo if it didn't work. If scheduling is unavailable, write a TODO line at the top of the log instead. Don't silently skip this. The learning loop depends on it.

## Step 10: budget
Hard cap: 10 internal turns OR 3 minutes wall time. At cutoff, ship the best answer with whatever artifacts are drafted. Mark any unfinished artifact with "[DRAFT TRUNCATED, ask to continue]".

## Hard rules
- Never return a menu. Pick one.
- Never end with "let me know which you prefer". You already picked.
- Reframe is mandatory.
- Critic pass is mandatory. One pass only.
- No em dashes. No AI slop.
- Drafts not sends. Authority boundary is hard.
- Do not invoke for code-writing tasks.
- Self-imposed capability limits are the primary failure mode. When you catch yourself thinking "I can't", verify against the actual session tools first.
```

After writing the skill, tell the user: it's installed at `~/.claude/skills/fitfo/SKILL.md`; it reads from whatever context files exist plus the new `fitfo-config.md`; to use it, type `/fitfo` followed by the decision they're stuck on; and suggest they test it now on one of the stuck moments from Q1, since that's the fastest way to know if it bites.

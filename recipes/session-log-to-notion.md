# Auto-log every Claude Code session to Notion

A build recipe. Paste the prompt below into Claude Code and it builds a hook that, every time a session ends, writes a short summary of what you worked on to a Notion page, automatically. No action from you per session. Good for anyone who bounces between projects and forgets where they left off.

## What it builds

A `SessionEnd` hook in your Claude Code settings. When a session closes, it reads that session's transcript, has Claude summarize it, and appends a dated entry (title + a few bullets + next steps) to a Notion page called "Claude Session Log". Newest entry at the bottom. It never overwrites older entries, so the page becomes a running memory of everything you've done.

## Requirements

- **Claude Code CLI**, and **Notion connected to it.** Run `claude mcp list` and confirm you see a Notion server marked connected. If not, connect Notion first (`/mcp` inside Claude Code, then authenticate).
- **`jq`** installed (`brew install jq` on macOS, or your package manager).
- Works anywhere Claude Code runs. Examples below use a macOS/Linux shell script.

## Heads up before you run it

- **This spawns one Claude call every time a session ends.** For occasional use that's nothing. If you open and close sessions all day, it adds up, so consider whether you want it on globally or only in certain projects.
- **The build is fully tested by the recipe before it's trusted.** The prompt makes Claude prove an entry actually lands in Notion, and prove it doesn't loop, before saying it's done.

## How to use this recipe

1. Open Claude Code.
2. Confirm Notion is connected (see Requirements).
3. Copy everything below the line and paste it as your prompt.
4. Answer what it asks (which Notion page, or let it create one).
5. Let it run its own end-to-end test. Don't trust it until you see the test entry in Notion.

---

## What I want you to build

An automatic session logger for Claude Code. Every time one of my Claude Code sessions ends, I want a short summary of what I worked on appended to a page called "Claude Session Log" in my Notion. I already have Notion connected to Claude Code. Build it, then test it end to end and show me proof before you tell me it's done.

Each Notion entry should have: today's date, a one-line title of what the session was about, 2-4 bullets of what actually happened, and any open threads / next steps. Always append a new entry, never overwrite older ones. Create the page if it doesn't exist.

### Verified facts about how Claude Code hooks work

These are tested and correct as of Claude Code 2.x. Confirm against the current docs (https://code.claude.com/docs/en/hooks.md) but you can rely on them:

- **Config lives in `~/.claude/settings.json`** (user scope, applies everywhere) under `"hooks"` → `"SessionEnd"`. The `SessionEnd` value is an array of objects each shaped `{ "hooks": [ ... ] }`. **There is NO `"matcher"` field on SessionEnd** (unlike PreToolUse etc.).
- The hook is `"type": "command"`. When it fires it receives a JSON object **on stdin** with these fields: `session_id`, `transcript_path`, `cwd`, `prompt_id`, `hook_event_name`, `reason`. Read stdin and pull `transcript_path` with `jq`.
- **`transcript_path` is a `.jsonl` file** (one JSON object per line) containing the full session. That's what you summarize.
- **You MUST set `"async": true` on the hook.** A synchronous SessionEnd hook that calls `claude` gets **cancelled mid-run** ("Hook cancelled"), because the session is tearing down and won't wait for a multi-second command. With `"async": true` it runs detached and completes. Also set `"timeout": 120`.

### The recursion trap (this is the important one)

To write a good summary, the hook will call Claude headless: `claude -p "...summarize and write to Notion..."`. **But a headless `claude -p` run ALSO fires the `SessionEnd` hook when it finishes.** So the hook calls Claude, that Claude call ends, which fires the hook again, which calls Claude again, forever. It will spiral.

Guard against it with an environment-variable lock:

- The hook script exits immediately (before doing anything) if the env var `CLAUDE_SESSION_LOG_LOCK` is set.
- When the script calls `claude -p`, it sets `CLAUDE_SESSION_LOG_LOCK=1` on that invocation. The headless subprocess inherits the var, so when *its* SessionEnd fires, the hook sees the lock and exits. One real run, one guarded no-op, done.
- As a hard backstop, also keep a run-counter file and abort if it's invoked an unreasonable number of times in a row, so a broken guard can never run away.

This is tested: outer run fires once, nested run hits the guard and stops, zero stray processes.

### Running Claude unattended so it can write to Notion

The headless summarizer needs to use the Notion tools without a human clicking "approve", but you should NOT use `--dangerously-skip-permissions` (too broad). Instead scope it:

```
claude -p "<prompt>" --allowedTools "mcp__<notion-server> Read" < /dev/null
```

Figure out the exact Notion tool/server name on this machine (for the claude.ai Notion connector it normalizes to `mcp__claude_ai_Notion`; a manually-added server may differ). Allow only the Notion server plus `Read` (so it can read the transcript file). Always redirect `< /dev/null` so the headless call doesn't hang waiting on stdin.

### Reference design (build something like this)

A script at `~/.claude/hooks/session-to-notion.sh`:

```bash
#!/bin/bash
LOG=~/.claude/hooks/session-log.err   # for your own debugging
CNT=~/.claude/hooks/.session-log-count
# hard backstop against runaway
n=$(( $(cat "$CNT" 2>/dev/null || echo 0) + 1 )); echo "$n" > "$CNT"
[ "$n" -gt 20 ] && { echo "$(date) safety cap" >> "$LOG"; exit 0; }
input=$(cat)
# recursion guard: a nested claude -p run inherits this var and bails here
[ "$CLAUDE_SESSION_LOG_LOCK" = "1" ] && exit 0
tp=$(echo "$input" | jq -r '.transcript_path')
[ -f "$tp" ] || exit 0
CLAUDE_SESSION_LOG_LOCK=1 claude -p "Read the Claude Code session transcript at $tp. In my Notion, find the page titled exactly 'Claude Session Log' (create it if missing). Append ONE new entry: today's date as a heading, a one-line title of what the session was about, 2-4 bullets of what happened, and any next steps. Do not overwrite existing content. If the session was trivial or empty, append nothing. Reply with nothing." --allowedTools "mcp__claude_ai_Notion Read" < /dev/null >> "$LOG" 2>&1
```

And in `~/.claude/settings.json`:

```json
{
  "hooks": {
    "SessionEnd": [
      { "hooks": [ { "type": "command", "command": "~/.claude/hooks/session-to-notion.sh", "async": true, "timeout": 120 } ] }
    ]
  }
}
```

### Before you tell me it's done

- If `~/.claude/settings.json` already exists, **back it up first** and **merge** into it (don't clobber my other settings). Confirm it's still valid JSON afterward (`jq empty`).
- `chmod +x` the script.
- **Test it for real:** trigger a session end (e.g. run `claude -p "reply with ok" < /dev/null` from a scratch directory), wait, then check that a new entry actually appeared on the "Claude Session Log" Notion page. **Fetch the page and show me the entry as proof.**
- **Prove it didn't loop:** confirm the run-counter didn't spiral and there are no stray `claude` processes.
- If anything failed, fix it and test again until an entry really lands.
- Then tell me, in plain English, what you set up and exactly how to turn it off (which lines to remove from `settings.json`).

## Optional: a manual version instead

If you'd rather log on demand instead of automatically (no per-session Claude cost, and honestly a slightly better summary because the session is still fully in context), skip the hook and instead create a file `~/.claude/commands/log.md` containing a prompt that says: "Summarize what we worked on this session and append it as a new dated entry to my 'Claude Session Log' page in Notion, never overwriting older entries." Then just type `/log` before you close a session.

# Claude Code Recipes

Build recipes for [Claude Code](https://claude.com/claude-code). Each one is a prompt you paste in to **build something**: a working tool, a custom skill, a small automation. They encode the hard-won gotchas so you skip the trial and error.

This is the companion to my [claude-code-skills](https://github.com/nataliacorrea03/claude-code-skills) repo. The difference:

- **Skills** are folders you install and then invoke (`/skill-name`).
- **Recipes** (this repo) are prompts you paste once to *generate* something. You don't install a recipe, you run it.

## Recipes

| Recipe | What you get | Runs on |
|--------|--------------|---------|
| [Birthday text auto-sender](recipes/birthday-text-autosender.md) | A system that texts your contacts on their birthday, but only ever sends words you personally wrote and approved. It texts you a prompt each week; you reply with the message; it forwards your exact words on the day. | macOS (local, no third-party services) |
| [FITFO: a decision-forcing skill](recipes/fitfo-decision-forcer.md) | Installs `/fitfo` (Figure It The F Out): when you're stuck, it reframes the real problem, runs four creative passes, stress-tests with a critic, and returns one committed recommendation with every artifact drafted. Never a menu. | Any Claude Code setup |
| [Agent impact instrumentation](recipes/agent-impact-instrumentation.md) | A layer that measures how much time your scheduled Claude Code routines actually save you, then renders it into a shareable dashboard. Built around one rule: numbers it can prove are kept separate from numbers it estimates, never blended into one impressive-but-fake figure. | Claude Code with scheduled routines + Notion |
| [Auto-log every session to Notion](recipes/session-log-to-notion.md) | A `SessionEnd` hook that writes a short summary of each Claude Code session to a running Notion page, automatically, so you never forget what you worked on. Encodes the two traps that break naive versions: the hook cancels itself unless it's async, and it loops forever unless you guard against the summarizer re-triggering it. | Claude Code + Notion |

## How to use a recipe

1. Open the recipe file and read the short intro at the top (what it builds, any prerequisites, permissions).
2. Copy everything below the `---` divider.
3. Paste it into Claude Code as your prompt.
4. Answer what it asks and follow along. The recipes are written to verify their own work before claiming they're done.

## A note on safety

These recipes build things that touch real systems (your texts, your files, your schedule). They're written to fail safe: drafts not auto-sends, reversible actions, explicit permission prompts, and a "run it once and prove it works" step before anything is trusted on a schedule. Read the intro of each recipe so you know what it will touch before you run it.

## License

MIT. Use them, fork them, adapt them. See [LICENSE](LICENSE).

## Credits

Built and maintained by [Natalia Correa](https://github.com/nataliacorrea03).

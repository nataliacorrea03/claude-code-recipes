# Birthday text auto-sender (Mac + Claude Code)

A build recipe. Paste the prompt below into Claude Code on your Mac and it builds a system that texts your contacts a happy-birthday message on their birthday, but **only ever sends words you personally wrote and approved.**

## What it builds

Once a week the system texts *you* a prompt for each upcoming birthday. You reply with the exact message you want sent. On the birthday itself, it forwards your exact words to that person. No reply from you means nothing sends. You get per-person customization with zero risk of an AI texting your friend something weird.

Runs entirely on macOS built-ins (Contacts, Messages, `launchd`, `sqlite3`, `python3`). No third-party services, no APIs, nothing paid.

## Requirements

- **A Mac with iMessage set up.** Messages must be signed into your Apple ID and able to send texts from this Mac. This recipe drives the Messages app directly, so it only works on macOS, not on Windows, Linux, or a phone alone.
- **Set up iMessage sending first.** Before running this, install the [`imessage` skill](https://github.com/nataliacorrea03/claude-code-skills/tree/main/skills/imessage) from the companion repo. It walks you through (and verifies) the exact macOS permissions iMessage sending needs. This recipe builds scheduled birthday sending on top of that same mechanism, so getting the permissions sorted there first saves you the headache here.

## How to use this recipe

1. Open Claude Code on your Mac.
2. Make sure iMessage sending works (see Requirements above).
3. Copy everything below the line and paste it as your prompt.
4. Answer anything it asks (your phone number, who to exclude).
5. Grant the macOS permissions it requests (see the permissions section in the prompt).
6. Let it run the weekly script once so you can confirm the prompt text lands before trusting the schedule.

The prompt is written in the first person on purpose, so you can paste it as if you are the one asking.

---

## What I want you to build

A system on my Mac that texts my contacts a personalized happy-birthday message on their birthday, but never sends anything I didn't personally approve and write.

The trick: it does NOT auto-generate the message. Once a week it texts ME a prompt for each upcoming birthday. I reply with the exact text I want sent. On the actual birthday, it forwards my exact words to that person. No reply from me means nothing sends. This gives me per-person customization with zero "an AI texted my friend something weird" risk.

Everything runs locally with macOS built-ins (Contacts, Messages, `launchd`, `sqlite3`, `osascript`, `python3`). No third-party services, no APIs, no paid tools.

## Architecture (two stages)

**Stage 1, weekly prompt (runs Sunday late morning).** A script `run.sh`:
1. Reads macOS Contacts for everyone whose birthday falls in the current Sunday-through-Saturday window.
2. Drops anyone on an exclude list.
3. For each remaining person, texts MY OWN phone a numbered prompt: `🎂 #1 Jane Doe, Tuesday June 30, reply with '1: <text>' or '1: skip'`.
4. Records the GUID of that outbound prompt message.
5. Writes a small JSON state file per birthday (name, phone, date, prompt index, prompt GUID, timestamp).
6. Schedules a one-shot `launchd` agent per birthday to fire at the send time ON that birthday.

**Stage 2, day-of send (one agent per birthday).** A script `send_one.sh <state_file>`:
1. Reads the state file.
2. Looks up my reply in the Messages database.
3. If I replied with a real message, forwards that exact text to the recipient.
4. If I replied "skip" or didn't reply at all, sends nothing and exits quietly.
5. Deletes its own scheduled agent afterward so nothing lingers.

## How I reply

Each weekly prompt is numbered. I reply in a normal text with `<N>: <message>`:
- `1: HAPPY BIRTHDAY 🎂🥳` sends that to person #1.
- `3: skip` sends nothing to person #3.
- I can reply in any order, and re-replying with the same number overwrites (most recent wins).

## CRITICAL gotchas: bake these in, they were all learned the hard way

1. **Contacts won't auto-launch under `launchd`.** Calling `tell application "Contacts"` from a `launchd` job fails with error -600 because the job can't launch the app. Fix: run `open -a Contacts` first, then `sleep 4` to let it come up, THEN read. Do the same defensively for Messages if needed.

2. **Read the Messages DB from a COPY, never live.** Messages keeps `~/Library/Messages/chat.db` open in WAL mode and writes to it constantly. Querying it live gives lock errors and partial reads. Always `shutil.copy` it to a temp file (e.g. `/tmp/chat.db.snap`), query the copy, then delete the copy.

3. **chat.db lags the send.** After you send a message via `osascript`, the row isn't in chat.db instantly. `sleep 1.5` before you snapshot-and-query for the GUID you just created.

4. **To capture "the message I just sent," query most-recent-outbound AFTER sending.** Send first, then `SELECT m.guid FROM message m JOIN handle h ON m.handle_id = h.ROWID WHERE h.id = ? AND m.is_from_me = 1 ORDER BY m.date DESC LIMIT 1`. The newest outbound row to that number is the one you just sent.

5. **Apple timestamp math.** chat.db `message.date` is nanoseconds since 2001-01-01 UTC, not Unix epoch. To compare against a Python datetime: `apple_ts = int((dt.timestamp() - 978307200) * 1_000_000_000)`. Forgetting the 978307200 offset (seconds between 1970 and 2001) makes every time filter wrong.

6. **Match my reply by NUMBERED PREFIX as the primary path, not iMessage's Reply feature.** Originally this relied on iMessage's swipe-to-Reply, which links via `associated_message_guid`. That's fragile and easy to forget to use. The robust primary path is: find my most recent inbound message (after the prompt was sent) whose text matches `^\s*#?<index>\s*[:.\-)]\s*(.+)`. Keep the `associated_message_guid LIKE %prompt_guid%` lookup as a FALLBACK only. Also support an optional `manual_reply` field in the state file as a third path.

7. **Strip the numbered prefix before forwarding.** If I reply `3: happy bday!!`, send "happy bday!!" not "3: happy bday!!". Regex-strip a leading `^\s*#?\d+\s*[:.\-)]\s*` before sending. If nothing remains after stripping, send nothing.

8. **"skip" check must be FIRST-WORD only.** Skip only if the first word (lowercased, punctuation stripped) equals "skip". This way `skip`, `skip!`, `skip.`, `skip her` all skip, but a real message that happens to contain "skip" later still sends.

9. **Escape AppleScript strings.** Before injecting my reply into the `osascript` send command, escape double-quotes (`"` to `\"`) and convert newlines to `\n`. Unescaped quotes break the AppleScript and the send silently fails.

10. **Normalize phone numbers.** Strip everything except digits and `+`. If 10 digits, prepend `+1`. If 11 digits starting with `1`, prepend `+`. Contacts stores numbers in wildly inconsistent formats.

11. **Per-birthday agents must not fire on load.** Set `RunAtLoad` to `false` in each one-shot plist, and use `StartCalendarInterval` with only Month + Day + Hour + Minute (no year). Otherwise loading the plist fires it immediately.

12. **Self-cleanup, both ends.** After a day-of agent fires, it should `launchctl unload` and delete its own plist. The weekly run should also sweep state files and stale plists older than 7 days so the LaunchAgents folder doesn't fill up with dead one-shots.

13. **Compute the week window from today's weekday.** Sunday = `today - ((weekday + 1) % 7)` days, then the 7 days from there. Match a contact's birthday on month+day only (ignore birth year).

## Things to make configurable (do NOT hardcode)

Put these at the top of `run.sh` as clearly labeled variables:
- `MY_PHONE`: the number that receives the prompts and sends the replies (in `+1XXXXXXXXXX` form).
- `EXCLUDE`: a set of lowercased full names to never prompt for. Start it empty and let me fill it.
- `LAUNCH_LABEL_PREFIX`: namespace for the launchd labels (e.g. `com.<myname>.bday`).
- `PROMPT_TIME` (weekly, e.g. Sunday 11:13am) and `SEND_TIME` (day-of, e.g. 12:18pm), both local time.

## Permissions I'll need to grant (tell me to do these, don't assume they're done)

- **Full Disk Access** for the terminal/process running the scripts, so it can read `~/Library/Messages/chat.db`. This is a broad permission. Flag it to me explicitly before relying on it.
- **Automation, Messages** so `osascript` can send texts.
- **Contacts access** so the script can read birthdays.

## Deliverables

1. `run.sh` (weekly) and `send_one.sh` (day-of), commented.
2. A weekly `launchd` plist (e.g. `~/Library/LaunchAgents/<prefix>-weekly.plist`) firing `run.sh` at `PROMPT_TIME`.
3. A short README covering: how to load the weekly agent, how to run it manually to test, how to check status (`launchctl list | grep <prefix>`), and how I reply.
4. After building, run `run.sh` manually once so I can confirm I get the prompt texts, before trusting the schedule.

Don't claim it works until you've actually run the weekly script once and shown me the prompt landed. If a step is permission-blocked, tell me what to click rather than guessing.

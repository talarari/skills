---
name: cloud-scheduled-runs
description: Create recurring scheduled runs (Routines/triggers) in a Claude Code cloud session that actually fire and actually have repo access. The obvious setup fails silently and you only discover it a day later, reporting "no access to the repo" every morning. Covers session binding, UTC cron, unattended prompts, permissions, connectors, and how to verify a routine is really working. TRIGGER when the user asks to schedule something daily/recurring, mentions a "routine", "cron", "scheduled task", "run this every morning", "check X every day", or when an existing scheduled run keeps failing with repo-access, permission, or silent-no-op errors.
allowed-tools: [Bash, Read, Write, Edit]
---

# Scheduled runs in a Claude Code cloud session

A trigger that fires but cannot reach the repo looks identical to a trigger that works —
until you read the transcript a day later. Get the binding right **first**; everything
else in this file is secondary.

## The failure that costs you a week

**Do not use `create_new_session_on_fire: true` for anything that touches a repo.**

Each cloud session reaches GitHub through a *per-session* git proxy — roughly
`http://local_proxy@127.0.0.1:<port>/git/<owner>/<repo>`, wired in by a global
`url.<proxy>.insteadOf=https://github.com/` git config. That proxy authorizes only the
repos attached to **that specific session**. A session spawned fresh by a trigger has no
repo attached, so every `git clone` / `git fetch` returns 401 and the run fails the same
way every single day.

Worse, the spawned session has no `add_repo` tool either, so it cannot rescue itself. It
will report something vague like "I don't have access to the repository" and move on.

Prove the mechanism rather than trusting this text — request an attached repo through the
proxy (expect `200`) and an unattached one (expect `401`):

```bash
git config --get-regexp 'url\..*\.insteadof'      # find the proxy base
curl -s -o /dev/null -w '%{http_code}\n' "$PROXY/<owner>/<attached-repo>/info/refs?service=git-upload-pack"
curl -s -o /dev/null -w '%{http_code}\n' "$PROXY/<owner>/<other-repo>/info/refs?service=git-upload-pack"
```

`GITHUB_TOKEN` / `GH_TOKEN` do **not** rescue this. They are API-only credentials; GitHub
rejects them for git transport with *"Password authentication is not supported for Git
operations."* Don't spend an hour there.

## The binding that works

Fire into a session that already has the repo:

- **Self-bind** — omit `persistent_session_id` entirely. The default binds to the calling
  session. This is what you want when the user says "post it here every morning".
- **Bind to a named session** — `persistent_session_id: "session_..."`. Get the id from
  `get_session` with no arguments.

Firings then arrive as ordinary user turns inside that conversation, which keeps its repo
access, its working directory, and its permission grants.

```
create_trigger({
  name: "Daily report",
  cron_expression: "0 11 * * *",     // UTC — see below
  initiation: "human_request",
  prompt: "<the unattended prompt — see below>"
  // no persistent_session_id  => self-bind
  // no create_new_session_on_fire
})
```

**The binding mode is not editable afterwards.** `update_trigger` changes name, schedule,
enabled state and prompt while preserving run history — use it for those. To change *what
the trigger fires into*, you must delete and recreate.

## Cron is UTC

Convert from local time before writing the expression. Verified: 14:00 Israel time in
summer (UTC+3) is `0 11 * * *`.

If the conversion crosses midnight, shift the **day** fields too — day-of-week and/or
day-of-month, whichever you set — not just the hour. Minimum interval is normally hourly.

After creating, read `next_run_at` back from `list_triggers` and check it against the wall
clock you actually intended. This catches a timezone error before it costs a day.

## Writing the prompt that fires

The prompt executes unattended. Nobody is there to answer anything.

- **Say it is unattended.** Open with "this is an automated, unattended run — do not ask
  questions, do not ask for approvals, complete every step." Without this the run stops at
  the first judgement call and does nothing.
- **Make it self-contained.** Spell out the repo path, branch, exact commands, and any
  URLs. Even a self-bound session may have been compacted or idle for days; do not rely on
  it "remembering" how the task works.
- **Step one is always syncing.** `git fetch origin <branch> && git checkout <branch> &&
  git pull origin <branch>`.
- **Define the failure behavior explicitly.** This is the part people skip and regret. If a
  data source is down, the run will otherwise publish stale output as though it were fresh.
  Require a literal one-line statement — "this did not refresh, showing data from
  `<date>`" — in *both* the chat message and any notification.
- **Say when to stay quiet.** A routine that reports "nothing to do" every day trains the
  user to ignore it. State the no-op behavior: no push notification, one short line.
- **No secrets in the prompt.** It is stored server-side and echoed back by
  `list_triggers`.

## Permissions

An unattended run cannot answer a permission prompt — it stalls, and the routine silently
produces nothing. Pre-approve everything it needs in `.claude/settings.json`:

```json
{
  "permissions": {
    "allow": [
      "Bash(git fetch:*)", "Bash(git checkout:*)", "Bash(git pull:*)",
      "Bash(git add:*)", "Bash(git commit:*)", "Bash(git push:*)",
      "Bash(./update.sh)", "Bash(python3 script.py)"
    ]
  }
}
```

Add every tool the routine calls, not just shell commands. Note that `extra_allowed_tools`
on a spawned session can only *narrow* what the parent already has — it can never widen it.

## Connectors

A trigger created from inside a session can only pass through connectors that **that
session itself holds**. If it holds none, the fired runs have no connector tools at all and
the create call warns you.

So a routine that needs Gmail, Calendar, or similar cannot be created from a session
lacking them — no amount of prompt wording fixes it. Create it from a session that holds
those connectors, or from the claude.ai Routines UI. If the connectors are disabled at the
account level (`enabledInChat: false`), that is a settings change the user has to make; say
so plainly instead of building a routine that will never have the data.

## Other constraints worth knowing up front

- **`notifications` is rejected** for self-bind and `persistent_session_id` triggers. It
  only applies to fresh-session-per-fire routines. Send a push from inside the prompt
  instead.
- **Queued firings arrive together.** If the bound session is idle for a day or two,
  several firings stack up and land at once. Handle it: run **once** against current data
  and state which days were missed — don't replay each one and publish three reports.
- **Make the underlying script idempotent and incremental.** Save progress as you go so a
  killed, blocked, or rate-limited run resumes instead of starting over.
- **GitHub Actions is a different answer.** If the user specifically wants a scheduled
  *Claude session* — one that can read the data, judge it, and write a message — an Action
  is not a substitute; say so rather than quietly swapping it in. (And `schedule:` in
  Actions only fires from the default branch, which surprises people separately.)

## Verify it is actually working

Creating the trigger is not evidence. Come back after the first real firing.

```
list_triggers({})   # check next_run_at, enabled, and last_run
```

A `FAILED` or repeatedly non-`SUCCEEDED` `last_run` means the routine is not doing its job,
however healthy the trigger looks.

**Self-bound triggers that wake their own session may not record `last_run` at all.** For
those, verify by looking for the work itself: a commit on the branch, a republished
artifact, a message in the conversation. Absence of an error is not success.

## Turning one off

`update_trigger({trigger_id, enabled: false})` keeps the routine and its history so it can
be re-enabled in one call. Prefer it over `delete_trigger` when the work might come back —
a backfill that has caught up, a monitor for a finished migration.

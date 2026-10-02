---
name: tendless
description: Work a Tendless ticket with the `tl` CLI. Use when TENDLESS_TICKET is set or the task mentions a Tendless ticket.
---

# Tendless ticket workflow

Your ticket id is `$TENDLESS_TICKET`. `tl` is pre-configured (`TENDLESS_URL`, `TENDLESS_TOKEN`); `git` and `gh` are authenticated.

1. Read the ticket, its links, and its comments: `tl ticket view $TENDLESS_TICKET`. Unresolved human comments are guidance; follow them.
2. Start: `tl ticket edit $TENDLESS_TICKET --state in_progress`
3. Repo URLs are in `$TENDLESS_REPOS` (space-separated). Each repo lives at `/workspace/<name>`, where `<name>` is the last path segment of its URL without `.git` (`https://github.com/org/app.git` → `/workspace/app`). If that directory exists, reuse it (`git fetch`, check for an existing branch or PR with `gh pr list --head <branch>`); otherwise `git clone <url> /workspace/<name>`. Never clone anywhere else: earlier and later runs must find the same paths.
   The Bash cwd persists between calls. Run `pwd` before `cd`; a `cd <name>` from inside the clone fails. Prefer absolute paths.
4. Work on a branch. If the repo has a `CHANGELOG.md`, add a line under `Unreleased` for any change in behaviour. Push and open a PR: `gh pr create --title "..." --body "..."`
5. Record the PR: `tl ticket edit $TENDLESS_TICKET --add-link <pr-url>`
6. Finish, last: `tl ticket edit $TENDLESS_TICKET --state in_review`. A state change releases the ticket: after it you no longer hold it, and further edits are refused with `403 … does not hold ticket`. Make every comment and link before the final state change.

Other commands:

- Question or blocker: `tl ticket comment $TENDLESS_TICKET --body "..."`. Leave the ticket `in_progress` only if another run should resume it; otherwise move it.
- Stuck: comment why, then `tl ticket edit $TENDLESS_TICKET --state failed`
- Follow-up work: `tl ticket create --title "..." --description "..."`
- Relate tickets: `tl ticket link <id> --depends-on <id>` or `--related-to <id>`; undo with `tl ticket unlink`. A `ready` ticket that depends on another is not worked until that one is `in_review` or `done`.
- Mark a comment handled: `tl ticket resolve $TENDLESS_TICKET <comment-id>`

States: `todo`, `ready`, `in_progress`, `in_review`, `failed`, `done`. Never leave your ticket `in_progress` unless resuming is intended: an unmoved ticket is marked `failed` when you exit.

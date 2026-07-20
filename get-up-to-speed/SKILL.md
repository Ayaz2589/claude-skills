---
name: get-up-to-speed
description: Recover working context after a context clear (or at the start of a fresh session). Reads the documentation the user points at — or, with no arguments, the standard recovery set (latest context summary, the project's agent-guidance file, README, active feature docs) plus git state — and reports back what has been done so far and what the next steps are. Companion to /remember, which writes the summaries this skill reads.
user-invocable: true
argument-hint: "[optional: path(s) to documentation to read, space-separated]"
---

# get-up-to-speed skill

Invoked right after a `/clear` (or in a brand-new session) to rebuild working
context. The deliverable is a **briefing**: what the project/feature is, what
has been done so far, and what the next steps are (if any). Do NOT start doing
the pending work — report, then stop and let the user direct.

When this skill is invoked, do the following in order:

## Step 1: Gather the sources

**If the user passed arguments**, treat them as the documentation to read.
Arguments may be file paths, directories (read the obvious entry points inside,
e.g. `index.md`, `README.md`, `spec.md`), or glob-ish hints. Read every one.
If a named file doesn't exist, say so — don't silently skip it.

**Always also read the standard recovery set** (skip any that don't exist):

1. `.claude/context-summaries/latest.md` — the previous session's handoff
   (written by `/remember`). This is the highest-signal source; read it first.
2. Your project's agent-guidance file at the repo root (`CLAUDE.md` or
   `AGENTS.md`) — architecture and project guidance. If it points at other docs
   (e.g. a plan or a docs index), follow those pointers.
3. `README.md` at the repo root — quick-start / operational knowledge.
4. **Active feature docs** (only IF your project uses a spec/feature-dir
   convention): if the current git branch maps to a feature directory — for
   example, a `NNN-<feature>` branch mapping to a `specs/NNN-<feature>/`
   directory — read its `spec.md`, `plan.md`, and `tasks.md` if present.
   `tasks.md` (or its equivalent) is the best source for done-vs-pending — note
   which tasks are checked off. If the project has no such convention, skip this.

To find the workspace root: walk up from the current working directory until
you find a `.claude/` directory.

## Step 2: Cross-check against reality

Docs and summaries reflect the moment they were written. Verify against the
repo's actual state before reporting:

```bash
git branch --show-current
git log --oneline -15
git status --short
```

- Commits newer than the latest summary mean work happened after it was
  written — fold those commits into "what has been done".
- Uncommitted changes in `git status` are in-flight work; call them out
  explicitly (the previous session may have stopped mid-task).
- If the summary references files, branches, or tasks, spot-check that they
  still exist as described. Where the docs and the repo disagree, **trust the
  repo** and flag the discrepancy.
- If the project has CI-driven feedback (check the agent-guidance file), check
  the latest run status if it's cheap to do so (e.g. `gh run list --limit 3`).

## Step 3: Report the briefing

Output a concise briefing to the user with these sections:

```markdown
## Where we are
2-4 sentences: the project/feature being worked on, its goal, and the current
overall state (e.g. "the auth-refresh feature is implemented through the migration
step; CI is green; the rollout step is operator-pending").

## What has been done
Bulleted list of completed work, most recent first. Draw from the session
summary, checked-off tasks, and the git log. Include commit SHAs where useful.

## In flight / unverified
Anything started but not finished: uncommitted changes, open PRs awaiting CI,
tasks marked in-progress, known issues found but not fixed. Omit the section
if there's nothing.

## Next steps
Numbered list of pending work in priority order, drawn from the summary's
"Pending / unresolved" section, unchecked tasks, and any TODOs the docs call
out. If there are genuinely no next steps, say so plainly.

## Open questions
Decisions that were deferred to the user or blockers waiting on input.
Omit the section if there are none.
```

End by asking which next step to pick up, or whether the user has something
else in mind. Do not begin executing next steps unprompted.

---

## What NOT to do

- **Don't start working.** This skill's output is the briefing itself. Wait
  for the user to choose a direction.
- Don't dump file contents back at the user — synthesize. The briefing should
  be readable in under a minute.
- Don't trust stale docs over the repo. If `latest.md` says "tests failing"
  but the log shows a later fix commit, report the current truth.
- Don't fabricate next steps. If the sources don't state any, say "no pending
  next steps recorded" rather than inventing plausible ones.
- Don't re-read the entire docs tree "just in case" — the recovery set plus
  whatever the user passed is enough. Follow pointers only when a source
  explicitly directs you to another doc.

## Relationship to the remember skill

The remember skill (`.claude/skills/remember/SKILL.md`) and this skill are two
halves of one loop:

- **remember** (end of session): write the handoff → user runs `/clear`.
- **get-up-to-speed** (start of session): read the handoff + docs → brief the
  user → continue where the last session left off.

If `.claude/context-summaries/latest.md` is missing, this skill still works —
it just leans harder on the agent-guidance file, README, feature docs, and git
history, and should mention that no session handoff was found.

## Adapt to the project at hand

This skill is project-agnostic. Adapt what "current state" means to the
project (deployment state for web apps, CI status for library work, published
version for a package, etc.). If your project's agent-guidance file
(`CLAUDE.md` / `AGENTS.md`) documents a session-continuity convention, follow
that convention first.

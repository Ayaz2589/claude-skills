---
name: remember
description: Summarize the current session into a persistent file in the project's .claude/context-summaries/ directory, then prompt the user to clear context. After clearing, future sessions can recover state by reading the README, the agent-guidance file (CLAUDE.md / AGENTS.md), and the latest summary. Use when context is filling up (>75%) and you want to continue work in a fresh session without losing important state.
user-invocable: true
---

# remember skill

When this skill is invoked, do the following in order:

## Step 1: Determine the summary directory

The summary goes in the **workspace's** `.claude/context-summaries/` directory
— the same workspace that contains this skill at `.claude/skills/remember/`.

To find the workspace root: walk up from the current working directory until
you find a `.claude/skills/` directory. That's the workspace root. The summary
file goes in `<workspace-root>/.claude/context-summaries/`.

If you're at the workspace root already, that's just `.claude/context-summaries/`.
If you're in a sub-project (e.g., a nested package or app directory), it's
`../.claude/context-summaries/` relative to that sub-project.

Create the directory if it doesn't exist:

```bash
mkdir -p <workspace-root>/.claude/context-summaries
```

One workspace, one shared summary history — covers every sub-project under it.

## Step 2: Write a comprehensive session summary

Create a markdown file at:

```
.claude/context-summaries/{YYYY-MM-DD-HHmm}.md
```

(Use the current UTC timestamp in the filename. Format: `2026-05-28-1538.md`.)

The summary must include the following sections (use markdown headers):

### Required sections

```markdown
# Session summary — {date}

## Date and duration
- Started: {first user message timestamp}
- Ended: {now}
- Approximate duration: {hours/minutes}

## What we worked on
2-4 sentences describing the high-level goal of the session.
Be specific. "We worked on the dashboard" is too vague.
"We discovered and fixed two bugs in the sync path: records
expiring early, and the dedup step not accounting for existing
rows." is the right level.

## Key decisions made
Bulleted list of architectural / strategic decisions reached in this
session. Each decision should be 1-2 sentences explaining what was
decided and why.

## Code changes (commits made)
Bulleted list of git commits made during this session, formatted:
- `{commit-sha}` — short description (one line)

## Current state of the system
Start with what's true for any project:
- Latest commit on the main branch
- Test count and pass/fail status
- Any long-running processes: running/stopped
- Open PRs and their status
- Deployment / release state
- In-flight work not yet committed

Then adapt with whatever domain-specific state matters. For example:
- Trading systems: positions, open orders, stops, halt-lock files
- Web apps: deploy state, env config, open PRs
- Data pipelines: last run timestamp, recent failures
- Libraries: published version, open issues, pending releases

## Pending / unresolved
Bulleted list of:
- Open questions waiting on user input
- Tasks identified but not done
- Known issues we found but haven't fixed
- Decisions deferred to later

## Files to read next session
List the canonical files that should be loaded to recover context:
- `README.md` — quick start
- Your project's agent-guidance file (`CLAUDE.md` / `AGENTS.md`) — architecture reference (if present)
- Any other docs that were central to the session

## Quote-worthy phrases / important commitments
Anything the user said that should be remembered as a guiding principle.
For example: "I commit to NOT building the controls phase until I've
used the first three phases for 60 days."

## What I would tell the next Claude
A short paragraph (3-5 sentences) you'd want a fresh Claude to know.
Most important: the user's current goal, the most recent decision, and
any subtle context that isn't obvious from the files.
```

## Step 3: Update the latest-summary pointer

Update `.claude/context-summaries/latest.md` to be a copy of the file you just
wrote. This is what the project's session-start instruction references.

Default to copying (not symlinking) for compatibility:

```bash
cp .claude/context-summaries/{YYYY-MM-DD-HHmm}.md .claude/context-summaries/latest.md
```

## Step 4: Tell the user what to do next

Output a brief message to the user with:
1. Path to summary file you wrote
2. Brief preview of the summary (first 2-3 lines)
3. Clear instruction: "Run `/clear` to clear context. When you start a new session, the latest summary will be loaded automatically (if your agent-guidance file has a session-continuity instruction), or run the get-up-to-speed skill to load it."

## Step 5: Do NOT actually clear the context

You can't clear the context — only the user can via `/clear`. Just write the
summary, update the pointer, and inform the user.

---

## What NOT to do

- Don't write a summary that's just a bullet list of every message. Be selective.
- Don't include sensitive data (API keys, account numbers, passwords).
- Don't write a summary if `.claude/context-summaries/` doesn't exist — create it first.
- Don't overwrite existing dated summaries — only `latest.md` should be overwritten.
- Don't write the summary to `~/.claude/` — it belongs in the project's `.claude/`.

## Storage and lifecycle

- Dated summaries (`{YYYY-MM-DD-HHmm}.md`) are permanent records of past sessions.
- `latest.md` is updated every time this skill runs.
- The `.claude/context-summaries/` directory should be **gitignored** by the
  project — these are local artifacts, not part of the repo history.
- If the project wants to commit a summary intentionally, the user can
  `git add -f path/to/summary.md`.

## Why this exists

Claude's context window is finite. Long sessions naturally accumulate context
that's mostly irrelevant to current work. Periodically summarizing and clearing
keeps the working set focused without losing institutional knowledge.

The three-file recovery (`README.md` + your project's agent-guidance file
(`CLAUDE.md` / `AGENTS.md`) + `latest.md`) gives a fresh Claude:
- Quick-start operational knowledge (README)
- Deep architecture + decision rationale (agent-guidance file)
- Recent session continuity (latest summary)

Together, they're usually enough to pick up where the previous session left off.

The companion **get-up-to-speed** skill (`.claude/skills/get-up-to-speed/SKILL.md`)
is the read side of this loop: it loads the latest summary plus the recovery set
at the start of a fresh session. This skill writes the summaries that one reads.

## Adapt to the project at hand

This skill is project-agnostic. The summary template uses placeholders that work
for a wide range of projects. Adapt the "current state" section to whatever
matters for the project:

- Trading systems: positions, orders, live process state
- Web apps: deployment state, env config, open PRs
- Data pipelines: last run timestamp, recent failures
- Libraries: published version, open issues, pending releases

If the project's agent-guidance file (`CLAUDE.md` / `AGENTS.md`) already
documents what to track, follow that.

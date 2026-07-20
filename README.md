# claude-skills

A small, portable collection of [Claude Code](https://claude.com/claude-code) skills, extracted and generalized from a real project so any project can drop them in. Each skill is a self-contained folder with a `SKILL.md`; the collection is opinionated about workflow (session continuity, sandboxed parallel agents) but written to be project-agnostic — anywhere a path or command is specific to your stack, the `SKILL.md` uses `<placeholders>` you fill in, so you can adopt a convention as-is or adapt it.

## The skills

| Skill | What it does | User-invocable | Companion |
|-------|--------------|:--------------:|-----------|
| `docker-sandbox` | Spin up and manage Docker Sandboxes (the `sbx` CLI) — one isolated microVM per feature for parallel or overnight agent work; clone/branch/direct modes, bootstrapping a fresh sandbox, and git push/PR from inside one. Args: optional feature name(s) to spin up sandboxes for. | yes | `kill-sandbox` |
| `kill-sandbox` | Tear down a Docker Sandbox by its git branch, with an unpushed-work safety gate, and keep the sandbox registry current. Args: `<branch> [--force]`. | yes | `docker-sandbox` |
| `remember` | Summarize the current session into `.claude/context-summaries/`, then prompt you to `/clear`. | yes | `get-up-to-speed` |
| `get-up-to-speed` | Rebuild working context after a `/clear` (or in a fresh session): read the recovery set plus git state and report a briefing. Args: optional path(s) to read. | yes | `remember` |
| `review` | Review the current (local, uncommitted or current-branch) work: run the project's tests/checks, review the changed code, fix any real issues, then re-test and re-review in a bounded loop until green and clean. Not a GitHub PR review. Args: optional scope — a path, `staged`, or a base branch. | yes | — |

## Installing a skill into your project

Copy the skill's folder into your project's `.claude/skills/<name>/`:

```
cp -r docker-sandbox /path/to/your-project/.claude/skills/docker-sandbox
```

Claude Code discovers skills placed under `.claude/skills/` and makes each one invocable as `/<name>` (e.g. `/docker-sandbox`). Copy only the skills you want — they're independent, except that the two pairs below are designed to be used together. For how skills are discovered and invoked, see the Claude Code skills documentation.

> **A note on the `review` name:** `review` deliberately reuses the name of Claude Code's built-in `/review` (which reviews a published GitHub PR). Installed at `.claude/skills/review/`, it **shadows** that built-in — `/review` will instead mean "test-and-fix my local work." If you want to keep both, install it under a different folder name (e.g. `verify-work`) and match the `name:` in its frontmatter to that folder.

## The two skill pairs

Four of the skills form two companion pairs (`review` stands alone):

- **`remember` ↔ `get-up-to-speed`** — a session-continuity loop. When context fills up, `/remember` writes a summary of the current session to `.claude/context-summaries/` and prompts you to `/clear`. In the next session, `/get-up-to-speed` reads the latest summary (plus your agent-guidance file, README, and git state) and briefs you on what was done and what's next. Both sides expect the shared `.claude/context-summaries/` directory as the handoff location.
- **`docker-sandbox` ↔ `kill-sandbox`** — a sandbox lifecycle pair. `/docker-sandbox` creates and manages sandboxes (one per feature/branch); `/kill-sandbox <branch>` resolves a branch back to its sandbox and removes it after checking for unpushed work. Both keep a shared **sandbox registry file** (default `docs/sandbox/sandbox-history.md`) current so the set of active and killed sandboxes stays accurate.

## docker-sandbox needs a one-time setup

`docker-sandbox` relies on two helper scripts — `bootstrap-sandbox.sh` (prepare a fresh sandbox: credentials, keys, local env files) and `set-github-secret.sh` (wire up git push/PR from inside a sandbox). **These scripts are intentionally not shipped**, because they depend on your project's dependency manager, its local backing services, and its env files — details that can't be generalized. Instead, the `SKILL.md` walks the agent through generating both scripts tailored to your stack. Run this one-time setup once per project before relying on the sandbox workflow.

## Conventions

These skills assume a couple of common project files:

- A **project agent-guidance file** — `CLAUDE.md` or `AGENTS.md` — and a `README.md`; `get-up-to-speed` reads them as part of its recovery set.
- Where a path or command is specific to your project, the `SKILL.md` uses `<placeholders>` (e.g. the sandbox registry path, env-file locations) that you replace with your project's real values.

## License

MIT (placeholder — add your preferred license).

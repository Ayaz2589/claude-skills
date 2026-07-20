---
name: pr-context
description: Fetch the full context of a GitHub pull request — metadata, description, diff, commits, and any linked issue-tracker keys — and present it as one structured briefing. Works in any repo with the `gh` CLI; on first run it asks how your project links issues (Jira / Linear / GitHub) and remembers it.
user-invocable: true
argument-hint: "[PR number or URL]"
---

# pr-context skill

Gather everything needed to understand a pull request in one shot and present it
as a single structured briefing. **Read-only** — this skill never edits, comments
on, approves, or merges the PR.

Requires the GitHub CLI (`gh`), authenticated against the repo's host. If `gh`
isn't installed or authenticated (`gh auth status`), say so and stop.

## Step 0 — Load or initialize config

How issue keys are linked is project-specific, so it's stored in a small config
file the first time the skill runs.

**Locate the workspace root:** the nearest ancestor directory that contains a
`.claude/` folder. If there is none, use the git repo root
(`git rev-parse --show-toplevel`) and create `.claude/` there.

**Config file:** `<workspace-root>/.claude/pr-context.config.md`.

- **If it exists**, read it and extract: tracker type, tracker base URL, and the
  ticket-key prefixes. Skip the rest of this step.
- **If it does NOT exist**, this is the first run. Gather the info with one
  `AskUserQuestion` block:
  1. **Issue tracker** — `Jira`, `Linear`, `GitHub Issues`, or `None`.
  2. **Ticket-key prefixes** (only meaningful for Jira/Linear) — the project's
     issue prefixes, e.g. `EN, CS, CT`. These are the letters before the dash in
     keys like `EN-1234`. (For GitHub Issues, keys are just `#123` — no prefixes
     needed. For None, skip.)

  Then resolve the base URL used to build links:
  - **Jira** → `https://<org>.atlassian.net/browse/` — infer `<org>` from the
    git remote if you can, otherwise ask.
  - **Linear** → `https://linear.app/<org>/issue/`.
  - **GitHub Issues** → link `#<n>` to `<repo-url>/issues/<n>` (derive the repo
    URL from `gh repo view --json url`).
  - **None** → no linking; issue extraction is skipped on every run.

  Write the config exactly in this shape:

  ```markdown
  # pr-context config

  ## Issue tracker
  - Type: <jira | linear | github | none>
  - Base URL: <url, or "-">

  ## Ticket-key prefixes
  - <comma-separated prefixes, or "-">
  ```

  Tell the user where you saved it and that they can hand-edit it later — commit
  it to share the convention with the team, or gitignore it; their call.

Proceed with the loaded/collected values.

## Step 1 — Resolve the PR

The PR is given by `$ARGUMENTS` — a number (e.g. `1234`) or a full PR URL.

- If `$ARGUMENTS` is empty, try the current branch's PR: `gh pr view --json number`.
- If that finds nothing, ask the user for a PR number and stop until they answer.

## Step 2 — Fetch everything

Run these against the resolved PR (`<pr>` is the number or URL):

1. **Metadata** —
   `gh pr view <pr> --json number,title,state,isDraft,author,labels,createdAt,updatedAt,url,headRefName,baseRefName,reviewRequests,additions,deletions,changedFiles,mergeable,reviewDecision`
   If the PR isn't found, tell the user and stop.
2. **Body** — `gh pr view <pr> --json body`.
3. **Diff** — `gh pr diff <pr>`.
4. **Commits** — `gh pr view <pr> --json commits`.

## Step 3 — Extract linked issue keys

Skip entirely if tracker Type is `none`.

Build a case-insensitive regex from the configured prefixes and scan the PR
**title + body** (and branch name):

- Jira/Linear: `\b(PREFIX1|PREFIX2|…)-\d+\b` (e.g. `\b(EN|CS|CT)-\d+\b`).
- GitHub Issues: also capture `#\d+`.

Dedupe the matches, preserving first-seen order. For each key, render a markdown
link using the base URL (`[EN-1234](https://org.atlassian.net/browse/EN-1234)`).

## Step 4 — Present the briefing

Output one structured summary:

```
## PR #<number>: <title>   <"(draft)" if isDraft>
**URL:** <url>
**State:** <state> · **Mergeable:** <mergeable> · **Review:** <reviewDecision or "none">
**Author:** <author> · **Branch:** <head> → <base>
**Created:** <date> · **Updated:** <date>
**Changes:** +<additions> −<deletions> across <changedFiles> files
**Labels:** <labels or "none">
**Reviewers:** <requested reviewers or "none">
**Linked issues:** <rendered links, or "none">

### Description
<PR body — verbatim, or "(no description)">

### Commits
<one line per commit: short SHA — message>

### Diff
<full diff, in a fenced ```diff block>
```

If the diff is very large, show the `--stat` summary first, then the full diff
below it (don't silently truncate — say if you're abbreviating and why).

## What NOT to do

- **Don't act on the PR.** No `gh pr review`, `gh pr comment`, `gh pr merge`,
  `gh pr close`, or edits. This skill reports; it does not change state.
- **Don't invent issue keys.** Only link keys that actually appear in the PR and
  match the configured prefixes.
- **Don't skip the diff** unless it's genuinely too large to include — and if you
  abbreviate, say so.

## Adapt to the project at hand

The only project-specific piece is issue linking, captured in
`.claude/pr-context.config.md`. To change trackers or prefixes later, edit that
file (or delete it to re-run first-run setup). Everything else is standard `gh`
and works in any GitHub repo. For non-GitHub hosts (GitLab/Bitbucket), the same
shape applies but you'd swap `gh` for that host's CLI — not covered here.

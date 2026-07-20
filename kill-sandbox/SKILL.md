---
name: kill-sandbox
description: Use when killing/removing/tearing down a Docker Sandbox (`sbx`) identified by its git branch — e.g. "/kill-sandbox feat/savings-goals" — or when the sandbox registry needs reconciling. Resolves the branch to its sandbox, removes it after a safety check for unpushed work, and keeps the project's sandbox registry (active + killed history) current. Host-only — `sbx` is not available inside a sandbox.
user-invocable: true
argument-hint: "<branch> [--force]   e.g. feat/savings-goals"
---

# kill-sandbox skill

Remove a Docker Sandbox **by its git branch** (not its `sbx` name), safely, and
keep the project's sandbox registry up to date. The registry lives at
`docs/sandbox/sandbox-history.md` by default (project-configurable — use whatever
path your project already uses); it is the **same file the docker-sandbox skill
maintains**.

**Core principle:** a branch (`feat/savings-goals`) is what the user knows; the
`sbx` name (`goals`) is what removal needs. The registry file maps one to the
other and is the source of truth — so **every run reconciles it against `sbx ls`
first**, then resolves, safety-checks, removes, and records. The registry staying
current is the whole point; treat reconcile + record as non-optional.

Companion to the **docker-sandbox** skill (`.claude/skills/docker-sandbox/SKILL.md`,
creation/management). Read its "Environment guard" and "Gotchas" — they apply
here verbatim.

## ⚠️ Host-only + environment guard — do this FIRST, every invocation

`sbx` runs on the **host**, never inside a sandbox. Detect where you are before
anything:

```bash
command -v sbx >/dev/null 2>&1 && echo "sbx: present" || echo "sbx: ABSENT"
echo "SANDBOX_VM_ID=${SANDBOX_VM_ID:-<unset>}"
```

- **Inside a sandbox** (`sbx` ABSENT / `SANDBOX_VM_ID` set): you cannot run `sbx`.
  Emit the host commands for the user to run, prepended with the HOST-ONLY banner
  from the docker-sandbox skill. You also can't reliably reconcile the registry
  (no `sbx ls`), so say so — the file updates on the next host-side run.
- **On the host** (`sbx` present, `SANDBOX_VM_ID` unset): run the flow below.

**All `sbx` commands must run with Claude Code's command-sandbox DISABLED**
(docker-sandbox gotcha #2 — otherwise `sbx ls` collides on its daemon DB lock).

## The flow — `/kill-sandbox <branch> [--force]`

Run these in order. Do not skip step 1 or step 5.

### 1. Reconcile the registry (freshness — always first)
- Run `sbx ls`. Keep only sandboxes whose **workspace is this project's repo path**
  (ignore other projects).
- Read the registry file (default `docs/sandbox/sandbox-history.md`; create it
  from the template below if missing). Diff `sbx ls` against the **Active** table:
  - **Discovered** (in `sbx ls`, not in Active) → add an Active row (branch/feature
    unknown unless you can infer them; `Last seen` = its current status).
  - **Vanished** (in Active, not in `sbx ls`) → it was removed outside this skill →
    move it to **Killed**, carrying over its existing Branch/Feature/Created, with
    `Killed` = today, `Unpushed at kill?` = unknown, Notes = "removed outside skill".
  - **Survivors** → refresh `Last seen` (running/stopped).
- Set the header's `Last reconciled:` to today (`date +%F` on the host). If nothing
  changed and it already reads today, leave the file untouched — don't rewrite it
  just to restamp the date (avoids dirtying git status on every run).

### 2. Resolve branch → sandbox
- Look up `<branch>` in the reconciled **Active** table → the `sbx` name.
- **Miss?** Probe live sandboxes:
  `sbx exec <name> bash -lc 'cd <workspace>; git branch --show-current'` for each
  of this project's sandboxes. **Warn the user first: probing starts stopped
  sandboxes.**
- **Still a miss?** Report "no sandbox found for `<branch>`", print the Active
  table, and stop. Do not guess.

### 3. Safety gate — NEVER remove without checking for unpushed work
Removing a clone-mode sandbox destroys its private clone and any unpushed commits.
Before ANY `sbx rm`:
```bash
sbx exec <name> bash -lc 'cd <workspace>; git status -sb; git log --oneline --branches --not --remotes'
```
- **Clean** (no uncommitted changes AND no commits missing from a remote) → proceed.
- **Dirty** → **STOP. Do not remove.** Show the user exactly what is uncommitted /
  unpushed. The two flavors of dirty need different rescue — don't conflate them:
  - **Unpushed *commits*** → push them from inside the sandbox, or mirror them to
    the host with `git fetch sandbox-<name>` (into `refs/sandboxes/<name>/*`).
  - **Uncommitted / untracked files** (`git status` shows ` M` / `??` lines) →
    `git fetch` does **NOT** save these; only committed refs are mirrored. They must
    be committed (then pushed/fetched) or copied out of the sandbox first, or they
    are gone on removal.
  Proceed to removal only if the user passed `--force` this turn (which destroys
  whatever is still unsaved), or after the work above has been preserved.

**No exceptions:**
- Not because the invocation is literally "kill" — "kill" authorizes removing a
  *clean* sandbox; discovering unpushed work is new information that overrides it.
- Not with `sbx rm --force` unless the user explicitly passed `--force`. (Note:
  `sbx rm` may still need its own `--force` flag just to skip the interactive
  confirmation in a non-terminal shell — that is fine *after* this gate passes;
  it is not permission to skip the gate.)
- Don't assume "it was probably pushed" or "the PR merged" — verify with the
  command above.

### 4. Remove
`sbx rm <name>` (add the `sbx`-level `--force` only to skip the non-terminal
confirmation prompt, and only once step 3 passed).

### 5. Record (registry stays current)
Move the entry from **Active** → **Killed** with: `Killed` = today,
`Unpushed at kill?` = yes/no (from step 3), Notes (PR link / reason / "--force").
Re-confirm the header `Last reconciled:` date. Save the file.

### 6. Report
Tell the user what was killed, the safety-gate result, and that the registry was
updated. If dirty and you stopped, report the unpushed work and the options.

## Red flags — STOP and restart the flow
- About to `sbx rm` without having run the step-3 inspection **this turn**
- "They obviously want it gone" / "it was surely pushed"
- Editing the registry tables by hand instead of via reconcile + record
- Skipping step 1 because "I just want to kill one" (skipping reconcile is how the
  file goes stale — the exact failure this skill exists to prevent)

All of these mean: go back and run the flow in order.

## The registry file — `docs/sandbox/sandbox-history.md`
Host-maintained by this skill + docker-sandbox (use whatever path your project has
adopted if it differs from the default). Not hand-edited; not edited from inside a
sandbox (state is host-local). If missing, create it with this shape:

```markdown
# Sandbox history — <your-project>
> Registry of `sbx` Docker Sandboxes for this project. Maintained host-side by the
> docker-sandbox + kill-sandbox skills; reconciled against `sbx ls` every run.
> Host-local (sandboxes are per-machine). Last reconciled: <YYYY-MM-DD>.

## Active (not killed)
| Sandbox | Branch | Feature | Mode | Created | Last seen |
|---|---|---|---|---|---|

## Killed (history)
| Sandbox | Branch | Feature | Created | Killed | Unpushed at kill? | Notes |
|---|---|---|---|---|---|---|
```

## Quick reference (host)
```console
# resolve + safety-check + remove the sandbox on a branch, updating the registry:
/kill-sandbox feat/savings-goals
# same, but proceed even if the clone has unpushed work (destroys it):
/kill-sandbox feat/savings-goals --force

# the underlying sbx pieces (see docker-sandbox skill):
sbx ls
sbx exec <name> bash -lc 'cd <ws>; git status -sb; git log --oneline --branches --not --remotes'
sbx rm <name>
git fetch sandbox-<name>   # preserve committed-but-unpushed commits ONLY (not uncommitted/untracked files)
```

## Sources
- `.claude/skills/docker-sandbox/SKILL.md` — environment guard, gotchas
  (#2 command-sandbox disabled, #4 never blind-remove a clone sandbox), lifecycle.
- [`sbx` CLI reference](https://docs.docker.com/reference/cli/sbx/).

---
name: review
description: Review the current work — run the project's tests/checks, review the changed code for correctness and quality, fix any real issues found, then re-test and re-review until it is green and clean. A bounded verify → fix → re-verify loop over your uncommitted (or current-branch) changes, NOT a GitHub PR review. Use when the user asks to review the current work, check that a change is correct before committing, or "test and review and fix what's broken".
user-invocable: true
argument-hint: "[optional: scope — a path, 'staged', or a base branch like main]"
---

# review skill

Take the **current work** — the changes in the working tree (or the current
branch vs its base) — run the project's tests and checks, review the changed code,
**fix any real problems**, then **re-test and re-review**, looping until it is green
and clean. The deliverable is verified, reviewed work plus a short report of what was
found and fixed.

This is a **verify → fix → re-verify loop**, not a one-shot review. It is distinct
from a GitHub PR review (the built-in `review`) and from a read-only diff review
(`code-review`): this skill actually **runs the checks and fixes what it finds**,
scoped to the current change.

## Guardrails — read before doing anything

The whole risk of an auto-fix loop is that it "succeeds" dishonestly or never stops.
These are non-negotiable:

- **Never make a check pass by weakening the check.** Do not delete, skip, `xfail`,
  `.only`, comment out, or loosen a test/assertion; do not lower a coverage/lint
  threshold; do not swallow an error (empty `catch`, bare `except: pass`); do not
  hardcode a function to return the expected value; do not add or alter a
  mock/stub/monkeypatch so the assertion stops exercising the real code under test;
  and **do not edit a test to match the code's current (buggy) output** — don't change
  an assertion's expected value, a golden/snapshot file, or a fixture to whatever the
  code happens to produce. When code and test disagree, decide which is actually
  correct and fix that one; changing the expectation is allowed only if you can
  independently justify the new value as the correct behavior. If a test is genuinely
  wrong, say so and explain why — don't silently neuter it.
- **Never discard the user's work with a destructive git command.** No `git reset
  --hard`, `git checkout -- <path>` / `git checkout .`, `git clean`, `git stash` /
  `stash drop`, or branch reset — under ANY circumstance, including reverting a failed
  fix or "resetting to a clean state." The default scope is *uncommitted* work with no
  backup; it is the deliverable, not disposable. To undo a specific edit you made this
  session, re-edit that file — never blow away the working tree.
- **Fix causes, not symptoms.** A failing test means the code (or, rarely, the test)
  is wrong — diagnose which. Do not paper over failures.
- **Stay in scope.** Fix issues in (or directly caused by) the current change. Don't
  refactor unrelated code or fix pre-existing failures that the change didn't
  introduce — surface those separately instead. (If a fix legitimately must touch a
  file outside the original changeset, that edit is in scope — re-review it — but don't
  expand into unrelated refactors of that file.)
- **The loop is bounded.** At most **3 fix attempts total** (the first fix counts as
  attempt 1); after the 3rd, stop and report regardless of state. Stop earlier if an
  attempt makes no progress (same failures/findings as the previous one) — repeating
  won't help.
- **Don't commit, push, or open a PR** unless the user asked you to. This skill
  verifies work; it does not publish it.

## The loop — run in order

### 0. Scope the work — what is "current work"?
Determine the changeset before anything else:
```bash
git status --short
git diff HEAD --stat           # staged + unstaged vs HEAD — the full working-tree diff
git diff --stat --staged       # staged only
git diff --stat                # unstaged only (the split, if you need it)
```
Mind the `git diff` semantics — a wrong scope silently skips changes: plain `git diff`
shows **unstaged only**, `git diff HEAD` shows **staged + unstaged**, `git diff
--staged` shows **staged only**, and `git diff <base>...HEAD` is the **branch's
committed diff** vs the merge-base.

- **Default scope = the full working-tree diff** (`git diff HEAD` — staged + unstaged).
- If the argument is a **path**, scope to changes under it. If it's **`staged`**, use
  `git diff --staged`. If it's a **base branch** (e.g. `main`), review the **union** of
  `git diff <base>...HEAD` (the branch's committed changes vs the merge-base) **and**
  `git diff HEAD` (any uncommitted working-tree changes).
- **Nothing changed?** If the tree is clean and no base was given, say so and stop —
  there is no "current work" to review. (If the user clearly meant the last commit,
  offer `git show HEAD` scope instead.)

Read the actual diff (`git diff HEAD` for the default scope, or `git diff <scope>` for
a path / `--staged` / `<base>...HEAD`) so the review in step 3 is grounded in what
changed, not a guess.

### 1. Discover the project's checks
Find how this project tests/validates, in this order — **don't hardcode**:
1. The project's **agent-guidance file** (`CLAUDE.md` / `AGENTS.md`) and
   `CONTRIBUTING` — many state the exact commands and which are CI gates. Prefer these.
2. **Manifest scripts / task runners**: `package.json` `scripts` (`test`, `lint`,
   `typecheck`, `build`), a `Makefile`, `pyproject.toml` / `tox.ini`, `Cargo.toml`,
   `go.mod`, `justfile`, etc.
3. **Ecosystem fallback** if nothing is declared: e.g. `npm test` / `pnpm test` /
   `yarn test` + `tsc --noEmit` + lint (JS/TS); `pytest` / `ruff` / `mypy` (Python);
   `cargo test` / `cargo clippy` (Rust); `go test ./...` / `go vet` (Go).

Prefer the **narrowest checks that cover the change** for the inner loop (e.g. the
test file/package touched, plus typecheck/lint), then a **full run before declaring
done**. Prefer `test`/`typecheck`/`lint` in the inner loop; run `build` only if it's a
declared CI gate (build scripts can have side effects — codegen, clean). **In a
monorepo/workspace**, run checks from the package/app dir that owns the changed files
(the nearest enclosing manifest to the diff), or via a declared root task runner
(turbo/nx/workspace script) — not blindly from the repo root. If you cannot find any
check, say so and review statically — don't invent a command that might do harm.

### 2. Run the checks
Run the discovered tests + typecheck/lint/build, capturing output. Record each as
**pass / fail (with the failure)**. **If step 1 found no runnable check, don't invent
one** — report that and fall back to static review.

Do **not** label a failure "flaky" or "environmental" just to avoid fixing it — that's
an unfalsifiable escape hatch that defeats the whole loop. Before dismissing any
failure, **prove it**: re-run it (a genuine flake passes on retry with no code change;
a real bug fails deterministically) **and** show it's independent of the change (it
fails on a clean checkout without your diff, or the touched code can't reach it). If a
failure needs a service that isn't running, prefer the project's documented setup
command to start it; only mark it environmental when no such command is declared or it
genuinely can't run here (e.g. no Docker in this sandbox) — and say what setup it needs.
If you can't show both flake criteria, treat it as a real failure.

### 3. Review the changed code
Review the **diff from step 0** (not the whole repo), in two passes:
- **Correctness (must-fix):** logic errors, off-by-one, wrong conditionals, unhandled
  nulls/errors, race conditions, resource leaks, security issues (injection, secrets
  in code, unsafe input), broken edge cases, and mismatches between the change and its
  apparent intent. Also: does the change actually have test coverage? A correct-looking
  change with no test for its new behavior is a finding.
- **Quality (fix if safe & in-scope):** duplication that should reuse existing code,
  needless complexity, dead code, unclear names, violations of patterns the surrounding
  code follows. Apply the high-confidence ones; list the judgment-call ones for the user
  rather than forcing them.

### 4. Fix the real issues
Fix the union of **check failures (step 2)** and **must-fix findings (step 3)**, plus
the safe in-scope quality findings. Keep fixes **minimal and consistent with the
surrounding code**. If a fix needs a decision only the user can make (an ambiguous
requirement, a breaking API change, a "which behavior is correct" question), **stop
and ask** rather than guessing. If a proper fix would balloon in scope, note it and
leave it for the user instead of a hacky patch.

### 5. Re-test and re-review
Re-run the checks (step 2) and re-review anything you touched (step 3) — your fix may
have introduced a new issue or broken a sibling. Then:
- **Green + clean** (checks pass AND no remaining must-fix findings) → go to step 6.
- **Still failing / new findings** → loop back to step 4, **but respect the bound**:
  the 3-attempt total from the Guardrails, and stop immediately if an attempt didn't
  reduce the problem (same failures/findings as last time). Before a final full run,
  run the whole check suite (not just the narrowed inner-loop checks) so nothing
  regressed elsewhere.

### 6. Report
Tell the user, concisely:
- **Scope** reviewed (the changeset).
- **Checks** run and their final status (green, or what's still red and why).
- **Found & fixed** — the issues and the fix for each (correctness, then quality).
- **Left for you** — anything not fixed: needs-a-decision items, out-of-scope /
  pre-existing failures, flaky/environmental issues, judgment-call quality notes, or
  "hit the 3-pass bound without converging — here's what's stuck."
- **Final state** — "green and clean" or an honest "not converged." Do **not** claim
  success if any check is red or any must-fix finding remains.

Do not commit or push unless the user asks. If they do, follow the project's commit
conventions.

## Stop and ask / stop and report — don't push through
- A fix requires a product/requirements decision, or "which behavior is correct?" is
  genuinely ambiguous → **ask**.
- The same check keeps failing after a fix attempt (no progress) → **stop, report**
  what's stuck; don't thrash.
- The only way you can see to make a check pass is to weaken it → **stop, report** —
  that's a signal the change or the test is actually wrong.
- A failure is pre-existing / unrelated to the current change → **report separately**,
  don't silently adopt it into the change.
- 3 fix attempts reached without converging → **stop, report** the remaining state.

## What NOT to do
- Don't game the checks (skip/delete/loosen tests, edit expected values to match buggy
  output, ignore errors, hardcode outputs, mock away the real code path, lower
  thresholds) to force green — the top failure mode of this skill.
- Don't discard the user's uncommitted work with a destructive git command
  (`reset --hard`, `checkout .`, `clean`, `stash`) — ever.
- Don't review the entire repository — review the **current change**.
- Don't fix unrelated or pre-existing issues as if they were part of this work.
- Don't loop forever — honor the 3-pass bound and the no-progress stop.
- Don't commit, push, or open a PR unprompted.
- Don't report "all good" while anything is red or unresolved. Be honest about what's
  left.

## Relationship to other review tools
- The built-in **`review`** reviews a published **GitHub PR** (already-shipped work).
- The built-in **`code-review`** does a **read-only** review of your working diff — it
  finds issues but does not change code.
- This skill is the third mode: it **runs the checks, applies fixes, and re-verifies**
  the local change. Use it when you want the current work actually tested, corrected,
  and re-checked before it goes anywhere.

> **Naming:** this skill deliberately reuses the name `review`. Installed at
> `.claude/skills/review/`, it **shadows the built-in `/review`** in that project, so
> `/review` will mean "test-and-fix my local work," not "review a PR." If you want both,
> install this under a different folder name (e.g. `verify-work`) — the `name:` in the
> frontmatter and the folder should match whatever you choose.

---
name: docker-sandbox
description: Start and manage Docker Sandboxes (the `sbx` CLI) — one isolated microVM per feature for parallel or overnight agent work on a repo. Covers sbx run/create/ls/stop/rm, running multiple named sandboxes for the SAME repo, clone vs branch vs direct mode, daemon/container isolation (nothing is shared between sandboxes), reaching shared services via host.docker.internal + network policy, backing services in multi-sandbox setups, git push/PR from a sandbox, and bootstrapping a fresh sandbox so it has everything to get going (GitHub secret, agent login, gitignored env files, network policy). Use when the user wants to spin up sandboxes, run agents in parallel, set up an overnight multi-feature workflow, or asks whether a new sandbox has the credentials/keys it needs.
user-invocable: true
argument-hint: "[optional: feature name(s) to spin up sandboxes for, e.g. 'auth checkout analytics']"
---

# docker-sandbox skill

Spin up and manage **Docker Sandboxes** — isolated microVMs (own filesystem,
Docker daemon, and network) that run a coding agent against your repo. The main
use is **parallel / overnight feature work**: one named sandbox per feature, each
on its own branch, with no chance of two agents clobbering each other's edits.

Reference: Docker Sandboxes docs — https://docs.docker.com/ai/sandboxes/ and the
`sbx` CLI reference — https://docs.docker.com/reference/cli/sbx/. When a flag isn't
covered here, `sbx <cmd> --help` on the host is the source of truth.

Companion skill: **kill-sandbox** (`.claude/skills/kill-sandbox/SKILL.md`) tears a
sandbox down by branch and keeps the registry current. This skill and kill-sandbox
share the same registry file.

**Placeholders used below** (fill in for your project): `<repo>` = your repo's
directory name, used literally in paths and sandbox ids (`~/dev/<repo>`,
`claude-<repo>`, `<repo>/.sbx/`); `<your-project>` = its human display name, used in
prose; `<app-dir>` = the app subdir if any (e.g. `web/`, `apps/api/`), else the repo
root; `<env-file>` = the gitignored env file your app reads (e.g. `.env.local`);
`<registry-file>` = the sandbox registry (default `docs/sandbox/sandbox-history.md`).

## ⚠️ Read this first: `sbx` runs on the HOST, not inside a sandbox

A Claude Code session (like this one) usually runs **inside** a sandbox, where the
`sbx` binary is not available. **You cannot create sandboxes from inside one.** So
this skill's job is to hand the user the exact commands to run in their **host
terminal** (macOS/Windows/Linux), plus the workflow and the gotchas. Present the
commands in a copyable block and tell the user to run them on the host. Do not try
to execute `sbx ...` from a Bash tool call inside the sandbox — it will fail.

### Environment guard — do this FIRST, every invocation

Before emitting anything, detect whether you are inside a sandbox:

```bash
command -v sbx >/dev/null 2>&1 && echo "sbx: present" || echo "sbx: ABSENT"
echo "SANDBOX_VM_ID=${SANDBOX_VM_ID:-<unset>}"
```

- **Inside a sandbox** — `sbx` is **ABSENT**, or `SANDBOX_VM_ID` is set (e.g.
  `claude-<repo>`). You are a passenger: **do NOT run any `sbx …` command.** Prepend
  this banner, verbatim (substituting the real id), to every command block you emit:

  > 🖥️ **Run these on your HOST terminal — not here.** This Claude Code session is
  > inside sandbox `<SANDBOX_VM_ID>`, where `sbx` isn't installed, so it cannot
  > create sandboxes (and would only spawn *sibling* VMs on the host, never a nested
  > one). Copy the commands below into your host shell.

- **On the host** — `sbx` is **present** and `SANDBOX_VM_ID` is unset. You *may* run
  the commands directly, but creating/removing sandboxes is consequential: show the
  block and get an explicit go-ahead first, and **never auto-run `sbx rm`**.

(The one-time install is host-side too: `brew install docker/tap/sbx` on macOS,
`winget install -h Docker.sbx` on Windows, the apt path on Ubuntu 24.04+, then
`sbx login`. Pick the **Balanced** network policy at login.)

## When invoked

0. **Guard: detect the environment (always first).** Run the check in "Environment
   guard" above. If inside a sandbox (`sbx` absent / `SANDBOX_VM_ID` set), switch to
   "emit for host" mode and **prepend the HOST-ONLY banner** to any command block —
   do not execute `sbx`. If on the host, you may offer to run the commands after an
   explicit go-ahead.
1. If the user named one or more features (via arguments or the message), emit the
   ready-to-run block that creates **one clone-mode, uniquely-named sandbox per
   feature** for this repo (recipe below), then tell them to run it on the host.
   For each feature, also produce a **per-feature handoff** (enter + bootstrap +
   paste-ready agent prompt) using the **Feature handoff template** below — every
   prompt carries **your project's required workflow contract**.
2. If they just want "a sandbox," give the single-sandbox command.
3. If they're asking a conceptual question (can I run several? do they share
   containers? how do I reconnect?), answer from the "Facts" section below.
4. Always surface the two load-bearing caveats: **unique `--name` per sandbox**, and
   **push before you `sbx rm`** (removal deletes a clone-mode sandbox's private clone).

## Can I run multiple sandboxes for the same repo? — Yes

`--name` identifies a sandbox **independent of the working directory**, so several
independently-named sandboxes can target the same workspace. Without `--name`,
re-running `sbx run` in the same path just **re-attaches** to the existing sandbox
instead of making a new one. So the rule is simple: **one unique `--name` per
concurrent sandbox.**

```console
# From the repo root — two sandboxes, same project, different names:
$ sbx run claude --name feature ~/dev/<repo>
$ sbx run claude --name spike   ~/dev/<repo>
```

## Modes: direct vs clone vs branch (pick per how isolated you need to be)

| Mode | Flag | Working tree | Use it when |
|---|---|---|---|
| **direct** | *(default)* | your real host repo dir, mounted **read-write** | a single agent, or read-only/clearly-separate-file work |
| **clone** | `--clone` | an **in-container git clone** of the host repo (wired back via a git-daemon) — fully isolated | **parallel agents** that both edit code; overnight fan-out |
| **branch** | `--branch` | a git **worktree** under `<repo>/.sbx/` sharing the host's `.git` object DB | if your project already standardizes on worktrees (check your project's agent-guidance file) |

- **Avoid multiple `direct`-mode sandboxes on one repo** — they all mount the same
  files, so two agents will overwrite each other. Use `--clone` (or `--branch`) for
  concurrency.
- **Clone mode is fixed at create time.** To switch an existing sandbox to clone
  mode, `sbx rm` it and recreate with `sbx create --clone`.
- **Clone mode is rejected from inside a Git worktree other than the main one** —
  run `sbx create --clone` from the primary checkout, not from a `.sbx/…` worktree.

## Recommended workflow — one named clone-mode sandbox per feature

```console
# On the HOST, from your repo root (or pass the path as the last arg):
$ cd ~/dev/<repo>

# Create one isolated sandbox per feature (backgrounded), unique names:
$ sbx create --clone --name auth-feature      claude .
$ sbx create --clone --name checkout-feature  claude .
$ sbx create --clone --name analytics-feature claude .

# See them all:
$ sbx ls

# Enter one (agent optional once it exists — read from the sandbox spec):
$ sbx run --name auth-feature
```

Or create-and-enter in one step: `sbx run --clone claude --name auth-feature .`

**Inside each sandbox** the agent must branch before editing (clone mode checks out
whatever ref the host had checked out; it does **not** auto-create a branch). Don't
hand it a bare "implement X" line — use the **Feature handoff template** below, which
carries **your project's required workflow contract**. One branch per sandbox keeps
PRs clean: `auth-feature → feat/auth`, `checkout-feature → feat/checkout`.

### Record it in the registry
Keep a **sandbox registry** current — the project's list of active + killed
sandboxes, and the branch↔`sbx`-name map the **kill-sandbox** skill relies on. The
recommended default path is `docs/sandbox/sandbox-history.md` (project-configurable;
if you change it, point kill-sandbox at the same file). On **create**, append an
**Active** row (sandbox, branch, feature, mode, created = `date +%F`, last seen =
running). To tear one down by branch and record it, use the **kill-sandbox** skill
(`.claude/skills/kill-sandbox/SKILL.md`) — it removes the sandbox after a safety
check for unpushed work, then moves the row to **Killed**. Both skills reconcile the
file against `sbx ls` every run so it self-heals — write-through on create just keeps
the branch/feature/created columns accurate, which reconcile can't infer.

## First-time project setup — build the two helper scripts

This portable skill does **not** ship helper scripts, because a productive bootstrap
depends on your stack (package manager, local backing services, env-file layout).
Build these two scripts **once for your project**, commit them at the paths below,
and the rest of this skill references them from there:

- `.claude/skills/docker-sandbox/bootstrap-sandbox.sh`
- `.claude/skills/docker-sandbox/set-github-secret.sh`

### (a) `bootstrap-sandbox.sh` — make a fresh clone/branch sandbox productive

A fresh `--clone`/`--branch` sandbox has the tracked source but **not** the repo's
gitignored env files, and no branch of its own. This script closes that gap. Build
spec — it MUST:

0. **Preflight.** Verify the CLIs your steps depend on are present up front
   (`for bin in <node npm curl …>; do command -v "$bin" || { echo "missing $bin"; exit 1; }`).
   If a needed CLI is missing from the minimal base image, self-install it (see the
   self-install note after the skeleton) rather than failing.
1. **Run from the WRITABLE workspace clone.** Refuse if the checkout is the read-only
   `/run/sandbox/source` mount (probe by trying to create a temp file).
2. **Host-guard.** It rewrites gitignored env file(s), so refuse to run on the host
   (`sbx` present *and* `SANDBOX_VM_ID` unset) unless `--force` is passed.
3. **(Optional) create + switch to a feature branch** from a branch-name argument —
   clone mode does **not** auto-branch. Omitting the arg skips branching.
4. **Install dependencies** with your project's package manager (e.g. `npm ci`,
   `pnpm i --frozen-lockfile`, `yarn install --immutable`, `pip install -r …`, `uv
   sync`, `bundle install`, `go mod download`, …).
5. **Start whatever LOCAL backing services the project needs**, then **write the
   project's gitignored env file(s)** from those services. **Adapt to your stack:
   Postgres, Docker Compose, a hosted URL, or none.** Get these details right — they
   are what makes the difference between a working env file and a clobbered one:
   - Make the **start step idempotent** — a re-run where services are already up must
     not abort under `set -e` (probe status as a fallback: `if ! start; then status || fail`).
   - **Write to a temp file, then validate** you extracted the expected keys (fail
     loudly and dump raw status if not) — never write a half-populated env file.
   - **Back up any existing env file** before overwriting, then **atomically `mv`**
     the temp file into place.
   - **Print a redacted confirmation** (values hidden — never echo secret values) and
     **assert the target points at the local stack** (warn if the URL looks hosted).

   Worked example (adapt — this is ONE stack, e.g. a project using a local Supabase
   stack): install the Supabase CLI if the image lacks it, start idempotently, then
   auto-extract keys with `supabase status -o env` into a temp file, validate, back
   up, and `mv` — no hand-copying:

   ```bash
   supabase start >/dev/null 2>&1 || supabase status >/dev/null 2>&1 || { echo "is Docker up?" >&2; exit 1; }
   tmp="$(mktemp)"
   supabase status -o env \
     --override-name api.url=NEXT_PUBLIC_SUPABASE_URL \
     --override-name auth.anon_key=NEXT_PUBLIC_SUPABASE_ANON_KEY \
     --override-name auth.service_role_key=SUPABASE_SERVICE_ROLE_KEY 2>/dev/null \
     | grep -E '^(NEXT_PUBLIC_SUPABASE_URL|NEXT_PUBLIC_SUPABASE_ANON_KEY|SUPABASE_SERVICE_ROLE_KEY)=' > "$tmp" || true
   [ "$(grep -c . "$tmp")" -ge 3 ] || { echo "error: missing keys" >&2; supabase status >&2; exit 1; }
   [ -f <env-file> ] && cp <env-file> "<env-file>.bak.$$"
   mv "$tmp" <env-file>
   sed -E 's/=(.*)$/=<hidden>/' <env-file>   # redacted confirmation
   ```

Portable annotated skeleton — fill in the four project-specific steps (the exact
commands are yours):

```bash
#!/usr/bin/env bash
# Bootstrap a fresh Docker Sandbox to build/run/test <your-project>. Run INSIDE the
# sandbox, from the WRITABLE workspace clone (NOT /run/sandbox/source). Idempotent.
# Usage: ./.claude/skills/docker-sandbox/bootstrap-sandbox.sh [--force] [branch-name]
set -euo pipefail

force=0
if [ "${1:-}" = "--force" ]; then force=1; shift; fi
branch="${1:-}"

repo_root="$(cd "$(dirname "${BASH_SOURCE[0]}")/../../.." && pwd)"   # adjust ../ depth to your layout
cd "$repo_root"

# Guard 1 — READ-ONLY mount (the /run/sandbox/source trap). The writable private
# clone is the path you land in on `sbx run`; /run/sandbox/source is read-only.
writetest=".sbx-writetest.$$"
if ! ( : > "$writetest" ) 2>/dev/null; then
  echo "error: this checkout is READ-ONLY ($repo_root)." >&2
  echo "  Run from your sandbox's WRITABLE workspace clone (the 'Workspace' path" >&2
  echo "  sbx printed on 'sbx run'), NOT /run/sandbox/source." >&2
  exit 1
fi
rm -f "$writetest"

# Guard 2 — host guard: this rewrites gitignored env file(s); don't clobber the
# host's real ones by accident.
if [ "$force" -ne 1 ] && [ -z "${SANDBOX_VM_ID:-}" ] && command -v sbx >/dev/null 2>&1; then
  echo "This looks like the HOST (sbx present, SANDBOX_VM_ID unset)." >&2
  echo "bootstrap-sandbox.sh is meant to run INSIDE a sandbox. Re-run with --force to override." >&2
  exit 1
fi

# Guard 0 — preflight: fail early if a required CLI is missing.
export PATH="$HOME/.local/bin:$PATH"   # so a self-installed CLI is found this run
for bin in git <node npm curl>; do   # <fill in the CLIs your steps below need>
  command -v "$bin" >/dev/null 2>&1 || { echo "error: '$bin' not found." >&2; exit 1; }
done

# --- PROJECT-SPECIFIC: ensure any CLI the base image lacks (self-install) -----
# The base image is minimal (typically Node + Docker). Install missing release
# binaries to ~/.local/bin (on PATH, no sudo), and persist PATH for later shells:
#   if ! command -v <cli> >/dev/null; then
#     curl -fsSL <release-url> | tar -xz -C "$HOME/.local/bin" ... ;
#     grep -qs 'HOME/.local/bin' /etc/sandbox-persistent.sh || \
#       echo 'export PATH="$HOME/.local/bin:$PATH"' | sudo tee -a /etc/sandbox-persistent.sh >/dev/null
#   fi
#   <fill in, or delete if the image has everything>

# Optional feature branch (clone mode checks out the host's ref; it does NOT branch).
if [ -n "$branch" ]; then
  git checkout -b "$branch" 2>/dev/null || git checkout "$branch"
  echo "==> on branch $(git branch --show-current)"
fi

# --- PROJECT-SPECIFIC: install dependencies ---------------------------------
# e.g. ( cd <app-dir> && npm ci )   |   uv sync   |   bundle install   | …
#   <fill in>

# --- PROJECT-SPECIFIC: start local backing services (IDEMPOTENTLY) -----------
# e.g. supabase start   |   docker compose up -d   |   (none) …
# Re-runs must not abort under set -e when services are already up:
#   if ! <start> >/dev/null 2>&1; then <status-probe> || { echo "is Docker up?" >&2; exit 1; }; fi
#   <fill in>

# --- PROJECT-SPECIFIC: write the gitignored env file(s) from those services --
# temp -> validate -> back up existing -> atomic mv -> redacted print -> assert local.
#   tmp="$(mktemp)"; <extract vars> > "$tmp"
#   [ "$(grep -c . "$tmp")" -ge <N> ] || { echo "error: missing keys" >&2; exit 1; }
#   [ -f <env-file> ] && cp <env-file> "<env-file>.bak.$$"
#   mv "$tmp" <env-file>
#   sed -E 's/=(.*)$/=<hidden>/' <env-file>            # never echo secret values
#   <fill in>

echo "✓ Sandbox bootstrapped. Next: run the test suite, then start the app."
```

The self-install + PATH handling above is load-bearing: without exporting
`~/.local/bin` onto PATH **within** the script (and persisting it to the sandbox
env file for later shells), a just-installed CLI raises "command not found" mid-run.
Alternatively bake the CLI into a
[Docker Sandbox kit](https://docs.docker.com/ai/sandboxes/customize/kits/) so every
sandbox has it preinstalled and you can drop the self-install step.

### (b) `set-github-secret.sh` — set the sbx GitHub push secret (HOST-only)

Fixes the recurring "I can't push from a new sandbox" blocker. Build spec — it MUST:
run on the **host** (`sbx` present); source the token in this precedence,
`$GITHUB_TOKEN` → `./.secrets` (gitignored) → `gh auth token`; support `-g` (global —
all future sandboxes) and `<sandbox-name>` (one existing sandbox, effective
immediately); and **never print the token**.

This one is nearly project-agnostic, so building it is mostly a copy step — create
`.claude/skills/docker-sandbox/set-github-secret.sh` with the reference
implementation below (the only thing to check for your layout is the `repo_root`
`../` depth):

```bash
#!/usr/bin/env bash
# Set the GitHub push credential for Docker Sandboxes — HOST-ONLY (uses `sbx`).
# Token read (in order): $GITHUB_TOKEN → ./.secrets (gitignored) → `gh auth token`.
# It's injected at the proxy — never lands on the sandbox FS, never printed here.
# Usage:
#   ./.claude/skills/docker-sandbox/set-github-secret.sh -g              # global: ALL future sandboxes
#   ./.claude/skills/docker-sandbox/set-github-secret.sh <sandbox-name>  # one existing sandbox
set -euo pipefail

target="${1:-}"
if [ -z "$target" ]; then
  echo "usage: $0 <sandbox-name> | -g" >&2
  exit 2
fi
if ! command -v sbx >/dev/null 2>&1; then
  echo "error: sbx not found. Run this on your HOST, not inside a sandbox." >&2
  exit 1
fi

repo_root="$(cd "$(dirname "${BASH_SOURCE[0]}")/../../.." && pwd)"   # adjust ../ depth to your layout

token="${GITHUB_TOKEN:-}"
if [ -z "$token" ] && [ -f "$repo_root/.secrets" ]; then
  # shellcheck disable=SC1091
  set -a; . "$repo_root/.secrets"; set +a
  token="${GITHUB_TOKEN:-${GH_TOKEN:-}}"
fi
if [ -z "$token" ] && command -v gh >/dev/null 2>&1; then
  token="$(gh auth token 2>/dev/null || true)"
fi
if [ -z "$token" ]; then
  echo "error: no GitHub token. Set GITHUB_TOKEN, add it to .secrets, or run 'gh auth login'." >&2
  exit 1
fi

case "$target" in
  -g|--global)
    sbx secret set -g github -t "$token"
    echo "✓ Global GitHub secret set — every FUTURE sandbox inherits push access." ;;
  *)
    sbx secret set "$target" github -t "$token"
    echo "✓ GitHub secret set for sandbox '$target' — push works now (no restart needed)." ;;
esac
```

## Feature handoff template

Once a feature sandbox exists, hand the user **one block per sandbox** to paste
straight into their host terminal and the in-sandbox agent. A bare "implement X"
prompt is not enough — it drops the workflow your project requires. Every block has
the same required parts, in this order — do not omit any:

1. **Enter + bootstrap** — the two commands (host `sbx run`, then in-sandbox
   bootstrap on the branch).
2. **A paste-ready agent prompt** containing, in order:
   a. the branch line ("You're on branch `feat/<name>` …"),
   b. the **Workflow (required)** contract — **insert your project's required
      workflow contract here** (verbatim, identical in every prompt),
   c. the feature body (what to build; the tables/RPCs/UI to touch; any "reuse
      existing X" notes),
   d. the docs to read first,
   e. the closing line — commit, push to the branch, open a PR; work only in the
      writable clone (not `/run/sandbox/source`); push before the sandbox is removed.

The **Workflow (required)** contract should be **whatever your project's
agent-guidance file (CLAUDE.md / AGENTS.md) or CONTRIBUTING mandates** — e.g.
spec-first, TDD, a review gate, whatever it is. Make it non-negotiable and identical
in every prompt. One example a project *could* use (adapt or replace entirely):

> **Workflow (required):** Promote this backlog item into a real spec via the
> project's spec-first flow, then work **fully TDD**: for every unit, write a
> **failing test first (RED)**, implement to green (GREEN), then refactor — never
> write implementation code before a failing test exists.

### Fill-in template

```console
# [host]
$ sbx run --name <sandbox>
# [in-sandbox]  (build it first — see "First-time project setup")
$ ./.claude/skills/docker-sandbox/bootstrap-sandbox.sh feat/<branch>
```
> You're on branch `feat/<branch>` in <your-project>.
>
> **Workflow (required):** …[your project's contract, verbatim]…
>
> **Feature — <short title>** (`<docs path>`): <what to build; tables/RPCs/UI to
> touch; any "reuse existing X" notes>.
>
> Read <the relevant docs> first. When done, commit, push to `feat/<branch>`, and
> open a PR. Work only in the writable clone (not `/run/sandbox/source`), and push
> before this sandbox is ever removed.

### Worked example (generic placeholder — swap in your real feature)

```console
# [host]
$ sbx run --name widget-export
# [in-sandbox]
$ ./.claude/skills/docker-sandbox/bootstrap-sandbox.sh feat/widget-export
```
> You're on branch `feat/widget-export` in <your-project>.
>
> **Workflow (required):** …[your project's contract, verbatim]…
>
> **Feature — CSV export for widgets** (`docs/<area>.md`): the `exportWidgets()`
> helper in `<app-dir>/lib/export.ts` is already built and unit-tested but imported
> by no UI. Wire it into a single "Export to CSV" button on the widgets list, scoped
> to the current filter. **No new DB migration** — reuse the existing helper. Write
> the button/component test first (RED), then wire to green.
>
> Read `docs/index.md` and `docs/<area>.md` first. When done, commit, push to
> `feat/widget-export`, and open a PR. Work only in the writable clone (not
> `/run/sandbox/source`), and push before this sandbox is ever removed.

### Parallel-collision caveat (fan-out of ≥2 features)

If your project has **monotonic counters** that every new feature increments off the
same base branch, parallel agents collide on them — flag this when handing off a
batch. Common examples:
- **Sequential spec/dir numbers** (e.g. `specs/NNN-…/`) — each agent grabs the same
  next number.
- **DB migration indexes** — each migration-adding feature grabs the same next index.

Neither blocks parallel *building*; they collide at *merge*. Merge one branch at a
time and **renumber** the collided artifact on each later branch. Merge collision-
free features (e.g. a pure-frontend one) first — least friction.

## Bootstrapping a fresh sandbox (make it productive)

A fresh `--clone` or `--branch` sandbox has the tracked source but **not** the repo's
gitignored secrets (env files, local-credential notes, …), and its agent isn't
logged in yet. So "does a new sandbox have everything to get going?" → **not
automatically.** Run this checklist. Steps are tagged **[host]** (your host shell,
via `sbx`) or **[in-sandbox]** (inside the sandbox).

### One-time host setup — so every FUTURE sandbox inherits it
```console
# [host] GitHub push/PR for all future sandboxes (the proxy injects the creds, so
#        `gh auth status` still shows "not logged in" inside — that's expected):
$ sbx secret set -g github -t "$(gh auth token)"
# [host] allow project domains the default "Balanced" policy may block — only if the
#        agent reaches them (e.g. a hosted DB/API host your app talks to):
$ sbx policy allow network -g <your-service-host>
```
Claude agent auth is **per sandbox**: run `/login` inside each, or store an API key
as a secret so the `claude` agent starts authenticated.

### Per-sandbox bootstrap — one command (the gitignored bits git won't clone)

A fresh clone/branch sandbox has the source but not the gitignored env file(s). Run
the project's bootstrap script (build it first — see "First-time project setup"):

```console
# [in-sandbox] from the WRITABLE workspace clone (not /run/sandbox/source):
$ ./.claude/skills/docker-sandbox/bootstrap-sandbox.sh feat/<your-branch>
#   → installs deps; git checkout -b feat/<your-branch>; starts this sandbox's LOCAL
#     backing services; writes the gitignored env file(s) pointing at them.
#     Nothing hosted/shared is touched.
```
Omit the branch arg to skip branching. Guards: it **refuses to run on the read-only
`/run/sandbox/source` mount** (run it from the writable clone — the dir you land in
on `sbx run`), **refuses to run on the host** (unless `--force`) so it can't clobber
your real env file, and backs up any existing one before writing.

> If a sandbox was created **before** the script existed on your default branch, its
> clone has the old copy — refresh it first:
> `git fetch origin main && git checkout origin/main -- .claude/skills/docker-sandbox/bootstrap-sandbox.sh`.

**Prefer a shared hosted backend instead?** (concurrent agents then share one DB —
see "Backing services in a multi-sandbox setup" for the tradeoffs; prefer a per-agent
schema): skip the script's service step and set the env file to the hosted URL + keys
from your host's env file or local-credential notes.

### Verify it's ready
```console
# [in-sandbox]
$ <run your test suite>               # suite runs (no network needed if fully local)
$ git push -u origin <branch>         # succeeds via the injected GitHub creds
```
If `git push` prompts for a username, the github secret isn't set for this sandbox —
run `sbx secret set <sandbox-name> github -t "$(gh auth token)"` on the host (the
per-sandbox form takes effect immediately; the `-g` global form applies to newly
created sandboxes).

## What is and isn't shared between sandboxes

Each sandbox is **fully isolated**: its own Docker daemon, network, containers,
images, build cache, and named volumes. **Containers started in sandbox A are
invisible to sandbox B**, and sandboxes can't talk over a shared sandbox network.
Running the same `docker compose` in two sandboxes launches **two independent
copies** of every service.

They *can* share: the host source (in direct mode), the same Compose files, and
**external services reachable over the network** (a hosted DB, an API).

### Sharing a service across sandboxes
Run the shared service on the **host** Docker daemon and reach it from each sandbox
via `host.docker.internal:<port>` (sandboxes can't use the host's `localhost`).
The **network policy must allow that host:port** — see your project's agent-guidance
file ("Accessing services on the host" + `sbx policy allow`).

```console
# host: shared infra
$ docker compose up -d postgres redis
# in each sandbox's app config: host.docker.internal:5432 / :6379
```

## Backing services in a multi-sandbox setup

If your project's local backing services run in Docker (e.g. a Postgres container, a
`docker compose` stack, or a local Supabase stack via `supabase start`), they are
**per sandbox** — a stack started in one sandbox is invisible to the others. Three
options for parallel agents:

1. **Own local stack per sandbox** — each sandbox starts its own (ports are internal
   to each sandbox, so no collision). Isolated but heaviest; run migrations per DB.
2. **Shared hosted backend** — point every sandbox's env file at the hosted URL.
   Simplest, but concurrent agents share one DB → **watch for conflicting migrations
   / shared data**. Prefer separate schemas or DBs per agent.
3. **Shared service on host Docker** — reach it via `host.docker.internal:<port>`
   with a network-policy allow. Shared but outside the sandboxes.

For migration-heavy parallel work, isolate the database per sandbox (option 1 or a
per-sandbox schema). If your project has a seed/fixture CLI that only writes to a
local target by design, it's safe to run inside any sandbox pointed at that sandbox's
own local stack.

## Fix push once — the "can't push from a new sandbox" blocker

A fresh sandbox can *read* a public repo anonymously but **can't `git push`** until a
GitHub token secret is set for it — the #1 recurring blocker. The proxy injects the
credential at the network layer once the secret exists (so `gh auth status` still
shows "not logged in" inside — that's expected, not a failure).

**Do it once, globally — every future sandbox inherits it (recommended):**
```console
# [host]
$ sbx secret set -g github -t "$(gh auth token)"
# …or the project helper (same thing, with a .secrets/gh fallback — build it first,
#   see "First-time project setup"):
$ ./.claude/skills/docker-sandbox/set-github-secret.sh -g
```

**Unblock a sandbox that's already running** (takes effect immediately, no restart):
```console
# [host]
$ sbx secret set <sandbox-name> github -t "$(gh auth token)"
$ ./.claude/skills/docker-sandbox/set-github-secret.sh <sandbox-name>   # same, via helper
```

**Optional host `.secrets` file** — for scripted/overnight fleets that don't want to
call `gh` each time. Copy a `.secrets.example` → `.secrets` (gitignored — **never
commit**; a leaked push token compromises the repo) and set `GITHUB_TOKEN=`; the
helper reads it. Two things to know:
- It's **host-side only.** A gitignored file is **not** in a clone/branch sandbox
  (same as your env file), so a sandbox can't read `.secrets` itself — the host sets
  the `sbx` secret from it, and only the proxy-injected credential reaches the
  sandbox. (This is why the file can't be "read from inside the sandbox.")
- Prefer the **global secret** above when you can — it's simpler and keeps no token
  in the working tree.

## Git push / PR from a sandbox

Pushing and opening PRs from a sandbox is the expected end of each feature. Your
project's agent-guidance file has the full guide — key points:
- Git auth is injected by the sandbox proxy; `gh auth status` showing "not logged
  in" is normal and does **not** mean push will fail.
- If `git push` fails with a username prompt, the host needs a GitHub token secret:
  `sbx secret set <sandbox-name> github -t "$(gh auth token)"` (or `-g` globally).
- For a `--branch`-mode sandbox whose `origin` points at the local sandbox source,
  add the real GitHub remote before pushing.

## Gotchas & safe procedures (learned the hard way)

These are the failure modes that actually bite when running this skill. Check them.

**1. Two dirs inside a clone sandbox — one is READ-ONLY.**
`/run/sandbox/source` is a **read-only** mount of your host repo (virtiofs `ro`). The
sandbox's **writable private clone** is at the workspace path (mirrors the host path
*inside the container*) — that's where you land on `sbx run`. **Always work in the
writable clone.** Reading `/run/sandbox/source` shows the *host's* files (including a
gitignored env file that isn't really in the clone) and writes there fail. The
bootstrap script refuses to run on the RO mount for this reason.

**2. `sbx ls` fails with "database already in use" / `ps`/`pgrep` say "operation not
permitted".** That's **Claude Code's own Bash command-sandbox (seatbelt)** blocking the
`sbx` daemon socket — not a real daemon conflict. `sbx` then tries to start a second
daemon and collides on its DB lock. Fix: run the `sbx` command with the command
sandbox **disabled** (re-run outside the restricted Bash). It's an environment quirk,
not an `sbx` bug.

**3. Check the name is free BEFORE `sbx create`.** `sbx ls` first. Creating with a name
that already exists **re-attaches** to the existing sandbox instead of making a fresh
one — easy to mistake for "it worked."

**4. NEVER blind `sbx rm --force` a clone-mode sandbox.** Removal deletes its private
clone and any unpushed commits. `--force` **skips the safety fetch**. Before removing:
- Inspect the **writable clone** (not `/run/sandbox/source`) for unpushed work:
  ```console
  # [host] find the workspace path from `sbx ls`, then:
  $ sbx exec <name> bash -lc 'cd <workspace-clone>; git status -sb; git log --oneline --branches --not --remotes'
  ```
- If there's anything, push it from inside first, or run the **preservation fetch**
  `sbx rm` prints (it mirrors the sandbox's commits into `refs/sandboxes/<name>/*` on
  your host). Only then `sbx rm <name>`.

**5. The base sandbox image is minimal** (typically Node + Docker, little else). If
your bootstrap needs a CLI that isn't there, install the release binary to
`~/.local/bin`, or bake it into a
[Docker Sandbox kit](https://docs.docker.com/ai/sandboxes/customize/kits/) so every
sandbox has it preinstalled.

## Lifecycle & cleanup

```console
$ sbx ls                     # list sandboxes (one entry per sandbox, even with N worktrees)
$ sbx run --name auth-feature   # reconnect from any directory
$ sbx stop auth-feature      # stop but preserve state
$ sbx rm auth-feature        # remove permanently (clone-mode: deletes its private clone!)
$ sbx rm --force auth-feature   # force-remove during an active session
```

**Push or fetch anything you need BEFORE `sbx rm`** — removing a clone-mode sandbox
deletes its private clone and any unpushed commits with it.

## Quick reference (host commands)

```console
# create / run
sbx run claude --name <name> [path]         # create+enter (direct mode)
sbx run --clone claude --name <name> [path]  # create+enter (clone mode)
sbx create --clone --name <name> claude .    # create backgrounded (clone mode)
sbx run --name <name>                        # reconnect (agent optional)
# manage
sbx ls
sbx stop <name>
sbx rm [--force] <name>
# access / policy (host services, ports, secrets — see your project's agent-guidance file)
sbx policy ls
sbx policy allow network -g <domain>
sbx ports <name> --publish 8080:8080/tcp
sbx secret set <name> github -t "$(gh auth token)"
```

## Sources
- [Docker Sandboxes](https://docs.docker.com/ai/sandboxes/) · [Usage](https://docs.docker.com/ai/sandboxes/usage/) · [Get started](https://docs.docker.com/ai/sandboxes/get-started/) · [`sbx` CLI reference](https://docs.docker.com/reference/cli/sbx/)
- Your project's agent-guidance file (CLAUDE.md / AGENTS.md) — "Network access", "Publishing ports", "Accessing services on the host", "Git Authentication", and any worktree / `--branch` push/PR flow it documents.

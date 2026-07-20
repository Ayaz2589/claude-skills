---
name: env-status
description: Report the state of the local development environment — which project containers are up and what data they hold — from a per-project set of read-only checks. On first run it learns your stack (containers + data probes, scaffolded from your compose file) and remembers it. Strictly read-only; never starts, stops, or reseeds anything.
user-invocable: true
argument-hint: "[none — or `reconfigure` to rebuild the checks]"
---

# env-status skill

Report what's running locally and what data is inside it, as a quick health read.
**Strictly read-only** — this skill never starts, stops, restarts, reseeds, or
otherwise mutates anything. It runs a project-specific list of read-only checks
and interprets the result.

Which containers and which "data probes" matter is entirely project-specific, so
the skill learns them once (first run) and stores them in a config file.

## Step 0 — Load or initialize config

**Locate the workspace root:** the nearest ancestor directory containing a
`.claude/` folder; if there is none, use the git repo root
(`git rev-parse --show-toplevel`) and create `.claude/` there.

**Config file:** `<workspace-root>/.claude/env-status.config.md`.

- **If it exists** and the argument is not `reconfigure`, read it and go to
  Step 1.
- **If it does NOT exist** (or the user passed `reconfigure`), run first-run
  setup below, write the config, then continue.

### First-run setup

The goal is a short list of **containers to report** and **read-only data
probes** (a labelled shell command whose stdout is the value to show).

1. **Detect the stack.** Look for a compose file at the repo root —
   `docker-compose.yml`, `docker-compose.yaml`, `compose.yml`, `compose.yaml`.
   If found, list its services (`docker compose config --services`, or read the
   file). Also run `docker ps --format '{{.Names}}'` to see what's currently up.
   This gives you concrete names to propose instead of guessing.

2. **Propose probes from what you found**, then confirm/adjust with the user.
   Match detected services to these read-only recipes (fill the `<placeholders>`
   from the user's real service/table/port names — ask when unsure):

   | If you see… | Propose a probe like |
   |---|---|
   | a Postgres/MySQL service | row counts for the tables that matter: `docker exec <db> psql -U <user> -d <db> -tAc "SELECT count(*) FROM <table>"` |
   | an Elasticsearch/OpenSearch service | doc count: `curl -sf -m5 http://localhost:9200/_all/_count \| python3 -c "import sys,json;print(json.load(sys.stdin)['count'])"` |
   | a Redis service | key count: `docker exec <redis> redis-cli DBSIZE` |
   | a web/app server | HTTP reachability: `curl -s -o /dev/null -m5 -w '%{http_code}' http://localhost:<port>` |
   | anything else | any **read-only** command whose stdout is a meaningful number/string |

   Keep it to the handful of checks the user actually cares about — this is a
   glance, not an audit.

3. **Safety gate.** Every probe command MUST be read-only (SELECT/count/GET/
   health only). Do not accept commands that write, migrate, seed, restart, or
   delete. If the user proposes a mutating command, refuse and ask for a
   read-only equivalent.

4. **Write the config** in exactly this shape:

   ```markdown
   # env-status config

   ## Containers
   Names or patterns to report (or `auto` to show all currently-running
   containers whose name matches the project). One per line.
   - auto

   ## Probes
   One probe per bullet: a short label, then an indented `cmd:` line with a
   read-only shell command. Its trimmed stdout is shown as the value.
   - <label, e.g. "Postgres · orders">
     cmd: <read-only command>
   - <label, e.g. "Web · HTTP status">
     cmd: curl -s -o /dev/null -m5 -w '%{http_code}' http://localhost:<port>
   ```

   Tell the user where you saved it and that they can hand-edit it anytime, or
   re-run with `reconfigure`. (Commit it to share with the team, or gitignore it.)

## Step 1 — Check the Docker daemon

```bash
docker info >/dev/null 2>&1 || echo "Docker daemon not reachable"
```

If it's unreachable, report that Docker isn't running and stop — there's nothing
else to check.

## Step 2 — Report containers

Run `docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'`. Filter to
the configured container names/patterns (or, for `auto`, show the project's
containers — those matching the compose project or the names your probes
reference). Note which configured containers are **down** (in the config but not
in `docker ps`).

## Step 3 — Run the probes

For each probe in config, run its `cmd`, capturing stdout. **Continue past
failures** — a down service or failed query reports as `<unavailable>` (or the
error, briefly), never aborts the rest. Never let a probe's failure stop the run.

## Step 4 — Present and interpret

Show a compact report:

```
Containers
  <name>    <status>    <ports>
  <name>    down
Data
  <label>                 <value>
  <label>                 <unavailable>
```

Then add a **one-to-three-line read** on it:
- Which containers are up vs down.
- Whether the data looks loaded vs empty (e.g. core tables at 0 rows ⇒
  migrations/seed likely not run; search index empty ⇒ search/listing pages
  would be blank; web server unreachable ⇒ frontend not started).
- Anything anomalous (containers up but datastore empty, etc.).

## What NOT to do

- **Never mutate.** No start/stop/restart/up/down, no migrate/seed/reset, no
  writes. If the user wants to *fix* the environment, that's a different tool —
  point them at the project's local-dev setup, don't do it here.
- **Don't abort on a single failure.** Every check is independent; one down
  service must not hide the status of the others.
- **Don't hardcode another project's names.** Everything specific comes from
  `.claude/env-status.config.md`.

## Adapt to the project at hand

The entire project-specific surface is `.claude/env-status.config.md` — the
container list and the probe commands. Edit it directly to add/remove checks, or
run `/env-status reconfigure` to rebuild it interactively. Ports/hosts that
differ per machine belong inside the probe commands (or as env vars you reference
there), so the same config works across a team with light edits.

---
name: explain
description: Explain a question, file(s), symbol, or concept plainly — read the real sources first, then give a straightforward, technical-but-lay walkthrough in a consistent structure. On first run it captures your default depth, audience, and where the project's authoritative docs live, and remembers them. Use for follow-ups too.
user-invocable: true
argument-hint: "[question | file path(s) | symbol | concept — or empty to explain the thing in context]"
---

# explain skill

Explain the requested thing in a straightforward way, grounded in the real
sources rather than memory. The audience is technical, so being technical is
fine — but keep it as plain and jargon-light as the topic allows, defining terms
in passing rather than assuming them.

## Step 0 — Load or initialize preferences

Preferences are stored so the skill matches how *this* project likes explanations.

**Locate the workspace root:** the nearest ancestor directory containing a
`.claude/` folder; if there is none, use the git repo root
(`git rev-parse --show-toplevel`) and create `.claude/` there.

**Config file:** `<workspace-root>/.claude/explain.config.md`.

- **If it exists**, read it and apply the defaults. Skip the rest of this step.
- **If it does NOT exist** (first run), gather preferences with **one**
  `AskUserQuestion` block — and make clear the user can just pick the defaults:
  1. **Default depth** — `Adaptive` (match depth to the ask — recommended),
     `Concise` (tight answers), or `Thorough` (full structure every time).
  2. **Audience** — `Engineers` (assume fluency) or `Mixed` (explain for
     non-specialists too).
  3. **Docs entry points** *(optional, free text)* — paths the skill should
     consult to ground itself fast, e.g. `README.md`, `docs/architecture.md`,
     `CLAUDE.md`. Blank is fine.

  Write the config:

  ```markdown
  # explain config

  - Default depth: <adaptive | concise | thorough>
  - Audience: <engineers | mixed>
  - Docs entry points: <comma-separated paths, or "-">
  ```

  Note where you saved it; the user can edit it or delete it to reconfigure.

These are defaults, not a straitjacket — a specific request always overrides them
(a one-line question gets a one-line answer even under "Thorough").

## Step 1 — Get the ground truth first

The request is: `$ARGUMENTS`. It may be a question, one or more file paths (or
globs), a symbol/function name, a concept, or a mix — or **empty**, in which case
the target is whatever the user is referring to in the surrounding conversation
("explain that", "explain this file"). If, after checking the conversation, you
still can't tell what to explain, ask **one** short question to pin it down.

Never explain from memory or assumption when the real source is available:

- If the request names files, symbols, functions, or config, **Read them fully**
  before writing a word. Follow one hop of context when it matters (a function's
  callers, an imported helper, the type it returns) so the explanation is correct,
  not just plausible.
- If it's a "how does X work" question about this codebase, locate X (Grep/Glob)
  and read it. Consult the configured docs entry points when they're relevant.
- If it's a general/conceptual question with no code target, answer from
  knowledge — no need to search.
- If the target is large, read the important parts, explain those, and say
  plainly what you skipped.

## Step 2 — Structure every explanation the same way

Consistency is the point — the user should always get the same shape:

1. **One-sentence plain summary** — what it *is* or *does*, in lay terms, before
   any detail.
2. **How it works** — the key pieces, in the order they matter or execute. Walk
   the real flow; don't narrate the code line by line.
3. **How it connects** — where it fits: who calls it, what it depends on, the
   data flow in and out. Skip only when there genuinely is no wider context.
4. **Gotchas / why it's like this** — non-obvious decisions, traps, edge cases,
   constraints — but only when they exist. Don't invent them to fill space.

Close with a one-line **bottom line** ("In short: …") when it helps the point
land. Omit it for short answers.

## Step 3 — Style

- **Ground it.** Reference concrete `file:line` locations and quote **short**
  snippets (a few lines, not whole functions).
- **Plain first, precise second.** Lead with the plain-language idea, then add
  the technical detail. Prefer a concrete example over an abstract description.
- **No filler.** Skip "great question", skip restating the request, skip hedging
  ("it seems", "I think") when you've read the code and know. State what's true;
  flag genuine uncertainty explicitly.
- **Right altitude.** Honor the configured default depth, then adjust to the ask.
  Under `Mixed` audience, define specialist terms on first use; under `Engineers`,
  assume fluency and move faster.
- Use headings, short paragraphs, and lists so it's skimmable. A small ASCII
  diagram is worth including when it clarifies a flow.

## Step 4 — Follow-up questions

When the user asks a follow-up, **apply this exact approach again**: read anything
new that's referenced, keep the earlier context, and build on what you already
explained instead of repeating it. Stay consistent in tone, structure, and
grounding — a follow-up should feel like a continuation, not a fresh essay.

## What NOT to do

- **Don't explain from memory** when the source is readable — read it first.
- **Don't dump everything you read.** Explain what the user needs to understand
  the thing, not every line you looked at.
- **Don't fabricate gotchas or connections** to fill the template. Omit sections
  that don't apply.

## Adapt to the project at hand

Defaults live in `.claude/explain.config.md` — edit it to change depth/audience,
or point "docs entry points" at the files that best ground explanations for this
codebase. Delete the file to re-run first-run setup.

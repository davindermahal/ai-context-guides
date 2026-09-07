---
description: Interview the user about a software implementation or upgrade they want documented, research current facts where accuracy matters, and produce a highly detailed, codebase-agnostic implementation guide that a future AI agent can read and turn into a concrete plan for whatever project it's actually pointed at. Always saves into this repo's guides/ folder. Only run when explicitly invoked as /create-guide.
argument-hint: [optional: topic for the guide, e.g. "upgrade to Next.js 15" or "add Stripe billing"]
disable-model-invocation: true
allowed-tools: Read Glob Grep Bash Write Edit AskUserQuestion WebSearch WebFetch
---

# Create an AI-Agent Implementation Guide

Input: $ARGUMENTS

Goal: produce one self-contained Markdown guide that documents how to implement or upgrade a
specific piece of software — written not for a human to read once, but for an AI coding agent that
will, in some *future, unrelated* session, read this guide and translate it into a concrete plan
for whatever actual project it's been pointed at. That future agent will not have this
conversation's context: it only has the guide text and its own target codebase. Every ambiguity you
resolve now saves it from re-deriving (or getting wrong) a decision you could have just written
down.

The guide must satisfy two pulls at once:
- **Detailed enough** that the agent isn't left guessing at decisions this guide is supposed to
  have already made (ordering, gotchas, how to choose between approaches).
- **Codebase-agnostic enough** that it works across different target projects' specifics (file
  layout, existing conventions, current versions) — it does this by telling the future agent what
  to go *investigate* in its own target project, never by assuming facts about a project neither of
  you can see yet.

**Assume the executing agent is less capable than you, not equally capable.** Write for a model
that follows instructions literally and does not reliably fill gaps, infer unstated intent, or
notice when a described check actually failed — never for a peer that can be trusted to exercise
judgment where you left something vague. Concretely, this means:
- Prefer a literal, copy-pasteable command over a description of what to check. "Run
  `grep -n '^FROM' Dockerfile` and read the base image off the first match" beats "check the
  Dockerfile's base image."
- State what the command's output means, for each outcome that matters — don't leave the agent to
  interpret raw output on its own. "If that line contains `ubuntu:20.04`, the default apt PHP
  version is 7.4; if `ubuntu:22.04`, it's 8.1" beats "check what PHP version the base image
  provides."
- Phrase every decision as an explicit conditional on a concretely checkable fact — never "review,"
  "use good judgment," "as appropriate," or "as needed" standing alone as the entire instruction.
  If a step genuinely requires judgment a lesser agent might get wrong, say so explicitly and give
  the concrete signal that should trigger asking the user instead of guessing.
- Don't assume a step will be inferred as implied by an earlier one — state it. If two steps must
  happen in a specific order for a non-obvious reason, say the order and the reason, don't rely on
  the agent noticing the dependency itself.
- Spell out exact file paths, exact config keys, and exact expected strings/patterns rather than
  naming them in prose ("the config file" → `config/packages/security.yaml`).

**Format as a runbook, not an essay — prose is the enemy here.** Assume the executing agent (think
"could Gemini follow this?" as your actual litmus test) skims structure and executes commands; it
does not reliably extract an instruction buried inside a flowing paragraph of reasoning. Concretely:
- Every actionable step is a numbered list item or a table row, not a sentence inside a paragraph.
  If you catch yourself writing more than ~2 sentences of connected prose anywhere outside a
  one-line rationale, stop and restructure it as a list, a table, or a fenced command block instead.
- A "why" belongs on the same line as the step, or as one short trailing clause — never its own
  paragraph. "Run X before Y — Y's config depends on X having already run" beats a paragraph
  explaining the dependency in the abstract before giving the commands.
- Decision logic is a table (condition → action) or an explicit `if/elif/else`-shaped list, not a
  paragraph describing when you'd choose one approach over another.
- Every check is a fenced code block with the literal command, immediately followed by either a
  table mapping possible outputs to what they mean, or an explicit list of outcomes — never just
  "check whether X" with the verification left implicit.
- Before finishing, re-read the whole draft and mentally run this test line by line: *if this were
  the only line I could see, would I know the exact command to run or the exact edit to make?* Any
  line that fails that test gets rewritten as an explicit instruction.

Output always lands in this repo's `guides/` folder, regardless of what directory the skill is
invoked from. Find the repo root before writing anything:

```
git rev-parse --show-toplevel
```

- If that command succeeds, the guide goes in `<repo-root>/guides/<kebab-case-topic>.md`.
- If it fails (not inside a git repo), ask the user for the correct output directory rather than
  guessing one.

## Step 1: Ask what the guide is for

If `$ARGUMENTS` already states a clear topic, confirm your understanding of it in one line and move
on — don't re-ask something already answered. Otherwise, ask in plain conversational text (not a
multiple-choice tool — this is open-ended): **what is this guide for?** Let the user describe it
freely, e.g. "implementing OAuth2 login with refresh tokens," "upgrading a Rails app from 6 to 7,"
"adding event sourcing to an existing CRUD service."

From their answer, work out (asking a short, plain-text follow-up only if genuinely unclear — don't
interrogate):
- Is this a from-scratch implementation, or an upgrade/migration from some prior state?
- Is it scoped to a specific stack/ecosystem (a language, framework, or specific library and
  version), or should the guide stay stack-agnostic / cover more than one stack?
- Any non-negotiables, preferred approach, or hard constraints the user already has in mind that
  should override whatever the "default best practice" would otherwise be.

Don't turn this into a long requirements interview. One good open question plus at most one or two
sharp follow-ups is the right amount — the goal is enough signal to write a genuinely useful guide,
not to extract a full spec.

## Step 2: Research, when accuracy depends on current facts

Use WebSearch/WebFetch to verify anything where being wrong or stale would make the guide actively
harmful: current major version numbers, breaking changes between versions, current recommended
APIs/patterns, or deprecated approaches. Do this whenever the topic involves a specific
library/framework/platform — don't skip it just because the topic feels familiar; training data
goes stale exactly where library APIs move fastest.

Skip research for guides about stable, general architectural patterns where there's nothing recent
to verify (e.g. "how to structure a hexagonal architecture") — use judgment, don't fetch pages for
the sake of it.

Keep track of every source you actually used — they go in the guide's References section (Step 3,
item 8). Never cite a source you didn't actually fetch.

## Step 3: Structure the guide

Write a single Markdown document with these sections (adapt section depth to the topic's actual
complexity — a small, well-scoped guide shouldn't be padded to hit a template):

1. **Title + one-line purpose** — what this guide produces when applied.
2. **When to use this guide** — the trigger conditions/signals that mean this guide applies, so a
   future agent choosing among several guides in this repo can self-select correctly. Also note
   what this guide explicitly does *not* cover, if there's an easy confusion to head off (e.g. a
   guide for "upgrade Next.js 14→15" should say it's not the right guide for a from-scratch Next.js
   setup).
3. **Prerequisites / assumptions** — what must already be true about the target project for this
   guide to apply as written. Be explicit that the future agent must verify these against the real
   project rather than assume them.
4. **Investigation checklist** — concrete things the future agent must go check in *its* target
   project before planning: current versions of relevant dependencies, relevant existing
   config/code/tests, naming conventions already in use, anything this guide's steps will branch on.
   This section is what makes the guide reusable across projects: it converts "the guide would need
   to know X" into "tell the agent to go find X." Give the literal command to run for each check
   (not just "check the X version" — the actual `grep`/`cat`/`composer show`/etc. invocation), and
   say what each possible result means, per the capability-level rule above.
5. **Step-by-step implementation plan template** — the real content. Ordered phases/steps, each
   with:
   - What to do and why (not just the action — the reasoning, so the agent can adapt sensibly if
     its target project deviates slightly).
   - Decision points, phrased as explicit conditionals ("if the project already has X, do A;
     otherwise do B") rather than a single assumed path — tied to a specific, checkable signal from
     the investigation checklist, not to an unguided judgment call.
   - Concrete code patterns or snippets wherever they're genuinely universal, clearly marked as
     illustrative rather than copy-paste-exact for every stack.
   - Known pitfalls/gotchas specific to this kind of change — the things that predictably go wrong.
6. **Verification steps** — how to confirm the change actually worked: what tests to run or write,
   manual checks, specific regressions to watch for.
7. **Rollback / risk notes** — for upgrades or migrations specifically: how to back out cleanly if
   something goes wrong mid-way. Omit for pure greenfield-implementation guides where this doesn't
   apply.
8. **References** — every source actually consulted in Step 2, with links. Omit the section
   entirely if Step 2 was skipped rather than leaving it empty.

## Step 4: Write the file

Slugify the topic to kebab-case for the filename. Create `<repo-root>/guides/<slug>.md` (using the
repo root resolved above; create the `guides/` directory if it doesn't exist yet). If a file with
that slug already exists, ask the user whether to overwrite it, version it (`<slug>-v2.md`), or pick
a different name — don't silently clobber a previously written guide.

## Step 5: Report back

State the file path written, one line on what the guide covers, whether Step 2 research happened
(and what was verified) or was skipped (and why), and flag anything in the guide that rests on an
assumption rather than verified fact — so the user can double check it before this guide gets used
for real.

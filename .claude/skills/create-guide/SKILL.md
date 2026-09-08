---
name: create-guide
description: Interview the user about a software implementation or upgrade they want documented, research current facts where accuracy matters, and produce a highly detailed, codebase-agnostic implementation guide that a future AI agent can read and turn into a concrete plan for whatever project it's actually pointed at. Always saves into this repo's guides/ folder. Only run when explicitly invoked as /create-guide — never trigger this automatically on your own judgment.
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

## Step 2: Research — verify, never assume

Never answer from memory or training data when a fact can be checked instead. Training data goes
stale exactly where library/framework APIs move fastest, and a wrong "current" fact doesn't just
sit there unused — it actively misleads a future, less-capable agent that has no way to know it's
stale. Treat every claim about versions, APIs, defaults, config keys, or behavior as something to
verify with WebSearch/WebFetch before it goes in the guide, never as something to recall and trust.
Do this whenever the topic involves a specific library/framework/platform — don't skip it because
the topic feels familiar; familiarity is not verification.

Skip research only for guides about stable, general architectural patterns where nothing
version-specific exists to check (e.g. "how to structure a hexagonal architecture"). If the topic
names any concrete technology at all, this exception does not apply — go verify.

### For upgrade/migration guides specifically: read every CHANGELOG, not just the highlights

A guide built from a single blog post's "top breaking changes" list is not comprehensive — blog
summaries skew toward what's interesting to write about, not what will actually break someone's
build. Do all of the following before writing anything:

1. Pin down the exact starting version and exact target version the guide covers. If `$ARGUMENTS`
   or the user's answer in Step 1 leaves either end ambiguous (e.g. "Symfony 5 to 6" doesn't say
   which 5.x or which 6.x), ask a short follow-up rather than guessing one.
2. Find the project's own primary changelog source(s) — never a third-party summary or aggregator
   site as the primary source:
   - `CHANGELOG.md` / `CHANGELOG.txt` / `HISTORY.md` in the project's own repository.
   - Dedicated per-version upgrade files, if the project publishes them (e.g. Symfony ships one
     `UPGRADE-X.Y.md` per minor version; Rails and Django ship per-version upgrade guides in their
     docs).
   - The project's official "release notes" / "what's new" pages on its own docs site.
   - The repo's GitHub/GitLab "Releases" page, only as a fallback when no dedicated changelog file
     exists.
3. Enumerate **every** version between the start and target — every major, every minor, and, if the
   project documents breaking changes at the patch level, every patch. Do not jump straight from the
   start version's notes to the target version's notes: each intermediate version carries its own
   deltas, and skipping any one of them is exactly how the guide ends up silently missing a real
   breaking change that a project sitting on that intermediate version would hit.
4. WebFetch the changelog/upgrade notes for every version enumerated in step 3, individually — not
   just the two endpoints.
5. From each version's notes, extract every breaking change, removed feature, deprecation (note the
   version it was deprecated in and the version it's slated for or actually removed in), default-value
   change, and behavior change — not only the ones a summary would flag as "major." Deprecations
   matter even when they don't error yet: a target project sitting several versions behind the
   target may already depend on something that was deprecated a few versions ago and gets removed
   partway through the upgrade path this guide describes.
6. Compile the full result into the version-by-version breaking-changes list required by Step 3
   (new item 5, below) — this list is the backbone of the guide's implementation plan, not a
   supplementary appendix to skim past.

Keep track of every source actually fetched — they go in the guide's References section (Step 3,
final item). Never cite a source you didn't actually fetch, and never paper over a gap with a guess:
if a specific version's official changelog can't be located, say so explicitly in the guide (which
version, and what you tried) rather than silently omitting that version's coverage.

## Step 3: Structure the guide

Write a single Markdown document with these sections. Err toward comprehensiveness: a longer guide
that captures every real edge case beats a shorter one that reads cleanly but leaves one out — a
future agent only pays for the lines it actually needs (it skips what doesn't apply) but pays
dearly for a line that was cut to save space. Only shorten or omit a section when it's genuinely
inapplicable to the topic (e.g. drop "Rollback / risk notes" entirely for a greenfield-only guide),
never to keep the document short for its own sake:

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
   project before planning. Be exhaustive here, not representative: every dependency this guide's
   steps touch or branch on, every relevant existing config file, current versions (not just the
   headline framework/library — its supporting packages, runtime, and build tooling too where the
   steps depend on them), existing code/test conventions, and CI/build config if the upgrade or
   feature touches how the project builds or deploys. This section is what makes the guide reusable
   across projects: it converts "the guide would need to know X" into "tell the agent to go find X."
   A checklist item you left out doesn't just make the guide less thorough — it's a decision point
   later in the plan that the future agent will hit with no signal to branch on. For each item, give
   the literal command to run (not just "check the X version" — the actual
   `grep`/`cat`/`composer show`/etc. invocation), and say what each possible result means, per the
   capability-level rule above.
5. **Complete breaking-changes list, version by version (upgrade/migration guides only)** — the
   direct output of the Step 2 changelog research, omitted only for pure greenfield-implementation
   guides. Group entries by the version that introduced each change; for every single change
   extracted in Step 2 (not a curated "notable changes" shortlist — every one), give:
   - What changed — the literal old vs. new API/behavior/config key/default.
   - Exactly how the future agent checks whether *its* target project is affected — a literal
     `grep`/search pattern, file path, or config key to look for, never "check if you use X" left
     unresolved into a command.
   - Exactly what to change if it is affected, and what happens if the check comes back negative
     (usually: skip this entry, nothing to do).
   Do not compress this list to save space. An entry that affects few target projects still costs an
   unaffected agent nothing (it runs the check, gets a negative result, moves on) — but omitting it
   costs an affected agent a broken upgrade with no warning.
6. **Step-by-step implementation plan template** — the real content. Ordered phases/steps, each
   with:
   - What to do and why (not just the action — the reasoning, so the agent can adapt sensibly if
     its target project deviates slightly).
   - Decision points, phrased as explicit conditionals ("if the project already has X, do A;
     otherwise do B") rather than a single assumed path — tied to a specific, checkable signal from
     the investigation checklist, not to an unguided judgment call.
   - Concrete code patterns or snippets wherever they're genuinely universal, clearly marked as
     illustrative rather than copy-paste-exact for every stack.
   - Known pitfalls/gotchas specific to this kind of change — the things that predictably go wrong.
     For upgrade guides, cross-reference the breaking-changes list in item 5 by version rather than
     re-deriving pitfalls from scratch.
7. **Verification steps** — how to confirm the change actually worked: what tests to run or write,
   manual checks, specific regressions to watch for.
8. **Rollback / risk notes** — for upgrades or migrations specifically: how to back out cleanly if
   something goes wrong mid-way. Omit for pure greenfield-implementation guides where this doesn't
   apply.
9. **References** — every source actually consulted in Step 2, with links. For upgrade guides, this
   must include every per-version changelog/upgrade-notes page actually fetched in Step 2 — not just
   the two endpoint versions. Omit the section entirely if Step 2 was skipped rather than leaving it
   empty.

## Step 4: Write the file

Slugify the topic to kebab-case for the filename. Create `<repo-root>/guides/<slug>.md` (using the
repo root resolved above; create the `guides/` directory if it doesn't exist yet). If a file with
that slug already exists, ask the user whether to overwrite it, version it (`<slug>-v2.md`), or pick
a different name — don't silently clobber a previously written guide.

## Step 5: Report back

State the file path written, one line on what the guide covers, whether Step 2 research happened
(and what was verified) or was skipped (and why), and flag anything in the guide that rests on an
assumption rather than verified fact — so the user can double check it before this guide gets used
for real. For upgrade/migration guides, explicitly list which versions between start and target had
their changelog/upgrade notes fetched and which (if any) couldn't be located — don't let a gap pass
silently.

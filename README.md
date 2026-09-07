# AI Context Guides

A collection of reusable **guides** for AI coding agents. Each guide gives an
agent everything it needs to turn a developer's request (e.g. "upgrade this
project to Symfony 5") into a concrete, verifiable plan for a *specific*
project — investigation steps, decision tables, STOP conditions, and
verification checks.

## Audience

These guides are written for AI agents, not humans. A developer states a
goal; an agent selects the matching guide and uses it to investigate the
target project, build a plan, and execute against that project's actual
state (directory layout, dependency versions, config, etc.).

## Format

Every guide is a **runbook**, not prose. At minimum:

- An investigation checklist of commands to run, with output mapped to
  decision tables (not free-form judgment calls).
- Explicit "When to use this guide" / "Do NOT use for" conditions.
- Explicit STOP conditions — when a case isn't covered, the agent stops and
  asks rather than guessing.
- Phases done in a fixed order.
- A verification checklist at the end.

See [`guides/upgrade-symfony-4-to-symfony-5.md`](guides/upgrade-symfony-4-to-symfony-5.md)
as the reference example.

## Repo layout

```
guides/   One markdown file per guide, self-contained.
.ai/      Agent-maintained project context (see documentation-mcp).
```

## Distribution

Guides live here as markdown but can also be uploaded to Confluence or
another wiki for use elsewhere. `ai-intake-mcp` can search for and fetch
guides stored in Confluence, so a guide doesn't have to live in this repo to
be usable by an agent.

## Contributing a guide

1. Copy the structure of an existing guide rather than starting from a blank
   page — consistency matters more than creativity here.
2. Write for an agent with no memory of this conversation: every assumption
   must be verified with a command, not asserted.
3. Prefer decision tables over prose explanations.
4. Keep guides scoped to one migration/task and one clearly stated set of
   starting conditions. Don't try to cover every variant in one file — write
   a second guide instead.

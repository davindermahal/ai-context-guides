# Purpose

This repo is a collection of reusable "guides" for AI agents. Each guide gives an AI agent everything it needs to form a plan for a specific, well-defined task on a specific project — e.g. `guides/upgrade-symfony-4-to-symfony-5.md` walks an agent through upgrading a Symfony 4 app to Symfony 5.

## Audience

Target users are AI agents, not humans. A developer states a goal ("upgrade this to Symfony 5"); an agent selects/loads the matching guide and uses it to build a plan and execute against the developer's actual project. Guides must be personalizable/adaptable to the specifics of whatever project the agent is pointed at (directory layout, existing config, dependency versions, etc.) rather than assuming one fixed codebase.

## Format requirement

Guides are runbooks, not prose: investigation commands with decision tables, explicit STOP conditions, phases done in order, verification checklists. See `guides/upgrade-symfony-4-to-symfony-5.md` as the reference example. (Also recorded as a standing preference — see memory: guide-format-runbook-not-prose.)

## Distribution

Guides live here as markdown files but are also meant to be uploaded to Confluence or another wiki for storage/use elsewhere. `ai-intake-mcp` has functionality to search for guides in Confluence — that's the retrieval path when guides aren't sourced from this repo directly.

## Open questions (from initial scan, still unanswered)

- Success criteria / "done" for this project specifically (e.g. N guides covering X stack breadth?)
- Hard constraints on guide content (compliance, security disclaimers, etc.)
- No README/CI/infra config exists yet — unclear if that's intentional for a docs-only repo or still to be added.

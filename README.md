# grill-plan-execute

An **orchestration-only** Claude Code skill that chains 5 skills from 3 external repositories into one workflow: clarify the requirement by grilling the user and modelling the domain, check with the user how complex the business really is, then produce implementation plans and hand off to execution.

This repository contains chaining logic only. **It holds no copy of any dependency skill's content** — every dependency is fetched from its original repository, so upstream improvements reach users without this repo changing at all.

> **About `obra/superpowers-skills`**: that repository is archived (no updates since 2025-10). The `writing-plans` skill this workflow needs now lives in the actively maintained [obra/superpowers](https://github.com/obra/superpowers) (v7.x). If you need the archived version, fork it and maintain it yourself.

## Dependencies

| Namespace | Role | Plugin / marketplace | Source repo |
| --- | --- | --- | --- |
| `mattpocock-skills:grill-with-docs` | Entry point: runs the interview and writes the docs in one call | `mattpocock-skills@mattpocock` ^1.3.1 | [mattpocock/skills](https://github.com/mattpocock/skills) |
| `mattpocock-skills:grilling` | Decision-tree interview across multiple rounds | same | same |
| `mattpocock-skills:domain-modeling` | Maintains the glossary and decision records as you go | same | same |
| `engineering-discipline:milestone-planning` | Parallel codebase exploration → milestone DAG → adversarial critic rounds | `engineering-discipline@engineering-discipline` ^1.7.0 | [tmdgusya/engineering-discipline](https://github.com/tmdgusya/engineering-discipline) |
| `superpowers:writing-plans` | Detailed implementation plan written for an engineer with zero context | `superpowers@superpowers-dev` ^7.0.0 | [obra/superpowers](https://github.com/obra/superpowers) |

The three execution skills used in Phase 5 (`subagent-driven-development`, `executing-plans`, `finishing-a-development-branch`) ship in the same `superpowers` plugin, so they add no extra dependency.

## Workflow

```
Phase 1: mattpocock-skills:grill-with-docs   (grilling + domain-modeling)
    ↓  requirement settled, user confirms
Phase 2: ask the user whether the business logic is complex
         (>8 implementation steps, or >3 core modules to touch/understand)
    ├─ not complex ─────────────────────────────┐
    └─ complex → Phase 3                        │
Phase 3: engineering-discipline:milestone-planning
         full flow: explore → draft → adversarial critique → revise until dry
         ↓  milestone DAG locked                │
Phase 4: superpowers:writing-plans  ←───────────┘
         complex: one plan per milestone; not complex: one plan overall
    ↓
Phase 5: hand off to execution
    ├─ Subagent-driven → superpowers:subagent-driven-development
    └─ Native          → superpowers:executing-plans
         ↓
    superpowers:finishing-a-development-branch (merge / PR / cleanup)
```

The complexity judgement in Phase 2 belongs to the user — the skill always asks, never decides for them.

---

# Installation — pick one of two paths

The same repository supports both CC Switch and Claude Code Plugin. CC Switch discovers skills through the `skills/` directory; Claude Code reads the manifests in `.claude-plugin/`.

## Path 1: CC Switch

CC Switch looks for skills in a fixed directory layout, so the `skills/` level in this repo exists for you.

**1. Add this repository**

| Field | Value |
| --- | --- |
| Owner | `zhy-ty` |
| Name | `grill-plan-execute` |
| Branch | `main` |
| Subdirectory | `skills` |

Then install `grill-plan-execute`.

**2. Install the dependencies by hand (this step cannot be skipped)**

CC Switch **does not read the `dependencies` field in `plugin.json`**. After installing this skill you must add and install all three upstream repositories yourself. Each one also uses Subdirectory `skills`:

- [mattpocock/skills](https://github.com/mattpocock/skills) — Subdirectory: `skills` (provides `grill-with-docs`, `grilling`, `domain-modeling`)
- [tmdgusya/engineering-discipline](https://github.com/tmdgusya/engineering-discipline) — Subdirectory: `skills` (provides `milestone-planning`)
- [obra/superpowers](https://github.com/obra/superpowers) — Subdirectory: `skills` (provides `writing-plans`)

Use the workflow only once all three are installed. If one is missing, the workflow stops at the phase that needs it and tells you (that is what Step 0 in `SKILL.md` checks for).

> Note: after a CC Switch install these skills may not carry a plugin prefix. `SKILL.md` uses Claude Code Plugin's `plugin:skill` form; if your CC Switch exposes bare skill names, just drop the prefix — it is the same skill.

## Path 2: Claude Code Plugin

Dependencies are pulled automatically, but **the three upstream marketplaces must be configured first** — that part cannot be skipped.

**1. Add the three marketplaces that host the dependencies**

```bash
claude plugin marketplace add mattpocock/skills
claude plugin marketplace add tmdgusya/engineering-discipline
claude plugin marketplace add obra/superpowers
```

Their marketplace names are `mattpocock`, `engineering-discipline` and `superpowers-dev`. `allowCrossMarketplaceDependenciesOn` in this repo's `marketplace.json` already declares those three (plus `claude-plugins-official`, see below).

**2. Add this repo's marketplace and install**

```bash
claude plugin marketplace add zhy-ty/grill-plan-execute
claude plugin install grill-plan-execute@grill-plan-execute-marketplace
```

Claude Code then installs and enables all three dependencies listed in `plugin.json`.

**3. Use it**

```
/grill-plan-execute:grill-plan-execute
```

### Two behaviours worth knowing

Both observed on Claude Code 2.1.278 (resolved dependency versions: `mattpocock-skills` 1.3.1, `engineering-discipline` 1.7.0, `superpowers` 7.0.0):

1. **A missing upstream marketplace does not fail the install.** `plugin install` still reports success, but the plugin then shows `✘ failed to load` with an error naming the unsatisfied dependency and the command to fix it:
   `Dependency "mattpocock-skills@mattpocock" is not installed — run `claude plugin install mattpocock-skills@mattpocock`, or check that its marketplace is added`
   So verify with `claude plugin list` that this plugin reads `✔ enabled` — do not trust the checkmark on the install line.
2. **Order does not matter.** Installing this plugin first and adding the upstream marketplaces later still self-heals: `marketplace add` output shows `(+ 1 dependency: mattpocock-skills)`, meaning it installs the owed dependency the moment the marketplace is configured.

Other things to check:

- Version range unsatisfiable (e.g. upstream ships `superpowers` 8.0.0 while this repo declares `^7.0.0`) → widen or bump `version` in `plugin.json`
- You installed `superpowers` from the official `claude-plugins-official` marketplace (its version usually trails upstream) → change that dependency's `"marketplace": "superpowers-dev"` to `"claude-plugins-official"` in `plugin.json`; both names are already in the allow list, so either passes the permission check. They can also coexist — `superpowers@superpowers-dev` and `superpowers@claude-plugins-official` side by side was tested and does not conflict.

## Versioning and updates

- **Dependency content updates**: this repo holds no copies of the dependencies, so once upstream releases a new version you get it by running `claude plugin update <plugin>@<marketplace>` (CC Switch users re-sync the repository). **Nothing needs to change in this repo.**
- **This repo's version number**: bump `plugin.json`'s `version` and tighten `dependencies[].version` only when a dependency changes incompatibly — a skill renamed or removed, or its output restructured so the chaining no longer holds.
- The current `dependencies` use caret (`^`) ranges: they follow features and fixes within the same major version, and deliberately do not follow a major upgrade.

## License

MIT — see [LICENSE](./LICENSE). Copyright for the dependency skills belongs to their respective authors; this repo claims no rights over them.

# grill-plan-execute

[中文](./README.zh.md)

An **orchestration-only** Claude Code skill that chains skills from 3 external repositories into one workflow: clarify the requirement by grilling the user and modelling the domain, check with the user how complex the business really is, then produce implementation plans and hand off to execution.

This repository contains chaining logic only. **It holds no copy of any dependency skill's content** — every dependency is fetched from its original repository, so upstream improvements reach users without this repo changing at all.

> **About `obra/superpowers-skills`**: that repository is archived (no updates since 2025-10). The `writing-plans` skill this workflow needs now lives in the actively maintained [obra/superpowers](https://github.com/obra/superpowers) (v7.x). If you need the archived version, fork it and maintain it yourself.

## Dependencies

| Namespace | Required | Role | Plugin / marketplace | Source repo |
| --- | --- | --- | --- | --- |
| `mattpocock-skills:grilling` | **yes** | Decision-tree interview across multiple rounds | `mattpocock-skills@mattpocock` ^1.3.1 | [mattpocock/skills](https://github.com/mattpocock/skills) |
| `mattpocock-skills:domain-modeling` | **yes** | Maintains the glossary and decision records as you go | same | same |
| `engineering-discipline:milestone-planning` | **yes** (complex path) | Parallel codebase exploration → milestone DAG → adversarial critic rounds | `engineering-discipline@engineering-discipline` ^1.7.0 | [tmdgusya/engineering-discipline](https://github.com/tmdgusya/engineering-discipline) |
| `superpowers:writing-plans` | **yes** | Detailed implementation plan written for an engineer with zero context | `superpowers@superpowers-dev` ^7.0.0 | [obra/superpowers](https://github.com/obra/superpowers) |
| `mattpocock-skills:grill-with-docs` | no | Shortcut that runs the two skills above in one call | same as `grilling` | same |
| `superpowers:subagent-driven-development`, `superpowers:executing-plans`, `superpowers:finishing-a-development-branch` | no | Where Phase 5 hands off to | same as `writing-plans` | same |

Four skills are required; the rest only make the workflow shorter to drive. Note that on the plugin path you install whole plugins, so you cannot cherry-pick this list — see [Granularity](#granularity-what-the-plugin-path-costs-you).

## Workflow

```
Phase 1: mattpocock-skills:grilling + mattpocock-skills:domain-modeling
         (single call to grill-with-docs when that shortcut is installed)
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

CC Switch **does not read the `dependencies` field in `plugin.json`**. You must add each upstream repository and install the skills it hosts. Add a repo with Subdirectory `skills`, then pick only the skills this workflow needs:

| Repository (Subdirectory: `skills`) | Required | Only needed for Phase 5 | Optional |
| --- | --- | --- | --- |
| [mattpocock/skills](https://github.com/mattpocock/skills) | `grilling` (`skills/productivity/grilling`), `domain-modeling` (`skills/engineering/domain-modeling`) | — | `grill-with-docs` (`skills/engineering/grill-with-docs`) |
| [tmdgusya/engineering-discipline](https://github.com/tmdgusya/engineering-discipline) | `milestone-planning` (`skills/milestone-planning`) | — | `clarification`, `plan-crafting`, `long-run`, `run-plan`, `review-work` |
| [obra/superpowers](https://github.com/obra/superpowers) | `writing-plans` (`skills/writing-plans`) | `subagent-driven-development`, `executing-plans`, `finishing-a-development-branch` | `requesting-code-review`, `receiving-code-review`, `test-driven-development`, `using-git-worktrees`, `verification-before-completion`, `dispatching-parallel-agents` |

That is **4 skills minimum** (`grilling`, `domain-modeling`, `milestone-planning`, `writing-plans`), 7 if you also want the Phase 5 handoff. The "Optional" column is never required: `SKILL.md` falls back to calling `grilling` + `domain-modeling` itself when `grill-with-docs` is absent, and the other optional skills are only named as destinations when an upstream document points at one — if missing, the workflow tells you instead of improvising.

Use the workflow only once the required skills are installed. If one is missing, the workflow stops at the phase that needs it and tells you (that is what Step 0 in `SKILL.md` checks for).

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

### Granularity: what the plugin path costs you

Plugin dependencies are all-or-nothing: you get every skill the upstream plugin exposes, not just the four this workflow calls. Measured on Claude Code 2.1.278 with the pinned versions:

| Plugin | Skills installed | Always-on cost |
| --- | --- | --- |
| `mattpocock-skills` | 27 | ~1,169 tok |
| `engineering-discipline` | 11 | ~997 tok |
| `superpowers` | 15 | ~706 tok |
| **total** | **53** | **~2,872 tok in every session's system prompt** |

That is on top of this plugin's own skill (~57 tok). We cannot trim it from our side: the skill list lives in each upstream `plugin.json` (Matt's explicitly enumerates 27 paths), and a marketplace entry has no supported way to subset somebody else's plugin.

If you only want the four skills, use the CC Switch path above. To stay on the plugin path with a smaller footprint you would have to fork this repo, delete the `dependencies` array from `plugin.json`, and install the upstream skills by hand — which also gives up automatic dependency resolution and automatic upstream updates.


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

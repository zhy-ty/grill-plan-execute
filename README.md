# grill-plan-execute

[中文](./README.zh.md)

Repository: **<https://github.com/zhy-ty/grill-plan-execute>**

An **orchestration-only** Claude Code skill that chains skills from 3 external repositories into one workflow: clarify the requirement by grilling the user and modelling the domain, check with the user how complex the business really is, then produce implementation plans and hand execution to a fresh session — planning on a strong model, execution optionally on a cheaper one with strong-model review.

This repository contains chaining logic only. **It holds no copy of any dependency skill's content** — every dependency is fetched from its original repository, so upstream improvements reach users without this repo changing at all.

> **About `obra/superpowers-skills`**: that repository is archived (no updates since 2025-10). The `writing-plans` skill this workflow needs now lives in the actively maintained [obra/superpowers](https://github.com/obra/superpowers) (v7.x). If you need the archived version, fork it and maintain it yourself.

## What this skill does not promise

**It does not guarantee token savings.** No number here is measured against a baseline, chained subagents still consume tokens, and a cheaper execution model can cost more by retrying. What it is for is narrower: **fewer omissions and fewer mistakes in the code that gets written, so there is less rework.** The requirement is confirmed before it is planned, the plan is reviewed before execution is authorised, and the domain language and decisions live in documents rather than in one long chat transcript that the next session cannot see.

`zhy-ty` already works complex requirements through exactly these steps by hand — grill the requirement, model the domain, judge how complex it really is, split it into milestones, write the plan, hand execution to a fresh session. This skill exists only to make that habit one command.

### What zhy-ty actually observed

The origin is a capability problem, not a cost problem. When a requirement was run end to end by an ordinary model, the model's limited intelligence meant the drift only became visible at the very end — the code no longer matched the original requirement, and nothing along the way had flagged it. **That is why he went looking for skills in the first place**, and the workflow in this repository is what he gradually settled on while using them.

The result of that process, from his own usage: **token consumption clearly went up, and rework dropped a lot** — the "agent only reveals at the end that it went off track" situation basically stopped happening. No number here is measured against a baseline, so treat it as one user's experience, not a benchmark.

The separate recommendation to plan on a strong model and execute on a cheaper one is a step beyond that, not the reason this workflow exists. It is an **ideal case**: put the design work on a capable model to push the error rate down further, then let a cheaper model execute a plan that has already survived review. It has not been validated by measurement here, and if the plain workflow above already holds your rework in check, you do not need to buy a second model arrangement.

So: **go read the source skills themselves, and call them directly if that is enough.** Each is strong on its own, and the full chain is not always the right size for the job.

### Using a subset instead

| Your situation | Call | What you get |
| --- | --- | --- |
| Simple requirement you can hold in your head end to end | `grill-with-docs` (which wraps `grilling` + `domain-modeling`) | The requirement interrogated and the glossary / decision records written down. No planning stage. |
| Worried the agent drifts mid-execution | `grill-with-docs` + `writing-plans` | A detailed implementation plan written for an engineer with zero context — that document is the track the agent has to stay on. |
| You assess the change as large and don't trust yourself to hold it | `grill-with-docs` + `milestone-planning` + `writing-plans` | Exploration and a milestone DAG with adversarial review first, then one plan per milestone. |

The last row **is** this skill's flow — the integrated version adds the Phase 2 complexity question that picks between row 2 and row 3, plus the checkpointed handoff between the phases. If you already know which row applies, calling those skills directly is a legitimate choice.

**And this chain will keep growing.** `SKILL.md` is a pure orchestration layer, and any external skill that proves useful gets inserted into it — that is the intended way for this project to grow (see *Extending the workflow with another skill* further down). All the more reason to know the individual skills and run only the stages your task actually needs, because the honest token picture is this: the integrated chain buys **fewer rewrites, not a lower bill**. What genuinely lowers the bill is picking two call sites where the full chain would use four.

## Dependencies

| Namespace | Required | Role | Plugin / marketplace | Source repo |
| --- | --- | --- | --- | --- |
| `mattpocock-skills:grilling` | **yes** | Decision-tree interview across multiple rounds | `mattpocock-skills@mattpocock` ^1.3.1 | [mattpocock/skills](https://github.com/mattpocock/skills) |
| `mattpocock-skills:domain-modeling` | **yes** | Maintains the glossary and decision records as you go | same | same |
| `engineering-discipline:milestone-planning` | **yes** (complex path) | Parallel codebase exploration → milestone DAG → adversarial critic rounds | `engineering-discipline@engineering-discipline` ^1.7.0 | [tmdgusya/engineering-discipline](https://github.com/tmdgusya/engineering-discipline) |
| `superpowers:writing-plans` | **yes** | Detailed implementation plan written for an engineer with zero context | `superpowers@superpowers-dev` ^7.0.0 | [obra/superpowers](https://github.com/obra/superpowers) |
| `engineering-discipline:clarification` | **format resource required** | Read the Context Brief template and Complexity Assessment rules; do not run its interview | same as `milestone-planning` | same |
| `mattpocock-skills:grill-with-docs` | no | Convenience entry point for `grilling` and `domain-modeling`; not a token-saving guarantee | same as `grilling` | same |
| `superpowers:subagent-driven-development`, `superpowers:executing-plans`, `superpowers:finishing-a-development-branch` | no | Where Phase 5 hands off to | same as `writing-plans` | same |
| `superpowers:verification-before-completion`, `engineering-discipline:review-work` | no (strongly recommended) | Phase 5 review safety nets: executor self-verification and independent plan-vs-code review — required in practice whenever execution runs on a cheaper model | same as `writing-plans` / `milestone-planning`, no extra plugin | same |

There are four workflow skill calls, plus a required format resource from `clarification`. The plugin path includes that resource through the existing `engineering-discipline` dependency; CC Switch users must also install `clarification` unless its source files are already locally readable. It is not an extra interview stage. Plugin installation is all-or-nothing — see [Granularity](#granularity-what-the-plugin-path-costs-you).

## Workflow

```
Phase 1: grilling + domain-modeling → confirmed Context Brief
Phase 2: user confirms complexity → checkpoint / fresh context
    ├─ not complex ─────────────────────────────┐
    └─ complex → Phase 3                        │
Phase 3: milestone-planning                     │
         full exploration and adversarial review (up to 2 rounds)
         environment doc closes here; user locks DAG → checkpoint / fresh context
Phase 4: writing-plans ←────────────────────────┘
         one milestone; subagent drafts, main agent handles questions
Phase 5: user chooses execution → fresh execution session
    ├─ Subagent-driven → subagent-driven-development
    └─ Native          → executing-plans
         ↓
    finishing-a-development-branch (merge / PR / cleanup)
         ↓  checkpoint / fresh context
    if milestones remain → Phase 4, resume text offered unprompted; otherwise finish
```

Phase 3 also closes out an environment document that every later session reads (see rule 2 below).

The complexity judgement in Phase 2 belongs to the user, using `Complex` / `Not complex` options when the host supports them. The recommendation comes from the `Complexity Assessment` rules actually read from the installed upstream resource, not an invented score. The existing rule of thumb remains more than eight implementation steps or more than three core modules. Prompts and explanations follow the user's language; resuming unchanged work does not repeat an already confirmed choice.

### How the workflow limits redundant work

1. **Bound the interview.** Stop once goals, scope/non-goals, constraints and acceptance criteria are confirmed and no issue blocks planning. User-accepted non-blocking assumptions have a review point before the affected work; safety, permissions, data integrity and irreversible migration questions cannot be deferred just to save tokens. Explicit interview-only requests stop before the document/planning stages and do not require the template resource. Tag every decision the user settles with a status: `locked` (final, cannot be reopened) or `deferred-with-criteria` (the user allows the execution phase to decide by criteria). Adversarial review may attack milestone splits and criteria, but it may not turn a `locked` decision into a question asked again at execution time — it escalates to the user instead.
2. **Batch documents and pass paths.** Update terminology, decisions and assumptions after each confirmed topic, and finish writing before any phase transition. Read the upstream Context Brief template and scoring rules once at entry; record their source path rather than copying them into this repository. The brief lives at `docs/engineering-discipline/context/YYYY-MM-DD-<topic>-brief.md` and supplies the plan's `Spec:` reference. Environment facts (repository paths including out-of-workspace ones, build tool absolute paths, database/service addresses, runtime state, shell quirks) go into a single sibling file `docs/engineering-discipline/context/YYYY-MM-DD-<topic>-environment.md`, created when Phase 3 review closes (or before the first plan on the simple path), updated incrementally, and read by every planning and execution session. Plans and the brief only reference its path instead of copying it, and a fact already recorded there is never re-asked as a user question. Credentials follow their source: a value already in a config file is referenced by file path and key rather than copied, an account the user hands over in conversation is recorded verbatim together with what it unlocks, and something with no source at all becomes an explicit gap instead of a guessed default. Because it can carry credentials, this file is never part of the commit set and opens with a "contains credentials, do not commit" line. Like the brief, the plans and the review artifacts it is a process artifact: whether those get committed is the user's call, but anything meant to outlive the work belongs in ADRs and the glossary rather than only in process documents.
3. **Use real context checkpoints.** After Phase 2, DAG lock and each completed milestone, update a short `Workflow Resume` section at the end of the Context Brief, then continue in fresh context. It records the next phase, confirmations, artifact paths including the environment document, pending assumptions, code baseline and verification evidence. Upstream `state.md` remains authoritative for milestone status. If new sessions are unavailable, use actual host compaction; if neither exists, provide manual resume instructions unless the user explicitly chooses to stay in place. On resume, check relevant committed and uncommitted changes, not the entire interview history. Handing over a checkpoint also means telling the user its boundary in the same message: which single milestone or stage this session covers, how many remain, and what the user must do afterwards. After a milestone closes, the main agent produces the next milestone's resume text on its own initiative rather than waiting to be asked.
4. **Isolate work with explicit return states.** Planning subagents return `needs-input`, `plan-ready` or `blocked`. The main agent handles questions and execution choices; answers resume the existing draft rather than restart planning. A ready result carries a path and at most five summary lines, with execution options supplied separately. Questions and blockers are not truncated to meet that limit. Before any pause, save the status, existing draft path and outstanding questions in `Workflow Resume`; resume that draft after unblocking and never treat it as an executable plan. The main agent checks the actual plan before handoff. If the host cannot delegate this skill, the main agent calls it in fresh planning context instead. Before anything is put to the user — including a subagent question being relayed — the main agent first searches the decision records, `locked` decisions and `Workflow Resume` by decision id, file name and code path; a hit is cited and acted on instead of asked again, and only new evidence that contradicts the recorded decision is escalated, with that evidence attached.
5. **Plan one milestone at a time.** Write later plans against the actual resulting code, and execute with the current plan and necessary references in a separate context after the user's choice. Record verification and completion before resuming the next milestone; execution authorization does not silently carry over between milestones. Before locking the DAG, report its shape: if the milestones form one pure chain with no concurrent branches, say out loud that execution can only be serial and that total time is roughly the sum of all milestones, and offer the user a chance to re-slice. Do not decide on their behalf whether to parallelize. When a milestone closes, move any decision rationale that only process documents carry — especially negative decisions such as "this part stays unchanged" or "this path is exempt" — into an ADR or glossary entry with its source id, and re-check once all milestones are done, so the next requirement neither re-asks nor re-learns it.

These rules target redundant work, not a promised token reduction. Writing files does not erase tool-call history; subagents still consume tokens and may inherit context. Cache pricing and host compaction affect cost, while cheaper models may need more retries. Self-contained plan headers help a new session resume, but do not guarantee a less capable model can execute correctly. Compare all-agent usage, actual cost, rework and acceptance results before claiming savings. The existing full milestone review, execution-mode default and verification recommendations remain unchanged.

---

# Installation — pick one of two paths

The same repository supports both CC Switch and Claude Code Plugin. CC Switch discovers skills through the `skills/` directory; Claude Code reads the manifests in `.claude-plugin/`.

## Path 1: CC Switch

CC Switch looks for skills in a fixed directory layout, so the `skills/` level in this repo exists for you.

**1. Add this repository**

| Field | Value |
| --- | --- |
| Repository URL | <https://github.com/zhy-ty/grill-plan-execute> |
| Owner | `zhy-ty` |
| Name | `grill-plan-execute` |
| Branch | `main` |
| Subdirectory | `skills` |

Then install `grill-plan-execute`.

**2. Install the dependencies by hand (this step cannot be skipped)**

CC Switch **does not read the `dependencies` field in `plugin.json`**. You must add each upstream repository and install the skills it hosts. Add these three, all with Subdirectory `skills`:

| Repository URL | Owner | Name | Branch | Subdirectory |
| --- | --- | --- | --- | --- |
| <https://github.com/mattpocock/skills> | `mattpocock` | `skills` | `main` | `skills` |
| <https://github.com/tmdgusya/engineering-discipline> | `tmdgusya` | `engineering-discipline` | `main` | `skills` |
| <https://github.com/obra/superpowers> | `obra` | `superpowers` | `main` | `skills` |

Then pick only the skills this workflow needs (paths are relative to each repository root):

| Repository (Subdirectory: `skills`) | Required | Only needed for Phase 5 | Optional |
| --- | --- | --- | --- |
| [mattpocock/skills](https://github.com/mattpocock/skills) | `grilling` (`skills/productivity/grilling`), `domain-modeling` (`skills/engineering/domain-modeling`) | — | `grill-with-docs` (`skills/engineering/grill-with-docs`) |
| [tmdgusya/engineering-discipline](https://github.com/tmdgusya/engineering-discipline) | `milestone-planning` (`skills/milestone-planning`); `clarification` for template/scoring access, unless already locally readable | — | `plan-crafting`, `long-run`, `run-plan`, `review-work` |
| [obra/superpowers](https://github.com/obra/superpowers) | `writing-plans` (`skills/writing-plans`) | `subagent-driven-development`, `executing-plans`, `finishing-a-development-branch` | `requesting-code-review`, `receiving-code-review`, `test-driven-development`, `using-git-worktrees`, `verification-before-completion`, `dispatching-parallel-agents` |

For a fresh CC Switch setup with no readable upstream source, select **5 skills**: the four workflow skills plus `clarification` for its format resource; select 8 with the three Phase 5 execution/finishing skills. If the `clarification` template and scoring rules are already readable in a local upstream checkout, four installed workflow skills suffice. No extra `clarification` interview runs. `grill-with-docs` and the other optional destinations remain optional; missing items are reported, not simulated.

Step 0 checks skill availability when needed, and checks template/scoring access before the full workflow starts interviewing. Missing resources stop the workflow with installation guidance rather than guessed fields or scores.

> Note: after a CC Switch install these skills may not carry a plugin prefix. `SKILL.md` uses Claude Code Plugin's `plugin:skill` form; if your CC Switch exposes bare skill names, just drop the prefix — it is the same skill.


## Path 2: Claude Code Plugin

Dependencies are pulled automatically, but **the three upstream marketplaces must be configured first** — that part cannot be skipped.

**1. Add the three marketplaces that host the dependencies**

```bash
claude plugin marketplace add mattpocock/skills
claude plugin marketplace add tmdgusya/engineering-discipline
claude plugin marketplace add obra/superpowers
```

Their marketplace names are `mattpocock`, `engineering-discipline` and `superpowers-dev`. `allowCrossMarketplaceDependenciesOn` in this repo's `marketplace.json` already declares those three. It also allows `claude-plugins-official`, but this workflow declares no direct dependency from that marketplace.

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

Plugin dependencies are all-or-nothing: you get every skill the upstream plugin exposes, not just the four this workflow calls.

## Sibling skills referenced by upstream

Upstream skill bodies mention sibling skills from their own repositories. All of them really exist once the dependency plugins are installed — call them by namespace, do not treat them as missing and do not improvise their behaviour from memory:

| Mentioned as | Actual location |
| --- | --- |
| `clarification`, `plan-crafting`, `long-run`, `run-plan`, `review-work` | `engineering-discipline:<same name>` |
| `test-driven-development`, `using-git-worktrees`, `requesting-code-review`, `receiving-code-review`, `verification-before-completion`, `dispatching-parallel-agents`, `brainstorming` | `superpowers:<same name>` |

## Extending the workflow with another skill

Yes — this is the intended way to grow the project. `SKILL.md` is a pure orchestration layer, so you can insert another external skill at any phase, and the zero-copy rule still holds: **never paste that skill's body text into this repo**, only register the reference and the call site.

Keep `SKILL.md` focused on runtime instructions: its body enters context when invoked and can persist until the host compacts or replaces that context. Install steps, the dependency table, this guide and the sibling table deliberately live here instead; repeated-input cost depends on the host and caching.

1. `SKILL.md` — one line in its compact `Dependencies` list (namespace + role), the call site and its **completion criterion** in the right phase, what it consumes from upstream and what it leaves for downstream, and the `Order Matters` diagram if the shape changes. Add it to Step 0 as well if it is required.
2. `plugin.json` — only if the skill lives in a plugin that is not already a dependency. Skills from `mattpocock-skills`, `engineering-discipline` or `superpowers` need no change, because the plugin path installs those plugins whole.
3. `marketplace.json` — only if you introduce a **new marketplace**: add its name to `allowCrossMarketplaceDependenciesOn`.
4. `README.md` and `README.zh.md` — the dependency table, the CC Switch per-repository table above, the `marketplace add` commands, and the Granularity discussion if a new plugin adds context overhead.
5. `plugin.json`'s `version` — a new *required* dependency changes what users must install, so bump at least a minor.

The one thing to think about before adding a required skill: the plugin path cannot cherry-pick, so a required dependency from a fresh repository forces every user to install that whole plugin. Prefer optional call sites when the skill only helps in some situations.

## Versioning and updates

- **Dependency content updates**: this repo holds no copies of the dependencies, so once upstream releases a new version you get it by running `claude plugin update <plugin>@<marketplace>` (CC Switch users re-sync the repository). **Nothing needs to change in this repo.**
- **This repo's version number**: bump for workflow or installation-contract changes; tighten `dependencies[].version` when upstream compatibility requires it. Version 2.5.0 adds the environment document as a single cross-session source of setup facts, decision `locked` / `deferred-with-criteria` statuses that adversarial review may not override, checkpoint handovers that state their own boundary, a proactive resume for the next milestone, the pre-question decision-ledger lookup, a serial-chain warning before DAG lock, source-based credential recording inside a process artifact that is never committed, and a close-out pass that moves decision rationale out of process documents into ADRs and the glossary. Version 2.4.0 makes the previously implicit `clarification` format resource an explicit prerequisite, adds bounded interviews and checkpoint resumes, and assigns user interaction to the main agent. Neither release adds a plugin or copies upstream content.
- The current `dependencies` use caret (`^`) ranges: they follow features and fixes within the same major version, and deliberately do not follow a major upgrade.

## License

MIT — see [LICENSE](./LICENSE). Copyright for the dependency skills belongs to their respective authors; this repo claims no rights over them.

---
name: grill-plan-execute
description: 从模糊想法到可执行计划的端到端编排工作流。先深度访谈并沉淀领域文档，再询问用户业务复杂度：复杂则走含对抗性评审的里程碑规划、逐里程碑产出实现计划，不复杂则直接产出一份整体计划。本 skill 只做编排，不重复依赖 skill 的规则内容。
when_to_use: 用户有一个尚未澄清的产品需求或多日技术任务，希望系统地走完"澄清 → 里程碑规划 → 实现计划"的完整链路；用户说"帮我从零规划这个功能"或"先想清楚再拆里程碑再排计划"。
disable-model-invocation: true
---

# Grill, Plan, Execute

## Overview

一个编排层 skill。它把工作流固定为四个必需的外部 skill（`grilling`、`domain-modeling`、`milestone-planning`、`writing-plans`，来自三个不同仓库），加若干可选项：访谈阶段可用 `grill-with-docs` 一次顶两次，执行阶段可移交 `superpowers` 的执行与收尾 skill。本 skill 不定义任何访谈、里程碑或计划规则——那些规则只存在于被依赖的 skill 里。

流程形状：访谈收敛后先问用户任务复杂不复杂，复杂才走里程碑规划与对抗性评审，然后产出实现计划并移交执行。

**本 skill 不含任何依赖 skill 的内容副本。** 依赖的实际文本由用户从原始仓库获取，因此上游更新会直接生效。

## Dependencies（前置条件，必须先装好）

按命名空间引用，本仓库不含这些 skill 的文件：

| 命名空间 | 必需？ | 作用 | 所属插件 / marketplace | 原始仓库 |
| --- | --- | --- | --- | --- |
| `mattpocock-skills:grilling` | **必需** | 决策树式多轮访谈，产出共识理解 | `mattpocock-skills@mattpocock` (^1.3.1) | [mattpocock/skills](https://github.com/mattpocock/skills) |
| `mattpocock-skills:domain-modeling` | **必需** | 实时维护领域词汇与决策记录 | 同上 | 同上 |
| `engineering-discipline:milestone-planning` | **必需**（复杂路径） | 并行探索代码 → 起草里程碑 DAG → 对抗性评审循环 | `engineering-discipline@engineering-discipline` (^1.7.0) | [tmdgusya/engineering-discipline](https://github.com/tmdgusya/engineering-discipline) |
| `superpowers:writing-plans` | **必需** | 面向零上下文工程师的详细实现计划 | `superpowers@superpowers-dev` (^7.0.0) | [obra/superpowers](https://github.com/obra/superpowers) |
| `mattpocock-skills:grill-with-docs` | 可选 | 聚合入口，一次调用同时跑访谈与文档；缺省时由本 skill 依次调上面两个 | 同 `grilling` | 同上 |
| `superpowers:subagent-driven-development`、`superpowers:executing-plans`、`superpowers:finishing-a-development-branch` | 可选（仅 Phase 5 移交目标） | 执行与收尾 | 同 `writing-plans`，不额外增加依赖 | 同上 |

依赖解析因分发方式而异：

- **Claude Code Plugin**：`plugin.json` 已声明这三条依赖，安装本插件时由 Claude Code 自动从各自 marketplace 拉取，用户无需手动装。前提是那三个 marketplace 已在用户机器上配置（见 README 第二节）。注意依赖是按**插件**整体安装的，不止这 4 个 skill，常驻上下文成本约 2.9k token，明细见 README。
- **CC Switch**：**不会解析依赖**。用户必须自己在 CC Switch 里添加并安装上面三个仓库，再按需勾选具体 skill，否则本 skill 运行到第一个依赖就断。

### Step 0: 自检

开始前确认四个**必需**命名空间可用：`grilling`、`domain-modeling`、`writing-plans`，以及（走复杂路径时）`milestone-planning`。任一必需项缺失就停下来，按用户所用的分发方式给出补齐指引，不要凭记忆模拟缺失 skill 的行为。

可选项缺失不阻塞：`grill-with-docs` 缺失走下面的等价退路，Phase 5 的执行类 skill 缺失则把计划交给用户自行执行。

## Workflow

### Phase 1: 澄清与领域建模

跑 `mattpocock-skills:grilling` 对用户做多轮结构化提问，同时用 `mattpocock-skills:domain-modeling` 把确认下来的术语与决策实时写进词汇表和决策记录。

若 `grill-with-docs` 可用，优先调它一次即可——它就是上述两个 skill 的聚合入口。注意它自己标了 `disable-model-invocation: true`，部分宿主环境会拒绝由 agent 触发，此时退回上面两个。

只有在用户明确只要访谈、不要写文档时，才只调 `grilling`。

**完成判据：** 依赖 skill 自己判定访谈收敛（设计树没有遗留的隐性假设），且用户确认达成共识。产出：词汇表与决策记录文件。

### Phase 2: 复杂度评估

向用户提问：

> "根据当前理解，这个任务的业务逻辑是否复杂？判断标准：预计实现步骤超过 8 个，或需要修改/理解超过 3 个核心模块，就算复杂。复杂的话我会先做深入的代码探索和对抗性评审来查缺补漏，再拆里程碑；不复杂就直接出一份实现计划。"

- 用户回答 **不复杂** → 跳到 Phase 4
- 用户回答 **复杂** → 进入 Phase 3

这个判断权在用户，不要替他决定。可以用 `engineering-discipline:clarification` 之类的提问方式辅助，但最终以用户答复为准。

### Phase 3: 里程碑规划（仅复杂路径）

调用 `engineering-discipline:milestone-planning`，**完整执行它的流程**，不裁剪：并行探索 → 起草 DAG → 独立批评者对抗性评审 → 修订直到无阻塞项。

把 Phase 1 的共识理解、词汇表和决策记录作为输入交给它。

对抗性评审的价值在于：当你自己把握不住需求时，让独立批评者基于代码事实攻击这份划分。所以这一步不能压缩成"顺手拆几个里程碑"。

**完成判据：** 里程碑 DAG 通过评审且用户确认锁定。产出：里程碑划分 + 探索阶段得到的代码事实。

### Phase 4: 生成执行计划

**复杂路径**：对 Phase 3 锁定的每个里程碑，调用 `superpowers:writing-plans` 一次。

顺序：按里程碑依赖顺序，从第一个开始，做完一个再下一个，不要把多个里程碑合成一份计划。

**不复杂路径**：跳过 Phase 3，直接对整体任务调用一次 `superpowers:writing-plans`，输入用 Phase 1 的共识理解与文档。

两条路径每次调用的输入：

- 范围与验收目标（整个任务，或该里程碑）
- Phase 1 产出的术语表与决策记录（`GLOSSARY.md`、ADR）
- Phase 3 产出的代码事实与评审结论（`Problem Brief`、`Exploration Digest`、`Critique Resolution Log`）——仅复杂路径有这部分

要求每份计划在文档头部固定三段，便于后续会话零上下文接手：

- **Domain Context** — 本里程碑涉及的核心术语
- **Key Decisions** — 已经定下的设计决策，含出处
- **Code Facts** — 探索得到的关键事实，含文件路径

复杂路径下一个里程碑一份计划，它们会落在同一目录：给每份计划带上里程碑编号以免互相覆盖（具体保存路径由 `writing-plans` 决定）。

**产出：** 不复杂路径一份实现计划；复杂路径每个里程碑各一份。

### Phase 5: 移交执行

把选择权交给用户，不要自行开始实现。

`writing-plans` 自己会问用户选哪种执行方式，它当前给的是两个选项，对应：

| 用户选择 | 转交 | 特点 |
| --- | --- | --- |
| **Subagent-driven** | `superpowers:subagent-driven-development` | 每个任务派新 subagent 实现、新 reviewer 检查，最后整分支评审；最彻底，每任务与每轮评审各吃一份新上下文 |
| **Native** | `superpowers:executing-plans` | 本会话由当前 agent 实现，结束时派一个独立 reviewer 看整分支；最省最快，代价是前期没有独立评审 |

两条路做完都进入 `superpowers:finishing-a-development-branch`，由它给 merge / PR / cleanup 选项。

**里程碑之间不要串用执行方式**：先完成当前里程碑的执行与收尾，再回到 Phase 4 要下一份计划。

## Sibling skills

上游 skill 的正文会提到自己仓库里的兄弟 skill。它们现在都随依赖插件真实存在，按命名空间直接调用即可，不要当成缺失项，也不要凭记忆复述它们的行为：

| 被提到时 | 实际位置 |
| --- | --- |
| `clarification`、`plan-crafting`、`long-run`、`run-plan`、`review-work` | `engineering-discipline:<同名>` |
| `test-driven-development`、`using-git-worktrees`、`requesting-code-review`、`receiving-code-review`、`verification-before-completion`、`dispatching-parallel-agents`、`brainstorming` | `superpowers:<同名>` |


## Order Matters

```
Phase 1: mattpocock-skills:grilling + mattpocock-skills:domain-modeling
         （装了 grill-with-docs 可用一次调用代替）
    ↓  需求收敛、用户确认
Phase 2: 询问用户业务是否复杂（>8 步骤 或 >3 核心模块）
    ├─ 不复杂 ────────────────────────────┐
    └─ 复杂 → Phase 3                     │
Phase 3: engineering-discipline:milestone-planning
         （完整流程：探索 → 起草 → 对抗性评审 → 修订到 dry）
         ↓  里程碑 DAG 锁定               │
Phase 4: superpowers:writing-plans        ←┘
         复杂路径：每里程碑一份；不复杂：整体一份
    ↓
Phase 5: 移交执行（复杂路径下每个里程碑走完再取下一个）
    ├─ Subagent-driven → superpowers:subagent-driven-development
    └─ Native          → superpowers:executing-plans
         ↓
    superpowers:finishing-a-development-branch（merge / PR / cleanup）
```

阶段严格串行：上游阶段未达完成判据就进入下一阶段，会让里程碑建立在未被确认的需求上。唯一例外是 Phase 2 的分支，它决定 Phase 3 是否执行。

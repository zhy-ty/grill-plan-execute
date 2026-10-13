---
name: grill-plan-execute
description: 从模糊想法到分里程碑执行计划的端到端编排工作流。先用深度访谈加领域建模收敛需求，再生成经过对抗性评审的里程碑，最后为每个里程碑产出详细执行计划。本 skill 只做编排，不重复依赖 skill 的规则内容。
when_to_use: 用户有一个尚未澄清的产品需求或多日技术任务，希望系统地走完"澄清 → 里程碑规划 → 实现计划"的完整链路；用户说"帮我从零规划这个功能"或"先想清楚再拆里程碑再排计划"。
disable-model-invocation: true
---

# Grill, Plan, Execute

## Overview

一个编排层 skill。它把三个外部仓库的 5 个 skill 串成一条固定顺序的工作流，本身不定义任何访谈、里程碑或计划规则——那些规则只存在于被依赖的 skill 里。

**本 skill 不含任何依赖 skill 的内容副本。** 依赖的实际文本由用户从原始仓库获取，因此上游更新会直接生效。

## Dependencies（前置条件，必须先装好）

按命名空间引用，本仓库不含这些 skill 的文件：

| 命名空间 | 作用 | 所属插件 / marketplace | 原始仓库 |
| --- | --- | --- | --- |
| `mattpocock-skills:grill-with-docs` | 聚合入口：一次调用同时跑访谈与文档产出 | `mattpocock-skills@mattpocock` (^1.3.1) | [mattpocock/skills](https://github.com/mattpocock/skills) |
| `mattpocock-skills:grilling` | 决策树式多轮访谈，产出共识理解 | 同上 | 同上 |
| `mattpocock-skills:domain-modeling` | 实时维护领域词汇与决策记录 | 同上 | 同上 |
| `engineering-discipline:milestone-planning` | 并行探索代码 → 起草里程碑 DAG → 对抗性评审循环 | `engineering-discipline@engineering-discipline` (^1.7.0) | [tmdgusya/engineering-discipline](https://github.com/tmdgusya/engineering-discipline) |
| `superpowers:writing-plans` | 面向零上下文工程师的详细实现计划 | `superpowers@superpowers-dev` (^7.0.0) | [obra/superpowers](https://github.com/obra/superpowers) |

依赖解析因分发方式而异：

- **Claude Code Plugin**：`plugin.json` 已声明这三条依赖，安装本插件时由 Claude Code 自动从各自 marketplace 拉取，用户无需手动装。前提是那三个 marketplace 已在用户机器上配置（见 README 第二节）。
- **CC Switch**：**不会解析依赖**。用户必须自己在 CC Switch 里添加并安装上面三个仓库，否则本 skill 运行到第一个依赖就断。

### Step 0: 自检

开始前确认上述五个命名空间可用。任一缺失就停下来，按用户所用的分发方式给出补齐指引，不要凭记忆模拟缺失 skill 的行为。

## Workflow

### Phase 1: 澄清与领域建模

调用 `mattpocock-skills:grill-with-docs`（它的做法就是把 `grilling` 与 `domain-modeling` 各调一次）。

只有在用户明确只要访谈、不要写文档时，才单独调 `mattpocock-skills:grilling`。

**完成判据：** 依赖 skill 自己判定访谈收敛（设计树没有遗留的隐性假设），且用户确认达成共识。产出：词汇表与决策记录文件。

### Phase 2: 里程碑规划

调用 `engineering-discipline:milestone-planning`，**完整执行它的流程**，不裁剪：并行探索 → 起草 DAG → 独立批评者对抗性评审 → 修订直到无阻塞项。

把 Phase 1 的共识理解、词汇表和决策记录作为输入交给它。

**完成判据：** 里程碑 DAG 通过评审且用户确认锁定。产出：里程碑划分 + 探索阶段得到的代码事实。

### Phase 3: 逐里程碑生成执行计划

对 Phase 2 锁定的每个里程碑，调用 `superpowers:writing-plans` 一次。

顺序：按里程碑依赖顺序，从第一个开始，做完一个再下一个，不要把多个里程碑合成一份计划。

每次调用都带上：该里程碑的范围与验收目标、Phase 1 的术语与决策、Phase 2 给出的代码事实。

**产出：** 每个里程碑一份详细实现计划（保存路径由 `writing-plans` 自己决定）。

### Phase 4: 移交执行

把选择权交给用户，不要自行开始实现。

`writing-plans` 会给出执行方式选项。若用户已装 `superpowers`，它同时提供 `subagent-driven-development` 与 `executing-plans` 两条执行路径，可以引导过去；未安装则把计划文件路径交给用户即可。

## Order Matters

```
Phase 1: mattpocock-skills:grill-with-docs  (grilling + domain-modeling)
    ↓  需求收敛、用户确认
Phase 2: engineering-discipline:milestone-planning  (完整流程，含对抗性评审)
    ↓  里程碑 DAG 锁定
Phase 3: superpowers:writing-plans  × 每个里程碑
    ↓
Phase 4: 移交执行
```

阶段严格串行：上游阶段未达完成判据就进入下一阶段，会让里程碑建立在未被确认的需求上。

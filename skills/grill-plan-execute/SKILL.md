---
name: grill-plan-execute
description: Orchestration-only workflow from a vague idea to executable plans — grill the user, model the domain, ask how complex the work is, then milestone-plan (complex) or plan directly, and hand execution to a fresh session with model tiering. Chains external skills and copies none of their content. 纯编排工作流：访谈澄清 → 领域建模 → 复杂度确认 → 里程碑规划（复杂路径）或直接出计划 → 移交新会话执行、模型分层。只串联外部 skill，不含它们的内容副本。
when_to_use: The user has an unclarified requirement or multi-day task and wants the whole "clarify → milestones → implementation plan → handoff" chain. 用户有尚未澄清的需求或多日技术任务，想走完"澄清 → 里程碑 → 实现计划 → 移交"整条链路。
disable-model-invocation: true
---

# Grill, Plan, Execute

## Overview

只编排外部 skill，定义阶段边界、完成判据与交接契约，不复制上游正文。流程：访谈 → 用户确认复杂度 → 复杂才走里程碑规划与对抗性评审 → 逐里程碑出计划 → 用户选择后移交执行。按检查点恢复规划，不依赖完整访谈历史。

安装、依赖明细、扩展指南与兄弟 skill 对照表见 README；本文件只放运行时规则。

## Output language（面向用户的语言）

本文件的中文是写给 agent 的指令，不是给用户看的台词。**所有面向用户的输出——提问、选项标签、阶段报告、缺失依赖的补齐指引——一律跟随用户当前使用的语言**，用户用英文就整句英文，不要中英混排。

下文给出提问原文处都并列 EN / 中文 两版：按用户语言取其中一版发给用户，**不要把另一版一起显示**。

## Token 纪律

- **路径交接，按需读取。** 传任务范围、必要决策和产物路径，不重贴全文或复述依赖规则；不能为了短摘要省略阻塞信息。
- **按主题批量落盘。** 一个主题确认后统一更新术语、决策与假设；转阶段或结束会话前写完，不随每条回答重写文件。
- **隔离重活。** 探索、批评、写计划交给能加载所需 skill 的 subagent，给最小必要上下文，产物落盘、回摘要；不要假设宿主不会继承历史。
- **计划 just-in-time。** 一次只写当前里程碑，后续根据实际代码状态规划。
- **检查点续接。** 按下节切换上下文；写文件本身不会清除历史，subagent 的输入输出同样消耗 token。

## Dependencies

四个必需项，按命名空间引用，本仓库不含这些 skill 的文件：

- `mattpocock-skills:grilling` — 决策树式多轮访谈，产出共识理解
- `mattpocock-skills:domain-modeling` — 实时维护领域词汇与决策记录
- `engineering-discipline:milestone-planning` — 并行探索 → 里程碑 DAG → 对抗性评审（仅复杂路径）
- `superpowers:writing-plans` — 面向零上下文工程师的详细实现计划

另需**格式资源**：`engineering-discipline:clarification` 的 Context Brief 模板与 Complexity Assessment 评分规则。只读取，不运行其访谈；资源缺失会阻塞完整工作流，不能把它当成可选项。

可选项（缺失不阻塞，只告知用户）：

- `mattpocock-skills:grill-with-docs` — 一次调用依次跑上面前两个
- `superpowers:subagent-driven-development`、`superpowers:executing-plans`、`superpowers:finishing-a-development-branch` — Phase 5 移交目标
- `superpowers:verification-before-completion`、`engineering-discipline:review-work` — Phase 5 评审兜底，强烈建议；执行端用便宜模型时实际必配

Plugin 整包安装，CC Switch 需手工安装；命名空间不带插件前缀时使用已注册的裸名。安装与版本见 README。

### Step 0: 自检

从宿主索引确认当前所需 skill 可用，不提前加载后续阶段正文。完整工作流开始访谈前，还要在已安装插件或本地上游目录定位 `clarification`，只读取 Context Brief 模板及评分规则，记录来源路径。缺必需 skill 或资源就停下，按分发方式给补齐指引；**不猜格式、不编分数、不凭记忆模拟**。

可选项缺失不阻塞：`grill-with-docs` 缺失就调 `grilling` + `domain-modeling`；执行类 skill 缺失就交计划给用户自行执行；评审兜底项缺失则告知用户"缺少独立验证，建议人工复核"。

### 上下文断点与恢复

在 Context Brief 末尾维护简短的 **Workflow Resume** 续接索引，批量更新，不另建平行进度表：

- 下一阶段、用户确认的复杂度、当前里程碑、DAG 是否已锁定、执行方式是否已选择；未确认项明确标为待定。
- 模板来源、词汇表、决策、当前草稿／就绪计划及状态、复杂路径 `state.md` 等必要路径；里程碑状态以上游 `state.md` 为准，不复制整份 DAG 或评审日志。
- 未决阻塞项、用户接受的非阻塞假设及其复核时点；详情引用原章节。
- 代码事实的版本基线（Git 项目记 HEAD 与未提交改动范围）和当前里程碑的验证结果／证据路径。

**断点：** Phase 1 共识与 Phase 2 选择完成后、Phase 3 的 DAG 确认锁定后、每个里程碑执行与收尾完成后，更新索引再用新上下文进入下一阶段。宿主不能开新会话时用实际可用的压缩能力；两者都没有就给出续接指令由用户开新会话，除非用户明确原地继续。不要声称写文件已清空历史。

**恢复：** 只携带本 skill 的调用入口、Context Brief 路径与下一阶段；读取续接索引及本阶段所需文件，不重读全部访谈或批评记录。核对阶段完成状态、相关代码变更和到期假设；未提交改动也要核对，不能只比较 HEAD。事实失效时只补查受影响范围；决策需变更则请用户重新确认受影响项，不重跑整个访谈或重复询问未变化的复杂度。续接信息不构成开始实现的授权。

## Workflow

### Phase 1: 澄清与领域建模

调用 `mattpocock-skills:grilling` 访谈，配合 `mattpocock-skills:domain-modeling` 按主题批量更新术语与决策。可用 `grill-with-docs` 作为入口，宿主拒绝触发时退回两个独立调用。用户明确只要访谈、不写文档时，只需 `grilling`，不检查模板资源，也不继续 Phase 2–5。

把本阶段停止条件作为访谈边界交给 `grilling`：**目标、范围／非目标、约束与验收标准已确认，且没有阻塞当前规划的问题。** 不为穷尽设计树无限追问；能查证的环境事实交给探索，不让用户回答，也不复述探索过程。

非阻塞问题经用户接受后记为显式假设，并注明受影响里程碑或最迟复核时点；规划或执行受影响工作前必须复核。会影响安全、权限、数据一致性、不可逆迁移或使范围／验收无法确定的问题仍是阻塞项，不能靠延期或默认答案放行。

**交付 Context Brief**：按 Step 0 实际读取的上游模板与评分规则写，不额外运行 `clarification` 访谈，也不在本仓库复制模板：

```
docs/engineering-discipline/context/YYYY-MM-DD-<topic>-brief.md
```

`Technical Context` 记录代码事实、文件路径与版本基线，供后续从已知事实起步；`Complexity Assessment` 按上游规则填，作为 Phase 2 推荐依据。Phase 3 消费本文件，Phase 4 的 `Spec:` 指向它。**不要提前造 `Problem Brief`**，它由 `milestone-planning` 探索后生成。

**完成判据：** 上述停止条件满足，用户确认需求与保留的假设，Context Brief、词汇表和决策记录已写完。随后进行 Phase 2，不在这里重复访谈。

### Phase 2: 复杂度评估

用宿主的结构化提问工具（如 `AskUserQuestion`）以**选项式**问用户，两个选项 `Complex / 复杂` 与 `Not complex / 不复杂`；宿主没有该工具时退化为纯文本 Y/N 提问。提问时把 Context Brief 里 `Complexity Assessment` 的分数与 verdict 一并报给用户作为你的推荐，下面的判断标准只是给用户的解释口径。提问正文按 Output language 一节取用户语言的那一版，原文如下：

> **EN:** "Based on what we've settled so far — is the business logic here complex? Rule of thumb: complex means more than 8 implementation steps, or more than 3 core modules to modify or understand. If it's complex, I'll run a deep code exploration plus adversarial review to catch gaps before splitting milestones; if not, I'll go straight to one implementation plan."
>
> **中文：** "根据当前理解，这个任务的业务逻辑是否复杂？判断标准：预计实现步骤超过 8 个，或需要修改/理解超过 3 个核心模块，就算复杂。复杂的话我会先做深入的代码探索和对抗性评审来查缺补漏，再拆里程碑；不复杂就直接出一份实现计划。"

- 用户选 **Not complex / 不复杂** → 跳到 Phase 4
- 用户选 **Complex / 复杂** → 进入 Phase 3

这个判断权在用户，不要替他决定，也不要因为"看起来简单"就跳过这一步。将用户选择与下一阶段写入 Workflow Resume，按断点规则续接。

### Phase 3: 里程碑规划（仅复杂路径）

调用 `engineering-discipline:milestone-planning`，**完整执行它的流程**，不裁剪：并行探索 → 两副骨架起草 → 独立批评者并行对抗评审 → 修订直到无 Blocking（唯一偏离是下面的轮次上限，它限制循环次数，不取消评审）。这一步不能压缩成"顺手拆几个里程碑"——评审的意义正是让独立批评者基于代码事实攻击这份划分。

把 Phase 1 的 Context Brief、词汇表、决策记录以**文件路径**交给它，不要复述全文。它自己的 Problem Brief 与 Exploration Digest 由它探索后 compose，不要替它写。

**评审产物落盘，只回摘要。** 上游已规定批评者产物存到 `docs/engineering-discipline/harness/<slug>/planning/critiques/round-N-<lens>.md`、逐字记录汇总进同目录的 `resolution-log.md`。据此要求每个批评者 subagent：完整 findings 写进自己那个文件，**回主会话的正文压到 10 行以内**（各严重度计数 + 每条 Blocking 一句话 + 文件路径）。resolution log 引用路径，不把全文抄进对话。这不削弱评审——逐字记录一条不少，只是不进主上下文。

**评审轮次上限：** 上游允许 3 轮，本工作流收紧到默认最多 2 轮修订（实践中大多在第二轮前收敛）；每轮 = 4–6 个批评者 + 修订 + 再评审，是整个流程最烧 token 的部分。两轮后仍有 Blocking 就**不要继续自动修订**：把未决分歧整理成清单（批评者论点、作者回应、未解决项，各项指向 critiques 文件路径）交用户人工裁决，拍板后锁定 DAG 进入 Phase 4。省轮次靠把交接信息给足，不靠削弱评审。

**完成判据：** 里程碑 DAG 通过评审且用户确认锁定；或到达轮次上限后由用户裁决遗留分歧并经用户确认。产出：`harness/<slug>/` 下 `state.md`、`milestones/M*.md`、`planning/problem-brief.md`、`planning/resolution-log.md` 的路径；更新 Workflow Resume，按断点规则进入 Phase 4。

### Phase 4: 生成执行计划——一次只做一个里程碑

按依赖顺序选择一个就绪里程碑；不复杂路径处理整体任务。先核对到期假设、相关代码变化与验收目标，再写计划，不提前写后续里程碑。

**委派契约：** 让能加载 `superpowers:writing-plans` 的 subagent 按上游步骤只写当前范围的计划；给出范围、验收目标及以下路径，不转发访谈历史。不支持此委派时，在新规划上下文由主代理调用该 skill。两者都不得自行实现、合并或替用户选择执行方式。输入文件：

- 当前里程碑的 `milestones/Mk-<name>.md`（不复杂路径没有这一项）
- Phase 1 的 Context Brief —— 填进计划头部的 `Spec:` 字段
- 复杂路径再加 `planning/problem-brief.md`（含 Exploration Digest）与 `planning/resolution-log.md`
- 词汇表与决策记录文件

子代理返回状态，不把计划正文贴回主会话：

- **`needs-input`**：上游需要用户答复时，返回具体问题、必要上下文／选项及已有草稿路径，暂停。主代理转达并记录答案，再从草稿继续，不重新访谈或从头生成。
- **`plan-ready`**：计划路径 + ≤5 行摘要（目标、任务数、验收命令）；上游提供的执行方式选项另附，交 Phase 5 由主代理询问。
- **`blocked`**：缺依赖、输入不可读或契约不兼容时，返回缺项与已有草稿路径，不猜测补齐，也不伪报完成。阻塞信息和用户问题不受摘要行数限制。

任何暂停前，主代理都要将状态、已有草稿路径和待解问题写入 Workflow Resume；解除阻塞或重新委派后从该草稿继续。草稿不能当作就绪计划执行。

主代理核对实际计划的范围、验收条件与交接字段后才进入 Phase 5；收到摘要不等于计划已通过检查。

保存路径由 `writing-plans` 决定（默认 `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`）；复杂路径下文件名带里程碑编号，避免覆盖。

计划头部保留三段自足内容，供不带访谈历史的会话接手：

- **Domain Context** — 本里程碑涉及的核心术语
- **Key Decisions** — 已经定下的设计决策，含出处
- **Code Facts** — 探索得到的关键事实，含文件路径

计划就绪后更新 Workflow Resume 中的计划路径，标明是否仍待执行方式确认。执行默认新会话，只带当前计划、必要引用及用户已确认的执行选择，不带完整规划历史；会话能力不足时按断点规则处理。

### Phase 5: 移交执行

**用户交互由主代理负责。** 依据 `writing-plans` 当前版本的执行选项，用用户语言解释代价并询问；子代理返回的问题由主代理转达，不让它自行跨入执行。选择已经通过主代理确认并记录在当前里程碑的 Workflow Resume 中时，不重复问；未确认则等待，不能自行开始实现。

保留选项名 `Subagent-driven` / `Native`：分别转交 `superpowers:subagent-driven-development` / `superpowers:executing-plans`；若上游版本不再提供对应选项，先说明契约差异，不假装兼容。两条路做完都进入 `superpowers:finishing-a-development-branch`，由它给 merge / PR / cleanup 选项。

**模型分层（默认建议）：** 强模型规划与评审，便宜模型按计划执行。自足计划降低对访谈历史的依赖，不保证低推理能力模型能正确完成；根据验证与返工情况判断是否需强模型执行。执行端配验证与独立评审：

- `superpowers:verification-before-completion` —— 执行者自证：没有新鲜的验证证据（当场跑测试/构建并读输出）不许声称完成
- `engineering-discipline:review-work` —— 独立他证：只用计划文档 + 代码库判定实现是否符合计划，结论落成文档；由**强模型**跑，天然与"执行开新会话"对齐

这两个 skill 缺失时不阻塞，但要告知用户"缺少独立验证，建议人工复核"。

**搭配默认：** `Subagent-driven` + 便宜模型执行 + 强模型评审——每任务一份新上下文，天然就是模型与上下文的断点。计划薄、任务小时可退到 `Native` + 便宜模型执行 + 收尾整分支强模型评审；意外决策多、便宜模型覆盖不了时才全程强模型。

本里程碑执行与收尾都走完，核对验证证据并更新上游状态与 Workflow Resume；仍有未完成里程碑才按断点规则回 Phase 4，否则结束。下一里程碑的执行选择重新标为待定，不跨里程碑沿用授权，也不在当前里程碑中途串用执行方式。

## Order Matters

```
Phase 1  grilling + domain-modeling → Context Brief（含续接索引）
Phase 2  用户确认复杂度 → 检查点／新上下文
   ├─ 不复杂 ─────────────────────────────────┐
   └─ 复杂 → Phase 3  milestone-planning       │
             完整探索与评审（≤2 轮，超限交用户裁决）
             ↓ DAG 确认锁定 → 检查点／新上下文 │
Phase 4  writing-plans（一次一个里程碑）←───────┘
         子代理写计划；主代理处理问题与执行选择
Phase 5  用户选择后移交新执行会话
         subagent-driven-development 或 executing-plans
         → finishing-a-development-branch
         ↓ 检查点／新上下文；有下一里程碑才回 Phase 4
```

阶段严格串行，未达到完成判据不能前进；Phase 2 的用户选择决定是否执行 Phase 3。切换会话不绕过完成判据，也不自动授予执行权限。

# grill-plan-execute

[English](./README.md)

仓库地址：**<https://github.com/zhy-ty/grill-plan-execute>**

一个**只做编排**的 Claude Code skill，把三个外部仓库的 skill 串成一条工作流：先靠逼问式访谈和领域建模把需求问清楚，再和用户确认业务到底有多复杂，然后产出实现计划，并把执行移交到一个新会话——规划用强模型，执行可切便宜模型、由强模型负责评审。

本仓库只包含编排逻辑，**不含任何依赖 skill 的内容副本**——依赖正文全部来自各自原始仓库，所以上游的改进会直接触达用户，本仓库不需要跟着改。

> **关于 `obra/superpowers-skills`**：该仓库已被作者归档（2025-10 起不再更新）。本工作流需要的 `writing-plans` 现在在持续维护的 [obra/superpowers](https://github.com/obra/superpowers)（v7.x）里。如果你依赖归档仓库的那个版本，需要自行 fork 维护。

## 这个 skill 不承诺什么

**它不保证省 token。** 这里没有任何数字经过基线对照，串起来的子代理照样消耗 token，便宜模型还可能因为反复重试更贵。它真正的作用是更窄的一件事：**减少写代码时的遗漏和错误，从而避免返工。** 需求在规划前被确认，计划在授权执行前被评审，领域词汇和决策落在文档里，而不是留在一段下一个会话根本看不到的长对话里。

`zhy-ty` 平时做复杂项目需求就是按这几步手工走的：先逼问需求，再做领域建模，然后判断业务到底多复杂，拆成里程碑，写出计划，最后把执行交给一个新会话。做这个整合版 skill 只是为了把已有的习惯变成一条命令，而不是取代那几个 skill。

### zhy-ty 自己的实际感受

起因是能力问题，不是费用问题。以前一个需求从头到尾交给普通模型做，受限于它的智力，往往要到最后才看出来偏了——代码已经和原有需求对不上，而过程中没有任何环节提示过。**正是这个场景让他主动去找好用的 skill**，本仓库这套工作流就是他在用的过程中逐渐固定下来的做法。

这个过程的结果，按他自己的使用：**token 消耗量明显变多了，但返工率下降了很多**——"agent 执行到最后才跑偏"这种问题基本不再出现。这里没有任何经过基线对照的数字，请当成单个使用者的体验，而不是评测结论。

至于"高智商模型做规划、低智商模型做执行"这条推荐，是再往前一步的理想情况，不是这套工作流存在的原因：让能力更强的模型出方案，是想把错误率压得更低，再让便宜模型去执行已经过评审的计划。**它只是理论上的推荐，本仓库没有实测数据支撑**。如果上面这套流程已经能管住你的返工，不必再额外折腾模型搭配。

所以：**建议去单独了解这些源头 skill，够用的时候就只装它们。** 它们每个单用都很强，完整链路并不是永远的优选。

### 只想用其中一段

| 你的情况 | 调用 | 你会得到 |
| --- | --- | --- |
| 需求简单，自己全程能把控 | `grill-with-docs`（它本身已含 `grilling` + `domain-modeling`） | 需求被问清楚，术语表和决策记录落到文档，不进入规划阶段。 |
| 担心 agent 执行过程中跑偏 | `grill-with-docs` + `writing-plans` | 一份写给零上下文工程师的详细实现计划，这份文档就是 agent 必须沿着的轨道。 |
| 自己评估改动量较大，已经把握不住 | `grill-with-docs` + `milestone-planning` + `writing-plans` | 先做完整探索和对抗性评审、锁定里程碑 DAG，再一个里程碑一份计划。 |

第三行**就是本 skill 的流程**——整合版多出来的是 Phase 2 那道复杂度确认（它负责在第 2 行和第 3 行之间做选择），以及各阶段之间的检查点交接。如果你本来就能判断自己属于哪一行，直接调用那几个 skill 也是合理选择。

**而且这条链路还会继续变长。** `SKILL.md` 是纯编排层，以后凡是能降低错误率、或者能让流程更完整的上游 skill，都会继续插进来——这就是本项目预期的成长方式，见下文「以后还能不能往工作流里再插一个 skill」。所以更该去认识那几个源头 skill，搞清楚各自适合什么场景，**针对性地挑着用**，这才是真正省力的做法。token 上的实话是：完整链路买来的是**少返工，不是低账单**；真能把消耗压下去的，是本该串四个 skill 的场景你只调了两个。

## 依赖的外部 skill

| 命名空间 | 必需？ | 作用 | 插件 / marketplace | 原始仓库 |
| --- | --- | --- | --- | --- |
| `mattpocock-skills:grilling` | **必需** | 决策树式的多轮访谈 | `mattpocock-skills@mattpocock` ^1.3.1 | [mattpocock/skills](https://github.com/mattpocock/skills) |
| `mattpocock-skills:domain-modeling` | **必需** | 边聊边维护词汇表与决策记录 | 同上 | 同上 |
| `engineering-discipline:milestone-planning` | **必需**（复杂路径） | 并行探索代码 → 里程碑 DAG → 对抗性评审轮次 | `engineering-discipline@engineering-discipline` ^1.7.0 | [tmdgusya/engineering-discipline](https://github.com/tmdgusya/engineering-discipline) |
| `superpowers:writing-plans` | **必需** | 写给零上下文工程师的详细实现计划 | `superpowers@superpowers-dev` ^7.0.0 | [obra/superpowers](https://github.com/obra/superpowers) |
| `engineering-discipline:clarification` | **格式资源必需** | 只读取 Context Brief 模板与 Complexity Assessment 评分规则，不运行其访谈 | 同 `milestone-planning` | 同上 |
| `mattpocock-skills:grill-with-docs` | 可选 | `grilling` 与 `domain-modeling` 的便利入口，不保证减少 token | 同 `grilling` | 同上 |
| `superpowers:subagent-driven-development`、`superpowers:executing-plans`、`superpowers:finishing-a-development-branch` | 可选 | Phase 5 的移交目标 | 同 `writing-plans` | 同上 |
| `superpowers:verification-before-completion`、`engineering-discipline:review-work` | 可选（强烈建议） | Phase 5 的评审兜底：执行者自证 + 独立对照计划评审；执行端切便宜模型时实际必配，缺失只告知不阻塞 | 分别同 `writing-plans` / `milestone-planning`，不新增插件 | 同上 |

工作流调用项仍是四个，另需 `clarification` 的格式资源。Plugin 路径已通过现有 `engineering-discipline` 依赖包含资源；CC Switch 用户若没有可直接读取的本地上游源文件，还需安装 `clarification`，但不会额外运行一轮访谈。Plugin 路径安装的是**整个插件**，见[粒度：Plugin 路径的代价](#粒度plugin-路径的代价)。

## 工作流

```
Phase 1: grilling + domain-modeling → 用户确认 Context Brief
Phase 2: 用户确认复杂度 → 检查点／新上下文
    ├─ 不复杂 ──────────────────────────────┐
    └─ 复杂 → Phase 3                       │
Phase 3: milestone-planning                 │
         完整探索与对抗性评审（≤2 轮）       │
         用户锁定 DAG → 检查点／新上下文    │
Phase 4: writing-plans ←────────────────────┘
         一次一个里程碑；子代理写计划，主代理处理用户问题
Phase 5: 用户选择执行方式 → 新执行会话
    ├─ Subagent-driven → subagent-driven-development
    └─ Native          → executing-plans
         ↓
    finishing-a-development-branch（merge / PR / cleanup）
         ↓  检查点／新上下文
    主代理主动交付续接文案；仍有里程碑才回 Phase 4，否则结束
```

Phase 3 评审收口时同步建立环境文档，此后每个里程碑的执行会话都只读它（见下文规则 2）。

Phase 2 的复杂度判断权在用户，宿主支持时使用 `Complex` / `Not complex` 选项。推荐依据是从实际安装的上游资源读取的 `Complexity Assessment` 规则，不编造分数；原有“超过 8 个实现步骤或涉及超过 3 个核心模块”的解释口径保留。提问和说明跟随用户语言；恢复未发生变化的工作时，不重复询问已经确认的选择。

### 这套工作流怎么减少重复工作

1. **访谈有停止条件。** 目标、范围／非目标、约束、验收标准已确认，且没有阻塞规划的问题即可推进。非阻塞假设需用户接受并约定在受影响工作前复核；安全、权限、数据一致性和不可逆迁移问题不能为省 token 而延期。用户明确只要访谈时，停在文档与规划阶段之前，不要求模板资源。用户拍板的每条决策都带状态位：`locked`（结论已定，不得重开）或 `deferred-with-criteria`（授权执行期按判据自决）。对抗评审可以攻击里程碑划分与判据，但不能把 `locked` 决策改写成执行期再问一遍——只能列成 Blocking 交用户裁决。
2. **文档按主题批量更新，交接走路径。** 一个主题确认后统一更新术语、决策和假设，转阶段前写完。入口读取上游 Context Brief 模板和评分规则并记住来源路径，不把模板复制进本仓库。简报落在 `docs/engineering-discipline/context/YYYY-MM-DD-<topic>-brief.md`，作为计划的 `Spec:` 引用。环境事实（含工作区之外仓库的代码库清单、构建工具绝对路径、数据库与服务地址、运行时状态、shell 怪癖）集中写进同目录的 `YYYY-MM-DD-<topic>-environment.md`：复杂路径在 Phase 3 评审收口时建立，不复杂路径在首次规划前建立，此后只做增量更新，各规划与执行会话都只读它。计划和简报只引用该路径而不复制正文，文档里已经记下的事实也不再回头问用户。凭据按来源记录：配置文件里已有的只记文件路径与键名、不抄值；用户在对话中口头给出的账户与口令原样记进本文件并写明用途；既无来源也拿不到的记成显式缺口，不靠默认答案绕过。因为它可能含凭据，这份文件不进提交清单，头部固定写一行"含凭据的过程产物，勿提交"。它和简报、计划、评审产物一样都属过程产物——要不要入库由用户定，但要跨需求留下的结论与理由必须落在 ADR 或词汇表里，不能只活在过程文档中。同目录还有一份只追加的**发现台账** `YYYY-MM-DD-<topic>-discoveries.md`，专门承载"前面犯的错"：只要用户纠正了 agent，或实测证伪了既有认知（因错误原因而通过的验证脚本、误判的可达性、过时的契约表述），当场追加一行——日期、里程碑、发现或纠正、类型、影响哪些后续里程碑、是否已提升进正式文件。之后每份计划都把它当输入，已记录的错不许再犯；能归类的条目提升进环境文档、冻结契约或词汇表，原行保留以便追溯。
3. **规划也设置真正的上下文断点。** Phase 2 完成、DAG 锁定、每个里程碑完成后，在 Context Brief 末尾更新简短的 `Workflow Resume`，再用新上下文继续。索引记录下一阶段、确认状态、产物路径（含环境文档）、待复核假设、代码基线和验证证据；里程碑状态仍以上游 `state.md` 为准。宿主不能开新会话时使用真实的压缩能力，两者都没有则提供手动续接指令，除非用户明确原地继续。恢复时检查相关已提交与未提交变更，不重读全部访谈历史。交付续接文案时同时宣告边界：本会话只覆盖哪一个里程碑或阶段、还剩几个、会话结束后用户要做什么。里程碑收尾后由主代理主动产出下一份续接文案，不等用户开口索要。同一会话连做多个里程碑是允许的选择，但不是抄近路：宿主压缩会悄悄让历史失真，而且越到后面越严重，所以每个里程碑仍然要**重新读取**续接索引、本里程碑文件、冻结契约、环境文档与发现台账，而不是靠回忆；与对话历史冲突时一律以文件为准。开工前用 10 行以内声明本里程碑适用的 `locked` 决策与契约口径，让你能在动手前纠偏，而不是做一半喊停。
4. **子代理交接有明确状态。** 写计划返回 `needs-input`、`plan-ready` 或 `blocked`，主代理负责用户问题和执行选择，收到答案后继续已有草稿而非重新规划。就绪时回传路径与最多五行摘要，执行选项另附；问题和阻塞信息不因行数限制被截断。任何暂停前都把状态、已有草稿路径和待解问题写入 `Workflow Resume`，解除阻塞后继续该草稿，不把草稿当作可执行计划。主代理检查实际计划后才移交；宿主不能委派该 skill 时，由主代理在新规划上下文调用。任何抛给用户的问题（包括转达子代理问题之前）都要先按决策编号、文件名、代码路径检索决策记录、`locked` 决策与 `Workflow Resume`：命中就直接引用原结论继续，不重问；只有新证据与原结论冲突时才升级，并附上冲突证据。
5. **一次只规划一个里程碑。** 后续计划根据实际代码写；用户选择执行方式后，另开上下文，只带当前计划和必要引用。完成验证与收尾并记录状态后才恢复下一里程碑，不跨里程碑静默沿用执行授权。DAG 锁定前把拓扑形状说给用户：若里程碑是一条纯链、没有任何可并发分支，明确告知只能串行、总耗时约等于各里程碑之和，并给用户一次改拆的机会，不替用户决定是否并行。里程碑收尾时，把只有过程文档承载的决策理由——尤其"这块为什么不改""为什么豁免"这类负向决策——写进 ADR 或词汇表并附出处编号；全部里程碑结束后再复核一遍，别让下一个需求重新问、重新踩。计划要新建查询、方法、常量或映射表之前，先在既有代码和已冻结的契约里检索同族实现：命中就复用、给它加一个可选参数、或在计划的 `Key Decisions` 写清为什么必须另开一条。另外，冻结契约里标注"由 Mx 实现"的条目不是范围锁——当前里程碑需要它就可以提前实现并回写落点，越界指的是新增契约之外的字段或口径。用户纠正过的点若还只活在对话里，这个里程碑就不算收尾完成。

这些规则减少冗余，但不承诺具体 token 降幅。写文件不会删除工具调用历史，子代理仍消耗 token，也可能继承上下文；缓存计价与宿主压缩会影响费用，便宜模型也可能增加重试。自足计划头部有助于新会话接手，但不保证较弱模型能正确执行。应比较所有 agent 合计的用量、实际费用、返工与验收结果，再判断是否节省。原有完整里程碑评审、默认执行方式和验证建议保持不变。

---

# 安装：两种方式选一种

同一个仓库同时支持 CC Switch 和 Claude Code Plugin。CC Switch 通过 `skills/` 目录发现 skill，Claude Code 读取 `.claude-plugin/` 下的清单。

## 方式一：CC Switch

CC Switch 按固定目录发现 skill，本仓库的 `skills/` 这一层就是给它用的。

**1. 添加本仓库**

| 字段 | 值 |
| --- | --- |
| 仓库地址 | <https://github.com/zhy-ty/grill-plan-execute> |
| Owner | `zhy-ty` |
| Name | `grill-plan-execute` |
| Branch | `main` |
| Subdirectory | `skills` |

然后安装 `grill-plan-execute`。

**2. 手动安装依赖（这一步不能省）**

CC Switch **不解析 `plugin.json` 里的 `dependencies`**。你需要逐个添加下面的上游仓库，每个仓库的 Subdirectory 都填 `skills`：

| 仓库地址 | Owner | Name | Branch | Subdirectory |
| --- | --- | --- | --- | --- |
| <https://github.com/mattpocock/skills> | `mattpocock` | `skills` | `main` | `skills` |
| <https://github.com/tmdgusya/engineering-discipline> | `tmdgusya` | `engineering-discipline` | `main` | `skills` |
| <https://github.com/obra/superpowers> | `obra` | `superpowers` | `main` | `skills` |

添加完再只勾选本工作流用得上的 skill（路径相对各仓库根目录）：

| 仓库（Subdirectory: `skills`） | 必需 | 只在 Phase 5 需要 | 可选 |
| --- | --- | --- | --- |
| [mattpocock/skills](https://github.com/mattpocock/skills) | `grilling`（`skills/productivity/grilling`）、`domain-modeling`（`skills/engineering/domain-modeling`） | — | `grill-with-docs`（`skills/engineering/grill-with-docs`） |
| [tmdgusya/engineering-discipline](https://github.com/tmdgusya/engineering-discipline) | `milestone-planning`（`skills/milestone-planning`）；`clarification` 用于读取模板与评分规则，已有可读本地来源时免装 | — | `plan-crafting`、`long-run`、`run-plan`、`review-work` |
| [obra/superpowers](https://github.com/obra/superpowers) | `writing-plans`（`skills/writing-plans`） | `subagent-driven-development`、`executing-plans`、`finishing-a-development-branch` | `requesting-code-review`、`receiving-code-review`、`test-driven-development`、`using-git-worktrees`、`verification-before-completion`、`dispatching-parallel-agents` |

没有可读上游源文件的全新 CC Switch 安装，应勾选 **5 个 skill**：四个工作流项，加上提供格式资源的 `clarification`；连三个执行／收尾项一起用就是 8 个。如果本地上游目录已经提供可读的模板和评分规则，则只安装四个工作流项即可。无论哪种情况，都不额外运行 `clarification` 访谈。`grill-with-docs` 与其他可选目标仍可不装，缺失时只告知，不凭空模拟。

Step 0 按需检查 skill 可用性，并在完整工作流开始访谈前检查模板与评分资源。资源缺失时暂停并给安装指引，不猜字段或分数。

> 注意：CC Switch 装完后这些 skill 可能不带插件前缀。本仓库 `SKILL.md` 写的是 Claude Code Plugin 的 `插件名:skill名` 形式；如果你的 CC Switch 暴露的是裸 skill 名，去掉前缀理解即可，调用的还是同一个 skill。

## 方式二：Claude Code Plugin

依赖会被自动拉取，但**三个 marketplace 必须先配置好**，这一步省不掉。

**1. 添加依赖所在的三个 marketplace**

```bash
claude plugin marketplace add mattpocock/skills
claude plugin marketplace add tmdgusya/engineering-discipline
claude plugin marketplace add obra/superpowers
```

它们的 marketplace 名分别是 `mattpocock`、`engineering-discipline`、`superpowers-dev`。本仓库 `marketplace.json` 的 `allowCrossMarketplaceDependenciesOn` 已声明这三个，也允许 `claude-plugins-official`，但本工作流没有声明来自该 marketplace 的直接依赖。

**2. 添加本仓库 marketplace 并安装**

```bash
claude plugin marketplace add zhy-ty/grill-plan-execute
claude plugin install grill-plan-execute@grill-plan-execute-marketplace
```

Claude Code 会接着把 `plugin.json` 里声明的三个依赖装上并启用。

**3. 使用**

```
/grill-plan-execute:grill-plan-execute
```

### 粒度：Plugin 路径的代价

Plugin 依赖是整包安装的：你会拿到该插件暴露的**全部** skill，而不是本工作流真正调用的那 4 个。

## 上游正文提到的兄弟 skill

上游 skill 的正文会提到自己仓库里的兄弟 skill。依赖插件装齐后它们都真实存在，按命名空间直接调用即可，不要当成缺失项，也不要凭记忆复述它们的行为：

| 被提到时 | 实际位置 |
| --- | --- |
| `clarification`、`plan-crafting`、`long-run`、`run-plan`、`review-work` | `engineering-discipline:<同名>` |
| `test-driven-development`、`using-git-worktrees`、`requesting-code-review`、`receiving-code-review`、`verification-before-completion`、`dispatching-parallel-agents`、`brainstorming` | `superpowers:<同名>` |

## 以后还能不能往工作流里再插一个 skill

能，而且这就是本项目预期的成长方式。`SKILL.md` 是纯编排层，你可以在任意阶段插入新的外源 skill，同时**零副本约束依旧成立：不要把那个 skill 的正文抄进本仓库**，只登记引用方式和调用点。

`SKILL.md` 只放运行时指令：正文调用后进入上下文，可能持续保留到宿主压缩或替换上下文。安装步骤、依赖明细、扩展指南、兄弟 skill 对照表都放在 README；重复输入的实际费用取决于宿主和缓存。

**动任何规则之前先读 [`docs/problem-log.md`](docs/problem-log.md)。** `SKILL.md` 里每条规则都对应那份文档中一次具体故障——含真实根因、已被排除的解释、以及分析过但被否决的替代方案。要改或删某条规则，请在那个条目里**追加**你的理由和日期，不要静默删掉；否则下一个人会把同一个 bug 再引入一次。

1. `SKILL.md`：在它那份精简的 `Dependencies` 清单里加一行（命名空间 + 作用），在对应 Phase 写调用点与**完成判据**、说明它消费上游哪个产物又给下游留下什么，形状变了要更新 `Order Matters` 流程图；若它是必需项，Step 0 自检也要加上。
2. `plugin.json`：只有当新 skill 来自尚未声明的插件时才动。来自 `mattpocock-skills`、`engineering-discipline`、`superpowers` 的不用改——Plugin 路径本来就整插件安装。
3. `marketplace.json`：只有引入**新 marketplace** 时，才把名字加进 `allowCrossMarketplaceDependenciesOn`。
4. `README.md` 与 `README.zh.md`：依赖表、上面的 CC Switch 逐仓库表、`marketplace add` 命令；新增插件带来上下文开销时，同步粒度说明。
5. `plugin.json` 的 `version`：新增**必需**依赖会改变用户的安装要求，至少进 minor。

唯一要提前想清楚的是：Plugin 路径没法挑，一个必需项若来自新仓库，就等于让所有人整包装下那个插件。只在部分场景有帮助的 skill，建议做成可选调用点。

## 版本与更新

- **每次工作流改动都要在 `docs/problem-log.md` 留条目**：加规则就记下它源自哪次故障，删规则就追加撤销理由和日期。这份日志是本仓库不重复踩同一个坑的唯一保障。

- **依赖内容更新**：本仓库不持有依赖副本，上游发布新版后你执行 `claude plugin update <插件名>@<marketplace>`（CC Switch 用户重新同步仓库）就能拿到最新正文，**本仓库无需改动**。
- **本仓库版本号**：工作流或安装契约改变时更新；上游兼容性需要时再收紧 `dependencies[].version`。2.5.0 新增跨会话统一读取的环境文档、`locked` / `deferred-with-criteria` 决策状态位（对抗评审不得推翻 `locked`）、续接文案自宣告会话边界、里程碑收尾后主动交付下一份续接文案、抛问题前先查决策台账，以及 DAG 锁定前的串行链提示、按来源记录凭据且永不提交的环境文档、收尾时把决策理由从过程文档搬进 ADR 与词汇表，以及新建查询前的同族查重和"契约的实现时点不是范围锁"这条判定、承载纠错并向下继承的发现台账、原地继续多个里程碑时"以文件为准 + 开工声明"的补偿动作。2.4.0 将原先隐含的 `clarification` 格式资源明确为前置条件，加入有界访谈与检查点续接，并把用户交互职责交给主代理。两个版本都没有新增插件，也没有复制上游正文。
- 当前 `dependencies` 用的是 caret（`^`）区间：跟随同一 major 内的新增功能与修复，major 升级不会自动跟进。

## License

MIT，见 [LICENSE](./LICENSE)。依赖 skill 的版权归各自作者，本仓库不对它们主张任何权利。

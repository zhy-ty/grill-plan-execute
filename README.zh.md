# grill-plan-execute

[English](./README.md)

仓库地址：**<https://github.com/zhy-ty/grill-plan-execute>**

一个**只做编排**的 Claude Code skill，把三个外部仓库的 skill 串成一条工作流：先靠逼问式访谈和领域建模把需求问清楚，再和用户确认业务到底有多复杂，然后产出实现计划，并把执行移交到一个新会话——规划用强模型，执行可切便宜模型、由强模型负责评审。

本仓库只包含编排逻辑，**不含任何依赖 skill 的内容副本**——依赖正文全部来自各自原始仓库，所以上游的改进会直接触达用户，本仓库不需要跟着改。

> **关于 `obra/superpowers-skills`**：该仓库已被作者归档（2025-10 起不再更新）。本工作流需要的 `writing-plans` 现在在持续维护的 [obra/superpowers](https://github.com/obra/superpowers)（v7.x）里。如果你依赖归档仓库的那个版本，需要自行 fork 维护。

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
    仍有里程碑才回 Phase 4，否则结束
```

Phase 2 的复杂度判断权在用户，宿主支持时使用 `Complex` / `Not complex` 选项。推荐依据是从实际安装的上游资源读取的 `Complexity Assessment` 规则，不编造分数；原有“超过 8 个实现步骤或涉及超过 3 个核心模块”的解释口径保留。提问和说明跟随用户语言；恢复未发生变化的工作时，不重复询问已经确认的选择。

### 这套工作流怎么减少重复工作

1. **访谈有停止条件。** 目标、范围／非目标、约束、验收标准已确认，且没有阻塞规划的问题即可推进。非阻塞假设需用户接受并约定在受影响工作前复核；安全、权限、数据一致性和不可逆迁移问题不能为省 token 而延期。用户明确只要访谈时，停在文档与规划阶段之前，不要求模板资源。
2. **文档按主题批量更新，交接走路径。** 一个主题确认后统一更新术语、决策和假设，转阶段前写完。入口读取上游 Context Brief 模板和评分规则并记住来源路径，不把模板复制进本仓库。简报落在 `docs/engineering-discipline/context/YYYY-MM-DD-<topic>-brief.md`，作为计划的 `Spec:` 引用。
3. **规划也设置真正的上下文断点。** Phase 2 完成、DAG 锁定、每个里程碑完成后，在 Context Brief 末尾更新简短的 `Workflow Resume`，再用新上下文继续。索引记录下一阶段、确认状态、产物路径、待复核假设、代码基线和验证证据；里程碑状态仍以上游 `state.md` 为准。宿主不能开新会话时使用真实的压缩能力，两者都没有则提供手动续接指令，除非用户明确原地继续。恢复时检查相关已提交与未提交变更，不重读全部访谈历史。
4. **子代理交接有明确状态。** 写计划返回 `needs-input`、`plan-ready` 或 `blocked`，主代理负责用户问题和执行选择，收到答案后继续已有草稿而非重新规划。就绪时回传路径与最多五行摘要，执行选项另附；问题和阻塞信息不因行数限制被截断。任何暂停前都把状态、已有草稿路径和待解问题写入 `Workflow Resume`，解除阻塞后继续该草稿，不把草稿当作可执行计划。主代理检查实际计划后才移交；宿主不能委派该 skill 时，由主代理在新规划上下文调用。
5. **一次只规划一个里程碑。** 后续计划根据实际代码写；用户选择执行方式后，另开上下文，只带当前计划和必要引用。完成验证与收尾并记录状态后才恢复下一里程碑，不跨里程碑静默沿用执行授权。

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

1. `SKILL.md`：在它那份精简的 `Dependencies` 清单里加一行（命名空间 + 作用），在对应 Phase 写调用点与**完成判据**、说明它消费上游哪个产物又给下游留下什么，形状变了要更新 `Order Matters` 流程图；若它是必需项，Step 0 自检也要加上。
2. `plugin.json`：只有当新 skill 来自尚未声明的插件时才动。来自 `mattpocock-skills`、`engineering-discipline`、`superpowers` 的不用改——Plugin 路径本来就整插件安装。
3. `marketplace.json`：只有引入**新 marketplace** 时，才把名字加进 `allowCrossMarketplaceDependenciesOn`。
4. `README.md` 与 `README.zh.md`：依赖表、上面的 CC Switch 逐仓库表、`marketplace add` 命令；新增插件带来上下文开销时，同步粒度说明。
5. `plugin.json` 的 `version`：新增**必需**依赖会改变用户的安装要求，至少进 minor。

唯一要提前想清楚的是：Plugin 路径没法挑，一个必需项若来自新仓库，就等于让所有人整包装下那个插件。只在部分场景有帮助的 skill，建议做成可选调用点。

## 版本与更新

- **依赖内容更新**：本仓库不持有依赖副本，上游发布新版后你执行 `claude plugin update <插件名>@<marketplace>`（CC Switch 用户重新同步仓库）就能拿到最新正文，**本仓库无需改动**。
- **本仓库版本号**：工作流或安装契约改变时更新；上游兼容性需要时再收紧 `dependencies[].version`。2.4.0 将原先隐含的 `clarification` 格式资源明确为前置条件，加入有界访谈与检查点续接，并把用户交互职责交给主代理；没有新增插件或复制上游正文。
- 当前 `dependencies` 用的是 caret（`^`）区间：跟随同一 major 内的新增功能与修复，major 升级不会自动跟进。

## License

MIT，见 [LICENSE](./LICENSE)。依赖 skill 的版权归各自作者，本仓库不对它们主张任何权利。

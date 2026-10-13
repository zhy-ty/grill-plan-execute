# grill-plan-execute

[English](./README.md)

仓库地址：**<https://github.com/zhy-ty/grill-plan-execute>**

一个**只做编排**的 Claude Code skill，把三个外部仓库的 skill 串成一条工作流：先靠逼问式访谈和领域建模把需求问清楚，再和用户确认业务到底有多复杂，然后产出实现计划并移交执行。

本仓库只包含编排逻辑，**不含任何依赖 skill 的内容副本**——依赖正文全部来自各自原始仓库，所以上游的改进会直接触达用户，本仓库不需要跟着改。

> **关于 `obra/superpowers-skills`**：该仓库已被作者归档（2025-10 起不再更新）。本工作流需要的 `writing-plans` 现在在持续维护的 [obra/superpowers](https://github.com/obra/superpowers)（v7.x）里。如果你依赖归档仓库的那个版本，需要自行 fork 维护。

## 依赖的外部 skill

| 命名空间 | 必需？ | 作用 | 插件 / marketplace | 原始仓库 |
| --- | --- | --- | --- | --- |
| `mattpocock-skills:grilling` | **必需** | 决策树式的多轮访谈 | `mattpocock-skills@mattpocock` ^1.3.1 | [mattpocock/skills](https://github.com/mattpocock/skills) |
| `mattpocock-skills:domain-modeling` | **必需** | 边聊边维护词汇表与决策记录 | 同上 | 同上 |
| `engineering-discipline:milestone-planning` | **必需**（复杂路径） | 并行探索代码 → 里程碑 DAG → 对抗性评审轮次 | `engineering-discipline@engineering-discipline` ^1.7.0 | [tmdgusya/engineering-discipline](https://github.com/tmdgusya/engineering-discipline) |
| `superpowers:writing-plans` | **必需** | 写给零上下文工程师的详细实现计划 | `superpowers@superpowers-dev` ^7.0.0 | [obra/superpowers](https://github.com/obra/superpowers) |
| `mattpocock-skills:grill-with-docs` | 可选 | 一次调用顶上面两个 skill | 同 `grilling` | 同上 |
| `superpowers:subagent-driven-development`、`superpowers:executing-plans`、`superpowers:finishing-a-development-branch` | 可选 | Phase 5 的移交目标 | 同 `writing-plans` | 同上 |

必需项只有 4 个，其余只是让流程跑起来更省事。注意：走 Plugin 路径时安装的是**整个插件**，没法按这个清单挑，见[粒度：Plugin 路径的代价](#粒度plugin-路径的代价)。

## 工作流

```
Phase 1: mattpocock-skills:grilling + mattpocock-skills:domain-modeling
         （装了 grill-with-docs 可用一次调用代替）
    ↓  需求收敛，用户确认达成共识
Phase 2: 询问用户业务逻辑是否复杂
         （>8 个实现步骤，或 >3 个需要改动/理解的核心模块）
    ├─ 不复杂 ──────────────────────────────┐
    └─ 复杂 → Phase 3                       │
Phase 3: engineering-discipline:milestone-planning
         完整流程：探索 → 起草 → 对抗性评审 → 修订到无阻塞项
         ↓  里程碑 DAG 锁定                 │
Phase 4: superpowers:writing-plans  ←───────┘
         复杂路径：每个里程碑一份；不复杂：整体一份
    ↓
Phase 5: 移交执行
    ├─ Subagent-driven → superpowers:subagent-driven-development
    └─ Native          → superpowers:executing-plans
         ↓
    superpowers:finishing-a-development-branch（merge / PR / cleanup）
```

Phase 2 的复杂度判断权在用户，skill 只会问，不会替他决定。提问在宿主支持结构化选项时走选项（`Complex` / `Not complex` 两选一），并且**所有面向用户的文字跟随用户当前语言**——`SKILL.md` 里的提问原文中英两版都给了，非中文用户不会看到中文。

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
| [tmdgusya/engineering-discipline](https://github.com/tmdgusya/engineering-discipline) | `milestone-planning`（`skills/milestone-planning`） | — | `clarification`、`plan-crafting`、`long-run`、`run-plan`、`review-work` |
| [obra/superpowers](https://github.com/obra/superpowers) | `writing-plans`（`skills/writing-plans`） | `subagent-driven-development`、`executing-plans`、`finishing-a-development-branch` | `requesting-code-review`、`receiving-code-review`、`test-driven-development`、`using-git-worktrees`、`verification-before-completion`、`dispatching-parallel-agents` |

也就是说**最少装 4 个**（`grilling`、`domain-modeling`、`milestone-planning`、`writing-plans`），要连执行移交一起用就是 7 个。"可选"那一列永远不是必需：`grill-with-docs` 缺失时 `SKILL.md` 会自己去调 `grilling` + `domain-modeling`，其余可选 skill 只是上游文档指路时才会提到，缺了就告知用户，不会让 agent 凭空模拟。

四个必需项装齐再用本工作流。缺任何一个，工作流会在需要它的那个阶段停下来告诉你（`SKILL.md` 的 Step 0 就是干这个的）。

> 注意：CC Switch 装完后这些 skill 可能不带插件前缀。本仓库 `SKILL.md` 写的是 Claude Code Plugin 的 `插件名:skill名` 形式；如果你的 CC Switch 暴露的是裸 skill 名，去掉前缀理解即可，调用的还是同一个 skill。

## 方式二：Claude Code Plugin

依赖会被自动拉取，但**三个 marketplace 必须先配置好**，这一步省不掉。

**1. 添加依赖所在的三个 marketplace**

```bash
claude plugin marketplace add mattpocock/skills
claude plugin marketplace add tmdgusya/engineering-discipline
claude plugin marketplace add obra/superpowers
```

它们的 marketplace 名分别是 `mattpocock`、`engineering-discipline`、`superpowers-dev`。本仓库 `marketplace.json` 的 `allowCrossMarketplaceDependenciesOn` 已声明这三个（外加 `claude-plugins-official`，见下文）。

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

Plugin 依赖是整包安装的：你会拿到该插件暴露的**全部** skill，而不是本工作流真正调用的那 4 个。在 Claude Code 2.1.278 上按当前锁定的版本实测：

| 插件 | 装进来的 skill 数 | 常驻上下文成本 |
| --- | --- | --- |
| `mattpocock-skills` | 27 | ~1,169 tok |
| `engineering-discipline` | 11 | ~997 tok |
| `superpowers` | 15 | ~706 tok |
| **合计** | **53** | **每个会话系统提示里约 2,872 tok** |

这还没算本插件自己的 skill（约 57 tok）。我们这边裁不动：skill 白名单写在上游各自的 `plugin.json` 里（Matt 的就是显式枚举了 27 个路径），而 marketplace 条目没有任何受支持的写法可以去子集化别人的插件。

只要你想要的就是那 4 个 skill，请走上面的 CC Switch 路径。想留在 Plugin 路径又压缩体积，只能 fork 本仓库、删掉 `plugin.json` 里的 `dependencies` 数组、再手工安装你需要的那几个上游 skill——代价是失去依赖自动解析和上游自动更新。

### 两个实测行为

以下都在 Claude Code 2.1.278 上实测（依赖落版：`mattpocock-skills` 1.3.1、`engineering-discipline` 1.7.0、`superpowers` 7.0.0）：

1. **漏加某个上游 marketplace 不会让安装失败。** `plugin install` 照样报成功，但插件随后是 `✘ failed to load`，错误信息会直接点名哪个依赖没装、并给出补救命令，形如：
   `Dependency "mattpocock-skills@mattpocock" is not installed — run `claude plugin install mattpocock-skills@mattpocock`, or check that its marketplace is added`
   所以装完请用 `claude plugin list` 确认本插件是 `✔ enabled`，别只看安装那行的对勾。
2. **顺序不敏感。** 先装本插件、后补上游 marketplace 也能自愈：`marketplace add` 的输出里会带 `(+ 1 dependency: mattpocock-skills)`，也就是补配市场的那一刻就把欠的依赖装上了。

其他排查项：

- 版本区间不满足（例如上游发了 `superpowers` 8.0.0，而这里声明的是 `^7.0.0`）→ 放宽或跟进 `plugin.json` 里的 `version`
- 你的 `superpowers` 是从官方 `claude-plugins-official` 装的（版本通常落后于上游）→ 把那条依赖的 `"marketplace": "superpowers-dev"` 改成 `"claude-plugins-official"`；两个名字都已在 allow 清单里，改哪边都能通过权限检查。两者也能共存，实测 `superpowers@superpowers-dev` 与 `superpowers@claude-plugins-official` 同时存在不冲突。

## 以后还能不能往工作流里再插一个 skill

能，而且这就是本项目预期的成长方式。`SKILL.md` 是纯编排层，你可以在任意阶段插入新的外源 skill，同时**零副本约束依旧成立：不要把那个 skill 的正文抄进本仓库**，只登记引用方式和调用点。`skills/grill-plan-execute/SKILL.md` 里有 `Extending this workflow` 一节列明全部改动点，简述：

1. `SKILL.md`：依赖表加一行（命名空间、必需？、作用、插件/marketplace 与版本区间、原始仓库链接），在对应 Phase 写调用点与**完成判据**，形状变了要更新流程图；若它是必需项，Step 0 自检也要加上。
2. `plugin.json`：只有当新 skill 来自尚未声明的插件时才动。来自 `mattpocock-skills`、`engineering-discipline`、`superpowers` 的不用改——Plugin 路径本来就整插件安装。
3. `marketplace.json`：只有引入**新 marketplace** 时，才把名字加进 `allowCrossMarketplaceDependenciesOn`。
4. `README.md` 与 `README.zh.md`：依赖表、上面的 CC Switch 逐仓库表、`marketplace add` 命令；新增了插件的话同步粒度成本表（每个额外插件都给所有人增加常驻 token）。
5. `plugin.json` 的 `version`：新增**必需**依赖会改变用户的安装要求，至少进 minor。

唯一要提前想清楚的是：Plugin 路径没法挑，一个必需项若来自新仓库，就等于让所有人整包装下那个插件。只在部分场景有帮助的 skill，建议做成可选调用点。

## 版本与更新

- **依赖内容更新**：本仓库不持有依赖副本，上游发布新版后你执行 `claude plugin update <插件名>@<marketplace>`（CC Switch 用户重新同步仓库）就能拿到最新正文，**本仓库无需改动**。
- **本仓库版本号**：只有当依赖发生破坏性变化（skill 改名、被删除、产出结构变化导致编排逻辑不再成立）时，才需要改 `SKILL.md`、把 `plugin.json` 的 `version` 跟进，并收紧 `dependencies[].version` 区间。
- 当前 `dependencies` 用的是 caret（`^`）区间：跟随同一 major 内的新增功能与修复，major 升级不会自动跟进。

## License

MIT，见 [LICENSE](./LICENSE)。依赖 skill 的版权归各自作者，本仓库不对它们主张任何权利。

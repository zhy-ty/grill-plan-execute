# grill-plan-execute

一个**编排层** Claude Code Skill：把三个外部仓库的 5 个 skill 串成"澄清需求 → 里程碑规划 → 逐里程碑实现计划"的固定工作流。

本仓库只包含编排逻辑，**不含任何依赖 skill 的内容副本**。依赖正文全部由用户从原始仓库动态获取，所以上游更新会直接生效，本仓库不需要跟着改。

> **关于 `obra/superpowers-skills`**：该仓库已被作者归档（2025-10 起不再更新），本工作流依赖的 `writing-plans` 现取自在维护的 [obra/superpowers](https://github.com/obra/superpowers)（v7.x）。如果你依赖归档仓库的版本，需要自行 fork 维护。

## 依赖的外部 skill

| 命名空间 | 作用 | 插件 / marketplace | 原始仓库 |
| --- | --- | --- | --- |
| `mattpocock-skills:grill-with-docs` | 聚合入口，一次调用同时跑访谈与文档产出 | `mattpocock-skills@mattpocock` ^1.3.1 | [mattpocock/skills](https://github.com/mattpocock/skills) |
| `mattpocock-skills:grilling` | 决策树式多轮访谈 | 同上 | 同上 |
| `mattpocock-skills:domain-modeling` | 实时维护领域词汇与决策记录 | 同上 | 同上 |
| `engineering-discipline:milestone-planning` | 并行探索 → 里程碑 DAG → 对抗性评审循环 | `engineering-discipline@engineering-discipline` ^1.7.0 | [tmdgusya/engineering-discipline](https://github.com/tmdgusya/engineering-discipline) |
| `superpowers:writing-plans` | 面向零上下文工程师的详细实现计划 | `superpowers@superpowers-dev` ^7.0.0 | [obra/superpowers](https://github.com/obra/superpowers) |

## 工作流

```
Phase 1: mattpocock-skills:grill-with-docs   （grilling + domain-modeling）
    ↓  需求收敛，用户确认达成共识
Phase 2: engineering-discipline:milestone-planning   （完整流程，含对抗性评审）
    ↓  里程碑 DAG 锁定
Phase 3: superpowers:writing-plans   ×  每个里程碑各一份
    ↓
Phase 4: 移交执行
    ├─ Subagent-driven → superpowers:subagent-driven-development
    └─ Native          → superpowers:executing-plans  →  finishing-a-development-branch
```

Phase 4 的三个 skill 同样来自 `superpowers`，不额外增加依赖。


---

# 安装：两种方式选一种

同一个仓库同时支持 CC Switch 和 Claude Code Plugin。CC Switch 走 `skills/` 目录发现，Claude Code 走 `.claude-plugin/` 清单。

## 方式一：CC Switch

CC Switch 按固定目录发现 skill，本仓库的 `skills/` 一层就是给它用的。

**1. 添加本仓库**

| 字段 | 值 |
| --- | --- |
| Owner | `zhy-ty` |
| Name | `grill-plan-execute` |
| Branch | `main` |
| Subdirectory | `skills` |

然后安装 `grill-plan-execute`。

**2. 手动安装依赖（这一步不能省）**

CC Switch **不解析 `plugin.json` 里的 `dependencies`**。装完本 skill 后，你需要逐个把下面三个仓库也添加进 CC Switch 并安装。每个仓库的 Subdirectory 都是 `skills`：

- [mattpocock/skills](https://github.com/mattpocock/skills) — Subdirectory: `skills`（提供 `grill-with-docs`、`grilling`、`domain-modeling`）
- [tmdgusya/engineering-discipline](https://github.com/tmdgusya/engineering-discipline) — Subdirectory: `skills`（提供 `milestone-planning`）
- [obra/superpowers](https://github.com/obra/superpowers) — Subdirectory: `skills`（提供 `writing-plans`）

三个都装齐后再用本 skill。少装任何一个，工作流会在对应阶段停下来报缺依赖（`SKILL.md` 的 Step 0 就是做这个自检的）。

> 注意：CC Switch 安装后这些 skill 的命名空间可能不带插件前缀。本仓库 `SKILL.md` 里写的是 Claude Code Plugin 的 `插件名:skill名` 形式；如果你的 CC Switch 暴露的是裸 skill 名，把前缀去掉理解即可，调用的还是同一个 skill。

## 方式二：Claude Code Plugin

依赖由 Claude Code 自动拉取，你不需要手动安装依赖的 skill——但**三个 marketplace 必须先配置**，这一步省不掉。

**1. 添加依赖所在的三个 marketplace**

```bash
claude plugin marketplace add mattpocock/skills
claude plugin marketplace add tmdgusya/engineering-discipline
claude plugin marketplace add obra/superpowers
```

它们各自的 marketplace 名是 `mattpocock`、`engineering-discipline`、`superpowers-dev`，本仓库 `marketplace.json` 的 `allowCrossMarketplaceDependenciesOn` 已声明这三个（外加 `claude-plugins-official`，见下方说明）。

**2. 添加本仓库 marketplace 并安装**

```bash
claude plugin marketplace add zhy-ty/grill-plan-execute
claude plugin install grill-plan-execute@grill-plan-execute-marketplace
```

Claude Code 会按 `plugin.json` 的 `dependencies` 自动把三个依赖装上并启用。

**3. 使用**

```
/grill-plan-execute:grill-plan-execute
```

### 关于依赖解析的两个实测结论

以下行为在 Claude Code 2.1.278 上实测（依赖落版：`mattpocock-skills` 1.3.1、`engineering-discipline` 1.7.0、`superpowers` 7.0.0）：

1. **漏加某个上游 marketplace 不会让安装失败。** `plugin install` 照样报成功，但插件随后是 `✘ failed to load`，错误信息会直接点名是哪个依赖没装、并给出补救命令，形如：
   `Dependency "mattpocock-skills@mattpocock" is not installed — run `claude plugin install mattpocock-skills@mattpocock`, or check that its marketplace is added`
   所以装完请执行 `claude plugin list` 确认本插件是 `✔ enabled`，别只看安装那行的对勾。
2. **顺序不敏感。** 先装本插件、后补上游 marketplace 也能自愈：`marketplace add` 的输出里会带 `(+ 1 dependency: mattpocock-skills)`，即补配市场的那一刻就把欠的依赖装上了。

其他排查项：

- 版本区间不满足（例如上游发了 `superpowers` 8.0.0，而依赖写的是 `^7.0.0`）→ 放宽或跟进 `plugin.json` 里的 `version`
- `superpowers` 你是从官方 `claude-plugins-official` 装的（版本通常落后于上游）→ 把 `plugin.json` 里那条依赖的 `"marketplace": "superpowers-dev"` 改成 `"claude-plugins-official"`；两个名字都已在 allow 清单里，改哪边都能通过权限检查。二者也可共存，实测 `superpowers@superpowers-dev` 与 `superpowers@claude-plugins-official` 同时存在不冲突。


---

## 关于版本与更新

- **依赖内容更新**：本仓库不持有依赖副本，上游发布新版后，你执行 `claude plugin update <插件名>@<marketplace>`（CC Switch 用户重新同步仓库）就能拿到最新正文，**本仓库无需改动**。
- **本仓库版本号**：只有当依赖发生**破坏性变化**（skill 改名、被删除、产出结构变化导致编排逻辑不再成立）时，才需要改 `SKILL.md` 并把 `plugin.json` 的 `version` 跟进，同时收紧 `dependencies[].version` 区间。
- 当前 `dependencies` 用的是 caret 区间（`^`）：跟随同一 major 内的新增功能与修复，major 升级不会自动跟进。

## License

MIT，见 [LICENSE](./LICENSE)。依赖 skill 的版权归各自作者，本仓库不对它们主张任何权利。

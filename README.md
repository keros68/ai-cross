# ai-cross

AI agent 用的跨厂商交叉验证和分层派发 skill。它把关键产出交给不同厂商的模型互相复查，也能按任务风险把扫描、实现和把关分到不同档位，并把派发过程留痕到项目目录。

它不是自动替人做最终判断的多模型投票器。交叉验证能暴露单模型容易漏掉的错误和分歧；可独立验证的数字、代码、格式和结论，仍然要由编排者或人工复核。

> 中文为主，English summary below.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Agent Skill](https://img.shields.io/badge/Agent%20Skill-SKILL.md-green.svg)](SKILL.md)
[![Cross Vendor](https://img.shields.io/badge/Cross--Vendor-Review-orange.svg)](SKILL.md)

## 适用场景

- 重要代码改动、科研分析或长文档处理完成后，想让另一个厂商的模型独立复查。
- 已经同时使用 Claude Code、Codex、Qoder、GLM、Kimi、StepFun 等多个模型入口，想按任务风险分配给不同通道。
- 订阅或包月用户想把简单扫描和批量整理交给低成本档，把强模型额度留给架构、审查和方法学把关。
- 需要保留每次派发的任务、模型、输出、结论和耗时，方便后续复查。
- 想避免把同一家模型的多个 agent 当成真正独立的交叉验证。

## 你需要什么

- **宿主 agent**：Claude Code 或 Codex 至少有一个。两者是一等支持目标。
- **多厂商通道**：做真正交叉验证时，建议先配置 [cc-switch](https://github.com/farion1231/cc-switch)，再由 ai-cross 只读配置并按进程注入。
- **至少两家不同厂商**：只有一家模型也能做分层派发，但不能做跨厂商交叉验证，skill 会如实提示不可用。
- **本地 shell 能力**：外部派发依赖命令行入口；不支持 shell 的纯聊天环境不能完整运行。

API key 和 token 不写进本仓库，也不写入 ai-cross 产出的留痕文件。

## 它做什么

- 盘点可用模型、厂商、档位和调用通道，生成本机能力清单。
- 按任务类型选择低档扫描、常规执行、强模型把关或跨厂商复查。
- 对关键代码和分析结果启动不同厂商模型的交叉审查。
- 对模型真身做冒烟验证，避免第三方端点静默降级后还被误当成目标模型。
- 把外部派发的任务全文、通道、模型、原始输出、结论和耗时写入 `.dispatch/`。
- 在多轮实现、审查、修正时保留状态，避免会话中断后从头再来。
- 用本地脚本或测试复核可机器验证的结果，不把模型自述当作最终证据。

## 不做什么

- 不保证多个模型达成共识就一定正确。
- 不把同厂商不同档位包装成真正独立的交叉验证。
- 不预测论文录用、代码合并或方案一定成功。
- 不绕过登录、验证码、付费墙、机构权限或各平台使用限制。
- 不自动读取、上传或回显密钥；需要访问本地凭据前会先说明。
- 不承诺所有 agent 宿主都开箱即用。Claude Code 和 Codex 是主要支持对象，其他宿主需要按各自规则适配。

## 工作流程

```text
用户任务
  ↓
盘点模型和通道，生成 manifest
  ↓
判断任务类型：扫描 / 实现 / 审查 / 科研分析 / 精确计算
  ↓
选择档位和厂商
  ├─ 日常任务：低成本通道优先
  ├─ 关键代码：实现后换厂商审查
  ├─ 科研分析：默认双保险
  └─ 精确数值：必须写代码或本地复核
  ↓
记录 .dispatch 留痕
  ↓
汇总共识、分歧、验证项和未验证项
```

## 使用方式

在支持 GitHub skill 安装的 agent 里，可以直接发送：

```text
请从 GitHub 安装这个 skill，并在需要多模型分工、跨厂商交叉审查或模型派发时优先使用它：
https://github.com/keros68/ai-cross
```

安装后，新开窗口或重启 agent，然后先盘点模型：

```text
使用 $ai-cross 盘点模型
```

第一次盘点会自动做这些事（全程只读，不读取也不显示任何密钥）：

1. 检测本机已装的 agent CLI（只跑 `--version`，不登录、不发模型请求）；
2. 若装了 cc-switch，只读其供应商清单（token 不输出，只有"是否已存 key"）；
3. 聚合各 CLI 的用量元数据，找出"装了但你没说"的入口；
4. 把检测到的入口摆成一张表请你**一次确认**，然后逐项冒烟，生成 `manifest.md`。

你要做的只有一件事：勾选要纳入的入口。全新机器什么都检测不到时，它会列出可选入口和对应安装命令。

常规调用示例：

```text
使用 $ai-cross 扫描这个项目，实现 XX 功能，并让不同厂商模型做交叉审查。
```

### 手动安装

直接把整个仓库 clone 到对应 skills 目录，改名为 `ai-cross` 即可，无需再挑子目录：

Claude Code：

```text
git clone https://github.com/keros68/ai-cross ~/.claude/skills/ai-cross
agents/*.md（仓库内）  ->  ~/.claude/agents/
```

Codex：

```text
git clone https://github.com/keros68/ai-cross ~/.codex/skills/ai-cross
```

Windows 上 `~` 即 `C:\Users\<用户名>`。装完后新开会话生效。

其他 agent 如果支持 `SKILL.md`，把整个仓库放到对应 skills 目录即可；如果不支持正式 skill loader，可以把仓库根的 `SKILL.md` 作为项目规则或 agent instruction 使用。

## 密钥安全

ai-cross 的密钥处理规则是：本地只读、不外传、不回显、不写入仓库、不写入留痕文件、不进模型上下文。需要调用外部通道时，密钥只按进程注入给对应子命令。

完整规则见 [`references/security.md`](references/security.md)。如果使用 cc-switch，ai-cross 通过只读桥接脚本读取已有配置，不要求你把 key 再填一遍。

## 设计原则

- 交叉验证看厂商独立性，不看 agent 数量。
- 默认先用低成本通道，验证失败或任务不可机器验证时再升级。
- 精确数值不交给模型心算；能跑代码就跑代码，不能跑就人工复核。
- 共识只说明多个模型给出了相同结论，不自动等于证据可靠。
- 外部派发必须留痕，关键任务要能回看原始输出。
- review 循环最多三轮；仍有分歧时并列证据交给人裁决。

## 文件结构

- `SKILL.md` - skill 主说明、路由规则、执行闭环和稳健性规则。
- `references/setup.md` - 模型盘点和接入向导。
- `references/channels.md` - 各通道命令模板和模型漂移维护。
- `references/cc_switch.py` - cc-switch 只读桥。
- `references/verify_model.py` - 模型真身校验脚本。
- `references/security.md` - 密钥处理规则。
- `references/dispatch-design.md` - 并行边界和派发设计说明。
- `references/playbooks.md` - 可选编排方案。
- `agents/scout.md` - Claude Code 低档只读侦察 agent。
- `agents/worker.md` - Claude Code 常规执行 agent。
- `agents/heavy.md` - Claude Code 高档只读审查 agent。
- `agents/advisor.md` - Claude Code 决策点顾问 agent。
- `qoder/` - Qoder 适配示例。

## Attribution and Redistribution

This project is the original ai-cross skill by keros68:

https://github.com/keros68/ai-cross

The project is released under the MIT License. Redistribution, forks, modified versions, and repackaged copies must preserve the copyright notice and license text. Please do not present modified copies as the original project or imply endorsement by the original author.

## English

ai-cross is an AI-agent skill for cross-vendor model review and tiered task dispatch. It routes routine scanning, implementation, and critical review to different model tiers, then uses models from different vendors to check important outputs.

The goal is not automatic majority voting. ai-cross keeps an audit trail, reports agreement and disagreement, and requires independently verifiable facts such as numbers, code behavior, and formatting checks to be validated by tools or humans.

Typical use cases include code review after implementation, research-analysis cross-checking, quota-aware task routing for subscription users, and preserving dispatch logs in `.dispatch/`.

Quick start:

```text
Install this skill from GitHub and use it for cross-vendor model review and task dispatch:
https://github.com/keros68/ai-cross
```

Then restart or open a new agent window and call:

```text
Use $ai-cross to inventory my available models.
```

First run auto-detects installed agent CLIs and cc-switch providers (read-only, keys are never read aloud or shown), asks you to confirm once, then smoke-tests and writes `manifest.md`. On a blank machine it falls back to asking what you have, with install commands.

## License

MIT. See [LICENSE](LICENSE).

---

**同系列 Agent Skills**：[sci-select](https://github.com/keros68/sci-select)（选刊+投稿前审查） · [academic-reference-matcher](https://github.com/keros68/academic-reference-matcher)（文献引用） · [abstract-fig](https://github.com/keros68/abstract-fig)（图形摘要） · [cugb-doctoral-thesis-format](https://github.com/keros68/cugb-doctoral-thesis-format)（学位论文格式） · [ai-cross](https://github.com/keros68/ai-cross)（多模型交叉验证）｜全览见 [keros68](https://github.com/keros68)

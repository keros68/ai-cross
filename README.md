# ai-cross

AI agent 用的跨厂商交叉验证与分层派发 skill。关键产出交给不同厂商的模型互相复查；扫描、实现、把关按任务风险分到不同档位；每次派发留痕到项目目录。

它不是多模型投票器。交叉验证用来暴露单模型漏掉的错误和分歧，可独立验证的数字、代码、结论仍由编排者跑代码复核，或并列证据交人裁决。

> 中文为主，English summary below.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Agent Skill](https://img.shields.io/badge/Agent%20Skill-SKILL.md-green.svg)](SKILL.md)
[![Cross Vendor](https://img.shields.io/badge/Cross--Vendor-Review-orange.svg)](SKILL.md)

## 使用场景

- 重要代码改动、科研分析或长文档处理完成后，让另一家厂商的模型独立复查。
- 手里同时有 Claude Code、Codex、GLM、Kimi、StepFun 等多个入口，想按任务风险分配：粗活走低成本档，强模型额度留给架构、审查和方法学把关。
- 需要保留每次派发的任务全文、模型、原始输出和耗时，事后能逐路复查。

## 环境要求

- **宿主 agent**：Claude Code 或 Codex 至少一个。内部 subagent 分层（scout/worker/heavy/advisor）只有 Claude Code 有，Codex 宿主的低中高档全部走外部通道。
- **交叉验证需要至少两家不同厂商的模型**。只有一家也能用分层派发，skill 会直说交叉验证不可用，不会拿同厂商复查冒充。
- 外部派发依赖本地 shell，纯聊天环境跑不完整。
- 多通道用户建议先配好 [cc-switch](https://github.com/farion1231/cc-switch)，ai-cross 只读其配置、按进程注入，key 不用再填一遍。

API key 和 token 不写进本仓库，也不写入 ai-cross 产出的任何文件。

## 功能

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

盘点生成本机能力清单（厂商 × 档位矩阵），路由按它走。派发前对第三方端点做模型真身校验，防止静默降级后还被当成目标模型。派发全文和原始输出写入 `.dispatch/`（目录自带 `.gitignore`，不随代码库提交），多轮 review 中断后可从断点恢复。可机器验证的结果由本地脚本或测试复核，模型自述不算证据。

## 边界

- 多模型共识不等于正确，同厂商不同档位也不算独立的交叉验证。
- review 循环最多三轮，仍有分歧就并列双方证据交人裁决，不强行仲裁。
- 不预测论文录用、代码合并或方案成败；不绕过登录、验证码、付费墙和各平台使用限制。
- 不自动读取、上传或回显密钥，访问本地凭据前先说明。
- Claude Code 和 Codex 是主要支持对象，其他宿主需要按各自规则适配，不承诺开箱即用。

## 快速开始

在支持 GitHub skill 安装的 agent 里发送：

```text
请从 GitHub 安装这个 skill，并在需要多模型分工、跨厂商交叉审查或模型派发时优先使用它：
https://github.com/keros68/ai-cross
```

装完新开窗口，然后：

**第一步，盘点。**

```text
使用 $ai-cross 盘点模型
```

首次盘点只读检测本机已装的 agent CLI 和 cc-switch 供应商清单（跑 `--version`，不登录、不发模型请求、不读取也不显示任何密钥），把检测到的入口摆成一张表请你勾选，确认后逐项冒烟，生成 `manifest.md`。报告会按你的实际组合说清楚：解锁了什么、缺什么、最便宜的补齐路径是什么。全新机器什么都检测不到时，改为列出可选入口和安装命令。

**第二步，演示派发（可选）。** 回一句"跑一次演示"，它用一段内置的含缺陷代码（不碰你的项目文件）向两家厂商各发一次盲审，然后给你看两家的共识、分歧和每路的 token 账单。日常工作就是这个样子。

**第三步，真实任务。**

```text
使用 $ai-cross：实现 XX 功能，完成后让另一家厂商的模型交叉审查。
```

### 手动安装

把仓库 clone 到对应 skills 目录，改名 `ai-cross` 即可：

```text
# Claude Code
git clone https://github.com/keros68/ai-cross ~/.claude/skills/ai-cross
# agents/*.md 另拷到 ~/.claude/agents/

# Codex
git clone https://github.com/keros68/ai-cross ~/.codex/skills/ai-cross
```

Windows 上 `~` 即 `C:\Users\<用户名>`。装完新开会话生效。其他支持 `SKILL.md` 的 agent 把仓库放进对应 skills 目录即可；不支持 skill loader 的，把 `SKILL.md` 当项目规则或 agent instruction 用。

## 密钥安全

本地只读、不外传、不回显、不写入仓库和留痕文件、不进模型上下文；调用外部通道时密钥只按进程注入给对应子命令。完整规则见 [`references/security.md`](references/security.md)。

## 文件结构

- `SKILL.md` - 路由规则、执行闭环、稳健性规则（决策核心）。
- `references/setup.md` - 盘点与接入向导，含组合解锁表和演示派发。
- `references/channels.md` - 各通道命令模板与漂移维护。
- `references/evidence.md` - 各规则背后的实测数据，样本量与局限如实标注。
- `references/dispatch-design.md` - 派发设计与并行边界。
- `references/security.md` - 密钥处理规则。
- `references/playbooks.md` - 可选编排方案。
- `references/cc_switch.py` / `verify_model.py` / `usage_probe.py` - 只读桥脚本。
- `agents/` - Claude Code 用的 scout / worker / heavy / advisor 定义。
- `qoder/` - Qoder 宿主适配。

## Attribution and Redistribution

This project is the original ai-cross skill by keros68:

https://github.com/keros68/ai-cross

The project is released under the MIT License. Redistribution, forks, modified versions, and repackaged copies must preserve the copyright notice and license text. Please do not present modified copies as the original project or imply endorsement by the original author.

## English

ai-cross is an AI-agent skill for cross-vendor model review and tiered task dispatch. It routes scanning, implementation, and critical review to different model tiers, then has models from different vendors check important outputs. It is not a majority-voting tool: it keeps an audit trail in `.dispatch/`, reports agreement and disagreement, and requires independently verifiable facts (numbers, code behavior, formatting) to be validated by tools or humans.

Quick start:

```text
Install this skill from GitHub and use it for cross-vendor model review and task dispatch:
https://github.com/keros68/ai-cross
```

Then open a new agent window and run:

```text
Use $ai-cross to inventory my available models.
```

First run auto-detects installed agent CLIs and cc-switch providers (read-only, keys are never shown), asks you to confirm once, smoke-tests each entry, and writes `manifest.md`. An optional demo dispatch then sends a built-in code sample to two vendors for blind review, so you can see consensus, divergence, and per-route token cost before running real tasks.

## License

MIT. See [LICENSE](LICENSE).

---

**同系列 Agent Skills**：[sci-select](https://github.com/keros68/sci-select)（选刊+投稿前审查） · [academic-reference-matcher](https://github.com/keros68/academic-reference-matcher)（文献引用） · [abstract-fig](https://github.com/keros68/abstract-fig)（图形摘要） · [cugb-doctoral-thesis-format](https://github.com/keros68/cugb-doctoral-thesis-format)（学位论文格式）｜全览见 [keros68](https://github.com/keros68)

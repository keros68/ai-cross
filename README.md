# ai-cross — 跨厂商交叉验证 + 分层派发 skill

核心能力：**用不同厂商的模型交叉验证关键产出**——抓出单模型（以及任何同厂商多 agent，如 Claude Code 的 ultracode）抓不出的系统性错误。顺带做分层派发（粗活走便宜档），并全程留痕可审计。

对订阅 / 包月用户，分层省下的是**会限流的额度**，不是美元；跨厂商交叉验证才是别处没有的能力。

支持宿主：Claude Code、Codex、Cursor、WorkBuddy、Qoder 等**任何能跑 shell 命令**的 agent。

## 三条实测结论（它们决定了本 skill 长什么样）

本项目做了三轮基准（含 4 个自造硬任务，避训练集污染；仪器先过 13 项正/负样本自检）。结果推翻了几条"常识"：

1. **「档位」几乎不影响正确率。** 最便宜档在全部自造硬任务上与最贵档持平；最贵档只多烧约 2× 推理 token。
   → 所以**默认往最低档派，验证失败再升级**，别预防性上高档。

2. **真正决定成败的是「要不要模型亲自算」。** 同一个模型：写校验和算法的**代码** 100% 正确（算术交给机器），但让它**亲自**迭代 51 步只有 33%，**肉眼数**数量也只有 33%。而且**所有**模型的自算都会随机出错——最贵的那个也会。
   → 铁律：**要精确数值就让模型写代码；做不到就自己跑一遍验证。绝不采信任何模型的自算。**

3. **固定开销是「壳」的，不是「模型」的。** 同一个 haiku、同一个空任务：内部 subagent **3,963** token，跨 CLI 外派 **30,230** token，差 **7.6×**。
   → **有内部通道先用内部**（分层几乎免费）。跨 CLI 外派的唯一正当理由是**换厂商**做交叉验证——为省钱而外派小任务多半是净亏损。

完整数据与"本数据不能支持的结论"见 `bench/RESULTS-HARD-20260708.md`。

## 你需要什么

- **至少一个** AI CLI 订阅（Claude Code 或 Codex 任一即可起步）
- 想解锁**交叉验证**：需要 ≥2 个**不同厂商**的模型（如 Claude + Codex，或 Codex + GLM）
- 想接 GLM / Kimi / DeepSeek 等：一个对应的 API key（skill 会引导你，只需粘贴）

> 只有 1 个厂商也能用——分层派发照样生效（省的是额度），只是交叉验证这个主功能会如实告诉你"不可用"。

## 安装

### Claude Code
```
skill/ai-cross/  →  ~/.claude/skills/
agents/ 下 3 个 .md    →  ~/.claude/agents/
```
Windows 上 `~` 即 `C:\Users\<用户名>`。装完新开会话生效。

### Codex
```
skill/ai-cross/  →  ~/.codex/skills/
```
（`agents/` 是 Claude Code 专用的内部通道，Codex 不需要——skill 会自动全走外部命令。）

### 其他 agent
支持 SKILL.md 的照各自 skills 目录放；不支持的，把 `SKILL.md` 内容加进项目规则文件（如 `AGENTS.md`）。

## 三步上手

**第 1 步 · 盘点**，对 agent 说：
```
盘点模型
```
它会问你有哪些订阅（申报制，不乱扫你的电脑），逐个冒烟验证，生成能力清单。
装了 [cc-switch](https://github.com/farion1231/cc-switch) 的话它会**只读**你已有的配置，**一个字都不用重填**。

**第 2 步 · 正常派活**：
```
用 ai-cross：扫描这个项目，实现 XX 功能，做交叉审查
```

**第 3 步 · 复查**（可选）：每次派发和每轮 review 的完整过程都存在项目的 `.dispatch/` 目录下，想看哪一路直接打开对应 `.md` 文件。

## 派发强度

| 强度 | 何时用 | 要求 |
|---|---|---|
| **标准**（默认） | 日常 | 无 |
| **双保险** | 科研分析默认；防幻觉 | ≥2 个不同厂商 |
| **全力** | 明说"彻底/全面/ultra"才启用 | 烧多份额度，慎用 |

## 密钥安全

**本地只读、绝不外传、不回显（一律打码）、不写入本 skill 产出的任何文件、不进模型上下文（只注入子进程环境变量）、读凭据库前告知你。**

完整规则见 `skill/ai-cross/references/security.md`——**读一遍就能审计**它对你的 key 做了什么。`cc_switch.py` 把这些做成了代码级保证：token 只在脚本子进程内部读取注入，连编排的 agent 都看不到它。

建议只从可信来源获取本 skill，避免运行被篡改的副本。

## 升级阶梯（按实测收益排序，不是按传统 cascade）

低档产出没通过验证时，**依次**试：

| 顺序 | 动作 | 实测收益 | 代价 |
|---|---|---|---|
| ① | 查输入是否送达（多行 prompt 被截断？） | 曾把整套基准误判为 0% | 零 |
| ② | 改任务框架：让模型写代码，别让它心算 | 33% → 100% | 零 |
| ③ | 换厂商（失败模式独立） | 修复系统性缺陷 | 一次重派 |
| ④ | 咨询顾问一次（advisor） | 未被检验 | 2.1× input |
| ⑤ | 升推理强度 / 升档位 | **零准确率增益** | 约 2× 推理 token |

**传统 cascade 把 ⑤ 当第一手；我们的数据说它是最后一手。**

## 设计原则（想改造它的人看）

- 多模型协作一律走 **CLI 命令派工**，不绑定任何宿主的内部机制
- 外部通道抽象为"**命令模板 + 模型参数**"：不想用某个 CLI 就换掉模板，其余逻辑不变
- **模型名会过时**：所有型号集中在 `references/channels.md` 一处，漂了改那里
- **共识不等于正确**：三个模型可以对同一个结论达成一致，而其中一个的推导是坏的（实测）。可独立验证的量由编排者**亲自跑一遍**，不采信任何模型的自述
- **先证明尺子是准的**：判定模型失败前，先 dump 出它实际收到的输入
- review 循环 ≤3 轮；双跑分歧**不仲裁**，并列证据交人裁决

## 文件清单

```
README.md                                    本文件
skill/ai-cross/SKILL.md                核心（通道、门槛、路由表、闭环、稳健性）
skill/ai-cross/references/setup.md     申报式接入向导 + CLI 安装附录
skill/ai-cross/references/channels.md  各通道命令模板 + 漂移维护
skill/ai-cross/references/cc_switch.py cc-switch 只读桥（list / exec）
skill/ai-cross/references/verify_model.py 模型真身校验器（打端点比对 model 字段，抓静默降级）
skill/ai-cross/references/security.md  密钥六铁律（可审计）
skill/ai-cross/references/dispatch-design.md  并行边界/能力判定/框架参照
skill/ai-cross/references/playbooks.md 编排方案复用（可选）
agents/scout.md   低档侦察员（Haiku，只读）— 仅 Claude Code
agents/worker.md  中档执行者（Sonnet）— 仅 Claude Code
agents/heavy.md   高档审查/架构（Opus high，只读）— 仅 Claude Code
```

（`manifest.md` 是你本机的能力清单，盘点时自动生成，不随包分发。）

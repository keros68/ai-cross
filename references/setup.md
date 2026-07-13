# 接入与盘点向导（申报制）

面向不会安装配置的新人。逐步执行，每步失败就停下报错，不要跳步、不要盲扫本机。

## 第 0 步 — 识别宿主

判断当前跑在哪个 agent 里。**宿主自己不算外部通道。**

- 宿主是 **Claude Code**：内部 subagent 三档可用；claude CLI 仍可作 coding plan 载体（分支 B）。
- 宿主是 **Codex / WorkBuddy / Qoder / 其他**：**无内部通道**，全部走外部命令。盘点阶段第一个实际动作是**询问用户已有订阅/API**，不要尝试内部派发。

**壳 ≠ 模型**：zcode(ZCode)、Qoder 桌面、Hermes、OpenClaw、WorkBuddy 这类是 harness/壳，不是可派发的模型。派发目标永远是**模型**，经三种方式之一触达：①官方 CLI（claude/codex/gemini/qoder）②Anthropic/OpenAI 兼容端点（GLM/Kimi/DeepSeek）③按量 API。用户报"我有 zcode/z.ai"时，其底层模型是 GLM，走分支 B 直连智谱端点，**不需要装它的桌面，也不需要经过任何路由壳**。只有桌面 GUI、无 CLI 也无 API 的工具无法被任何方式派发（路由壳也救不了——它自己也得靠 API 触达模型）。

## 第 1 步 — 申报

用当前宿主的交互提问机制让用户勾选自己拥有的订阅/API：

| 宿主 | 提问机制 |
|---|---|
| Claude Code | `AskUserQuestion`（支持多选） |
| Codex | `request_user_input`；不可用时退化为逐条文本提问 |
| 其他 | 逐条文本提问 |

**推荐提问模板**（可直接复用）：

> 请勾选你已有的模型入口（可多选）：
> ① Claude（Claude Code / Pro / Max 订阅）
> ② Codex（ChatGPT 订阅）
> ③ Gemini（⚠️ 独立 CLI 已下线，2026-07 核实；若未来恢复见 `channels.md`）
> ④ Qoder（CLI 与 IDE 共享 Credits，算一个源）
> ⑤ CodeBuddy / WorkBuddy（同账号通用，算一个源）
> ⑥ 智谱 GLM Coding Plan（订阅制 → 分支 B）
> ⑦ Kimi 会员 / Kimi Code（订阅制 → 分支 B）
> ⑧ 按量 API（DeepSeek / OpenRouter / 其他 → 分支 C）
> ⑨ 其他 agent 或模型（请说明名字）

### 可选：从 cc-switch 预填（本机已装时优先，用户零重输）

很多多模型用户用 [cc-switch](https://github.com/farion1231/cc-switch) 管配置。它把用户**手动添加**的各 CLI 供应商存在 `~/.cc-switch/cc-switch.db`（SQLite）。注意：这是用户申报过的清单，不是探测结果——信任级别同申报，好处是免重输。

- **固化做法**：运行随附只读桥
  ```
  python <本skill目录>/references/cc_switch.py list
  ```
  Windows 上若 `python` 不存在，改用 `py -3 <...>/cc_switch.py list`。
  输出每个 provider 的 `app_type / name / category / endpoint / tier_models(档位→模型) / has_token`，**token 从不输出**。据此建 manifest 与厂商×档位矩阵。**无 python / 读不到 / schema 对不上 → 回退手动申报，不阻塞。**

- **派发时**用同一脚本的 exec 模式：
  ```
  python cc_switch.py exec --provider "Zhipu GLM" --tier sonnet --task "..."
  python cc_switch.py exec --provider "Zhipu GLM" --tier sonnet --task-file task.txt
  ```
  任务含引号/花括号/换行时**务必用 `--task-file`**（shell 会拆碎 `{}` 和转义引号，实测踩过）。token 只在脚本子进程内读取注入，**绝不进主 agent 上下文**。仅支持 `app_type=claude`（Anthropic 端点）；codex/gemini 官方订阅走各自 CLI。

- **实测边界（cc-switch v3.8+，2026-07-08）**：有保存 key 的 provider，token 明文存于 `settings_config.env.ANTHROPIC_AUTH_TOKEN`（cn_official 订阅模板 GLM/StepFun 与 custom 类讯飞均如此，cc-switch 不加密第三方 key）。空/陈旧条目无 token → 端点+模型仍可自动填，key 让用户补。
- **额外红利**：env 里的 `ANTHROPIC_DEFAULT_HAIKU/SONNET/OPUS_MODEL` 就是**用户配的当前档位→模型映射**，直接当该 provider 的 tier→model 表用，模型 ID 取实时值、免猜、免维护漂移。
- **铁律**：只读，绝不修改其 db；**绝不采用它的"切换"机制**（它靠把当前供应商写进 `~/.claude/settings.json` 来切换、一次只激活一个，正是要避开的全局污染）。我们的价值恰是把它存的多个供应商用**按进程环境变量并发跑起来**。提取到的 key 只在派发时按进程注入，manifest 只记"来源=cc-switch / 是否可自动提取"。
- 把读到的列表拿给用户勾选要纳入的，再写 manifest。

## 第 2 步 — 逐项验证（只验证勾选项）

### 分支 A：官方 agent CLI

每项三小步：存在检测 → 登录 → 冒烟。

| CLI | 存在检测 | 冒烟（最便宜档） | 模型枚举 |
|---|---|---|---|
| claude | `claude --version` | `claude -p --model haiku "只回复OK"` | 不支持，用静态档位表 haiku<sonnet<opus |
| codex | `codex --version` | `codex exec -m gpt-5.4-mini -c model_reasoning_effort="low" -s read-only --skip-git-repo-check "reply OK"` | 不支持，静态：mini<标准<深度 |
| gemini | ⚠️ 独立 CLI 已下线（2026-07 核实，见 `channels.md`），跳过验证 | 若未来恢复：先 `gemini --version` 冒烟确认存在再用 | 不支持，静态表 |
| qoder | `qoder --version` | `qoder -p "只回复OK"`（参数以本地 `qoder --help` 为准） | 静态，`--model` 选档 |
| codebuddy | `codebuddy --version` | 命令形态以本地 `codebuddy --help` 为准 | 静态表 |
| hermes | `hermes --version` | `hermes chat -q "只回复OK" -Q` | 以其配置为准，额度归属问用户 |

- **档位与当前偏好**：官方 CLI 不支持枚举，也不需要——档位少且顺序由厂商定义，用静态表。用户在 TUI `/model` 里选的型号**会落盘**：读 `~/.claude/settings.json` 的 `model`、`~/.codex/config.toml` 的 `model` / `model_reasoning_effort`，作为"用户当前偏好"记入 manifest（读文件即可，无需进 TUI）。**字段缺失时只记录版本与可用性，不推断偏好模型。**
- **CLI 不存在**：按下方附录给出安装命令，征得同意后代为安装；装不了则记"不可用 + 原因"。
- **版本**：记录版本号，**不自动升级**（可能有破坏性变更）；仅冒烟失败且疑似过旧时才建议升级。
- **认证错误**：agent 无法代替用户完成 OAuth 登录——提示用户在自己终端运行登录命令（`claude` 首次启动 / `codex login` / `gemini` 首次启动），完成后回来重测。
- **参数报错**（`unrecognized arguments` 等）：跑 `<cli> --help` 看当前参数，去掉非必要项用最小命令重试，并提示更新 `channels.md`。

### 分支 B：coding plan（GLM / Kimi 等订阅）

载体是 claude CLI + 按进程环境变量。**不要用 aichat 接 coding plan**——多数条款限定其只能用于 coding agent，且网关常拒非 agent 流量。

1. 确认本机有 claude CLI，没有则先装（**免费；装了不等于要买 Claude 订阅**，第三方端点用 env 覆写认证）。
2. 引导用户到对应控制台创建 API key（智谱：开放平台 → 个人编程套餐 → 套餐概览建 key；Kimi：Kimi Code 控制台）。此 key 与 zcode/ZCode 桌面用的是同一个、同一份订阅额度池，直连端点即可，无需装桌面。
3. key 存为用户级环境变量，不落明文：Windows `setx GLM_CODING_KEY <key>`（Kimi 同理），类 Unix 写入 shell profile。**若 key 已在 cc-switch 里，直接用 `cc_switch.py exec`，跳过本步。**
4. 用 `channels.md` 的 coding plan 模板冒烟。
5. 记入 manifest，额度归属为对应 coding plan，算独立源。

### 分支 C：按量 API（aichat，最后的兜底）

1. **检测**：`aichat --version`，已装跳到第 3 小步。
2. **代为安装**：Windows `winget install sigoden.aichat`（失败试 `scoop install aichat`）；macOS `brew install aichat`；Linux 从 https://github.com/sigoden/aichat/releases 下载二进制入 PATH。全部失败才让用户手动装并停在这步。
3. **收集三项**：先选预设，预设内只需粘贴 API key：

| 预设 | api_base | 默认模型 ID |
|---|---|---|
| DeepSeek | `https://api.deepseek.com/v1` | `deepseek-chat` |
| 智谱按量 API（非 Coding Plan） | `https://open.bigmodel.cn/api/paas/v4` | 以官网当前型号为准 |
| OpenRouter | `https://openrouter.ai/api/v1` | 用户指定 |
| 自定义 | 用户填 URL | 用户填模型 ID |

4. **写配置**：Windows `%APPDATA%\aichat\config.yaml`；macOS `~/Library/Application Support/aichat/config.yaml`；Linux `~/.config/aichat/config.yaml`。**已存在则先读再追加 client，绝不整体覆盖**：

```yaml
model: <name>:<model_id>
clients:
  - type: openai-compatible
    name: <name>
    api_base: <api_base>
    api_key: <api_key>
```

5. **冒烟**：`aichat -m <name>:<model_id> "只回复两个字：正常"`。已配置的全部模型可用 `aichat --list-models` 枚举。

## 第 3 步 — 写 manifest

写入本 skill 目录下 `manifest.md`，带盘点日期。**复制同目录现成 `manifest.md` 的表头填写**；若不存在，用此最小模板：

```markdown
# 能力清单 manifest
盘点日期:YYYY-MM-DD ｜ 宿主:<宿主名>

| 通道 | 模型/档位 | 强项 | 相对成本 | 额度归属 | 冒烟结果 |
|---|---|---|---|---|---|
| codex CLI | gpt-5.4-mini(low) | 快速琐事 | 低 | Codex 订阅 | ✅ YYYY-MM-DD |
| cc_switch→Zhipu GLM | glm-5-turbo / glm-5.2 / glm-5.2[1m] | 实现/分析 | 低/中/高 | GLM Coding Plan | ✅ YYYY-MM-DD |
| gemini | — | — | — | — | ❌ CLI 未安装 |

## 厂商 × 档位矩阵(路由查这张表)
| 厂商 | 低档 | 中档 | 高档 |
|---|---|---|---|
| OpenAI(codex) | gpt-5.4-mini | gpt-5.4 | gpt-5.5 |
| 智谱(cc_switch) | glm-5-turbo | glm-5.2 | glm-5.2[1m] |

## 源与强度解锁
独立厂商数:N → 分层省钱 ✅ ｜ 交叉验证(≥2 厂商) ✅/❌ ｜ 全力 ✅/❌
```

- 验证失败的勾选项也记入（标"不可用 + 原因"），避免下次重复试错。
- 末尾写明独立**厂商**数及解锁的强度。源仅用于计费提示与独立性判断，不作强度门槛。

### ⛔ 不要给通道打「自算可靠性」评级（实测：这个抽象是错的）

曾计划用一道递推题给每个通道打 `自算 k/n` 分。**实测证明该评级不稳定、会误导**：

| 同一道 51 步递推题 | GLM | StepFun |
|---|---|---|
| 硬基准（n=3） | 3/3 | 1/3 |
| 独立探针（n=3） | **1/3** | **2/3** |
| 合计 | 4/6 | 3/6 |

**完全反转，都接近抛硬币。** 对照 `tools` 字段：答对的那几次模型**写了代码去跑**，答错的那几次它**心算**了。

**「自算准不准」不是厂商属性，是「它这次选没选择写代码」的随机结果。** 而探针里那句「你可以心算、手算或写代码，方式不限」正是邀请失败的元凶。

**→ 正确做法（见 SKILL.md 铁律）**：不评级，而是在**每次派发精确计算任务时**，prompt 明确命令「写代码并执行，给出运行结果」；模型不能执行代码时，**编排者独立核验**。

manifest 只记**可复现的具体缺陷**（如"肉眼计数 33%"、"统计推导方法标注错误"），不记自算评分。

**判定"它不会算"之前，先排除传参问题**：多行 prompt 经 argv 传给 `codex exec` 会在首个换行截断（见 `channels.md`）。**先确认它收到了完整题目。**

## 第 4 步 — 报告

告知用户：清单写在哪、有几个厂商、解锁了什么强度、路由表中哪些任务会走哪些通道、哪些通道未冒烟需首次派发前先测。

---

## 附录：常见 CLI 安装命令

| CLI | Windows | macOS | Linux |
|---|---|---|---|
| claude | `npm i -g @anthropic-ai/claude-code` | 同左 | 同左 |
| codex | `npm i -g @openai/codex` | 同左 | 同左 |
| gemini | ⚠️ 独立 CLI 已下线（2026-07 核实，见 `channels.md`），不建议安装；若未来恢复再执行 `npm i -g @google/gemini-cli` | — | — |
| aichat | `winget install sigoden.aichat`（或 `scoop install aichat`） | `brew install aichat` | [releases](https://github.com/sigoden/aichat/releases) 下载二进制入 PATH |

以上为通用形态，**以各家官方文档当前值为准**；安装前先跑 `<cli> --version` 确认是否已装。装完让用户自行完成 OAuth 登录（agent 代替不了）。

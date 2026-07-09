# 外部通道命令模板

抽象定义："命令模板 + 模型参数"。换任何等价 CLI 只需替换模板，其余逻辑不变。模型名会过时，以各 CLI 当前版本为准替换。

## Codex 三档

```bash
# 低档：快速琐事
codex exec -m gpt-5.4-mini -c model_reasoning_effort="low" -s read-only \
  --skip-git-repo-check "[任务]" 2>/dev/null

# 中档：常规第二意见
codex exec -m gpt-5.4 -c model_reasoning_effort="medium" -s read-only \
  --skip-git-repo-check "[任务]" 2>/dev/null

# 高档：深度分析/关键审查
codex exec -m gpt-5.5 -c model_reasoning_effort="xhigh" -s read-only \
  --skip-git-repo-check "[任务]" 2>/dev/null

# 代码特化档（可选）
codex exec -m gpt-5.3-codex-spark -c model_reasoning_effort="high" -s read-only \
  --skip-git-repo-check "[任务]" 2>/dev/null

# 长文本走 stdin（会作为 <stdin> 块附加到 prompt 后）
cat file.txt | codex exec -m gpt-5.4 -c model_reasoning_effort="medium" -s read-only \
  --skip-git-repo-check "总结要点" 2>/dev/null
```

- **Codex 支持 stdin 管道**（实测 2026-07-08：`echo "1,2,3" | codex exec ... "求和"` → 正确返回）。`--help` 明载：stdin 被 pipe 时作为 `<stdin>` 块附加；prompt 用 `-` 亦可全部从 stdin 读。
- 推理强度旋钮：`-c model_reasoning_effort="low|medium|high|xhigh"`。
- 不要用 `--full-auto`：当前 `codex exec --help` 已不列出该参数（虽仍被接受，属未文档化遗留别名，随时可能移除）。`-s read-only` 已够。
- **`gpt-5.3-codex` 不存在**（实测 2026-07-08：ChatGPT 账号报 `The 'gpt-5.3-codex' model is not supported when using Codex with a ChatGPT account`）。ChatGPT 订阅可用型号实测为：`gpt-5.4-mini` / `gpt-5.4` / `gpt-5.5` / `gpt-5.3-codex-spark`。**这是"未冒烟就别假设可用"的活教材**——该型号曾被本文件当作高档默认，直到实测才发现全线不可用。

## ⚠️ 已停用/不适用的通道（2026-07 核实，勿再假设可用）

- **gemini / qoder / codebuddy CLI**：这些产品的独立 CLI **已下线**（用户 2026-07 核实）。曾经的命令模板（`gemini -m …` / `qoder -p …` / `codebuddy …`）**不再有效**，别再照抄。若将来它们恢复 CLI，先 `<cli> --version` 冒烟确认存在再用。
- **Hermes**：`hermes chat -q "…" -Q`（跨供应商路由壳）——需单独部署，多数场景不划算（多一层壳、多一层折损）。仅当用户已在某机器上部署好、且明确要用时才走。

结论：**当前实测可用的外部通道就是 `codex exec` + coding plan（claude -p + 端点覆写）+ cc_switch 桥 + 裸 API/aichat**。上面这些是历史遗留，保留仅为说明"命令模板+模型参数"抽象可随时接新 CLI。

## coding plan（GLM / Kimi 等，载体为 claude CLI）

环境变量**按进程生效**：每次派工新开子进程并只给它设覆写变量，宿主会话登录不受影响（宿主是 Claude Code 也不冲突）。key 引用用户级环境变量，不落明文。

**安全铁律（不遵守就会串官方订阅）**：`ANTHROPIC_BASE_URL` / `ANTHROPIC_AUTH_TOKEN` 只能作为**子进程环境变量临时传入**，**绝不写进 `~/.claude/settings.json` 的 env 或任何全局配置**——一旦写全局，所有 claude 调用（含官方订阅的宿主会话）都会被重定向到第三方端点，这是最常见的冲突根因。智谱等厂商的"一键助手"常写全局配置，若用户已被它改过，盘点时提示用户清掉 settings.json 里的这两行。

只对 **Anthropic 协议端点**有效（GLM/Kimi/DeepSeek-anthropic 等）；GPT 走 codex、Gemini 走 gemini，不要硬塞进 claude CLI。覆写是否成功盖过 OAuth 因 CLI 版本而异，接入时务必冒烟测试确认走到了第三方端点。

**⚠️ 冒烟判据必须是「回答的是不是它」，不是「有没有回答」（2026-07-09 实测，血泪）**：GLM 的 Anthropic 兼容端点对**格式合法但它不提供**的模型名（如任何 Anthropic 官方 ID、`haiku`/`sonnet`/`opus` 别名）**不报错，而是静默用 `glm-4.7` 应答**；只有完全无法解析的名字才吃 400。后果：你以为在用 opus 档，其实拿到的是最便宜模型的回答，全程零报错。
- **冒烟不能只看"有回复"**——必须直接打 `{base}/v1/messages`，比对**响应体的 `model` 字段**与请求是否一致；不一致即静默降级，结论记入 `manifest.md`。现成工具：`python <本skill目录>/references/verify_model.py --provider "<cc-switch里的名字>"`（只读 cc-switch 取端点，逐档打端点比对真身；也可 `--models "a,b,c"` 测指定 ID）。**连官方文档给的 ID 也要过这一关**——实测出现过文档说可用、该账号却 400 的情况（如某些 1M 长上下文版按套餐开通）。
- CLI 的 `modelUsage` 记的是**请求值**不是服务端返回值，**不能**用来判断真身。
- 复现脚本见 `FINDINGS-glm-model-id.md`（项目根）。

**已验证（claude 2.1.204 实测，2026-07-08）**：子进程覆写 `ANTHROPIC_BASE_URL`+token 时，CLI 明确以注入 auth 源**优先于 claude.ai 登录**（会打印一行提示说明这点）；实测宿主会话登录、`~/.claude/settings.json`（env 保持 `{}`）、宿主进程环境三者均不受影响；被重定向的子进程与走官方订阅的子进程可**并发共存、互不干扰**。隔离是操作系统进程级的，前提是覆写只按子进程传、不写全局配置（见上条铁律）。

```bash
# 智谱 GLM Coding Plan（zcode/ZCode 壳底层就是这个模型，直连端点即可，不碰桌面）
# 变量名是 AUTH_TOKEN 不是 API_KEY。
# 模型 ID 必须小写、精确（官方文档 docs.bigmodel.cn/cn/coding-plan，2026-07-09 核实）：
#   在售仅三个：glm-4.7 / glm-5-turbo / glm-5.2；glm-5.2[1m] = 5.2 的 1M 长上下文版（方括号小写 m 是 ID 一部分）
#   ⚠️ glm-5.1 / glm-5 已下线（调用会被静默切到 glm-5.2）；大写 [1M] 会吃 400；不认识的名字静默降级到 glm-4.7
#   下面是示例，具体以官方文档 / 你 cc-switch 里的实时配置为准
ANTHROPIC_BASE_URL=https://open.bigmodel.cn/api/anthropic ANTHROPIC_AUTH_TOKEN=$GLM_CODING_KEY \
  claude -p --model glm-4.7 "[任务]"          # 粗活
ANTHROPIC_BASE_URL=https://open.bigmodel.cn/api/anthropic ANTHROPIC_AUTH_TOKEN=$GLM_CODING_KEY \
  claude -p --model "glm-5.2[1m]" "[任务]"    # 硬活/长上下文（1M），常规硬活用 glm-5.2 即可

# Kimi Code（模型 ID 统一 kimi-for-coding）
ANTHROPIC_BASE_URL=https://api.kimi.com/coding/ ANTHROPIC_API_KEY=$KIMI_CODING_KEY \
  claude -p --model kimi-for-coding "[任务]"
```

PowerShell 宿主下用 `cmd /c "set ANTHROPIC_BASE_URL=… && set ANTHROPIC_AUTH_TOKEN=… && claude -p …"` 保证变量只作用于子进程。端点 URL 以各家官方文档当前值为准。

**⚠️ API 错误可能伪装成正常回答**：`claude -p` 在 API 报错时（如 529 过载、或 400「模型 ID 不存在」）**仍可能 exit 0**，并把错误文本塞进 `result` 字段。必须用 `--output-format json` 并检查 `is_error` / `api_error_status`，否则会把 `"API Error: 400..."` 当成模型答案交付。`cc_switch.py` 现已**始终**走 json 并在**任何**模式下校验（错误时 exit 8、stdout 为空），不再只在 `--usage` 分支检查。自己写命令模板时务必同样处理——纯文本直连时也要看响应体是不是 error。

**若 key 已在 cc-switch 里**：优先用 `python <本skill目录>/references/cc_switch.py exec --provider "<名字>" --tier haiku|sonnet|opus --task "..."`——它只读取 token 注入子进程、绝不回显进上下文，档位→模型映射也直接取 cc-switch 里用户配的实时值（免手填模型 ID），并默认加只读护栏（见下）。这是 B 类用户的首选派发方式。

### 只读护栏：`claude -p` 用 `--tools`，不是 `--permission-mode`

被派发的模型**继承宿主的 cwd**。它们只出意见、不落盘（落盘由宿主 agent 负责），所以必须显式剥夺写能力，否则并发派发时几个子进程会在同一个工作区里互相覆盖。

`codex exec` 用 `-s read-only`。`claude -p` 的等效物是 **`--tools` 白名单**：

```bash
# 只读咨询/代码审查：能读能搜，不能写
echo "[任务]" | claude -p --model X --tools Read,Grep,Glob
# 纯文本任务：禁用全部工具，最省
echo "[任务]" | claude -p --model X --tools ""
```

**实测（claude 2.1.205，2026-07-09）**：

- `--permission-mode` **没有只读档**（枚举只有 `acceptEdits/auto/bypassPermissions/manual/dontAsk/plan`），别指望它。`plan` 档虽禁编辑，但会改变模型输出形态，不适合当咨询通道。
- `--tools` 作用在**工具注册层**，比权限层更硬：`--tools Read,Grep,Glob` 叠加 `--permission-mode bypassPermissions`，写依然失败（`Write exists but is not enabled in this context`）。**bypass 压不过白名单。**
- 该护栏是 CLI 本地行为，**与端点无关**：GLM 覆写端点下同样生效（已冒烟）。
- 不加任何参数时，默认 `-p` 也会拒写——但那是靠"非交互无法确认"兜底，且**被拦截时 exit 0**、把"需要你批准"当成正常答案返回（与下方 529 伪装同源）。**别依赖默认值，显式写死。**
- ⚠️ **`--tools` 是 variadic**，`claude -p --tools Read,Grep,Glob "任务"` 会把任务当成第四个工具名吞掉，然后报 `Input must be provided either through stdin or as a prompt argument`——看着像"没给 prompt"，实为参数被吞。**prompt 一律走 stdin。**

`cc_switch.py` 已默认 `--tools Read,Grep,Glob` 并把 task 经 stdin 送入；纯文本任务传 `--tools ""`。

**合规注**：多数 coding plan 条款限定用于 coding agent（Claude Code 等）。本通道载体就是 claude CLI，属限定范围内的用法；**不要**用 aichat 直连 coding plan 端点（可能违反条款，网关也常拒非 agent 流量）——aichat 只兜按量 API。

## 纯文本任务：裸 API 直调（最省，地板 ~11 token）

纯文本任务（分类/摘要/翻译/抽取/自包含问答，**不需要工具**）**不该走 harness**，主 agent 直接打 OpenAI 兼容端点即可——无系统提示、无工具定义，input 地板实测 **11 token**（对比 `claude -p` ~30k）。

```bash
# OpenAI 兼容端点（DeepSeek/硅基流动/OpenRouter 等）
curl -s https://<host>/v1/chat/completions \
  -H "authorization: Bearer $API_KEY" -H "content-type: application/json" \
  -d '{"model":"<model>","max_tokens":512,
       "messages":[{"role":"user","content":"[任务]"}],
       "enable_thinking":false}'
# 取 .choices[0].message.content；推理模型 content 可能为空、答案在 .reasoning_content
```

- **`enable_thinking:false`（或各家等价参数）** —— 纯文本任务必加。实测推理模型开着思考纯烧 output：Qwen 抽取任务 512→18 token（28×）、且从错变对。不支持该参数的模型，把 max_tokens 给足（≥512）避免推理没跑完被截断。
- **批量合并**：50 条要分类的，拼成一次调用，别发 50 次——固定开销才摊得开。
- key 用用户级环境变量引用（`$API_KEY`），不落明文；**这是纯按量 API，不是 coding plan**（后者不能这么直连）。

## aichat（按量 API 的 CLI 封装，不想写 HTTP 时用）

`aichat` 是裸 API 的薄封装，适合不想手写 curl 的场景；它的系统提示极短，成本接近裸调用。功能同上，命令更短：

```bash
aichat -m <provider>:<model> "[任务]"
cat file.txt | aichat -m <provider>:<model> "总结要点"   # 长文本走 stdin
```

关思考的参数以 aichat 的 provider 配置为准（`enable_thinking` 等写进 config.yaml 的 `patch` 段）。

多数按量模型无推理强度旋钮；个别推理模型有专用参数，以 provider 文档为准。

## 复用与维护

**一次配好，永久复用**：API key（用户级环境变量 `setx`）、aichat config.yaml、manifest.md 三者都持久化，跨会话跨重启有效。之后每次派工自动读取，用户无需重输。每次派工"重新设置"的只有子进程那次性环境变量——自动、隐形，且正是隔离安全的来源，不算重复配置。key 过期/轮换时重跑一次 `setx` 即可（当前 shell 需重启才见新值，新开的 shell 直接生效）。

**模型 ID 漂移**（模型在迭代，如 glm-4.6→GLM-4.7→GLM-5.2、gpt-5.4→5.5）：
- 本文件是**唯一**需要维护模型 ID 的地方，漂了改这里即可，skill 其余逻辑不动。
- 派发命中"unknown model / 模型不存在"类错误：先去掉 `-m`/`--model` 用该 CLI **默认模型**重试（默认通常跟随当前版），再提示用户更新本文件对应行。
- 支持枚举的 CLI（`aichat --list-models`）以枚举为准；不支持的（claude/codex/gemini）以各家官方文档当前型号为准。

**CLI 参数漂移**（命令行参数被改名/移除，如本文件曾误用已下线的 `--full-auto`）：
- 命中 `unrecognized arguments` / `unknown option` / `error: unexpected argument` 类错误时：
  1. 跑 `<cli> --help`（或 `<cli> <子命令> --help`）看当前可用参数；
  2. 去掉所有非必要参数，用**最小可用命令**重试（只保留模型、沙箱/只读、必要的跳过检查）；
  3. 成功后提示用户更新本文件对应模板行，并在 manifest 备注该 CLI 版本。
- 任务文本含引号/花括号/换行时，**用文件传参**（如 `cc_switch.py --task-file`）或 stdin，别在 shell 里拼——PowerShell/cmd 会把 `{}` 和转义引号拆碎（实测踩过）。
- **⚠️ 多行 prompt 绝不能走 argv（Windows 实测）**：`codex exec ... "<多行文本>"` 会在**第一个换行处截断**，模型只收到第一行。它会（完全正确地）回答"你没把规格贴出来"，而你会误以为它能力不行。**正确做法：`codex exec ... -` 并把 prompt 从 stdin 送入**（`cat spec.txt | codex exec ... -`，或 Python `subprocess.run(cmd, input=prompt)`）。
  - 这个 bug 曾让一整套高难度基准误判为"codex 全线 0%"，而单行 prompt 的简单基准却全 100%——差点被错误归因为"目标 CLI 的技能污染"。**判定模型失败前，先确认它到底收到了什么。**

**CLI 二进制版本**：记录版本号、**不自动升级**（升级可能带破坏性变更）；仅冒烟失败且疑似过旧时才向用户建议升级命令。

**用户说「更新模型 / 升级清单」**：重跑盘点流程（`setup.md`）+ 逐 CLI 核对当前模型 ID（能枚举的枚举、不能的查文档或问用户）+ 刷新本文件与 manifest.md 的 ID 和日期。

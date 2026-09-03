---
name: headless-cli-agents
display-name: ZCode / Grok / Codex / Claude Code 命令行（headless）使用指南
description: 如何发现本机已安装的 agent CLI，并以无界面（headless）方式调用 ZCode / Grok / Codex / Claude Code，供脚本或其他 agent 做代码审查等任务。先按探活协议确认可用 CLI，再给抄命令模板；后半是各 CLI 的配置、参数、JSON 格式与排错。
version: 2.0.0
author: MieMieeeee
tags: zcode, grok, codex, claude, CLI, headless, multi-agent, 代码审查, agent协作
category: 工具使用
---

# ZCode / Grok / Codex / Claude Code 命令行（headless 模式）使用指南

给**其他 agent / 脚本**看的调用手册：先看 §1 有哪些 CLI，再抄 §2 的命令。ZCode / Grok / Codex / Claude Code 的细参数分别在 §3–§13 / §14 / §15 / **§16**。

- 本文所有 [已实测] 结论验证于 2026-09-03：zcode 0.16.5 / Grok CLI 1.0.5 / Codex CLI 0.153.0-alpha.5 / Claude Code 2.1.204（Windows 11, build 26200）。本技能**不设版本门槛**；版本不同时结论可能漂移，按 §1 的发现方法与各章自检清单重新验证。各章自检清单里的「期望输出」同样是当日快照，随本机与版本而变，不是合格标准。
- 验证环境：Windows 11 (win32 10.0.26200 x64)，Node v24.5.0，Git Bash / PowerShell
- 标注图例：**[已实测]** = 技能方在本机验证通过；**[外部实测]** = 验证 agent 实战数据；**[仅帮助文档]** = 来自 `--help` 输出、未实测；**[实测不可用]** = 当前版本拒绝，勿用

## 1. 发现已安装的 CLI agents

本技能覆盖的四家均支持 headless 单次跑完退出；**是否已安装以逐家探活为准**。短名不全在可靠 PATH 里，调用建议用全路径或 spawnSync 的 `cwd` 选项。[已实测]  本技能**不引导安装**——安装路径随厂商 / 平台 / 桌面版差异极大，已超出 headless 调用手册的职责；调用方按 §1 初始化协议 probe 后只使用实际存在的。

### 通用发现手段

短名可靠度按家区分：`zcode` / `grok` 不在 PATH；`codex` 的 PATH 指向过期哈希目录与旧 Store 包；`claude` 的 native installer 通常会把自己加进 PATH（`where.exe claude` 常可命中，脚本里仍建议全路径 / `Test-Path` 更稳）。逐家探活用：

| CLI | 探活命令 | 典型安装位置 |
|---|---|---|
| **ZCode** | `Test-Path "$env:LOCALAPPDATA\Programs\ZCode\resources\glm\zcode.cjs"` | 同左（ZCode 桌面版自带） |
| **Grok** | `Test-Path "$env:USERPROFILE\.grok\bin\grok.exe"` | 同左（xAI 官方 native installer） |
| **Codex** | 先 `Test-Path "$env:USERPROFILE\.codex\config.toml"`（不存在即未安装，勿直接 Select-String）；存在再 `Select-String -Path "$env:USERPROFILE\.codex\config.toml" -Pattern 'CODEX_CLI_PATH'` 取行、按 §2.3 / §15.1 拆出 `.exe` 全路径后 `Test-Path` | `%LOCALAPPDATA%\OpenAI\Codex\bin\<哈希>\codex.exe`（哈希随桌面升级变，**必须**从 `~/.codex/config.toml` 动态取） |
| **Claude Code** | `where.exe claude`（或 `Test-Path "$env:USERPROFILE\.local\bin\claude.exe"`） | `%USERPROFILE%\.local\bin\claude.exe`（Anthropic 官方 native installer） |

### 本技能深度文档化的四家

| CLI | 二进制 | headless 入口 | JSON 字段 | 续接方式 |
|---|---|---|---|---|
| **ZCode** | 见 §1 通用发现手段 / §3 | `--prompt` / `-p` | `--json` → 字段 `response` | `--resume sess_xxx` |
| **Grok** | 见 §1 通用发现手段 / §14.1 | `-p` / `--single` | `--output-format json` → 字段 `text`（stderr 有 WARN，**禁止 `2>&1`**） | `-r <sessionId>` |
| **Codex** | 见 §1 通用发现手段 / §15.1 | `exec`；长 prompt 用 stdin `-` | `--json` 是 **JSONL**；更稳加 `-o last.txt` 读最终回答 | `exec resume <thread_id>` |
| **Claude Code** | 见 §1 通用发现手段 / §16.1 | `-p` / `--print` | `--output-format json` → 字段 `result`（单个 JSON 对象） | `--resume <session_id>`（同目录实测可用，跨目录找不到，§16.7） |

### 默认模型怎么查

默认值随桌面 / 账号 / 路由配置变化，请直接读源而不是记硬编码：

| CLI | 怎么查 |
|---|---|
| **ZCode** | 读 `~/.zcode/cli/config.json` → `model.main`（格式 `<provider>/<model-id>`，例 `bigmodel/glm-5.3`） |
| **Grok** | 免费：docs / `--help` 口径；精确生效模型需一次最小调用，看 JSON `modelUsage` 块 key（付费，如 `grok-4.6-build`；`usage` 字段无 model 信息） |
| **Codex** | 读 `~/.codex/config.toml` → `model =` 行（路由 / cc-switch 会替换默认值，不一定等于官方 GPT） |
| **Claude Code** | 最便宜的确定方式是发一次最小调用，读 JSON `modelUsage` 块 key（实测约 $0.14）；或查 `~/.claude/settings.json`（路由机制下不保证等于实际生效模型） |

### 初始化协议（首次使用本技能时）

**Layer 1 — 免费、必跑**（按 §1 通用发现手段 逐家探活 + 按 §1 默认模型怎么查 读配置源），把结果汇报给上游 / 用户，如「本机可用 ZCode / Grok，Codex 当前未安装」。只做免费探测：`Test-Path` / 读 `config.json` / `config.toml` / `settings.json` / `--help`——**不发任何真实模型调用**；Grok / Claude 的精确生效模型在免费层拿不到，留给 Layer 2。

**Layer 2 — 付费、按需**（首次在一台新机器采用本技能，或 Layer 1 结果疑似错时跑）：对探活命中的每家发**一次**最小调用——抄对应自检清单的**单条**命令（§12 第 3 项 / §14.10 第 2 项 / §15.7 第 3 项 / §16.10 第 3 项），不要整节照跑（整节含多轮真实调用）。**Claude Code 的最小调用是四家里最贵的，验证时约 $0.14**；其余几乎免费。

### 调用方注意

- 打 `codex` / `zcode` / `grok` 当短命令都不可靠（Codex PATH 指向过期哈希目录与旧 Store 包，zcode / grok 压根不在 PATH）；`claude` 通常在 PATH（`where.exe claude` 常命中），但脚本里仍建议用全路径。
- 桌面升级后 Codex 的哈希子目录会变（v1.4.0 时的 `f79cfd43cf28a53a`、v1.4.1 时的 `8fffe69425752027` 目录均已随升级失效——这条**作为哈希机制漂移证据**保留）。发现当前二进制：在 `~\.codex\config.toml` 搜 `CODEX_CLI_PATH`，按 §2.3 / §15.1 的动态取路径一行；不要用 `...\OpenAI\Codex\bin\codex.exe` 根上那份（已知的旧副本 0.130）。
- Codex 的 `approval_policy` 由 `~/.codex/config.toml` 决定（验证时本机为 `never`）；review 类任务**必须**显式 `--sandbox read-only`，与 `approval_policy` 无关。不要给 `exec` 传 `--ask-for-approval`（clap 直接 exit 2）。
- 不要对 Codex 加 `--ignore-user-config` 去打 `api.openai.com`：若 `auth.json` 持 `sk-cp-...`（ChatGPT plan 风格）key 会 401（exit 1）。实际 auth 状态用 `codex login status` 查。
- WindowsApps 里的 `codex.exe` 仍是 ACL 锁死（Access is denied），不要 spawn 那条路径。

## 2. 给其他 agent 的推荐调用

统一约定（四家都适用）：

1. 复杂要求写成工作目录里的 `review_task.md`，命令行只留一句短指令（两跳；ZCode 细述见 §5.2）。ZCode / Grok 用 `--cwd`，Codex 用 `-C`，Claude Code 没有 `--cwd` flag（走 spawnSync 的 `cwd` 选项）。
2. 只读 review，禁止改文件。修复类等**要改文件**的任务走 Codex `--sandbox workspace-write`，模板与坑见 §2.5。
3. 程序化调用分管道 stdout/stderr，调大 maxBuffer，超时 15–20 分钟（修复类任务更长，实测 ~35 分钟，见 §2.5）。
4. 两跳的任务文件含中文时，**用 UTF-8 带 BOM 保存，或干脆写英文**：Codex 用它的 shell 工具读 UTF-8 无 BOM 中文文件会按 GBK 解码成乱码，带着乱码执行必然跑偏（实测坑，详见 §15.2）。

PowerShell 抄 §2.1–§2.3 / §2.6 这四条（修复类任务用 §2.5）。

### 2.1 ZCode（只读 review）

```powershell
node "$env:LOCALAPPDATA\Programs\ZCode\resources\glm\zcode.cjs" `
  --prompt "阅读 ./review_task.md 并严格执行其中的任务" `
  --cwd "E:/你的项目" `
  --json `
  --mode plan `
  --disallowed-tools "Edit Write Bash"
# stdout 是单个 JSON：.response = 最终回答，.sessionId 给 --resume
```

### 2.2 Grok（只读 review）

```powershell
& "$env:USERPROFILE\.grok\bin\grok.exe" `
  -p "阅读 ./review_task.md 并严格执行其中的任务" `
  --cwd "E:/你的项目" `
  --output-format json `
  --always-approve `
  --permission-mode plan `
  --disallowed-tools "Edit,Write,run_terminal_cmd"
# stdout 是单个 JSON：.text = 最终回答，.sessionId 给 -r。不要 2>&1
```

### 2.3 Codex（只读 review）

验证时本机 `~/.codex/config.toml` 默认 `sandbox_mode = "danger-full-access"`、`approval_policy = "never"`（详见 §15.1）；review **必须**显式 `--sandbox read-only`，与 `approval_policy` 无关——只看你的 `~/.codex/config.toml` 实际值。PowerShell 不要把带空格的 prompt 当 `Start-Process -ArgumentList` 元素（会拆词）；走 stdin `-`。

```powershell
# 哈希目录随桌面升级变化，从 config.toml 动态取当前二进制（已实测一行可用）
$CX = (Select-String -Path "$env:USERPROFILE\.codex\config.toml" -Pattern 'CODEX_CLI_PATH').Line.Split("'")[1]
# Windows PowerShell 5.1 必加：默认 $OutputEncoding 是 ASCII，中文经管道喂给 codex 会变 "???"（pwsh 7 无此问题）[已实测]
$OutputEncoding = [System.Text.Encoding]::UTF8
"阅读 ./review_task.md 并严格执行其中的任务" |
  & $CX exec --skip-git-repo-check --color never --json --sandbox read-only `
    -C "E:/你的项目" -o "$PWD\codex_last.txt" -
# 最终回答读 codex_last.txt
# ⚠ review_task.md 若含中文：存成 UTF-8 带 BOM，或干脆写英文——Codex 的 shell 工具读 UTF-8 无 BOM 中文文件会按 GBK 解码成乱码（§15.2 读盘层坑）
# JSONL 里 {"type":"thread.started","thread_id":"..."} 给续接（- 是 PROMPT 位，结尾不要再多挂一个 -）：
#   "追问内容" | & $CX exec resume <thread_id> - --skip-git-repo-check --json -c 'sandbox_mode="read-only"' -o last2.txt
#   （resume 不认 --sandbox/-C/--color，沙箱用 -c 覆盖，详见 §15.3）
```

### 2.4 四家最小对照

| 项 | ZCode | Grok | Codex | Claude Code |
|---|---|---|---|
| 工作目录 | `--cwd` | `--cwd` | `-C` / `--cd` | spawnSync 的 `cwd` 选项（无 `--cwd` flag） |
| JSON | 单个对象 `response` | 单个对象 `text`（stderr 有 WARN） | **JSONL** + `-o` 文件 | 单个对象 `result` |
| 只读 | `--mode plan` + `--disallowed-tools "Edit Write Bash"` | `--permission-mode plan` + `--disallowed-tools "Edit,Write,..."`（逗号） | `--sandbox read-only`（无 disallowed-tools） | `--permission-mode plan` + `--disallowed-tools "Edit,Write,Bash"` |
| 续接 | `--resume sess_xxx` | `-r <sessionId>` | `exec resume <thread_id>` | `--resume <session_id>`（同目录实测可用，§16.7） |
| 默认模型 | 见 §1 默认模型怎么查 | 见 §1 默认模型怎么查 | 见 §1 默认模型怎么查 | 见 §1 默认模型怎么查 |
| 无人值守批准 | headless 默认 yolo | **必须** `--always-approve` | 看 `~/.codex/config.toml` 的 `approval_policy`（验证时本机为 `never`）；不要给 `exec` 传 `--ask-for-approval`（clap 直接 exit 2） | headless `-p` 默认自动批准；只读场景显式 `--permission-mode plan` |

### 2.5 Codex 修复类任务（workspace-write）[外部实战验证，2026-09-03]

修 bug / 修测试这类**要改文件**的任务，把 `--sandbox` 换成 `workspace-write`，外层超时给足（实战修 13 个测试文件全程 **~35 分钟**，远超只读 review 的 15–20 分钟）：

```powershell
$CX = (Select-String -Path "$env:USERPROFILE\.codex\config.toml" -Pattern 'CODEX_CLI_PATH').Line.Split("'")[1]
$OutputEncoding = [System.Text.Encoding]::UTF8   # PS 5.1 必加（§15.2）；任务文件建议直接写英文
"Read ./repair_task.md and execute it strictly." |
  & $CX exec --skip-git-repo-check --color never --json --sandbox workspace-write `
    -C "E:/目标仓库" -o "$PWD\codex_last.txt" -
```

任务文件模板——三条防护约束都是实战踩出来的，别省（模板本身用英文书写，规避 §15.2 的读盘乱码坑）：

```markdown
# Task: fix the failing test files under tests/unit

## Constraints (mandatory)
- Encoding: this repo contains non-ASCII (Chinese) text.
  - Never read/write source files via shell text tools (`Get-Content`/`cat` + redirection):
    on this machine their output decodes UTF-8 as GBK and mojibakes it.
  - Use your native file read/edit tools instead; if a shell read is unavoidable,
    pass `-Encoding UTF8` explicitly.
  - After each edit, re-check the changed region; abort and report if you see mojibake
    (e.g. `浠诲姟` where `任务` should be).
- Verification: run tests with the system interpreter exactly as:
  `python -m pytest tests/unit --basetemp .pytest_tmp -p no:cacheprovider`
  (`--basetemp` is required: the sandbox denies the default temp location.)
  - Do NOT create a virtualenv (in-repo or anywhere); do NOT pip-install missing
    packages. If a dependency is missing, mark affected tests "blocked (missing dep)"
    in the report instead of "fixed".
- Do not modify files outside the scope above.

## Deliverable
- Write report to notes/repair_report.md: per-file root cause, change summary,
  tests still failing and why.
```

**调用方必做复核**（两件，缺一不可）：

1. **乱码扫描**：对 diff 全量检查中文是否被写坏。实战靠上面的任务约束 + 事后全量扫描 diff 才确认无乱码——次生风险是 Codex 用 shell 读改含中文注释的源文件，一改就坏。
2. **调用方环境复跑**：Codex 报的「全绿」必须在**你自己的环境**重跑确认。实战中它在沙箱里自建 venv 装了 pytest-mock，于是把「调用方全局环境缺 pytest-mock 导致的 7 个 fixture `'mocker' not found`」误诊为沙箱 harness 问题、标记无需修复；调用方复跑才抓出来。**它沙箱里的验证结果 ≠ 调用方环境**，依赖 fixture / 环境差异的用例尤其如此。

运行期监控与量级 [外部实战]：

- `workspace-write` + stdin 短指令 + `-o last.txt` + 调用方后台运行的组合工作良好：JSONL 可实时监控（数 `item.started` / `item.completed` 事件 + 看最新 command_execution），中途随时 `git status` 审计它动了哪些文件。
- 量级参考：修 13 个测试文件 ~35 分钟 / 数百事件 / exit 0 / 最终回答 ~400 字 / 报告 18.5KB 落在 `-o` 文件。§13 的耗时经验对 Codex 大致适用。
- 若漏写约束，Codex 会在目标仓库里自建 venv（`.test_venv_new/`）和 `tests/.pytest_tmp/` 污染仓库；后者可能被句柄锁死（`rm -rf` 报 Permission denied，重启才能删）。事后要 `git status` 检查并清理这些产物。

### 2.6 Claude Code（只读 review）

Claude Code 与 §2.1–§2.3 同为两跳：任务写文件，命令行只留一句短指令。只读模式走 `--permission-mode plan`（实测可用，~7.3s 回 `READ_ONLY_OK`，exit 0；与 Grok 的 `--permission-mode plan` 同名同语义，详见 §16）。

```powershell
Set-Location "E:/你的项目"   # Claude Code 没有 --cwd；shell 用例先切目录，详见 §16.3 / §16.9
& "$env:USERPROFILE\.local\bin\claude.exe" `
  -p "阅读 ./review_task.md 并严格执行其中的任务" `
  --output-format json `
  --permission-mode plan `
  --disallowed-tools "Edit,Write,Bash"
# stdout 是单个 JSON：.result = 最终回答，.session_id 给 --resume（两次调用须同目录，§16.7）。
# ⚠ 实测（2026-09-03）：--disallowed-tools 单写 "Bash" 禁不住终端（Grok 与 Claude Code 均实测翻车），
#   只读必须靠 --permission-mode plan（§14.4.1 / §16.4）
```

下面是各 CLI 手册。调用方不需要也可以不往下读。

## 3. ZCode：它是什么 / 入口在哪

以下 §3–§13 都是 ZCode。Grok 见 §14，Codex 见 §15，Claude Code 见 §16。

zcode 除了桌面版和全屏 TUI，还支持**无界面单次运行**（headless / print 模式）：给它一段任务文字，它执行完把结果打印到 stdout 后退出。这是给脚本和其他 agent 调用 zcode 的标准方式。

zcode **不在 PATH 里**，它是桌面版自带的 CLI 脚本，实际入口：

```
%LOCALAPPDATA%\Programs\ZCode\resources\glm\zcode.cjs
```

必须用 node 调用。等价写法：

```bash
# Git Bash
node "$LOCALAPPDATA/Programs/ZCode/resources/glm/zcode.cjs" <参数>
```

```bat
:: CMD
node "%LOCALAPPDATA%\Programs\ZCode\resources\glm\zcode.cjs" <参数>
```

```powershell
# PowerShell
node "$env:LOCALAPPDATA\Programs\ZCode\resources\glm\zcode.cjs" <参数>
```

建议在 shell 配置里加别名（Git Bash，加到 `~/.bashrc`）：

```bash
alias zcode='node "$LOCALAPPDATA/Programs/ZCode/resources/glm/zcode.cjs"'
```

## 4. 前置条件：CLI 配置文件（缺了会报 Model config is missing）

独立 CLI **不读桌面版的配置**。桌面版登录的凭据在 `~/.zcode/v2/config.json`，而 CLI 读的是：

```
%USERPROFILE%\.zcode\cli\config.json
```

该文件不存在或没有 provider 时，任何 `--prompt` 调用都会报错：

```
Error: Model config is missing. Create %USERPROFILE%\.zcode\cli\config.json with an explicit model provider before running ZCode.
```

用以下模板检查 / 创建该文件（与官方 `login` 命令写入的格式逐字段一致，此格式从 zcode.cjs 0.16.3 源码中的配置写入函数核对得出）。存在性检查见 §1 通用发现手段。

```json
{
  "provider": {
    "bigmodel": {
      "kind": "anthropic",
      "name": "BigModel Coding Plan",
      "options": {
        "apiKey": "<你的API Key>",
        "apiKeyRequired": true,
        "baseURL": "https://open.bigmodel.cn/api/anthropic"
      },
      "models": {
        "glm-5.1": { "name": "GLM-5.1" },
        "glm-4.7": { "name": "GLM-4.7" },
        "glm-5.3": { "name": "GLM-5.3" }
      }
    }
  },
  "model": {
    "main": "bigmodel/glm-5.3",
    "lite": "bigmodel/glm-4.7"
  }
}
```

说明：

- `apiKey` 可从桌面版配置 `~/.zcode/v2/config.json` 的 `provider["builtin:bigmodel-coding-plan"].options.apiKey` 字段复用（同一账号同一把 key，两端通用）。
- 切换默认模型：改 `model.main`，格式为 `<provider>/<model-id>`，且该 model-id 要出现在 `provider.bigmodel.models` 记录里。已实测 `bigmodel/glm-5.1` 与 `bigmodel/glm-5.3` 均可正常调用。
- **[已实测]** CLI 的 `login` 子命令只支持 OAuth 网页流程（给它传 `bigmodel-coding-plan-api-key <key>` 位置参数会被忽略、直接走 OAuth 并报 `OAuth response is not valid JSON`），所以静默环境下请直接按上面模板写配置文件，不要用 `login`。
- `logout` 子命令会清除共享登录凭据，**会连带影响桌面版登录状态，慎用**。

## 5. 无界面单次运行（核心用法）

两种等价形式 **[已实测]**：

```bash
zcode --prompt "你的任务描述" [--cwd <目录>] [其他选项]
zcode -p "你的任务描述" [--cwd <目录>] [其他选项]     # -p 后跟位置参数
```

最小可运行示例（在任意目录）**[已实测]**：

```bash
node "$LOCALAPPDATA/Programs/ZCode/resources/glm/zcode.cjs" \
  --prompt "只回复两个字：收到" \
  --cwd "$USERPROFILE/.zcode/workspace/default"
# 输出：收到
```

- 不带任何参数直接运行 `zcode` 会打开全屏 TUI（交互模式，不适合脚本调用）。
- headless 模式默认权限模式为 `yolo`（自动批准工具调用），无人值守时注意任务描述的安全性，或配合 `--disallowed-tools` 限制能力。
- 任务文字以 `-` 开头时，需放在 `--` 之后（帮助文档中的提示）。

### 5.1 大文档 review 的推荐姿势（long-review recipe）

**[外部实测]** 审查大文档（几十 KB 以上）时**不要用 `--attach`**，改用 `--cwd` 让 zcode 自己读文件；详细审查要求按 §5.2 写进任务文件，prompt 保持一句短指令：

```bash
node "$LOCALAPPDATA/Programs/ZCode/resources/glm/zcode.cjs" \
  --prompt "阅读 ./review_task.md 并严格执行其中的任务" \
  --cwd "E:/你的仓库" \
  --json \
  --mode plan \
  --disallowed-tools "Edit Write Bash"
```

- `--attach` 大文件是超时重灾区：**[外部实测]** 72KB 附件 + 1.5KB prompt 跑满 8 分钟超时无任何输出；而 `--cwd` 自读约 75KB 上下文 266 秒完成。小附件（几 KB）**[已实测]** 没有问题。
- `--mode plan` 显式只读（**[已实测]** headless 下 plan 模式可正常完成），比默认 `yolo` 安全。
- 运行期间 stdout/stderr 完全没有进度输出，结束才一次性出结果（详见 §7、§13.1），外层超时要给足。

**典型耗时预算**（**[外部实测]** + **[已实测]** 汇总，供预估 wall time）：

| 任务规模 | wall time | 参考 token |
|---|---|---|
| 最小任务（"只回复OK"） | ~7s | 输入 13k，输出 3 |
| 中等 review（~75KB 上下文，产出 8906 字报告） | 266s（~4.4min） | 输入 55k，输出 21.5k |
| 大 prompt + 多个大附件（11KB+27KB+45KB） | **>480s 无输出（超时）** | — |

> 经验值：简单任务 5-15s，中等 review 4-10 分钟；超过 10 分钟大概率是任务方向不对，用 `--resume sess_xxx --prompt "仅列出前 3 条最关键修复"` 收缩任务重试，不要干等。

### 5.2 复杂任务写进文件，prompt 保持短指令（推荐实践）

任务说明一长（多行、编号要求、中文、代码片段），直接塞进 `--prompt` 就是自找麻烦：

- **转义地狱**：引号、换行、反引号、`$` 在 cmd / PowerShell / Git Bash 三套 shell 里转义规则各不相同，跨 shell 复用时几乎必错；
- **长度限制**：Windows 命令行有硬上限（cmd.exe 约 8KB，CreateProcess 约 32KB），复杂 review 要求很容易超；
- **不可复用**：每次调用都要重新拼一遍长字符串，而调用方（上游 agent）生成一个文件要简单可靠得多。

**推荐模式**：详细要求写成任务文件，命令行只留一句短指令，让 zcode 在 `--cwd` 里自己读：

```bash
# 第 1 步：调用方把详细要求写进任务文件（放在 --cwd 目录里）
cat > "E:/你的项目/review_task.md" <<'EOF'
请审查 ./脚本.md 是否合理反映了 ./info/canonical.md 的事实。
读这 2 个文件，输出 markdown 报告：
(1) 已对齐事实 + 行号
(2) 遗漏 / 不准确 + 行号
(3) 官方没说但脚本里有的内容
(4) 评估结论 1-3 句
EOF

# 第 2 步：prompt 只有一句话
node "$LOCALAPPDATA/Programs/ZCode/resources/glm/zcode.cjs" \
  --prompt "阅读 ./review_task.md 并严格执行其中的任务" \
  --cwd "E:/你的项目" \
  --json \
  --mode plan \
  --disallowed-tools "Edit Write Bash"
```

**[已实测]** 两跳模式（短 prompt → 读任务文件 → 按任务文件读目标文件 → 作答）工作正常，退出码 0。

要点：

- 任务文件放 `--cwd` 目录里用相对路径引用，或放任意位置用绝对路径引用均可；
- 任务文件本身就是最好的"需求文档"：可版本化、可复用、上游 agent 改要求只改文件不碰命令行；
- `--attach` 不是这个模式的替代品（大文件超时风险见 §5.1），zcode 在 cwd 里用 Read 工具自读最稳；
- 短 prompt 也让日志和进程列表（`ps` / 任务管理器命令列）保持可读。

## 6. 参数清单（验证快照见文首；版本漂移时按 §12 自检清单复验）

### 6.1 实测可用

| 参数 | 作用 | 实测方式 |
|---|---|---|
| `--prompt <text>` | 单次任务，不开 TUI，结果输出 stdout | 真实运行 |
| `-p <text>` | 同上的位置参数简写 | 真实运行 |
| `--cwd <path>` | 指定工作目录（review 哪个仓库就指哪） | 真实运行 |
| `--json` | 输出机器可读 JSON（结构见 §7） | 真实运行 |
| `--attach <path>` | 附带本地文件给任务，可重复多次。**小附件没问题；大文件（几十 KB+）有超时风险 [外部实测]，大文档改用 `--cwd` 自读（§5.1）** | 真实运行（模型正确读出附件内容） |
| `--disallowed-tools <list>` | 禁用工具，空格或逗号分隔，例 `"Edit Write"` 或 `"Edit,Write"`；支持模式如 `Bash(git *)`；驼峰别名 `--disallowedTools` 同样有效。**headless review 场景强烈建议加 `"Edit Write"`（至少），防止 review 变改代码** | 真实运行 |
| `--resume <sessionId>` | 续接指定会话（`sess_...`），配合 `--prompt` 追问。注意：续接的可能是别的 agent 留下的 session，若其原本跑在 `yolo` 下，续接时建议同步显式 `--mode` + 收窄 `--disallowed-tools` | 真实运行（错误路径：不存在时报 `Session not found`） |
| `--continue`（长格式） | 续接当前目录最近一次会话 | 参数接受性验证 |
| `--mode <mode>` | 权限模式：`build` / `edit` / `plan`（只读） / `yolo`（自动批准一切）。**headless 默认 yolo**；review 场景推荐显式 `--mode plan`，测试命令场景 `--mode build` | 真实运行（`--mode plan` headless 下正常完成） |
| `--target <text>`、`--target-replace` | 设置/替换会话目标 | 参数接受性验证 |
| `--surface <surface>` | headless 呈现面：`terminal` 或 `desktop` | 参数接受性验证 |
| `--locale <locale>` | `en-US` / `zh-CN` / `auto` | 参数接受性验证 |
| `--no-color` / `--verbose` | 关闭 ANSI 颜色 / 输出诊断细节 | 参数接受性验证 |
| `--browser-use <mode>` | 启用 Browser Use 后端（支持 `headless`） | 参数接受性验证 |
| `--browser-executable <path>` | 指定 Chrome/Chromium 可执行文件 | 参数接受性验证 |
| `--force-mcs` / `--no-browser` | Anthropic 投影开关 / login 时只打印 OAuth URL 不开浏览器 | 参数接受性验证 |
| `-h, --help` / `-v, --version` | 帮助 / 版本号（`0.16.5`） | 真实运行 |

### 6.2 帮助里列出、但 0.16.3 / 0.16.5 实测拒绝（传入直接报 Unknown option，勿用）

| 参数 | 现象 |
|---|---|
| `--max-turns <n>` | `Unknown option '--max-turns'` |
| `--allowed-tools <list>` | `Unknown option '--allowed-tools'` |
| `--permission-mode <mode>` | `Unknown option '--permission-mode'`（用 `--mode` 代替） |
| `--settings <path>` | `Unknown option '--settings'` |
| `--allow-main-worktree-yolo` | `Unknown option` |

这类"帮助与实现不一致"的坑在升级版本后可能修复，升级后请按 §12 方法重新检测。

## 7. `--json` 输出格式 [已实测]

```bash
zcode --prompt "1+1等于几？只回答数字" --json
```

```json
{
  "sessionId": "sess_c25fe17b-...",
  "traceId": "c4e58f79-...",
  "turnId": "turn_a26aa33e-...",
  "response": "2",
  "usage": {
    "source": "provider",
    "modelRequestCount": 1,
    "inputTokens": 13067,
    "outputTokens": 3,
    "totalTokens": 13070,
    "cacheReadTokens": 7168,
    "cacheWriteTokens": 0,
    "reasoningTokens": 0,
    "webFetchRequests": 0,
    "webSearchRequests": 0
  },
  "eventCount": 11,
  "projection": {
    "status": "idle",
    "turnCount": 1,
    "totalTokenCount": 13070,
    "contextUsed": 13070,
    "contextWindow": 200000
  }
}
```

上游 agent 程序化取结果：`response` 是最终回答文本；`sessionId` 可存下来用 `--resume` 续聊；`usage` 是 token 计量。

> ⚠ **运行期间没有任何进度输出** [外部实测+已实测确认]：headless 模式下 zcode 不输出 thinking/进度占位文本，子进程的 stdout/stderr 在任务结束前一直是空的（中等任务空 1-3 分钟起步）。程序化调用必须等进程退出才能拿到内容；想观测进度，需在 spawn 端把输出 pipe 到本地日志文件再自行 tail（见 §13.1）。多轮/streaming 进度需求考虑 `app-server`（§8）。

## 8. 子命令 [已实测，除注明外]

| 命令 | 作用 | 实测输出示例 |
|---|---|---|
| `version` | 打印版本 | `0.16.5` |
| `doctor` | 检查运行环境 | `version: 0.16.5 / process: zcode-cli / node: v24.5.0 / platform: win32/x64 ...` |
| `skills list` | 列出本地 skills（验证时本机 85 个，数量随机器变，含路径） | `- agent-browser (user/agents) ...` |
| `plugins list` | 列出插件及启用状态 | `- browser-use@zcode-plugins-official [enabled] ...` |
| `commands list` | 列出自定义斜杠命令 | `No custom commands found.` |
| `login` | OAuth 登录（**只支持网页流程**，见 §4） | 实测：传 key 参数被忽略，走 OAuth |
| `logout` | 清除共享登录凭据（影响桌面版登录，慎用） | 未实测 |
| `tui` | 打开终端 TUI | 未实测 |
| `app-server` | 启动 ZCode Protocol stdio 应用服务器（程序化集成入口，支持流式进度/多轮交互） | 启动行为已实测：起后静默等待 stdin 协议输入（5s 内无任何 stdout/stderr）；协议本身未深入实测。单轮 headless review 用 `--prompt` + `--json` 已够，多轮/streaming 场景再考虑它 |

## 9. 退出码 [已实测]

| 场景 | 退出码（shell 直接看 `$?`） | Node spawnSync 视角 [外部实测] |
|---|---|---|
| 成功（version、成功的 --prompt 运行等） | `0` | `result.status === 0` |
| 失败（未知参数、`--resume` 会话不存在、配置缺失等） | `1` | `result.status === 1` |
| **外层 timeout 强杀（不是 zcode 失败）** | 被信号终止，无正常退出码 | `result.status === null` 且 `result.signal === 'SIGTERM'` |
| Node spawnSync 上层罕见 panic | — | `result.status === null` 且无 signal |

注意：

- 在 shell 里用管道（`zcode ... | head`）取退出码会拿到管道末端的退出码，脚本里请用 `${PIPESTATUS[0]}`（bash）或避免接管道。
- `status === null` + SIGTERM 说明是你自己的超时设置杀掉的进程，此时没有任何 stdout 可拿、也无法 `--resume` 出结果——对策是第一次 spawn 就给足超时（15-20 分钟）并按 §13.2 拆任务。

## 10. 典型场景：让 zcode review 其他 agent 的改动

其他 agent（Claude Code / Cursor / 其他 zcode 会话等）改完代码后，调用 zcode 做只读审查：

```bash
node "$LOCALAPPDATA/Programs/ZCode/resources/glm/zcode.cjs" \
  --prompt "review 当前工作区的改动（先看 git status 和 git diff），逐文件指出正确性问题、边界条件、安全隐患和风格问题，按严重程度排序输出。不要修改任何文件。" \
  --cwd "E:\你的仓库" \
  --disallowed-tools "Edit Write" \
  --json
```

要点：

- `--cwd` 指向被审查的仓库，zcode 会在该目录下自己跑 git 命令读 diff。
- `--disallowed-tools "Edit Write"` 强制只读，防止 review 变成改代码；如需完全禁止执行命令可再加 `Bash`。
- 加 `--json` 便于上游 agent 解析；不加则直接输出纯文本回答。
- **多轮追问**：从 `--json` 输出里记下 `sessionId`，然后：

```bash
zcode --resume sess_xxxx --prompt "第2条问题展开讲讲，给出修复建议的代码" --cwd "E:\你的仓库"
```

- **附带材料**：`--attach report.md --attach screenshot.png` 可把别的 agent 产出的文件直接喂给 zcode；但材料很大时（几十 KB+）改用 `--cwd` + 在 prompt 里指路径让它自读（§5.1）。

### 10.bis 审查多份文档是否一致（大文档模式）[外部实战验证]

验证"某个文档是否如实反映了另一个事实来源"这类任务：详细要求按 §5.2 写进 `review_task.md`，`--cwd` 自读、`--json` 收结果：

```bash
# 先写任务文件（内容见 §5.2 的 review_task.md 示例），再短指令调用：
node "$LOCALAPPDATA/Programs/ZCode/resources/glm/zcode.cjs" \
  --prompt "阅读 ./review_task.md 并严格执行其中的任务" \
  --cwd "E:/你的项目" \
  --json \
  --mode plan \
  --disallowed-tools "Edit Write Bash"

# 从 stdout 的 JSON 拿 sessionId，后续追问：
node "$ZC" --resume sess_xxx --prompt "第 2 节 #5 具体错在第几行？给修复示例" --cwd "E:/你的项目"
```

预期时长 4-10 分钟，输出 8-12k token 是常态（模型要逐行核对），外层超时按 §13.2 给足。

## 11. 已知坑（都是实测踩过的）

1. **`--help` 与实现不一致**：§6.2 列的 5 个参数帮助里有、实际不认。传入未知参数时 zcode 打印一条错误加整页帮助，**错误文本在第一行**，接了管道 `tail` 很容易漏看，误以为只是"打印了帮助"。
2. **zcode 不在 PATH**，必须 node + 完整路径调用（§3 别名）。
3. **CLI 与桌面版配置分离**：桌面版登录不会自动生成 `~/.zcode/cli/config.json`，需按 §4 手工配置。
4. **CLI login 不接受命令行传 key**，只走 OAuth；CI/无人值守环境直接写配置文件。
5. headless 默认 `yolo` 模式会自动批准工具调用，无人值守场景务必用 `--disallowed-tools` 收窄权限（或 `--mode plan` 之类更保守的模式）。
6. **`--attach` 大文件会拖死任务** [外部实测]：72KB 附件 8 分钟超时无输出。大文档一律 `--cwd` 自读（§5.1）。
7. **BigModel API 长任务偶发 hang** [外部实测]：部分大 prompt 会卡超过 10 分钟。对策：拆任务、在 prompt 末尾加"仅回复，不要写完整分析"压缩输出、超时给足但不超过 20 分钟（§13.2 / §13.5）。
8. **Node spawnSync 默认 maxBuffer 约 1MB** [已实测：600KB 通过、2MB 报 ENOBUFS 截断]：大报告场景必须显式调大，否则静默截断（§13.3）。
9. **长 prompt 直接写在命令行上必踩转义坑**：引号/换行/中文在 cmd、PowerShell、Git Bash 三套 shell 里转义规则不同，且 Windows 命令行有长度硬上限（cmd.exe 约 8KB）。复杂要求一律按 §5.2 写任务文件，prompt 只留一句短指令。

## 12. 快速自检清单（供验证 agent 复现）

以下命令在 Git Bash 中执行，全部应得到所述结果：

```bash
ZC="$LOCALAPPDATA/Programs/ZCode/resources/glm/zcode.cjs"

# 1. 版本与运行环境（退出码 0）
node "$ZC" version          # → 0.16.5
node "$ZC" doctor           # → process: zcode-cli, node: v24.5.0 ...

# 2. 配置文件存在且含 bigmodel provider（不含 key 内容的检查）
node -e "const c=require(process.env.USERPROFILE+'/.zcode/cli/config.json');console.log(c.provider.bigmodel?'OK':'MISSING')"
# → OK

# 3. headless 最小运行（真实调用模型，退出码 0）
node "$ZC" --prompt "只回复两个字：收到" --cwd "$USERPROFILE/.zcode/workspace/default"
# → 收到

# 4. 位置参数形式 + 附件
echo "hello from zcode cli test" > /tmp/zcode-skill-test.txt
node "$ZC" -p "附件文件第一行写了什么？只引用原文" --attach /tmp/zcode-skill-test.txt \
  --cwd "$USERPROFILE/.zcode/workspace/default"
# → hello from zcode cli test

# 5. JSON 输出与只读限制
node "$ZC" --prompt "1+1等于几？只回答数字" --json --disallowed-tools "Edit Write Bash" \
  --cwd "$USERPROFILE/.zcode/workspace/default"
# → JSON，其中 "response": "2"

# 6. 未知参数报错与退出码
node "$ZC" --prompt hi --max-turns 5 2>&1 | head -1   # → Unknown option '--max-turns'...
node "$ZC" --prompt hi --max-turns 5 >/dev/null 2>&1; echo $?   # → 1

# 7. 不存在的会话
node "$ZC" --resume sess_nonexistent --prompt hi 2>&1 | head -1
# → Error: Session not found: sess_nonexistent ...

# 8. 子命令
node "$ZC" skills list | head -3        # → Available skills (85) ...
node "$ZC" plugins list | head -3       # → Plugins (8) ...
node "$ZC" commands list                # → No custom commands found.

# 9. §5.2 两跳模式：短 prompt + 任务文件（真实调用模型）
TD="$TEMP/zcode-task-test"; mkdir -p "$TD"
printf '# 任务\n1. 读取同目录下的 data.txt\n2. 回答：该文件第一行写了什么？只引用原文\n' > "$TD/task.md"
echo "THE ANSWER IS FILE-BASED TASK 42" > "$TD/data.txt"
node "$ZC" --prompt "阅读 ./task.md 并严格执行其中的任务" --cwd "$TD" \
  --mode plan --disallowed-tools "Edit Write Bash"
# → 输出包含 THE ANSWER IS FILE-BASED TASK 42
```

> 注意：第 3-5、9 项会真实调用模型、消耗少量 token；第 8 项 skills 数量随本机安装变化。若换机器验证，第 2、3 项依赖 `~/.zcode/cli/config.json` 已按 §4 配置。

## 13. 排错与性能（程序化调用实战）[外部实测为主]

### 13.1 长任务没有进度输出

症状：`--json` 调用后子进程 stdout 前 1-3 分钟完全为空。这是正常行为（stream-to-buffer），不是卡死。
对策：spawn 端把 stdout/stderr pipe 到本地日志文件异步 tail；要真正的流式进度走 `app-server`（§8）。

### 13.2 外层超时被杀后无法续看

症状：spawnSync 超时触发 SIGTERM，子进程被杀，无完整 stdout，也拿不到 sessionId 去 `--resume`。
对策：

- 第一次 spawn 就给 15-20 分钟超时，别用 8-10 分钟默认值赌运气；
- 任务太大时在 prompt 里写"只输出 5 条最关键修复，不要展开"压缩输出；
- 用 `--mode plan`（只读）减少工具调用折返。

### 13.3 输出超过 spawnSync maxBuffer 会被静默截断

Node v24 默认 maxBuffer **约 1MB** [已实测：600KB 正常、2MB 报 `ENOBUFS` 且 stdout 截断为约 1.07MB；外部报告所写"默认 200KB"为旧版本信息，已修正]。
对策：`spawnSync(..., { maxBuffer: 64 * 1024 * 1024 })`，或改用流式读取。

### 13.4 `--attach` 大文件不如 `--cwd` 自读

[外部实测] 72KB 附件 8 分钟超时无输出；同样内容让 zcode 在 `--cwd` 里自己 Read 反而 266 秒完成。结论：附件只用于小文件，大文档一律 `--cwd` + prompt 指路径。

### 13.5 稳定的 spawnSync 调用范例（三状态判断）

```javascript
const { spawnSync } = require("child_process");
const path = require("path");

const ZCODE = path.join(process.env.LOCALAPPDATA, "Programs", "ZCode", "resources", "glm", "zcode.cjs");

const result = spawnSync("node", [
  ZCODE,
  // 详细要求已由调用方写入 E:/你的项目/review_task.md（见 §5.2），prompt 保持短指令
  "--prompt", "阅读 ./review_task.md 并严格执行其中的任务",
  "--cwd", "E:/你的项目",
  "--json",
  "--mode", "plan",                      // 显式只读，不用默认 yolo
  "--disallowed-tools", "Edit Write Bash",
], {
  encoding: "utf8",
  stdio: "pipe",
  maxBuffer: 64 * 1024 * 1024,           // 默认约 1MB，大报告必须调大（§13.3）
  timeout: 20 * 60 * 1000,               // 20 分钟，中等 review 实测 ~4.4min
});

if (result.status === 0) {
  const parsed = JSON.parse(result.stdout);
  console.log(parsed.response);          // 最终回答
  console.log(parsed.sessionId);         // 存下来给 --resume 追问
  console.log(parsed.usage);             // token 计量
} else if (result.status === 1) {
  console.error("zcode error:", result.stderr);
} else if (result.status === null && result.signal === "SIGTERM") {
  console.error("外层超时被杀：拆任务 + 收缩输出后重试，无结果可续看（§13.2）");
}
```

## 14. Grok CLI（xAI）headless 调用 [本节均已实测，2026-08-25]

xAI 官方 Grok CLI 的 headless 模式与 ZCode 形态接近，但有几个跟 README 不一致的细节，**踩过的标出**。当前版本见 §1 通用发现手段 / §14.10 自检。

### 14.1 它是什么 / 入口在哪

- 验证环境：Windows 11 (win32 10.0.26200 x64)，`grok --version` → `grok 1.0.5 (5115b46bc9)`
- 二进制入口（**不是** `~/.grok/grok.exe`，那个目录列表里的项在 `bin/` 子目录里）：

```
%USERPROFILE%\.grok\bin\grok.exe
```

⚠ **路径陷阱** [已实测]：直接在 `%USERPROFILE%\.grok\` 下 `Get-ChildItem` 会显示 `grok.exe` / `agent.exe` 两个 142MB 的"幻影"条目，但 PowerShell 是把 `bin/` 里的内容拍平渲染的，**真实路径就是 `bin\grok.exe`**。

### 14.2 认证检查

探活：

- 是否登录：跑 `grok -p "x"` 看是否走浏览器（未登录会触发）。要稳一点看 `auth.json`：`Test-Path "$env:USERPROFILE\.grok\auth.json"`，存在且 `auth_mode: "oidc"` 即已登录；不存在就是没登录或已过期。
- 过期判定：OIDC token 7 天后过期（已知机制，验证时本机 token 即 7 天内有效）；过期后 `grok -p` 会自动重新走浏览器登录流程，或者用 `grok login` 主动续。
- CI / 静默环境不走 OIDC：设 `XAI_API_KEY=xai-...` 环境变量，**API key 优先于浏览器凭据**。
- 探测成本：`grok -p "x"` 会真实发一次最小调用（验证时约 1s，token 级）；`Test-Path` 一次文件 stat 完全免费。

### 14.3 Headless 单次运行

[已实测] 两种等价形式：

```bash
grok -p "你的任务描述" [--cwd <目录>] [--output-format json] [...]
grok --single "你的任务描述" [...]
```

最小可运行示例 [已实测，~3-10s 出结果]：

```bash
"%USERPROFILE%\.grok\bin\grok.exe" ^
  -p "只回复一个字：好" ^
  --cwd "%TEMP%" ^
  --output-format json
```

输出（一段 WARN 前置 + JSON，详见 §14.5）：

```json
{
  "text": "好",
  "stopReason": "end_turn",
  "sessionId": "01a03938-...",
  "requestId": "8d84a7f0-...",
  "thought": "The user asked me to reply with only one character: 好 ...",
  "usage": {
    "input_tokens": 22074,
    "cache_read_input_tokens": 1408,
    "cache_creation_input_tokens": 0,
    "output_tokens": 54,
    "reasoning_tokens": 49,
    "total_tokens": 23536
  },
  "num_turns": 1,
  "total_cost_usd": 0.00767992,
  "total_cost_usd_ticks": 76799200,
  "modelUsage": {
    "grok-4.6-build": {
      "inputTokens": 22074,
      "outputTokens": 54,
      "cacheReadInputTokens": 1408,
      "cacheCreationInputTokens": 0,
      "modelCalls": 1,
      "costUSD": 0.00767992
    }
  }
}
```

### 14.4 参数清单

#### 14.4.1 实测可用（headless 模式下）

| 参数 | 作用 | 实测 |
|---|---|---|
| `-p, --single <PROMPT>` | 单次任务 prompt（headless 必需） | 真实运行 |
| `--cwd <PATH>` | 工作目录，agent 在里面用工具自读文件 | 真实运行 |
| `--output-format <FMT>` | `plain`（默认）/ `json` / `streaming-json` / `streaming-messages-json` | 真实运行 |
| `--always-approve` | 自动批准工具调用（无人值守脚本必加） | 真实运行 |
| `--permission-mode <MODE>` | `default` / `acceptEdits` / `auto` / `dontAsk` / `bypassPermissions` / `plan` | 真实运行（help 列出 6 值） |
| `--disallowed-tools <list>` | 逗号分隔，移除工具；支持 `Agent(type)` 阻断 subagent | 真实运行（flag 接受 `Edit,Write,Bash` / `run_terminal_cmd`）。⚠ 2026-09-03 实测：只写 `Bash` **禁不住**终端工具（echo 命令照跑）；写 `run_terminal_cmd` 能移除主 shell，但模型自述仍可走兜底 command runner——**只读场景不要单靠工具名黑名单，必须配合 `--permission-mode plan`**（§2.2 即此写法） |
| `--tools <list>` | 逗号分隔 allowlist；同时设了 `--disallowed-tools` 时后者再扣 | 仅帮助文档 |
| `--max-turns <N>` | 限制 agentic turn 数 | 仅帮助文档 |
| `-m, --model <MODEL>` | 模型 ID，例 `grok-4.6-build`；不传走默认 | 仅帮助文档 |
| `--reasoning-effort <LEVEL>` / `--effort` | 推理强度（**实测只接受 `low|medium|high|xhigh`**，见 §14.4.2） | 真实运行 |
| `-r, --resume <ID>` | 按 sessionId 续接；可省略 ID 取当前目录最近 | 真实运行（2026-09-03 实测：暗号续答正确，见 §14.10 第 6 项） |
| `-c, --continue` | 续接当前目录最近一次会话 | 仅帮助文档 |
| `-s, --session-id <ID>` | CI/CD 用的命名 session（不存在则新建） | 仅帮助文档 |
| `--rules <TEXT>` | 追加到系统 prompt | 仅帮助文档 |
| `--system-prompt-override <PROMPT>` | 整段替换系统 prompt | 仅帮助文档 |
| `--prompt-file <PATH>` | 从文件读单次 prompt（替代 `-p`） | 仅帮助文档 |
| `--prompt-json <JSON>` | 直接以 JSON content blocks 形式喂 prompt | 仅帮助文档 |
| `--json-schema <SCHEMA>` | 强约束输出 JSON；隐含 `--output-format json` | 仅帮助文档 |
| `--no-plan` / `--no-subagents` / `--disable-web-search` | 关掉对应能力 | 仅帮助文档 |
| `--verbatim` | 原样发送 prompt（不做 chat 上下文包装） | 仅帮助文档 |
| `-v, --version` / `-h, --help` | 版本 / 帮助 | 真实运行 |

#### 14.4.2 跟官方 README 不一致（实测拒绝）

[已实测] `--reasoning-effort none` / `--reasoning-effort minimal` / `--reasoning-effort max` / `--reasoning-effort deep` **全部报**：

```
Error: --effort/--reasoning-effort: unknown effort level 'xxx'; use one of: xhigh, high, medium, low
```

官方 README 列出 `none, minimal, low, medium, high, xhigh, max, deep`，实测只认 `low|medium|high|xhigh` 四个。**升级版本后请按 §14.10 自检**。

### 14.5 stderr WARN 前置 — 解析 JSON 的核心坑

[已实测] `grok -p ... --output-format json` 的 stderr 上一定有 0~N 行 WARN，例如：

```
2026-08-25T13:59:27.323114Z  WARN skill name does not match expected name ... declared_name=self-learning expected_name=self-learning-skill
2026-08-25T13:59:27.538363Z  WARN hook failed hook_name=global/settings:session_start[0].hooks[0] elapsed_ms=0 hook_failure=hook not executed: required env var(s) not set: ${COUNT}
```

这是 grok 在启动时扫描本地 skills / 跑 session_start hook 触发的，**不属于错误**。但你必须按下面任一方式处理：

| 场景 | 推荐做法 |
|---|---|
| Node spawnSync | `stdout: "pipe"`、`stderr: "pipe"`，**不要 `stdio: 'inherit'`**；`JSON.parse(result.stdout)` 直接解析 stdout，stderr 走自己的日志 |
| Shell `2>&1` 后 grep / parse | ❌ **错** —— WARN 行会污染 stdout 解析；禁止 |
| 只看最终回答 | `grok -p "..." --output-format plain`，plain 模式 stdout 干净，stderr 的 WARN 还在但可以 `2>$null` 吞掉 |

实测 plain 模式 [已实测 ~3s]：

```bash
"%USERPROFILE%\.grok\bin\grok.exe" ^
  -p "only say OK" ^
  --reasoning-effort low ^
  --output-format plain 2>$null
# → OK
```

### 14.6 输出格式对照

| 格式 | stdout 形态 | 适用 |
|---|---|---|
| `plain`（默认） | 纯文本 | 人读 / shell 管道 |
| `json` | **单个** JSON 对象，字段见 §14.3 | 程序化取结果（`text` / `sessionId` / `usage` / `total_cost_usd`） |
| `streaming-json` | NDJSON，每行一个事件：`{type,data,sessionId,...}` | 实时进度 / 流式 UI |
| `streaming-messages-json` | Anthropic Messages API wire format 的 NDJSON | 与 Anthropic 生态工具对接 |

`streaming-json` 事件类型（[仅帮助文档]）：

```json
{"type":"text","data":"Here's"}
{"type":"text","data":" a summary"}
{"type":"thought","data":"Analyzing the directory..."}
{"type":"end","stopReason":"end_turn","sessionId":"abc","requestId":"xyz"}
```

### 14.7 退出码 [已实测]

| 场景 | 退出码 |
|---|---|
| 成功 | `0` |
| 未知 flag / `--reasoning-effort none` / cwd 不存在 / 模型错 | `1` |
| OIDC token 过期未带 API key | `1`（stderr 给 reauth 提示） |
| 外层 timeout 强杀 | 跟随 shell（signal-based），spawnSync 拿 `result.signal` |

### 14.8 与 ZCode headless 的关键差异

| 项 | ZCode | Grok |
|---|---|---|
| 默认模型 | 见 §1 默认模型怎么查 | 见 §1 默认模型怎么查 |
| JSON 字段命名 | camelCase（`inputTokens`） | snake_case（`input_tokens`） |
| 成本字段 | 无 | `total_cost_usd` |
| 续接会话 | `--resume sess_xxx` | `-r <sessionId>` 或 `-r` 续最近 |
| 强制只读模式 | `--mode plan` | `--permission-mode plan` |
| 工具 disable | `--disallowed-tools "Edit Write"`（空格） | `--disallowed-tools "Edit,Write"`（**逗号分隔**） |
| 自读大文件 | `--cwd + Read`（§5.1 推荐） | `--cwd + read_file`（一致；grok 没有 `--attach`） |
| 权限/工具的关系 | 工具 disable = 直接移除 | 同上；外加 `--allow/--deny glob` 是 gate 不删 |
| headless 缺省权限 | `yolo`（自动批准） | `default`（会问）—— **脚本必加 `--always-approve`** |

### 14.9 最小 spawnSync 范例（Node）

```javascript
const { spawnSync } = require("child_process");
const path = require("path");

const GROK = path.join(process.env.USERPROFILE, ".grok", "bin", "grok.exe");

const result = spawnSync(GROK, [
  "-p", "阅读 ./review_task.md 并严格执行其中的任务",  // §5.2 两跳模式，prompt 保持短指令
  "--cwd", "E:/你的项目",
  "--output-format", "json",
  "--always-approve",                                  // headless 默认会问，脚本必加
  "--permission-mode", "plan",                         // 只读
  "--disallowed-tools", "Edit,Write,run_terminal_cmd",
], {
  encoding: "utf8",
  stdio: ["ignore", "pipe", "pipe"],                   // 关键：stderr 单独走，不要 2>&1
  maxBuffer: 64 * 1024 * 1024,
  timeout: 20 * 60 * 1000,
});

if (result.status === 0) {
  const parsed = JSON.parse(result.stdout);            // stdout 是干净的 JSON，§14.5
  console.log(parsed.text);                            // 最终回答（不是 response，命名跟 ZCode 不一样）
  console.log(parsed.sessionId);                       // 存下来给 -r 续接
  console.log(parsed.total_cost_usd);                  // 美元成本，ZCode 没有
} else if (result.status === 1) {
  console.error("grok error:", result.stderr);
} else if (result.signal === "SIGTERM") {
  console.error("外层超时被杀：拆任务 + 收缩输出后重试");
}
```

### 14.10 快速自检清单（验证用）

```bash
# PowerShell
$GRK = "$env:USERPROFILE\.grok\bin\grok.exe"

# 1. 版本
& "$GRK" --version
# → grok 1.0.5 (5115b46bc9)

# 2. 最小 headless（plain）
& "$GRK" -p "only say OK" --reasoning-effort low --output-format plain 2>$null
# → OK

# 3. JSON 输出 + 字段名验证（不要 2>&1，stdout 是干净的 JSON）
& "$GRK" -p "只回复一个字：好" --cwd "$env:TEMP" --output-format json | Select-Object -Last 12
# → 含 "text": "好"、"sessionId": "..."、"total_cost_usd": ...

# 4. cwd 不存在 → 报错 exit 1
& "$GRK" -p "x" --cwd "E:/no_such_dir" --output-format json 2>$null
# exit 1

# 5. --reasoning-effort none → 报错 exit 1（README 不一致）
& "$GRK" -p "x" --reasoning-effort none --output-format json 2>$null
# Error: --effort/--reasoning-effort: unknown effort level 'none'; ...
# exit 1

# 6. 续接 -r（2026-09-03 实测可用）
$J = & "$GRK" -p "记住暗号：冬瓜77号。只回复：记住了" --cwd "$env:TEMP" --output-format json --always-approve 2>$null
$SID = [regex]::Match(($J -join "`n"), '"sessionId":"([^"]+)"').Groups[1].Value
& "$GRK" -r $SID -p "暗号是什么？只回复暗号本身" --always-approve --output-format plain 2>$null
# → 冬瓜77号
```

## 15. Codex CLI（ChatGPT 桌面版）headless 调用 [本节已实测 / 仅帮助文档]

桌面是 Microsoft Store 的 ChatGPT Codex，CLI 已被同步到用户目录，**可以直接 spawn**。默认模型由 `~/.codex/config.toml` 决定（查法见 §1 默认模型怎么查），**不一定是官方 GPT**——本机验证时经 cc-switch 路由为 MiniMax-M3。调用方先看 §1–§2；这里是参数和坑。

### 15.1 入口在哪

| 路径 | 版本 | spawn |
|---|---|---|
| `%LOCALAPPDATA%\OpenAI\Codex\bin\<哈希>\codex.exe`（示例 `994e8469124a0d31`，随桌面升级变；**动态取法见下方代码块，勿抄示例哈希**） | 0.153.0-alpha.5（验证快照） | ✅ [已实测] `--version` / `exec` / `exec resume`（修复类实战见 §2.5） |
| `...\OpenAI\Codex\bin\<旧哈希>\codex.exe`（如 v1.4.1 时的 `8fffe69425752027`、v1.4.0 时的 `f79cfd43cf28a53a`） | 已随桌面升级失效 | ❌ 目录不存在（旧哈希目录会在 bin\ 下残留多个，均勿依赖） |
| `%LOCALAPPDATA%\OpenAI\Codex\bin\codex.exe` | 0.130.0-alpha.5 | 能跑，但是旧副本，不要用 |
| `C:\Program Files\WindowsApps\OpenAI.Codex_26.818.8289.0_x64__2p2nqsd0c76g0\app\resources\codex.exe` | Store 包 26.818.8289.0 自带 | ❌ Access is denied（Store ACL） |

发现当前二进制（桌面升级后哈希会变）：

```powershell
Select-String -Path "$env:USERPROFILE\.codex\config.toml" -Pattern 'CODEX_CLI_PATH'
```

验证时（本机 2026-09-03 实测）`codex login status` → `Logged in using an API key`，`~/.codex/config.toml` 关键字段：`model_provider = "custom"`、`model = "MiniMax-M3"`、`model_catalog_json = "cc-switch-model-catalog.json"`、`base_url = "https://api.minimaxi.com/v1"`、`sandbox_mode = "danger-full-access"`、`approval_policy = "never"`。**升级后请按 §1 通用发现手段 / §15.7 自检清单重新确认**——`approval_policy = "never"` + `sandbox_mode = "danger-full-access"` 的组合意味着 review 类任务不显式 `--sandbox read-only` 就会直接动文件。

### 15.2 Headless 单次运行

> ⚠ **中文编码坑有两层，是两个不同的失败面，对策不同**：
>
> 1. **stdin 层：PS 5.1 管道喂中文变 `???`** [已实测]：PS 5.1 的 `$OutputEncoding` 默认 ASCII，`"中文prompt" | & $CX exec ... -` 到模型端全是问号（模型会直说"我只能看到问号"）；pwsh 7 与 Git Bash 均正常。PS 5.1 里先 `$OutputEncoding = [System.Text.Encoding]::UTF8` 再管道；读 `-o` 输出文件时配 `Get-Content -Encoding UTF8`。
> 2. **读盘层：Codex 的 shell 工具读 UTF-8 无 BOM 中文文件按 GBK 解码成乱码** [外部实战，2026-09-03]：stdin 环节完全正常（它正确理解短指令并去读任务文件），坏在 Codex 自己 spawn 的 pwsh（本机 7.6.5）用 `Get-Content` 读盘：`任务：修复` → `浠诲姟锛氫慨澶`（典型 UTF-8 字节按 GBK 解码）。它带着乱码任务继续执行必然跑偏，只能终止重派。pwsh 文档口径是默认 utf8NoBOM、实测不符（可能受本机 profile / 控制台代码页影响，机制未深挖）。**对策：两跳的任务文件用英文（ASCII）书写，或存成 UTF-8 带 BOM**；修复类任务文件加编码防护约束并让调用方事后扫 diff（模板见 §2.5）。
>    一键复现：`echo '读取 x.md 并原样引用第一行' | & $CX exec --skip-git-repo-check --json --sandbox read-only -C <目录> -`，x.md 为 UTF-8 无 BOM 中文文件，观察 JSONL 里 command_execution 的 aggregated_output 是否乱码。

```powershell
$CX = (Select-String -Path "$env:USERPROFILE\.codex\config.toml" -Pattern 'CODEX_CLI_PATH').Line.Split("'")[1]

# 短 prompt 可以直接当参数；长 prompt / 含空格：走 stdin
"Reply with only the two letters: OK" |
  & $CX exec --ephemeral --skip-git-repo-check --color never --json --sandbox read-only -C "$env:TEMP" -o "$env:TEMP\codex-last.txt" -
# [已实测] exit 0，约 8s，last.txt = OK
```

显式钉死模型（**仅当必须固定模型时才传 `-m`**；默认不传、用 CLI 配置的默认模型，查法见 §1 默认模型怎么查）[已实测 ~4.8s]：

```powershell
"Reply with only this exact line: MiniMax-M3-CLI-OK" |
  & $CX exec --ephemeral --skip-git-repo-check --color never --json --sandbox read-only -m MiniMax-M3 -C "$env:TEMP" -o "$env:TEMP\codex-last.txt" -
# last.txt = MiniMax-M3-CLI-OK
```

`--json` stdout 是 JSONL，不是单个对象 [已实测]：

```json
{"type":"thread.started","thread_id":"01a03947-..."}
{"type":"turn.started"}
{"type":"item.completed","item":{"id":"item_0","type":"agent_message","text":"OK"}}
{"type":"turn.completed","usage":{"input_tokens":23417,"cached_input_tokens":4480,"output_tokens":18,"reasoning_output_tokens":0}}
```

上游取结果：读 `-o` 文件；或扫 `type==item.completed` 且 `item.type==agent_message` 的 `item.text`。`thread_id` 给 `exec resume`。

### 15.3 `exec` 参数（0.149 `exec --help`；0.153.0-alpha.5 实战复用未发现新变化 [外部实测，2026-09-03]）

| 参数 | 作用 | 实测 |
|---|---|---|
| `[PROMPT]` 或 `-`（stdin） | 任务；stdin 与参数同时有则 stdin 追加为 `<stdin>` 块 | 真实运行 |
| `-C, --cd <DIR>` | 工作根 | 真实运行 |
| `--json` | stdout 打 JSONL 事件 | 真实运行 |
| `-o, --output-last-message <FILE>` | 最终回答写文件 | 真实运行 |
| `-s, --sandbox <MODE>` | `read-only` / `workspace-write` / `danger-full-access`。**`-s` 不是 session**；`workspace-write` 修复类任务的坑（venv 漂移 / pytest `--basetemp`）见 §2.5 | 真实运行 |
| `-m, --model <MODEL>` | 覆盖默认模型（值由 §1 默认模型怎么查 / §15.1 决定；当前章节唯一保留的 `-m MiniMax-M3` 例子见上文「显式钉死模型」） | 真实运行 |
| `--ephemeral` | 不落 session 文件 | 真实运行 |
| `--skip-git-repo-check` | 允许非 git 目录 | 真实运行 |
| `--color never` | 关 ANSI | 真实运行 |
| `--ignore-user-config` | 不读 `config.toml`（仍用 `CODEX_HOME` 认证） | 真实运行 → 打 api.openai.com **401** |
| `exec resume [SESSION_ID] [PROMPT]` | 续接；prompt 位置可用 `-` 走 stdin；省略 ID 时配 `--last` 取最近（按 cwd 过滤，跨目录加 `--all`） | 真实运行（前轮暗号续答正确） |
| `review` / `--uncommitted` / `--base` | 非交互 code review | 仅帮助文档 |
| `-i, --image` | 附图 | 仅帮助文档 |
| `--output-schema` | 约束最终 JSON 形状 | 仅帮助文档 |
| `--dangerously-bypass-approvals-and-sandbox` | 跳过确认与沙箱 | 仅帮助文档；review 不要用 |

**不要传：**

| 参数 | 现象 |
|---|---|
| `--ask-for-approval` | `exec` 没有此 flag（只在顶层 TUI）。传入 `unexpected argument`，exit **2** [已实测] |
| `--full-auto` | 0.149 `exec --help` 无此 flag；0.153 实测传入报 `unexpected argument`（2026-09-03，README 曾误用已修） |
| `-s <sessionId>` | `-s` 是 sandbox。续接用 `exec resume` |
| `exec resume` 后跟 `--sandbox` / `-C` / `--color` | [已实测] resume 只认自己的 flag 子集（`--json` `-o` `-m` `--skip-git-repo-check` `-c` `--ephemeral` `--last` `--all` 等），上述三个报 `unexpected argument` exit 2。沙箱用 `-c 'sandbox_mode="read-only"'` 覆盖，工作目录沿用原 session |
| `exec resume <id> - ... -o file -`（结尾多挂 `-`） | [已实测] resume 的位置参数只有 `[SESSION_ID] [PROMPT]`，`-`（stdin）放 PROMPT 位即可；结尾再挂一个报 `unexpected argument '-'`，exit 2 |

### 15.4 退出码 [已实测]

| 场景 | 退出码 |
|---|---|
| 成功的 `exec` / `--version` / `--help` | `0` |
| 401 / 任务失败 | `1` |
| 未知参数（如 `--ask-for-approval`） | `2` |
| 外层 timeout 强杀 | spawnSync 看 `result.signal` |

### 15.5 与 ZCode / Grok 的差异

见 §2.4。额外：Codex 没有 `--attach`、没有 `--disallowed-tools`；大文档同样 `-C` 自读（沿用 §5.1）。PowerShell 长 prompt 必须 stdin，`Start-Process -ArgumentList` 会把 `Reply with only...` 拆成多个 argv。

### 15.6 最小 spawnSync 范例（Node）

```javascript
const { spawnSync } = require("child_process");
const fs = require("fs");
const path = require("path");

// 哈希目录随桌面升级变化，从 config.toml 动态取（发现方法见 §15.1），不要写死哈希
const CX = fs.readFileSync(path.join(process.env.USERPROFILE, ".codex", "config.toml"), "utf8")
  .match(/CODEX_CLI_PATH\s*=\s*'([^']+)'/)[1];
const last = path.join(process.cwd(), "codex_last.txt");

const result = spawnSync(CX, [
  "exec",
  "--skip-git-repo-check",
  "--color", "never",
  "--json",
  "--sandbox", "read-only",
  "-C", "E:/你的项目",
  "-o", last,
  "-",
], {
  encoding: "utf8",
  input: "阅读 ./review_task.md 并严格执行其中的任务",
  stdio: ["pipe", "pipe", "pipe"],
  maxBuffer: 64 * 1024 * 1024,
  timeout: 20 * 60 * 1000,
});

if (result.status === 0) {
  console.log(fs.readFileSync(last, "utf8")); // 最终回答
  const thread = (result.stdout || "").split(/\r?\n/).map((line) => {
    try { return JSON.parse(line); } catch { return null; }
  }).find((ev) => ev && ev.type === "thread.started");
  if (thread) console.log(thread.thread_id);
} else if (result.status === 1) {
  console.error("codex error:", result.stderr);
} else if (result.status === 2) {
  console.error("bad flag:", result.stderr);
} else if (result.signal) {
  console.error("外层超时被杀：拆任务后重试");
}
```

### 15.7 快速自检清单

```powershell
$CX = (Select-String -Path "$env:USERPROFILE\.codex\config.toml" -Pattern 'CODEX_CLI_PATH').Line.Split("'")[1]

# 1. 版本
& $CX --version
# → codex-cli 0.153.0-alpha.5

# 2. 登录（不打印 key）
& $CX login status
# → Logged in using an API key

# 3. 最小 headless（不传 -m：默认模型由 §1 默认模型怎么查 / §15.1 决定）
"Reply with only the two letters: OK" |
  & $CX exec --ephemeral --skip-git-repo-check --color never --json --sandbox read-only -C "$env:TEMP" -o "$env:TEMP\codex-last.txt" -
# exit 0，codex-last.txt = OK

# 4. JSONL 含 thread.started / agent_message / turn.completed

# 5. 错误 flag
& $CX exec --ask-for-approval never "x"
# unexpected argument '--ask-for-approval'，exit 2

# 6. 续接（中文 stdin 先修 PS 5.1 编码；- 是 PROMPT 位，结尾不要再挂 -）
$OutputEncoding = [System.Text.Encoding]::UTF8
$J = "记住暗号：冬瓜55号。只回复：记住了" |
  & $CX exec --skip-git-repo-check --json --sandbox read-only -C "$env:TEMP" -
$TID = [regex]::Match(($J | Select-Object -First 1), '"thread_id":"([^"]+)"').Groups[1].Value
"暗号是什么？只回复暗号本身" |
  & $CX exec resume $TID - --skip-git-repo-check --json -c 'sandbox_mode="read-only"' -o "$env:TEMP\codex-resume.txt"
# exit 0，Get-Content -Encoding UTF8 "$env:TEMP\codex-resume.txt" → 冬瓜55号
```

### 15.8 可选路径（本机不必走）

- **ChatGPT 官方模型**：需要 `codex login`（ChatGPT OAuth / `--device-auth`），不要 `--ignore-user-config` + 现有 `sk-cp-` key 打 Platform。本次未改登录。
- **npm `@openai/codex`**：会装第二份 CLI，和桌面升级通道打架。验证时本机未装，也不推荐为了 headless 再装。

## 16. Claude Code（Anthropic 官方 native installer）headless 调用 [本节已实测 / 仅帮助文档]

### 16.1 入口与版本

| 路径 | spawn |
|---|---|
| `%USERPROFILE%\.local\bin\claude.exe` | ✅ [已实测] `--version` / `-p` / `--permission-mode plan` / `--resume`。`where.exe claude` 命中此路径。 |
| `claude-code-router` / `claude-code-ui`（npm 目录） | ❌ 与 Claude Code **不同**的工具，不要混用 |

`--version` 输出（实测）：

```
2.1.204 (Claude Code)
```

### 16.2 鉴权 / 配置（不打印任何 key）

探活：检查 `~/.claude/settings.json`（OAuth token 路径）是否存在；存在即已配置，无需 `claude auth` 也无需 `ANTHROPIC_API_KEY` 环境变量。验证时（本机 2026-09-03 实测）`modelUsage` 字段显示 `glm-5.2`（key 是 `glm-5.2`，值含 `inputTokens/outputTokens/cacheReadInputTokens/cacheCreationInputTokens/webSearchRequests/costUSD/contextWindow:200000/maxOutputTokens:32000`），说明请求被路由到非 Anthropic 官方模型——**路由机制（router / env / settings）未深究**；调用方用 JSON 里 `modelUsage` 的 key 当「实际跑的是哪个模型」的判据即可（每次跑都看一次，因为路由可能换）。

### 16.3 Headless 单次运行（最小示例）[已实测]

```powershell
# 切工作目录（Claude Code 没有 -C / --cwd flag）
Set-Location "$env:TEMP"
"Reply with only the two letters: OK" |
  & "$env:USERPROFILE\.local\bin\claude.exe" -p - --output-format json
# 或 stdin 不便时直接：
& "$env:USERPROFILE\.local\bin\claude.exe" -p "Reply with only the two letters: OK" --output-format json
# → exit 0，约 7s，stdout = 单个 JSON 对象
# spawnSync 用例见 §16.9（cwd: "..." 选项）
```

⚠ **Claude Code 没有 `--cwd` flag**；工作目录走 spawnSync 的 `cwd` 选项（shell 调用时通过 `Set-Location` 或 `Push-Location` 切）。详见 §16.9。

### 16.4 参数清单

| 参数 | 作用 | 实测 |
|---|---|---|
| `-p, --print` | 单次任务（headless 必需；非交互默认进 TUI） | 已实测 |
| `--output-format <FMT>` | `text`（默认）/ `json`（单个 result 对象）/ `stream-json`（NDJSON） | 已实测（json）；stream-json 仅帮助文档 |
| `--input-format <FMT>` | `text`（默认）/ `stream-json`（与 `--output-format stream-json` 配对做双向流） | 仅帮助文档 |
| `--model <MODEL>` | 覆盖默认模型（值由 §1 默认模型怎么查 / §16.2 决定；详见 §16.2 的快照说明） | 仅帮助文档（路由机制下未实测自定义 model） |
| `--effort <LEVEL>` | 推理强度，实测可接受 `low, medium, high, xhigh, max`（与 Grok 同名同语义） | 仅帮助文档 |
| `--permission-mode <MODE>` | `acceptEdits` / `auto` / `bypassPermissions` / `manual` / `dontAsk` / `plan` | 已实测（plan ~7.3s 回 READ_ONLY_OK，exit 0） |
| `--allowed-tools` / `--allowedTools <list>` | 逗号或空格分隔的允许工具列表 | 仅帮助文档 |
| `--disallowed-tools` / `--disallowedTools <list>` | 逗号或空格分隔的禁用工具列表 | 已实测（flag 接受 `Edit,Write,Bash`）。⚠ 2026-09-03 实测：`bypassPermissions` 下禁 `"Edit,Write,Bash"` **未拦住**终端命令（echo 照跑，`permission_denials` 为空）——只读必须靠 `--permission-mode plan`，别单靠工具黑名单（与 Grok §14.4.1 同教训） |
| `--append-system-prompt <TEXT>` | 追加到默认系统 prompt | 仅帮助文档 |
| `--system-prompt <TEXT>` | 整段替换默认系统 prompt | 仅帮助文档 |
| `--session-id <UUID>` | 指定本次 session ID | 仅帮助文档 |
| `--resume [value]` / `-r` | 按 session ID 续接；省略 value 弹交互选择器；**两次调用须同 cwd**（跨目录报 `No conversation found`，§16.7） | 已实测（同目录 exit 0，暗号续答正确） |
| `--continue` / `-c` | 续接当前目录最近一次会话（无 ID 走 cwd 推断） | 仅帮助文档 |
| `--from-pr [value]` | 按 PR 编号 / URL 续接 | 仅帮助文档 |
| `--fork-session` | 续接时创建新 session ID（与 `--resume` / `--continue` 配对） | 仅帮助文档 |
| `--no-session-persistence` | 不落 session 文件，session 不能续接（仅 `-p` 模式） | 仅帮助文档 |
| `--add-dir <dirs...>` | 追加允许工具访问的目录 | 仅帮助文档 |
| `--mcp-config <configs...>` | 加载 MCP servers JSON 文件或字符串 | 仅帮助文档 |
| `--settings <file-or-json>` | 临时加载 settings JSON | 仅帮助文档 |
| `--setting-sources <srcs...>` | 逗号分隔允许加载的 setting 来源（user/project/local） | 仅帮助文档 |
| `--bare` | 极简模式：跳过 hooks / LSP / plugin / auto-memory / CLAUDE.md 自动发现；强约束 anthropic key 路径 | 仅帮助文档 |
| `--safe-mode` | 关闭所有自定义（CLAUDE.md / skills / plugins / hooks / MCP / 主题等），保留内置工具与权限 | 仅帮助文档 |
| `--debug [filter]` / `--debug-file <path>` | debug 日志 | 仅帮助文档 |
| `--max-budget-usd <AMOUNT>` | 单次 API 美元预算上限（仅 `-p`） | 仅帮助文档 |
| `--json-schema <SCHEMA>` | 强约束输出 JSON 形状 | 仅帮助文档 |
| `-v, --version` / `-h, --help` | 版本 / 帮助 | 已实测 |
| `install [target]` | 装/重装 native build（`stable` / `latest` / 特定版本） | 仅帮助文档 |
| `auth` / `setup-token` / `update` / `upgrade` / `doctor` / `mcp` / `plugin(s)` | 子命令 | 仅帮助文档 |

### 16.5 JSON 输出形状 [已实测]

`--output-format json` 时 stdout 是**单个 JSON 对象**（不是 JSONL）。关键字段（实测一次 `-p "Reply with only the two letters: OK"` 的截取，会话 ID 已脱敏）：

```json
{
  "type": "result",
  "subtype": "success",
  "is_error": false,
  "duration_ms": 7185,
  "num_turns": 1,
  "result": "OK",
  "stop_reason": "end_turn",
  "session_id": "bb03a644-...",
  "total_cost_usd": 0.143612,
  "usage": {
    "input_tokens": 28502,
    "cache_creation_input_tokens": 0,
    "cache_read_input_tokens": 704,
    "output_tokens": 30,
    "server_tool_use": { "web_search_requests": 0, "web_fetch_requests": 0 }
  },
  "modelUsage": {
    "glm-5.2": {
      "inputTokens": 28502, "outputTokens": 30,
      "cacheReadInputTokens": 704, "cacheCreationInputTokens": 0,
      "webSearchRequests": 0, "costUSD": 0.143612,
      "contextWindow": 200000, "maxOutputTokens": 32000
    }
  },
  "permission_denials": [],
  "uuid": "..."
}
```

字段说明：

- `result` — 最终回答（取代 ZCode 的 `response` / Grok 的 `text`，命名与 Codex 不同）
- `session_id` — 本次调用所属 session 的 UUID（详见 §16.7 的 resume 机制坑）
- `usage` — snake_case 字段（与 Grok 同风格，与 ZCode camelCase 不同）
- `modelUsage` — 实际生效模型 + 成本明细；key 由 §1 默认模型怎么查 / §16.2 的探活决定（验证时本机为 `glm-5.2`）
- `stop_reason` — `end_turn` / `max_tokens` / `tool_use` 等
- `is_error` / `permission_denials` — 出错判定（与 `subtype == "error"` 配对）

**程序化取结果**：`JSON.parse(result.stdout).result` 取最终回答；`.session_id` 留作续接 key（两次调用须同 cwd，§16.7）。

### 16.6 退出码 [已实测]

| 场景 | 退出码 |
|---|---|
| 成功 | `0` |
| 未知 flag / `--resume <id>` 跨目录找不到会话 | `1`（实测：两次调用 cwd 不同时返 `No conversation found with session ID: ...`，同目录则 exit 0，§16.7） |
| 外层 timeout 强杀 | spawnSync 看 `result.signal` |

### 16.7 续接 session（实测：同目录可用，跨目录找不到）[已实测]

`--resume <id>` 直接用第一次调用 JSON 里的 `session_id`，**但两次调用必须在同一个工作目录**——Claude Code 的会话按目录存储（`~/.claude/projects/<cwd 转写>/`），跨目录 resume 直接报错：

```powershell
# 两次调用都在同一目录（例如都在被 review 的仓库根）：
$J1 = & "$env:USERPROFILE\.local\bin\claude.exe" -p "记住暗号：香蕉66号。只回复：记住了" --output-format json --permission-mode plan
$SID = ($J1 | ConvertFrom-Json).session_id
& "$env:USERPROFILE\.local\bin\claude.exe" --resume $SID -p "暗号是什么？只回复暗号本身" --permission-mode plan --output-format json
# [已实测 2026-09-03] 同目录：exit 0，result = 香蕉66号
# [已实测 2026-09-03] 跨目录（两次调用 cwd 不同）：exit 1，
#   stderr: No conversation found with session ID: ...（不是 ID 错，是目录不匹配）
```

要点：

- 生成会话与续接必须同 cwd；spawnSync 场景两次都要带同样的 `cwd` 选项（§16.9）。
- 首轮实测曾把跨目录报错误判为「JSON `session_id` 不能用于 resume」——调用方别被表象带偏，先核对两次调用的目录。
- 兜底（**仅帮助文档**）：`-c` 续接当前目录最近一次会话；`-r` 不带 ID 弹交互选择器，headless 不友好。

### 16.8 与其它三家的差异

| 项 | ZCode | Grok | Codex | Claude Code |
|---|---|---|---|---|
| 默认模型 | 见 §1 默认模型怎么查 | 见 §1 默认模型怎么查 | 见 §1 默认模型怎么查 | 见 §1 默认模型怎么查 |
| JSON 字段命名 | camelCase（`inputTokens`） | snake_case（`input_tokens`） | JSONL + 多种事件 | snake_case（`usage`），camelCase（`modelUsage.*`）混用 |
| 成本字段 | 无 | `total_cost_usd` | 无 | `total_cost_usd` + `modelUsage.{model}.costUSD` |
| 续接 key | `--resume sess_xxx` | `-r <sessionId>` | `exec resume <thread_id>` | `--resume <session_id>`（同目录实测可用，§16.7） |
| 只读模式 | `--mode plan` | `--permission-mode plan` | `--sandbox read-only` | `--permission-mode plan` |
| 工具 disable | 空格分隔 | 逗号分隔 | 无（用 sandbox 替代） | 逗号或空格分隔（flag 接受；实测单禁 `Bash` 拦不住终端，§16.4） |
| 自读大文件 | `--cwd + Read` | `--cwd + read_file` | `-C + shell` | spawnSync `cwd` 选项（无 `--cwd` flag） |
| headless 缺省权限 | `yolo` | `default`（必加 `--always-approve`） | 看 `~/.codex/config.toml` 的 `approval_policy`（验证时本机为 `never`） | `-p` 默认自动批准；只读场景显式 `--permission-mode plan` |

### 16.9 最小 spawnSync 范例（Node）

```javascript
const { spawnSync } = require("child_process");
const path = require("path");

const CLAUDE = path.join(process.env.USERPROFILE, ".local", "bin", "claude.exe");

const result = spawnSync(CLAUDE, [
  "-p", "阅读 ./review_task.md 并严格执行其中的任务",
  "--output-format", "json",
  "--permission-mode", "plan",                  // 只读
  "--disallowed-tools", "Edit,Write,Bash",      // 配合 plan 锁只读
], {
  cwd: "E:/你的项目",                            // Claude Code 没有 --cwd；走 spawnSync 选项
  encoding: "utf8",
  stdio: ["ignore", "pipe", "pipe"],            // stdout 干净 JSON；stderr 走自己的日志
  maxBuffer: 64 * 1024 * 1024,
  timeout: 20 * 60 * 1000,
});

if (result.status === 0) {
  const parsed = JSON.parse(result.stdout);     // 单个 JSON 对象，§16.5
  console.log(parsed.result);                   // 最终回答
  console.log(parsed.session_id);               // 续接 key（两次调用须同 cwd，§16.7）
  console.log(parsed.total_cost_usd);           // 美元成本
  console.log(Object.keys(parsed.modelUsage));  // 实际生效模型；key 由 §1 默认模型怎么查 / §16.2 决定（验证时本机 = ["glm-5.2"]）
} else if (result.status === 1) {
  console.error("claude error:", result.stderr);
} else if (result.signal === "SIGTERM") {
  console.error("外层超时被杀：拆任务 + 收缩输出后重试");
}
```

### 16.10 快速自检清单

```powershell
$CL = "$env:USERPROFILE\.local\bin\claude.exe"

# 1. 版本
& $CL --version
# → 2.1.204 (Claude Code)

# 2. where.exe 命中
where.exe claude
# → %USERPROFILE%\.local\bin\claude.exe（示意；实际输出为本机用户目录下的真实路径）

# 3. 最小 headless（plain）
& $CL -p "only say OK" --output-format plain
# → OK

# 4. JSON 输出 + 字段名验证
& $CL -p "只回复一个字：好" --output-format json | Select-Object -Last 12
# → 含 "result": "好"、"session_id": "..."、"total_cost_usd": ...、"modelUsage": { "glm-5.2": {...} }

# 5. 只读 plan 模式（实测 ~7.3s，回 READ_ONLY_OK）
& $CL -p "Reply with only: READ_ONLY_OK" --permission-mode plan --output-format json | Select-Object -Last 5
# exit 0

# 6. 续接（两次调用同目录；跨目录会 No conversation found，§16.7）
$J = & $CL -p "记住暗号：香蕉66号。只回复：记住了" --output-format json --permission-mode plan
$SID = ($J | ConvertFrom-Json).session_id
& $CL --resume $SID -p "暗号是什么？只回复暗号本身" --permission-mode plan --output-format json | Select-Object -Last 3
# → result = 香蕉66号（exit 0；实测 2026-09-03）
```

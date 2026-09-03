# headless-cli-agents

Multi-agent CLI (ZCode / Grok / Codex / Claude Code) headless usage guide.

## 这是什么

中文技能 (skill), 给脚本或其他 agent 用的调用手册, 讲解如何以无界面 (headless) 方式调用 ZCode / Grok / Codex / Claude Code 四家 CLI 做代码审查等任务. 覆盖命令行模板、参数坑、JSON 输出格式、排错、跨 agent 协同 ("两跳" 模式) 等实战内容.

完整内容见 [SKILL.md](./SKILL.md).

## 前置条件

- **ZCode CLI**: 由 ZCode 桌面版自带
- **Grok CLI**: 由 xAI 官方提供
- **Codex CLI**: 由 Codex 桌面版同步到用户目录, 路径见 `~/.codex/config.toml` 的 `CODEX_CLI_PATH`
- **Claude Code**: 由 Anthropic 官方 native installer 提供, 路径 `%USERPROFILE%\.local\bin\claude.exe`

具体版本适配与获取方式见 SKILL.md §1.

## 五分钟上手

最小调用示例 (Codex 只读调用, PowerShell):

```powershell
# codex.exe 路径从 config.toml 动态取 (桌面升级后哈希目录会变, 见 SKILL.md §15.1)
$CX = (Select-String -Path "$env:USERPROFILE\.codex\config.toml" -Pattern 'CODEX_CLI_PATH').Line.Split("'")[1]
"Reply with only the two letters: OK" |
  & $CX exec --skip-git-repo-check --color never --json --sandbox read-only -o "$env:TEMP\codex-last.txt" -
Get-Content "$env:TEMP\codex-last.txt"   # → OK
```

完整命令模板 (四家对照 / 只读 review / 修复类任务) 见 SKILL.md §2.

**能力探测 (capability probing)**: 调用方在 spawn 之前先按 SKILL.md §1.1 的子句 probe 本机实际有哪些 CLI 可用, 只使用存在的. 本技能不引导安装 — 安装路径随厂商 / 平台 / 桌面版差异极大, 已超出 headless 调用手册的职责.

## 仓库结构

- `SKILL.md` -- 技能主体
- `LICENSE` -- MIT License
- `README.md` -- 本文件
- `.gitignore` -- 本地工作区排除规则

验证环境: Windows 11 (win32 10.0.26200 x64), Node v24.5.0, Git Bash / PowerShell. 其它环境结论未验证.

## License

MIT -- see [LICENSE](./LICENSE).

# headless-cli-agents

Multi-agent CLI (ZCode / Grok / Codex) headless usage guide.

## 这是什么

中文技能 (skill), 给脚本或其他 agent 用的调用手册, 讲解如何以无界面 (headless) 方式调用 ZCode / Grok / Codex 三家 CLI 做代码审查等任务. 覆盖命令行模板、参数坑、JSON 输出格式、排错、跨 agent 协同 ("两跳" 模式) 等实战内容.

完整内容见 [SKILL.md](./SKILL.md).

## 前置条件

- **ZCode CLI**: 由 ZCode 桌面版自带
- **Grok CLI**: 由 xAI 官方提供
- **Codex CLI**: 由 Codex 桌面版同步到用户目录, 路径见 `~/.codex/config.toml` 的 `CODEX_CLI_PATH`

具体版本适配与获取方式见 SKILL.md §1.

## 五分钟上手

最小调用示例 (Codex 修复类任务):

```bash
"$CODEX_CLI_PATH" exec --full-auto --sandbox workspace-write "你的任务描述"
```

完整命令模板与多家 CLI 对比见 SKILL.md §2.

## 仓库结构

- `SKILL.md` -- 技能主体
- `LICENSE` -- MIT License
- `README.md` -- 本文件
- `.gitignore` -- 本地工作区排除规则

验证环境: Windows 10 (win32 10.0.26200 x64), Node v24.5.0, Git Bash / PowerShell. 其它环境结论未验证.

## License

MIT -- see [LICENSE](./LICENSE).

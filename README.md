# Zsh Completion for Claude Code

为 [Claude Code](https://claude.ai/code) CLI 提供 zsh 补全支持。

## 安装

```zsh
# 克隆到 fpath 目录
git clone https://github.com/heathcliff-hu/zsh-completion-claude.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-completion-claude

# 或在 .zshrc 中添加
fpath=(/path/to/zsh-completion-claude $fpath)
autoload -U _claude
compdef _claude claude
```

## 补全覆盖

### 主命令 (13 个)

`agents` `auth` `auto-mode` `doctor` `install` `mcp` `plugin` `plugins` `project` `setup-token` `ultrareview` `update` `upgrade`

### 全局选项 (~60 个)

`--model` `--resume` `--print` `--continue` `--debug` `--verbose` `--worktree` `--permission-mode` `--mcp-config` `--system-prompt` `--agent` 等

### 子命令详细补全

| 命令 | 补全内容 |
| ------ | ---------- |
| `claude --model` | 从 settings 文件解析模型别名，显示来源文件 |
| `claude --resume` | 列出 `~/.claude/projects/<slug>/` 下的会话，显示 displayName |
| `claude mcp` | 9 个子命令 + 服务器名动态补全（1h 缓存） |
| `claude auth` | login / logout / status 子命令及选项 |
| `claude plugin` | 16 个子命令 + 已安装插件名/市场名动态补全 |
| `claude plugin marketplace` | add / list / remove / update 子命令 |

### 动态补全

- **模型别名** — 按优先级读取 `.claude/settings.local.json` > `.claude/settings.json` > `~/.claude/settings.json`，解析 `ANTHROPIC_DEFAULT_{SONNET,OPUS,HAIKU}_MODEL` 等 env 变量
- **会话名** — 遍历 `~/.claude/projects/<slug>/*.jsonl`，提取 `displayName`
- **MCP 服务器** — 执行 `claude mcp list`，结果缓存 1 小时
- **已安装插件** — 执行 `claude plugin list`
- **已配置市场** — 执行 `claude plugin marketplace list`

## 开发

```zsh
# 语法检查
zsh -n _claude

# 部署（清除缓存 + 编译 + 重载 shell）
rm -f ~/.zcompdump* _claude.zwc && zcompile _claude && exec zsh
```

> `.zwc` 或 `.zcompdump` 任一残留都会导致旧版生效。

## 文件结构

```shell
_claude          # 单一补全文件（601 行，30 个函数）
CLAUDE.md        # Claude Code 项目上下文
README.md        # 本文件
```

### 函数命名

```shell
_claude                  # 主入口，全局选项 + 子命令分发
_claude_cmd              # 顶层子命令列表
_claude_<group>_cmds     # 二级子命令列表
_claude_<group>          # 二级子命令分发
_claude_<group>_<sub>    # 叶子命令补全（选项 + 位置参数）
_claude_<name>s          # 动态候选生成（如 _claude_models）
_claude_<name>           # 候选生成辅助（如 _claude_resume）
```

## 许可

MIT

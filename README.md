# Zsh Completion for Claude Code

为 [Claude Code](https://claude.ai/code) CLI 提供 zsh 补全支持。

## 安装

```shell
# 克隆到 fpath 目录
git clone https://github.com/heathcliff-hu/zsh-completion-claude.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-completion-claude

# 并在 .zshrc 中启用
plugins=(... zsh-completion-claude)

# 或在 .zshrc 中添加
fpath=(/path/to/zsh-completion-claude $fpath)
autoload -U _claude
compdef _claude claude

# 如果需要缓存清理命令，需在 .zshrc 中添加以下内容
claude() {
  if [[ "$1" == "mcp" && "$2" == "cache-clear" ]]; then
    local cache_file=${ZSH_CACHE_DIR:-$HOME/.cache/zsh}/zsh_claude_mcp_servers
    rm -f -- "$cache_file"
    print "Cleared Claude MCP completion cache: $cache_file"
    return 0
  fi
  command claude "$@"
}
```

## 补全覆盖

### 主命令 (21 个)

`agents` `attach` `auth` `auto-mode` `daemon` `doctor` `install` `kill` `logs` `mcp` `plugin` `plugins` `project` `remote-control` `respawn` `rm` `setup-token` `stop` `ultrareview` `update` `upgrade`

### 全局选项 (~80 个)

`--model` `--resume` `--print` `--continue` `--debug` `--verbose` `--worktree` `--permission-mode` `--mcp-config` `--system-prompt` `--agent` 等

### 子命令详细补全

| 命令                        | 补全内容                                                     |
| --------------------------- | ------------------------------------------------------------ |
| `claude --model`            | 从 settings 文件解析模型别名，显示来源文件                   |
| `claude --resume`           | 列出 `~/.claude/projects/<slug>/` 下的会话，显示 displayName |
| `claude agents`             | 完整选项（--model、--effort、--permission-mode、--add-dir 等）|
| `claude attach/kill/stop/rm/logs` | 后台会话 ID 补全（含名称、状态）                       |
| `claude auth`               | login / logout / status 子命令及选项                         |
| `claude auto-mode`          | config / critique / defaults / help 子命令                   |
| `claude daemon`             | logs/run/status/stop/uninstall 子命令 + 配置文件/日志路径补全 |
| `claude install`            | stable / latest / 自定义版本号                               |
| `claude mcp`                | 9 个子命令 + 服务器名动态补全（1h 缓存）                     |
| `claude plugin`             | 16 个子命令 + 已安装插件名/市场名动态补全                    |
| `claude plugin marketplace` | add / list / remove / update 子命令                          |
| `claude project`            | purge 子命令（--all/--dry-run/--interactive）                |
| `claude remote-control`     | --name / --remote-control-session-name-prefix 选项            |
| `claude respawn`            | --all / 会话 ID 互斥补全                                     |
| `claude ultrareview`        | --json / --timeout 选项                                      |

### 动态补全

- **模型别名** — 按优先级读取 `.claude/settings.local.json` > `.claude/settings.json` > `~/.claude/settings.json`，解析 `ANTHROPIC_DEFAULT_{SONNET,OPUS,HAIKU}_MODEL` 等 env 变量
- **交互会话** — 遍历 `~/.claude/projects/<slug>/*.jsonl`，提取 `displayName`/`customTitle`
- **后台会话** — 遍历 `~/.claude/jobs/*/state.json`，提取名称与状态，按 mtime 降序取最近 20
- **MCP 服务器** — 执行 `claude mcp list`，结果缓存 1 小时
- **已安装插件** — 执行 `claude plugin list`
- **已配置市场** — 执行 `claude plugin marketplace list`

### 缓存清理

`claude mcp cache-clear` 用于清除 MCP 服务器列表补全缓存。由于补全在 zsh 进程中执行，无法直接删除文件，需要 shell 包装拦截命令：

```shell
# 添加到 ~/.zshrc
claude() {
  if [[ "$1" == "mcp" && "$2" == "cache-clear" ]]; then
    local cache_file=${ZSH_CACHE_DIR:-$HOME/.cache/zsh}/zsh_claude_mcp_servers
    rm -f -- "$cache_file"
    print "Cleared Claude MCP completion cache: $cache_file"
    return 0
  fi
  command claude "$@"
}
```

> `$ZSH_CACHE_DIR` 默认为 `~/.cache/zsh`（oh-my-zsh 自动创建），未使用 oh-my-zsh 时可能不存在，需要先 `mkdir -p`。

## 开发

```shell
# 语法检查
zsh -n _claude

# 部署（清除缓存 + 编译 + 重载 shell）
rm -f ~/.zcompdump* _claude.zwc; zcompile _claude && exec zsh
```

> `.zwc` 或 `.zcompdump` 任一残留都会导致旧版生效。

## 文件结构

```shell
_claude                          # 单一补全文件（~930 行，50 个函数）
CLAUDE.md                        # Claude Code 项目上下文
README.md                        # 本文件
.claude/commands/update-completions.md  # 更新提示词
.claude/rules/zsh-completion.md  # Zsh Completion 编写指南
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

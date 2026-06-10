# Zsh Completion for Claude Code

## 开发循环

编辑 `_claude` 后必须：

```shell
zsh -n _claude                       # 语法检查
rm -f ~/.zcompdump* _claude.zwc; zcompile _claude && exec zsh   # 部署
```

`.zwc` 或 `.zcompdump` 任一残留都会导致旧版生效。

## Zsh 陷阱

- `$var:string` 裸写会被 zsh 解析为 `${var:flag}` 而非字符串拼接，必须写 `${var}:string`
- `_describe` 只能在补全函数上下文内调用，独立测试需要用 `zsh -fc 'source file; ...'`
- `_describe -V` — 默认按 key 字母排序，加 `-V` 保持数组传入顺序不变
- `_arguments` 中 `(-x --long){-x,--long}'[描述]'` 共享描述即可自动合并显示，无需额外处理
- 子命令别名（如 plugin/plugins）共用相同描述时 `_describe` 自动分组
- `_arguments` 规格中 ACTION 不要内联 `_describe "multi word" arr`——空格导致 zsh 把后续词当独立补全词泄漏，必须抽成独立辅助函数
- `_arguments` message 中的 `:` 必须转义为 `\:`，否则被解析为 spec 分隔符导致 `parse error near ')'`
- **_arguments ACTION 函数不可包装中间层** — `_arguments '1: :_wrapper'` 中 `_wrapper` 直接调 `_describe`，不能委托给另一函数：`_wrapper() { _helper 'tag' items }` 会导致补全失效。补全逻辑必须直接在 ACTION 函数中内联
- **_arguments 可选选项参数 `::` + 固定候选列表 → 补全泄漏** — zsh 无法判断下一词是选项值还是位置参数，同时显示两者（如 `--prompt-suggestions` 的 `::value:(true false)` 会混入子命令）。改用 `:`（必填）——用户不按 Tab 仍可直接 Enter 跳过该值。含 `=` 的选项（如 `--tmux=`）因值必须跟 `=`，`::` 不会泄漏
- **_arguments `1:` 位置参数守卫** — ACTION 函数开头加 `(( CURRENT == 2 )) || return 1`，阻止选项后补全时泄漏子命令候选（`claude --model x <Tab>` 不再显示 agents/auth 等）
- **文件补全选择准则** — 日常用 `_files`；需精细路径控制（如 `_alternative` 内过滤）用 `_path_files -g`；需严格过滤无匹配不回退时手动 `items=( *.json(N) ); compadd -a items`
- **_files -g 无匹配时回退** — `_files -g "*.json(-.)"` 找不到匹配会回退显示所有文件，泄漏非目标文件。严格过滤场景用手动 `compadd` 替代
- **_arguments -C 子命令选项泄漏** — 当子命令（如 `agents`）直接用 `_arguments` 而非 `_arguments -C` + `->state` 时，`_arguments -C` 的 `*:: :->subargs` 回调中调用子命令函数会看到父级的全部选项。修复：在父函数 `_arguments -C` **之前**拦截，用 `(( CURRENT > 2 ))` + `shift words` + `(( CURRENT-- ))` 重排上下文，让子命令函数只看到自身选项。典型模式：

  ```shell
  if (( CURRENT > 2 )); then
    case "$words[2]" in
      agents)
        curcontext="${curcontext%:*:*}:claude-agents:"
        (( CURRENT-- ))
        shift words
        _claude_agents "$@"
        return
      ;;
    esac
  fi
  ```

## 跨平台兼容

- `stat -f '%m'` 是 BSD 专属，Linux 用 `stat -c '%Y'`；跨平台方案用 zsh 内置 `zstat -F %s +mtime`
- `date -r $ts` 是 BSD 专属，Linux 用 `date -d @$ts`；跨平台方案用 zsh 内置 `strftime`

## jq 查询 settings 文件

```shell
jq -r '.model // empty' settings.json                                # 顶层字段
jq -r '.env | to_entries[] | select(.key | test("MODEL")) | .value'  # 模型相关环境变量
```

## ~/.claude.json 数据

- `.projects` — 已注册项目的路径列表，用 `jq -r '.projects | keys[] // empty' ~/.claude.json` 提取

## 模型优先级

`.claude/settings.local.json` > `.claude/settings.json` > `~/.claude/settings.json`

env 变量映射：

- `ANTHROPIC_MODEL` — 默认模型
- `ANTHROPIC_DEFAULT_{SONNET,OPUS,HAIKU}_MODEL` — 别名解析
- `ANTHROPIC_DEFAULT_{SONNET,OPUS}_MODEL_NAME` — 别名解析备用

## 子命令补全模式

用 `_arguments -C` + `->state` 实现子命令分发，参考 `_claude_mcp`：

```shell
_cmd() {
  local curcontext="$curcontext" state line
  typeset -A opt_args
  _arguments -C \
    '-h[display help]' '--help[display help]' \
    '1: :_cmd_subcmds' '*:: :->subargs' && return
  case "$state" in
    subargs)
      case "$line[1]" in
        sub1) _cmd_sub1 "$@" && return ;;
        *)    _default && return ;;
      esac
      ;;
  esac
}
```

## 会话文件

- `~/.claude/projects/<slug>/` — 会话 `.jsonl` 存放目录，slug = `${PWD//\//-}` 再 `${slug//./-}`
- 若 slug 目录不存在，遍历 `~/.claude/projects/*/sessions-index.json` 用 `jq -r '.originalPath'` 匹配 `$PWD` 回退
- 文件名即 session ID（UUID），`jq -r '.displayName // empty'` 提取显示名

## 后台会话补全

会话数据来自 `$CLAUDE_CONFIG_DIR/jobs/`（默认 `~/.claude/jobs/`），每个会话一个子目录，内含 `state.json`。

- 目录名即 session ID，`state.json` 中 `name`/`displayName`/`title`/`sessionName`/`intent`/`daemonShort` 按优先级取首个非空作为显示名
- `state`/`status`/`lifecycle` 同理取首个非空作为状态
- `attach`/`kill`/`stop`/`rm`/`logs`/`respawn` 共用此函数

## 动态补全候选

从 CLI 子命令输出提取候选，用 `${(f)"$(...)"}` 按行分割：

```shell
_helper() {
  local -a items
  items=(${(f)"$(claude cmd list 2>/dev/null | grep '❯' | sed 's/.*❯ //')"})
  [[ ${#items} -eq 0 ]] && return 1
  _describe 'tag' items
}
```

按 mtime 降序排列用 `sort -rn` + `cut -d' ' -f2-`（比 zsh glob `om`/`Om` 更可靠）：

```shell
items=(${(f)"$(
  for f in $dir/*.jsonl(N); do
    print -r -- "$(stat -f '%m' "$f") $f"
  done | sort -rn | cut -d' ' -f2-
)"})
_describe -V 'tag' items
```

## Plugin 子命令通用选项

- `-s`/`--scope` — `(user project local)` 或含 `managed`（update）
- `-y`/`--yes` — 跳过确认提示
- `--json` — JSON 输出

## 动态列表缓存

耗时 CLI 命令结果缓存到 `$ZSH_CACHE_DIR`，TTL 用 epoch 存首行：

```shell
_helper() {
  local -a items
  local cache_file=$ZSH_CACHE_DIR/zsh_claude_xxx
  if [[ -f $cache_file ]] && (( $(date +%s) - $(head -1 $cache_file) < 3600 )); then
    items=(${(f)"$(tail -n +2 $cache_file)"})
  else
    items=(${(f)"$(claude xxx list 2>/dev/null)"})
    [[ ${#items} -gt 0 ]] && { date +%s; print -l -- $items } > $cache_file
  fi
  [[ ${#items} -eq 0 ]] && return 1
  _describe 'tag' items
}
```

## 嵌套子命令

每层子命令用 `_arguments -C` + `->state` 分发，三层嵌套（如 `plugin marketplace <sub>`）每层独立，参考 `_claude_plugin_marketplace` 实现。

## _arguments 候选模式选择

- `(a b)` — 固定列表，仅允许列出的值
- `_describe` → 辅助函数 — 提供候选建议但允许自由输入（如 install target 选 stable/latest 外可输任意版本号）

## _numbers 用法

```shell
'--timeout[specify timeout]: :_numbers -u minutes -d 30 timeout'  # 带单位和默认值
'::timeout (min) [30]:_numbers'                                   # 简写：message 中标注默认值
```

## 可重复选项 `*` 前缀

- `--help` 输出中 `...`（如 `<directories...>`）或 `(repeatable)` 标注的选项需加 `*--flag`
- 例：`'*--add-dir[additional directories]:directory:_directories'`

## 逗号分隔补全

用 `_values -s ,` 在独立辅助函数中实现：

```shell
_claude_tool_names() {
  _values -s , 'tool' \
    'Bash' 'Read' 'Write' 'Edit' \
    'Glob' 'Grep' 'WebSearch' 'WebFetch' \
    'TaskCreate' 'TaskUpdate' 'TaskGet' 'TaskList' 'TaskOutput' 'TaskStop' \
    'NotebookEdit' 'Agent' 'AskUserQuestion' \
    'EnterPlanMode' 'EnterWorktree' 'ExitPlanMode' 'ExitWorktree' 'Skill' \
    'CronCreate' 'CronDelete' 'CronList' \
    'ScheduleWakeup' 'Workflow' \
    'mcp__*'
}
```

## 补全对齐检查

以 `claude <cmd> --help` 实际输出为准，逐子命令对比补全代码：

- 选项缺失 → 补充
- 参数必填/可选与 `[]`/`<>` 标注不一致 → 修正 `1:`/`::`
- 候选列表不完整 → 对齐 --help 描述

## 更新补全脚本

`.claude/commands/update-completions.md` — 可复用提示词，自动对比 CLI help 输出与系统工具列表，
覆盖命令/选项缺失、工具名称遗漏、可重复选项 `*` 前缀，完成后更新 `_claude` 头部 `# Version:` 行。

## 架构与风格

- `_claude` 为单文件自包含架构（~930 行，50 个函数），无外部依赖，直接放入 `$fpath` 即可
- 补全函数命名：`_claude` → `_claude_<group>` → `_claude_<group>_<sub>` 三级对应 CLI 命令层级
- 代码块语言标识统一用 `shell`，不用 `zsh`

## 子命令补全覆盖

| 命令                                            | 补全深度 | 备注                                                   |
| ----------------------------------------------- | -------- | ------------------------------------------------------ |
| `agents`                                        | 完整     | 含 `--model`/`--effort`/`--permission-mode` 等完整选项 |
| `attach` / `kill` / `stop` / `rm` / `logs`      | 基础     | 会话 ID 补全 (含名称/状态)                             |
| `auth` (login/logout/status)                    | 完整     | login 含 `--console`/`--sso` 等                        |
| `auto-mode` (config/critique/defaults)          | 完整     | critique 含 `--model`                                  |
| `daemon` (logs/run/status/stop/uninstall)       | 完整     | run 含 `--json-path`/`--log-file`                      |
| `doctor` / `setup-token` / `update` / `upgrade` | 基础     | 仅 `-h`/`--help`                                       |
| `install`                                       | 完整     | target: stable/latest/version                          |
| `mcp` (add/get/remove/list/serve/...)           | 完整     | add 含 transport/scope/headers；server 名缓存 1h TTL   |
| `plugin/plugins`                                | 完整     | 三层嵌套 (marketplace add/list/remove/update)          |
| `project` (purge)                               | 完整     | purge 含 `--all`/`--dry-run`/路径补全                  |
| `remote-control`                                | 完整     | 含 `--name`/前缀选项                                   |
| `respawn`                                       | 完整     | 互斥 `--all` / session ID                              |
| `ultrareview`                                   | 完整     | 含 `--json`/`--timeout`                                |

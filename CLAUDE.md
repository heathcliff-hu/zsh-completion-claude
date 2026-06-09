# Zsh Completion for Claude Code

## 开发循环

编辑 `_claude` 后必须：

```zsh
zsh -n _claude                       # 语法检查
rm -f ~/.zcompdump* _claude.zwc && zcompile _claude && exec zsh   # 部署
```

`.zwc` 或 `.zcompdump` 任一残留都会导致旧版生效。

## Zsh 陷阱

- `$var:string` 裸写会被 zsh 解析为 `${var:flag}` 而非字符串拼接，必须写 `${var}:string`
- `_describe` 只能在补全函数上下文内调用，独立测试需要用 `zsh -fc 'source file; ...'`
- `_arguments` 中 `(-x --long){-x,--long}'[描述]'` 共享描述即可自动合并显示，无需额外处理
- 子命令别名（如 plugin/plugins）共用相同描述时 `_describe` 自动分组
- `_arguments` 规格中 ACTION 不要内联 `_describe "multi word" arr`——空格导致 zsh 把后续词当独立补全词泄漏，必须抽成独立辅助函数
- `_arguments` message 中的 `:` 必须转义为 `\:`，否则被解析为 spec 分隔符导致 `parse error near ')'`
- **_arguments ACTION 函数不可包装中间层** — `_arguments '1: :_wrapper'` 中 `_wrapper` 直接调 `_describe`，不能委托给另一函数：`_wrapper() { _helper 'tag' items }` 会导致补全失效。补全逻辑必须直接在 ACTION 函数中内联

## jq 查询 settings 文件

```bash
jq -r '.model // empty' settings.json                                # 顶层字段
jq -r '.env | to_entries[] | select(.key | test("MODEL")) | .value'  # 模型相关环境变量
```

## 模型优先级

`.claude/settings.local.json` > `.claude/settings.json` > `~/.claude/settings.json`

env 变量映射：
- `ANTHROPIC_MODEL` — 默认模型
- `ANTHROPIC_DEFAULT_{SONNET,OPUS,HAIKU}_MODEL` — 别名解析
- `ANTHROPIC_DEFAULT_{SONNET,OPUS}_MODEL_NAME` — 别名解析备用

## 子命令补全模式

用 `_arguments -C` + `->state` 实现子命令分发，参考 `_claude_mcp`：

```zsh
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
      esac
      ;;
  esac
}
```

## 会话文件

- `~/.claude/projects/<slug>/` — 会话 `.jsonl` 存放目录，slug = `${PWD//\//-}`
- 文件名即 session ID（UUID），`jq -r '.displayName // empty'` 提取显示名

## 动态补全候选

从 CLI 子命令输出提取候选，用 `${(f)"$(...)"}` 按行分割：

```zsh
_helper() {
  local -a items
  items=(${(f)"$(claude cmd list 2>/dev/null | grep '❯' | sed 's/.*❯ //')"})
  [[ ${#items} -eq 0 ]] && return 1
  _describe 'tag' items
}
```

## Plugin 子命令通用选项

- `-s`/`--scope` — `(user project local)` 或含 `managed`（update）
- `-y`/`--yes` — 跳过确认提示
- `--json` — JSON 输出

## 动态列表缓存

耗时 CLI 命令结果缓存到 `$ZSH_CACHE_DIR`，TTL 用 epoch 存首行：

```zsh
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

每层子命令用 `_arguments -C` + `->state` 分发，三层嵌套（如 `plugin marketplace <sub>`）每层独立。

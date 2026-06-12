# Zsh Completion 编写指南

参考：[zsh-completions-howto.org](https://github.com/zsh-users/zsh-completions/blob/master/zsh-completions-howto.org)

## 基础

### 注册补全函数

补全文件以 `#compdef <command>` 开头，放在 `$fpath` 目录中，文件名为 `_<command>`。

```bash
#compdef foobar
```

也可用 `compdef` 直接注册：

```bash
compdef _function foobar                    # 单命令
compdef _function foobar goocar hoodar      # 多命令共用
compdef '_function arg1 arg2' foobar        # 带参数
```

### 通用 GNU 命令

```bash
compdef _gnu_generic foobar                 # 自动从 --help 解析选项
```

### 复制补全

```bash
compdef cmd1=cmd2                           # cmd1 使用 cmd2 的补全
```

## 核心工具函数

| 函数 | 用途 |
|---|---|
| `_describe` | 简单补全，词+描述列表 |
| `_alternative` | 多种类型候选混用，支持 ACTION |
| `_arguments` | Unix 风格选项+参数补全，最常用 |
| `_regex_arguments` | 复杂命令行的正则匹配补全 |
| `_regex_words` | 简化 `_regex_arguments` 规格生成 |
| `_values` | 任意关键字及其参数补全 |
| `_multi_parts` | 分隔符分隔的多段补全（如路径） |
| `_sep_parts` | 不同分隔符的多段补全 |

## 内置辅助函数

| 函数 | 用途 |
|---|---|
| `_files` | 文件路径补全 |
| `_path_files` | 路径文件补全，更底层/灵活 |
| `_directories` | 目录补全 |
| `_dir_list` | 目录列表补全 |
| `_users` / `_groups` | 用户名/组名补全 |
| `_net_interfaces` | 网卡名称补全 |
| `_parameters` | Shell 变量名补全 |
| `_options` | Shell 选项名补全 |
| `_message` | 无补全时显示帮助信息 |
| `_guard` | 在 ACTION 中检查当前词是否符合条件 |
| `_sequence` | 包装其他补全函数，补全分隔符分隔的列表 |
| `_numbers` | 数字范围补全 |
| `_email_addresses` | 邮箱地址补全 |
| `_urls` | URL 补全 |
| `_message` | 显示提示信息 |
| `_nothing` | 明确不提供补全 |
| `_default` | 默认补全行为 |
| `_arrays` | 数组参数补全 |

### 缓存相关

| 函数 | 用途 |
|---|---|
| `_cache_invalid` | 判断缓存是否需要重建 |
| `_retrieve_cache` | 从缓存文件取补全数据 |
| `_store_cache` | 将补全数据存入缓存文件 |

## ACTION 形式

用于 `_arguments`、`_alternative`、`_regex_arguments`、`_values` 等函数的规格中：

| 形式 | 含义 |
|---|---|
| `( )` | 必填参数，无候选生成 |
| `(ITEM1 ITEM2)` | 候选列表 |
| `((ITEM1\:'DESC1' ITEM2\:'DESC2'))` | 带描述的候选列表（注意引号不冲突） |
| `->STRING` | 设置 `$state` 为 STRING，后续用 case 分支处理 |
| `FUNCTION` | 调用函数生成候选，如 `_files` |
| `{EVAL-STRING}` | 执行 shell 代码生成候选，如 `{_values -s , letter a b c}` |
| `=ACTION` | 插入哑词但不改变补全位置 |

> `->STRING` 不可用于 `_regex_arguments` 和 `_alternative`。

## _describe — 简单补全

```bash
#compdef cmd

local -a subcmds
subcmds=('c:description for c command' 'd:description for d command')
_describe 'command' subcmds
```

多组列表用 `--` 分隔：
```bash
local -a subcmds topics
subcmds=('c:description for c' 'd:description for d')
topics=('e:description for e' 'f:description for f')
_describe 'command' subcmds -- topics
```

## _alternative — 混合候选

规格格式：`'TAG:DESCRIPTION:ACTION'`

```bash
_alternative \
  'args:custom arg:((a\:"desc a" b\:"desc b" c\:"desc c"))' \
  'files:filename:_files -/'
```

变量展开用双引号：
```bash
_alternative \
  "dirs:user directory:($userdirs)" \
  "pids:process ID:($(ps -A o pid=))"
```

嵌套其他工具函数（注意前置空格）：
```bash
_alternative \
  'options:comma-separated opt: _values -s , letter a b c'
```

## _arguments — Unix 风格选项+参数

### 选项规格

```bash
'-OPT[DESCRIPTION]'                         # 布尔选项
'-OPT[DESCRIPTION]:MESSAGE:ACTION'          # 带参数的选项
'--long[DESCRIPTION]'                       # 长选项
```

### 位置参数规格

```bash
'N:MESSAGE:ACTION'                          # 第 N 个参数
'::MESSAGE:ACTION'                          # 可选参数（双冒号）
':MESSAGE:ACTION'                           # 下一个参数
```

### 完整示例

```bash
_arguments \
  '-s[sort output]' \
  '-f[input file]:filename:_files' \
  '1:first arg:_net_interfaces' \
  '::optional arg:_files' \
  ':next arg:(a b c)'
```

### 用 ->STRING 分支

```bash
_arguments '-m[music file]:filename:->files' '-f[flags]:flag:->flags'

case "$state" in
    files)
        local -a music_files
        music_files=( Music/**/*.{mp3,wav,flac,ogg} )
        _multi_parts / music_files
        ;;
    flags)
        _values -s , 'flags' a b c d e
        ;;
esac
```

## _regex_arguments — 正则匹配补全

用于复杂命令行，多关键字+可变参数序列。

先创建函数再调用：
```bash
_regex_arguments _cmd OTHER_ARGS...
_cmd "$@"
```

序列用 `|` 分隔，分组用 `\(` `\)`。

### PATTERN 要求

- 用 `$'...\0'` 包裹模式（`\0` 是内部词分隔符）
- 变量展开：`"$somevar"$'\0'`

**特殊字符**：

| 字符 | 含义 |
|---|---|
| `*` | 通配任意数量字符 |
| `?` | 通配单个字符 |
| `#` | 前一字符零或多次（同正则 `*`） |
| `##` | 前一字符一或多次（同正则 `+`） |

### _regex_words 简化

```bash
local -a firstword secondword

_regex_words commands 'The first word' \
  'foo:do foo' 'man:yeah man' 'chu:at chu'
firstword=("$reply[@]")

_regex_words word 'The second word' \
  'woo:tang clan' 'hoo:not me'
secondword=("$reply[@]")

_regex_arguments _cmd /$'[^\0]##\0'/ "${firstword[@]}" "${secondword[@]}"
_cmd "$@"
```

## _values / _sep_parts / _multi_parts

```bash
# 空格分隔的 mp3 文件列表
_values 'mp3 files' ~/*.mp3

# 逗号分隔的会话 ID 列表
_values -s , 'session id' "${(uonzf)$(ps -A o sid=)}"

# 分段补全：foo@news:woo 模式
_sep_parts '(foo bar)' @ '(news ftp)' : '(woo laa)'

# 逐段补全 MAC 地址
_multi_parts : '(00:11:22:33:44:55 00:23:34:45:56:67)'
```

## compadd — 底层直接添加候选

```bash
compadd foo bar blah                                      # 基本
compadd -X 'Some completions' foo bar blah                # 带说明
compadd -P what_ foo bar blah                             # 自动加前缀
compadd -S _todo foo bar blah                             # 自动加后缀
compadd -q -S _todo foo bar blah                          # 输空格自动移除后缀
compadd -a wordsarray                                      # 从数组添加
```

## 调试

| 操作 | 命令 / 快捷键 |
|---|---|
| 显示补全上下文 | `Ctrl+x h` |
| 显示更多调试信息 | `Alt+2 Ctrl+x h` |
| 追踪补全执行 | `Ctrl+x ?` |
| 重载补全函数 | `unfunction _func && autoload -U _func` |
| 清除缓存 | `rm -f ~/.zcompdump*; exec zsh` |
| 语法检查 | `zsh -n <file>.zsh` |

## 常见陷阱

1. 文件开头必须有 `#compdef` 行
2. `_arguments` 规格注意引号类型：需要变量展开用双引号，否则用单引号；描述内部的引号不能与外层冲突
3. `_arguments` / `_alternative` 规格中 `:` 的数量和位置要正确
4. `_regex_arguments` 必须有匹配命令本身的初始 PATTERN
5. `_regex_arguments` 的 PATTERN 必须以 `$'\0'` 结尾
6. `_arguments` 的 ACTION 不要内联 `_describe "multi word" arr`——空格导致 zsh 把 `word`、`arr` 当独立补全词泄漏。必须抽成独立辅助函数再引用
7. `_arguments -C` 的 `->state` 回调里 `$words[1]` 永远是命令名；子命令匹配用 `$line[1]`（非选项参数数组的第一个元素）

## 官方风格约定

以下摘录自 [Zsh Completion Style Guide](https://github.com/zsh-users/zsh/blob/master/Etc/completion-style-guide)，作为本项目补全脚本的书写规范。

### 代码风格

- **缩进**：用 2 空格缩进，4 空格续行。`_arguments` / `_values` 规格例外（行较长，少缩进更可读）
- **行长**：尽量不超过 79 字符。`_arguments` / `_values` 规格例外
- **禁止非标准语法**：不要用 `for x in $y; myfunc $x`、`if { [[ ... ]] } { ... }`、`foreach` 等非 POSIX 写法
- **if/while**：`then` / `do` 放在条件行末（分号+空格分隔），放不下才换行顶格写

```bash
# 正确
if (( $+commands[cmd] )); then
  ...
fi

while _next_label tag expl 'desc'; do
  compadd "$expl[@]" - foo bar
done
```

### 描述（Description）规范

- **不用句号结尾**，**不用首字母大写**（专有缩写除外）
- **祈使语气**：`"recurse subdirectories"` 而非 `"recurses subdirectories"`
- **单位放括号里**：`'--timeout[specify connection timeout]:timeout (ms)'`
- **默认值放方括号里**：`'--timeout[specify connection timeout]:timeout (ms) [5000]'`
- 也可用 `_numbers`：`'--timeout[specify connection timeout]: :_numbers -u ms -d 5000 timeout'`
- **组描述用单数**（即使列出多项）。tag、类型函数、state 名用复数
- 从 man/--help 抄描述时，去掉对当前上下文无意义的词（如 `(this one)`），把 `NAME` 之类占位符改成 `specified <thing>`
- 相同含义的匹配用相同描述，以便 compsys 分组。可用 brace expansion：
  ```bash
  '(--context -C)'{--context=,-C-}'[specify lines of context]:lines'
  ```

### 何时省略组描述

- helper/type 函数已自带描述时省略：
  ```bash
  # 省略 'file'——_files 自带
  '--import=[import specified file]: :_files'
  ```
- `->state` 的回调中描述无效，省略以减少噪音：
  ```bash
  # 不要这样
  '--config=[use specified config file]:config file:->config-files'
  # 应该这样
  '--config=[use specified config file]: :->config-files'
  ```

### 描述透传

允许调用方传参覆盖默认描述：

```bash
_wanted directories expl directory _files -/ "$@" -
# "$@" 透传上层描述；"-" 告知 _wanted 选项插入位置
```

### Context / Tag / Style

- `curcontext` 是层级上下文名。`_arguments -C` 会修改它，所以必须 `local curcontext="$curcontext"`
- `_wanted` = 测试 tag 是否被请求 + `_all_labels`（处理 tag alias 循环）：
  ```bash
  _wanted names expl 'name' compadd - alice bob
  ```
- 多类型匹配用 `_tags` + `_requested` 循环，或直接用 `_alternative`：
  ```bash
  _alternative \
      'friends:friend:(alice bob)' \
      'users:: _users' \
      'hosts:: _hosts'
  ```
- 多 `->state` 可能同时命中同一词时，用循环处理 `$state` 和 `$context` 数组
- tag 名用简短复数。优先复用已有 tag 名，方便用户跨上下文统一定制样式
- **始终使用描述**。写了 `compadd` 却没有 `"$expl[@]"`（或等效写法）就是 bug
- 辅助函数可以把 `"$@"` 放在默认描述后面，让调用方更具体的描述覆盖默认值
- `_description` 接受 `-V`/`-J`（可选 `1`/`2` 前缀），不要往 `compadd` 里手动塞 `-1V` 等：
  ```bash
  _description -1V tag expl '...'
  compadd "$expl[@]" -
  ```

### 返回值

- 添加了匹配返回 0，没添加返回非 0
- 需要精确判断时，保存并比较 `$compstate[nmatches]`：
  ```bash
  local nm=$compstate[nmatches]
  # ... 添加匹配 ...
  [[ nm -ne compstate[nmatches] ]]
  ```
- 辅助函数应把 `$@` 原样透传给 `compadd`

### 缓存

- 生成匹配耗时较长时，用全局变量缓存，变量名前缀 `_cache_`，用 `typeset -g` 声明
- 列表特别大或生成特别慢，用 `_store_cache` / `_retrieve_cache` / `_cache_invalid` 做持久化
- 缓存时注意 style 对不同 context 的影响

### 其他要点

1. 有合理的 match spec 就给 `compadd` 传
2. 用 `_arguments` / `_values` 处理选项和参数——不要手动解析
3. 用 `_users`、`_groups`、`_hosts` 等标准 helper，不用 ad hoc 实现
4. 带公共前缀的匹配（如 `-`、`--`、`+`）要*带前缀*添加；然后用 `prefix-needed` / `prefix-hidden` 样式控制
5. 命令及其子命令的补全尽量放一个文件；大命令用状态机模式（参考 `_rpm`、`_pbm`）
6. 多模式命令用 `_call_function` 支持用户覆盖：
   ```bash
   _call_function ret _command_$subcommand && return ret
   ```
7. `$words` 数组元素来自用户输入的原始文本（含引号和转义）
8. `$opt_args` 中多值选项用冒号分隔，反斜杠和冒号会被转义
9. `_call_program` 内部用 `eval`，传参用 `${(q-)var}` 防止注入
10. 命令支持 `--help` 时用 `compdef _gnu_generic cmd` 快速生成草稿：
    ```bash
    print -r -- ${(F)${(@qqq)_args_cache_cmd}} > _cmd
    ```

## 技巧

- 子命令后只有一个选项时，zsh 会自动补全它。要显示描述，加空选项：`((opt1\:"desc" \:))`
- 所有 `_arguments` 规格支持互斥选项、重复选项、`+` 开头的选项等，详见 [官方文档](https://zsh.sourceforge.net/Doc/Release/Completion-System.html)

## 外部资源

- [Zsh Completion System 官方文档](https://zsh.sourceforge.net/Doc/Release/Completion-System.html)
- [Zsh Completion Style Guide](https://github.com/zsh-users/zsh/blob/master/Etc/completion-style-guide)

## 本项目约定

- 补全文件以 `#compdef <command>` 开头，命名为 `_<command>`
- 放插件目录或 `src/` 子目录，确保 `fpath` 包含该目录
- 修改 `_command` 后必须：`rm -f ~/.zcompdump*; exec zsh`
- `.zcompdump` 残留会导致旧版生效

# Zsh Completion 本项目约定

通用写法参考（`_arguments`/`_describe`/`_alternative`/`_regex_arguments`/`_values`/`compadd` 用法、ACTION 形式、官方风格约定）已迁移至技能 `zsh-completion-reference`（`.claude/skills/zsh-completion-reference/SKILL.md`），编写具体补全时按需加载。本文件只保留项目特定陷阱与约定。

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

## 本项目约定

- 补全文件以 `#compdef <command>` 开头，命名为 `_<command>`
- 放插件目录或 `src/` 子目录，确保 `fpath` 包含该目录
- 修改 `_command` 后必须：`rm -f ~/.zcompdump*; exec zsh`
- `.zcompdump` 残留会导致旧版生效

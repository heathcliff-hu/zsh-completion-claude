# 检查 _claude 补全文件是否与当前 Claude Code 版本一致

自动对比 CLI 命令/选项、系统工具列表与 `_claude` 补全文件，发现缺失项并更新，最后同步版本号。

---

## 一、命令与选项检查

### 1. 版本号

```shell
claude --version
```

记录版本号，最后写入 `_claude` 头部 `# Version:` 行。

### 2. 主命令与主选项

```shell
claude --help
```

对比 `_claude_cmd` 子命令列表与主函数 `opts`：

- 缺失命令 → 补充 `_claude_cmd` + 主函数分发 case + 对应补全函数
- 缺失选项 → 补充
- `...`/`(repeatable)` 标注但缺 `*` 前缀 → 修复

### 3. 子命令 `--help`

逐一运行并对比对应函数（命令列表以 help 输出为准，下表为参考）：

| 命令 | 对应补全函数 |
| --- | --- |
| `agents` | `_claude_agents` |
| `auth` | `_claude_auth` + `_claude_auth_cmds` |
| `auto-mode` | `_claude_auto_mode` + `_claude_auto_mode_cmds` |
| `daemon` | `_claude_daemon` + `_claude_daemon_cmds` |
| `doctor` | `_claude_help_opts` |
| `gateway` | `_claude_gateway` |
| `import` | `_claude_import` |
| `install` | `_claude_install` |
| `mcp` | `_claude_mcp` + `_claude_mcp_cmds` |
| `plugin` | `_claude_plugin` + `_claude_plugin_cmds` |
| `project` | `_claude_project` + `_claude_project_cmds` |
| `setup-token` | `_claude_help_opts` |
| `ultrareview` | `_claude_ultrareview` |
| `update`/`upgrade` | `_claude_help_opts` |

### 4. 深层子命令 `--help`

对比子命令内部选项与参数标注（`1:` vs `::`）：

- `claude mcp add/remove/login/add-json/serve --help`
- `claude auth login/status --help`
- `claude auto-mode critique/defaults/reset --help`
- `claude plugin details/disable/enable/eval/init/install/list/prune/tag/uninstall/update/validate --help`
- `claude plugin eval init --help`
- `claude plugin marketplace add/list/remove/update --help`
- `claude project purge --help`
- `claude import/gateway --help`

### 5. 隐藏命令与选项验证

以下不在 `claude --help` 输出中，属手动保留，需验证仍有效：

```shell
claude attach --help    # 隐藏命令：能显示 Usage 即存在
claude --exec x -p hi   # 隐藏选项：报 unknown option 即已移除，需从补全删除
```

曾验证有效的隐藏项：`attach` `kill` `stop` `rm` `logs` `respawn` `remote-control` 命令；`--advisor` `--bg` `--init` `--init-only` `--maintenance` `--rc` `--cloud` `--remote` `--teleport` `--tmux` `--max-turns` `--autocompact` 等选项。已移除的：`--exec` `--mcp-debug`。

**副作用警告**：`--tmux`/`--worktree`/`--bg` 测试会实际创建 worktree、会话或后台进程，测试后必须清理（`git worktree remove`、`git worktree prune`、`claude kill <id>`）。验证耗时选项时加 `timeout 10` 防挂起。

### 6. 对比要点

- 选项缺失 → 补充
- 参数必填/可选与 `[]`/`<>` 标注不一致 → 修正 `1:`/`::`
- 候选列表（如 `--scope` 的 user/project/local）不完整 → 对齐 help 描述
- 可重复选项漏标 `*` → 修复

---

## 二、系统工具列表检查

对比 `_claude_tool_names_impl` 中的工具名与当前系统工具列表（本会话系统 prompt 的 Tools 部分，随版本变化）：

- 新增工具 → 补充
- 移除工具 → 删除
- `mcp__*` 通配符保留

---

## 三、交叉检查

读取[官方文档](https://code.claude.com/docs/en/cli-reference)，检查 `_claude` 中遗漏的命令/选项，补充至补全文件。若网络不可达则以 CLI help 实际输出为准。

---

## 四、更新版本号

将第一步获取的版本号写入 `_claude` 头部 `# Version:` 行。

---

## 五、语法检查 + 部署

```shell
zsh -n _claude && shellcheck -x -o all _claude && rm -f ~/.zcompdump*; exec zsh
```

---

## 六、交互式验证（可选）

用 zpty 模拟真实补全，验证关键变更：

```shell
zmodload zsh/zpty
zpty ztest 'zsh -f -i'
zpty -w ztest 'autoload -Uz compinit; compinit -d /tmp/zc'
zpty -w ztest 'fpath=(/path/to/repo $fpath); autoload -Uz _claude; compdef _claude claude'
zpty -w ztest 'claude im'
zpty -w ztest $'\t'
sleep 1
zpty -r ztest
zpty -d ztest
```

注意：必须 `compdef _claude claude` 注册，否则 Tab 不触发补全而启动真实 claude 会话。整个测试用 `timeout 10 zsh -fc '...'` 包裹防挂起。

---

## 七、更新文档

更新 CLAUDE.md 的子命令覆盖表与 README.md（命令数、命令列表、选项列表）。

---

## 注意

- 系统 prompt 工具列表和 CLI help 输出随版本更新可能变化，运行时以实际输出为准
- `.zcompdump` 残留会导致旧版生效，部署必须清除
- 子命令 `help` 通常不需加入补全（仅代理到 `--help`），忽略即可
- `cache-clear` 为手动添加的隐藏命令，不在 help 输出中属正常
- help 输出用 `|` 标注的别名（如 `plugin|plugins`、`install|i`、`init|new`）需同时保留两侧

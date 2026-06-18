# 检查 _claude 补全文件是否与当前 Claude Code 版本一致

自动对比 CLI 命令/选项、系统工具列表与 `_claude` 补全文件，发现缺失项并更新，最后同步版本号。

---

## 一、检查命令与选项

自动运行 `claude` 及各子命令 `--help`，对比 `_claude` 中的补全实现。

### 步骤

1. **获取 CLI 命令列表**：`claude --help`
2. **获取主选项**：同上，逐项对比 `_claude` 主函数中的选项规格
   - 缺失选项 → 补充
   - 选项 `...`/`(repeatable)` 标注但补全缺 `*` 前缀 → 修复
3. **获取各子命令**：逐一运行 `claude <cmd> --help`
   - `claude agents --help`
   - `claude auth --help`
   - `claude auto-mode --help`
   - `claude doctor --help`
   - `claude install --help`
   - `claude mcp --help`
   - `claude plugin --help`
   - `claude project --help`
   - `claude setup-token --help`
   - `claude ultrareview --help`
   - `claude update --help`
   - `claude daemon --help`
4. **对比子命令列表**：逐一确认 `_claude_cmd`、`_claude_mcp_cmds`、`_claude_plugin_cmds` 等函数中的子命令与 help 输出一致
5. **对比子命令选项**：运行 `claude <cmd> <sub> --help` 获取深层选项，对比对应补全函数
   - 必填/可选参数位置标注是否一致（`1:` vs `::`）
   - 候选列表是否完整
6. **对比要点**：
   - 选项缺失 → 补充
   - 参数必填/可选与 `[]`/`<>` 标注不一致 → 修正 `1:`/`::`
   - 候选列表（如 `--scope` 的 user/project/local）不完整 → 对齐 help 描述
   - 可重复选项漏标 `*` → 修复

---

### 二、交叉检查

读取[官方文档](https://code.claude.com/docs/en/cli-reference)，检查 `_claude` 中遗漏的命令/选项，补充至补全文件。

## 三、更新版本号

修改完成后，获取当前版本并更新 `_claude` 头部注释：

```shell
claude --version
```

将版本号写入 `_claude` 文件的 `# Version:` 行。

---

## 四、语法检查 + 部署

```shell
zsh -n _claude && rm -f ~/.zcompdump*; exec zsh 
```

## 五、更新文档

更新 CLAUDE.md 以及 README.md

---

## 注意

- 系统 prompt 工具列表和 CLI help 输出随版本更新可能变化，运行时以实际输出为准
- `.zcompdump` 残留会导致旧版生效，部署必须清除
- 子命令 `help` 通常不需加入补全（仅代理到 `--help`），忽略即可
- `cache-clear` 为手动添加的隐藏命令，不在 help 输出中属正常

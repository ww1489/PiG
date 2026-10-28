> PiG 可以生成提示模板。让它按你的工作流做一个。

# 提示模板

提示模板是一段 Markdown，展开后成为完整提示。在编辑器里输入 `/name` 调用模板，`name` 是去掉 `.md` 的文件名。

> 本页是英文页 `prompt-templates.md` 的中文译本，对应 PiG 提交 `c64bebd8`。英文页是规范来源；两边不一致时以英文页为准。

## 位置

PiG 从这些地方加载提示模板：

- 全局：`~/.pig/agent/prompts/*.md`
- 项目：`.pig/prompts/*.md`（仅在项目被信任之后）
- Package：`prompts/` 目录，或 `package.json` 里的 `pi.prompts` 条目
- 设置：`prompts` 数组，值为文件或目录
- 命令行：`--prompt-template <path>`（可重复）

用 `--no-prompt-templates` 关闭发现。在活动会话里新增或修改模板后，执行 `/reload`。

`prompts` 设置条目开头的 `~` 会展开为主目录，`!`、`+`、`-` 和通配符条目也一样，例如 `"!~/prompts/old.md"`，与 Pi 1.4.0 一致。

## 格式

```markdown
---
description: Review staged git changes
---
Review the staged changes (`git diff --cached`). Focus on:
- Bugs and logic errors
- Security issues
- Error handling gaps
```

- 文件名就是命令名。`review.md` 变成 `/review`。
- `description` 可选。缺失时取第一行非空文本。
- `argument-hint` 可选。设置后，补全下拉框会在描述前显示参数提示。

### 参数提示

在 frontmatter 里用 `argument-hint` 展示预期参数。必填参数用 `<尖括号>`，可选参数用 `[方括号]`：

```markdown
---
description: Review PRs from URLs with structured issue and code analysis
argument-hint: "<PR-URL>"
---
```

补全下拉框里显示为：

```text
→ pr   <PR-URL>      : Review PRs from URLs with structured issue and code analysis
  is   <issue>       : Analyze GitHub issues (bugs or feature requests)
  wr   [instructions]: Finish the current task end-to-end
  cl  : Audit changelog entries before release
```

## 使用

在编辑器里输入 `/` 加模板名。补全会列出可用模板和描述。PiG 在文本到达 Agent 之前展开模板。扩展先通过 `input` 事件拿到原始输入，除非有同名扩展命令处理了它。

```text
/review                           # 展开 review.md
/component Button                 # 带一个参数展开
/component Button "click handler" # 多个参数
```

## 参数

模板支持位置参数、默认值和简单切片：

- `$1`、`$2`……位置参数
- `$@` 或 `$ARGUMENTS` 表示全部参数拼接
- `${1:-default}` 在参数存在且非空时用第 1 个参数，否则用 `default`
- `${@:-default}` 或 `${ARGUMENTS:-default}` 在参数存在且非空时用全部参数，否则用 `default`
- `${@:N}` 表示从第 N 个位置开始的参数（从 1 计数）
- `${@:N:L}` 表示从第 N 个位置起取 L 个参数

参数遵循类 shell 的引号规则，所以 `/review "API compatibility"` 提供的是一个带空格的参数。

示例：

```markdown
---
description: Create a component
---
Create a React component named $1 with features: $@
```

默认值适合可选参数：

```markdown
Summarize the current state in ${1:-7} bullet points.
```

用法：`/component Button "onClick handler" "disabled support"`

## 加载规则

- `prompts/` 里的模板发现不递归。
- 想放到子目录，就用 `prompts` 设置或 Package 清单显式加入。

## 完整示例

把下面内容保存为 `~/.pig/agent/prompts/review.md`：

```markdown
---
description: Review staged git changes
argument-hint: "[focus]"
---
Review the staged changes (`git diff --cached`). Focus on: $1
Bugs, security issues and error handling gaps.
```

在编辑器里输入斜杠和名字开头（这里是 `re`）。补全列表显示模板的参数提示、描述和来源（`[u]` 是你自己的模板，`[p]` 是项目的）：

```text
  reload      Reload keybindings, extensions, skills, prompts, themes, and context files
  review      [focus] — [u] Review staged git changes
```

输入 `/review security` 并回车。PiG 发出的是展开后的文本，不是命令本身。在 Provider 请求中核对：

```text
Review the staged changes (`git diff --cached`). Focus on: security
Bugs, security issues and error handling gaps.
```

参数形式在 PiG 上核对过。对模板 `Create a React component named $1 with features: $@`，输入 `/component Button "onClick handler" "disabled support"` 会发出：

```text
Create a React component named Button with features: Button onClick handler disabled support
```

`$@` 表示**全部**参数，包含第一个。对模板 `Summarize the current state in ${1:-7} bullet points. Rest: ${@:2}`，保存为 `sum.md`：

```text
you type:   /sum
PiG sends:  Summarize the current state in 7 bullet points. Rest:

you type:   /sum 3 alpha beta
PiG sends:  Summarize the current state in 3 bullet points. Rest: alpha beta
```

模板名和内置命令冲突时永远不会执行：内置命令优先（见 [斜杠命令](/docs/latest/slash-commands)）。

参考实现的完整说明见 [上游 Pi 的提示模板文档](https://pi.dev/docs/latest/prompt-templates)。

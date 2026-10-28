# 技能

技能（Skill）是一段任务说明，加载后进入当前会话。扩展增加的是**能力**（工具、命令），技能增加的是**知识**：怎样把某件事做好，用你自己的话写，不用写代码。PiG 会告诉模型有哪些技能；任务匹配时，模型读取技能正文。你也可以用 `/skill:<name>` 自己加载。

适合做成技能的是**流程和约定**，不是新工具：提交信息格式、这个仓库怎么跑测试、发布步骤。

> 本页是英文页 `skills.md` 的中文译本，对应 PiG 提交 `c64bebd8`。英文页是规范来源；两边不一致时以英文页为准。

## 磁盘上的技能长什么样

一个技能是一个以技能命名的目录，里面有 `SKILL.md`：

```text
~/.pig/agent/skills/commit/SKILL.md
```

`SKILL.md` 是带 YAML frontmatter 的 Markdown：

```markdown
---
name: commit
description: Write a commit message in this repo's format.
---

Write commits as `type(scope): summary`. Keep the summary under 72 chars.
Explain *why* in the body, not what. Never mention tooling.
```

- **`name`**：技能标识，默认取目录名。
- **`description`**：一行摘要。
- **正文**：具体说明。PiG 不会把正文放进系统提示，只列出每个技能的名称、描述和文件位置；模型需要正文时，用自己的 `read` 工具打开文件。模型没有 `read` 工具时，提示里不会点名某个工具。`/skill:<name>` 则把正文直接放进你的消息（见下面的例子）。

如果某个技能只想通过 `/skill:<name>` 使用，在 frontmatter 里写 `disable-model-invocation: true`。这样它不会出现在系统提示的技能列表里。

## 加载技能

只对某一次运行生效：

```bash
pig --skill commit
pig --skill ./path/to/skill-dir      # 直接给路径也可以
```

写进 Piglet，让该 Agent 每次运行都加载（见 [Piglets](/docs/latest/piglets)）：

```yaml
skills:
  - name: commit
    origins: [local:./skills/commit]
```

Piglet 里的每个技能条目都必须有 `origins`，通常是已声明的 `package:<alias>` 成员或 `local:<相对路径>`。只写名字没有来源，`pig piglet validate` 会报 `FAIL: skill "commit": no origin specified`（`coding/piglet/origins.go:262`）。

## 发现规则

除了显式 `--skill` 和 Piglet 条目，PiG 还能发现项目或主目录里已有的技能。把发现当成本地便利，不要当成交给别人的 Agent 的依赖契约。可移植的 Piglet 应当声明技能来源。

PiG 扫描这些位置：

- `~/.pig/agent/skills/`：你自己的技能；
- `~/.agents/skills/`：与其他 Agent 共用的用户级技能；
- 从工作区逐级向上到仓库根的每个目录下的 `.agents/skills/`；不在仓库里时一直到文件系统根；
- 工作区里的 `.pig/skills/`。

`~/.pig/skills/`（没有 `agent` 那一层）不会被扫描，放在那里的技能会被静默忽略。项目技能只在你信任该项目后才加载（见 [安全](zh-security.md)）。

`skills` 设置项可以追加文件和目录，并过滤发现结果（见 [设置](/docs/latest/settings)）。普通条目以及 `!`、`+`、`-` 和通配符条目里的开头 `~` 都展开为你的主目录，例如 `"!~/claude-backup/skills/pdf"`，与 Pi 1.4.0 一致。更早的版本会把这类模式当字面文本匹配，结果什么也匹配不到（Pi #10765）。

扫描是否执行由 Piglet 控制，受限 Agent 不会拿到没有声明的东西：

```yaml
discovery:
  skills: [workspace]  # 只发现项目 Package 或项目约定的技能
```

## 完整示例

按上面的格式创建技能：

```bash
mkdir -p ~/.pig/agent/skills/commit
cat > ~/.pig/agent/skills/commit/SKILL.md <<'EOF'
---
name: commit
description: Write a commit message in this repo's format.
---

Write commits as `type(scope): summary`. Keep the summary under 72 chars.
Explain *why* in the body, not what. Never mention tooling.
EOF
```

启动 PiG，输入 `/skill:`。列表会标出每个技能的作用域：`[u]` 是你自己的，`[p]` 是项目的。在 PiG 上核对过，两个项目技能加两个用户技能：

```text
→ skill:cat     [p] Skill cat.
  skill:dog     [p] Skill dog.
  skill:commit  [u] Write a commit message in this repo's format.
  skill:amy     [u] Skill amy.
```

输入 `/skill:commit write one for my change` 并回车。PiG 会把技能正文和你的话一起发出去。模型收到的消息（在 Provider 请求中核对）是：

```text
<skill name="commit" location="/home/you/.pig/agent/skills/commit/SKILL.md">
References are relative to /home/you/.pig/agent/skills/commit.

Write commits as `type(scope): summary`. Keep the summary under 72 chars.
Explain *why* in the body, not what. Never mention tooling.
</skill>

write one for my change
```

不带 `/skill:` 前缀时，模型仍然知道这个技能存在，因为系统提示里有下面这段；当请求与描述匹配时，它可以自己读文件：

```text
<available_skills>
  <skill>
    <name>commit</name>
    <description>Write a commit message in this repo&apos;s format.</description>
    <location>/home/you/.pig/agent/skills/commit/SKILL.md</location>
  </skill>
</available_skills>
```

`--no-skills` 会去掉这段列表，同时移除 `/skill:` 命令。`enableSkillCommands: false` 只移除命令（见 [设置](/docs/latest/settings)）。

## 分享技能

技能和其他资源一样分发：打包进 [Package](zh-packages.md) 后用 `pig install` 安装，或者发布到 Marketplace。一个 Package 可以同时带技能、扩展、提示模板和主题。

## 另见

- [概念](/docs/latest/concepts)：技能在资源、Package 和 Piglet 中的位置。
- [Piglets](/docs/latest/piglets)：把技能集合声明为 Agent 的一部分，并控制发现开关。
- [Packages](zh-packages.md)：分发和安装技能。
- [上游 Pi 的技能文档](https://pi.dev/docs/latest/skills)：参考实现的技能格式与位置。

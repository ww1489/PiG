# Packages

Package 是带有版本的 PiG/Pi 资源分发封套：扩展、技能、提示、主题、钩子、MCP 定义、Agent 环境和兼容的跨工具 Agent。安装 Package 可以给 PiG 增加能力。用独立的 [Piglet](/docs/latest/piglets) 把能力选择并限定到一个有名字的 Agent。

回到 [文档导航](/docs/latest)，或先读简短的 [概念模型](/docs/latest/concepts)。

> 本页是英文页 `packages.md` 的中文译本，对应 PiG 提交 `c64bebd8`。英文页是规范来源；清单字段和命令以英文页与当前二进制为准。

## Package 可以携带什么

| 资源 | 用途 |
|---|---|
| 扩展 | 可执行能力：工具、命令、钩子、Provider、UI |
| 技能 | 任务说明 |
| 提示 | 可复用的提示或模板 |
| 主题 | 终端外观 |
| 钩子 | 事件命令元数据 |
| MCP 定义 | 外部工具服务器配置 |
| Agent 环境 | 标准 Dev Container 定义与可移植的本地闭包 |
| 跨工具 Agent | 兼容的插件根可以暴露 Agent 声明 |

Package 不负责编写 Piglet、Piglet Binary 或 Piglet Image 清单。

## Package 源码布局

把自己的资源直接放在 Package 根下：

```text
base-coding/
├── package.json
├── extensions/
├── skills/
├── prompts/
├── themes/
├── hooks/
├── mcp/
└── .devcontainer/
```

不要在 Package 根和这些目录之间再加一层 `resources/`。兼容上游的 `package.json` `pi` 块可以显式选择成员：

```json
{
  "name": "@example/base-coding",
  "version": "1.0.0",
  "pi": {
    "extensions": ["extensions/*"],
    "skills": ["skills/*"],
    "prompts": ["prompts/*.md"],
    "themes": ["themes/*.json"]
  }
}
```

PiG 专有的成员关系放在增量式 `pig` 块里，不改 Pi 的字段。与 Pi 一致，带 `pi` 块的 Package 只加载它声明的内容：钩子、MCP 服务器和 Agent 环境来自 `pig` 块，而不是约定目录。

```json
{
  "pig": {
    "hooks": ["hooks/*.json"],
    "mcpServers": ["mcp/*.json"],
    "agentEnvironments": [".devcontainer/go/devcontainer.json"]
  }
}
```

## 编写与校验

成员关系直接写在普通 `package.json` 里，使用标准 `pi` 块和上面的可选 PiG 资源字段。不安装也能校验：

```bash
pig package validate ./base-coding
pig package validate ./base-coding --json
```

校验会拒绝畸形清单和 JSON 资源、缺失成员、重复的非技能公开名称、词法或符号链接逃逸、缺失的 Compose 服务，以及绝对或逃逸的 bind mount。

启动时，重复的技能名保留第一个有效定义，并产生一条 `[Skill conflicts]` 警告，列出选中和被跳过的路径，与 Pi 一致。解析出的项目、用户和 Package 技能排在额外的 `--skill` 路径之前。用 `--no-skills` 只加载显式请求的技能。项目技能（包括祖先目录的 `.agents/skills`）需要项目信任。技能条目可以指向 Markdown 文件或目录。清单里的 `prompts/**/*.md` 这类 globstar 会匹配该目录根和任意深度的文件。带 `*` 或 `?` 的清单条目按 Pi 中 Node `fs.globSync` 的规则展开，因此花括号集合、`*.@(md|txt)` 这类 extglob 和 POSIX 字符类都有效；在 Windows 和 macOS 上，通配段不区分大小写。Pi 无法编译的清单 glob、覆盖或 Package 过滤器，会让本地 Package 丢掉该资源类型及其后的类型；对 npm 或 git Package 则直接停止启动，与 Pi 一致。

直接安装 Package 保留上游行为：在设置里记录 Package，常规发现暴露已启用成员（默认全部，除非 Package 过滤器收窄）。安装不会复制 Piglet 或 Agent、合并 MCP 配置、创建诊断或生成入口。Piglet 的 Package 依赖通过同一个 materializer 解析，但不修改全局或项目设置；只有显式成员选择才激活。

## 安装

```bash
pig install ./local-package
pig install git:https://github.com/example/package.git
pig install 'git:https://github.com/example/monorepo.git@v2#subdirectory=packages%2Freview'
pig install npm:@example/package
pig install 'npm:@example/private@1.2.3?registry=https%3A%2F%2Fnpm.example.com%2Fteam'
```

注册表来源用 `npm:`，Git 简写（如 `github.com/user/repo` 或 `git@github.com:user/repo`）用 `git:`。没有前缀时，这些写法是本地路径，即使路径不存在。显式的 HTTPS 和 SSH URL 不需要 `git:`。

PiG 按 Pi 的方式解析 Git 来源，差异如下（D110）。`git://host/org/repo` 是 `git://` 协议 URL；`HTTPS://` 这类大写 scheme 按其小写形式工作。Pi 会把两者变成任何 Git 客户端都克隆不了的 URL。以 `subdirectory=` 开头的 fragment 选择子目录；Pi 把每个 `#<text>` 都读成 ref。

普通 npm 库（如 `npm:is-number@7.0.0`）即使不含 Pi 资源也能安装并出现在 `pig list` 里。PiG 安装时不会导入它们的 JavaScript 来分类。根目录静态证明存在 Go、Rust 或 Python 扩展契约、却不贡献任何 Package 资源时会被拒绝（D57）；这种扩展请直接用 `pig -e <path>` 加载。

Git 子目录选择器写成 `#subdirectory=<URL 转义的相对路径>`。PiG 在规范 Git 根下只做一次仓库检出，在内部解析选中的 Package 根，拒绝缺失或符号链接逃逸的选择，并在另一个已配置 Package 仍使用相邻子目录时保留该检出。

自定义 npm 注册表选择器写成 `?registry=<URL 转义的绝对 HTTPS URL>`。PiG 给每个注册表一个独立的受管根，并把不含秘密的 URL 传给 npm、pnpm 或 Bun。带内嵌凭据、查询串或 fragment 的注册表 URL 会被拒绝。令牌和自定义 CA 通过该包管理器正常的 `.npmrc`、环境变量或证书机制配置；PiG 从不接受或写出注册表令牌。

对用户作用域的 Package，版本元数据查询在受管包存储中运行，不读取调用项目里的 `.npmrc`（安全差异 D79）。你的用户配置和显式 `npmCommand` 参数仍然生效。受信任的项目作用域 Package 保留 Pi 的项目 `.npmrc` 行为。

PiG 根据 `npmCommand` 可执行文件名（或其最后一个 `--` 之后的名字）选择 npm、pnpm 或 Bun 参数。它只去掉 `.cmd` 或 `.exe` 后缀并保留大小写，不运行版本探测来识别包管理器。

产品可以通过 PiG 的安装解析器契约增加来源 scheme。例如，启用产品的 PiG 可以接受 `marketplace:` 来源；裸 PiG 不需要知道该产品的 HTTP 或认证协议。

## 只校验不安装

```bash
pig install ./extension --validate-only --json
pig install --validate-only --set ./ext-a,./ext-b --json
```

校验会解析、构建、启动并注册选定的扩展集合，不修改 Package 设置。发布前、或声称某个 Piglet 的扩展能协同工作之前，用它验证。

## 作用域

安装默认是用户作用域。`--local` 在工作区的 `.pig/settings.json` 里记录依赖：

```bash
pig install ./workspace-extension --local
```

PiG 只物化受管来源一次：

| 作用域/来源 | 根目录 |
|---|---|
| 用户 npm | `$PIG_CODING_AGENT_DIR/npm`（默认 `~/.pig/agent/npm`） |
| 用户 Git | `$PIG_CODING_AGENT_DIR/git` |
| 下载的目录 bundle | `$PIG_CODING_AGENT_DIR/catalog` |
| 项目 npm | `<workspace>/.pig/npm` |
| 项目 Git | `<workspace>/.pig/git` |
| 本地路径 | 保留在作者给出的路径 |

插件和 Package 记录可以指向同一个物化来源根；PiG 不会把它复制成多份缓存。

## 分享 Package

把 Package 发布到 npm 或 Git 托管。其他用户用 `pig install npm:<name>` 或 `pig install git:<url>` 安装，具体命令见上文“安装”一节。

要让 Package 出现在 npm 上，在 `package.json` 里加关键字 `pig-package` 和描述，然后 `npm publish`。`pig package publish [<dir>] --to npm` 是可选的便捷命令：它会校验 Package、检查目录读取的字段与 `name@version` 是否可用、展示 npm 的 dry run，只有带 `--yes` 才真正发布。pi-in-go.dev 目录站每天构建时按 npm 关键字发现包。见 [发布 Package 或 Piglet](/docs/latest/publishing)。

## 编辑器与 ACP：pig-acp

`pig install npm:@pi-in-go/pigpen-acp` 安装 Package，其中的 `/acp` 命令会打印编辑器配置。它不会安装 `pig-acp` 可执行文件。支持 Agent Client Protocol 的编辑器（Zed、JetBrains）启动 `pig-acp`，`pig-acp` 再为每个会话启动 `pig --mode rpc`。目前没有预构建的 `pig-acp` 下载，所以用一条命令从 Package 附带的源码构建。它没有依赖，需要 Go 1.26 或更新版本（`pig setup go` 可以安装）。

```sh
pig install npm:@pi-in-go/pigpen-acp
pig acp install
pig-acp --version        # pig-acp 0.1.0 (ACP protocol version 1)
```

`pig acp install` 用已安装 Package 的文件离线工作，用本地 Go 工具链构建，并把 `pig-acp`（Windows 上是 `pig-acp.exe`）写到 `pig` 旁边：`$PIG_BIN` 目录，否则 `~/.local/bin`。它会打印路径；目录不在 `PATH` 上时会提示，此时在编辑器里用完整路径。

然后让编辑器指向它。Zed 的 `settings.json`：

```json
{
  "agent_servers": {
    "PiG": {
      "type": "custom",
      "command": "pig-acp",
      "args": ["--pig", "/path/to/pig"],
      "env": {}
    }
  }
}
```

`--pig` 指定要驱动的 `pig` 可执行文件（或构建好的 Piglet Binary）；不给时 `pig-acp` 使用自己的覆盖变量（见 `pig-acp --help`），再用 `PATH` 上的 `pig`。如果 pig 还没有模型或密钥，编辑器会显示认证错误，并提示 `pig-acp --terminal-login`，它会在编辑器终端里启动 pig 供你登录。安装 Package 或构建 `acp` Piglet 都不会把 Piglet Binary 变成 ACP 服务器；编辑器始终运行 `pig-acp`。

## 管理已安装的 Package

PiG 保留上游 Pi 的顶层 Package 动词：

```bash
pig package list
pig update
pig update --self
pig update <source>
pig update --extension <source>
pig update --extensions
pig update --models
pig update --all
pig update --force
pig remove <source>
pig uninstall <source>
pig uninstall <source> --local --approve
```

`uninstall` 是 `remove` 的别名，包括 `--local` 和帮助。两条命令都从选定的设置作用域移除 Package。移除最后一个 Package 后，该设置文件里仍保留 `"packages": []`。

Package 命令在开始工作前显示 `Installing`、`Removing` 或 `Updating`。npm 和 Git 把自己的输出写到终端。子命令失败时，先打印它的诊断，再由 CLI 报告 `Error: <command> failed with code <n>` 并以状态 1 退出。

npm 更新按作用域批量处理符合条件的 Package。Git 更新会核对配置的 ref；检出变化或依赖缺失时重装依赖。为已配置的 Package 安装另一个 ref 会替换来源，但保留资源过滤器。新的 Git 安装失败会删除不完整的检出。

启动时，npm Package 需要已安装清单版本满足配置的 pin 或范围。安装失败或离线模式阻止安装时，旧安装不提供资源。Package 列表仍报告物理安装。带 `autoload: false` 的项目覆盖会沿用继承的用户 Package 的安装和版本要求，同时保留项目资源元数据。

裸 `pig update`、`--self` 和 `self`/`pig` 位置目标更新 PiG。`--extensions` 只更新已安装 Package；`--all` 先更新 Package 再更新 PiG。`--extension` 指定一个 Package，`--force` 重装当前 PiG 版本。`--models` 通过共享模型运行时强制刷新模型目录，不更新 PiG 或已安装 Package，且不能与另一个更新目标组合。冲突目标在任一更新路径开始前失败。

打开全局 `extensionsAutoUpgrade` 设置后，`pig update` 还会对为旧 SDK 编写的 Go 扩展运行 `pig extension upgrade` 规则（见 [扩展](zh-extensions.md)）。

PiG 自己的管理命令使用 `pig package list` 这类完整单词的名词/动词形式，同时保留与上游兼容的安装、移除和更新行为。裸 `pig list` 保留上游兼容的 Package 列表。以你运行的那个二进制的 `pig --help` 为准。

### 启用或禁用一个已安装资源

不想移除整个 Package、只想停用一个资源时，用 Package 过滤器。过滤器在清单选择之后应用，所以无法找回被清单排除的资源。过滤器内，普通模式选择资源，`!pattern` 排除 glob 匹配，`+path` 恢复确切路径，`-path` 禁用确切路径（即使已被恢复）。这些组按此顺序应用，与数组里的顺序无关。空过滤器数组会禁用该类型的所有资源。模式开头的 `~` 展开为主目录，与普通条目和 Pi 1.4.0 一致。更早的版本会把这类模式当字面文本匹配，什么都匹配不到（Pi #10765）。

```bash
pig config
pig config --local
```

用 `pig config --local` 编辑项目作用域覆盖。新的 Package 覆盖条目带 `autoload: false`。循环回“继承”会移除仅覆盖条目，但保留显式安装的项目 Package 和无关的资源过滤器。设置保存并重新加载后，空的 Package 与资源过滤器数组保持为空。

TUI 更新设置，但不更新 Package 的 `package.json`。启动时，声明了却什么都匹配不到的扩展、技能、提示或主题会被跳过（与 Pi 一致），Package 其余部分继续加载；`pig config` 会把它列为缺失，方便你禁用。`pig package validate` 对发布仍然严格。

启动时忽略畸形的 `package.json` JSON，转去发现约定目录。`pi` 对象只加载其有效资源数组。类型错误的字段不声明任何资源，也不会丢弃有效的同级字段。这些清单情况不产生诊断，与 Pi 一致。显式安装和发布校验仍然严格。

## 工具链与首次使用

在约定的 `extensions/` 目录里，每个顶层 `.ts` 或 `.js` 文件都是独立扩展，除非 `pi.extensions` 清单或 `index.ts`/`index.js` 选择了目录条目。`main.ts`、`main.js` 或 `extension.ts` 文件名不会隐藏同级扩展。没有扩展条目的 `package.json` 不会自行选择该目录。

安装会物化源码，不一定立刻构建每个扩展。源码扩展在首次使用时构建进内容寻址的运行时单元：

| 扩展语言 | 首次源码构建的宿主要求 |
|---|---|
| Go | Go 工具链 |
| Rust | Cargo/Rust 工具链 |
| Python | Python 与依赖 |
| Node/TypeScript | Node 运行时与依赖 |

缓存让之后的匹配加载很快，但缓存路径不是包版本或可复现身份。

## 交付 Piglet

要把 Piglet 交给别人，就分发它的源码，或用 `pig piglet build` 构建的 Piglet Binary。使用你自己管理的基础设施，例如 Git 仓库或文件共享。接收方用 `pig piglet add` 安装源码，或直接运行 Binary。

用 `pig piglet publish <name> --to github --repo <owner/repo> --sign-key <key>` 把已签名 Binary 发布为一个 GitHub Release。没有 `--yes` 时发布只是 dry run。接收方用 `pig piglet pull <release-index-url|github:owner/repo@version>` 安装已签名 Binary。见 [Piglet Binaries](/docs/latest/piglet-binaries#publish-to-github-releases)。

用 `pig piglet publish <name> --to npm` 把 Piglet 源码发布到 npm；`pig piglet add npm:<name>` 安装它。见 [发布 Package 或 Piglet](/docs/latest/publishing)。`pig piglet pull` 不接受已安装 Piglet 的名字。见 [Piglets](/docs/latest/piglets#planned-not-in-this-release-named-pulls-and-image-artifacts)。

## Package 与 Piglet

| Package | Piglet |
|---|---|
| 分发源码/资源 | 选择/限定一个独立的具名 Agent，可以构建 Binary 制品；Image 保留 |
| 作为整体安装与更新 | 通过 Host 或 Binary 载体运行；Image 输出保留 |
| 永远不拥有 Piglet | 可以引用零个或多个 Package 以及直接来源 |
| 保留源码来源信息 | 记录 Agent 默认值与确切的组件实现/物化方式 |

Piglet 与 Package 保持独立。Piglet 发布用 `pig piglet publish --to github` 把已签名 Binary 面发布到 GitHub Releases，用 `--to npm` 把源码发布到 npm。`pig piglet build --format image` 是保留功能，不产生 Image。

## 编写时的安全

不要隐式把运行中会话的所有资源复制进新 Package。会话里可能有许可证和来源都不允许重新打包的第三方资源。编写 Package 时应显式加入资源并保留其来源/版本信息。从会话创建 Piglet 更安全，因为它写的是引用，而不是复制源码。

参考实现见 [上游 Pi 的 Package 文档](https://pi.dev/docs/latest/packages)。

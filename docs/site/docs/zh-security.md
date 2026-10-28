# 安全

PiG 以启动它的用户权限运行。PiG 不是模型输出、工具、扩展、技能、钩子或 shell 命令的安全沙箱。

需要隔离时，请使用容器、虚拟机或其他操作系统边界。

> 本页是英文页 `security.md` 的中文译本，对应 PiG 提交 `c64bebd8`。英文页是规范来源；安全相关的具体行为以英文页和当前源码为准。

## 项目信任

仓库里可能有改变 PiG 行为的项目设置和资源。PiG 在加载项目控制的、可执行或可配置的输入之前，要求一个信任决定。

项目信任不是完整的启动边界。与 Pi 一样，PiG 在选择或创建会话时会读取项目的 `sessionDir` 设置，这发生在解析项目信任之前。项目信任也不限制工具调用能访问什么：已启用的工具使用 PiG 进程的操作系统权限。

受信任状态影响的项目路径包括：

```text
.pig/settings.json
.pig/mcp.json
.pig/extensions/
.pig/skills/
.pig/prompts/
.pig/themes/
.pig/SYSTEM.md
.pig/APPEND_SYSTEM.md
.agents/skills/
```

单独的 `.pig` 目录本身不需要项目信任。`AGENTS.override.md`、`AGENTS.md`、`CLAUDE.md` 这类上下文文件无论信任决定如何都会加载，除非你关闭上下文加载。即使拒绝项目信任，也要把文件夹里的指令当作不可信输入。

`.pig/piglets/` 里的 Piglet 也属于项目输入。`--piglet <name>` 只有在已有保存的信任决定、`defaultProjectTrust: "always"` 或 `--approve` 时才使用它；裸名字永远不会从工作目录读取。

PiG 按与 Pi 相同的方式决定信任：

1. 命令行的 `--approve` 或 `--no-approve` 优先。
2. 用户级和命令行扩展可以处理 `project_trust` 事件。第一个返回是或否的扩展拥有决定权。
3. 否则使用当前目录或其最近父目录已保存的决定。决定按规范目录存放在 `~/.pig/agent/trust.json`。
4. 否则遵循全局 `defaultProjectTrust`，默认值是 `"ask"`。

Print、JSON 和 RPC 模式无法显示信任提示。没有覆盖、扩展或已保存决定时，`defaultProjectTrust: "always"` 会加载受保护的项目文件，`"ask"` 或 `"never"` 会跳过它们。

用 `/trust` 管理当前项目的信任决定。选择器分别显示已保存的决定和当前会话的信任状态；浏览时勾号停留在已保存的选择上。选择 **Trust parent folder** 会信任父目录并清除当前项目的覆盖。修改信任后重启 PiG，让启动发现按新决定运行。

## 会话与凭据

会话可能包含提示、工具参数、命令输出、文件内容，以及对话中出现过的凭据。导出或分享会话前先检查。

PiG 创建 Agent 目录、`sessions` 目录和每个项目的会话目录时使用 0700 权限。Pi 使用 umask 默认值创建，通常是 0755（D132，Pi #10770）。已经存在的目录保持原权限。`auth.json` 以仅所有者可访问创建（POSIX 是 0600，Windows 是仅所有者的 DACL，D68）。

## 扩展

扩展会执行代码。子进程边界可以隔离崩溃并支持取消，但它不是权限沙箱。

加载扩展之前：

- 核实它的源码和来源；
- 审查依赖；
- 检查它需要哪些命令和网络访问；
- 弄清它能读哪些文件和密钥；
- 对不可信代码优先使用受限的操作系统环境。

融合扩展与 PiG 共享进程。要更严格地审查融合代码，因为 panic、死锁或资源泄漏会影响整个 Agent 应用。

## Packages

Package 可以分发可执行扩展和指令资源。安装 Package 可能让这些资源进入常规发现。

安装前检查 Package 成员：

```bash
pig package validate ./package
```

在不修改设置的情况下验证扩展行为：

```bash
pig install ./extension --validate-only --json
```

安装 Package 永远不会激活 Piglet。Piglet 另行选择 Agent 应用。

## Piglets

Piglet 是显式组合，不是安全边界。它可以收窄工具和发现范围、声明密钥、要求特定环境。

应用不该继承环境资源时，使用精确的工具列表和空的发现列表：

```yaml
tools: [read, grep, find]
discovery:
  extensions: []
  skills: []
```

命令行和平台策略可以收窄 Piglet，但不得静默放宽它。

## 密钥

密钥值不要放进可移植的 Piglet 源、Package 元数据、记录、会话、日志、命令参数或镜像层。

使用环境变量、仅所有者可读的文件，或受控的密钥解析器。PiG 会校验声明；所需值缺失或未授权时在启动前失败。

不要把凭据放进：

- Piglet YAML 文件；
- 扩展源码归档；
- 提示或技能；
- 网站示例；
- 生成的 Binary 记录。

## 模型 Provider

Provider 凭据可能授予付费服务和私有数据的访问权。使用提供方文档化的环境变量或 PiG 登录存储。不要把令牌复制进仓库设置。

把模型输出当作不可信输入。即使 Provider 连接可信，模型也可能提出有害命令。

## 连接失败

PiG 在有限的生命周期规则内检测扩展传输和心跳失败。它会取消自己拥有的待处理调用、关闭连接拥有的 UI 状态，并隔离或隔离处理受影响的扩展或单元。

PiG 不会自动重放被中断的操作。自动重放可能重复文件、shell、网络、消息或事务副作用。

见 [扩展](zh-extensions.md)。

## 不受信任的证书颁发机构

PiG 自己的 HTTPS 和 WebSocket 连接（模型 Provider、OAuth、MCP、包与工具下载、自更新）会拒绝包含八个 eMudhra 中间 CA 之一或三个根 CA 之一的证书链。这八个中间 CA 的私钥可以从公开证书中恢复（Mozilla bug 2083449，2026 年 10 月）；Sectigo 交叉签发了相同密钥，Apple 和 Microsoft 仍信任那个根。三个根是 Mozilla 从 NSS 3.131.1 移除的：`emSign ECC Root CA - G3`、`emSign Root CA - C1` 和 `emSign ECC Root CA - C3`。

检查比较公钥摘要，所以交叉签发的证书也会被拒绝；它运行在操作系统信任库之上，而后者在厂商更新前可能仍信任这些 CA。PiG 的检查拒绝证书链时，错误会指出服务器和 CA。PiG 固定的工具链自带的 Go 1.27.3 校验器已经用 `x509: no valid chains built` 拒绝这八个中间 CA；PiG 的检查增加了三个根，并覆盖使用旧版 Go 构建的情况。没有设置可以关闭它；使用 Provider 显式 `insecure` 选项建立的连接会完全跳过校验，因此没有可校验的链。扩展进程自己建立的连接使用该运行时的信任库：请运行 Node.js 22.23.4、24.22.1、26.12.1 或更新版本，它们带有相同的拒绝规则。

## Piglet Binary 校验

Piglet Binary 在分发命令前校验内嵌的 Piglet 和组件计划。用 `--sign-key` 构建的 Binary 还会离线校验 Ed25519 签名，并在可执行文件、清单或签名变化时拒绝运行。文件自上次成功启动以来未变化时，它会跳可执行文件字节的哈希。见 [校验缓存](/docs/latest/piglet-binaries#verification-cache) 了解这样保留了什么、放弃了什么。启动检查是自检：它无法阻止能改写 Binary 的攻击者。

用 `pig verify <binary>` 在不运行 Binary 的情况下报告签名状态。用 `pig piglet trust add <key.pub>` 信任一个已审查的发布者密钥，用 `pig piglet trust revoke <key-id>` 拒绝它，用 `pig piglet trust require on` 要求本地受信任密钥的签名。内嵌作者密钥证明的是从该密钥开始的连续性，不是持有者的人类身份。

Sigstore 无密钥证明是独立的构建来源证据。`pig verify --provenance --repo <owner/name> --signer-workflow <workflow> <binary>` 委托给 `gh attestation verify`，可能访问 GitHub。启动过程从不发起该网络请求。

它不会静默安装、下载、替换或削弱缺失的要求。执行前请校验发布校验和、离线签名和可用来源。

Binary 仍可能需要外部服务、配置、运行时和密钥。读它的记录，不要假定一个文件就自包含。

## 网络内容

文档站的包和模型目录包含第三方元数据。出现在列表里不代表 PiG 兼容、安全或背书。

使用前审查发布者、许可证、源码和完整性证据。

## 报告漏洞

不要为未公开漏洞开公开 issue。按仓库 `.github/SECURITY.md` 里的私密报告说明操作。不要公开漏洞利用细节。

## 相关文档

- [扩展](zh-extensions.md)
- [Packages](zh-packages.md)
- [Piglets](/docs/latest/piglets)
- [Piglet Binaries](/docs/latest/piglet-binaries)
- [容器化](/docs/latest/containerization)
- [上游 Pi 的安全文档](https://pi.dev/docs/latest/security)

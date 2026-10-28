# 快速开始

PiG 仍在积极开发中。macOS 和 Linux 可以用 `curl -fsSL https://pi-in-go.dev/install.sh | sh` 安装；支持平台也可以用 `npm install -g @pi-in-go/pig` 或 `go install github.com/MichaelKinsy/PiG/cmd/pig@latest`，或者从 [GitHub Releases](https://github.com/MichaelKinsy/PiG/releases) 下载压缩包。Windows 支持是预览状态。想从源码构建，按下面的步骤操作。Pi 1.4.0 自己的安装方式见 [Pi 的快速开始](https://github.com/earendil-works/pi/blob/v1.2.0/packages/coding-agent/docs/quickstart.md)。

如果你没用过终端，先读 [最初的十分钟](/docs/latest/first-run)，它会逐步解释每一步。

> 本页是英文页 `quickstart.md` 的中文译本，对应 PiG 提交 `c64bebd8`。英文页是规范来源；命令与版本以英文页和实际二进制为准。

## 用安装脚本安装

在 macOS 和 Linux 上，安装脚本下载最新的正式发布，用发布自带的 `SHA256SUMS` 校验，然后把 `pig` 安装到 `~/.local/bin`：

```bash
curl -fsSL https://pi-in-go.dev/install.sh | sh
pig --version
```

如果 `~/.local/bin` 不在 `PATH` 里，脚本会打印需要执行的 `export PATH=...` 那一行。[安装排错](/docs/latest/install-troubleshooting) 列出了脚本选项，例如用 `PIG_VERSION` 安装指定版本。

## 环境要求

只有从源码构建 PiG 才需要这些工具。用 Go 1.27.3 构建 `pig` 可执行文件。Go 1.27 的 macOS 发布二进制要求 macOS 13 或更新版本。

完整验证套件还会用到：

- Git；
- Node.js 24.22.1 和 npm 12.1.0；
- Python 3.12；
- Rust 1.97.1；
- Unix 上的 tmux。

运行已经构建好的可执行文件，不需要这些验证工具。

## 从源码构建

克隆仓库并构建 PiG：

```bash
git clone https://github.com/MichaelKinsy/PiG.git
cd PiG
go build -o bin/pig ./cmd/pig
./bin/pig --version
```

不要用上游 Pi 的 npm 命令安装 PiG。`@earendil-works/pi-coding-agent` 安装的是 Pi，不是 PiG。

## 用 npm 安装

用 npm 安装命令（Node.js 18 或更新版本）：

```bash
npm install -g @pi-in-go/pig
pig --version
```

也可以只运行一次、不安装：

```bash
npx @pi-in-go/pig --version
```

`pig` 本身就是原生二进制：npm 把对应平台的包（`@pi-in-go/pig-<os>-<cpu>`，覆盖 macOS、Linux 和 Windows 的 x64 与 arm64）作为可选依赖装进来，Node.js 只跑一个小小的启动器。不要用 `--omit=optional` 或 `--no-optional` 安装，那会把二进制排除掉。用 `npm update -g @pi-in-go/pig` 或 `pig update` 更新（后者会对它找到的版本执行 `npm install -g @pi-in-go/pig@<version>`），用 `npm uninstall -g @pi-in-go/pig` 卸载。

## 用 Go 安装

Go 1.26 或更新版本可以不克隆仓库直接安装。Go 会下载 PiG 的源码和依赖并在本机编译，第一次可能要几分钟：

```bash
go install github.com/MichaelKinsy/PiG/cmd/pig@latest
```

需要可复现安装时指定确切版本：

```bash
go install github.com/MichaelKinsy/PiG/cmd/pig@v0.5.0
```

Go 把 `pig` 写到 `GOBIN`。`GOBIN` 为空时，Go 使用 `GOPATH` 第一个路径下的 `bin`，即 `$(go env GOPATH)/bin`，默认是 `~/go/bin`（Windows 上是 `%USERPROFILE%\go\bin`）。使用较旧的 Go（1.21 及以后）且保持默认 `GOTOOLCHAIN=auto` 时，Go 会自动下载足够新的工具链。把这个目录加入 `PATH`，再检查安装的模块身份：

```bash
pig version
go version -m "$(command -v pig)"
```

更新 Go 安装就是再次执行 `go install`，换用更新的确切版本或 `@latest`。PiG 不把 Go 安装的可执行文件当作独立发布的下载，所以 `pig update` 在没有匹配安装凭据时会拒绝替换它。

显式 `pig update` 的版本检查会在十秒预算内最多重试两次暂时性的传输失败和 HTTP 状态。启动检查不重试。签名或清单错误会终止更新。原生独立更新拒绝并发的同一个可执行文件替换，并持有安装锁直到凭据提交或回滚完成（D39）。`<executable>.update.lock` 伴随文件只在更新期间存在，更新结束时删除。

每个发布会在同一个提交上给根模块（`v0.5.0`）和嵌套的 Go 扩展 SDK 模块（`extensions/sdk/v0.5.0`）打标签；`go install` 两个都需要。

## 启动 PiG

在你希望它工作的项目目录里运行：

```bash
cd /path/to/project
pig
```

源码构建在 PiG 检出目录的 `bin/pig`。用完整路径运行，例如 `/path/to/PiG/bin/pig`，或者把那个目录加入 `PATH`。

PiG 把用户状态存放在 `~/.pig` 下。只有需要隔离配置根时才用 `PIG_HOME`。

在默认 Agent 目录（其中还没有 `settings.json`，每个全新的 `PIG_HOME` 都算）交互式启动时，PiG 会显示首次设置界面：主题、精灵和用量分析选项。按 `Esc` 跳过，不保存任何内容。自定义 Agent 目录（`PIG_CODING_AGENT_DIR`）以及 print、JSON、RPC 模式会跳过它（`coding/cli/extensions.go:93-95`）。Pi 1.4 只有在以下条件全部成立时才显示它的界面（`startup-ui.ts` 的 `shouldRunFirstTimeSetup`）：二进制是 Pi 自己的发行版（包 `@earendil-works/pi-coding-agent`、应用 `pi`、配置目录 `.pi`）、设置了 `PI_EXPERIMENTAL=1`、没有设置 `PI_CODING_AGENT_DIR`、且 `~/.pi/agent/settings.json` 尚不存在。这是差异 D88。之后用 `/settings` 改主题，用 `/sprite` 换精灵。[最初的十分钟](/docs/latest/first-run) 展示了这个界面。

## 配置 Provider 认证

列出当前二进制可用的登录目标：

```bash
pig login --list
```

发起可用的 OAuth 登录：

```bash
pig login github-copilot
```

使用 API key 的 Provider，按文档设置对应环境变量：

```bash
export OPENAI_API_KEY=...
pig --model openai/gpt-4.1
```

以 `pig --help` 和 `pig login --help` 作为当前二进制的权威说明。

## 开始第一次会话

输入请求并回车：

```text
Summarize this repository and tell me how to run its checks.
```

除非命令行、设置或活动 Piglet 收窄了范围，Stock PiG 会把常规内置编码工具交给模型。

## 使用 print 模式

不开交互式 TUI，只跑一次提示：

```bash
pig -p "Summarize this repository"
```

需要时显式指定模型：

```bash
pig --model github-copilot/gpt-5-mini -p "Find the test entry points"
```

## 运行 PiG Standard

PiG Standard 不在裸 Stock PiG 的启动路径里。要从源码检出显式选择：

```bash
pig --piglet piglets/standard/pig-standard.yaml
```

当前 Standard 组合通过普通扩展资源提供它自己的 PiG 登录、`/sprite`、`/runner` 和 `/pig-runner`。

## 运行自己的 Piglet

校验并运行一个 Agent 应用：

```bash
pig piglet validate ./agents/reviewer.yaml
pig --piglet ./agents/reviewer.yaml
```

想按名字调用时，先登记可移植的 Piglet 源：

```bash
pig piglet add ./agents/reviewer.yaml
pig --piglet reviewer
```

见 [Piglets](/docs/latest/piglets)。

## 构建 Piglet Binary

```bash
pig piglet build ./agents/reviewer.yaml \
  --format binary \
  --out ./pig-reviewer
./pig-reviewer
```

Piglet Binary 是某个固定 Piglet 组合的目标平台原生可执行文件。它仍可以有明确的外部依赖。构建需要 Go；使用容器构建器时需要 Docker 或 Podman。自己构建的 `pig` 还需要由 `PIG_SOURCE_ROOT` 指定的 PiG 源码检出；已发布的正式版本会获取自己的源码。见 [Piglet Binaries](/docs/latest/piglet-binaries)。

## 添加扩展

Go 扩展需要 Go 工具链。`pig setup` 会显示是否找到 `go`，`pig setup go` 负责安装。创建并校验一个 Go 工厂：

```bash
pig extension init ./review --lang go
pig install ./review --validate-only --json
```

只在一个会话里运行：

```bash
pig -e ./review
```

如果扩展属于某个有名字的 Agent 应用，用 Piglet。[第一个扩展](/docs/latest/first-extension) 逐步演示。

## 验证源码检出

准备精确的 Pi 对照程序和源码镜像：

```bash
make upstream-mirror
```

运行主门禁：

```bash
make check
```

使用仓库声明的工具链版本。构建通过不能替代对等审查、漏洞审查、制品清单或发布授权。

## 下一步

- [使用 PiG](/docs/latest/usage)
- [PiG 概念](/docs/latest/concepts)
- [扩展](zh-extensions.md)
- [Packages](zh-packages.md)
- [Piglets](/docs/latest/piglets)
- [构建派生的 harness](/docs/latest/derivative-harnesses)

# 命令行集成

在终端里不带参数运行 `pig` 会打开交互式终端界面。脚本和其他程序改用三种非交互模式之一：print、JSON 或 RPC。

所有模式共用同一套 Agent 循环、会话、工具和资源。模式决定输入怎样到达 PiG、PiG 往标准输出写什么、以及进程是否为了后续命令继续存活。选择模型、工具、资源和会话存储的选项在各模式下行为一致。见 [命令行](/docs/latest/cli)。

[Go SDK](zh-sdk.md) 不是一种 CLI 模式。它在 Go 程序内部运行会话，不需要另起进程。Pi 的 SDK 则是 TypeScript 库。

> 本页是英文页 `cli-integration.md` 的中文译本，对应 PiG 提交 `c64bebd8`。英文页是规范来源；行为与选项以英文页和当前二进制为准。

## 选择模式

| 模式 | 启动方式 | 输出 | 生命周期 | 适用场景 |
|---|---|---|---|---|
| 交互式 | `pig` | 终端界面 | 直到你退出 | 人直接和 PiG 工作 |
| Print | `pig -p "prompt"` | 最终回答的文本 | 一次调用 | 脚本只需要最终答案 |
| JSON | `pig --mode json "prompt"` | 事件，JSON Lines | 一次调用 | 程序需要一次运行的过程 |
| RPC | `pig --mode rpc` | 响应和事件，JSON Lines | 直到标准输入关闭 | 程序需要发送多条命令 |

标准输入或标准输出不是终端时，PiG 会自动选择 print 模式。所以管道或重定向就够了：

```bash
git diff | pig "Write a commit message for this diff" > message.txt
```

## Print 模式

Print 模式执行提示，把最终 assistant 消息的文本写到标准输出，然后退出：

```bash
pig -p "Summarize the changes in this repository"
```

`--print`（`-p`）不带值。PiG 按这个顺序组装提示：管道输入的标准输入、`@file` 参数、第一条消息参数。第一条跑完后，后续每条消息参数作为独立提示依次发送。

PiG 会一直读取管道输入直到结束，与 Pi 一致。调用方如果让管道保持打开，启动就会一直等。用 `--no-stdin` 跳过这次读取，例如 `pig --no-stdin -p "prompt"`。Pi 没有这个选项（D150，Pi #10415）。

标准输出不是终端时，PiG 把它留给答案，并把工具或扩展写到那里的内容重定向到标准错误。扩展错误以 `Extension error (<path>): <message>` 的形式出现在标准错误。

| 结果 | 退出状态 |
|---|---|
| 最终 assistant 消息正常完成 | `0` |
| 最终 assistant 消息的停止原因是 `error` 或 `aborted` | `1`，错误写到标准错误 |
| PiG 无法开始运行，例如没有配置模型 | `1` |
| `SIGINT`、`SIGTERM` 或 `SIGHUP` 终止运行 | `128` 加信号编号：`130`、`143` 或 `129` |

收到 `SIGTERM` 或 `SIGHUP` 时，PiG 会杀掉工具启动的子进程、停止写标准输出、向扩展发送 `session_shutdown`，并在所有 `session_shutdown` 处理函数结束后以 143 或 129 退出，与 Pi 一致。标准输出在收到信号时就结束。被中断的运行停在原处，同时 `session_shutdown` 处理函数继续执行；前提是它们不等待。处理函数一旦等待定时器或 I/O，运行就继续，与 Pi 一致：它的工具结果和下一轮仍可能写入会话文件和 Provider；文本模式下，处理函数结束前完成的运行会打印结果。

## JSON 模式

JSON 模式每行写一个 JSON 对象。第一行是会话头，之后是 Agent 和会话事件：

```bash
pig --mode json "Review this repository" > events.jsonl
```

输出是事件流，不是要求模型用 JSON 回答。

所有提示来自命令行和标准输入。进程流式输出这次运行的事件后退出，不接受后续命令。

失败或中止的 assistant 消息会出现在事件里，但不改变退出状态。需要判断成功时读 `message_end`。运行本身失败时 PiG 以状态 `1` 退出。

`message_update` 事件携带一个增量，不是到目前为止的完整消息。先用增量拼出实时输出，再在 `message_end` 里用完整消息替换。

`agent_end` 之后还可能有自动重试或排队工作。要知道运行确实没有后续工作，等 `agent_settled`；它的 `aborted` 字段区分取消和正常结束。

标准输出只有 JSON Lines。PiG 把诊断和日志写到标准错误。事件形状见 [JSON 事件流模式](/docs/latest/json)。

## RPC 模式

RPC 模式让 PiG 保持运行，由另一个程序发送命令：

```bash
pig --mode rpc --no-session
```

客户端向标准输入每行写一条 JSON 命令。PiG 向标准输出每行写一条 JSON 响应或事件。

给命令加 `id`，就能和响应配对；响应会带回同一个 `id`。事件没有 `id`，因为它们描述会话而不是某条命令。

`prompt` 的成功响应只表示 PiG 接受了提示，不表示运行结束。需要结果时继续读事件，直到 `agent_settled`。

命令可以切换模型、读取会话状态、管理会话、运行 shell 命令、回答扩展对话框。见 [RPC 命令](/docs/latest/rpc-commands)。扩展对话框走请求与应答，见 [RPC 扩展 UI](/docs/latest/rpc-extension-ui)。其他扩展 UI 记录是通知，客户端可以显示或忽略。需要终端界面的功能在 RPC 模式下不可用。

关闭标准输入即可停止 PiG。会替换会话的命令（`new_session`、`fork`、`clone`、`switch_session`）在标准输入结束时仍在运行，会先拿到响应再退出；Pi 不等这个响应就退出（D146）。帧格式和事件见 [RPC 模式](zh-rpc.md)。

### Go 客户端

Go 程序可以使用 `github.com/MichaelKinsy/PiG/coding/rpcclient`，它是 Pi 的 TypeScript `RpcClient` 的 Go 移植。它会启动 `pig --mode rpc`、把响应和命令配对、为每条命令提供一个方法，并把事件交给监听器：

```go
package main

import (
	"fmt"
	"log"
	"time"

	"github.com/MichaelKinsy/PiG/coding/rpcclient"
)

func main() {
	client := rpcclient.NewRpcClient(rpcclient.RpcClientOptions{
		CliPath: "pig",
		Model:   "openai/gpt-5",
		Args:    []string{"--no-session"},
	})
	if err := client.Start(); err != nil {
		log.Fatal(err)
	}
	defer func() {
		client.Stop()
		client.Wait()
	}()

	events, err := client.PromptAndWait("List the Go packages in this repository.", nil, 5*time.Minute)
	if err != nil {
		log.Fatalf("%v\n%s", err, client.GetStderr())
	}
	for _, event := range events {
		fmt.Println(event.Type)
	}
}
```

`PromptAndWait` 在发送提示之前就开始收集事件，所以运行再快也不会早于客户端开始监听。它返回直到并包含 `agent_settled` 的所有事件。需要分步处理时，在调用 `Prompt` 之前用 `OnEvent` 注册监听器，并且只在运行活动期间调用 `WaitForIdle`。

其他语言的程序直接使用 JSON Lines 协议。[RPC 模式](zh-rpc.md) 里有一个最小 Python 客户端。

## 在 PiG 上构建自己的 Agent

Pi 允许源码分支在 `package.json` 里用 `piConfig` 块改命令名和配置目录。PiG 没有这个块。命令名 `pig` 和目录 `~/.pig` 是二进制的一部分（D2）。

要发布一个有自己的名字、扩展和默认值的 Agent，写一个 [Piglet](/docs/latest/piglets)。用 `pig --piglet <name|path>` 运行，或者用 `pig piglet build --format binary` 构建成原生可执行文件。见 [派生 harness](/docs/latest/derivative-harnesses)。

## 相关

- [命令行](/docs/latest/cli) 列出命令行选项。
- [JSON 事件流模式](/docs/latest/json) 说明事件记录。
- [RPC 模式](zh-rpc.md) 说明 RPC 协议。
- [RPC 命令](/docs/latest/rpc-commands) 列出全部 RPC 命令。
- [Go SDK](zh-sdk.md) 在 Go 程序内部运行会话。
- [消息类型](/docs/latest/message-types) 定义事件里的消息。
- [上游 Pi 的命令行集成文档](https://pi.dev/docs/latest/cli-integration) 描述参考实现。

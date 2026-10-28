# Go SDK

PiG 把编码 Agent 以 Go 包的形式暴露出来。Go 应用需要创建和控制 PiG 会话、又不想把命令行程序作为子进程启动时，用 SDK。

调用方不是 Go 程序，或需要进程边界时，用 RPC 模式。

> 本页是英文页 `sdk.md` 的中文译本，对应 PiG 提交 `c64bebd8`。英文页是规范来源；函数签名与行为以英文页和当前源码为准。

## 模块

```text
github.com/MichaelKinsy/PiG
```

PiG 仍在积极开发。嵌入时请固定确切的发布版本或提交。首个公开版本之前不要假定 API 稳定。

仓库和版本标签公开后，可以使用 pkg.go.dev 上的生成文档：

- [`github.com/MichaelKinsy/PiG`](https://pkg.go.dev/github.com/MichaelKinsy/PiG)
- [`github.com/MichaelKinsy/PiG/agent`](https://pkg.go.dev/github.com/MichaelKinsy/PiG/agent)
- [`github.com/MichaelKinsy/PiG/ai`](https://pkg.go.dev/github.com/MichaelKinsy/PiG/ai)
- [`github.com/MichaelKinsy/PiG/coding`](https://pkg.go.dev/github.com/MichaelKinsy/PiG/coding)
- [`github.com/MichaelKinsy/PiG/tui`](https://pkg.go.dev/github.com/MichaelKinsy/PiG/tui)
- [`github.com/MichaelKinsy/PiG/extensions/sdk`](https://pkg.go.dev/github.com/MichaelKinsy/PiG/extensions/sdk)

扩展 SDK 是嵌套的 Go 模块。它的模块版本使用标签前缀 `extensions/sdk/`，命令和其他公开包使用根模块标签。

## 最小流程

程序通常：

1. 创建 `coding.AgentSessionServices` 容器；
2. 创建 `coding.Runtime`；
3. 解析模型；
4. 创建或打开 `coding.Session`；
5. 发送提示；
6. 消费返回的消息或会话事件；
7. 关闭 Session 和 Runtime。

```go
package main

import (
    "context"
    "log"

    "github.com/MichaelKinsy/PiG/coding"
)

func main() {
    services, err := coding.CreateAgentSessionServices(coding.CreateAgentSessionServicesOptions{
        CWD:      ".",
        AgentDir: coding.DefaultAgentDir(),
    })
    if err != nil {
        log.Fatal(err)
    }

    runtime, err := coding.NewRuntime(coding.RuntimeOptions{
        Services: services,
    })
    if err != nil {
        log.Fatal(err)
    }
    defer runtime.Close()

    model, err := coding.BuildModel("github-copilot/gpt-4o-mini", services)
    if err != nil {
        log.Fatal(err)
    }

    session, err := runtime.New(coding.SessionStartOptions{
        Model:        model,
        SystemPrompt: "You are a concise coding assistant.",
    })
    if err != nil {
        log.Fatal(err)
    }
    defer session.Close()

    _, err = session.Send(context.Background(), "Summarize this repository.")
    if err != nil {
        log.Fatal(err)
    }
}
```

仓库里有一个完整示例：`examples/sdk/main.go`。

### 完整示例

下面是最小流程加两点：打印每个会话事件类型，并用 `Session.LastAssistantText` 读取回复。它对着 `models.json` 里注册为 `mock` 的脚本化测试服务器运行，所以回复措辞只是示例；事件类型、顺序和消息数量是程序实际打印的。

```go
package main

import (
    "context"
    "fmt"
    "log"
    "os"

    "github.com/MichaelKinsy/PiG/agent"
    "github.com/MichaelKinsy/PiG/coding"
)

func main() {
    cwd, _ := os.MkdirTemp("", "sdk-demo-")
    defer os.RemoveAll(cwd)

    services, err := coding.CreateAgentSessionServices(coding.CreateAgentSessionServicesOptions{
        CWD:      cwd,
        AgentDir: coding.DefaultAgentDir(),
    })
    if err != nil {
        log.Fatal(err)
    }
    defer services.Close()

    runtime, err := coding.NewRuntime(coding.RuntimeOptions{Services: services})
    if err != nil {
        log.Fatal(err)
    }
    defer runtime.Close()

    model, err := coding.BuildModel("mock/mock-1", services)
    if err != nil {
        log.Fatal(err)
    }
    session, err := runtime.New(coding.SessionStartOptions{
        Model:        model,
        SystemPrompt: "You are a concise assistant.",
    })
    if err != nil {
        log.Fatal(err)
    }
    defer session.Close()

    unsubscribe := session.Subscribe(func(event agent.AgentEvent) {
        fmt.Printf("event: %T\n", event)
    })
    defer unsubscribe()

    messages, err := session.Send(context.Background(), "Say hello")
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println("messages returned:", len(messages))
    if text := session.LastAssistantText(); text != nil {
        fmt.Println("reply:", *text)
    }
}
```

输出：

```text
event: agent.AgentStartEvent
event: agent.TurnStartEvent
event: agent.MessageStartEvent
event: agent.MessageEndEvent
event: agent.MessageStartEvent
event: agent.MessageEndEvent
event: agent.MessageStartEvent
event: agent.MessageUpdateEvent
event: agent.MessageUpdateEvent
event: agent.MessageUpdateEvent
event: agent.MessageEndEvent
event: agent.TurnEndEvent
event: agent.AgentEndEvent
event: agent.AgentSettledEvent
messages returned: 3
reply: Hello from the test model.
```

三对 message start/end 分别是系统消息、你的提示和回复；三行 `MessageUpdateEvent` 是回复的流式增量。`Send` 阻塞到 `AgentSettledEvent`。监听器是同步的：它在 Session 发布事件时运行（`coding/session.go:2172-2175`），所以要写短。`Send` 返回 3 条消息（system、user、assistant）。[RPC 模式](zh-rpc.md) 里的 `get_session_stats` 同样把这三条算作 `totalMessages`。

这个例子里有两处容易踩：

1. **Provider 失败时 `Send` 不返回错误。** 缺少凭据时运行以 `err == nil` 结束，失败在返回的 assistant 消息里。检查它的 `StopReason` 和 `ErrorMessage`。没有该 Provider 凭据时，打印这两个字段的变体显示 `stop="error" err="provider is not configured: mock"`，`LastAssistantText` 返回 `nil`，所以没有打印 `reply:` 行。Pi 行为相同：它的 prompt 调用正常返回，错误在 assistant 消息上。
2. **`coding.DefaultAgentDir()` 是 `pig` 命令使用的目录，因此 `PIG_CODING_AGENT_DIR` 优先。** 它返回 `PIG_CODING_AGENT_DIR` 的值（展开开头的 `~`），否则返回 `<root>/agent`，其中 `<root>` 是[配置根](/docs/latest/configuration)：`$PIG_HOME`，否则 `$XDG_CONFIG_HOME/pig`，否则 `~/.pig`（`internal/codingagent/settings.go:1770-1785`、`internal/configroot/configroot.go:50-66`）。设置 `PIG_USE_PI_DIRS=1` 时返回 `PI_CODING_AGENT_DIR`，否则 `~/.pi/agent`。这是 Pi 的 `getAgentDir()`（`.upstream/current/packages/coding-agent/src/config.ts:605-611`）配上 PiG 的目录名：Pi 只读 `PI_CODING_AGENT_DIR`，否则 `~/.pi/agent`。这是差异 D2。在具备该行为的构建上核对过（提交 `086395a8bd`）：把 `PIG_CODING_AGENT_DIR` 指向存放 `models.json` 的文件夹、`PIG_HOME` 指向另一个空文件夹时，先打印 `DefaultAgentDir:` 的变体输出的是 `PIG_CODING_AGENT_DIR` 文件夹，然后打印 `reply: Hello from the test model.`。没有 `PIG_CODING_AGENT_DIR` 时它打印 `<PIG_HOME>/agent`，在那里找不到 `models.json`，也没有回复。请确认拿到的是哪个目录：`fmt.Println(coding.DefaultAgentDir())`。该提交之前的构建返回 `<PIG_HOME>/agent` 并忽略 `PIG_CODING_AGENT_DIR`（`pig` 自己会读它），所以只设置该变量启动的程序读到的 `models.json` 和 `auth.json` 与 `pig` 不同，会像第 1 点那样失败。面对这种构建，把 `AgentDir` 从 `CreateAgentSessionServicesOptions` 里去掉：`CreateAgentSessionServices` 会像 `pig` 命令那样解析目录（`coding/services.go:161-166`），正如 Pi 的 `createAgentSessionServices` 回退到 `getAgentDir()`（`.upstream/current/packages/coding-agent/src/core/agent-session-services.ts:139`）。设置 `PIG_HOME`，或自己在 `AgentDir` 里传目录，也可以。

## Services

`coding.AgentSessionServices` 拥有设置、认证、模型注册表状态和路径等共享协作者。把它传给 Runtime，而不是在内部创建隐藏的全局依赖。

应用不使用命令行默认值时，请显式设置工作目录和 Agent 目录。

常规 Agent 目录是：

```text
~/.pig/agent
```

隔离测试和应用请用 `PIG_HOME` 或显式选项。

用 `coding.NewInMemorySettingsManager(initialSettings)` 在不做文件 I/O 的情况下保存设置，并通过 `CreateAgentSessionServicesOptions.SettingsManager` 传入。Services 容器保留它，Session 的设置改动在 `Reload` 后仍留在内存里。显式的 `CreateAgentSessionServicesOptions.ProjectTrusted` 值作用于传入的管理器；否则其信任状态不变。文件支持的设置仍是默认。重载会丢弃临时覆盖，显式的空工具和资源列表仍为空。

`SettingsManager.GetThemeSetting()` 返回 `*string`：nil 表示省略，指向空字符串的指针表示显式空名称。JSON 加载、克隆、项目合并和 `SetTheme("")` 都保留这个区别。`GetTheme()` 返回固定主题名，省略或斜杠分隔的自动设置返回空字符串。

## Runtime

`coding.Runtime` 拥有扩展执行并创建 Session。

所有 Session 结束后关闭 Runtime。Runtime 可能拥有扩展进程和取消状态，丢弃它可能泄漏资源。

### 工厂拥有的替换

嵌入的 Go 应用必须替换当前 Session 时，用 `coding.CreateAgentSessionRuntime`。提供一个 `CreateAgentSessionRuntimeFactory`，构造目标 CWD 的 Services、资源、工具和新的扩展 runner，再在传入的 `SessionManager` 上创建 Session。把该 Session 及其拥有的 Services 放在 `CreateAgentSessionRuntimeResult` 里返回。

通过 `Runtime.Session()` 读取当前 Session。`Runtime.NewSession`、`Runtime.SwitchSession`、`Runtime.Fork` 和 `Runtime.ImportFromJsonl` 使用保留的工厂。替换会先让离开的响应安定下来，再关闭，并在安装下一个 Session 之前使其 runner 失效。被取消的 before-switch 或 before-fork 处理器会让当前 Session 留在原处。导入时若原文件名已存在，会选择带后缀的文件名。

安装 `Runtime.SetRebindSession` 来订阅新 Session 的事件并绑定其扩展。Runtime 在返回前等待该回调。`Session.BindExtensions` 在绑定后按配置的 `session_start` 原因派发。关闭 Runtime 以释放它当前的 Session。

在 UI 宿主里，渲染替换后的 Session 并挂上事件消费者，要在绑定扩展之前完成。`session_start` 处理器可能在那次绑定中通知或发送消息。请在等待之前捕获 Session。若等待绑定期间另一次替换胜出，不要让过期的完成回调挂上消费者或更新终端标题。

Go 原生替换选项接受 `WithSession func(*extension.ReplacedSessionContext) error`。Runtime 在重新绑定后调用它，并把错误返回给调用方。使用提供的上下文，而不是保留的旧命令上下文。它的 `SendMessage` 和 `SendUserMessage` 会等待 Session 操作。用户消息的命令/模板展开默认关闭。`NewSessionOptions.Setup` 在重新绑定前运行，并从初始化后的日志刷新 Session。

`pig` CLI 在 print、JSON、RPC 和交互模式下都通过这条路径创建和替换 Session。它的工厂会为每次替换重建目标工作目录的 Services、资源和扩展宿主。所以 `Session` 的 `NewSession`、`SwitchSession`、`CloneInPlace`、`ForkToNewSession`、`ForkToNewSessionWithText`、无界面的 `Session.DispatchSlash` `/fork` 路径和 `ImportFromJsonl`，以及默认的 `Session.ExtensionCommandActions` 的 new、fork、switch 动作，都不使用工厂。它们保留宿主作用域的 runner 和启动项目的 Services，因此 D30（第 2 部分）和 D61 仍适用于调用它们的代码。D30（第 1 部分）也适用于 CLI 模式下的子进程扩展：替换后其宿主用 Pi 的过期消息拒绝后续调用，但 SDK 本地 getter 返回离开的 Session 的值，宿主调用不在调用点抛出（按 SDK 不同，返回错误、抛出异常或写标准错误）。子进程扩展 SDK 尚未暴露 `withSession` 回调。见[差异记录](https://github.com/MichaelKinsy/PiG/blob/main/docs/parity/DIVERGENCES.md)。

## 会话

用 `Runtime.New` 创建新 Session。打开已有 Session 用 Runtime 中适合保存路径的方法。

`Runtime.Open` 校验并拥有一个文件快照。Session 构造使用该快照，而不是再次打开文件。恢复模型和思考级别只读分支设置，不投影消息内容；记录投影另行运行。

`Session.Send` 阻塞到 Agent 工具循环结束或上下文被取消。不要为了看起来异步而把它放进无人拥有的 goroutine。

`Session.Prompt(ctx, text, options)` 只返回错误，正如 Pi 的 `prompt()` 返回 `void`；消息从 Session 事件或 `Session.Send` 读取。`PromptOptions` 控制调用时的图片、输入来源、流式投递和命令/模板展开，不替代系统提示构造选项。`Session.SendUserMessage(ctx, content, options)` 等待一个由扩展发起的 user 轮次，展开默认关闭。`Session.SendCustomMessage(ctx, message, options)` 等待自定义投递及可能触发的轮次。绑定的扩展动作使用 `SendExtensionUserMessage` 和 `SendMessage`；Session 拥有它们的后台续体。nil 的 `TriggerTurn` 要与显式 false 分开保留。

`Session.Compact(ctx, customInstructions)` 返回 `(*CompactionResult, error)`，并拒绝没有可压缩内容的会话，正如 Pi 的 `compact()`。`Session.ExecuteBash(ctx, command, onChunk, *ExecuteBashOptions)` 运行用户 shell 命令。`Session.Dispose()` 按 Pi 的 `dispose()` 释放 Session：事件流关闭，模型注册表不再持有它。`agent_settled` 扩展处理器追加自定义消息时，其消息事件在 Session 监听器和模式输出上都先于最终安定事件。

应用需要流式更新时，请在发送之前订阅 Session 事件。

### 提供 SessionManager

`coding.SessionManager` 是真正的追加日志，不是消息快照。用 `coding.NewInMemorySessionManager(cwd)` 创建内存管理器。它提供项目 CWD 默认值时，传给 `CreateAgentSessionServicesOptions.SessionManager`；同时把同一个指针传给 `SessionStartOptions.SessionManager` 或 `SessionOptions.SessionManager` 以选择底层日志。

```go
manager, err := coding.NewInMemorySessionManager(".")
if err != nil {
    return err
}
services, err := coding.CreateAgentSessionServices(coding.CreateAgentSessionServicesOptions{
    AgentDir:       agentDir,
    SessionManager: manager,
})
if err != nil {
    return err
}
runtime, err := coding.NewRuntime(coding.RuntimeOptions{Services: services})
if err != nil {
    return err
}
defer runtime.Close()
session, err := runtime.New(coding.SessionStartOptions{
    Model:          model,
    SessionManager: manager,
    ThinkingLevel:  ai.ThinkingHigh,
})
if err != nil {
    return err
}
defer session.Close()
// session.SessionManager() == manager
```

显式的 Services CWD 优先于管理器的 CWD。没有管理器覆盖时，新的持久化 Session 使用所选 Services 的 Agent 目录。`GetSessionFile` 在内存会话里返回 nil。持久化管理器在第一条 user 或 assistant 消息创建文件之前就选定路径，所以 `IsPersisted` 不表示文件已存在。Fork 和克隆保留源持久化模式。没有 user 或 assistant 消息的保留分支会推迟文件写入。工厂拥有的 Runtime fork 替换 Session，但保留内存 SessionManager 对象。

直接调用 Session 的工具会拿到它当前的标识、模型和思考状态。带显式 ToolEnvironment 的工具调用保留该上下文。克隆各自保留上下文绑定。

### 提供 Agent 或缓存预热器

`SessionOptions.Agent` 是你构建的 `*agent.Agent`。Session 驱动它，而不是自己构建，正如 Pi 的 `AgentSessionConfig.agent`（`agent-session.ts:253,486`）。Session 把钩子安装进去：它串联 Agent 已有的 `FinishTurn`、`PrepareRequest`、`PrepareNextTurn` 和 `TransformLLMMessages`，替换 `BeforeToolCall`、`AfterToolCall`、`PreparePrompt`、`PrepareToolResult` 和 `SessionFile`，并设置工具与会话 ID。Agent 其余配置由你拥有：流函数、`TransformContext`、`OnPayload` 和头钩子（因此除非你的钩子发出，否则不会有 `before_provider_request` 或 `before_provider_headers` 事件）、steering 与 follow-up 模式、思考预算、传输和图片转换。nil 的 `Model` 和 `ThinkingLevel` 采用 Agent 的值，所以 Session 不从历史恢复它们。一个 Agent 服务一个 Session：`Clone` 和替换会构建自己的。不提供时，Session 用缓存预热流函数构建自己的 Agent。Pi 的 `extensionRunnerRef` 单元在 PiG 中不存在（D113）：构造后读 `Session.ExtensionRunner()`。

`SessionOptions.CacheWarmer` 是你构建的 `coding.CacheWarmer`，正如 Pi 的 `AgentSessionConfig.cacheWarmer`（`agent-session.ts:266,494-497`）。Session 把它的 `OnWarmed` 接到 `entry_appended` 事件、报告 `Status`、转发 `OnAgentSettled` 和 `OnModeChanged`，并在关闭时取消它。你的流函数负责启动它。Session 替换会为新 Session 构建自己的预热器。

## 流观察

流事件交付的部分 assistant 消息，其导出字段等于消费者收到那一刻的流状态。直接读 `event.Partial` 即可，不需要特殊调用。字段之后不再变化，所以保留的部分消息读起来仍像交付时那样。要读取你保留的部分消息的生产者当前状态，调用 `partial.Observe()` 或 `ai.RefreshEvent(event)`；两者都返回独立拥有的数据。保留事件并在 `await` 之后读取的 JavaScript 监听器会看到生产者的后续变化；保留事件的 Go 监听器必须主动索取。Agent 监听器和 Session 监听器收到的是各自派发时刷新过的消息，所以直接读取与 Pi 监听器在该时刻读到的一致（`agent.RefreshEvent`）。扩展进程收到派发时的 JSON 快照（D82）。

Agent 的 `message_start` 和 `message_update` 消息保留 Pi 的浅拷贝语义：复制的顶层字段固定，引用的嵌套内容可以前进。例如，保留的 start 消息可以获得最终内容，而它复制的停止原因仍是 `pending`。最终消息不含临时 Provider 解析器字段。

`AssistantMessageEventStream.Result` 独立于事件迭代完成，并保留终态消息的身份。消费者需要再执行一次异步等待时，用 `ObserveEvents`。它为每个事件提供一个 `StreamObservation` 作用域。先同步准备回调要读的值，再用 `scope.Await` 做阻塞等待。已完成的操作用 `scope.Yield`，其续体仍需排队。不要在不释放作用域的情况下让被观察的回调阻塞在无关 channel 上。

需要固定大小的投影时，`partial.ObserveUsage()` 只复制当前用量，`partial.ObserveToolCallIdentity(index)` 只读取所选块的 ID、名称和 index/type 有效性。这些方法保留相同的完整或浅引用语义，不遍历内容参数或元数据。Agent 的 assistant 消息也提供 `ObserveUsage()`，保留“用量缺失”这一状态。

Agent 把流创建和迭代器接管放在同一个续体上。直接的流消费者可以在发布请求上下文之前调用 `ai.WithStreamContinuations(ctx)`，再用 `ai.RunStreamContinuation` 包住流创建和迭代。把该共享上下文传给生产者。把 `scope.Context(ctx)` 传给 `ObserveEvents`，让迭代器复用调用方的续体。这可以防止生产者的反应在调用方首次等待之前打断同步设置。`ai.StreamObservationFromContext` 取到活动作用域，而不替换 Agent 的共享中止信号对象。

## 模型与认证

`coding.BuildModel` 通过 Services 容器解析 Provider 和模型。创建模型之前，先用独立的 `pig` 命令认证，或提供应用自己的凭据机制。

从命令行验证模型：

```bash
pig --print --model github-copilot/gpt-4o-mini "hello"
```

不要把凭据复制进源码。

不创建 Session、只做模型查找和目录刷新时，用 `coding.CreateModelRuntime(ctx, options)`。`CreateModelRuntimeOptions` 选择 `AuthPath`、`ModelsPath`、`ModelsStore` 和 `ModelsStorePath`。除非 `AllowModelNetwork` 为 true，初始刷新从缓存目录恢复，不访问网络。把 `RefreshOnCreate` 设成指向 false 的指针可跳过该刷新。`ModelRefreshTimeoutMs` 限制启用的初始网络刷新。调用方上下文也会取消初始刷新。

`ModelsPath` 保留三种状态：nil 选择默认 `models.json`；指向 nil 的指针禁用文件；指向字符串指针的指针选择该路径。默认目录存储位于所选模型文件旁边。禁用文件时选择内存目录存储，除非提供了 `ModelsStore`。初始 Provider 错误或取消不会丢弃 runtime；调用方需要刷新状态时，检查显式 `Refresh` 调用的结果。

`ModelRuntime.Login` 和 `ModelRuntime.Logout` 在同一 Provider 上串行经过凭据持久化和本地目录/认证同步。不同 Provider 可以并发认证。排队中的操作可以被取消而不运行。凭据改动提交后，`*coding.CredentialSynchronizationError` 报告本地同步失败，但不表示凭据改动已回滚。决定如何恢复前，检查 `ProviderID`、`Operation`、`Credential` 和 `errors.Unwrap(err)`。本地同步不通过网络获取模型。

`ModelRegistry.GetAPIKeyAndHeaders` 和扩展模型认证查询对内置与配置模型共用同一套请求认证解析。保留返回 `Headers` 里的 nil 值；它们会移除 Provider 默认头。把返回的 `Env` 随兼容请求一起传，使凭据作用域的端点设置保持可用。模型运行时解析一次请求凭据，并在流式请求前与当前模型元数据合并。

`ai.RuntimeCredentials` 在自身的 `Read`、`List` 或 `Delete` 中止检查拒绝操作时，保留调用方提供的取消原因。删除失败或被取消会保留运行时 key。

文件支持的凭据 `Modify` 和 `Delete` 在同一存储上排在另一操作之后时可以取消。取消排队调用不会释放活动回调持有的文件锁。

需要确定性的 Provider 测试时，`ai.NewFauxProvider` 通过正常事件生命周期流式发送每个排队响应。空响应也会在终态事件前发出 `start`。排队的 `error` 和 `aborted` 响应发出错误事件，并保留提供的 reason、错误信息和时间戳。

使用可 JSON 序列化的模型元数据。`SamplingParams` 里的循环 map 或 slice 引用会让 JSON 序列化失败，而不是造成无界的模型投影复制。返回的模型元数据当作只读。

### 通过自己的客户端发送 Anthropic 请求

`ai.StreamOptions.AnthropicClient` 替换 Anthropic Messages Provider 的客户端构造，正如 Pi 的 `AnthropicOptions.client`（`anthropic-messages.ts:286-290`、`607-609`）。Provider 通过它发送请求，并跳过联邦、OAuth 和 API-key 头组装。其他 Provider 忽略它。适用于共享 Messages API 的替代客户端，例如 Vertex 上的 Anthropic。

`ai.AnthropicClient` 只有一个方法：`CreateMessage(ctx, ai.AnthropicCreateParams, ai.AnthropicRequestOptions) (*http.Response, error)`。`ctx` 携带请求的中止。Provider 从响应体读取流事件，并自行处理非成功状态。实现了 `ProviderRequestError` 的可重试失败会在 `StreamOptions.MaxRetries` 内重试。`AnthropicCreateParams.Body` 是不含 betas 列表的 JSON 请求体，`Betas` 是逗号连接的 `anthropic-beta` 头值（nil 不发该头）。`AnthropicRequestOptions.TimeoutMs` 就是 `StreamOptions.TimeoutMs`。

## 模型作用域

设置 `SessionOptions.ScopedModels` 或对应的 Runtime 会话启动选项，提供有序的循环作用域。`coding.ScopedModel` 是共享的原生类型：含一个 `*ai.Model` 和可选的思考级别偏好。空作用域允许所有可用模型。

`Session.ScopedModels()` 和原生扩展 Context getter 保留作用域列表和模型身份。返回的列表和模型当作只读。用 `SetScopedModels` 发布替换。这改变仅限会话的作用域，不改变配置的已启用模型列表。子进程 SDK 收到模型元数据值，而不是 Go 指针。

## 工具

通过 `coding.SessionStartOptions` 传入调用方拥有的工具。使用公开的 `agent.AgentTool` 契约。

程序化 Session 不需要自动带上交互式 CLI 的每个工具。只选属于该应用的工具。

最终工具输出放在 `agent.AgentToolResult.Content`，一个有序的 `[]ai.ToolResultMessageContent` 切片。按模型应收到的顺序使用 `ai.TextContent` 和 `ai.ImageContent`：

```go
result := agent.AgentToolResult{
    Content: []ai.ToolResultMessageContent{
        ai.TextContent{Text: "before"},
        ai.ImageContent{Data: imageBase64, MimeType: "image/png"},
        ai.TextContent{Text: "after"},
    },
}
```

没有内容时用 `[]ai.ToolResultMessageContent{}`；显式空文本块用 `[]ai.ToolResultMessageContent{ai.TextContent{Text: ""}}`。`result.Text()` 用换行连接文本块，`result.Images()` 选出图片块。这些展示投影不改变有序内容。非 nil 的 `agent.AfterToolCallResult.Content` 替换整个内容数组；空切片清空它，nil 保持不变。

## 扩展

应用拥有扩展配置时，通过 `coding.RuntimeOptions` 传入扩展定义。命令行和嵌入式使用共享同一套公开扩展行为。

`coding.DiscoverAndLoadExtensions(ctx, configuredPaths, cwd, agentDir)` 按 Pi 的 `discoverAndLoadExtensions` 顺序和入口规则，发现项目、全局和配置位置里的扩展，并在扩展宿主中加载。对返回的 `LoadExtensionsResult` 调用 `Close()` 以关闭宿主。Pi 的函数接收事件总线，PiG 的不接收（D117）。

如果组合还要能从 `pig` 可执行文件使用，或构建成 Piglet Binary，请用 Piglet。

## 无界面嵌入与二进制体积

不带终端嵌入 PiG 的程序仍会链接会话 API 能到达的所有东西。`examples/headless` 是一个完整的小程序：会话 API、来自 `mcp.json` 的 MCP 服务器和一个 Provider（Anthropic）。它没有终端，也没有扩展文件。按 PiG 构建发布的方式构建：

```text
CGO_ENABLED=0 go build -trimpath -ldflags "-s -w" -o headless ./examples/headless
```

在 linux/amd64 上是 40.1 MB，而 `pig` 命令是 69.8 MB。有三样东西无需选项就已经排除：终端 MCP 视图和工具渲染器（`mcptui` 包，见下）、带词典的终端编辑器（6.3 MB）、交互模式的公告图片（0.5 MB）。其余功能可以用构建标签移除。

### 裁剪标签

每个可裁掉的功能都有构建标签 `pig_strip_<id>`。[Piglet Binary](piglet-binaries.md) 使用同一套标签，所以一个机制覆盖两者。把不需要的传给 `go build -tags`。被裁掉的功能不链接，并表现为不存在：请求它会报告该功能已从此构建中裁掉，模型列表也会略去被裁 API 的模型。

| 标签 | 从二进制中移除 | 在 `examples/headless`（linux/amd64）上节省 |
| --- | --- | ---: |
| `pig_strip_node_extensions` | 运行 TypeScript/JavaScript 扩展的内嵌 Node 运行时 | 5.07 MB |
| `pig_strip_bedrock_converse_stream` | Amazon Bedrock API 与 AWS SDK | 3.29 MB |
| `pig_strip_syntax_highlight` | 代码高亮语法 | 0.63 MB |
| `pig_strip_google_vertex` | Google Vertex API 及其 OAuth 代码 | 0.46 MB |
| `pig_strip_mistral_conversations` | Mistral conversations API | 0.10 MB |
| `pig_strip_mermaid` | Mermaid 图表绘制 | 0.07 MB |
| `pig_strip_word_dictionaries` | 中日韩、泰、老、高棉、缅甸语词典 | 0（本来就未包含；在 `pig` 命令里是 6.3 MB） |

其余标签（`pig_strip_llama_cpp`、`pig_strip_codemode`、`pig_strip_tool_search`、`pig_strip_pig_login`、`pig_strip_docs`、`pig_strip_changelog`、`pig_strip_export_html`、`pig_strip_self_update`、`pig_strip_piglet_builder`、`pig_strip_extension_sdk_go`、`pig_strip_extension_sdk_python`、`pig_strip_extension_sdk_rust`）移除的是会话 API 本身不会链接的功能。它们在你同时链接 `pig` 命令的包时才有意义，在这里无害。除 `pig_strip_mcp` 外全部传入，`examples/headless` 在 linux/amd64 上是 30.4 MB，darwin/arm64 是 29.0 MB，windows/amd64 是 31.7 MB。

注意：

- 不要传 `pig_strip_mcp`。它会把 MCP 从 `pig` 命令的内置扩展列表移除。自己调用 `mcpext` 的程序（如 `examples/headless`）需要该包。
- `pig_strip_node_extensions` 意味着程序无法加载 TypeScript/JavaScript 扩展文件。不加载任何扩展时设置 `DefaultResourceLoaderOptions.NoExtensions`，示例就是这么做的。
- 没有“保留一个 Provider、移除其余”的标签。上面的标签移除的是自带大依赖的整块 API（Bedrock、Vertex、Mistral）。其他 Provider 共用一条代码路径，会保持链接。单个 Provider 的代码相对这些依赖很小。
- 标签作用于整个程序及其链接的每个包。用 `go build -tags` 和文件大小检查结果。

### 没有终端的 MCP

`mcpext.CreateMcpExtension(mcpext.Options{...})` 是无界面入口。把 `Options.Terminal` 留为 nil：扩展就没有 `/mcp` 管理视图和工具渲染器，`mcptui` 包也不会被链接。`pig` 命令把 `Options.Terminal` 设为 `mcptui.Terminal()`。`mcpext.NewHeadless` 为从自己的 `Host` 构建扩展的调用方做同样的事。

### 扩展视图里的编辑器

扩展可以在自己绘制的视图里放编辑器。构建它的组件使用终端编辑器，会拉入词典。`pig` 命令在创建扩展宿主处调用 `subprocess.EnableEditorViews()` 启用它。从不绘制扩展编辑器的程序不调用它，编辑器也就不会被链接。把扩展视图服务到自己终端的程序，在构建第一个视图之前调用一次 `subprocess.EnableEditorViews()`。在此之前，带编辑器的视图会失败，错误会点名该调用；无效的编辑器节点仍像以前那样被拒绝。

### 体积门禁

`TestHeadlessEmbedBinarySizeGate` 为每个发布平台构建 `examples/headless`（无标签，以及除 `pig_strip_mcp` 外的全部裁剪标签），二进制超过 `automation/perf/ratchet.toml` 的上限（`binary.headless.*`）时失败。上限只能下降。它与约束 `pig` 命令体积（`binary.pig.*`）的是同一个门禁。

## 0.5.0 的 Go API 变更

这些变更会破坏为 0.4.x 编写的 Go 代码：

- `agent.NewAgent(opts)` 返回 `(*Agent, error)`，正如 Pi 的 `Agent` 构造函数会抛出。没有 `StreamFn`、`DefaultStreamFn` 或已配置默认值时错误为 `agent.ErrNoDefaultStreamFunction`。模型的 `Provider` 不再代替流函数。`Agent.Continue(ctx)` 只返回 `error`。
- `Session.Prompt` 只返回错误，`Session.Compact` 返回 `(*CompactionResult, error)`，`Session.ExecuteBash` 取代了 `ExecuteBash`、`ExecuteBashWithUpdates` 和 `ExecuteBashWithOperations`。
- `coding.SessionEntry` 和 `extension.SessionEntry` 是 Pi 的 `SessionEntry` 联合：成员为 Pi `session-manager.ts` 条目类型的接口。用 `entry.Base()` 读 `type`、`id`、`parentId` 和 `timestamp`，用 `entry.Raw()` 读 JSON。没有成员能描述的记录是 `RawEntry`。
- 类型采用 Pi 的名字：`InteractiveOptions` 改为 `InteractiveModeOptions`，`StatusLine` 改为 `FooterComponent`，`Services` 改为 `coding.AgentSessionServices`，`NewServices` 改为 `coding.CreateAgentSessionServices`，`ServicesOptions` 改为 `coding.CreateAgentSessionServicesOptions`，`CLIFlags` 改为 `Args`（其 `Args` 字段是 `Messages`）。
- `SettingsManager.GetProviderRetrySettings` 的 `TimeoutMs` 和 `MaxRetries` 返回 `*int`（未设置时为 nil），`RetryConfig.MaxDelayMs` 改为 `MaxAgentDelayMs`。

`agent` 还导出 Pi 的独立循环：`RunAgentLoop`、`RunAgentLoopContinue`、`AgentLoop`、`AgentLoopContinue`、`AgentContext`、`AgentLoopConfig`、`AgentEventSink` 和 `AgentEventStream`。`AgentLoopConfig`、`AgentOptions` 和 `Agent` 接受 `GetAPIKey` 解析器，它在每次请求前运行，正如 Pi 的 `getApiKey`。

## 取消与关闭

给会话操作传 `context.Context`。调用操作结束时取消它。

低层 `harness.WithAbortSignal(parent, signal)` 保留父上下文的值。父上下文无法取消时，派生上下文保留所给信号的 `Done` channel。两者都能取消时，第一个取消决定派生上下文的原因。

按这个顺序关闭资源：

1. 结束或取消活动的 Session 操作；
2. 关闭每个 Session；
3. 关闭 Runtime；
4. 释放应用拥有的 Services 依赖。

除非应用知道操作是幂等的，否则不要自动重放失败的扩展操作。

### 像 Pi 一样打印和取消的上下文

`github.com/MichaelKinsy/PiG/chord/context`（包 `chordctx`）把 Pi 的 `@earendil-works/chord/context` 移植到 `context.Context`。0.5.0 之前它在 `internal/chord/chordctx`。中止信号就是取消，中止原因就是取消原因。`Background()` 和 `TODO()` 对应 Pi 的 `BACKGROUND_CONTEXT` 和 `TODO_CONTEXT`，由它们派生的上下文打印方式与 Pi 相同。`NewKey` 从只在诊断中命名的描述创建带类型 key。`WithValue` 和 `Value` 存取值，父上下文不变。`WithAbortSignal(parent, signal)` 由父上下文或信号中先到者取消，并带其原因。`WithoutAbortSignal` 保留所有值并忽略调用方取消，仅用于必须执行的清理。`WithCancel` 返回子上下文和 `cancel(cause)` 函数。`Await[T](ctx, work)` 返回工作的结果，或仅从此等待者取得的取消原因：它不取消工作。`Telemetry(ctx)` 是 Pi 的 `Context.telemetry`：显式的遥测父级，永不为 nil，对 `Background()`、`TODO()` 和任何不带遥测的上下文是 `telemetry.NoopTelemetryContext`。`WithTelemetry(parent, telemetry)` 派生带遥测父级的上下文，不改变父级、其取消或值。`TraceIn[T](ctx, options, fn)` 在 `Telemetry(ctx)` 下开一个子 span 并调用 `fn(child)`，其中 `child` 从 `ctx` 派生，`Telemetry(child)` 就是该 span，所以 span 通过向下传递的上下文嵌套，没有环境当前 span。`fn` 的结果和错误原样传回。Pi 的回调接收一个把 `telemetry` 成员收窄的上下文；Go 不能收窄上下文类型，所以 span 作为第二个参数。这些辅助函数不插桩 Chord 宿主、方法或网络帧。

## Harness 事件

低层 harness 恢复按源码顺序读取缓存的控制值。它先读活动操作的元数据再读其状态，恢复期间不写入。

失败的 drive 在观察者收到 harness 故障之前清除其已完成的 pass。关闭 harness 仍会立即拒绝观察未完成的效果；该效果在返回前保留 drive 所有权。

低层 harness 事件批次在绑定接收者之前克隆每个载荷。函数值会引发 `agentharness.DataCloneError`。被拒绝的批次不投递事件，也不阻止后续有效批次。已提交的 lane 命令在事件克隆失败时保留其持久和内存状态。

## RPC 替代方案

调用方需要语言中立的进程边界时，用 RPC 模式启动 PiG：

```bash
pig --mode rpc
```

见 [RPC 模式](zh-rpc.md)。

## TypeScript 扩展声明

Node 扩展使用 `@earendil-works/pi-coding-agent` 的 TypeScript API。PiG 启动扩展时把这些 import 映射到兼容的运行时模块。

源码树包含 `extensions/sdk-ts`，一个仅声明的包，固定确切的 Pi 版本，并为 `ui.setLogin` 等 PiG 专有 API 增加类型。它不包含第二套 TypeScript 运行时，也不发布到 npm；请从源码树安装。

## Pi 参考

npm 包 `@earendil-works/pi-coding-agent` 是 Pi 的 TypeScript SDK。PiG 的 Go SDK 和非 TypeScript 桥接不替代它。

参考实现见 [上游 Pi 的 SDK 文档](https://pi.dev/docs/latest/sdk)。

## 相关文档

- [扩展](zh-extensions.md)
- [Piglets](/docs/latest/piglets)
- [派生 harness](/docs/latest/derivative-harnesses)
- [RPC 模式](zh-rpc.md)

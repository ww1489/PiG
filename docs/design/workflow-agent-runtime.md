# 把 PiG 嵌入 Workflow Agent 节点（进程内扩展版）

本文记录一个嵌入式用法的设计结论：把 PiG 作为 workflow 中单个 agent 节点的执行器，节点绑定自己的工具、提示词和 skills，由宿主实施计费与工具权限。扩展全部编译进服务进程，不使用运行时加载方式。

## 结论

- **PiG 只负责“一个 agent 节点”**。节点状态机、步骤推进、重试、预算、租户、持久化属于 workflow 引擎，不属于 PiG。
- **一个 worker 共享 `Services` 和 `Runtime`，每个节点尝试创建一个新 `Session`**。不要每个节点重建 `Services`/`Runtime`，也不要并发复用同一个 `Session`。
- **硬策略放在宿主进程**：`BeforeToolCall` 阻断工具，`Session.Subscribe` 计量用量，`RequestAbort` 终止运行。计费和权限不能只依赖扩展。
- **扩展全部编译进服务进程**：使用 `extension.ExtensionFactory` 注册能力，服务是单二进制、单版本，扩展随服务一起发布。

## 三层职责

| 层 | 负责 | 不负责 |
|---|---|---|
| Workflow 引擎 | 节点状态机、步骤、重试、超时、租户、预算、持久化、审计 | 模型调用和工具循环 |
| Go SDK 宿主 | 创建 `Services` / `Runtime` / `Session`，绑定工具、提示词、skills、策略，记录用量 | 业务流程编排 |
| 进程内扩展 | 注册工具、命令、事件处理器、provider、渲染器 | 权威的计费与权限控制 |

```text
workflow engine
  └─ worker process
       ├─ coding.AgentSessionServices   （长期复用）
       ├─ coding.Runtime                （长期复用，共享扩展 runner）
       ├─ host policy                    （BeforeToolCall / Subscribe / ledger）
       ├─ in-process extension factories （工具、provider、报告）
       └─ Session per node attempt
```

## 本文范围：只用进程内扩展

本文只使用 PiG 根模块的嵌入式 Go SDK：

```text
github.com/MichaelKinsy/PiG
```

扩展使用 PiG 原生进程内接口：

```go
type ExtensionFactory func(api extension.API) error
```

工厂由服务自己的 `main` 包按值传入，属于编译期信任边界。它不能由配置、文件、Package、Piglet 或任何运行时输入提供。所有扩展代码随服务一起编译、测试和发布；新增能力需要重新构建二进制。

本文不引入任何运行时扩展加载方式。扩展注册统一通过 `RuntimeOptions.NewExtensions` 完成，不使用资源加载器管理扩展。

## 节点与会话生命周期

### 一个 worker

创建一次 `coding.AgentSessionServices`，再创建一次 `coding.Runtime`：

```go
services, err := coding.CreateAgentSessionServices(coding.CreateAgentSessionServicesOptions{
    CWD:      workDir,
    AgentDir: agentDir,
    // 固定 compaction、retry、cacheWarming 等设置，避免读取用户环境
    SettingsManager: coding.NewInMemorySettingsManager(settings),
    ResourceLoaderOptions: &coding.DefaultResourceLoaderOptions{
        NoExtensions:   true,  // 扩展由 RuntimeOptions.NewExtensions 固定注入
        NoContextFiles: true,  // 不让 AGENTS.md 污染 workflow 提示词
        AdditionalSkillPaths: skillPaths,
    },
})
runtime, err := coding.NewRuntime(coding.RuntimeOptions{
    Services:         services,
    NewExtensions:    compiledExtensions,
    ExtensionRuntime: extensionRuntime,
})
```

`Services` 负责设置、认证、模型目录和资源加载器；`Runtime` 负责扩展执行和创建 Session。二者都应在 worker 生命周期内复用。

### 一个节点尝试

每个节点尝试创建一个新 `Session`，跑完关闭：

```go
session, err := runtime.New(coding.SessionStartOptions{
    Model:                  model,
    SystemPromptSections:   spec.Sections,
    ExtraTools:             spec.Tools,
    SkipBuiltinTools:       true,
    AllowedTools:           spec.AllowedTools,
    InitialActiveToolNames: spec.ActiveTools,
    BeforeToolCall:         []agent.BeforeToolCallHook{policy.BeforeToolCall},
    NoSession:              true, // 或给每次尝试单独的 SessionDir
})
defer session.Close()
```

不要跨节点共享一个 `Session`。`Session.Send` 是阻塞调用，`Session` 不是并发执行单元。

### 关闭顺序

1. 结束或取消活跃的 Session 操作；
2. 关闭每个 Session；
3. 关闭 Runtime；
4. 关闭 Services。

Runtime 关闭时会让扩展 runner 失效，之后不应再调用该 Runtime 创建的 Session。

## 提示词与 Skills

PiG 的 skill 是**知识**，不是可执行流程。`SKILL.md` 的正文是给模型的指令，模型通常通过 `read` 工具读取。

对 workflow 来说，让模型自己发现并读取 skill 不够确定。推荐：

- 当前步骤明确使用哪个 skill，就把该 skill 的正文直接放进 `SystemPromptSections`。
- 每一步用 `SetSystemPromptSections` 替换段落，用 `SetActiveToolsByName` 切换工具。
- 如果确实需要模型自行选择 skill，再用 `AdditionalSkillPaths` 注册，并保留 `read` 工具；但工具权限仍要按步骤收紧。

```go
session.SetSystemPromptSections(ai.OrderedSections{
    {Name: "role", Value: text("你是 workflow 的节点执行器。")},
    {Name: "task", Value: text(step.Goal)},
    {Name: "skill", Value: text(skillBody)},
    {Name: "constraints", Value: text(step.Constraints)},
    {Name: "done", Value: text(step.CompletionCriteria)},
})
session.SetActiveToolsByName(step.AllowedTools)

_, err := session.Send(ctx, step.Input)
```

`SystemPromptSections` 优先于 `SystemPrompt`，并且按顺序渲染。section 名称必须是合法的 XML tag 名。

workflow 的顺序不要只写进 skill 里。顺序由宿主控制：

1. 当前步骤只暴露该步骤允许的工具；
2. `BeforeToolCall` 拒绝越界调用；
3. `Send` 返回后校验结果，再进入下一步。

如果某一步本质上不需要模型判断，直接在宿主的 Go 代码里执行那个工具，不要为了“agent 感”强行让模型调用。

## 工具与权限

### 工具来源

| 形式 | 适用 |
|---|---|
| `coding.SessionStartOptions.ExtraTools`（`[]agent.AgentTool`） | 宿主工具，需要访问数据库、队列、workflow 上下文 |
| `coding.SessionStartOptions.CustomTools`（`[]extension.ToolDefinition`） | 想使用 Pi 工具定义形状的工具 |
| 进程内扩展 `api.RegisterTool` | 可复用、需要随扩展一起注册的工具 |

内置工具默认不要全开。只给当前节点真正需要的工具，减少模型乱调用：

```go
coding.SessionStartOptions{
    SkipBuiltinTools:       true,
    ExtraTools:             node.Tools,
    AllowedTools:           node.AllowedTools,
    InitialActiveToolNames: node.ActiveTools,
}
```

### 硬权限

真正执行限制放在 `BeforeToolCall`：

```go
func (p *Policy) BeforeToolCall(
    ctx context.Context,
    toolCallID, toolName string,
    args json.RawMessage,
) agent.ToolCallHookResult {
    if !p.Allows(toolName) {
        return agent.ToolCallHookResult{
            Block:  true,
            Reason: "tool " + toolName + " is not allowed in this step",
        }
    }
    if p.Calls.Add(1) > p.MaxCalls {
        return agent.ToolCallHookResult{
            Block:  true,
            Reason: "tool call budget exceeded",
        }
    }
    if p.OverBudget() {
        return agent.ToolCallHookResult{
            Block:  true,
            Reason: "cost budget exceeded",
        }
    }
    return agent.ToolCallHookResult{}
}
```

补充：

- `BeforeToolCall` 在工具执行前运行，返回 `Block` 会阻止执行，模型会收到 `Reason`。
- 需要修改或脱敏工具结果时，用进程内扩展的 `tool_result` handler，或直接使用 `coding.NewSession` 并传入完整的 `SessionOptions`。`SessionStartOptions` 只暴露了 `BeforeToolCall`。
- 并发工具调用不会因为 `BeforeToolCall` 自动排队。需要串行的工具在 `Execute` 里加信号量，或使用顺序执行模式。
- `BeforeToolCall` 是安全网，不是沙箱。高风险节点应放在容器或独立 worker 中，工具实现自己也要校验路径和命令。

## 计费与用量

### 节点结束：权威对账

`Session.GetSessionStats()` 返回聚合用量：

```go
stats := session.GetSessionStats()
// stats.Tokens: Input / Output / CacheRead / CacheWrite / Total
// stats.Cost:   总费用
// stats.ToolCalls / AssistantMessages / ContextUsage
```

它会计入 assistant 消息、工具结果、压缩、分支摘要和缓存预热的 usage。节点结束时用它作为权威对账数据。

### 实时预算：事件累加

`Session.Subscribe` 的监听器同步执行，适合累加用量，不适合写数据库：

```go
unsub := session.Subscribe(func(ev agent.AgentEvent) {
    switch e := ev.(type) {
    case agent.MessageEndEvent:
        if e.Message.Assistant != nil && e.Message.Assistant.Usage != nil {
            policy.AddUsage(*e.Message.Assistant.Usage)
        }
    case agent.ToolExecutionEndEvent:
        if e.Result.Usage != nil {
            policy.AddUsage(*e.Result.Usage)
        }
    }
    if policy.OverBudget() {
        session.RequestAbort() // 非阻塞，适合在监听器里调用
    }
})
defer unsub()
```

要点：

- 不要重复计费。`message_end` 和 `tool_execution_end` 各记一次，不要再用 `agent_end` 把整段历史重加一遍。
- 监听器不要做慢操作。写数据库、调计费服务要丢到有界队列，由后台 worker 写入；关闭时 drain。
- 按 `run/node/attempt/sequence` 做幂等键，重试和重放不会重复扣费。
- 实时预算用事件累加，节点结束用 `GetSessionStats` 对账；最终以 provider 账单为准做校正。
- 重试和缓存预热会产生额外费用。短节点建议关闭 cache warming；workflow 自己重试时，考虑关闭 PiG 的自动重试，避免两层重试叠加。
- `Send` 不会因为 provider 失败而返回错误。节点结果要检查最后一条 assistant 消息的 `StopReason` / `ErrorMessage`，以及 `AgentSettledEvent.Aborted`。

## 进程内扩展：优势与代价

### 优势

对于“完整服务”的形态，进程内扩展更合适：

- 单二进制、单版本、单日志，没有扩展构建、进程监管、socket、心跳和序列化。
- 工具和事件调用是函数调用，延迟低。
- 可以直接访问服务内的 Go 对象、数据库连接、队列和 workflow 上下文。
- 普通 Go 调试、pprof、race detector 都能用。
- 天然没有“运行时扩展”的攻击面。

### 代价

- 没有运行时插件。加能力必须重新编译、重新部署。
- 扩展代码和宿主共享进程，goroutine 泄漏、阻塞、`os.Exit`、无限内存都会影响服务。
- 扩展 handler 和工具的 panic 会在 PiG 的边界被 recover，但你们自己起的 goroutine 不在保护范围内。
- `Runtime` 只有一个 runner，所有 Session 共用扩展记录和 handler。扩展状态必须线程安全，并按 session/tenant 隔离。
- 二进制会变大，但不需要额外的扩展运行时和工具链。

## 扩展注册：统一使用 `NewExtensions`

进程内扩展统一走 `RuntimeOptions.NewExtensions`。输入是已经加载好的 `[]extension.Extension` 记录：

```go
extRuntime := extension.CreateExtensionRuntime()
bus := extension.CreateEventBus()

ext, err := factoryload.LoadExtensionFromFactory(
    myFactory, cwd, bus, extRuntime, "builtin:workflow",
)
if err != nil {
    return err
}

runtime, err := coding.NewRuntime(coding.RuntimeOptions{
    Services:         services,
    NewExtensions:    []extension.Extension{ext},
    ExtensionRuntime: extRuntime,
})
```

特点：

- 工厂由你自己执行（通过 `factoryload`）。
- Runtime 构造时创建一个共享 runner，所有 Session 共用。
- 不参与 `Session.Reload` 的资源重载；重载后静态扩展不会自动保留。
- 提供完整的 `SessionStartOptions`，包括 `BeforeToolCall`、`SystemPromptSections`、`ExtraTools`。

这是 workflow 服务的唯一扩展注册路径。Runtime 的 Session 使用同一份 `NewExtensions` 共享 runner，扩展注册和生命周期都只有一个来源。

### 进程内工厂示例

```go
func toolFactory(deps *Deps) extension.ExtensionFactory {
    return func(api extension.API) error {
        api.RegisterTool(extension.ToolDefinition{
            Name:        "lookup_order",
            Description: "Look up an order.",
            Parameters:  json.RawMessage(`{"type":"object","properties":{"id":{"type":"string"}},"required":["id"]}`),
            Execute: func(
                ctx context.Context,
                toolCallID string,
                params json.RawMessage,
                onUpdate extension.AgentToolUpdateCallback,
            ) (extension.AgentToolResult, error) {
                // 直接调用服务内的依赖
                return extension.AgentToolResult{}, nil
            },
        })
        return nil
    }
}
```

工厂只做注册。`factoryload` 在加载阶段，`api` 的 action 方法还没有绑定；真正的工作放到 handler 或工具里。

## 参考骨架

以下代码省略错误处理，只表达结构。

```go
type Capability struct {
    Name    string
    Factory extension.ExtensionFactory
}

type Worker struct {
    services *coding.AgentSessionServices
    runtime  *coding.Runtime
    model    *ai.Model
}

func NewWorker(cfg Config) (*Worker, error) {
    services, err := coding.CreateAgentSessionServices(coding.CreateAgentSessionServicesOptions{
        CWD:             cfg.WorkDir,
        AgentDir:        cfg.AgentDir,
        SettingsManager: coding.NewInMemorySettingsManager(cfg.Settings),
        ResourceLoaderOptions: &coding.DefaultResourceLoaderOptions{
            NoExtensions:         true,
            NoContextFiles:       true,
            AdditionalSkillPaths: cfg.SkillPaths,
        },
    })
    if err != nil {
        return nil, err
    }

    extRuntime := extension.CreateExtensionRuntime()
    bus := extension.CreateEventBus()
    var exts []extension.Extension
    for _, capability := range cfg.Capabilities {
        ext, err := factoryload.LoadExtensionFromFactory(
            capability.Factory, cfg.WorkDir, bus, extRuntime, capability.Name,
        )
        if err != nil {
            services.Close()
            return nil, err
        }
        exts = append(exts, ext)
    }

    runtime, err := coding.NewRuntime(coding.RuntimeOptions{
        Services:         services,
        NewExtensions:    exts,
        ExtensionRuntime: extRuntime,
    })
    if err != nil {
        services.Close()
        return nil, err
    }

    return &Worker{services: services, runtime: runtime, model: cfg.Model}, nil
}

func (w *Worker) RunNode(ctx context.Context, spec NodeSpec, input string) (NodeResult, error) {
    policy := NewPolicy(spec)

    session, err := w.runtime.New(coding.SessionStartOptions{
        Model:                  w.model,
        SystemPromptSections:   spec.Sections,
        ExtraTools:             spec.Tools,
        SkipBuiltinTools:       true,
        AllowedTools:           spec.AllowedTools,
        InitialActiveToolNames: spec.ActiveTools,
        BeforeToolCall:         []agent.BeforeToolCallHook{policy.BeforeToolCall},
        NoSession:              spec.NoSession,
        SessionDir:             spec.SessionDir,
    })
    if err != nil {
        return NodeResult{}, err
    }
    defer session.Close()

    unsub := session.Subscribe(func(ev agent.AgentEvent) {
        switch e := ev.(type) {
        case agent.MessageEndEvent:
            if e.Message.Assistant != nil && e.Message.Assistant.Usage != nil {
                policy.AddUsage(*e.Message.Assistant.Usage)
            }
        case agent.ToolExecutionEndEvent:
            if e.Result.Usage != nil {
                policy.AddUsage(*e.Result.Usage)
            }
        }
        if policy.OverBudget() {
            session.RequestAbort()
        }
    })
    defer unsub()

    if _, err := session.Send(ctx, input); err != nil {
        return NodeResult{}, err
    }

    stats := session.GetSessionStats()
    return NodeResult{
        Stats:   stats,
        Aborted: policy.OverBudget(),
    }, nil
}
```

## 并发、重试和运行配置

- **一个 Session 只在一个 goroutine 里使用。** 不同节点可以并发，各自创建 Session。
- **重试归属唯一。** 要么 workflow 重试，关掉 PiG 自动重试；要么 PiG 重试，workflow 不再重试。每次 workflow 重试都创建新 Session。
- **超时用 `context.WithTimeout` 包住 `Send`。** 取消后关闭 Session，释放运行中的工具和扩展调用。
- **短节点关闭 compaction 和 cache warming。** 长任务再打开 compaction，避免上下文无限增长。
- **限制 provider 并发。** 在 worker 或租户粒度加信号量，避免供应商限流。
- **持久化仅用于调试。** workflow 的节点状态和结果才是权威；PiG Session 文件可以按 `run/node/attempt` 保存，便于回放。
- **安全边界。** 工具运行在服务权限下。高风险节点放进容器/沙箱，工具实现自己校验路径和命令。

## 常见坑

1. **扩展没有通过 `RuntimeOptions.NewExtensions` 注册。** 进程内扩展必须作为 `extension.Extension` 记录传给 Runtime；仅仅定义了工厂函数不会自动生效。
2. **每个节点重建 `Services` / `Runtime`。** 会重复加载模型目录、skills 和扩展注册；按 worker 复用。
3. **并发共用一个 Session。** `Send` 是阻塞的，Session 不是并发执行单元。
4. **只靠提示词限制工具。** 提示词是建议，不是权限；用 `BeforeToolCall`。
5. **把计费只放在扩展里。** 扩展可能在运行时出错；宿主 hook 才是权威执行点。
6. **用 `agent_end` 重复计费。** 用 `message_end` / `tool_execution_end` 各记一次，结束时用 `GetSessionStats` 对账。
7. **调用 `Session.Reload`。** 它从 ResourceLoader 重建 runner，`NewExtensions` 里的静态扩展不会自动保留。服务里重启进程即可。
8. **工厂里调用 action 方法。** 加载阶段 `api` 的 action 还没有绑定；工厂只做注册。
9. **不传 `ExtensionRuntime`。** 扩展注册的 provider 会留在队列里，到不了模型注册表。
10. **扩展状态跨 Session 共享。** Runtime 只有一个 runner；按 session/tenant 隔离，或每个租户独立 Runtime。
11. **期望 `Send` 返回 provider 错误。** provider 失败时 `err` 可能为 nil；检查 assistant 消息的 `StopReason` / `ErrorMessage`。
12. **忽略缓存预热和重试费用。** 它们是真实请求；短节点关闭 cache warming，明确重试归属。

## 落地顺序

1. 单节点最小闭环：`Services` + `Runtime` + `Session` + 一个宿主工具 + `GetSessionStats`。
2. 加 `BeforeToolCall`：步骤工具白名单、调用次数、预算。
3. 加 `SystemPromptSections` 和按步骤切换工具，确认模型不会越界。
4. 加事件计量和后台 ledger，节点结束用 `GetSessionStats` 对账。
5. 把领域工具、provider、报告等能力做成进程内扩展工厂。
6. 最后做并发 worker、重试策略、容器隔离和 provider 限流。

## 参考

- `docs/site/docs/sdk.md`：嵌入式 Go SDK。
- `docs/specs/extension-factory-trust.md`：编译进进程的工厂信任边界。
- `coding/runtime.go`：`RuntimeOptions`、`SessionStartOptions`、共享 runner。
- `coding/session.go`：Session 构建、ResourceLoader、工具和 hook。
- `coding/session_tool_registry.go`：`Reload` 与 runner 重建。
- `coding/services_extension_runtime.go`：扩展 provider 注册的排空。
- `coding/extension/extension.go`：`Extension`、`ExtensionFactory`。
- `coding/extension/factoryload/load.go`：进程内工厂加载、失败回滚。
- `examples/sdk/main.go`：最小嵌入式示例。

# RPC 模式

RPC 模式把 PiG 跑成一个无界面的 JSON Lines 进程。客户端往标准输入写命令，从标准输出读响应和事件。协议沿用 Pi 1.4.0 的 RPC 协议（[Pi 的 RPC 协议参考](https://github.com/earendil-works/pi/blob/v1.2.0/packages/coding-agent/docs/rpc.md)），本页列出的差异除外。

Go 应用不需要进程边界时，用 [PiG Go SDK](zh-sdk.md)。需要别的语言、IDE 或独立进程时，用 RPC 模式。

> 本页是英文页 `rpc.md` 的中文译本，对应 PiG 提交 `c64bebd8`。英文页是规范来源；帧格式和错误文本以英文页与当前源码为准。

## 启动 RPC 模式

```bash
pig --mode rpc [--model <provider/model>] [options]
```

常用选项：

- `--provider <name>`：模型参数里没带 Provider 时指定 Provider。
- `--model <provider/model>`：选择模型。
- `--name <name>` 或 `-n <name>`：设置初始会话名。
- `--no-session`：关闭会话持久化。
- `--session-dir <path>`：指定会话目录。
- `--piglet <name-or-path>`：选择一个 Agent 应用。

PiG 把运行诊断写到标准错误。标准输出只当作协议数据。

省略 `--model` 又没有默认模型时，RPC 模式以 Pi 的 `unknown` 模型记录启动。用 `get_state`、`get_available_models` 和 `set_model` 选择模型。在选中 Provider 配好认证之前，`prompt` 会在预检失败。恢复的会话如果其保存模型没有凭据，也会以 `unknown` 模型启动，与 Pi 一致。

初始活动工具列表和系统消息里的工具声明保留扩展加载与注册顺序。多个扩展注册同名工具时，第一个注册生效。

## 帧格式

每条输入或输出记录是一个 JSON 对象加 LF（`\n`）。

JSON 和 RPC 模式共用事件编码器。事件字段按 Pi 的构造顺序排列，而不是字母顺序。编码器把 JSON 字符串（包括嵌套记录）里的 `<`、`>`、`&` 和 Unicode 行分隔符写成字面字符。因此按 JSON 解析记录，不要依赖成员位置。

- 只用 LF 切分记录。
- 发送 CRLF 时去掉尾部 CR。
- 不要按 Unicode 行分隔符切分 JSON 字符串。
- 单条输入命令不要超过命令读取器 16 MiB 的限制。

每条命令都可以带 `id`，对应响应会带回它：

```json
{"id":"req-1","type":"get_state"}
```

```json
{"id":"req-1","type":"response","command":"get_state","success":true,"data":{}}
```

响应确认命令被接受，或报告命令错误。后续的模型或工具失败以事件形式到达。

`compact` 请求会先中止并汇合当前轮次，再做摘要。它不在输入循环上等待，所以客户端仍可发送状态查询和扩展对话框应答。同一输入批次里的多个 `compact` 依次执行；Pi 1.4.0 会同时启动它们（D147）。同一批次里的会话替换会中止运行中的压缩，压缩随后返回一个错误（D148）。

## Pi 兼容边界

PiG 实现固定 Pi `rpc-types.ts` 里的命令联合。固定的 Pi 命令没有对应的 PiG 分发分支时，生成的测试清单会失败。PiG 多了一个命令 `get_extensions`（D101）。见 [RPC 命令](/docs/latest/rpc-commands#loaded-extensions)。

关闭标准输入即可关闭 RPC 模式。PiG 在关闭进程前会取消活动提示、Bash 命令、会话切换、模型选择处理器、重试和扩展 UI 请求。标准输入结束时 `new_session`、`fork`、`clone` 或 `switch_session` 仍在替换会话，PiG 会完成替换，写出响应后再退出；Pi 1.4.0 不等这条命令的响应就退出（D146）。挂起的扩展命令的续体可能在其运行时停止前运行（D85）；无法报告等待的运行时，其收尾处理会让标准输入一直打开到它结束（D86）。

Go 程序可以用 `github.com/MichaelKinsy/PiG/coding/rpcclient`，它是 Pi TypeScript `RpcClient` 的移植。它启动 `pig --mode rpc`、配对响应、提供带类型的命令方法，并把事件交给监听器。其他语言直接使用 JSONL 协议。

`new_session`、`switch_session`、`fork` 和 `clone` 通过创建第一个会话的同一个运行时工厂替换会话，与 Pi 的 rpc-mode 使用运行时宿主的方式一致。替换会为目标会话工作目录创建新的扩展 runner、扩展进程和服务，旧 runner 失效。与 Pi 1.4.0 一致，这四条命令之后替换出的扩展会收到两次 `session_start`，因为 rpc-mode 在命令返回后又绑定了一次。扩展进程不会继承被替换进程的进程级状态（例如环境变量）。这是 D70，也覆盖 `/reload`。

## 事件

跨进程边界的部分消息观察是已知差距（D82，2026-09-28 的所有者决定）。PiG 发送快照，而不是活动的生产者引用。因此初始 assistant 消息可能比 Pi 的浅拷贝/活动观察内容更少，或停止状态更早。中间 Completions 快照省略 `partialArgs` 和 `streamIndex`；Responses 快照省略 `partialJson`。最终消息在两个宿主里都必须省略这些解析器属性。事件顺序、增量、终态结果和持久消息不在这项许可范围内。严格的 RPC33 比较保留完整原始差异，不算通过。

PiG 发出这些事件：

- `agent_start`；
- `agent_end`，带 `messages` 和 `willRetry`；
- `agent_settled`，带 `aborted`；运行因中止结束时为 `true`；
- `turn_start`；
- `turn_end`，带 assistant 消息和工具结果；
- `message_start`；
- `message_update`；
- `message_end`；
- `tool_execution_start`；
- `tool_execution_update`；
- `tool_execution_end`，工具运行过时带 `durationMs`；
- `bash_execution_update`；
- `queue_update`；
- `thinking_level_changed`；
- `compaction_start` 和 `compaction_end`；
- 自动重试与摘要重试事件；
- 扩展持久化自定义会话条目后的 `entry_appended`；
- `session_info_changed`；
- `extension_ui_request`；
- `extension_error`；
- `error`。

`message_update` 包含一个 `assistantMessageEvent`。文本与工具调用流使用对应的 start、delta、end 记录。增量不包含累计的部分消息。

assistant 的 `message_start` 可能在 Provider 响应头成功之后、任何响应体数据到达之前到达。OpenAI Completions 和 Responses 在 HTTP 建立完成前就返回事件流，并允许在读取响应体数据前开始。之后的取消或流失败仍会终结 assistant 消息；start 不保证响应成功。

部分消息是在事件编码时观察的，不是 Provider 最初推送时。`message_start` 可能已经包含缓冲的文本或工具调用。中间块可以包含 Provider 的解析器字段，与 Pi 一样：OpenAI Completions 工具调用带 `partialArgs`、`customInput` 或 `streamIndex`；OpenAI Responses 与 Codex 工具调用带 `partialJson`；Anthropic Messages 与 Bedrock Converse Stream 的打开块带 `index`，其工具调用带 `partialJson`；Mistral 工具调用带 `partialArgs`。最终消息会去掉这些字段。要持久化，请保存最终消息或会话条目，而不是中间部分消息。

文本流示例：

```json
{"type":"message_start","message":{"role":"assistant","content":[]}}
{"type":"message_update","assistantMessageEvent":{"type":"text_start","contentIndex":0}}
{"type":"message_update","assistantMessageEvent":{"type":"text_delta","contentIndex":0,"delta":"Hello"}}
{"type":"message_update","assistantMessageEvent":{"type":"text_end","contentIndex":0,"content":"Hello"}}
{"type":"message_end","message":{"role":"assistant","content":[{"type":"text","text":"Hello"}]}}
```

消息与会话条目的形状见 [会话文件格式](zh-session-format.md)。

## 完整示例

这个例子不用客户端库，直接在 shell 里驱动 `pig --mode rpc`。它先发两条命令，等运行结束，再发三条（其中一条无效），最后发一行不是 JSON 的内容。Provider 是脚本化测试服务器，总回答 `Hello from the test model.`，所以回复措辞只是示例；记录、顺序和错误文本是 PiG 实际打印的。Pi 1.4.0 对同样输入打印了相同的记录类型、相同顺序和相同解析错误。

```bash
(
  printf '%s\n' '{"id":"1","type":"get_state"}' \
                '{"id":"2","type":"prompt","message":"Say hello"}'
  sleep 3                       # let the run finish
  printf '%s\n' '{"id":"3","type":"get_last_assistant_text"}' \
                '{"id":"4","type":"get_session_stats"}' \
                '{"id":"5","type":"bogus"}' \
                'not json'
  sleep 1
) | pig --mode rpc --no-session --model mock/mock-1 2>rpc.stderr > rpc.jsonl
echo "exit status: $?"
```

```text
exit status: 0
```

标准输入关闭后，PiG 以状态 0 结束进程。`rpc.stderr` 为空。这里的 `sleep 3` 只是 shell 脚本的写法；真正的客户端应等待 `agent_settled` 事件。

**记录顺序。** 这个过滤器打印每行的类型、`id`、命令和结果：

```bash
jq -c '[.type, .id, .command, .success]' rpc.jsonl
```

```text
["response","1","get_state",true]
["response","2","prompt",true]
["agent_start",null,null,null]
["turn_start",null,null,null]
["message_start",null,null,null]
["message_end",null,null,null]
["message_start",null,null,null]
["message_end",null,null,null]
["message_start",null,null,null]
["message_update",null,null,null]
["message_update",null,null,null]
["message_update",null,null,null]
["message_end",null,null,null]
["turn_end",null,null,null]
["agent_end",null,null,null]
["agent_settled",null,null,null]
["response",null,"parse",false]
["response","3","get_last_assistant_text",true]
["response","4","get_session_stats",true]
["response","5","bogus",false]
```

注意三点：

- `prompt` 的响应在模型回答**之前**到达。`"data":{"disposition":"started"}` 表示 PiG 接受了提示；答案以事件形式到达，`agent_settled` 是最后一个。模型调用失败是事件，不是失败的响应（见上文“帧格式”）。
- 事件不带 `id`。用 `id` 配对响应与命令；没有 `"type":"response"` 的记录都当作事件。
- 非 JSON 行的响应（`"command":"parse"`，没有 `id`，因为本来就没有可回显的）在命令 3 到 5 的响应**之前**到达，尽管后者在同一批里先发出。一次发多条命令时，不要依赖响应按发送顺序到达；用 `id` 配对。

**状态。** 任何提示之前的 `get_state`：

```json
{"id":"1","type":"response","command":"get_state","success":true,"data":{"model":{"id":"mock-1","name":"Mock","api":"openai-completions","provider":"mock","baseUrl":"http://127.0.0.1:18931/v1","reasoning":false,"input":["text"],"cost":{"input":0,"output":0,"cacheRead":0,"cacheWrite":0},"contextWindow":128000,"maxTokens":4096},"thinkingLevel":"off","isStreaming":false,"isCompacting":false,"steeringMode":"one-at-a-time","followUpMode":"one-at-a-time","sessionId":"01a12c60-0fd7-71bc-8a36-b557333c8aeb","autoCompactionEnabled":true,"messageCount":0,"pendingMessageCount":0}}
```

**答案与总计。**

```json
{"id":"3","type":"response","command":"get_last_assistant_text","success":true,"data":{"text":"Hello from the test model."}}
{"id":"4","type":"response","command":"get_session_stats","success":true,"data":{"sessionId":"01a12c60-0fd7-71bc-8a36-b557333c8aeb","userMessages":1,"assistantMessages":1,"toolCalls":0,"toolResults":0,"totalMessages":3,"tokens":{"input":100,"output":20,"cacheRead":0,"cacheWrite":0,"total":120},"cost":0,"contextUsage":{"tokens":120,"contextWindow":128000,"percent":0.09375}}}
```

`totalMessages` 是 3：系统消息、你的提示和回复。token 计数来自测试服务器。

**错误。** 未知命令和非 JSON 行各得到一个 `success:false` 响应，进程继续运行：

```json
{"id":"5","type":"response","command":"bogus","success":false,"error":"Unknown command: bogus"}
{"type":"response","command":"parse","success":false,"error":"Failed to parse command: Unexpected token 'o', \"not json\" is not valid JSON"}
```

解析消息是 JavaScript JSON 解析器给出的文本，与 Pi 一致。把 `error` 当自由文本，不要当稳定错误码。

## 最小客户端

```python
import json
import subprocess

process = subprocess.Popen(
    ["pig", "--mode", "rpc", "--model", "openai/gpt-5", "--no-session"],
    stdin=subprocess.PIPE,
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE,
    text=True,
    bufsize=1,
)

process.stdin.write(json.dumps({
    "id": "prompt-1",
    "type": "prompt",
    "message": "List the packages in this repository.",
}) + "\n")
process.stdin.flush()

for line in process.stdout:
    event = json.loads(line)
    print(event)
    if event.get("type") == "agent_settled":
        break

process.stdin.close()
```

要单独持续读取标准错误。写满的错误管道会阻塞子进程。

## 参考

- [RPC 命令](/docs/latest/rpc-commands) 列出每条命令及其响应。
- [RPC 扩展 UI](/docs/latest/rpc-extension-ui) 说明扩展对话框请求和客户端返回的响应。
- [JSON 事件流](/docs/latest/json) 说明 JSON 模式下也会写出的记录。
- [会话文件格式](zh-session-format.md) 说明会话命令返回的条目。

## 源码位置

PiG 实现：

- `coding/cli/rpc_mode.go`：命令循环。
- `coding/cli/rpc_types.go`：命令与响应类型。
- `coding/cli/rpc_ui.go`：扩展 UI 请求与响应传输。
- `coding/cli/rpc_events.go`：事件转换。
- `coding/rpcclient`：Go 子进程客户端。

Pi 参考（1.2.0）：

- [RPC 模式](https://github.com/earendil-works/pi/blob/v1.2.0/packages/coding-agent/src/modes/rpc/rpc-mode.ts)
- [RPC 类型](https://github.com/earendil-works/pi/blob/v1.2.0/packages/coding-agent/src/modes/rpc/rpc-types.ts)

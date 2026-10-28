# 会话文件格式

PiG 把每个会话存成 JSON Lines。第 1 行是会话头，之后每行是树上的一个条目。

格式沿用 Pi 1.4.0 的会话格式（[Pi 的会话文件格式](https://github.com/earendil-works/pi/blob/v1.2.0/packages/coding-agent/docs/session-format.md)）。PiG 保留读到的每个条目 JSON，所以未知字段、未知条目类型和成员顺序在加载与重写后仍然存在，与 Pi 一致。

> 本页是英文页 `session-format.md` 的中文译本，对应 PiG 提交 `c64bebd8`。英文页是规范来源；字段名和版本以英文页与当前源码为准。

## 文件位置

默认位置：

```text
~/.pig/agent/sessions/--<project-path>--/<timestamp>_<uuid>.jsonl
```

配置了不同的 Agent 目录时，PiG 使用 `PIG_CODING_AGENT_DIR`。

用 `/resume` 打开或删除会话。系统里有 `trash` 命令时，PiG 用它删除。

## 会话头

第一行没有 `id` 或 `parentId`：

```json
{"type":"session","version":3,"id":"session-uuid","timestamp":"2026-01-15T14:00:00Z","cwd":"/work/project"}
```

用 `/fork` 或 `/clone` 创建的会话还会包含 `parentSession`，即原会话文件的路径。

当前上游会话 schema 版本是 3。PiG 加载时迁移受支持的旧版上游会话结构。不要手工修改版本字段。

## 条目树

头之后的每个条目都包含：

```json
{
  "type": "message",
  "id": "a1b2c3d4",
  "parentId": null,
  "timestamp": "2026-01-15T14:00:01Z"
}
```

`parentId` 指向该分支上的前一个条目，根条目用 `null`。分支就是给较早的条目新增一个子节点。活动叶节点标识当前分支。

条目在文件里保持追加顺序。要得到当前对话，从活动叶节点往根走。不要把文件顺序当成一条线性对话。

## 条目类型

### message

`message` 条目包含一条嵌套的 Agent 消息：

```json
{"type":"message","id":"a1b2c3d4","parentId":null,"timestamp":"2026-01-15T14:00:01Z","message":{"role":"user","content":[{"type":"text","text":"Hello"}],"timestamp":1768485601000}}
```

支持的消息角色包括：

- `system`；
- `user`；
- `assistant`；
- `toolResult`；
- `bashExecution`；
- `custom`；
- `branchSummary`；
- `compactionSummary`。

`system` 消息携带系统提示和工具声明。会话的第一次请求会存一条消息，把所有提示 section 放在 `sections`，所有工具放在 `toolsAdded`。之后的改动存一条 system 消息，按名字补丁 `sections`（`null` 表示删除某个 section），并列出 `toolsAdded` 与 `toolsRemoved`。按顺序重放即可得到当前提示和工具。`sections` 为空的 system 消息会存成 `"sections": {}`，与 Pi 一致。

内容块使用 Pi 的字段名：

- `text`，含 `text` 和可选的 `textSignature`；
- `image`，含 `data`（base64 编码的图片字节）和 `mimeType`，例如 `image/png`；没有 URL 形式；
- `thinking`，含 `thinking`、可选的 `thinkingSignature` 和可选的 `redacted` 标志；
- `toolCall`，含 `id`、`name`、`arguments`，以及可选的 `thoughtSignature` 和 `namespace`。

例如，调用 `read` 工具的 assistant 轮次存为：

```json
{"type":"toolCall","id":"call_1","name":"read","arguments":{"path":"a.txt"}}
```

把签名当作 Provider 的不透明数据。旧版 PiG 写的会话可能包含用 `input` 代替 `arguments` 的 `tool_use` 块；PiG 仍能读取该形式，写出时用 `toolCall`。

完成的 assistant 消息可以包含 provider、model、API、usage、diagnostics、响应身份、`stopReason`、`errorMessage` 和 `durationMs`。`durationMs` 是从请求开始到最终消息的毫秒数。工具结果消息可以包含 `durationMs`，即工具 `execute` 的耗时（不含钩子）；没有运行的调用没有该字段。不使用的字段一律当作不透明数据。

Bash 条目使用常规的外层 `message` 形状，嵌套角色是 `bashExecution`：

```json
{"type":"message","id":"b2c3d4e5","parentId":"a1b2c3d4","timestamp":"2026-01-15T14:00:02Z","message":{"role":"bashExecution","command":"go test ./...","output":"ok","exitCode":0,"cancelled":false,"truncated":false}}
```

Bash 的可选字段有 `fullOutputPath`（保存过长输出的文件）、`timestamp`（Unix 毫秒）和 `excludeFromContext`（用 `!!` 运行的命令会设置，模型看不到）。

### model_change

```json
{"type":"model_change","id":"c3d4e5f6","parentId":"b2c3d4e5","timestamp":"2026-01-15T14:05:00Z","provider":"openai","modelId":"gpt-5"}
```

### usage

`usage` 条目记录不属于 assistant 消息的模型用量，例如缓存预热刷新（`"kind": "cache_warm"`）。它计入会话的 token 与成本总计，但不是模型上下文，树视图也不显示。

```json
{"type":"usage","id":"f6a7b8c0","parentId":"c3d4e5f6","timestamp":"2026-01-15T14:08:00Z","kind":"cache_warm","provider":"anthropic","model":"claude-sonnet-4-5","usage":{"input":0,"output":0,"cacheRead":50000,"cacheWrite":0,"totalTokens":50000,"cost":{"input":0,"output":0,"cacheRead":0.015,"cacheWrite":0,"total":0.015}}}
```

未知的 `kind` 按普通用量处理。

### thinking_level_change

```json
{"type":"thinking_level_change","id":"d4e5f6a7","parentId":"c3d4e5f6","timestamp":"2026-01-15T14:06:00Z","thinkingLevel":"high"}
```

### compaction

```json
{"type":"compaction","id":"e5f6a7b8","parentId":"d4e5f6a7","timestamp":"2026-01-15T14:10:00Z","summary":"Earlier work summary","firstKeptEntryId":"c3d4e5f6","tokensBefore":50000}
```

可选字段包括 `details`、`fromHook`、生成摘要的 `usage` 和 `systemMessage`。`systemMessage` 记录压缩边界处的系统提示与工具声明，它会成为压缩后上下文的开头系统消息。内置 `toolsAdded` 的参数 schema 在保存、加载后再次压缩时保留 Pi 的声明顺序，包括嵌套属性。

PiG 用摘要、`firstKeptEntryId` 和之后的条目构造压缩后上下文。当前格式没有定义 `retainedTail` 字段。

### branch_summary

```json
{"type":"branch_summary","id":"f6a7b8c9","parentId":"a1b2c3d4","timestamp":"2026-01-15T14:15:00Z","fromId":"e5f6a7b8","summary":"Summary of the branch that was left"}
```

可选字段包括 `details`、`fromHook` 和 `usage`。

### custom

自定义条目保存扩展状态，不进入模型上下文：

```json
{"type":"custom","id":"a7b8c9d0","parentId":"f6a7b8c9","timestamp":"2026-01-15T14:20:00Z","customType":"review-state","data":{"count":42}}
```

`customType` 请带命名空间。把其他扩展的数据当作不透明内容。

### custom_message

自定义消息可以参与模型上下文：

```json
{"type":"custom_message","id":"b8c9d0e1","parentId":"a7b8c9d0","timestamp":"2026-01-15T14:25:00Z","customType":"review-note","content":"Review the changed API.","display":true}
```

`content` 可以是字符串或内容块数组。`details` 可选。

### label

```json
{"type":"label","id":"c9d0e1f2","parentId":"b8c9d0e1","timestamp":"2026-01-15T14:30:00Z","targetId":"a1b2c3d4","label":"checkpoint"}
```

`label` 为 `null` 表示清除目标的标签。

### context_edit

`context_edit` 条目改变较早条目对未来模型上下文的贡献。较早条目本身在文件、树、导出和用量统计中保持不变。

```json
{"type":"context_edit","id":"e1f2a3b4","parentId":"c9d0e1f2","timestamp":"2026-01-15T14:32:00Z","targetId":"a1b2c3d4","replacement":null}
```

目标可以是 user、assistant、tool-result 或 custom-message 条目。`replacement` 为 `null` 表示从模型上下文省略目标；带 `content` 的 `replacement` 只替换目标的内容。多个编辑指向同一条目时，活动分支上最新的一个生效。

### session_info

```json
{"type":"session_info","id":"d0e1f2a3","parentId":"c9d0e1f2","timestamp":"2026-01-15T14:35:00Z","name":"Refactor authentication"}
```

`/resume` 里显示最新的会话名。

## 上下文构造

PiG 从活动的根到叶分支构造模型上下文：

1. 找到活动分支。
2. 应用最新的模型和思考级别条目。
3. 按 `firstKeptEntryId` 应用压缩。
4. 把 `context_edit` 条目应用到目标。
5. 把 message、custom-message、branch-summary 和 compaction 条目转换为模型消息。
6. 排除 custom 状态条目、usage 条目，以及标记 `excludeFromContext` 的 Bash 条目。

不要按文件顺序读取每一行 `message` 来重建模型上下文。

## 从扩展读取会话

Go 扩展 SDK 提供：

```go
branch, err := ctx.GetBranch()      // []sdk.BranchEntry for the active branch
entries, err := ctx.GetEntries()    // []json.RawMessage for all entries
leafID, err := ctx.GetLeafID()      // *string, nil for an empty session
file, err := ctx.GetSessionFile()   // *string, nil for an in-memory session
```

`GetBranch` 提供扁平化的读取视图，覆盖常用消息字段。需要精确的条目 JSON 时用 `GetEntries`。

只有当扩展第一次请求条目时，PiG 才为该扩展订阅会话日志复制。之后的追加更新本地 SDK 镜像。这样避免反复整份历史的 IPC，也不会把长会话复制给从不查看它的扩展。

## 用 Go 解析文件

用 `json.RawMessage` 让未知字段在你的解析器里保留下来：

```go
file, err := os.Open("session.jsonl")
if err != nil {
    return err
}
defer file.Close()

scanner := bufio.NewScanner(file)
scanner.Buffer(make([]byte, 64*1024), 128*1024*1024)
for scanner.Scan() {
    var entry struct {
        Type     string          `json:"type"`
        ID       string          `json:"id"`
        ParentID *string         `json:"parentId"`
        Message  json.RawMessage `json:"message"`
    }
    if err := json.Unmarshal(scanner.Bytes(), &entry); err != nil {
        return err
    }
    // Process known fields. Retain scanner.Bytes() for an exact copy.
}
return scanner.Err()
```

显式设置扫描上限。一行会话可能包含很大的工具结果或图片。

## 兼容规则

- 保留头部的 schema 版本。
- 保留未知字段和未知条目类型。
- 保留 `null` 与缺失 `parentId` 的区别。
- 不要重排条目。
- 不要改写时间戳或 ID。
- PiG 正在写会话时不要编辑它。
- 运行中的会话要追加扩展状态或消息，请使用扩展 API。

## 源码位置

PiG 实现：

- `internal/sessionentry/union.go` 定义 Pi `SessionEntry` 联合的条目类型。
- `internal/codingagent/session.go` 定义持久条目形状。
- `internal/codingagent/session_manager.go` 创建、打开和列出会话。
- `extensions/sdk/context.go` 定义 `BranchEntry`、`GetBranch` 和 `GetEntries`。

Pi 参考（1.2.0）：

- [Pi 会话管理器](https://github.com/earendil-works/pi/blob/v1.2.0/packages/coding-agent/src/core/session-manager.ts)
- [Pi coding-agent 消息](https://github.com/earendil-works/pi/blob/v1.2.0/packages/coding-agent/src/core/messages.ts)

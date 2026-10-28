# Provider

PiG 通过与 Pi 兼容模型共用的 Provider 注册表连接推理服务。每个内置 Provider 在 `ai/` 下都有 Go 实现，负责认证、请求构造和流解析。扩展可以通过公开扩展 API 注册额外的 Provider。见 [扩展](zh-extensions.md)。

大多数托管服务接受 API key，有些还支持浏览器或设备 OAuth 登录。Amazon Bedrock 和 Google Vertex AI 还可以使用云平台的环境凭据。运行 `/login` 或 `pig login --list` 查看各 Provider 的登录方式。

> 本页是英文页 `providers.md` 的中文译本，对应 PiG 提交 `c64bebd8`。英文页是规范来源；Provider 键、环境变量和错误文本以英文页与当前源码为准。

## 内置 Provider

用 Go、Rust 或 Python 写的扩展要跨进程访问 Provider 对象，因此 Pi 在堆内可用的部分 Provider 行为对它们不可用（D78）。已注册的原生 Provider 方法可以跨进程调用。Go、Rust、Python 的 `getProvider` 取不到每个内置或组合出来的原始 Provider 对象，遇到不可用的载体就报错。这类扩展读到的已注册配置是快照，不是作者对象的活动别名，所以属性写入不会同步。同进程内的 Node 扩展保持 Pi 的行为。正常的模型选择、认证、刷新、取消和注册清理不受影响。

PiG 与 Pi 1.4.0 提供相同的内置聊天 Provider 和分类器模型。`typesafe` 只有分类器模型；`cloudflare-workers-ai`、`openai`、`opencode`、`openrouter` 和 `vercel-ai-gateway` 把它们与聊天模型并列。Provider 键是 `provider/model` 标识的第一部分。Wire 列是该 Provider 内置聊天模型使用的 API。

| Provider 键 | 名称 | Wire | 凭据 |
|---|---|---|---|
| `amazon-bedrock` | Amazon Bedrock | `bedrock-converse-stream` | AWS 凭据链或 `AWS_BEARER_TOKEN_BEDROCK`。见 [Amazon Bedrock](#amazon-bedrock)。 |
| `ant-ling` | Ant Ling | `openai-completions` | `ANT_LING_API_KEY` |
| `anthropic` | Anthropic | `anthropic-messages` | `ANTHROPIC_API_KEY`、`ANTHROPIC_OAUTH_TOKEN`、`ANTHROPIC_AUTH_TOKEN`、工作负载身份联邦或 OAuth。见下方“Anthropic 工作负载身份联邦”。 |
| `azure` | Azure | `azure-openai-responses`、`openai-completions` | `AZURE_OPENAI_API_KEY` 加端点。见 [Azure OpenAI](#azure-openai)。 |
| `baseten` | Baseten | `openai-completions` | `BASETEN_API_KEY` |
| `cerebras` | Cerebras | `openai-completions` | `CEREBRAS_API_KEY` |
| `cloudflare-ai-gateway` | Cloudflare AI Gateway | `anthropic-messages`、`openai-completions`、`openai-responses` | `CLOUDFLARE_API_KEY`、`CLOUDFLARE_ACCOUNT_ID` 和 `CLOUDFLARE_GATEWAY_ID` |
| `cloudflare-workers-ai` | Cloudflare Workers AI | `openai-completions` | `CLOUDFLARE_API_KEY` 和 `CLOUDFLARE_ACCOUNT_ID` |
| `deepseek` | DeepSeek | `openai-completions` | `DEEPSEEK_API_KEY` |
| `fireworks` | Fireworks | `anthropic-messages`、`openai-completions` | `FIREWORKS_API_KEY` |
| `github-copilot` | GitHub Copilot | `anthropic-messages`、`openai-completions`、`openai-responses` | 通过 `pig login github-copilot` 的 OAuth，或 `COPILOT_GITHUB_TOKEN` |
| `google` | Google Gemini | `google-generative-ai` | `GEMINI_API_KEY` |
| `google-vertex` | Google Vertex AI | `google-vertex` | `GOOGLE_CLOUD_API_KEY` 或 Application Default Credentials。见 [Google Vertex AI](#google-vertex-ai)。 |
| `groq` | Groq | `openai-completions` | `GROQ_API_KEY` |
| `huggingface` | Hugging Face | `openai-completions` | `HF_TOKEN` |
| `kimi-coding` | Kimi For Coding | `anthropic-messages` | `KIMI_API_KEY` 或 OAuth |
| `meta` | Meta | `openai-responses` | `META_API_KEY` 或 OAuth |
| `minimax` | MiniMax | `anthropic-messages` | `MINIMAX_API_KEY` |
| `minimax-cn` | MiniMax（中国） | `anthropic-messages` | `MINIMAX_CN_API_KEY` |
| `mistral` | Mistral | `mistral-conversations` | `MISTRAL_API_KEY` |
| `moonshotai` | Moonshot AI | `openai-completions` | `MOONSHOT_API_KEY` |
| `moonshotai-cn` | Moonshot AI（中国） | `openai-completions` | `MOONSHOT_API_KEY` |
| `nvidia` | NVIDIA NIM | `openai-completions` | `NVIDIA_API_KEY` |
| `openai` | OpenAI | `openai-responses` | `OPENAI_API_KEY` |
| `openai-codex` | OpenAI Codex（ChatGPT 订阅） | `openai-codex-responses` | 通过 `pig login openai-codex` 的 OAuth |
| `opencode` | OpenCode Zen | `anthropic-messages`、`google-generative-ai`、`openai-completions`、`openai-responses` | `OPENCODE_API_KEY` |
| `opencode-go` | OpenCode Go | `anthropic-messages`、`openai-completions`、`openai-responses` | `OPENCODE_API_KEY` |
| `openrouter` | OpenRouter | `anthropic-messages`、`openai-completions` | `OPENROUTER_API_KEY` 或 OAuth |
| `qwen-token-plan` | Qwen Token Plan | `openai-completions` | `QWEN_TOKEN_PLAN_API_KEY` |
| `qwen-token-plan-cn` | Qwen Token Plan（中国） | `openai-completions` | `QWEN_TOKEN_PLAN_CN_API_KEY` |
| `qwen-token-plan-individual` | Qwen Token Plan（个人） | `openai-completions` | `QWEN_TOKEN_PLAN_API_KEY` |
| `radius` | Radius | `pi-messages` | `RADIUS_API_KEY` 或 OAuth。见 [Radius](#radius)。 |
| `together` | Together AI | `openai-completions` | `TOGETHER_API_KEY` |
| `vercel-ai-gateway` | Vercel AI Gateway | `anthropic-messages` | `AI_GATEWAY_API_KEY` |
| `xai` | xAI | `openai-responses` | `XAI_API_KEY` 或 OAuth |
| `xiaomi` | Xiaomi MiMo | `openai-completions` | `XIAOMI_API_KEY` |
| `xiaomi-token-plan-ams` | Xiaomi MiMo Token Plan（阿姆斯特丹） | `openai-completions` | `XIAOMI_TOKEN_PLAN_AMS_API_KEY` |
| `xiaomi-token-plan-cn` | Xiaomi MiMo Token Plan（中国） | `openai-completions` | `XIAOMI_TOKEN_PLAN_CN_API_KEY` |
| `xiaomi-token-plan-sgp` | Xiaomi MiMo Token Plan（新加坡） | `openai-completions` | `XIAOMI_TOKEN_PLAN_SGP_API_KEY` |
| `zai` | ZAI Coding Plan（全球） | `openai-completions` | `ZAI_API_KEY` |
| `zai-coding-cn` | ZAI Coding Plan（中国） | `openai-completions` | `ZAI_CODING_CN_API_KEY` |

`pig --list-models` 只显示已有凭据的 Provider。`pig --list-models <search>` 过滤该列表。

要加入本表之外的端点，例如本地 Ollama、LM Studio 或 vLLM，请在 `~/.pig/agent/models.json` 中声明。见 [自定义 Provider](/docs/latest/custom-provider)。

## 从环境变量使用 API key

启动 PiG 前设置对应变量：

```bash
export GEMINI_API_KEY=...
pig --model google/gemini-2.5-flash
```

上表每个 Provider 读取其凭据列里的变量。`google` 使用 `GEMINI_API_KEY`。PiG 不把 `GOOGLE_API_KEY` 当作 `google` 的凭据，`pig auth check`、`/login` 和 `--list-models` 都会忽略它，与 Pi 一致。

`anthropic` 读取三个变量。`ANTHROPIC_AUTH_TOKEN` 作为 `Authorization: Bearer` 令牌发送，优先级最高。`ANTHROPIC_OAUTH_TOKEN` 当作 API key 使用，优先于 `ANTHROPIC_API_KEY`。订阅令牌（`sk-ant-oat`）以 Claude Code 身份发送，使用订阅认证时会在会话启动时显示警告。

### Anthropic 工作负载身份联邦

没有 key、已存凭据、`ANTHROPIC_AUTH_TOKEN` 或授权头时，`anthropic` 可以用工作负载身份联邦认证。设置 `ANTHROPIC_FEDERATION_RULE_ID`、`ANTHROPIC_ORGANIZATION_ID` 和 `ANTHROPIC_IDENTITY_TOKEN_FILE`。`ANTHROPIC_SERVICE_ACCOUNT_ID` 和 `ANTHROPIC_WORKSPACE_ID` 可选。

PiG 每次交换都从文件重新读取身份令牌（JWT），因此轮换后的令牌会被采用。它用 `jwt-bearer` 授予在 `<base URL>/v1/oauth/token` 换取访问令牌，再以 `Authorization: Bearer` 发送。PiG 保留访问令牌并在过期前刷新；遇到 401 后下一次请求重新交换。令牌端点必须使用 `https`，只有 `localhost`、`127.0.0.1` 和 `::1` 允许 `http`。超过 16 KiB 的身份令牌在请求前被拒绝。交换失败就是该请求的错误，不会发送消息请求。错误响应体只保留 `error`、`error_description` 和 `error_uri` 字段。

联邦仅适用于 `anthropic` Provider。其他 `anthropic-messages` Provider 仍需 key 或授权头。

Anthropic 因语法体积、工具数量或联合类型限制拒绝请求的严格工具集时，PiG 会把所有 `strict: "prefer"` 工具降为非严格再发一次。`strict: "require"` 的工具仍保留错误。Pi 1.4.0 行为相同；更早的版本会让每一轮都失败（[Pi #10409](https://github.com/earendil-works/pi/issues/10409)）。

Anthropic Messages 请求与 Pi 1.4.0 有两处不同：

- 代理把 CRLF 拆到两个网络块时，不会提前结束行。Pi 会以 `Could not parse Anthropic SSE event message_start` 中止该轮（D121，Pi #10390）。
- 显式的 `anthropic-beta` 头（来自模型的 `headers` 或请求）不再让托管 effort 请求以 `Extra inputs are not permitted` 失败：PiG 只在对应 beta 存在时才发送该托管字段（D130，Pi #10759）。

`github-copilot` 只读 `COPILOT_GITHUB_TOKEN`。通用的 `GITHUB_TOKEN` 不是 Copilot 凭据。

`PI_CACHE_RETENTION` 设置 PiG 传给 Provider 的提示缓存保留策略。

Mistral 请求在带会话 ID 且缓存保留不为 `none` 时会包含 `prompt_cache_key` 和自动的 `x-affinity` 头。显式的 affinity 头保持不变，包括空头。重放的 assistant 消息使用 `prefix: false`，它们是已完成的轮次，不是补全前缀。请求超时只覆盖等待响应头，所以长流不会在 60 秒后被中止。没有按时收到响应头的请求以 `Mistral response headers timed out after <N>ms` 失败。Pi 1.4.0 行为相同；更早的版本把超时套在整条请求上（[Pi #10609](https://github.com/earendil-works/pi/issues/10609)）。

Google Gemini 与 Vertex 请求把系统指令编码成 user 角色内容，与 Pi 的 Google SDK 一致。没有显式选择或严格工具 schema 要求时，它们省略工具调用模式。显式的 `none` 和 `any` 优先于严格的 `VALIDATED` 模式。对 Gemini 2.x 模型，PiG 把一连串工具结果的图片放在该串最后一个 function response 之后，使同一模型轮次的所有 function response 留在同一个 user 轮次里。Pi 把每张图片紧跟在对应结果之后，Gemini 2.x 在并行工具调用后会拒绝这种请求（D115）。

## 认证

Provider 认证可能需要多次应答。Bedrock 支持 bearer token、AWS profile 或现有 AWS 凭据链。Vertex 支持 API key、Application Default Credentials 或 service-account 文件，并提示项目和位置。Cloudflare 会要求账户 ID，AI Gateway 还要在 key 之后要求 gateway ID。Provider 作用域的设置存进凭据的 `env` 对象，不写进聊天记录。

API key 按输入原样保存，前后空格保留。显式保存为空的 Cloudflare key 不会回退到 `CLOUDFLARE_API_KEY`。空的 Vertex key 不会选中 `GOOGLE_CLOUD_API_KEY`；Vertex 仍会检查其云凭据。用 `/logout` 删除已存凭据并回到仅环境变量认证。Provider 搜索包含认证方法名。没有凭据的 Provider 标记为 `not configured`；其他认证类型的凭据在 Provider 列表里标记为 `API key configured`、`subscription configured` 或 `account configured`。基于订阅的 OAuth 登录标记为 `subscription`；Radius 和 OpenRouter 等其他 OAuth 登录标记为 `account`。

PiG 把凭据存放在 `~/.pig/agent/auth.json`。Agent 启动时若文件不存在，会创建内容为 `{}`、仅所有者可访问的文件（POSIX 模式 `0600`；Windows 为仅所有者 DACL，D68）。启动不会改动已存在文件的内容和权限。已存凭据优先于环境变量回退；运行时 API key 可以为当前进程覆盖它们。

`auth.json` 可能包含 API key 和 OAuth 令牌。请保持私密，不要提交。

标记为 secret 的交互式登录输入使用 PiG 的 `maskSecretInput` 设置（默认 `true`）。`/settings` 里的 **Mask secret input** 会在输入时显示圆点、字符数和最后四个字符，提交后只保留掩码预览。少于五个字符的输入不显示后缀。把 `maskSecretInput` 设为 `false` 可恢复 Pi 1.4.0 的明文行为。这个可配置功能记录为差异 D80。普通文本和手动代码提示仍然可见。凭据仍应放在 `auth.json` 或 Provider 的凭据存储中；该设置保护的是对话框和认证诊断输出，不是凭据存储。

### 交互式登录与登出

用 Provider ID 或显示名运行 `/login <provider>` 配置该 Provider。参数补全会显示每个 Provider 支持的方法。只有一种方法时直接进入该流程；有多种方法时打开方法选择器；参数不匹配时打开可搜索的 Provider 列表。

不带参数运行 `/login` 先选择认证方法。最后一项 **Sign in with Radius** 直接开始 Radius 登录并显示状态；在其 Provider 列表按 Escape 回到方法选择器。在某个登录方法提示处（例如 Anthropic、Radius、Amazon Bedrock 或 Google Vertex AI 的提示），或关闭 Provider 的设置说明，按 Escape 回到启动登录的菜单。请继续回答登录对话框里的提示，直到它报告完成。在 key 或文本提示处按 Escape 会中止登录，已存凭据不变。Pi 把这种取消报告为 `Failed to save API key for <Name>: This operation was aborted`。

运行 `/logout` 删除已存的 API-key 或 OAuth 凭据。列表不含仅环境变量凭据。成功登出还会移除该 Provider 在当前运行中的 `--api-key` 覆盖。删除失败或被取消时，覆盖仍然可用。登出不改环境变量或 `models.json`。没有存凭据时，PiG 报告无可删除，而不是打开空选择器。

匹配的登录选项会报告凭据来源，例如 `stored` 或环境变量。登出列表把可删除的已存凭据标记为 `configured`。这些标签描述不同状态。

OpenAI 兼容的 HTTP 错误保留 Pi 的状态码和 Provider 错误体。补全错误直接使用状态码；Responses 错误带 Provider 的 API 错误前缀。

### 从命令加载 API key

想用密钥管理器又不把 key 写进磁盘，可把已存 key 设成以 `!` 开头的命令：

```json
{
  "anthropic": {
    "type": "api_key",
    "key": "!security find-generic-password -ws 'anthropic'"
  }
}
```

PiG 在第一次需要 key 时执行命令，并用其标准输出作为 key。

### 凭据里的 Provider 设置

已存的 API-key 凭据可以带 `env` 对象。其值在该 Provider 上优先于进程环境。模型运行时流式请求和扩展模型认证查询会把这些值带到 Provider 请求：

```json
{
  "cloudflare-workers-ai": {
    "type": "api_key",
    "key": "...",
    "env": {
      "CLOUDFLARE_ACCOUNT_ID": "account-id"
    }
  }
}
```

### OAuth Provider

在 Windows 上，PiG 把浏览器登录 URL 直接交给 Windows URL 处理器，不经过命令 shell。URL 参数和 shell 元字符仍是 URL 的一部分。

设备码登录显示验证 URL、用户码和等待状态，不打开浏览器；请自行打开显示的链接。浏览器授权 URL 事件仍会打开默认浏览器。

内置 OAuth 目标是 `anthropic`、`github-copilot`、`kimi-coding`、`meta`、`openai-codex`、`openrouter`、`radius` 和 `xai`。每个 Provider 拥有自己的流程。例如 GitHub Copilot 用设备授权；基于回调的 Provider 可以启动 localhost 回调服务器。Anthropic 的浏览器登录监听 `127.0.0.1:53692`，便于端口转发；端口无法绑定时使用空闲回环端口，登录 URL 和令牌交换会跟随实际绑定的端口。Sign in with ChatGPT 和 OpenAI Codex 登录向 OpenAI 标识 PiG（`PiG` 和 `pig`）；`models.json` 或请求上设置的 `originator` 或 `User-Agent` 模型头会替换 Codex 默认值。令牌会持久化到 `auth.json`，除非 Provider 拥有其他存储；支持的 Provider 会在需要时刷新。

Anthropic 登录会询问 **Browser login (default)** 或 **Copy code login (headless)**。浏览器登录在本机监听回调，也接受粘贴的重定向 URL。复制代码登录适用于浏览器在另一台机器的情况：通过打印的 URL 登录，再粘贴 Anthropic 显示的代码（`code#state`）。在登录对话框按 Escape 会报告 `Failed to login to Anthropic: This operation was aborted`。

授权成功或失败时，Anthropic 登录会取消并等待其上下文中的手动代码提示。`PI_OAUTH_CALLBACK_HOST` 选择回调监听主机；授权请求保留 localhost 重定向 URI。

Copilot 刷新失败会给出明确的重认证提示：

```
github-copilot: refresh failed - run 'pig login' to re-authenticate
```

### `pig login` 形式

```
pig login                 # interactive picker
pig login <provider>      # specific provider
pig login --list          # list the targets without reading credentials
pig logout <provider>     # remove credentials for one provider
```

`pig logout` 不会清除环境变量，也不会在 Provider 端吊销凭据。

### 凭据命令

`pig auth` 按模型请求的方式解析凭据，并把它们写给外部客户端。每条命令都需要 `--provider <provider>`、`--model <model>` 或两者。

```
pig auth check --provider openai --json      # ready / not_ready / invalid, exit 0 / 1 / 2
pig auth print-api-key --provider openai     # API key on stdout
pig auth print-bearer-token --provider openai-codex --min-expiry 1h
```

打印凭据的命令会把秘密写到标准输出。`auth check` 只有带 `--credentials` 时才打印凭据。

启动会话前，用 `pig auth check` 确认 PiG 能看到凭据。

#### 完整示例：从没有凭据到 ready

下面的输出都是 PiG 在全新主目录（`HOME` 和 `PIG_HOME` 都为空）且无网络时打印的。key 值都是虚构的。这些命令从不联系 Provider，所以只能证明 PiG 找到了 key，不能证明 OpenAI 接受它。

**1. 什么都没配置。**

```bash
pig auth check --provider openai --json; echo "exit status: $?"
```

```text
{"status":"not_ready","provider":"openai","reason":"credentials_not_configured"}
exit status: 1
```

首次运行时 PiG 创建了内容为 `{}` 的 `~/.pig/agent/auth.json`（模式 `-rw-------`），如本节开头的“认证”说明所述。

**2. 设置环境变量。**

```bash
OPENAI_API_KEY=sk-test-123 pig auth check --provider openai --json; echo "exit status: $?"
OPENAI_API_KEY=sk-test-123 pig auth check --provider openai
OPENAI_API_KEY=sk-test-123 pig auth print-api-key --provider openai
```

```text
{"status":"ready","provider":"openai","authType":"api_key"}
exit status: 0
ready
sk-test-123
```

不带 `--json` 的 `auth check` 只打印一个词，不打印 key。`print-api-key` 会把 key 打到标准输出，所以不要在会记录输出的地方运行它。

**3. 变量用错了。**

```bash
GOOGLE_API_KEY=x pig auth check --provider google; echo "exit status: $?"
```

```text
not_ready
exit status: 1
```

`google` 读取 `GEMINI_API_KEY`，不是 `GOOGLE_API_KEY`（见上文“从环境变量使用 API key”一节）。

**4. 已存 key 是命令，同时设置了环境变量 key。** 在 `~/.pig/agent/auth.json` 写入：

```json
{"openai":{"type":"api_key","key":"!echo key-from-command"}}
```

```bash
OPENAI_API_KEY=sk-env pig auth print-api-key --provider openai
pig auth check --provider openai --json --credentials
```

```text
key-from-command
{"status":"ready","provider":"openai","authType":"api_key","credentials":"key-from-command"}
```

已存凭据优先于 `OPENAI_API_KEY`，PiG 执行命令取得 key。`--credentials` 才会把 key 放进 JSON。

**5. 命令失败。** 使用 `"key":"!false"`：

```bash
pig auth print-api-key --provider openai; echo "exit status: $?"
pig auth check --provider openai --json
```

```text
Error: No usable API key is configured
exit status: 1
{"status":"not_ready","provider":"openai","reason":"credentials_not_configured"}
```

退出码非零的命令视为没有已存 key，PiG 不报告无效凭据。它随后检查环境：设置了 `OPENAI_API_KEY=sk-env` 时，`auth print-api-key` 打印 `sk-env`，`auth check` 报告 `ready`；没有设置时就是上面的输出。如果已存命令 key 显示 `not_ready`，请手工执行那条命令。

**6. 删除。**

```bash
pig logout openai; pig logout openai
```

```text
Logged out from openai. Credentials removed from auth.json
Logged out from openai. Credentials removed from auth.json
```

两次都打印同一行并以 0 退出，第二次也一样（当时没有存储）。`auth.json` 又变回 `{}`。`logout` 不碰 `OPENAI_API_KEY`。

**7. `invalid`。** `invalid`（退出状态 2）表示 PiG 连凭据都无法评估，例如 `models.json` 坏了：

```bash
echo '{ broken' > ~/.pig/agent/models.json
pig auth check --provider openai --json; echo "exit status: $?"
```

```text
{"status":"invalid","provider":"openai","reason":"invalid_state"}
exit status: 2
```

修复或删除 `models.json` 后重新检查。同时缺 `--provider` 和 `--model` 是用法错误：`Error: Auth checks require --provider <provider> or --model <model>`，退出状态 2。未知 Provider 是 `{"status":"not_ready","provider":"nosuch","reason":"provider_not_found"}`，退出状态 1（`coding/cli/auth_check.go:62-77`）。

OAuth 登录可能等待设备授权或浏览器回调。这段等待属于 Provider 流程。`pig login` 运行期间，PiG 不会启动模型会话。

## 云平台 Provider

下面的 Provider 需要多个设置，或可以使用云平台凭据。

### Azure OpenAI

设置 API key，再设置 base URL 或资源名：

```bash
export AZURE_OPENAI_API_KEY=...
export AZURE_OPENAI_BASE_URL=https://your-resource.ai.azure.com
# Or:
export AZURE_OPENAI_RESOURCE_NAME=your-resource
```

| 变量 | 作用 |
|---|---|
| `AZURE_OPENAI_API_KEY` | API key。 |
| `AZURE_OPENAI_BASE_URL` | Azure OpenAI 或 Foundry 端点。`.openai.azure.com`、`.cognitiveservices.azure.com` 和 `.ai.azure.com` 主机会规范化为 `/openai/v1`。 |
| `AZURE_OPENAI_RESOURCE_NAME` | `AZURE_OPENAI_BASE_URL` 的替代，构造 `https://<resource>.openai.azure.com/openai/v1`。 |
| `AZURE_OPENAI_API_VERSION` | API 版本，默认 `v1`。 |
| `AZURE_OPENAI_DEPLOYMENT_NAME_MAP` | 可选的逗号分隔 `model=deployment` 映射，适用于两种 API。 |

`azure` Provider 提供 OpenAI Responses API；对 Foundry 部署还提供 Chat Completions。内置的 `azure/deepseek-v4-pro` 使用 Chat Completions。其他 Foundry 模型请在 `models.json` 里加到 `azure` Provider 下，并在 Provider 或模型上写 `"api": "openai-completions"` 与 `"baseUrl"`，因为内置 Azure 模型没有 baseUrl。没有会以 `"baseUrl" is required when defining custom models.` 拒绝该条目。

Provider ID 在更早的版本里是 `azure-openai-responses`。API ID 和 `AZURE_OPENAI_*` 变量未变。PiG 不迁移旧条目。请重命名 `auth.json`（或重新 `/login`）、`models.json` 和 `settings.json` 里的 Provider 键（`defaultProvider`、`enabledModels` 模式、`modelThinkingLevels` 键）。使用旧 Provider 的会话恢复时会回退到另一个模型，其提示缓存也不会复用。

### Amazon Bedrock

Bedrock 使用 bearer token 或标准 AWS SDK 凭据链：

```bash
# Named profile
export AWS_PROFILE=your-profile

# IAM keys
export AWS_ACCESS_KEY_ID=AKIA...
export AWS_SECRET_ACCESS_KEY=...
export AWS_SESSION_TOKEN=...   # for temporary credentials

# Bedrock bearer token
export AWS_BEARER_TOKEN_BEDROCK=...

# Region, when the profile or SDK configuration does not supply one
export AWS_REGION=us-west-2    # AWS_DEFAULT_REGION also works
```

ECS 任务凭据和 IRSA 通过标准 `AWS_CONTAINER_CREDENTIALS_*` 和 `AWS_WEB_IDENTITY_TOKEN_FILE` 变量工作。

Claude Opus 5 使用自适应思考，并在 Bedrock 上支持原生 `xhigh` effort。PiG 识别应用推理配置的模型 ID 或显示名，并为支持的 Claude 模型添加提示缓存点。对 GovCloud 模型 ID、GovCloud ARN 或配置的 `us-gov-` 区域，PiG 省略 GovCloud schema 拒绝的 `thinking.display`。

Bedrock Converse 请求会把相邻的同角色消息合并成一条。因此工具执行中止后、或 assistant 轮次被丢弃后发出的提示，不会因角色交替错误而失败。Pi 1.4.0 会把两条 user 消息并排发送（D116）。

### Cloudflare AI Gateway

网关需要令牌、账户 ID 和 gateway ID：

```bash
export CLOUDFLARE_API_KEY=...
export CLOUDFLARE_ACCOUNT_ID=...
export CLOUDFLARE_GATEWAY_ID=...
```

账户和 gateway ID 可以来自进程环境，也可以来自 `auth.json` 中凭据的 `env` 对象。

### Cloudflare Workers AI

Workers AI 需要令牌和账户 ID：

```bash
export CLOUDFLARE_API_KEY=...
export CLOUDFLARE_ACCOUNT_ID=...
```

### Google Vertex AI

使用 Google Cloud API key：

```bash
export GOOGLE_CLOUD_API_KEY=...
```

改用 Application Default Credentials 时，设置项目和位置后登录：

```bash
export GOOGLE_CLOUD_PROJECT=your-project   # GCLOUD_PROJECT also works
export GOOGLE_CLOUD_LOCATION=us-central1
gcloud auth application-default login
```

使用 service-account key 文件时，设置 `GOOGLE_APPLICATION_CREDENTIALS` 以及项目和位置。

### Radius

Radius 是使用 Pi 自己消息协议 `pi-messages` 的网关：PiG 把对话 POST 到 `<baseUrl>/messages`，并把回复读成 Pi 事件流。`/login` → **Sign in with Radius**（或 **Sign in with an account** → **Radius**）提供浏览器登录（`127.0.0.1:1456` 上的回调）或用于另一台机器登录的设备码。也可以用 **Sign in with an API key** 粘贴 key，或设置 `RADIUS_API_KEY`。Radius 登录后，`/login` 会提议在全局 `mcp.json` 中用 `"auth": { "provider": "radius" }` 配置 Radius MCP 服务器；回答 **Yes** 会写入条目并执行 `/reload`。如果 Radius MCP URL 的全局服务器已经使用 Radius 登录，则不再询问。

在知道网关自己的列表之前，PiG 提供 Radius 公布的模型列表。配置凭据后，交互或 RPC 模式启动时、以及用 `/login` 登录后，PiG 会后台从 `<gateway>/v1/config` 获取网关当前列表，并缓存到 `~/.pig/agent/models-store.json`。拿到后，该列表替代公布列表，因此组织所有者禁用的模型不会出现。Print 模式和 `--list-models` 使用缓存列表。`PI_OFFLINE`（任意值）或 `PIG_OFFLINE`（`1`、`true`、`yes`）关闭获取。没有 Radius 凭据时，PiG 只在你登录期间联系网关。

要使用其他 Radius 网关，请在 `models.json` 里加一个带 `"oauth": "radius"` 的 Provider。`baseUrl` 必填；PiG 会去掉末尾的 `v1` 路径段来定位网关。公布的模型列表只适用于默认网关。

```json
{
  "providers": {
    "radius-dev": { "name": "Radius (dev)", "baseUrl": "http://localhost:8788", "oauth": "radius" }
  }
}
```

任何实现 `pi-messages` 的后端都可以作为 `models.json` 里的 Provider，配上 `"api": "pi-messages"`、`baseUrl` 和 `apiKey`。

### OpenRouter

用 `/login openrouter` 或 `OPENROUTER_API_KEY` 认证。

有 key 时，PiG 只列出该 key 可用的 OpenRouter 聊天模型，并带该 key 的上下文大小、最大输出和价格。这些来自 OpenRouter 的 `/models/user` 端点，反映 OpenRouter 中设置的护栏和 Provider 限制。图片和分类器模型不参与过滤。OpenRouter 列出但没有限额的模型沿用 PiG 目录的限额；路由模型（如 `openrouter/auto`，其价格被 OpenRouter 报为负数）价格为 0。

PiG 与其他模型目录一起刷新该列表。最后一次列表会保存并离线使用。在 `/login` 之外换 key 后，旧 key 的列表保留到下次刷新。刷新失败时保留已保存列表。

区域端点会按区域过滤模型。要使用区域端点，在 `models.json` 里设置 Provider 的 base URL：

```json
{
  "providers": {
    "openrouter": { "baseUrl": "https://us.openrouter.ai/api/v1" }
  }
}
```

如果 base URL 在 `/models/user` 拒绝该 key，或没有该端点（例如把代理配成 OpenRouter base URL），PiG 列出全部 OpenRouter 模型。

## 与 Pi 的流式与重放差异

这些差异涉及多个 Provider：

- Provider 流式给出没有 id 的工具调用时，在块结束时生成唯一的 `<name>_<epoch ms>_<counter>` id。重放会省略名称为空或 id 仍为空的工具调用及其结果。Pi 会重放它们，导致之后每一轮失败（D120，Pi #10299）。OpenAI Responses 中 `call_id` 为空的调用同样处理（D129）。
- 切换模型后，需要 `reasoning_content` 的推理型 Chat Completions 目标（DeepSeek、Kimi、MiMo 等）会把另一个模型的纯文本 thinking 作为 `reasoning_content` 收到。Pi 会把它追加到 `content`（D126，Pi #6167）。
- OpenRouter 为 Claude 提供的仅签名 `reasoning.text` 增量会保留，并随其 reasoning 块重放。Pi 会丢弃它（D122，Pi #9534）。
- `httpIdleTimeoutMs` 在每个流事件时重置，而不是每个字节。只发 keepalive 注释或 ping 的流会在超时后以错误结束。Pi 会无限等待（D131，Pi #10761、#10762）。
- 单条 server-sent-event 行最大 512 MiB，这是 Node 字符串长度给 Pi 的限制（D134）。
- 运行中途断开的响应流报告 `terminated (cause: ...)`。Pi 报告裸的 `terminated`（D144，Pi #10697）。

## Provider 解析

`--model`、`/model`、`Ctrl+P` 和扩展 `setModel()` 按 Pi 的方式解析模型标识。模型 ID 可以含斜杠，例如 `openrouter/openai/gpt-5.5`，循环保留完整 ID。Go 函数 `coding.BuildModel` 把没有 `/` 的标识读成 `openai/<id>`，所以从 Go 代码调用时请传带 Provider 的标识。

## Provider 扩展

Anthropic Messages 请求保留所选模型的推理能力、思考级别映射和输出限制。新提供的模型即使不在内置目录里，也可以使用 `compat.forceAdaptiveThinking`。

扩展可以在注册时加入推理 Provider。PiG 在替换已注册模型数据前校验传入的模型配置。注册 `streamSimple` 时要设置 `api`；新模型要在 Provider 或模型层提供 `api`。被拒绝的注册会保留现有注册表条目和原生 Provider 所有者。

宿主保留 Provider 配置直到注册表绑定。绑定按顺序应用，并报告每个失败条目及其所属扩展路径。之后的注册和移除立即生效。宿主管生命周期：扩展注册的 Provider 在关闭、重载失败或隔离分裂时自动注销。重载替换扩展、而新注册声明了同名 Provider 时，注册会跨替换保留，流式补全不会中断。

## 排错

| 现象 | 可能原因 | 修复 |
|---|---|---|
| `Missing bearer or basic authentication` | 环境里没有 API key，`auth.json` 里没有令牌 | `pig login <provider>` 或导出环境变量 |
| `pig auth check --provider google` 打印 `not_ready` | 只设置了 `GOOGLE_API_KEY` | 设置 `GEMINI_API_KEY` |
| `Bad credentials`（Copilot 401） | OAuth 令牌过期或被吊销 | `pig login github-copilot` |
| 模型选择器为空 | 没有 Provider 有有效认证 | 登录或设置环境变量；检查 `/login` |
| Go 辅助函数构建了错误的模型 | 把裸 ID 传给 `coding.BuildModel` | 传 `provider/model`；见 [模型](zh-models.md) |

参考实现见 [上游 Pi 的 Provider 文档](https://pi.dev/docs/latest/providers)。

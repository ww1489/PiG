# 模型

PiG 用**带 Provider 前缀的标识**指代模型：`provider/modelID`。`--model`、`/model`、`Ctrl+P` 循环和扩展的 `setModel()` 等选模型入口都接受这种形式。设置里的作用域模型条目也用这种形式。保存的默认值是例外：把 `defaultProvider` 设为提供方，把 `defaultModel` 设为不带前缀的模型 ID。带 Provider 前缀的 `defaultModel` 不会在启动时选中该模型。

和 Pi 一样，`--model` 和 `/model` 会在所有 Provider 中解析裸模型 ID。Go 函数 `coding.BuildModel` 不同：没有 `/` 的标识会被它读成 `openai/<id>`。调用 `BuildModel` 时始终传 `provider/id`。

> 本页是英文页 `models.md` 的中文译本，对应 PiG 提交 `c64bebd8`。英文页是规范来源；字段名、默认值和错误文本以英文页与当前源码为准。

## 选择模型

| 方式 | 效果 |
|---|---|
| `pig --model openai/gpt-5.5` | 覆盖本进程使用的模型 |
| `/model` 斜杠命令 | 打开选择器。回车为本次会话选择；`Ctrl+S` 选择并保存为默认 |
| `Ctrl+P` / `Shift+Ctrl+P` | 在作用域模型里向前/向后循环 |
| `/scoped-models` | 交互式编辑循环列表 |
| `--models a,b,c` | 只为本轮设置作用域，不持久化 |
| `setModel(spec)`（扩展 API） | 程序化切换。Go SDK 的 `Context.SetModel` 返回 `(applied, error)` |

选择器只列出已配置认证的 Provider 的模型。Google Gemini 使用 `GEMINI_API_KEY`，不是 `GOOGLE_API_KEY`。没有配置认证的 Provider 会被略去。

扩展可以加入“为每次请求挑选物理模型”的模型。见 [虚拟模型](virtual-models.md)。

## 连接本地模型

PiG 集成了 llama.cpp router。router 会发现 GGUF 文件并按需加载模型。`/llama` 命令管理 router，`/model` 从中选择模型。见 [llama.cpp](llama-cpp.md)。

Ollama、LM Studio、vLLM、SGLang 和其他兼容服务，请在 `models.json` 里配置兼容端点，做法见下文。

## 配置兼容端点

端点说的是 PiG 已支持的 API 时，使用 `~/.pig/agent/models.json`，具体做法见下一节。这覆盖大多数 Ollama、LM Studio、vLLM、SGLang 和代理部署：

```json
{
  "providers": {
    "ollama": {
      "baseUrl": "http://localhost:11434/v1",
      "api": "openai-completions",
      "apiKey": "ollama",
      "models": [
        { "id": "qwen2.5-coder:7b" }
      ]
    }
  }
}
```

占位 key 只是让模型可用；Ollama 不检查它。需要认证的端点，`apiKey` 和请求头值可以使用 `$NAME` 或 `${NAME}` 环境变量插值、字面值，或 `!command` 形式。[自定义 Provider](custom-provider.md) 有完整示例。

自定义模型必须在模型或它的 Provider 上有 `api`，除非它替换的是该 Provider 的内置模型。没有时会报 `Provider "<provider>": Provider <provider>, model <model>: no "api" specified. Set at provider or model level.`，并忽略该 Provider 的 `models.json` 条目，与 Pi 一致。PiG 0.4.x 曾给 `api` 设默认值。`contextWindow` 或 `maxTokens` 为零或负数的模型也会被同样拒绝（`Provider <provider>, model <model>: invalid contextWindow`）。

`models` 条目会新增模型，或替换该 Provider 上同 ID 的模型。想改内置或扩展提供模型的元数据、又不想替换 Provider 的模型列表，用 `modelOverrides`。未知的覆盖 ID 会被忽略。

### 描述模型输入与缓存

用 `inputLimits.images.resize` 控制 PiG 在把新图片附件、`read` 结果和工具结果图片存入对话历史之前如何编码：

```json
{
  "id": "vision-model",
  "input": ["text", "image"],
  "inputLimits": {
    "images": {
      "resize": {
        "maxWidth": 1568,
        "maxHeight": 1568,
        "maxBytes": 524288,
        "jpegQuality": 75
      }
    }
  }
}
```

`maxBytes` 限制 base64 编码后的载荷。省略的缩放字段使用默认值：2000×2000 像素、编码后 4.5 MiB、JPEG 质量 80。图片只编码一次；换模型不会重写历史图片。

`inputLimits.images.maxPerRequest` 限制单次请求的图片数量。请求超限时，PiG 把最旧的图片替换为文本 `[image omitted: this model accepts at most N images per request]` 并保留最新的 N 张，已保存的会话不变。Pi 会读取该限制却不执行，于是 Provider 拒绝请求（D137，Pi #10162）。带限制的目录模型（例如 Anthropic 的）无需配置就会受影响。一次运行超过限制后，每个新增图片的请求都会移动被省略的集合，因此 Provider 的提示缓从最旧保留图片处开始失效。

用 `promptCache` 声明 Provider 在 `short` 或 `long` 保留档位下的缓存寿命（秒）：

```json
{ "id": "claude-sonnet-5", "promptCache": { "short": 300, "long": 3600 } }
```

活动档位没有寿命的模型不参与缓存预热。`modelOverrides` 条目可以为内置或扩展模型设置 `inputLimits` 或 `promptCache`。见 [缓存预热](settings.md#cache-warming)。

### 按思考级别配置采样

OpenAI 兼容 API 支持 `models.json` 里的自由格式 `samplingParams` 默认值和 `samplingParamsByThinkingLevel` 覆盖。`samplingParamsByThinkingLevel` 的键是 PiG 的思考级别（`off`、`minimal`、`low`、`medium`、`high`、`xhigh`、`max`），不是 `thinkingLevelMap` 里的 Provider 值：

```json
{
  "id": "qwen-thinking-model",
  "reasoning": true,
  "samplingParams": {
    "temperature": 1.0,
    "top_p": 0.95
  },
  "samplingParamsByThinkingLevel": {
    "off": {
      "temperature": 0.7,
      "top_p": 0.8
    },
    "high": {
      "top_k": 20
    }
  }
}
```

PiG 先把不支持的思考级别收敛到支持的级别，然后依次合并模型的 `samplingParams`、生效级别的条目和请求的 `samplingParams`；同一个键后者覆盖前者。没有条目的级别使用模型默认值。`modelOverrides` 条目会按级别逐键与基础模型的条目合并。这些字段只对 `openai-completions`、`openai-responses` 和 `azure-openai-responses` 生效，其他 API 忽略。

## 作用域模型

作用域模型就是 `Ctrl+P` 循环的列表。它们在设置里以带 Provider 的 ID 保存；PiG 接受旧的裸 ID，并在下次保存时改写。循环同时保留 `provider` 和 `id`，包括含斜杠的 `id`，例如 `openrouter/openai/gpt-5.5`。

## 思考级别

PiG 用这些命名级别表达推理强度：

```text
off → minimal → low → medium → high → xhigh → max
```

模型只暴露它支持的级别，循环也按支持集合进行。

交互模式打开前，Session 会选定并收敛启动级别。显式 `--thinking` 优先。`--model provider/model:level` 中的思考后缀也会在交互启动前到达 Session。否则，恢复的 Session 保留已保存级别；新 Session 使用模型偏好、全局默认值或 `medium`。不支持的级别向上取到支持级别；没有更高的就用最高支持级别。页脚和编辑器显示 Session 的生效级别，Provider 请求通过模型的思考级别映射使用同一级别。打开交互模式不会用设置重置它。

- 在声明支持推理的模型上，`Shift+Tab` 循环思考级别。
- 扩展可以使用 `getThinkingLevel()` 和 `setThinkingLevel(level)`。
- 当前标签渲染在状态行；扩展可以用 `ui.setHiddenThinkingLabel` 覆盖隐藏标签。
- 不支持推理的模型会静默忽略变化，宿主不报错。

目录条目设置 `reasoning: true` 时，模型才具备推理能力；`thinkingLevelMap` 决定每个级别怎样映射进请求。

## 模型元数据

扩展通过 `ctx.model`（Node）或 `Context.GetModelInfo()`（Go SDK，返回 `(*ModelInfo, error)`）读取活动模型。Go SDK 的 `ModelInfo` 包含 `ID`、`Name`、`Provider`、`ContextWindow`、`MaxOutputTokens`、`Reasoning`、`InputLimits`，以及每百万 token 成本 `InputCostPer1M`、`OutputCostPer1M`、`CacheReadCostPer1M`、`CacheWriteCostPer1M`。

## 使用分类器模型

分类器模型不聊天。它们回答关于 JSON 状态的带类型问题：从若干选项里选一个、回答是或否、给出分数，并附带概率。内置目录包含来自这些 Provider 的 TypeSafe Jev、Workers AI 的 Cloudflare Clef 与 Clef Flash，以及通过 [Decisions API](https://developers.openai.com/api/docs/guides/decisions) 提供的 OpenAI GPT-6 Luna：

| Provider | 模型 ID | 认证 |
|---|---|---|
| `typesafe` | `jev-latest` | `TYPESAFE_API_KEY` |
| `openrouter` | `typesafe/jev-1.13`、`~typesafe/jev-latest` | `OPENROUTER_API_KEY` 或 `/login` |
| `cloudflare-workers-ai` | `typesafe/jev`、`@cf/cloudflare/clef`、`@cf/cloudflare/clef-flash` | `CLOUDFLARE_API_KEY` 和 `CLOUDFLARE_ACCOUNT_ID` |
| `vercel-ai-gateway` | `typesafe-ai/jev` | `AI_GATEWAY_API_KEY` |
| `opencode` | `jev-1.13`、`jev-1.13-free` | `OPENCODE_API_KEY` |
| `openai` | `gpt-6-luna` | `OPENAI_API_KEY` |

[llama.cpp router](llama-cpp.md#classification) 上的聊天模型也会被列为分类器模型。

OpenAI 的 Decisions API 需要 API key。ChatGPT 凭据登录无法使用它，所以当 `openai` 通过 `/login` 登录时，即使设置了 `OPENAI_API_KEY`，`gpt-6-luna` 也不会列为可用；要使用该 key，请先退出 `openai` 登录。GPT-6 Luna 还会判断通过 `images` 传入的图片（见 [Codemode](codemode.md#classify)）；其他分类器模型对图片返回错误。API 拒绝超过 922K token 的输入，但运行超过约五秒的请求（目前约 600K 输入 token 以上）会以网关超时失败。

分类器模型不出现在 `/model` 里。模型通过 [`codemode`](codemode.md) 工具使用它们，该工具默认关闭，除非 MCP 服务器打开。用[设置](settings.md#tools)里的 `"defaultTools": ["+codemode"]` 启用。脚本用 `models.getAvailableOfType("classifier")` 列出分类器模型，用 `models.classify(model, { state, questions })` 调用。[Codemode](codemode.md#classify) 说明问题与答案类型。

PiG 把脚本分类器调用的用量加到 `codemode` 工具结果里，因此计入会话成本。成本使用模型目录价格；没有价格的模型（例如 TypeSafe 直接的 `jev-latest`）只报告 token，不计费用。

扩展通过 `ctx.modelRegistry.classify()` 调用分类器，不需要 codemode。[虚拟模型](virtual-models.md) 可以用它们路由请求。

## 使用图片模型

图片模型根据提示和可选输入图片生成图片。PiG 把 OpenRouter 的图片模型（如 `google/gemini-2.5-flash-image`、`black-forest-labs/flux.2-pro`）列在 `openrouter` Provider 下，使用与聊天模型相同的 `OPENROUTER_API_KEY` 或 `/login` 凭据。OpenRouter 只通过它的 Images API 提供部分图片模型，例如 `openai/gpt-image-2.5-flare`。当 `chat/completions` 拒绝这类模型时，PiG 会再向 Images API 发送一次；Pi 把拒绝当作错误返回（D128，Pi #10652）。

图片模型不出现在 `/model` 里，模型通过 `codemode` 工具使用它们。脚本用 `models.getAvailableOfType("image")` 列出，用 `models.generateImages(model, { input })` 调用。结果的 `output` 含 base64 图片块，`image()` 把它们附到 `codemode` 结果上，让模型看到：

```js
const painter = await models.getModelOfType("image", "openrouter", "google/gemini-2.5-flash-image");
const result = await models.generateImages(painter, {
  input: [{ type: "text", text: "A red fox in the snow, watercolor" }],
});
if (result.stopReason !== "stop") return result.errorMessage;
for (const block of result.output) if (block.type === "image") image(block);
```

`input` 也可以包含 `{ type: "image", data, mimeType }` 块，用作编辑或参考。与分类器调用一样，PiG 把脚本图片调用的用量加到 `codemode` 工具结果。生成的图片不写入磁盘。[Codemode](codemode.md#generate-images) 说明完整 API。

扩展通过 `ctx.modelRegistry.generateImages()` 生成图片，不需要 codemode。

## 添加自定义 Provider

新的内置模型由 Pi 的目录生成到 `ai/models_generated.go`。Provider 需要自定义流式处理、模型发现或认证时，用扩展。见 [自定义 Provider](custom-provider.md) 与 [Providers](zh-providers.md)。

## Provider 成本档位

扩展注册的模型可以包含 `cost.tiers`。每个档位提供 `inputTokensAbove`，以及每百万 token 的 `input`、`output`、`cacheRead`、`cacheWrite` 费率。PiG 在 Provider 注册和模型查找过程中保留这些阈值与费率。成本计算根据总输入用量选择适用档位。

## 常见错误

模型配置读取、JSON 解析和 schema 错误会带一行 `File:`，指出选中的 `models.json` 路径。检查该文件的内容和权限。文件不存在是允许的。PiG 不会改动无效文件。

| 现象 | 原因 | 修复 |
|---|---|---|
| `defaultModel` 写成 `github-copilot/gpt-6-sol`，新会话却用了别的模型 | 保存的默认值要求 Provider 与模型分开写 | 设 `"defaultProvider": "github-copilot"` 和 `"defaultModel": "gpt-6-sol"` |
| 报 `no "api" specified. Set at provider or model level.` | `models.json` 里的自定义模型没有 `api` | 在 Provider 或模型上补 `"api"` |
| Go 代码里的模型被路由到 `openai` | 把裸 ID 传给了 `coding.BuildModel()` | 传 `provider/id` |
| `/model` 里看不到自定义模型 | 它的 Provider 没有可用凭据 | 配置 `apiKey` 或登录；llama.cpp 需先加载模型 |
| 思考级别循环没有效果 | 活动模型不支持推理 | 换用支持推理的模型 |
| `/login` 之后模型选择器为空 | 凭据写到了另一个配置根 | 检查 `PIG_HOME` 与 `~/.pig` |

参考实现见 [上游 Pi 的模型文档](https://pi.dev/docs/latest/models)。

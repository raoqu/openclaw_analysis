## 1) LLM 模型调用适配：核心入口在哪里？

这仓库里“LLM 调用”不是散落在各个 channel，而是**收敛到一个统一的 agent 执行链**，然后在底层根据 `provider/model` 走两类执行器：

- **Embedded Pi Agent（主路径）**：通过 `@mariozechner/pi-coding-agent` + `@mariozechner/pi-ai` 进行多 provider 的模型调用、工具调用、session 管理。
- **CLI Provider（旁路）**：如 `claude-cli`/`codex-cli` 或 `agents.defaults.cliBackends` 配置的 CLI 后端，走 `runCliAgent()`。

### 1.1 消息/网关/CLI 进入 agent 的统一入口

- **消息自动回复入口**
  - [src/auto-reply/reply/get-reply.ts](cci:7://src/auto-reply/reply/get-reply.ts:0:0-0:0)：把 inbound message 规范化、解析指令、解析模型选择，最后进入 [runPreparedReply()](cci:1://src/auto-reply/reply/get-reply-run.ts:110:0-462:1)
  - [src/auto-reply/reply/get-reply-run.ts](cci:7://src/auto-reply/reply/get-reply-run.ts:0:0-0:0)：组装 `FollowupRun`（包含 `provider/model/authProfileId/...`）并调用 [runReplyAgent()](cci:1://src/auto-reply/reply/agent-runner.ts:45:0-528:1)
  - [src/auto-reply/reply/agent-runner.ts](cci:7://src/auto-reply/reply/agent-runner.ts:0:0-0:0)：队列/steer/followup、typing、usage、payload 拼装
  - [src/auto-reply/reply/agent-runner-execution.ts](cci:7://src/auto-reply/reply/agent-runner-execution.ts:0:0-0:0)：**真正执行 LLM 的地方**（见下）

- **OpenAI 兼容网关入口**
  - [src/gateway/openai-http.ts](cci:7://src/gateway/openai-http.ts:0:0-0:0)：处理 `/v1/chat/completions`，把 `messages[]` 转为 OpenClaw prompt，然后调用 [agentCommand(...)](cci:1://src/commands/agent.ts:63:0-528:1)

- **CLI 入口（openclaw agent ...)**
  - [src/commands/agent.ts](cci:7://src/commands/agent.ts:0:0-0:0)：解析 session、模型、fallback、再调用 [runEmbeddedPiAgent()](cci:1://src/agents/pi-embedded-runner/run.ts:161:0-945:1) 或 `runCliAgent()`

### 1.2 真正发起 LLM 请求的入口（最关键）

- [src/auto-reply/reply/agent-runner-execution.ts](cci:7://src/auto-reply/reply/agent-runner-execution.ts:0:0-0:0)
  - [runAgentTurnWithFallback()](cci:1://src/auto-reply/reply/agent-runner-execution.ts:54:0-629:1) 内部：
    - 先走 [runWithModelFallback(...)](cci:1://src/agents/model-fallback.ts:208:0-320:1)
    - 对每个候选 `(provider, model)`：
      - **CLI provider**：`runCliAgent(...)`
      - **普通 provider**：[runEmbeddedPiAgent(...)](cci:1://src/agents/pi-embedded-runner/run.ts:161:0-945:1)

- [src/agents/pi-embedded-runner/run.ts](cci:7://src/agents/pi-embedded-runner/run.ts:0:0-0:0)
  - [runEmbeddedPiAgent(...)](cci:1://src/agents/pi-embedded-runner/run.ts:161:0-945:1)：模型解析、auth profile 选择/轮询、错误分类与 failover、compaction、最终调用 [runEmbeddedAttempt(...)](cci:1://src/agents/pi-embedded-runner/run/attempt.ts:206:0-1062:1)

- [src/agents/pi-embedded-runner/run/attempt.ts](cci:7://src/agents/pi-embedded-runner/run/attempt.ts:0:0-0:0)
  - [runEmbeddedAttempt(...)](cci:1://src/agents/pi-embedded-runner/run/attempt.ts:206:0-1062:1)：创建 pi-coding-agent session，注入系统 prompt / tools / hooks，然后通过 `streamSimple`（或 Ollama 原生 streamFn）把请求真正发到模型端。

结论：**你要找“LLM 调用适配”的权威入口**，优先看：
- [runAgentTurnWithFallback()](cci:1://src/auto-reply/reply/agent-runner-execution.ts:54:0-629:1)（上层策略：fallback、streaming 回调）
- [runEmbeddedPiAgent()](cci:1://src/agents/pi-embedded-runner/run.ts:161:0-945:1)（中层策略：auth profile、错误->failover、compaction）
- [runEmbeddedAttempt()](cci:1://src/agents/pi-embedded-runner/run/attempt.ts:206:0-1062:1)（底层：具体 SDK 如何拼 session/prompt/tools 并调用模型）

---

## 2) Provider / Model 是怎么被选择与解析的？

### 2.1 默认模型、allowlist、alias
- [src/agents/model-selection.ts](cci:7://src/agents/model-selection.ts:0:0-0:0)
  - [parseModelRef()](cci:1://src/agents/model-selection.ts:80:0-99:1)：支持 `provider/model` 或仅 `model`（默认 provider）
  - [normalizeProviderId()](cci:1://src/agents/model-selection.ts:32:0-47:1)：有别名归一（比如 `z.ai`→`zai`）
  - [buildModelAliasIndex()](cci:1://src/agents/model-selection.ts:128:0-154:1)：从 `agents.defaults.models` 里读 `alias`
  - [buildAllowedModelSet()](cci:1://src/agents/model-selection.ts:255:0-328:1)：如果配置了 allowlist（`agents.defaults.models` 非空），只允许目录/配置/CLI provider 内的模型

### 2.2 Session 级覆盖（/model、存储 override）
- [src/auto-reply/reply/model-selection.ts](cci:7://src/auto-reply/reply/model-selection.ts:0:0-0:0)：auto-reply 场景的更完整逻辑（stored override / fuzzy alias / allowlist 限制）
- [src/sessions/model-overrides.ts](cci:7://src/sessions/model-overrides.ts:0:0-0:0)：写入 session store 的 override 辅助（你在 [agent.ts](cci:7://src/commands/agent.ts:0:0-0:0) 里也能看到应用逻辑）

### 2.3 Fallback（模型降级/切换）
- [src/agents/model-fallback.ts](cci:7://src/agents/model-fallback.ts:0:0-0:0)
  - [runWithModelFallback()](cci:1://src/agents/model-fallback.ts:208:0-320:1)：把 primary + `agents.defaults.model.fallbacks`（或 per-agent override）变成候选序列
  - 会尊重 allowlist、alias，并跳过 cooldown 的 auth profiles

---

## 3) “模型适配/扩展”的方式有哪些？

这里有两条路线，适用场景不同：

### 3.1 方式 A：通过 `models.providers` 增加/适配一个 provider（推荐的“模型调用适配”方式）

这是 OpenClaw **对 Pi SDK 的适配层**：最终会把 config 归一化写进 `~/.openclaw/agents/<agent>/models.json`，供 `ModelRegistry` 读取。

关键文件：
- **配置类型**
  - [src/config/types.models.ts](cci:7://src/config/types.models.ts:0:0-0:0)
    - [ModelApi](cci:2://src/config/types.models.ts:0:0-7:13) 目前支持：
      - `openai-completions`
      - `openai-responses`
      - `anthropic-messages`
      - `google-generative-ai`
      - `github-copilot`
      - `bedrock-converse-stream`
      - `ollama`
    - `ModelsConfig.providers[providerId] = ModelProviderConfig`
- **写出 models.json 的入口**
  - [src/agents/models-config.ts](cci:7://src/agents/models-config.ts:0:0-0:0)：[ensureOpenClawModelsJson(...)](cci:1://src/agents/models-config.ts:80:0-143:1)
- **隐式 provider（根据 env/auth profiles 自动注入）**
  - [src/agents/models-config.providers.ts](cci:7://src/agents/models-config.providers.ts:0:0-0:0)：[resolveImplicitProviders()](cci:1://src/agents/models-config.providers.ts:658:0-810:1)（例如 ollama、vllm、together、huggingface、bedrock 等）

你要“新增一个 provider 的适配”，通常就是：
- **最小步骤**
  - **[1]** 在 `OpenClawConfig` 里写 `models.providers.<yourProvider>`（baseUrl、api、auth、headers、models[]）
  - **[2]** 确保 auth 能解析（env / auth profiles / models.providers.apiKey）
  - **[3]** [ensureOpenClawModelsJson()](cci:1://src/agents/models-config.ts:80:0-143:1) 会把它写入 `models.json`，随后 embedded runner 用 `ModelRegistry.find(provider, modelId)` 找到它

适合：
- 你的 provider 本质是 OpenAI/Anthropic/Gemini/Bedrock/Ollama 这几类协议之一
- 只需要“换 baseUrl/headers/模型列表/鉴权方式”，不需要做 CLI/OAuth wizard 的 UX

### 3.2 方式 B：通过插件扩展 provider（ProviderPlugin）

这是“**扩展认证/配置体验**”的路线：插件能注册 provider，用来支持 `openclaw models auth login` 这类交互式登录流程，并可附带写 config patch / 默认模型。

关键文件：
- [src/plugins/types.ts](cci:7://src/plugins/types.ts:0:0-0:0)
  - [ProviderPlugin](cci:2://src/plugins/types.ts:114:0-124:2)：
    - `id/label/docsPath/aliases/envVars`
    - `models?: ModelProviderConfig`（注意：这里允许 plugin 自带一段 models provider 配置）
    - `auth: ProviderAuthMethod[]`（核心：实现 OAuth/device_code/api_key/token 等登录流程）
- [src/plugins/registry.ts](cci:7://src/plugins/registry.ts:0:0-0:0)：`registerProvider()` 把 provider 放进 registry
- [src/plugins/providers.ts](cci:7://src/plugins/providers.ts:0:0-0:0)：`resolvePluginProviders()` 加载所有 plugins 并返回 providers
- [src/commands/models/auth.ts](cci:7://src/commands/models/auth.ts:0:0-0:0)：`modelsAuthLoginCommand()` 会列出这些 provider，让用户选择 auth method 并写入 auth profiles / config

适合：
- 你要做一个“可安装的 provider 插件”，包含 OAuth 登录、设备码登录、自动 patch config、提供 docs 链接等
- 你希望把 provider 支持放到 `extensions/*`（插件包）里，而不是改 core

注意：当前 **plugin provider 注册**主要用于“认证/配置”这条链（wizard/login），而真正运行时的模型调用仍是通过 embedded runner + `models.json`/`ModelRegistry` 来发现模型。插件可以通过 `ProviderPlugin.models` + `configPatch` 间接把 provider 配置写进 config，从而进入 `models.json`。

---

## 4) 一份“入口文件 + 关键类型 + 扩展步骤”导航清单

- **消息进入 LLM**
  - [src/auto-reply/reply/get-reply.ts](cci:7://src/auto-reply/reply/get-reply.ts:0:0-0:0)
  - [src/auto-reply/reply/get-reply-run.ts](cci:7://src/auto-reply/reply/get-reply-run.ts:0:0-0:0)
  - [src/auto-reply/reply/agent-runner.ts](cci:7://src/auto-reply/reply/agent-runner.ts:0:0-0:0)
  - [src/auto-reply/reply/agent-runner-execution.ts](cci:7://src/auto-reply/reply/agent-runner-execution.ts:0:0-0:0)  (**执行分发点**)

- **真正调用模型**
  - [src/agents/pi-embedded-runner/run.ts](cci:7://src/agents/pi-embedded-runner/run.ts:0:0-0:0) ([runEmbeddedPiAgent](cci:1://src/agents/pi-embedded-runner/run.ts:161:0-945:1))
  - [src/agents/pi-embedded-runner/run/attempt.ts](cci:7://src/agents/pi-embedded-runner/run/attempt.ts:0:0-0:0) ([runEmbeddedAttempt](cci:1://src/agents/pi-embedded-runner/run/attempt.ts:206:0-1062:1))
  - [src/agents/pi-embedded-runner/model.ts](cci:7://src/agents/pi-embedded-runner/model.ts:0:0-0:0) ([resolveModel](cci:1://src/agents/pi-embedded-runner/model.ts:58:0-117:1)：从 ModelRegistry/inline provider/forward-compat 解析模型)

- **模型配置 / provider 配置**
  - [src/config/types.models.ts](cci:7://src/config/types.models.ts:0:0-0:0)（[ModelApi](cci:2://src/config/types.models.ts:0:0-7:13)、[ModelProviderConfig](cci:2://src/config/types.models.ts:43:0-51:2)、[ModelsConfig](cci:2://src/agents/models-config.providers.ts:32:0-32:58)）
  - [src/agents/models-config.ts](cci:7://src/agents/models-config.ts:0:0-0:0)（写 `models.json` 的权威入口：[ensureOpenClawModelsJson](cci:1://src/agents/models-config.ts:80:0-143:1)）
  - [src/agents/models-config.providers.ts](cci:7://src/agents/models-config.providers.ts:0:0-0:0)（隐式 provider 自动注入/发现）
  - [src/agents/model-selection.ts](cci:7://src/agents/model-selection.ts:0:0-0:0)（alias/allowlist/默认模型解析）
  - [src/agents/model-fallback.ts](cci:7://src/agents/model-fallback.ts:0:0-0:0)（fallback 链）

- **插件扩展 provider（认证/配置 UX）**
  - [src/plugins/types.ts](cci:7://src/plugins/types.ts:0:0-0:0)（[ProviderPlugin](cci:2://src/plugins/types.ts:114:0-124:2)、[ProviderAuthMethod](cci:2://src/plugins/types.ts:106:0-112:2)）
  - [src/plugins/loader.ts](cci:7://src/plugins/loader.ts:0:0-0:0) / [src/plugins/registry.ts](cci:7://src/plugins/registry.ts:0:0-0:0) / [src/plugins/providers.ts](cci:7://src/plugins/providers.ts:0:0-0:0)
  - [src/commands/models/auth.ts](cci:7://src/commands/models/auth.ts:0:0-0:0)（`openclaw models auth login`）

---

## 你接下来想深入哪一种“扩展方式”？

- **[A]** 你要接一个新的“OpenAI/Anthropic 兼容”推理服务（只要 baseUrl + headers + model list）  
- **[B]** 你要做一个可安装插件（extensions）来提供 OAuth 登录、自动写配置、带 docs 链接  
- **[C]** 你要新增一种全新的 [ModelApi](cci:2://src/config/types.models.ts:0:0-7:13)（比如新协议），需要在 embedded runner/SDK 侧做适配

你选一个方向，我可以把“最小改动点/需要改哪些文件/怎么写测试（已有 e2e 模板）”进一步拆开到可执行步骤。

---

## 状态总结

- 已完成：
  - LLM 调用的主入口与调用链定位
  - provider/model 选择、allowlist、fallback 的关键路径梳理
  - 扩展方式（config-based provider vs plugin provider）的边界与入口整理
- 未做代码修改（本需求为分析与入口梳理）。

# anything-llm 聊天模块代码解析

## 1. 总体视角：一次聊天请求的完整链路

从前端发起一次聊天（包括普通聊天和带线程的聊天）到服务端返回流式 SSE 响应，整体调用链大致如下：

1. **HTTP 路由入口**：`server/endpoints/chat.js`
2. **校验与鉴权中间件**：`validatedRequest`、`flexUserRoleValid`、`validWorkspaceSlug` / `validWorkspaceAndThreadSlug`
3. **读取请求体与当前用户**：`reqBody`、`userFromSession`
4. **多用户配额控制**：`User.canSendChat`
5. **核心处理函数**：`streamChatWithWorkspace`（`server/utils/chats/stream.js`）
6. **命令与 Agent 分流**：`grepCommand` / `VALID_COMMANDS`、`grepAgents`
7. **向量库 & LLM 接入**：`getVectorDbClass`、`getLLMProvider`
8. **上下文构建**：历史消息、pinned 文档、解析文件、向量检索、上下文窗口填充
9. **Prompt 组装与 LLM 调用**：`chatPrompt` + `LLMConnector.compressMessages` + `getChatCompletion` / `streamGetChatCompletion`
10. **流式 SSE 输出与 token 统计**：`writeResponseChunk`、`handleDefaultStreamResponseV2`
11. **聊天记录落库 & 线程自动重命名**：`WorkspaceChats.new`、`WorkspaceThread.autoRenameThread`
12. **遥测与事件日志**：`Telemetry.sendTelemetry`、`EventLogs.logEvent`

---

## 2. 路由入口：`server/endpoints/chat.js`

### 2.1 无线程聊天：`POST /workspace/:slug/stream-chat`

- 路由定义：
  - `app.post("/workspace/:slug/stream-chat", [middlewares...], handler)`
- 中间件链：
  - `validatedRequest`：通用请求校验
  - `flexUserRoleValid([ROLES.all])`：多用户模式下的权限控制
  - `validWorkspaceSlug`：根据 `:slug` 查出 Workspace，挂载到 `response.locals.workspace`
- Handler 核心逻辑：
  1. `user = await userFromSession(request, response)` 获取当前用户
  2. `const { message, attachments = [] } = reqBody(request)` 读取请求体
  3. 校验 `message` 非空，否则返回 400 + `type: "abort"` JSON（非 SSE）
  4. 设置 SSE 响应头：
     - `Cache-Control: no-cache`
     - `Content-Type: text/event-stream`
     - `Access-Control-Allow-Origin: *`
     - `Connection: keep-alive`
  5. 多用户配额限制：
     - 如果 `multiUserMode(response)` 且 `!(await User.canSendChat(user))`：
       - 调 `writeResponseChunk(response, {... type: "abort" ...})` 直接写 SSE，提示 24 小时内额度用完
       - 结束 handler
  6. 调用核心函数：
     - `await streamChatWithWorkspace(response, workspace, message, workspace?.chatMode, user, null, attachments)`
  7. 成功后上报遥测与事件日志：
     - `Telemetry.sendTelemetry("sent_chat", {...})`
     - `EventLogs.logEvent("sent_chat", { workspaceName, chatModel }, user?.id)`
  8. `response.end()` 结束 SSE 响应

### 2.2 带线程聊天：`POST /workspace/:slug/thread/:threadSlug/stream-chat`

- 路由定义类似，只是路径多了 `thread/:threadSlug`
- 中间件链：
  - `validatedRequest`
  - `flexUserRoleValid([ROLES.all])`
  - `validWorkspaceAndThreadSlug`：解析并查出 `workspace` + `thread`，分别放在 `response.locals.workspace` 和 `response.locals.thread`
- Handler 核心逻辑：
  - 与无线程版本几乎一致，不同点：
    1. 调用 `streamChatWithWorkspace` 时传入 `thread`
    2. 在聊天结束后调用 `WorkspaceThread.autoRenameThread(...)`：
       - 只在线程名字仍为默认值 `"Thread"` 时会执行
       - 使用 `truncate(message, 22)` 取首条消息的前 22 个字符作为新线程名
       - 若实际改名成功，通过 `onRename` 回调写一条特殊 SSE：
         ```js
         writeResponseChunk(response, {
           action: "rename_thread",
           thread: { slug: thread.slug, name: thread.name },
         });
         ```
    3. Telemetry 与 EventLogs 在 metadata 中增加线程信息（`thread: thread.name`）

---

## 3. 聊天核心流程：`server/utils/chats/stream.js`

核心导出：

- `VALID_CHAT_MODE = ["chat", "query"]`
- `async function streamChatWithWorkspace(response, workspace, message, chatMode = "chat", user = null, thread = null, attachments = [])`

### 3.1 命令与 Agent 分流

1. **生成本次会话 uuid**：
   - `const uuid = uuidv4();`

2. **命令解析**：
   - `const updatedMessage = await grepCommand(message, user);`
   - 如果 `updatedMessage` 在 `VALID_COMMANDS` 的 key 集合中：
     - 执行对应命令：`VALID_COMMANDS[updatedMessage](workspace, message, uuid, user, thread)`
     - 将命令结果 `data` 直接用 `writeResponseChunk(response, data)` 写出
     - 返回，不再走普通聊天流程

3. **Agent 分流**：
   - `const isAgentChat = await grepAgents({ uuid, response, message: updatedMessage, user, workspace, thread });`
   - 如果返回 `true`：说明本轮请求走 Agent 逻辑，由 Agent 自己通过 SSE 向前端推送结果，这里直接 `return`

> 这一阶段可以理解为“特殊模式优先”：
> - 命令模式：把 prompt 当成命令处理
> - Agent 模式：进入智能体多步推理通道
> - 普通聊天：只在未命中以上两类时才继续

### 3.2 LLM 与向量库初始化

- 选择 LLM 提供方与模型：
  ```js
  const LLMConnector = getLLMProvider({
    provider: workspace?.chatProvider,
    model: workspace?.chatModel,
  });
  ```

- 获取当前向量库实现：
  ```js
  const VectorDb = getVectorDbClass();
  ```

- 历史消息条数上限：
  ```js
  const messageLimit = workspace?.openAiHistory || 20;
  ```

- 工作区向量状态：
  ```js
  const hasVectorizedSpace = await VectorDb.hasNamespace(workspace.slug);
  const embeddingsCount = await VectorDb.namespaceCount(workspace.slug);
  ```

### 3.3 query 模式早退（无向量数据）

当满足以下条件时：

- `chatMode === "query"`
- 且 `hasVectorizedSpace === false` 或 `embeddingsCount === 0`

直接：

1. 构造拒绝文案：
   ```js
   const textResponse =
     workspace?.queryRefusalResponse ??
     "There is no relevant information in this workspace to answer your query.";
   ```
2. 通过 `writeResponseChunk` 写一条 `type: "textResponse"` 的 SSE：
   - 不再去调用 LLM，避免模型在“无知识”的前提下胡编
3. 用 `WorkspaceChats.new` 记录这次对话（`include: false`，表示这条回答不参与后续检索）

### 3.4 构建上下文：历史 + pinned 文档 + 解析文件 + 向量检索

**1）历史消息**

```js
const { rawHistory, chatHistory } = await recentChatHistory({
  user,
  workspace,
  thread,
  messageLimit,
});
```

- `rawHistory`：原始数据库记录，供后续上下文填充函数使用
- `chatHistory`：已经格式化好的消息历史，供 LLM prompt 构造使用

**2）Pinned 文档**

通过 `DocumentManager` 获取当前工作区 pinned 文档，并直接把内容加入上下文：

```js
await new DocumentManager({
  workspace,
  maxTokens: LLMConnector.promptWindowLimit(),
})
  .pinnedDocs()
  .then((pinnedDocs) => {
    pinnedDocs.forEach((doc) => {
      const { pageContent, ...metadata } = doc;
      pinnedDocIdentifiers.push(sourceIdentifier(doc));
      contextTexts.push(doc.pageContent);
      sources.push({
        text:
          pageContent.slice(0, 1_000) +
          "...continued on in source document...",
        ...metadata,
      });
    });
  });
```

- `contextTexts`：给 LLM 看的完整上下文文本列表
- `sources`：给前端展示的“引用片段”，截断到 1000 字
- `pinnedDocIdentifiers`：记录 pinned 文档标识，用于后续向量检索时过滤，避免重复

**3）解析文件上下文**

```js
const parsedFiles = await WorkspaceParsedFiles.getContextFiles(
  workspace,
  thread || null,
  user || null
);
parsedFiles.forEach((doc) => {
  const { pageContent, ...metadata } = doc;
  contextTexts.push(doc.pageContent);
  sources.push({
    text:
      pageContent.slice(0, 1_000) + "...continued on in source document...",
    ...metadata,
  });
});
```

- 与 pinned 文档类似，只是这些是“解析过的临时文档”（例如用户刚上传还没完全索引的文件）。

**4）向量检索**

```js
const vectorSearchResults =
  embeddingsCount !== 0
    ? await VectorDb.performSimilaritySearch({
        namespace: workspace.slug,
        input: updatedMessage,
        LLMConnector,
        similarityThreshold: workspace?.similarityThreshold,
        topN: workspace?.topN,
        filterIdentifiers: pinnedDocIdentifiers,
        rerank: workspace?.vectorSearchMode === "rerank",
      })
    : {
        contextTexts: [],
        sources: [],
        message: null,
      };
```

- 若向量检索失败（`vectorSearchResults.message` 为非空）：
  - 输出一条 `type: "abort"` 的 SSE，携带错误信息，直接返回

**5）上下文窗口填充 `fillSourceWindow`**

```js
const { fillSourceWindow } = require("../helpers/chat");
const filledSources = fillSourceWindow({
  nDocs: workspace?.topN || 4,
  searchResults: vectorSearchResults.sources,
  history: rawHistory,
  filterIdentifiers: pinnedDocIdentifiers,
});

contextTexts = [...contextTexts, ...filledSources.contextTexts];
// sources 只追加当前检索结果的 sources
sources = [...sources, ...vectorSearchResults.sources];
```

- 注释里解释了一个细节设计：
  - **`contextTexts`** 可以包含更多历史引用，帮助 LLM 回答时“更聪明”；
  - **`sources`** 只展示“当前检索得到的片段”，避免前端看到一堆看似不相关的旧引用，从而产生“模型乱引用”的错觉。

**6）query 模式下无上下文的再早退**

如果 `chatMode === "query"` 且 `contextTexts.length === 0`：

- 直接返回前面提到的“没有相关信息”的 `textResponse`
- 写入 `WorkspaceChats` 历史，并 `return`

### 3.5 Prompt 组装与 LLM 调用

**1）压缩消息 / prompt 构造**

```js
const messages = await LLMConnector.compressMessages(
  {
    systemPrompt: await chatPrompt(workspace, user),
    userPrompt: updatedMessage,
    contextTexts,
    chatHistory,
    attachments,
  },
  rawHistory
);
```

- `chatPrompt(workspace, user)`：生成当前工作区 + 用户上下文下的 system prompt
- `compressMessages`：在 LLM 的 token 窗口限制内，对历史 + 文档上下文进行压缩，保证既不过长，又保留足够信息

**2）非流式模式**

当 `LLMConnector.streamingEnabled() !== true` 时：

- 使用 `LLMConnector.getChatCompletion(messages, { temperature, user })`
- 拿到完整 `textResponse` 和 `metrics`（token 用量等性能指标）
- 直接发一条 SSE：
  - `type: "textResponseChunk"`
  - `textResponse: completeText`
  - `close: true`

**3）流式模式**

当 `LLMConnector.streamingEnabled() === true` 时：

```js
const stream = await LLMConnector.streamGetChatCompletion(messages, {
  temperature: workspace?.openAiTemp ?? LLMConnector.defaultTemp,
  user: user,
});
completeText = await LLMConnector.handleStream(response, stream, {
  uuid,
  sources,
});
metrics = stream.metrics;
```

- 这里通常会用到 `handleDefaultStreamResponseV2`：
  - 遍历 provider 返回的 `stream`：
    - 从 `chunk.choices[0].delta.content` 取出 token
    - 每个 token 写一条 `type: "textResponseChunk"` 的 SSE
  - 遇到 `finish_reason` 时写入最后一个 `close: true` 的 chunk
  - 支持 `response.on("close")` 来处理中途取消
  - 统计 token usage 与耗时，填入 `metrics`

### 3.6 聊天记录落库与最后收尾

1. 如果 `completeText` 非空：

   ```js
   const { chat } = await WorkspaceChats.new({
     workspaceId: workspace.id,
     prompt: message,
     response: {
       text: completeText,
       sources,
       type: chatMode,
       attachments,
       metrics,
     },
     threadId: thread?.id || null,
     user,
   });

   writeResponseChunk(response, {
     uuid,
     type: "finalizeResponseStream",
     close: true,
     error: false,
     chatId: chat.id,
     metrics,
   });
   ```

   - 将问答写入 `workspace_chats` 表
   - 最后一条 SSE 使用 `type: "finalizeResponseStream"` 带上 `chatId` 和 `metrics`，通知前端整次对话结束

2. 如果 `completeText` 为空：

   - 仍然写一条 `finalizeResponseStream`，只是没有 `chatId`

---

## 4. SSE 与历史格式化工具：`utils/helpers/chat/responses.js`

### 4.1 `writeResponseChunk`

- 核心函数：
  ```js
  function writeResponseChunk(response, data) {
    response.write(`data: ${safeJSONStringify(data)}\n\n`);
    return;
  }
  ```
- 使用 SSE 标准格式：`data: <json>\n\n`
- 通过 `safeJSONStringify` 处理 BigInt 等特殊值，避免 JSON 序列化出错

### 4.2 `handleDefaultStreamResponseV2`

- 通用的 LLM 流式响应处理器：
  - 监听 `response.on("close")`，支持客户端主动终止流；终止时调用 `clientAbortedHandler`，结束监控并返回当前已生成文本
  - `for await (const chunk of stream)`：
    - 抽取 `chunk.choices[0].delta.content` 作为 token
    - 每个 token 立刻写一条 SSE：`type: "textResponseChunk"`, `close: false`
  - 检测 `finish_reason`：
    - 写入最终一条 `close: true` 的 chunk，并结束
  - 如果 provider 返回 `usage`，则直接用；否则根据 chunk 数量估算 `completion_tokens`

### 4.3 历史格式化函数

- `convertToChatHistory`：
  - 将数据库中的历史记录转成前端使用的“对话消息数组”（带 `role`、`content`、`sources`、`metrics` 等）
- `convertToPromptHistory`：
  - 更偏向“仅供 LLM 继续对话”的消息格式
- `formatChatHistory`：
  - 处理带附件的历史消息，适配不同 LLM provider 对消息格式的要求（`mode: 'asProperty' | 'spread'`）

---

## 5. 配套模型与功能

### 5.1 `Telemetry`：聊天事件遥测

- 文件：`server/models/telemetry.js`
- 聊天中典型用法：
  ```js
  await Telemetry.sendTelemetry("sent_chat", {
    multiUserMode: multiUserMode(response),
    LLMSelection: process.env.LLM_PROVIDER || "openai",
    Embedder: process.env.EMBEDDING_ENGINE || "inherit",
    VectorDbSelection: process.env.VECTOR_DB || "lancedb",
    multiModal: Array.isArray(attachments) && attachments?.length !== 0,
    TTSSelection: process.env.TTS_PROVIDER || "native",
    LLMModel: getModelTag(),
  });
  ```
- 用 PostHog 收集匿名使用情况，并支持事件级冷却（避免刷日志）。

### 5.2 `EventLogs`：行为日志

- 文件：`server/models/eventLogs.js`
- 聊天中典型用法：
  ```js
  await EventLogs.logEvent(
    "sent_chat",
    { workspaceName: workspace?.name, chatModel: workspace?.chatModel || "System Default" },
    user?.id
  );
  ```
- 使用 Prisma 写入 `event_logs` 表，方便后续在后台查看行为记录。

### 5.3 `WorkspaceThread.autoRenameThread`

- 文件：`server/models/workspaceThread.js`
- 聊天结束后自动重命名线程：
  - 规则：
    - 只在 thread 当前名称为默认值 `"Thread"` 时生效
    - 统计该线程下聊天条数，只有当 `chatCount === 1` 时才会改名（即首条消息后改名）
    - 用首条消息文本截断后的内容作新线程名
  - 改名完成后通过 `onRename` 回调触发 SSE，将新名字通知前端。

### 5.4 `User.canSendChat`

- 文件：`server/models/user.js`
- 用于多用户模式下限制普通用户在 24 小时内可发送的聊天次数：
  - 如果 `user` 不存在、`dailyMessageLimit === null`，或 `user.role === ROLES.admin`，则无限制
  - 否则统计最近 24 小时内该用户在 `WorkspaceChats` 中的聊天条数，和 `dailyMessageLimit` 对比
  - 在 `chat.js` 中，若返回 `false`，会直接以 `type: "abort"` 的 SSE 告知前端额度用完

---

## 6. 后续深入建议

如果后面想继续深入聊天模块，可以按下面几个方向看代码：

- **Prompt & 命令/Agent：**
  - `server/utils/chats/index.js`：`grepCommand`、`VALID_COMMANDS`、`chatPrompt`、`recentChatHistory`、`sourceIdentifier`
  - `server/utils/chats/agents.js` + `models/workspaceAgentInvocation.js`：Agent 流程
- **上下文窗口策略：**
  - `server/utils/helpers/chat/index.js`（或同类文件）里的 `fillSourceWindow`
- **聊天数据落库：**
  - `server/models/workspaceChats.js`：字段含义、include 标记、count 统计等
- **LLM Provider 实现：**
  - `server/utils/AiProviders/*`：不同 provider 的 `getChatCompletion` / `streamGetChatCompletion` / `handleStream`

> 这个文档旨在当你再次打开聊天相关代码时，可以快速回忆：
> - 请求从 `endpoints/chat.js` 进来之后完整经过哪些环节；
> - `streamChatWithWorkspace` 内部的各个阶段作用是什么；
> - 哪些 models/utils 参与了聊天的配额控制、日志记录与线程管理。

---

## 7. 如何调试 `streamChatWithWorkspace`（实践步骤）

下面是一套推荐的本地调试流程，帮助熟悉 `streamChatWithWorkspace` 的执行路径。

### 7.1 启动服务并确认路由就绪

1. 在项目根目录启动服务端（示例）：
   - `yarn server` 或查看 `server/package.json` 中的 scripts 使用对应命令。
2. 确认 `.env` / `.env.development` 中至少配置：
   - 数据库连接（Prisma）
   - LLM 相关配置（如 `LLM_PROVIDER`、模型、密钥等）
   - 向量库相关配置（`VECTOR_DB` 等）
3. 打开 `server/index.js`，确认 `chatEndpoints(apiRouter)` 已被调用，确保 `/api/workspace/:slug/stream-chat` 等路由已挂载。

### 7.2 用 HTTP 请求直接触发 `streamChatWithWorkspace`

假设：

- 服务监听在 `http://localhost:3001`
- 已创建一个 workspace，`slug` 为 `demo`（可在前端 UI 创建后从 URL 获取）

可以在命令行用 curl 简单验证（注意 SSE 是流式返回，不会一次性结束）：

```bash
curl -N \
  -H "Content-Type: application/json" \
  -X POST \
  -d '{"message":"Hello from curl"}' \
  http://localhost:3001/api/workspace/demo/stream-chat
```

观察终端输出的 `data: {...}` 行，即为 `writeResponseChunk` 写出的 SSE 数据。

如果要调试线程版接口（会触发自动重命名逻辑），可以调用：

```bash
curl -N \
  -H "Content-Type: application/json" \
  -X POST \
  -d '{"message":"First message in this thread"}' \
  http://localhost:3001/api/workspace/demo/thread/<threadSlug>/stream-chat
```

### 7.3 在关键节点加日志

在 IDE 中打开 `server/utils/chats/stream.js`，针对以下位置有选择地加 `console.log` 方便观察：

- **命令 / Agent 分流前后：**
  - 在 `grepCommand` 之后打印 `updatedMessage`
  - 在 `grepAgents` 返回后打印 `isAgentChat`
- **向量库状态与检索：**
  - 在计算 `hasVectorizedSpace`、`embeddingsCount` 后打印结果
  - 在 `performSimilaritySearch` 返回后打印 `vectorSearchResults.sources.length`
- **上下文大小：**
  - 打印 `contextTexts.length` 和 `sources.length`，观察 pinned 文档、解析文件、检索结果对上下文的影响
- **LLM 调用：**
  - 打印 `messages.length`、`LLMConnector.constructor.name`、`workspace?.chatModel`

加日志后再次用 curl 或前端 UI 触发请求，对照终端日志与 SSE 输出理解每一步发生了什么。

### 7.4 使用断点（Node 调试）

如果在 IDE 中支持 Node 调试（如 VS Code 的 `Launch Program` 或 `Attach` 配置），可以：

1. 在 `streamChatWithWorkspace` 函数开头和关键分支处打断点，例如：
   - 命令路径：`if (Object.keys(VALID_COMMANDS).includes(updatedMessage)) { ... }`
   - Agent 路径：`if (isAgentChat) return;`
   - query 无数据早退路径
   - 调用 `VectorDb.performSimilaritySearch` 前后
   - 调用 `LLMConnector.getChatCompletion` / `streamGetChatCompletion` 前
2. 启动调试模式的 Node 进程，或 attach 到已运行的服务端进程。
3. 再次发起请求，观察堆栈、局部变量值：
   - `workspace.slug`、`workspace.chatMode`、`workspace.topN` 等配置
   - `rawHistory`、`chatHistory` 的大小和内容结构
   - `messages` 对象中 system / user / context 消息的排列情况

### 7.5 有针对性地验证不同路径

可以故意构造不同场景，逐条验证代码中不同分支：

- **命令路径：**
  - 查看 `server/utils/chats/index.js` 中 `VALID_COMMANDS` 定义，构造一个匹配命令的 `message`
  - 观察 SSE 中是否直接返回特定结构，而不进入向量检索与 LLM 调用
- **Agent 路径：**
  - 配置/启用某个 Agent（参考 `server/utils/chats/agents.js` 与 `models/workspaceAgentInvocation.js`）
  - 发起触发 Agent 的消息，观察 `grepAgents` 是否返回 `true`，以及前端收到的 SSE 数据结构是否更复杂（包含 Agent 步骤）
- **无向量数据的 query 模式：**
  - 在一个没有任何文档的 workspace 上，将 `chatMode` 设置为 `query`
  - 发送任意消息，确认走到了“没有相关信息”的早退路径
- **向量检索失败：**
  - 可以暂时在 `performSimilaritySearch` 内部制造一个错误（例如抛异常或返回带 `message` 的结果），观察 SSE 中 `type: "abort"` 的数据结构
- **流式 / 非流式切换：**
  - 修改某个 LLM Provider 的 `streamingEnabled` 返回值，观察接口从“多次小 chunk”变成“一次完整响应”的差异

### 7.6 结合数据库观察效果

配合 Prisma 数据库，进一步验证 `streamChatWithWorkspace` 的写库行为：

- 查看 `prisma/schema.prisma` 中 `workspace_chats`、`event_logs`、`workspace_threads` 等表定义
- 在发送几条聊天后，查询数据库：
  - `workspace_chats` 中是否有对应的 prompt / response / sources / metrics
  - `event_logs` 中是否有 `sent_chat` 事件
  - 线程模式下，`workspace_threads` 中的 `name` 是否从 `"Thread"` 被自动更新为首条消息截断后的标题

通过以上调试步骤，可以从“黑盒调用”变成“白盒观察”，对 `streamChatWithWorkspace` 的执行路径和数据流有更直观的理解。

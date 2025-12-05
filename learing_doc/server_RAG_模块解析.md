# server 目录中文档解析与 RAG 流程模块概览

下面按“文档解析 → 嵌入/向量化 → 向量库存储 → 聊天检索(RAG)”链路，整理 `server` 目录下相关模块，便于阅读和调试。

---

## 一、文档解析：从文件到结构化文本

### 1. `server/utils/collectorApi/index.js`

- **职责**：封装对 Collector 服务的 HTTP 调用，用于文档和链接内容的解析。Collector 内部会根据文件后缀选择不同的解析器（如 PDF、DOCX 等），并返回统一结构的 `documents`。
- **关键方法：`parseDocument(filename)`**
  - 入参：`filename` 为需要解析的文件名（位于 Collector 热目录）。
  - 行为：
    - 组装请求体 `{ filename, options: this.#attachOptions() }`，其中 `options` 会携带 OCR 语言等解析配置。
    - 调用 `${this.endpoint}/parse` 的 POST 接口，由 Collector 侧的 `collector/processSingleFile` 根据文件扩展名路由到对应的解析器：
      - **PDF (`.pdf`)**：使用 `collector/processSingleFile/convert/asPDF/index.js`
        - 内部通过 `PDFLoader` 加载 PDF，每一页提取文字，按行简单拼接后，每页生成一个文档对象：
          `[{ pageContent, metadata: { source, pdf: { version, info, metadata, totalPages }, loc: { pageNumber } } }, ...]`。
        - 当前实现中构造 `PDFLoader` 时固定传入 `{ splitPages: true }`，因此默认是“按页切分”的结果返回，`documents` 通常为多条，每条对应一页。
      - **DOCX (`.docx`)**：使用 `collector/processSingleFile/convert/asDocx.js`
        - 使用 LangChain 的 `DocxLoader` 抽取文本，将 loader 返回的多段 `pageContent` 直接 `join("")` 合并为一个长字符串，生成单个文档对象：
          `{ pageContent: fullText, metadata: { source, createdDate, wordCount, token_count_estimate, ... } }`。
        - 因此 `documents` 通常只包含一条“整篇文档”的记录。
    - 将 Collector 返回的解析结果直接透传给调用方：`{ success, reason, documents }`。
  - 位置：大约在文件 255 行附近。
- **相关方法：`getLinkContent(link, captureAs)`**
  - 作用：对一个 URL 抓取文本或 HTML 内容，封装成与上面 `documents` 相似的结构，供后续作为文档输入使用。

### 2. 聊天附件解析：`server/utils/chats/apiChatHandler.js`

- **职责**：处理用户通过聊天上传的文档附件，将其转为可用的文本片段。
- **关键流程：`processDocumentAttachments(attachments)`**（函数定义在文件前半部分）
  - 接收前端上传的附件数组，每个元素包含：`name`、`mime`、`contentString(base64)`。
  - 步骤：
    - 将 Base64 字符串解码为二进制 Buffer。
    - 使用 `hotdirPath` 作为 Collector 的热目录，将附件写入磁盘。
    - 调用上面的 `Collector.parseDocument(filename)` 获得解析后的 `documents`。
    - 成功的解析结果推入 `parsedDocuments`，图片等非文本文档保留在 `imageAttachments`。
  - 返回：`{ parsedDocuments, imageAttachments }`。

---

## 二、文档入库与向量化：从解析结果到向量库

### 1. 解析文件记录与触发嵌入：`server/models/workspaceParsedFiles.js`

- **职责**：管理 `workspace_parsed_files` 表中的记录，以及从“解析完成的文件”到“嵌入入库”的迁移流程。
- **关键方法：`moveToDocumentsAndEmbed(fileId, workspace)`**
  - 步骤：
    1. 根据 `fileId` 在 `workspace_parsed_files` 表查找对应记录 `parsedFile`。
    2. 从 `parsedFile.metadata` 中读取 `location` 字段，作为源文件在磁盘中的位置。
    3. 将源文件从 `direct-uploads` 目录复制/移动到 `documents/custom-documents` 目录。
    4. 调用 `Document.addDocuments(workspace, ["custom-documents/xxx"], parsedFile.userId)`：
       - 这一步会触发真正的向量化和文档入库逻辑（见下文）。
    5. 如果嵌入失败，抛出错误；成功则通过 `Document.get` 再查回刚刚入库的文档记录并返回。
    6. 最终无论成功或失败，都会删除 `workspace_parsed_files` 中对应记录。

### 2. 文档模型与批量添加：`server/models/documents.js`

- **职责**：管理工作区文档表 `workspace_documents`，并负责把文件内容向量化并写入向量库。
- **关键方法：`addDocuments(workspace, additions = [], userId = null)`**
  - 入参：
    - `workspace`：当前工作区对象（含 `id`、`slug` 等）。
    - `additions`：字符串数组，每个元素为文档相对路径，如 `"custom-documents/foo.pdf"`。
    - `userId`：操作用户 ID，可为空。
  - 内部流程：
    1. 根据当前配置通过 `getVectorDbClass()` 获取向量库实现类 `VectorDb`。
    2. 对 `additions` 中每个文档路径：
       - 调用 `require("../utils/files").fileData(path)` 读取文件内容：
         - 返回对象中包含 `pageContent`（用于嵌入）和其他 `metadata`（如标题、类型等）。
       - 生成一个新的 `docId`，组装 `newDoc`：
         - `filename`：文件名。
         - `docpath`：相对路径（如 `custom-documents/foo.pdf`）。
         - `workspaceId`：当前工作区 ID。
         - `metadata`：序列化后的元数据 JSON 字符串。
       - 调用 `VectorDb.addDocumentToNamespace(workspace.slug, { ...data, docId }, path)`：
         - `namespace` 使用 `workspace.slug`，确保各工作区向量隔离。
         - 传入的 `documentData` 包含 `pageContent` 和 `docId` 等。
       - 根据返回的 `{ vectorized, error }`：
         - 若 `vectorized === false`，记录错误并跳过该文件。
         - 若成功，使用 Prisma `workspace_documents.create` 插入文档记录。
    3. 所有文档处理完成后：
       - 发送 Telemetry 统计，如 `documents_embedded_in_workspace`。
       - 写入事件日志 `workspace_documents_added`（工作区名称、文档数量等）。
    4. 返回：`{ failedToEmbed, errors, embedded }`。

---

## 三、嵌入引擎：文本向量化实现

### 1. 本地嵌入引擎：`server/utils/EmbeddingEngines/native/index.js`

- **职责**：提供基于本地模型的 embedding 能力，是默认或可选的嵌入实现之一。
- **关键方法：`embedTextInput(textInput)`**
  - 用于对单条文本（或少量文本）进行嵌入。
  - 实际调用 `embedChunks`，只取返回数组的第一个结果。

- **关键方法：`embedChunks(textChunks = [])`**
  - 核心批量嵌入逻辑，针对大文档做特别的内存优化。
  - 内部设计要点：
    - 将 `textChunks` 按 `maxConcurrentChunks`（默认 25）分组，避免一次性送太多 chunk 导致 OOM。
    - 每一批次：
      - 通过 `embedderClient()` 获取 pipeline。
      - 调用 pipeline，配置 `pooling: "mean"`、`normalize: true`。
      - 将结果写入临时文件，而不是全部留在内存中。
    - 所有批次完成后，从临时文件读回 embedding 结果 JSON，删除临时文件，最终展平返回向量数组。

- **其他嵌入引擎**
  - 在 `server/utils/EmbeddingEngines/` 目录下还包含如 OpenAI 等第三方嵌入实现，通过 `getEmbeddingEngineSelection()` 统一选择。

---

## 四、向量数据库 Provider：向量存储与相似度检索

### 1. Provider 选择入口：`server/utils/helpers/index.js`

- **函数：`getVectorDbClass(getExactly = null)`**
  - 根据 `getExactly` 或 `process.env.VECTOR_DB` 决定使用哪种向量库实现。
  - 支持的值包括：`pinecone`、`chroma`、`chromacloud`、`lancedb`、`weaviate`、`qdrant`、`milvus`、`zilliz`、`astra`、`pgvector` 等。
  - 默认回退为 `LanceDb`，并在控制台打印提示。

### 2. LanceDB 实现示例：`server/utils/vectorDbProviders/lance/index.js`

> 其他 Provider（Pinecone/Chroma 等）在结构上与此类似，可参考该实现理解整体设计。

#### 2.1 新文档向量化：`addDocumentToNamespace(namespace, documentData, fullFilePath, skipCache)`

- 入参：
  - `namespace`：工作区命名空间，一般为 `workspace.slug`。
  - `documentData`：包含 `pageContent`、`docId` 及其他元数据。
  - `fullFilePath`：源文件在磁盘的完整路径，用于缓存命中与存储。
  - `skipCache`：是否跳过向量缓存机制。

- 主要流程：

  1. **缓存复用路径**（若 `skipCache` 为 `false`）：
     - 调用 `cachedVectorInformation(fullFilePath)` 检查是否已有缓存向量。
     - 若命中：
       - 读取缓存中的 `chunks`，为每个向量生成新 `id`，组装 `submissions`。
       - 调用 `updateOrCreateCollection(client, submissions, namespace)` 将向量写入 LanceDB。
       - 调用 `DocumentVectors.bulkInsert(documentVectors)` 建立 `docId` 与 `vectorId` 映射。
       - 返回 `{ vectorized: true, error: null }`。

  2. **正常路径（新文档嵌入）**：
     - 使用 `TextSplitter` 对 `pageContent` 做分块：
       - `chunkSize` 由系统设置和 Embedder 配置共同决定。
       - `chunkOverlap` 支持配置，默认 20。
       - `chunkHeaderMeta`、`chunkPrefix` 用于给每个块增加上下文头信息或前缀。
     - 通过 `getEmbeddingEngineSelection()` 拿到 `EmbedderEngine`，调用 `EmbedderEngine.embedChunks(textChunks)`：
       - 得到 `vectorValues`，与 `textChunks` 一一对应。
     - 对每个向量：
       - 生成唯一 `id`。
       - `metadata` 中强制包含 `text: textChunks[i]` 字段，以兼容 LangChain 的检索逻辑。
       - 将向量和元数据推入 `vectors` 与 `submissions`，并记录 `{ docId, vectorId }` 到 `documentVectors`。
     - 若 `vectors.length > 0`：
       - 将 `vectors` 按 500 条一组拆分，批量插入到 LanceDB 集合中。
       - 调用 `storeVectorResult(chunks, fullFilePath)` 缓存嵌入结果，供下次复用。
     - 最后调用 `DocumentVectors.bulkInsert(documentVectors)` 完成数据库映射记录，并返回成功标记。

  - 进一步说明：

    - `chunkSize` 的默认来源：
      - 代码中通过 `SystemSettings.getValueOrFallback({ label: "text_splitter_chunk_size" })` 读取系统配置；
      - 再由 `TextSplitter.determineMaxChunkSize(配置值, EmbedderEngine.embeddingMaxChunkLength)` 做一次裁剪，保证不会超过嵌入模型允许的最大长度。因此实际生效的 `chunkSize` 受“系统配置 + 模型上限”共同约束。

    - `text_splitter_chunk_overlap` / `chunkOverlap` 的含义：
      - 表示**相邻两个文本块之间的重叠长度**（字符或 token 数），默认 fallback 为 `20`；
      - 举例：若 `chunkSize = 100`、`chunkOverlap = 20`，原始文本足够长时，切分出的块大致为：
        - 第 1 块覆盖 `[0, 99]`；
        - 第 2 块从 `100 - 20 = 80` 开始，覆盖 `[80, 179]`，与第 1 块在 `[80, 99]` 这 20 个单位上发生重叠；
        - 第 3 块从 `180 - 20 = 160` 开始，覆盖 `[160, 259]`，依此类推。
      - 通过这种重叠，可以在块与块之间保留一定的上下文，便于向量检索时捕获跨句子、跨段落的信息。

    - “一块文本对应一条向量记录”的过程：
      - `textSplitter.splitText(pageContent)` 得到 `textChunks` 数组；
      - `EmbedderEngine.embedChunks(textChunks)` 返回与之等长的 `vectorValues` 数组；
      - 随后的 `for (const [i, vector] of vectorValues.entries())` 循环中：
        - 第 `i` 个向量 `vector` 与第 `i` 个文本块 `textChunks[i]` 一一对应；
        - 为每个向量生成唯一 `id`，并构造 `vectorRecord`（含 `values` 与 `metadata.text = textChunks[i]`）；
        - 推入 `vectors` 与 `submissions`，同时在 `documentVectors` 中记录 `{ docId, vectorId }`；
      - 因此：**每一个切分出来的文本块都会生成一条独立的向量记录，并存入对应的 LanceDB collection 中。**

    - LanceDB 中一条记录的典型结构（概念上）：
      - 在 `updateOrCreateCollection(client, submissions, namespace)` 中，`submissions` 数组的每一项类似：
        - `{ id: string, vector: number[], text: string, source: string, docId?: string, ...其他 metadata }`；
      - 在 LanceDB 视角，即某个 `namespace` 下的一张表/集合：
        - 列包括：`id`（主键）、`vector`（向量列）、`text`（原文 chunk）、以及各种业务相关元数据字段；
      - 结合应用侧的 `DocumentVectors` 映射表 `{ docId, vectorId }`，可以在需要删除某个文档或检索其相关片段时，快速定位并操作 LanceDB 中对应的所有向量行。

#### 2.2 相似度检索：`performSimilaritySearch({...})`

- 入参（概念上）：
  - `namespace`：工作区命名空间。
  - `input`：用户当前查询/消息。
  - `LLMConnector`：用于获取模型窗口大小或做后处理（视实现而定）。
  - `similarityThreshold`：相似度阈值。
  - `topN`：返回前多少条相似文档。
  - `filterIdentifiers`：过滤某些已固定的文档（pinned docs）。
  - `rerank`：是否启用重排序。

- 返回：
  - `contextTexts`：推荐追加到 LLM Prompt 中的上下文片段。
  - `sources`：用于前端展示的文档引用（含文本摘要、路径、标题等）。
  - `message`：错误信息（若检索失败）。

---

## 五、RAG 检索与上下文构建：聊天时如何用向量记忆

### 1. 同步聊天入口：`server/utils/chats/apiChatHandler.js` 中的 `chatSync`

- **职责**：
  - 处理开发者 API 调用的同步聊天请求。
  - 根据工作区向量数据和附件解析等，构建 RAG 上下文并调用 LLM。

- **关键步骤摘录**：

#### 1.1 向量空间可用性检查

- 初始化：
  - `LLMConnector = getLLMProvider({ provider: workspace.chatProvider, model: workspace.chatModel })`。
  - `VectorDb = getVectorDbClass()`。
  - `messageLimit = workspace.openAiHistory || 20`。
  - `hasVectorizedSpace = await VectorDb.hasNamespace(workspace.slug)`。
  - `embeddingsCount = await VectorDb.namespaceCount(workspace.slug)`。
- 若当前模式为 `"query"` 且 `!hasVectorizedSpace || embeddingsCount === 0`：
  - 直接返回 `workspace.queryRefusalResponse`（或默认的拒答文案），避免在没有知识的前提下回答。

#### 1.2 固定文档（Pinned Docs）注入上下文

- 使用 `DocumentManager`：
  - `new DocumentManager({ workspace, maxTokens: LLMConnector.promptWindowLimit() }).pinnedDocs()`。
- 对每个 pinned 文档：
  - 将 `pageContent` 推入 `contextTexts`。
  - 构建 `sources`：只截取前 1000 字，并加上“...continued on in source document...”提示。
  - 记录 `pinnedDocIdentifiers`，用于后续检索过滤。

#### 1.3 聊天附件即时 RAG

- 调用前面提到的 `processDocumentAttachments(attachments)`：
  - 将解析得到的 `parsedDocuments` 同样追加到 `contextTexts` 和 `sources` 中。
  - 这些附件内容并不会写入向量库，而是作为“当前会话即时上下文”。

#### 1.4 向量相似度搜索

- 当 `embeddingsCount !== 0`：
  - 调用：
    - `VectorDb.performSimilaritySearch({ namespace: workspace.slug, input: message, LLMConnector, similarityThreshold: workspace.similarityThreshold, topN: workspace.topN, filterIdentifiers: pinnedDocIdentifiers, rerank: workspace.vectorSearchMode === "rerank" })`。
  - 获得：
    - `vectorSearchResults.sources`：当前查询最相关的文档片段列表。
    - `vectorSearchResults.contextTexts`：对应的文本内容。
- 若返回中存在 `message` 字段（表示搜索失败）：
  - 直接返回 `type: "abort"` 响应，结束本次请求。

#### 1.5 上下文窗口填充与 Citations 控制

- 通过 `fillSourceWindow({ nDocs: workspace.topN || 4, searchResults: vectorSearchResults.sources, history: rawHistory, filterIdentifiers: pinnedDocIdentifiers })`：
  - 在“当前检索结果 + 历史聊天记录”之间做一个平衡，挑选合适数量的文档作为最终上下文。
- 最终：
  - `contextTexts = [...contextTexts, ...filledSources.contextTexts]`。
  - `sources = [...sources, ...vectorSearchResults.sources]`。
- 设计考虑：
  - `contextTexts` 可以包含更多信息以帮助 LLM 回答更准确。
  - `sources` 更偏向“当前轮答案的显式引用”，避免用户看到“引用的文档好像跟问题无关”的情况。

#### 1.6 Query 模式下无上下文的拒答

- 若 `chatMode === "query"` 且 `contextTexts.length === 0`：
  - 返回 `queryRefusalResponse`，不让 LLM 在没有任何检索结果的情况下凭空作答。

---

## 六、推荐阅读顺序

如果需要系统理解或改造 RAG 流程，建议按以下顺序阅读代码：

1. **聊天入口和整体调度**
   - `server/utils/chats/apiChatHandler.js` → 重点看 `chatSync` 函数。

2. **向量库抽象与选择**
   - `server/utils/helpers/index.js` → `getVectorDbClass`。

3. **具体向量库实现（以 LanceDB 为例）**
   - `server/utils/vectorDbProviders/lance/index.js` → `addDocumentToNamespace`、`performSimilaritySearch`。

4. **文档入库与记录**
   - `server/models/documents.js` → `addDocuments`。
   - `server/models/workspaceParsedFiles.js` → `moveToDocumentsAndEmbed`。

5. **嵌入引擎**
   - `server/utils/EmbeddingEngines/native/index.js` 以及其他实际使用的引擎实现。

6. **Collector 文档解析链路**
   - `server/utils/collectorApi/index.js`。
   - 搭配 `apiChatHandler.js` 中对附件的处理逻辑一起理解。

通过以上这些文件，可以完整串起 AnythingLLM 服务端的“文档解析 → 向量化 → 向量库存储 → 聊天时 RAG 检索与上下文构建”全流程。

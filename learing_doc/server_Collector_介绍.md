# Collector 服务介绍

本文档介绍 AnythingLLM 项目中的 **Collector** 组件：它是什么、负责什么、解析不同格式文件时依赖哪些外部能力，以及当 Collector 未启动时，server 在“处理文档”上的行为表现。

---

## 一、Collector 是什么？

- **位置**：`/collector` 子项目，入口文件为 `collector/index.js`。
- **本质**：一个独立运行在 **端口 8888** 上的 Node.js 微服务，包名为 `anything-llm-document-collector`。
- **职责一句话**：
  - 将各种来源的原始内容（文件、网页链接、纯文本、部分媒体等），统一解析为标准化的 `documents` 结构（通常包含 `pageContent` 和元数据）。
  - 主 server 再基于这些 `documents` 做嵌入、向量化和入库，用于后续 RAG 检索。

可以把 Collector 理解为 AnythingLLM 的 **“文档 ETL（抽取-转换-加载）层里的抽取 + 预处理服务”**。

---

## 二、Collector 暴露的主要接口

Collector 使用 `express` 启动 HTTP 服务，在 `index.js` 中注册了多个路由端点（都带有 `verifyPayloadIntegrity` 签名校验中间件）：

### 1. `/process` —— 完整处理单个文件

- 请求体关键字段：
  - `filename`：需要处理的文件名（位于 Collector 的热目录 `hotdir` 等）。
  - `options`：可选处理参数。
  - `metadata`：可选元数据（标题、来源等）。
- 内部调用：`processSingleFile(targetFilename, options, metadata)`。
- 返回：`{ filename, success, reason, documents }`。
- 典型用途：
  - 对文件做“解析 + 进一步加工”，适合真正准备入库的文档处理。

### 2. `/parse` —— 只进行解析（parseOnly 模式）

- 请求体关键字段：`{ filename, options }`。
- 内部调用：`processSingleFile(targetFilename, { ...options, parseOnly: true })`。
- 返回：`{ filename, success, reason, documents }`。
- 特点：
  - 相比 `/process`，更偏向“只做解析，不做后续重度处理”。
  - 在主 server 中，`Collector.parseDocument(filename)` 就是调用这个接口，用于聊天附件的即时解析。

### 3. `/process-link` —— 处理网页链接

- 请求体关键字段：`{ link, scraperHeaders, metadata }`。
- 内部调用：`processLink(link, scraperHeaders, metadata)`。
- 返回：`{ url, success, reason, documents }`。
- 用途：
  - 把一个 URL（网页、文章、视频页等）抓取并解析成 `documents`，从而纳入知识库或上下文。

### 4. `/util/get-link` —— 抓取链接原始内容

- 请求体关键字段：`{ link, captureAs = "text" }`。
- 内部调用：`getLinkText(link, captureAs)`。
- 返回：`{ url, success, content }`。
- 用途：
  - 主 server 调用 `getLinkContent` 时使用，用于先拿到网页的纯文本或 HTML 内容，再自行决定如何进一步处理。

### 5. `/process-raw-text` —— 处理纯文本

- 请求体关键字段：`{ textContent, metadata }`。
- 内部调用：`processRawText(textContent, metadata)`。
- 返回：`{ filename: metadata.title, success, reason, documents }`。
- 用途：
  - 针对已经是纯文本的内容，做分段、清洗、估算 token 等，再输出统一的 `documents` 结构。

---

## 三、Collector 支持的文件/内容类型与外部依赖

Collector 通过 `package.json` 中的依赖支持多种文件格式和内容来源。下面按类型简单归类，便于理解可能的外部依赖或运行环境要求。

### 1. 文档类（PDF / Office / EPUB / 表格 / 邮件等）

- **PDF**：
  - 依赖：`pdf-parse`
  - 说明：无需额外本地二进制依赖，大部分场景下纯 JS 即可解析普通 PDF 文本。

- **Word / Office 文档（.docx, .pptx 等）**：
  - 依赖：`mammoth`、`officeparser`
  - 说明：
    - 解析基于 JS 库本身，通常不需要额外系统级依赖，但对部分复杂格式支持程度有限。

- **EPUB 电子书**：
  - 依赖：`epub2`（自家 fork 仓库）
  - 说明：解析 EPUB 结构和章节文本。

- **Excel / 表格**：
  - 依赖：`node-xlsx`
  - 说明：解析 .xlsx 等表格文件为结构化数据，再转文本片段。

- **MBOX 邮件**：
  - 依赖：`mbox-parser`

### 2. 网页与 HTML 内容

- **HTML / 纯文本解析**：
  - 依赖：`html-to-text`、`node-html-parser`
  - 用途：从 HTML 中抽取可读纯文本，或对 DOM 结构进行分析。

- **网页抓取 / 动态渲染页面**：
  - 依赖：`puppeteer`
  - 外部依赖提示：
    - Puppeteer 会下载/使用 Chromium，要求运行环境具备相应的系统依赖（如常见的 Linux 图形/字体库）。
    - 在 Docker / 服务器环境中，通常需要使用项目提供的 Dockerfile 或按照官方 Puppeteer 文档准备依赖。

### 3. 图片与 OCR

- **图像处理**：
  - 依赖：`sharp`
  - 外部依赖提示：
    - Sharp 对本地环境有一定要求（需支持 libvips 等）。
    - 在非 Docker 环境下安装时，如缺少编译工具链或系统库，可能需要手动安装依赖。

- **OCR 文字识别**：
  - 依赖：`tesseract.js`
  - 外部依赖提示：
    - 使用 JS/WASM 版 Tesseract，通常无需系统级 tesseract 二进制，但首次运行可能需要下载模型文件，需保证网络或预置模型。

### 4. 音频/视频及其它媒体

- **音视频处理**：
  - 依赖：`fluent-ffmpeg`、`wavefile`
  - 外部依赖提示：
    - `fluent-ffmpeg` 只是 Node 封装，真正处理依赖系统中安装的 `ffmpeg` 可执行程序。
    - 因此在使用与音频/视频转文本（如提取音轨、采样波形）相关能力时，需要在操作系统中预先安装 `ffmpeg`。

- **YouTube 等视频站点内容**：
  - 依赖：`youtubei.js`
  - 说明：可拉取视频的元数据、字幕等，以便转换为文本 documents。

### 5. 其它辅助依赖

- **Token 计数与分段**：
  - 依赖：`js-tiktoken`、`langchain`

- **文本/文件管理**：
  - 依赖：`slugify`、`mime`、`ignore` 等。

- **本地模型 / Transformer**：
  - 依赖：`@xenova/transformers`
  - 说明：Collector 也可在本地进行部分模型相关处理（如特定文本预处理或推理），但具体使用要看对应 utils/processor 实现。

整体来看，Collector 多数解析能力基于 **Node 模块本身** 即可运行，但涉及：

- 浏览器抓取（`puppeteer`）
- 媒体转码（`fluent-ffmpeg` → 依赖系统 `ffmpeg`）
- 图像处理（`sharp` → 依赖 libvips）

这三类场景对系统环境的要求相对更高，建议使用项目提供的 Docker 镜像或按文档准备运行环境。

---

## 四、主 server 如何调用 Collector？

在主 server 中，Collector 通过 `server/utils/collectorApi/index.js` 封装了一层 HTTP 客户端，典型用法包括：

- `parseDocument(filename)` → 调用 Collector 的 `/parse` 接口。
- `getLinkContent(link, captureAs)` → 调用 `/util/get-link`。

例如在聊天附件处理逻辑中：

- 用户在前端上传文件后，server 将附件写入 `collector/hotdir` 目录。
- 随后通过 `Collector.parseDocument(filename)` 请求 Collector，将该文件解析成 `documents`。 
- 解析结果可以直接作为本次对话的上下文（即时 RAG），或进一步走入库/向量化流程。

因此，在“文档解析”这一环节，主 server 本身**并不直接解析复杂文档**，而是把工作委托给 Collector 服务。

---

## 五、Collector 未启动时，server 的行为与影响

### 1. 聊天时上传附件的影响

在 `server/utils/chats/apiChatHandler.js` 中，处理聊天附件的代码大致流程是：

1. 构造 `CollectorApi` 客户端实例。
2. 调用 `CollectorApi.online()` 检查 Collector 是否可用。
3. 若不可用：
   - 记录一条警告日志（类似“Collector API is not online, skipping document attachment processing”）。
   - 返回 `{ parsedDocuments: [], imageAttachments }`，**直接跳过所有附件的文档解析**。

**结论**：

- 当 Collector 未启动或不可达时：
  - 聊天接口仍然可以正常返回回复（基于已有向量库内容或纯 LLM 对话）。
  - 但**本次会话上传的文档附件不会被解析为文本**，也就不会参与 RAG 上下文。

### 2. 文档上传 / 工作区构建的影响

- 上传新文档并希望将其向量化、加入工作区知识库时，通常需要经过“解析 → documents → 嵌入”的完整链路。解析环节高度依赖 Collector：
  - 对 PDF、Office、网页链接等复杂格式，如果 Collector 不在，主 server 无法完成解析。
  - 具体表现可能是：
    - 上传任务报错或卡在“解析中”。
    - 或者返回“无法解析/无法嵌入”的错误信息。

- 对于已经**完成解析并成功嵌入向量库的旧文档**：
  - 即便后续 Collector 停止，server 仍然可以基于现有向量库数据正常做 RAG 检索和回答。
  - 换言之，Collector 对“**已有文档的检索与回答**”不是运行时硬依赖。

### 3. 综述

- **必须依赖 Collector 的场景**：
  - 新上传文档（PDF/Office/网页/媒体等）需要被解析并加入知识库时。
  - 聊天会话中，想让**最新上传的附件内容**直接参与本轮对话上下文时。

- **不依赖 Collector 的场景**：
  - 只使用已经成功嵌入向量库的旧文档做 RAG 检索和回答。
  - 单纯的“无知识库纯聊天”（只走 LLM，不查向量）。

因此：

- Collector 未启动时，AnythingLLM **依然可以启动 server 并响应部分请求**，但“文档解析与新知识注入”能力会明显受限。
- 在生产部署中，应将 Collector 与主 server 一并启动，确保：
  - 文档上传、网页抓取、附件解析等功能正常工作；
  - 聊天会话中上传的文件可以即时参与 RAG 上下文。

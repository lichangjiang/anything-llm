# anything-llm 服务端环境配置与调试启动指南

本说明文档用于本地开发/调试 `server` 端（Express + Prisma + LLM/向量库）。建议先通读一遍，再按顺序操作。

---

## 1. 基础环境要求

- **操作系统**：Linux / macOS / WSL2（Windows）
- **Node.js 版本**：
  - 根目录 `package.json`：`"node": ">=18"`
  - `server/package.json`：`"node": ">=18.12.1"`
  - **建议**：使用 `Node 18.18+` 或 `Node 20 LTS`，并确保满足 `>=18.12.1`
- **包管理工具**：
  - 推荐 `yarn`（仓库脚本全部基于 yarn）
  - `yarn` 是基于 npm 的 JavaScript 包管理工具，相比直接使用 `npm`，在依赖安装速度、一致性和脚本管理上更友好，尤其适合 Monorepo 项目。
  - 安装方式通常为：
    - 已安装 Node.js 后执行：`npm install -g yarn`
    - 或通过发行版自带包管理器安装（如 `apt`, `brew` 等），以你当前系统最佳实践为准。
  - 在本项目中：
    - 所有初始化和运行脚本都以 `yarn xxx` 形式提供（如 `yarn setup`、`yarn dev:server`），建议全程使用 `yarn`，避免 `npm` / `pnpm` 混用导致的锁文件与依赖不一致。
- **数据库**：
  - 默认使用 Prisma 配置的数据库。
  - **Prisma 是什么？** 它是一个面向 TypeScript/JavaScript 的 ORM（对象关系映射）工具，用来：
    - 在 `prisma/schema.prisma` 中用声明式语法描述表结构和关联关系；
    - 通过 `npx prisma migrate` 等命令生成/更新数据库结构；
    - 在代码里用 `prisma.xxx.findMany/create/update` 这类方法读写数据库，而不用手写 SQL。
  - 在本项目中：
    - `server/models/*` 里大量通过 `require("../utils/prisma")` 来访问数据库；
    - 根目录的 `yarn prisma:setup`（或 `yarn prisma:migrate` 等脚本）实际就是调用 Prisma CLI 完成 schema 同步和初始化数据；
    - 对于你目前以“阅读 & 调试”为主的工作，只要知道 **Prisma 负责和数据库打交道**，能大概看懂它的查询/创建代码即可，并不需要精通其全部特性。
  - 本地开发默认可用 SQLite（参见 `server/.env.example` / `prisma/schema.prisma`），不需要额外安装数据库服务。

> 若你打算改为 MySQL / Postgres 等，请参考 `server/.env.example` 与 `prisma` 配置自行调整。

---

## 2. 全仓库依赖安装

在仓库根目录（`/home/lichangjiang/project/anything-llm`）执行一次性初始化：

```bash
# 1）安装根依赖 + server/frontend/collector 子项目依赖
yarn setup
```

`yarn setup` 会做几件事：

- 在 `server`、`frontend`、`collector` 等目录执行 `yarn`
- 调用 `yarn setup:envs` 拷贝各子项目 `.env.example` 到实际 `.env` 文件：
  - `frontend/.env.example` → `frontend/.env`
  - `server/.env.example` → `server/.env.development`
  - `collector/.env.example` → `collector/.env`
  - `docker/.env.example` → `docker/.env`
- 调用 `yarn prisma:setup`（`generate` + `migrate` + `seed`）完成 Prisma 初始化

**只要 `yarn setup` 成功执行一次，之后一般不需要重复。**

---

## 3. 服务端 `.env` 配置要点

主要文件：`server/.env.development`（由 `server/.env.example` 复制而来）。

建议按下面步骤检查/修改：

1. **端口与基础配置**
   - `SERVER_PORT`：开发环境默认常见为 `3001`，前端通过 `http://localhost:3001/api` 访问。
   - `NODE_ENV`：由 `yarn dev` 脚本通过 `cross-env NODE_ENV=development` 设置。

2. **数据库连接（Prisma）**
   - 找到形如 `DATABASE_URL` 的配置（具体键名以 `.env.example` 为准）。
   - **开发默认**通常是 SQLite 文件路径（例如 `file:./storage/anythingllm.db`），无需额外数据库服务。

3. **LLM 相关配置**（只列出常见关键项，具体以 `.env.example` 为准）：
   - `LLM_PROVIDER`：例如 `openai`、`ollama` 等。
   - 如果使用 OpenAI：
     - `OPENAI_API_KEY`（或与 provider 对应的 API Key 环境变量），必须配置正确。
   - 如果使用本地 LLM（如 Ollama）：
     - 确保本地服务已启动，并根据 `.env.example` 中的说明设置对应 URL / 模型名。

4. **向量库（Vector DB）配置**
   - `VECTOR_DB`：例如 `lancedb`（默认）、`qdrant`、`chroma` 等。
   - 对于远程向量库（Pinecone、Qdrant Cloud 等）：
     - 根据 `.env.example` 提示填写 API Key、URL、Index 名称等。
   - 对于本地向量库：
     - 确认本地服务已启动，或路径可写。
   - **调试/小规模使用建议**：
     - 如果只是本地开发、验证功能、向量数据量在单机可承受范围内，**直接使用默认的 `lancedb` 最简单**：
       - 不需要额外部署独立服务或数据库集群；
       - 只要磁盘可写即可开始使用，非常适合你当前的“阅读代码 + 本地调试”场景。
     - 只有在以下需求出现时才需要考虑 Pinecone / Qdrant / Weaviate 等远程向量库：
       - 多实例共享同一向量库、云上托管；
       - 数据量、并发或延迟要求超出本地嵌入式向量库能力时。

5. **遥测与隐私（可选）**
   - 关闭 Telemetry：设置 `DISABLE_TELEMETRY="true"`。

> 完整的可配置项请直接查看 `server/.env.example` 文件中的注释。

---

## 4. 仅启动服务端（Server）进行调试

### 4.1 在根目录使用统一脚本

在仓库根目录执行：

```bash
# 启动 server（开发模式）
yarn dev:server
```

这个脚本等价于：

```bash
cd server && yarn dev
```

### 4.2 在 `server` 目录单独启动

如果你已经在 `server` 目录下：

```bash
# 开发模式
cd server
yarn dev

# 生产模式（一般用于真实部署，不用于本地调试）
yarn start
```

`server/package.json` 中脚本说明：

- `dev`：
  - `cross-env NODE_ENV=development nodemon ... index.js`
  - 使用 `nodemon` 自动重启，适合开发/调试
- `start`：
  - `cross-env NODE_ENV=production node index.js`
  - 不带热重启，中/生产环境用

### 4.3 同时启动前端与 collector（可选）

如果你要从 UI 页面完整使用聊天功能进行调试，在根目录执行：

```bash
# 启动 server
yarn dev:server

# 启动 frontend
yarn dev:frontend

# 启动 collector（文档解析服务）
yarn dev:collector
```

或者一次性并行启动（会同时开三个进程）：

```bash
yarn dev:all
```

---

## 5. 验证 server 启动是否成功

1. **看日志输出**
   - 终端中应出现类似 `Server listening on port 3001` 或项目自定义的启动日志。
   - 若有 Prisma 连接错误、ENV 缺失等问题，会在这里报错。

2. **试探性访问健康接口 / 简单接口**
   - 查看 `server/endpoints/system.js` 等文件，选一个简单的 GET 或 POST 接口。
   - 在命令行用 `curl` 测试（示例，URL 以实际为准）：
     ```bash
     curl http://localhost:3001/api/system/health
     ```

3. **验证聊天接口可用性**（配合你前面写的 chat 文档）
   - 参考 `chat_模块解析.md` 中的第 7 节：
     - 使用 `curl -N` 访问：
       - `/api/workspace/:slug/stream-chat`
       - `/api/workspace/:slug/thread/:threadSlug/stream-chat`
     - 观察 SSE 输出与服务端日志，确认 `streamChatWithWorkspace` 正常执行。

---

## 6. 调试与依赖安装建议

### 6.1 调试建议

- **使用开发模式（`yarn dev`）**：
  - 依赖 `nodemon`，代码修改后自动重启，方便快速迭代。
- **日志位置**：
  - 核心入口：`server/index.js`
  - 聊天模块：`server/endpoints/chat.js`、`server/utils/chats/stream.js`
  - 向量库 & LLM：`server/utils/helpers.js`、`server/utils/AiProviders/*`、`server/utils/vectorDbProviders/*`
- **配合文档调试**：
  - Chat 模块调试：参考 `learing_doc/chat_模块解析.md` 中的调试章节。
  - 服务端整体结构：参考 `learing_doc/server_阅读指南.md`。

### 6.2 依赖安装与版本建议

- **Node 管理**：
  - 建议使用 `nvm` 或类似工具管理 Node 版本：
    - 仓库中有 `.nvmrc`（若存在），可以用 `nvm use` 对齐版本。
- **包安装**：
  - 全局依赖：`node`、`yarn`。
  - 项目依赖：全部由 `yarn setup` 自动安装。
- **Prisma 工具链（自动通过 devDependencies 安装）**：
  - `npx prisma generate`、`npx prisma migrate dev` 等命令均通过根目录脚本封装：
    - `yarn prisma:generate`
    - `yarn prisma:migrate`
    - `yarn prisma:seed`

> 一般情况下：**只需要在根目录执行一次 `yarn setup`，然后用 `yarn dev:server` 启动，就可以直接开始按照聊天模块文档进行调试学习。**

---

## 7. 推荐的学习/调试顺序（结合本地环境）

1. 按本指南完成环境准备：Node + yarn + `yarn setup`。
2. 检查并修改 `server/.env.development` 中的 LLM 与向量库配置，至少保证一种可用（例如：OpenAI + 默认 LanceDB）。
3. 执行：
   - `yarn dev:server`（必须）
   - 可选：`yarn dev:frontend`、`yarn dev:collector`。
4. 参考 `server_阅读指南.md` 理解整体结构。
5. 参考 `chat_模块解析.md`，从 `streamChatWithWorkspace` 开始逐步调试：
   - 先用 `curl` 触发请求
   - 再根据需要在关键点加日志或断点
6. 如需修改数据库结构或重置数据：
   - 使用根目录提供的 `yarn prisma:*` 系列命令（先确认清楚影响范围）。

通过上述步骤，你可以比较系统地从「环境准备 → Server 启动 → Chat 模块调试」这一整条链路来学习和验证 anything-llm 的服务端实现。

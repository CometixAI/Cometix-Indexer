## Cometix Indexer（MCP 服务器）

语义代码搜索的本地索引与检索服务。该项目实现了一个基于 Model Context Protocol（MCP）的服务端，封装了对 Cursor 后端 RepositoryService 的建库、同步与搜索流程，通过两类 MCP 工具对外提供能力：项目索引（index_project）与语义搜索（codebase_search）。

### 功能概述
- 索引：扫描本地工作区、生成文件清单、分批上传至 Cursor 服务端并完成建库标记。
- 增量同步：监听文件变更，按需进行轻量同步，保证搜索前的索引新鲜度。
- 语义搜索：调用远端检索接口并自动解密返回的加密路径，直观展示命中。
- 运行形态：作为 MCP 服务器通过 stdio 运行并响应工具调用。

### 目录结构（核心）
- `src/index.ts`：进程入口。解析 CLI/环境变量，创建 MCP `Server` 并接入 stdio 传输。
- `src/server.ts`：注册 MCP 工具：`index_project` 与 `codebase_search`。
- `src/services/repositoryIndexer.ts`：索引与同步核心逻辑（初次建库、分批上传、增量同步、定时器）。
- `src/services/codeSearcher.ts`：搜索逻辑（预同步、远端搜索、结果解密与规整）。
- `src/services/fileWatcher.ts`：文件变更监听，标记 `pendingChanges`。
- `src/services/stateManager.ts`：工作区状态持久化（`state.json`）。
- `src/crypto/pathEncryption.ts`：路径分段加解密方案与 Windows/Posix 互转。
- `src/client/proto.ts`：加载 `proto/repository_service.proto` 并以 protobuf 编解码发送 HTTP 请求。
- `src/client/cursorApi.ts`：封装调用的具体 RepositoryService 接口。
- `src/utils/env.ts`：配置解析、默认参数与请求头。
- `src/utils/fs.ts`：忽略规则、文件遍历与可嵌入文件清单读取。
- `src/utils/semaphore.ts`：并发控制与带重试的信号量。

### 工作原理
1) 初次索引
- 扫描工作区（忽略 `node_modules/`、`.git/`、`dist/` 等）并生成默认清单 `embeddable_files.txt`（每个工作区独立存放）。
- 基于 `@anysphere/file-service` 的 `MerkleClient` 构建目录 Merkle 树，获取 `rootHash` 与 `simhash`。
- 生成路径加密密钥（`pathKey`），并以 `V1MasterKeyedEncryptionScheme` 对相对路径逐段加密。
- 将文件按批（`INITIAL_UPLOAD_MAX_FILES`）执行完整流程：
  - `FastRepoInitHandshakeV2` 握手（返回 `codebaseId`）。
  - 上传本批文件（`FastUpdateFileV2`）。
  - `EnsureIndexCreated` 与 `FastRepoSyncComplete` 标记索引完成。
- 将 `codebaseId`、`pathKey`、`orthogonalTransformSeed` 等持久化到工作区状态 `state.json`。

2) 增量同步
- `chokidar` 监听文件变更，只标记 `pendingChanges = true`（轻量）。
- 搜索或定时器触发时，如存在变更：
  - 使用 `SyncMerkleSubtreeV2` 对目录节点进行比对，定位不匹配的子树与文件。
  - 对变更文件执行同批上传与 `EnsureIndexCreated`/`FastRepoSyncComplete`。
  - 清理 `pendingChanges` 标记并持久化。

3) 语义搜索
- 搜索前先触发一次按需增量同步以保证结果新鲜。
- 调用 `SearchRepositoryV2` 并对返回的加密路径用本地 `pathKey` 解密为 Posix 相对路径，输出 `{ path, score, startLine, endLine }`。

### 运行要求
- Node.js >= 18
- `proto/repository_service.proto` 必须存在（仓库已附带）。

### 安装与构建
```bash
npm install
npm run build
```

### 启动（npm scripts）
```bash
# 安装依赖并构建
npm install
npm run build

# 方式一：通过环境变量（PowerShell 示例）
$env:CURSOR_AUTH_TOKEN="你的Token"; npm run start

# 方式二：通过参数传递（-- 之后的参数会透传给脚本）
npm run start -- --auth-token 你的Token --base-url https://api2.cursor.sh --log-level info

# 开发模式（监听编译；运行需要另开终端执行 start）
npm run dev
# 另开一个终端
npm run start -- --auth-token 你的Token
```
可用环境变量：
- `CURSOR_AUTH_TOKEN`（必需）
- `CURSOR_BASE_URL`（默认 `https://api2.cursor.sh`）
- `LOG_LEVEL`（`debug` | `info` | `warning` | `error`，默认 `info`）

### 环境变量与默认值（可调优）
- `SYNC_CONCURRENCY`（默认 4）
- `SYNC_MAX_NODES`（默认 2000）
- `SYNC_MAX_ITERATIONS`（默认 10000）
- `SYNC_LIST_LIMIT`（默认 1000）
- `FILE_SIZE_LIMIT_BYTES`（默认 2MB，超出将跳过）
- `INITIAL_UPLOAD_MAX_FILES`（默认 10，初次索引分批大小）
- `PROTO_TIMEOUT_MS`（默认 30000）
- `PROTO_SEARCH_TIMEOUT_MS`（默认 60000）
- `AUTO_SYNC_INTERVAL_MS`（默认 5 分钟）

### MCP 工具
- `index_project`
  - 入参：`{ workspacePath: string; verbose?: boolean }`
  - 行为：初始化/刷新索引，按批全量上传并计划自动同步；当 `verbose=true` 时，额外返回本轮上传的相对路径文件列表。
  - 返回：`{ codebaseId, uploaded, batches, nextSyncAt }`

- `codebase_search`
  - 入参：`{ query: string; paths_include_glob?: string; paths_exclude_glob?: string; max_results?: number }`
  - 行为：在“唯一一个”已索引工作区内进行搜索；支持包含/排除 glob 过滤（基于工作区相对路径），并在搜索前进行按需增量同步。
  - 返回：`{ total, hits: Array<{ path, score, startLine, endLine }> }`

示例（概念性）：
```json
{
  "name": "index_project",
  "arguments": { "workspacePath": "E:/project" }
}
```
```json
{
  "name": "codebase_search",
  "arguments": { "query": "What is the xxx paper", "paths_include_glob": "src/**/*.rs", "paths_exclude_glob": "**/tests/**", "max_results": 50 }
}
```

### 状态与数据持久化
- 工作区专属数据目录：`%USERPROFILE%/.cometix/cursor-indexer/<safeName>-<hash>/`
  - `state.json`：保存 `codebaseId`、`pathKey`、`orthogonalTransformSeed` 等。
  - `embeddable_files.txt`：首次索引生成的可嵌入文件列表，可手动编辑以精确控制索引范围。

### 路径加密与兼容性
- 采用分段对称加密（`aes-256-ctr`）并在 Windows 相对路径（以 `./` 或 `.\` 起始）层面进行，避免泄露真实目录结构。
- 搜索结果会自动尝试使用本地 `pathKey` 解密为 Posix 相对路径，失败时回退为原始加密串。

### 忽略与限制
- 默认忽略：`node_modules/`、`.git/`、`.cursor/`、`dist/`、`build/`、`coverage/` 等。
- 超过 `FILE_SIZE_LIMIT_BYTES` 的文件会被跳过。

### 常见问题
- 报错 `repository_service.proto not found`：请确认项目根目录存在 `proto/repository_service.proto`。
- `Missing CURSOR_AUTH_TOKEN`：通过 `--auth-token` 传参或设置环境变量 `CURSOR_AUTH_TOKEN`。

### 许可
MIT



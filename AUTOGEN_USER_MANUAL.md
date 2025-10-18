# AutoGen 使用手冊（針對本倉庫）

> 適用對象：希望以 AutoGen 建構、擴充或維運多代理（multi-agent）應用的開發者與研究者。資料整理自倉庫所有原始碼、文件與範例，涵蓋 Python／.NET SDK、Studio、Magentic-One 等元件。

---

## 1. 整體概覽
- AutoGen 是一個多層式的代理框架，可在單機或分散式環境中協作 LLM、工具與人類。
- 主要 Python 套件：
  - `autogen-core`（核心訊息路由、Runtime、模型／工具介面）
  - `autogen-agentchat`（高階會話 API、代理與團隊）
  - `autogen-ext`（OpenAI/Azure/Anthropic 等模型、MCP、記憶、程式執行器…）
  - `autogen-studio`（視覺化無程式 GUI 與 Lite 模式）
  - `autogen-magentic-one` 與 `autogen_ext.teams.magentic_one`（Magentic-One 多代理工作流）
- 支援 .NET 生態系：新舊 API 併行，透過 `Microsoft.AutoGen.*` 套件提供事件驅動模型。
- 相關 Proto 定義（`protos/agent_worker.proto`, `protos/cloudevent.proto`）定義 gRPC Worker 與 CloudEvents 資料流。
- 承襲 AutoGen 0.2 的概念，同時大幅重構以提供事件驅動、彈性配置與跨語言能力。

---

## 2. 倉庫目錄速覽

| 路徑 | 說明 |
| --- | --- |
| `python/packages/autogen-core` | 核心 Runtime、Agent 基礎類別、訂閱與序列化、記憶、模型介面等 |
| `python/packages/autogen-agentchat` | 高階 Chat Agents、Teams、Termination、UI Console 等 |
| `python/packages/autogen-ext` | 模型客戶端、程式執行器、MCP/GraphRAG/HTTP 工具、記憶連接器、gRPC Runtime |
| `python/packages/autogen-studio` | Studio 後端（FastAPI）與前端（Gatsby），含 CLI `autogenstudio` |
| `python/packages/autogen-magentic-one` | 針對新版 AgentChat 的 Magentic-One 說明 |
| `python/packages/autogen-test-utils` | 遙測測試輔助函式 |
| `python/packages/component-schema-gen` | `gen-component-schema` CLI，用於產生組態 Schema |
| `python/packages/pyautogen` | 指向最新 `autogen-agentchat` 的代理套件 |
| `python/samples` | Python 範例（AgentChat、Core、Magentic-One、Task-Centric Memory 等） |
| `dotnet/` | .NET 解決方案、源碼、樣本、網站文件 |
| `docs/design` | 設計文檔：事件模型、Topic、Worker Protocol、服務拓撲等 |
| `protos/` | gRPC 與 CloudEvents Proto 定義 |
| `FAQ.md`, `TRANSPARENCY_FAQS.md` | 常見問題與負責任 AI 說明 |

---

## 3. 核心架構與資料流

### 3.1 訊息與 Runtime（`autogen-core`）
- **Agent 與 AgentType**：`autogen_core._agent.Agent` 為基底，搭配 `AgentId`（type + key）。`AgentType` 以反向 Domain 命名，和 `TopicId`（type + source）對應。
- **Runtime**：`AgentRuntime` 介面定義 `send_message` / `publish_message` / `register_factory` / `register_agent_instance` 等操作；`SingleThreadedAgentRuntime`（`_single_threaded_agent_runtime.py`）提供單執行緒 asyncio 佇列實作並支援介入（`InterventionHandler`）、取消權杖、OpenTelemetry。
- **訊息處理**：`RoutedAgent` 透過 `@message_handler`, `@event`, `@rpc` 裝飾器註冊處理函式；`MessageContext` 注入 sender/topic/是否為 RPC。
- **訂閱模型**：`Subscription` 介面與 `DefaultSubscription`、`TypeSubscription`、`TypePrefixSubscription` 允許以 Topic 轉換到 Agent；`default_subscription` 自動建立 `{AgentType}:` Prefix 訂閱。
- **序列化與工具**：`MessageSerializer` 提供 JSON/Protobuf；`autogen_core.tools` 有 `BaseTool`, `FunctionTool`, `Workbench`。
- **模型與記憶**：`autogen_core.models.ChatCompletionClient` 定義 `create` 與流式接口；`model_context` 提供記錄式、Token 限制上下文；`autogen_core.memory` 提供 `Memory` 抽象與 `ListMemory`。
- **程式執行**：`autogen_core.code_executor` 定義 `CodeExecutor` 基礎、需求宣告。
- **遙測與日誌**：`autogen_core.logging` 暴露 `LLMCallEvent`, `ToolCallEvent` 等結構化事件；TRACE / EVENT logger 名稱透過 `__init__.py` 匯出。

### 3.2 AgentChat 高階 API（`autogen-agentchat`）
- **ChatAgent 與 BaseChatAgent**：`base/_chat_agent.py`, `_base_chat_agent.py` 定義有狀態代理，提供 `run`, `run_stream`, `on_reset`, `save_state` 等；使用 `trace_create_agent_span`, `trace_invoke_agent_span` 注入遙測。
- **訊息型別**（`messages.py`）：`TextMessage`, `StructuredMessage`, `MultiModalMessage`, `HandoffMessage`, `ToolCallRequestEvent`, `ToolCallExecutionEvent`, `ThoughtEvent`, `ModelClientStreamingChunkEvent`, `StopMessage` 等，皆為 Pydantic 模型，支援 `to_text` / `to_model_message`.
- **主要代理**：
  - `AssistantAgent`：整合模型、工具、多模型上下文、串流模式、結構化輸出、工具總結、handoff。
  - `CodeExecutorAgent`：解析 Markdown 程式碼區塊，配合 `CodeExecutor` 執行，支援重試、人工核准（`ApprovalRequest/ApprovalResponse`）與工具模式。
  - `UserProxyAgent`：將人類輸入函式包裝成 Agent，可使用同步/非同步輸入函式、handoff 事件、取消 token。
  - `SocietyOfMindAgent`, `MessageFilterAgent` 等衍生類別。
- **Teams 與設計模式**（`teams/_group_chat/`）：
  - `RoundRobinGroupChat`, `SelectorGroupChat`, `SwarmGroupChat`, `GraphGroupChat`（可用 `GraphBuilder` 自訂結構）、`SequentialRoutedAgent`, `MagenticOneGroupChat`。
  - `Team` 基底類別提供 `run`, `reset`, `pause`, `save_state`，狀態儲存在 `state/_states.py`。
- **工具包裝**：`AgentTool`, `Team`, `TaskRunnerTool` 將 Agent/Team 暴露為可呼叫工具，供其他 Agent 使用。
- **Termination 條件**（`conditions/_terminations.py`）：`StopMessageTermination`, `MaxMessageTermination`, `TextMentionTermination`, `FunctionalTermination`, `HandoffTermination`, `SourceMatchTermination` 等。
- **UI**：`ui.Console` 文字流顯示、`autogen_ext.ui.RichConsole` 使用 rich 呈現，支援影像、token 統計與人機互動回呼。

### 3.3 擴充層（`autogen-ext`）
- **模型客戶端**：
  - `models/openai`：`OpenAIChatCompletionClient`, `AzureOpenAIChatCompletionClient`，支援串流、並行工具呼叫、Structured Output、Token 計數（`tiktoken`）、用量記錄、Stop reason 正規化。
  - `models/anthropic`, `models/azure`（Azure AI Inference）、`models/ollama`, `models/llama_cpp`, `models/semantic_kernel`, `models/cache`（快取包裝器）、`models/replay`（重播回應）等。
  - 所有模型皆符合 `autogen_core.models.ChatCompletionClient` 介面並提供 `Component` 組態。
- **程式執行**：`code_executors` 包含 `DockerCommandLineCodeExecutor`, `LocalCommandLineCodeExecutor`, `JupyterKernelCodeExecutor`, `Azure` 支援、`create_default_code_executor` 自動選擇最佳環境。
- **工具與工作台**：
  - `tools/mcp`：提供 `McpWorkbench`, `mcp_server_tools`, 支援 `StdioServerParams`, `SseServerParams`, `StreamableHttpServerParams`, `RootsProvider`、`Sampler` 等，將 MCP Server 工具化。
  - `tools/code_execution.PythonCodeExecutionTool`, `tools/graphrag`（GraphRAG Global/Local Search 工具）、`tools/http.HttpTool`, `tools/langchain`, `tools/semantic_kernel`, `tools/azure.AISearchTool`。
- **記憶與快取**：`memory` 連接 Canvas, Redis, Chroma, Mem0；`cache_store` 提供 DiskCache / Redis 結構化快取。
- **Runtime**：`runtimes/grpc` 實作 Worker Runtime 與 Host，對應 `protos/agent_worker.proto`，支援分散式 Worker <-> Gateway 佈署。
- **Experimental**：`experimental/task_centric_memory` 提供 Task-Centric Memory Controller、MemoryBank、字串相似度、快速學習流程與完整 README。
- **Agents**：`agents/web_surfer.MultimodalWebSurfer`, `agents/file_surfer.FileSurfer`, `agents/magentic_one.MagenticOneCoderAgent`, `agents/openai` 等專用代理。
- **UI**：`autogen_ext.ui.RichConsole` 提供彩色面板與影像輸出，在 iTerm2 可內嵌顯示圖片。

### 3.4 AutoGen Studio（`python/packages/autogen-studio`）
- FastAPI 後端 + Gatsby 前端，CLI：`autogenstudio`.
- 功能：
  - Studio（資料庫 + UI）透過 SQLModel 儲存 Agent/Skill/Workflow，支援 SQLite/PostgreSQL 等，使用 `--database-uri` 指定。
  - Studio Lite：快速實驗模式，可從 CLI 或程式碼呼叫 `LiteStudio`（`autogenstudio.lite`），無須資料庫。
- 重要 CLI 參數：`--appdir`, `--database-uri`, `--reload`, `--upgrade-database`, `--session-name`, `--team`, `--auto-open`。
- 需安裝 Node/Gatsby 以建置前端，或使用 Dev Container。

### 3.5 Magentic-One（`autogen_ext.teams.magentic_one`）
- 基於 AgentChat Team 實作的 Orchestrator，整合 `MultimodalWebSurfer`, `FileSurfer`, `MagenticOneCoderAgent`, `CodeExecutorAgent`, `UserProxyAgent`。
- `MagenticOne` 類別提供簡化建構，支援 Human-in-the-loop（`hil_mode`）、程式執行核准、Docker 執行器、預設計畫／回顧邏輯。
- README 指出安全建議：容器隔離、日誌監控、限制資源、警惕 Prompt Injection。

---

## 4. 安裝與環境設定

### 4.1 Python 套件（使用者）
```bash
# 基礎 AgentChat + OpenAI 擴充
pip install -U "autogen-agentchat" "autogen-ext[openai]"

# 安裝 Studio
pip install -U "autogenstudio"

# 取得 Magentic-One 全套工具
pip install -U "autogen-ext[magentic-one]"
```
環境變數：依模型供應商設定 `OPENAI_API_KEY`, `AZURE_OPENAI_API_KEY`, `ANTHROPIC_API_KEY` 等。

### 4.2 Python 開發環境（本倉庫）
1. 進入 `python/`。
2. 安裝 `uv` 後執行：
   ```bash
   uv sync --all-extras
   source .venv/bin/activate
   ```
3. 常用 `poe` 腳本：（`python/README.md`）
   - `poe format` / `poe lint` / `poe test` / `poe mypy` / `poe pyright`
   - `poe docs-build` / `poe docs-serve` / `poe docs-check`
   - `poe samples-code-check`（驗證 `python/samples`）
   - `poe check` 一次執行全部。
4. 更新依賴：`uv sync --all-extras`
5. 文件建置：`poe docs-clean && poe docs-build`

### 4.3 .NET 套件
- 於專案新增所需套件（`dotnet/website/articles/Installation.md`）：
  ```bash
  dotnet add package AutoGen      # 一次含核心與常用擴充
  dotnet add package Microsoft.AutoGen # 新事件驅動 API
  dotnet add package AutoGen.OpenAI   # 與 OpenAI 整合
  ```
- 若需夜間版，新增 Azure DevOps Feed：`dotnet nuget add source https://pkgs.dev.azure.com/AGPublish/...`
- .NET 參考範例：`dotnet/samples/Hello`, `AgentChat`, `GettingStartedGrpc`, `dev-team`。

---

## 5. 快速入門示例

### 5.1 單代理 Hello World（`README.md`）
```python
import asyncio
from autogen_agentchat.agents import AssistantAgent
from autogen_ext.models.openai import OpenAIChatCompletionClient

async def main() -> None:
    model_client = OpenAIChatCompletionClient(model="gpt-4.1")
    agent = AssistantAgent("assistant", model_client=model_client)
    print(await agent.run(task="Say 'Hello World!'"))
    await model_client.close()

asyncio.run(main())
```

### 5.2 MCP 工具整合
```python
from autogen_ext.tools.mcp import McpWorkbench, StdioServerParams

server_params = StdioServerParams(command="npx", args=["@playwright/mcp@latest", "--headless"])
async with McpWorkbench(server_params) as mcp:
    agent = AssistantAgent(..., workbench=mcp, max_tool_iterations=10)
```
> 僅連線信任的 MCP 伺服器，避免任意命令執行。

### 5.3 多代理協作
```python
math_agent_tool = AgentTool(math_agent, return_value_as_last_message=True)
chemistry_agent_tool = AgentTool(chemistry_agent, return_value_as_last_message=True)
assistant = AssistantAgent("assistant", tools=[math_agent_tool, chemistry_agent_tool], max_tool_iterations=10)
```

### 5.4 Magentic-One（`autogen_ext/teams/magentic_one.py`）
```python
from autogen_ext.teams.magentic_one import MagenticOne
client = OpenAIChatCompletionClient(model="gpt-4o")
m1 = MagenticOne(client=client, hil_mode=True)
result = await Console(m1.run_stream(task="Plan a 3-day trip to Seattle"))
```

### 5.5 Task-Centric Memory
`python/packages/autogen-ext/src/autogen_ext/experimental/task_centric_memory/README.md` 提供完整範例，可將記憶整合進 `RoutedAgent` 或 AgentChat Team。

---

## 6. Agent 與訊息生命週期

1. **Topic 發佈**：訊息以 CloudEvents 表示（`docs/design/01 - Programming Model.md`、`02 - Topics.md`）。`TopicId = (type, source)`，Agent 訂閱特定 Topic 或 Prefix。
2. **Runtime 路由**：`AgentRuntime` 依訂閱查詢對應 Agent，如 Agent 尚未實例化會動態建立。
3. **MessageContext**：`MessageContext(sender, topic_id, is_rpc, cancellation_token, message_id)`，供 handler 判斷來源與是否 RPC。
4. **Hand-off 與工具**：`AssistantAgent` 可侦測工具呼叫；透過 `Handoff` 產生 `FunctionTool` 將控制權交給指定 Agent。
5. **狀態管理**：Agent 與 Team 必須實作 `save_state` / `load_state`；常用 state 類別存放於 `autogen_agentchat.state`.
6. **事件記錄**：模型呼叫、工具執行、串流開始／結束會記錄 `LLMCallEvent` 等 JSON 日誌（`autogen_core/logging.py`）。
7. **取消**：`CancellationToken` 可用於停止長時間任務、取消用户輸入等待或工具執行。

---

## 7. AgentChat 元件總表

### 7.1 Agents
- `AssistantAgent` (`autogen_agentchat/agents/_assistant_agent.py`): 支援串流、Structured Output、工具並行、回顧模式、記憶查詢事件。
- `UserProxyAgent` (`_user_proxy_agent.py`): 人類代理，支援同步／非同步輸入函式與取消。
- `CodeExecutorAgent` (`_code_executor_agent.py`): 程式碼生成 + 執行，支援程式碼審批與多語言模式。
- `MessageFilterAgent`, `SocietyOfMindAgent`, `CodeExecutorAgent`, `ClosureAgent` 等。

### 7.2 Messages / Events
- Chat 訊息：`TextMessage`, `StructuredMessage`, `ToolCallSummaryMessage`, `HandoffMessage`.
- 事件：`ThoughtEvent`, `ToolCallRequestEvent`, `ToolCallExecutionEvent`, `ModelClientStreamingChunkEvent`, `MemoryQueryEvent`, `UserInputRequestedEvent`.
- 停止訊息：`StopMessage`.

### 7.3 Teams
- Group Chats：`RoundRobinGroupChat`, `SelectorGroupChat`, `SwarmGroupChat`, `GraphGroupChat`, `MagenticOneGroupChat`.
- `ChatAgentContainer` 用於包裝 Agent + 狀態；`BaseGroupChatManager` 在 `state` 中保存輪次、訊息串。

### 7.4 TerminationConditions
- `StopMessageTermination`, `MaxMessageTermination`, `TextMentionTermination`, `FunctionalTermination`, `ToolCallTermination`, `HandoffTermination`, `SourceMatchTermination` 等，用於控制 Team 停止條件。

### 7.5 工具與組態
- `AgentTool` / `TeamTool`（`autogen_agentchat/tools`）將 Agent/Team 暴露為工具；`TaskRunnerTool` 可從 TaskRunner 產出工具。
- `Component` 模型（`autogen_core._component_config`）允許將 Agent/Model/Tool 序列化為 `ComponentModel`，支援 `_to_config` / `_from_config` 與 `ComponentLoader.load_component`。
- `component-schema-gen` CLI 可輸出既有組件的 JSON Schema。

---

## 8. 擴充與整合

### 8.1 模型客戶端（`autogen_ext.models`）
- **OpenAI / Azure**：支援串流、工具、structured response、r1 content 解析（`_message_transform.py`）、Stop Reason 正規化。
- **Anthropic / Bedrock**：整合 `AsyncAnthropic`、工具選擇、影像 MIME 判斷、Thinking 模式。
- **Ollama / Llama.cpp**：本地模型支援。
- **Semantic Kernel**：作為 ChatCompletion Adapter。
- **Cache / Replay**：可包裝其他模型以實作快取或重播。

### 8.2 程式碼執行
- `DockerCommandLineCodeExecutor`, `LocalCommandLineCodeExecutor`, `JupyterKernelCodeExecutor`, `DockerJupyterCodeExecutor`, `AzureContainerApps` 等。
- `CodeExecutionInput` / `CodeExecutionResult` 描述輸入輸出，支援多語言。
- 安全建議：優先使用容器；`approval_func`、`hil_mode` 保持人類審核。

### 8.3 MCP 與工具
- `McpWorkbench` 提供工具清單與呼叫介面；支援 STDIO / SSE / HTTP Workbench；可結合 `AssistantAgent(workbench=...)`。
- `mcp_server_tools` 可直接從 MCP Server 產生工具列表。
- `ChatCompletionClientSampler`, `Sampler` 等可對工具進行取樣或根據模型輸出擷取資訊。

### 8.4 記憶與快取
- `autogen_ext.memory`：Canvas API、Redis、Chroma、Mem0；皆符合 `autogen_core.memory.Memory`。
- `task_centric_memory`：提供 `MemoryController`、`MemoryBank`、`MemoryPrompter` 等模組，可快速將記憶整合到 Agent。
- `cache_store`：`DiskCacheStore`, `RedisStore`，可供模型或其他組件快取結果。

### 8.5 gRPC Runtime 與 Proto
- `autogen_ext.runtimes.grpc` 實作 Worker Runtime Host，對應 `AgentRpc.OpenChannel` 雙向串流、`RegisterAgentType`, `AddSubscription` 等 RPC。
- `protos/agent_worker.proto` 定義 `RpcRequest`, `RpcResponse`, `ControlMessage`、`Subscription`；`cloudevent.proto` 引入 CloudEvents 標準欄位。
- 用例：在分散式環境中將 Agent Worker 部署於不同節點，由 Gateway 進行 Placement 與 Routing（`docs/design/03 - Agent Worker Protocol.md`, `05 - Services.md`）。

---

## 9. AutoGen Studio 操作重點
- **安裝**：`pip install autogenstudio` 或在倉庫 `pip install -e .`。
- **啟動**：
  ```bash
  autogenstudio ui --port 8080 --appdir ~/.autogenstudio --database-uri sqlite:///database.sqlite
  ```
- **Lite 模式**：
  ```bash
  autogenstudio lite --team ./team.json --session-name "My Experiment" --auto-open
  ```
  或程式化使用 `from autogenstudio.lite import LiteStudio`。
- **資料庫**：依 README 指示，SQLModel + Alembic；可使用 `--upgrade-database` 升級 schema。
- **前端建置**：需 Node/Yarn，於 `frontend` 執行 `yarn build`；若未安裝 git lfs 需補建。
- **更新紀錄**：2024-11-14 已開始改寫以支援 AgentChat 0.4 API。

---

## 10. Magentic-One 作業流程
- Orchestrator 制定 Task/Progress Ledger，指派 Web/File/Coder/Executor 子代理。
- `hil_mode=True` 時加入 `UserProxyAgent`，允許人類確認步驟；`approval_func` 可僅針對程式執行要求核准。
- 依 README 建議：
  1. 使用 Docker 隔離。
  2. 監控日誌、限制外部資源。
  3. 警惕 prompt injection 與自動化行為（例如接受 Cookie、嘗試招募人類協助）。

---

## 11. 範例與模板

### 11.1 Python Samples（`python/samples`）
- AgentChat：
  - `agentchat_fastapi`, `agentchat_chainlit`, `agentchat_streamlit`：整合 Web UI。
  - `agentchat_graphrag`, `agentchat_dspy`, `agentchat_azure_postgresql`, `agentchat_chess_game`.
- Core Runtime：
  - `core_streaming_response_fastapi`, `core_semantic_router`, `core_grpc_worker_runtime`, `core_distributed-group-chat`, `core_xlang_hello_python_agent`.
- Magentic-One：
  - `magentic-one-cli`（另有獨立套件）。
- Task-Centric Memory：
  - `samples/task_centric_memory`：展示如何在 Team 外掛記憶。
- 其他：`gitty`（CLI 草稿回覆生成），`core_async_human_in_the_loop` 等。

### 11.2 .NET Samples（`dotnet/samples`）
- `Hello`: 最小範例（`AutoGen.OpenAI`）。
- `AgentChat`: 多 Agent 聊天、群組聊天。
- `GettingStarted` / `GettingStartedGrpc`: 新事件驅動 API、gRPC Worker。
- `dev-team`: 包含多檔案與 DevOps 相關示例。

### 11.3 模板
- `python/templates/new-package`：Cookiecutter 模板，可快速建立新的 AutoGen 套件。

---

## 12. 測試與品質保證
- **Python**：
  - 執行 `poe test`, `poe mypy`, `poe pyright`，確保靜態型別與測試通過。
  - `poe docs-check` 驗證文件，`poe docs-check-examples` 檢查 API 參考範例程式碼區塊。
  - `poe samples-code-check` 保證 samples 可執行。
- **.NET**：
  - 在 `dotnet/` 使用 `dotnet test`.
  - `AutoGen.SourceGenerator` 需搭配 Roslyn Source Generator 測試。
- **CI**：倉庫包含 GitHub Actions（例如 `dotnet-build.yml`），可參考以建立自動驗證流程。

---

## 13. 日誌、遙測與除錯
- 事件記錄呼叫 `logging.getLogger(EVENT_LOGGER_NAME)`（`autogen_agentchat/__init__.py`, `autogen_core/__init__.py`）。
- `LLMCallEvent` / `LLMStreamStartEvent` / `LLMStreamEndEvent` / `ToolCallEvent` 提供 JSON 字串，可直接寫入記錄系統。
- `MessageHandlerContext.agent_id()` 讓工具／模型呼叫取得目前 Agent 標識，便於追蹤。
- 遙測：`autogen_core._telemetry` 使用 OpenTelemetry Tracer Provider；可設 `AUTOGEN_DISABLE_RUNTIME_TRACING=true` 停用。

---

## 14. 分散式架構與服務
- `docs/design/03 - Agent Worker Protocol.md` 描述 Worker 啟動、註冊 AgentType、接收 `Event` / `RpcRequest`、回覆 `RpcResponse`。
- `docs/design/05 - Services.md`：服務組件包括 Gateway（RPC + Event Bus bridge）、Registry、AgentState、Routing；Roadmap 包含排程與探索。
- `protos/agent_worker.proto` 定義 `AgentRpc` 服務與 Control Channel（存取狀態 Save/Load）。
- Gated placement：Service 根據 Agent 名稱（type + key）映射到 Worker；Worker Advertise 可 hosting 的 Agent 名稱集合。

---

## 15. .NET API 重點
- 舊版套件 `AutoGen.*` 保留 0.2 API；新版事件驅動 API 在 `Microsoft.AutoGen.*`。
- 基礎程式碼：
  ```csharp
  var openAIKey = Environment.GetEnvironmentVariable("OPENAI_API_KEY");
  var config = new OpenAIConfig(openAIKey, "gpt-3.5-turbo");
  var assistant = new AssistantAgent(name: "assistant", systemMessage: "...", llmConfig: new ConversableAgentConfig { ConfigList = [config] }).RegisterPrintMessage();
  var user = new UserProxyAgent(name: "user", humanInputMode: HumanInputMode.ALWAYS).RegisterPrintMessage();
  await user.InitiateChatAsync(assistant, "Hey assistant...", maxRound: 10);
  ```
- 功能：
  - ConversableAgent 支援函式呼叫、.NET Code Execution（透過 `dotnet-interactive`）。
  - 群組聊天、工具支援。
  - `AutoGen.SourceGenerator` 生成型別安全工具定義。
  - `AutoGen.DotnetInteractive` 可在 C#/F#/PowerShell/Python 執行程式碼區塊。
  - `AutoGen.WebAPI`／`AutoGen.AzureAIInference` 等擴充。

---

## 16. 常見問題與支援
- `FAQ.md`：說明 0.4 為重寫版本，仍維護 0.2；鼓勵早期採用者嘗試並提供回饋；官方支持渠道為 GitHub Issues/Discussions 與新 Discord（`aka.ms/autogen-discord`）。
- `TRANSPARENCY_FAQS.md`：闡述 AutoGen 的用途限制、負責任 AI 評估、可能風險（資料偏差、不透明、誤用、隱私、安全等），建議配合內容審查與人類監督。
- `SUPPORT.md`, `SECURITY.md`, `CONTRIBUTING.md`：提供貢獻、回報安全問題、取得協助的程序。

---

## 17. 最佳實務與安全建議
- **模型金鑰管理**：使用 `.env` 或安全祕密管理；避免硬編碼。
- **程式執行**：預設使用 Docker Executor，並設定 `approval_func` 或 `hil_mode` 進行人工審查。
- **MCP／工具**：僅導入信任的 Server；限制工具能力。
- **記憶體與快取**：儲存敏感資料前加密或置於隔離環境。
- **日誌**：避免在事件記錄中洩漏個資；必要時 redact。
- **責任使用**：遵守模型供應商政策（OpenAI Usage Policies、Azure OpenAI Code of Conduct 等）。

---

## 18. 進一步閱讀與資源
- 官方文件（最新版）：https://microsoft.github.io/autogen/
- AgentChat 遷移指南：`python/migration_guide.md`
- 設計文檔：`docs/design/*.md`
- Magentic-One 技術報告：https://arxiv.org/abs/2411.04468
- AutoGen 部落格與 Discord：README 徽章提供連結。

---

## 19. 推薦下一步
- 依需求選擇 API 層級：快速原型使用 `autogen-agentchat`，複雜互動／分散式場景深入 `autogen-core`。
- 使用 `python/samples` 作為起點建立自訂代理或工作流。
- 若採用視覺化流程，安裝並試用 AutoGen Studio（Lite 或完整版）。
- 打算整合至企業環境時，評估 gRPC Worker Runtime 與服務拓撲，並導入適當的日誌／遙測管線。
- 持續關注 FAQ 與版本公告，必要時更新遷移程式碼。

---

> 本手冊內容根據倉庫最新程式碼與文件歸納，實作時請搭配對應 `.py`/`.cs` 檔案（例如 `python/packages/autogen-agentchat/src/autogen_agentchat/agents/_assistant_agent.py:1`、`python/packages/autogen-core/src/autogen_core/_single_threaded_agent_runtime.py:1`、`dotnet/src/Microsoft.AutoGen` 等）查閱細節。


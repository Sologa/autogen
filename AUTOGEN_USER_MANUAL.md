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
| `python/packages/pyautogen` | 指向最新 `autogen-agentchat` 的代理套件 |
| `python/samples` | Python 範例（AgentChat、Core、Magentic-One、Task-Centric Memory 等） |
| `dotnet/` | .NET 解決方案、源碼、樣本、網站文件 |
| `docs/design` | 設計文檔：事件模型、Topic、Worker Protocol、服務拓撲等 |
| `protos/` | gRPC 與 CloudEvents Proto 定義 |
| `FAQ.md`, `TRANSPARENCY_FAQS.md` | 常見問題與負責任 AI 說明 |

---

## 3. 核心架構與資料流

### 3.1 訊息與 Runtime（`autogen-core`）
- **Agent 與識別**：`autogen_core._agent.Agent` 為共通介面 ([source](python/packages/autogen-core/src/autogen_core/_agent.py))，搭配 `AgentId(type, key)` 與 `AgentType` 字元限制定義於 ID 規格文件 ([source](docs/design/04%20-%20Agent%20and%20Topic%20ID%20Specs.md), [source](https://microsoft.github.io/autogen/dev/user-guide/core-user-guide/core-concepts/agent-identity-and-lifecycle.html))；Topic Type 需符合字元限制與型別/來源格式 ([source](docs/design/02%20-%20Topics.md), [source](https://microsoft.github.io/autogen/dev/user-guide/core-user-guide/core-concepts/topic-and-subscription.html)).
- **Runtime**：`AgentRuntime` 介面定義 `send_message` / `publish_message` / `register_factory` / `register_agent_instance` 等操作 ([source](python/packages/autogen-core/src/autogen_core/_agent_runtime.py), [source](https://microsoft.github.io/autogen/dev/user-guide/core-user-guide/framework/agent-and-agent-runtime.html))；`SingleThreadedAgentRuntime` 以 asyncio 事件迴圈與佇列路由訊息，支援介入器、取消權杖與 OpenTelemetry ([source](python/packages/autogen-core/src/autogen_core/_single_threaded_agent_runtime.py)).
- **訊息處理**：`RoutedAgent` 透過 `@message_handler`, `@event`, `@rpc` 裝飾器註冊處理函式 ([source](python/packages/autogen-core/src/autogen_core/_routed_agent.py))；`MessageContext` 注入 sender/topic/RPC 等上下文 ([source](python/packages/autogen-core/src/autogen_core/_message_context.py)).
- **訂閱模型**：`Subscription` 介面與 `DefaultSubscription`、`TypeSubscription`、`TypePrefixSubscription` 允許以 Topic 對應 Agent，`default_subscription` 預設建立 `{AgentType}:` 前綴直送通道 ([source](python/packages/autogen-core/src/autogen_core/_subscription.py)).
- **序列化與工具**：JSON / Protobuf 序列化由 `MessageSerializer` 實作 ([source](python/packages/autogen-core/src/autogen_core/_serialization.py))；`autogen_core.tools` 匯出 `BaseTool`, `FunctionTool`, `Workbench` 等基礎工具類別 ([source](python/packages/autogen-core/src/autogen_core/tools/__init__.py)).
- **模型與記憶**：`autogen_core.models.ChatCompletionClient` 定義模型呼叫介面 ([source](python/packages/autogen-core/src/autogen_core/models/__init__.py))；`model_context` 模組提供各式上下文管理器 ([source](python/packages/autogen-core/src/autogen_core/model_context/__init__.py))；`autogen_core.memory` 匯出 `Memory`、`ListMemory` 等實作 ([source](python/packages/autogen-core/src/autogen_core/memory/__init__.py)).
- **程式執行**：`autogen_core.code_executor` 匯出 `CodeExecutor` 抽象與需求宣告工具 ([source](python/packages/autogen-core/src/autogen_core/code_executor/__init__.py)).
- **遙測與日誌**：`autogen_core.logging` 暴露 `LLMCallEvent`, `ToolCallEvent` 等結構化事件 ([source](python/packages/autogen-core/src/autogen_core/logging.py)).
### 3.2 AgentChat 高階 API（`autogen-agentchat`）
- **ChatAgent 與 BaseChatAgent**：`base/_chat_agent.py` ([source](python/packages/autogen-agentchat/src/autogen_agentchat/base/_chat_agent.py)) 與 `agents/_base_chat_agent.py` ([source](python/packages/autogen-agentchat/src/autogen_agentchat/agents/_base_chat_agent.py)) 定義具狀態代理的共通行為，提供 `run`, `run_stream`, `on_reset`, `save_state` 等並串接遙測 ([source](https://microsoft.github.io/autogen/dev/user-guide/agentchat-user-guide/tutorial/agents.html)).
- **訊息型別**：`TextMessage`, `StructuredMessage`, `MultiModalMessage`, `ToolCallRequestEvent`, `ToolCallExecutionEvent`, `ThoughtEvent`, `ModelClientStreamingChunkEvent`, `StopMessage` 等皆以 Pydantic 建模，支援轉換與序列化 ([source](python/packages/autogen-agentchat/src/autogen_agentchat/messages.py), [source](python/packages/autogen-agentchat/src/autogen_agentchat/events.py), [source](https://microsoft.github.io/autogen/dev/user-guide/agentchat-user-guide/tutorial/messages.html)).
- **主要代理**：`AssistantAgent`, `CodeExecutorAgent`, `UserProxyAgent`, `MessageFilterAgent`, `SocietyOfMindAgent` 等提供模型對話、程式執行、人類互動、訊息過濾等能力 ([source](python/packages/autogen-agentchat/src/autogen_agentchat/agents/_assistant_agent.py), [source](python/packages/autogen-agentchat/src/autogen_agentchat/agents/_code_executor_agent.py), [source](python/packages/autogen-agentchat/src/autogen_agentchat/agents/_user_proxy_agent.py), [source](python/packages/autogen-agentchat/src/autogen_agentchat/agents/_message_filter_agent.py), [source](python/packages/autogen-agentchat/src/autogen_agentchat/agents/_society_of_mind_agent.py), [source](https://microsoft.github.io/autogen/dev/user-guide/agentchat-user-guide/custom-agents.html)).
- **Teams 與設計模式**：`RoundRobinGroupChat`, `SelectorGroupChat`, `SwarmGroupChat`, `GraphGroupChat`, `MagenticOneGroupChat` 等群聊管理器座落於 `teams/_group_chat` ([source](python/packages/autogen-agentchat/src/autogen_agentchat/teams/_group_chat)), 並在官方指南中介紹相應模式與使用情境 ([source](https://microsoft.github.io/autogen/dev/user-guide/agentchat-user-guide/tutorial/teams.html), [source](https://microsoft.github.io/autogen/dev/user-guide/agentchat-user-guide/selector-group-chat.html), [source](https://microsoft.github.io/autogen/dev/user-guide/agentchat-user-guide/swarm.html), [source](https://microsoft.github.io/autogen/dev/user-guide/agentchat-user-guide/magentic-one.html)).
- **工具與停止條件**：`AgentTool`、`TeamTool`、`TaskRunnerTool` 將 Agent/Team 暴露為工具 ([source](python/packages/autogen-agentchat/src/autogen_agentchat/tools/__init__.py), [source](https://microsoft.github.io/autogen/dev/user-guide/agentchat-user-guide/tutorial/agents.html#agent-as-a-tool))；`conditions/_terminations.py` 提供 `StopMessageTermination`, `TextMentionTermination`, `HandoffTermination` 等停止條件 ([source](python/packages/autogen-agentchat/src/autogen_agentchat/conditions/_terminations.py), [source](https://microsoft.github.io/autogen/dev/user-guide/agentchat-user-guide/tutorial/termination.html)).
- **UI**：`autogen_agentchat.ui.Console` 可串流觀察對話，亦可結合 `autogen_ext.ui.RichConsole` 呈現多模態輸出 ([source](python/packages/autogen-agentchat/src/autogen_agentchat/ui.py), [source](python/packages/autogen-ext/src/autogen_ext/ui.py), [source](https://microsoft.github.io/autogen/dev/user-guide/agentchat-user-guide/logging.html)).

### 3.3 擴充層（`autogen-ext`）
- **模型客戶端**：`autogen_ext.models.openai`、`anthropic`、`azure`、`ollama`、`llama_cpp`、`semantic_kernel`、`cache`、`replay` 等模組提供 OpenAI、人類回饋、Azure AI Inference、本地模型與快取重播能力，全部符合 `ChatCompletionClient` 介面 ([source](python/packages/autogen-ext/src/autogen_ext/models/openai/__init__.py), [source](python/packages/autogen-ext/src/autogen_ext/models/anthropic/__init__.py), [source](python/packages/autogen-ext/src/autogen_ext/models/azure/__init__.py), [source](python/packages/autogen-ext/src/autogen_ext/models/ollama/__init__.py), [source](python/packages/autogen-ext/src/autogen_ext/models/llama_cpp/__init__.py), [source](python/packages/autogen-ext/src/autogen_ext/models/semantic_kernel/__init__.py), [source](python/packages/autogen-ext/src/autogen_ext/models/cache/__init__.py), [source](python/packages/autogen-ext/src/autogen_ext/models/replay/__init__.py), [source](https://microsoft.github.io/autogen/dev/user-guide/extensions-user-guide/index.html)).
- **程式碼執行**：`code_executors/__init__.py` 匯出 Docker、本機、Jupyter、Azure Container Apps 等執行器與 `create_default_code_executor` 自動挑選邏輯 ([source](python/packages/autogen-ext/src/autogen_ext/code_executors/__init__.py), [source](https://microsoft.github.io/autogen/dev/user-guide/extensions-user-guide/code-executors/index.html)).
- **工具與工作台**：`tools/mcp` 提供 `McpWorkbench`, `mcp_server_tools`, `RootsProvider`, `Sampler` 等整合 MCP 能力 ([source](python/packages/autogen-ext/src/autogen_ext/tools/mcp/_workbench.py), [source](python/packages/autogen-ext/src/autogen_ext/tools/mcp/_factory.py), [source](python/packages/autogen-ext/src/autogen_ext/tools/mcp/_host.py), [source](https://microsoft.github.io/autogen/dev/user-guide/extensions-user-guide/mcp/index.html))；`tools/http`, `tools/graphrag`, `tools/langchain`, `tools/semantic_kernel` 等亦提供常用外部資源介接 ([source](python/packages/autogen-ext/src/autogen_ext/tools/http/__init__.py), [source](python/packages/autogen-ext/src/autogen_ext/tools/graphrag/__init__.py), [source](python/packages/autogen-ext/src/autogen_ext/tools/langchain/__init__.py), [source](python/packages/autogen-ext/src/autogen_ext/tools/semantic_kernel/__init__.py)).
- **記憶與快取**：`memory/__init__.py` 集成 Canvas、Redis、Chroma、Mem0 等記憶體；`cache_store/__init__.py` 提供磁碟與 Redis 快取實作 ([source](python/packages/autogen-ext/src/autogen_ext/memory/__init__.py), [source](python/packages/autogen-ext/src/autogen_ext/cache_store/__init__.py), [source](https://microsoft.github.io/autogen/dev/user-guide/extensions-user-guide/memory/index.html)).
- **Runtime 與範例**：`runtimes/grpc` 實作 Worker Runtime Host 對應 `agent_worker.proto`，可搭配 `python/samples/task_centric_memory` 等範例擴充記憶功能 ([source](python/packages/autogen-ext/src/autogen_ext/runtimes/grpc/__init__.py), [source](python/samples/task_centric_memory), [source](https://microsoft.github.io/autogen/dev/user-guide/extensions-user-guide/runtimes/grpc-runtime.html)).
- **代理與 UI**：`agents/web_surfer`, `agents/file_surfer`, `agents/magentic_one` 等提供專用代理；`autogen_ext/ui.py` 擴充 Console 輸出 ([source](python/packages/autogen-ext/src/autogen_ext/agents/web_surfer.py), [source](python/packages/autogen-ext/src/autogen_ext/agents/file_surfer.py), [source](python/packages/autogen-ext/src/autogen_ext/agents/magentic_one.py), [source](python/packages/autogen-ext/src/autogen_ext/ui.py), [source](https://microsoft.github.io/autogen/dev/user-guide/extensions-user-guide/index.html)).

- **AutoGen Studio（`python/packages/autogen-studio`）**
  - FastAPI 後端 + Gatsby 前端，CLI `autogenstudio` 進入點位於 `autogenstudio/cli.py` ([source](python/packages/autogen-studio/autogenstudio/cli.py), [source](https://microsoft.github.io/autogen/dev/user-guide/autogenstudio-user-guide/index.html)).
  - Studio 模式透過 SQLModel 管理資料庫，相關初始化流程在 `autogenstudio/database/db_manager.py` ([source](python/packages/autogen-studio/autogenstudio/database/db_manager.py)).
  - Lite 模式提供無資料庫的快速體驗，可由 CLI 或 `LiteStudio` 類別啟動 ([source](python/packages/autogen-studio/autogenstudio/lite/studio.py), [source](https://microsoft.github.io/autogen/dev/user-guide/autogenstudio-user-guide/usage.html)).
- 重要 CLI 參數：`--appdir`, `--database-uri`, `--reload`, `--upgrade-database`, `--session-name`, `--team`, `--auto-open`。
- 需安裝 Node/Gatsby 以建置前端，或使用 Dev Container。

### 3.5 Magentic-One（`autogen_ext.teams.magentic_one`）
- 基於 AgentChat Team 實作的 Orchestrator，整合 `MultimodalWebSurfer`, `FileSurfer`, `MagenticOneCoderAgent`, `CodeExecutorAgent`, `UserProxyAgent` ([source](python/packages/autogen-ext/src/autogen_ext/teams/magentic_one.py)).
- `MagenticOne` 類別提供簡化建構，支援 Human-in-the-loop（`hil_mode`）、程式執行核准、Docker 執行器、預設計畫／回顧邏輯。
- README 指出安全建議：容器隔離、日誌監控、限制資源、警惕 Prompt Injection ([source](python/packages/autogen-ext/src/autogen_ext/teams/magentic_one.py), [source](https://microsoft.github.io/autogen/dev/user-guide/agentchat-user-guide/magentic-one.html)).

---

## 4. 安裝與環境設定 (重要)

為了避免依賴衝突並確保一個乾淨的環境，強烈建議您在一個獨立的虛擬環境中安裝 AutoGen。

### 4.1 建立虛擬環境 (擇一即可)

**選項 A：使用 Conda (推薦)**
```bash
# 建立一個名為 autogen_env 的新環境，並指定 Python 版本
conda create -n autogen_env python=3.11

# 啟用該環境
conda activate autogen_env
```

**選項 B：使用 Python 內建的 venv**
```bash
# 建立一個名為 .venv 的虛擬環境目錄
python -m venv .venv

# 啟用該環境 (macOS/Linux)
source .venv/bin/activate

# 啟用該環境 (Windows)
# .\.venv\Scripts\activate
```

### 4.2 安裝 AutoGen 套件

> Python 套件的最新穩定版為 0.7.5，建議安裝時鎖定版本並一次解決所有相依，以避免組態錯配 ([source](https://github.com/microsoft/autogen/releases/tag/python-v0.7.5)、[source](README.md:42)、[source](python/packages/autogen-ext/pyproject.toml:21)).

- **完整功能（AgentChat + OpenAI 擴充 + MCP + Magentic-One）**

  ```bash
  pip install -U "autogen-agentchat==0.7.5" "autogen-ext[openai,mcp,magentic-one]==0.7.5" "PyYAML>=6.0"
  ```

- **加裝 Studio GUI（選用）**

  ```bash
  pip install -U "autogenstudio>=0.4.2.2"
  ```
  > Studio 套件獨立於核心版號維護，請以 PyPI 最新穩定版為準 ([source](https://pypi.org/project/autogenstudio/)).

- **僅需 AgentChat + OpenAI 功能**

  ```bash
  pip install -U "autogen-agentchat==0.7.5" "autogen-ext[openai]==0.7.5" "PyYAML>=6.0"
  ```

- **安裝完成後確認版本**

  ```bash
  pip show autogen-agentchat autogen-ext autogenstudio
  ```
  > `pip show` 可顯示已安裝套件版本與來源，便於核對是否已升級至指定版號 ([source](https://pip.pypa.io/en/stable/cli/pip_show/)).

### 4.3 設定模型 API 金鑰

AutoGen 需要知道如何存取 LLM。新版 AutoGen 推薦使用 YAML 檔案來進行組態設定。

**步驟 1: 建立 `model_config.yaml` 檔案**

在您的專案根目錄下，建立一個名為 `model_config.yaml` 的檔案。檔案內容定義了要使用的模型提供者和具體設定。

```yaml
- provider: autogen_ext.models.openai.OpenAIChatCompletionClient
  config:
    model: gpt-4o
    # api_key: "sk-..." # 由 OPENAI_API_KEY 環境變數提供

- provider: autogen_ext.models.anthropic.AnthropicChatCompletionClient
  config:
    model: claude-3-opus-20240229
    # api_key: "sk-..." # 由 ANTHROPIC_API_KEY 環境變數提供
```

**步驟 2: 使用環境變數 (推薦)**

為了安全起見，建議不要將 API 金鑰直接寫在程式碼或組態檔中。您可以在終端機中設定環境變數，AutoGen 會自動讀取。

```bash
export OPENAI_API_KEY="sk-..."
```

當 `model_config.yaml` 中的 `api_key` 欄位被省略或留空時，`OpenAIChatCompletionClient` 會自動查找 `OPENAI_API_KEY` 環境變數。

---

## 5. 快速入門示例

### 5.1 單代理 Hello World
> 需先設定 `OPENAI_API_KEY` 並在 `model_config.yaml` 定義至少一個 provider（參考 [README.md:51](README.md:51)）。模型客戶端可透過 `ChatCompletionClient.load_component` 動態載入組態 ([source](python/packages/autogen-core/src/autogen_core/_component_config.py:167))；執行流程與官方 Quickstart 一致 ([source](https://microsoft.github.io/autogen/stable/)).

```python
import asyncio
import yaml
from autogen_agentchat.agents import AssistantAgent
from autogen_core.models import ChatCompletionClient


async def main() -> None:
    with open("model_config.yaml", "r", encoding="utf-8") as f:
        model_config_list = yaml.safe_load(f)
    if not model_config_list:
        raise RuntimeError("model_config.yaml 為空，請至少定義一個模型提供者。")
    clients = []
    errors = []
    try:
        for entry in model_config_list:
            client = ChatCompletionClient.load_component(entry)
            clients.append(client)
            agent = AssistantAgent("assistant", model_client=client)
            try:
                task_result = await agent.run(task="Say 'Hello World!'")
                print(task_result)
                break
            except Exception as exc:
                model_name = entry.get("config", {}).get("model", entry)
                errors.append(f"{model_name}: {exc}")
        else:
            raise RuntimeError("所有模型皆無法完成任務：" + "; ".join(errors))
    finally:
        await asyncio.gather(*(client.close() for client in clients), return_exceptions=True)


asyncio.run(main())
```

> 上述程式會依序嘗試 `model_config.yaml` 中的每個 provider，並在第一個成功完成任務時停止；若多個模型都無法回應，將彙整錯誤訊息再拋出例外。

### 5.2 MCP 工具整合
> 依循官方 README 的 Playwright MCP 範例，需先安裝 `@playwright/mcp@latest`、匯出 `OPENAI_API_KEY`，並僅連線信任的 MCP 伺服器 ([source](README.md:40)、[source](README.md:60))。

```python
import asyncio
import yaml
from autogen_agentchat.agents import AssistantAgent
from autogen_agentchat.ui import Console
from autogen_core.models import ChatCompletionClient
from autogen_ext.tools.mcp import McpWorkbench, StdioServerParams


async def main() -> None:
    with open("model_config.yaml", "r", encoding="utf-8") as f:
        model_config_list = yaml.safe_load(f)
    model_client = ChatCompletionClient.load_component(model_config_list[0])

    server_params = StdioServerParams(
        command="npx",
        args=["@playwright/mcp@latest", "--headless"],
    )
    async with McpWorkbench(server_params) as mcp:
        agent = AssistantAgent(
            "web_browsing_assistant",
            model_client=model_client,
            workbench=mcp,
            model_client_stream=True,
            max_tool_iterations=10,
        )
        await Console(agent.run_stream(task="Find out how many contributors for the microsoft/autogen repository"))

    await model_client.close()


asyncio.run(main())
```

### 5.3 多代理協作
> 多代理工具化流程與官方 README 範例一致，需要將專家代理包成 `AgentTool` 並串流輸出；執行前請確認已匯出 `OPENAI_API_KEY` ([source](README.md:40)、[source](README.md:103))。

```python
import asyncio
import os
from autogen_agentchat.agents import AssistantAgent
from autogen_agentchat.tools import AgentTool
from autogen_agentchat.ui import Console
from autogen_ext.models.openai import OpenAIChatCompletionClient


async def main() -> None:
    api_key = os.getenv("OPENAI_API_KEY")
    if not api_key:
        raise RuntimeError("請先設定 OPENAI_API_KEY 環境變數。")

    model_client = OpenAIChatCompletionClient(model="gpt-4.1", api_key=api_key)

    math_agent = AssistantAgent(
        "math_expert",
        model_client=model_client,
        system_message="You are a math expert.",
        description="A math expert assistant.",
        model_client_stream=True,
    )
    math_agent_tool = AgentTool(math_agent, return_value_as_last_message=True)

    chemistry_agent = AssistantAgent(
        "chemistry_expert",
        model_client=model_client,
        system_message="You are a chemistry expert.",
        description="A chemistry expert assistant.",
        model_client_stream=True,
    )
    chemistry_agent_tool = AgentTool(chemistry_agent, return_value_as_last_message=True)

    agent = AssistantAgent(
        "assistant",
        system_message="You are a general assistant. Use expert tools when needed.",
        model_client=model_client,
        model_client_stream=True,
        tools=[math_agent_tool, chemistry_agent_tool],
        max_tool_iterations=10,
    )
    await Console(agent.run_stream(task="What is the integral of x^2?"))
    await Console(agent.run_stream(task="What is the molecular weight of water?"))

    await model_client.close()


asyncio.run(main())
```

### 5.4 Magentic-One（`autogen_ext/teams/magentic_one.py`）
> Magentic-One 官方文件提供完整使用範例，可直接串流團隊結果並印出回應；同樣需要事先設定 `OPENAI_API_KEY` ([source](README.md:40)、[source](python/packages/autogen-ext/src/autogen_ext/teams/magentic_one.py:81))。

```python
import asyncio
import os
import yaml
from autogen_ext.models.openai import OpenAIChatCompletionClient
from autogen_ext.teams.magentic_one import MagenticOne


async def main() -> None:
    api_key = os.getenv("OPENAI_API_KEY")
    if not api_key:
        with open("model_config.yaml", "r", encoding="utf-8") as f:
            for entry in yaml.safe_load(f):
                candidate = entry.get("config", {}).get("api_key")
                if candidate and not candidate.startswith("sk-XXX"):
                    api_key = candidate
                    break
    if not api_key:
        raise RuntimeError("請先設定 OPENAI_API_KEY 或在 model_config.yaml 中提供有效金鑰。")

    client = OpenAIChatCompletionClient(model="gpt-4o", api_key=api_key)
    team = MagenticOne(client=client, hil_mode=False)
    try:
        try:
            result = await asyncio.wait_for(team.run(task="Plan a 3-day trip to Seattle"), timeout=300)
            if result.final_response:
                print(result.final_response)
        except asyncio.TimeoutError:
            print("Magentic-One 任務在 300 秒內尚未完成，請縮短任務或改為 hil_mode=True 監督執行。")
    finally:
        await client.close()


asyncio.run(main())
```

### 5.5 Task-Centric Memory
`python/samples/task_centric_memory` 提供完整範例，可將記憶整合進 `RoutedAgent` 或 AgentChat Team。

---

## 6. Agent 與訊息生命週期

1. **Topic 發佈**：訊息以 CloudEvents 表示（[source](docs/design/01%20-%20Programming%20Model.md)、[source](docs/design/02%20-%20Topics.md)）。`TopicId = (type, source)`，Agent 訂閱特定 Topic 或 Prefix。
2. **Runtime 路由**：`AgentRuntime` 依訂閱查詢對應 Agent，如 Agent 尚未實例化會動態建立 ([source](python/packages/autogen-core/src/autogen_core/_agent_runtime.py))。
3. **MessageContext**：`MessageContext(sender, topic_id, is_rpc, cancellation_token, message_id)`，供 handler 判斷來源與是否為 RPC ([source](python/packages/autogen-core/src/autogen_core/_message_context.py))。
4. **Hand-off 與工具**：`AssistantAgent` 可偵測工具呼叫；透過 `Handoff` 產生 `FunctionTool` 將控制權交給指定 Agent ([source](python/packages/autogen-agentchat/src/autogen_agentchat/messages.py))。
5. **狀態管理**：Agent 與 Team 必須實作 `save_state` / `load_state`；常用 state 類別存放於 `autogen_agentchat/state/_states.py` ([source](python/packages/autogen-agentchat/src/autogen_agentchat/state/_states.py))。
6. **事件記錄**：模型呼叫、工具執行、串流開始／結束會記錄 `LLMCallEvent` 等 JSON 日誌 ([source](python/packages/autogen-core/src/autogen_core/logging.py))。
7. **取消**：`CancellationToken` 可用於停止長時間任務、取消用户輸入等待或工具執行 ([source](python/packages/autogen-core/src/autogen_core/_cancellation_token.py))。

---

## 7. AgentChat 元件總表

### 7.1 Agents
- `AssistantAgent` ([source](python/packages/autogen-agentchat/src/autogen_agentchat/agents/_assistant_agent.py)): 支援串流、Structured Output、工具並行、回顧模式、記憶查詢事件。
- `UserProxyAgent` ([source](python/packages/autogen-agentchat/src/autogen_agentchat/agents/_user_proxy_agent.py)): 人類代理，支援同步／非同步輸入函式與取消。
- `CodeExecutorAgent` ([source](python/packages/autogen-agentchat/src/autogen_agentchat/agents/_code_executor_agent.py)): 程式碼生成 + 執行，支援程式碼審批與多語言模式。
- `MessageFilterAgent` ([source](python/packages/autogen-agentchat/src/autogen_agentchat/agents/_message_filter_agent.py)), `SocietyOfMindAgent` ([source](python/packages/autogen-agentchat/src/autogen_agentchat/agents/_society_of_mind_agent.py)), `ClosureAgent` 等。

### 7.2 Messages / Events
- Chat 訊息：`TextMessage`, `StructuredMessage`, `ToolCallSummaryMessage`, `HandoffMessage` ([source](python/packages/autogen-agentchat/src/autogen_agentchat/messages.py))。
- 事件：`ThoughtEvent`, `ToolCallRequestEvent`, `ToolCallExecutionEvent`, `ModelClientStreamingChunkEvent`, `MemoryQueryEvent`, `UserInputRequestedEvent` ([source](python/packages/autogen-agentchat/src/autogen_agentchat/events.py))。
- 停止訊息：`StopMessage` ([source](python/packages/autogen-agentchat/src/autogen_agentchat/messages.py))。

### 7.3 Teams
- Group Chats：`RoundRobinGroupChat`, `SelectorGroupChat`, `SwarmGroupChat`, `GraphGroupChat`, `MagenticOneGroupChat` ([source](python/packages/autogen-agentchat/src/autogen_agentchat/teams/_group_chat))。
- `ChatAgentContainer` 用於包裝 Agent + 狀態；`BaseGroupChatManager` 在 `state` 中保存輪次、訊息串 ([source](python/packages/autogen-agentchat/src/autogen_agentchat/teams/_group_chat/base_group_chat_manager.py))。

### 7.4 TerminationConditions
- `StopMessageTermination`, `MaxMessageTermination`, `TextMentionTermination`, `FunctionalTermination`, `ToolCallTermination`, `HandoffTermination`, `SourceMatchTermination` 等，用於控制 Team 停止條件 ([source](python/packages/autogen-agentchat/src/autogen_agentchat/conditions/_terminations.py))。

### 7.5 工具與組態
- `AgentTool` / `TeamTool` 將 Agent/Team 暴露為工具；`TaskRunnerTool` 可從 TaskRunner 產出工具 ([source](python/packages/autogen-agentchat/src/autogen_agentchat/tools/__init__.py))。
- `Component` 介面允許將 Agent / Model / Tool 序列化為 `ComponentModel`，支援 `_to_config` / `_from_config` 與 `ComponentLoader.load_component` ([source](python/packages/autogen-core/src/autogen_core/_component_config.py)).
- `component-schema-gen` CLI 可輸出既有組件的 JSON Schema ([source](python/packages/component-schema-gen/src/component_schema_gen/__main__.py)).

---

## 8. 擴充與整合

### 8.1 模型客戶端（`autogen_ext.models`）
- **OpenAI / Azure**：支援串流、工具、structured response、r1 content 解析與 stop reason 正規化 ([source](python/packages/autogen-ext/src/autogen_ext/models/openai/__init__.py))。
- **Anthropic / Bedrock**：整合 `AsyncAnthropic`、工具選擇、影像 MIME 判斷、Thinking 模式 ([source](python/packages/autogen-ext/src/autogen_ext/models/anthropic/__init__.py))。
- **Ollama / Llama.cpp**：本地模型支援 ([source](python/packages/autogen-ext/src/autogen_ext/models/ollama/__init__.py), [source](python/packages/autogen-ext/src/autogen_ext/models/llama_cpp/__init__.py))。
- **Semantic Kernel**：作為 ChatCompletion Adapter ([source](python/packages/autogen-ext/src/autogen_ext/models/semantic_kernel/__init__.py))。
- **Cache / Replay**：可包裝其他模型以實作快取或重播 ([source](python/packages/autogen-ext/src/autogen_ext/models/cache/__init__.py), [source](python/packages/autogen-ext/src/autogen_ext/models/replay/__init__.py))。

### 8.2 程式碼執行
- `DockerCommandLineCodeExecutor`, `LocalCommandLineCodeExecutor`, `JupyterKernelCodeExecutor`, `DockerJupyterCodeExecutor`, `AzureContainerApps` 等皆由模組匯出 ([source](python/packages/autogen-ext/src/autogen_ext/code_executors/__init__.py))。
- `CodeExecutionInput` / `CodeExecutionResult` 描述輸入輸出，支援多語言。
- 安全建議：官方透明度 FAQ 建議以容器隔離並加入人工審核（`hil_mode`／`approval_func`）降低風險 ([source](TRANSPARENCY_FAQS.md:54)).

### 8.3 MCP 與工具
- `McpWorkbench` 提供工具清單與呼叫介面；支援 STDIO / SSE / HTTP Workbench；可結合 `AssistantAgent(workbench=...)` ([source](python/packages/autogen-ext/src/autogen_ext/tools/mcp/_workbench.py))。
- `mcp_server_tools` 可直接從 MCP Server 產生工具列表 ([source](python/packages/autogen-ext/src/autogen_ext/tools/mcp/_factory.py))。
- `ChatCompletionClientSampler`, `Sampler`、`RootsProvider` 等能力由 MCP host 模組提供，可應對 Sampling / Roots / Elicitation 請求 ([source](python/packages/autogen-ext/src/autogen_ext/tools/mcp/_host.py))。

### 8.4 記憶與快取
- `autogen_ext.memory`：Canvas API、Redis、Chroma、Mem0；皆符合 `autogen_core.memory.Memory` 介面 ([source](python/packages/autogen-ext/src/autogen_ext/memory/__init__.py))。
- `task_centric_memory`：提供 `MemoryController`、`MemoryBank`、`MemoryPrompter` 等模組，可快速將記憶整合到 Agent ([source](python/samples/task_centric_memory)).
- `cache_store`：`DiskCacheStore`, `RedisStore`，可供模型或其他組件快取結果 ([source](python/packages/autogen-ext/src/autogen_ext/cache_store/__init__.py)).

### 8.5 gRPC Runtime 與 Proto
- `autogen_ext.runtimes.grpc` 實作 Worker Runtime Host，對應 `AgentRpc.OpenChannel` 雙向串流、`RegisterAgentType`, `AddSubscription` 等 RPC ([source](python/packages/autogen-ext/src/autogen_ext/runtimes/grpc))。
- `protos/agent_worker.proto` 定義 `RpcRequest`, `RpcResponse`, `ControlMessage`、`Subscription`；`cloudevent.proto` 引入 CloudEvents 標準欄位 ([source](protos))。
- 用例：在分散式環境中將 Agent Worker 部署於不同節點，由 Gateway 進行 Placement 與 Routing（[source](docs/design/03%20-%20Agent%20Worker%20Protocol.md), [source](docs/design/05%20-%20Services.md)）。

---

## 9. AutoGen Studio 操作重點
- **安裝**：`pip install autogenstudio` 或在倉庫 `pip install -e .` ([source](python/packages/autogen-studio/pyproject.toml))。
- **啟動**：
  ```bash
  autogenstudio ui --port 8080 --appdir ~/.autogenstudio --database-uri sqlite:///database.sqlite
  ```
  ([source](python/packages/autogen-studio/autogenstudio/cli.py))
- **Lite 模式**：
  ```bash
  autogenstudio lite --team ./team.json --session-name "My Experiment" --auto-open
  ```
  或程式化使用 `from autogenstudio.lite import LiteStudio` ([source](python/packages/autogen-studio/autogenstudio/lite/studio.py))。
- **資料庫**：依 README 指示，SQLModel + Alembic；可使用 `--upgrade-database` 升級 schema ([source](python/packages/autogen-studio/autogenstudio/database/db_manager.py))。
- **前端建置**：需 Node/Yarn，於 `frontend` 執行 `yarn build`；若未安裝 git lfs 需補建 ([source](python/packages/autogen-studio/README.md))。
- **更新紀錄**：2024-11-14 已開始改寫以支援 AgentChat 0.4 API。

---

## 10. Magentic-One 作業流程
- Orchestrator 制定 Task/Progress Ledger，指派 Web/File/Coder/Executor 子代理 ([source](python/packages/autogen-ext/src/autogen_ext/teams/magentic_one.py))。
- `hil_mode=True` 時加入 `UserProxyAgent`，允許人類確認步驟；`approval_func` 可僅針對程式執行要求核准。
- 依 README 建議 ([source](python/packages/autogen-ext/src/autogen_ext/teams/magentic_one.py))：
  1. 使用 Docker 隔離。
  2. 監控日誌、限制外部資源。
  3. 警惕 prompt injection 與自動化行為（例如接受 Cookie、嘗試招募人類協助）。

---

## 11. 範例與模板

### 11.1 Python Samples（[source](python/samples)）
- AgentChat：
  - `agentchat_fastapi` ([source](python/samples/agentchat_fastapi)), `agentchat_chainlit` ([source](python/samples/agentchat_chainlit)), `agentchat_streamlit` ([source](python/samples/agentchat_streamlit))：整合 Web UI。
  - `agentchat_graphrag` ([source](python/samples/agentchat_graphrag)), `agentchat_dspy` ([source](python/samples/agentchat_dspy)), `agentchat_azure_postgresql` ([source](python/samples/agentchat_azure_postgresql)), `agentchat_chess_game` ([source](python/samples/agentchat_chess_game))。
- Core Runtime：
  - `core_streaming_response_fastapi`, `core_semantic_router`, `core_grpc_worker_runtime`, `core_distributed-group-chat`, `core_xlang_hello_python_agent`.
- Magentic-One：
  - `magentic-one-cli`（於 `python/packages/magentic-one-cli` 提供可安裝的 CLI 範例）([source](python/packages/magentic-one-cli)).
- Task-Centric Memory：
  - `samples/task_centric_memory`：展示如何在 Team 外掛記憶 ([source](python/samples/task_centric_memory))。
- 其他：`gitty`（CLI 草稿回覆生成），`core_async_human_in_the_loop` 等。

### 11.2 .NET Samples（[source](dotnet/samples)）
- `Hello`: 最小範例（`AutoGen.OpenAI`）([source](dotnet/samples/Hello))。
- `AgentChat`: 多 Agent 聊天、群組聊天 ([source](dotnet/samples/AgentChat))。
- `GettingStarted` / `GettingStartedGrpc`: 新事件驅動 API、gRPC Worker ([source](dotnet/samples/GettingStarted), [source](dotnet/samples/GettingStartedGrpc))。
- `dev-team`: 包含多檔案與 DevOps 相關示例 ([source](dotnet/samples/dev-team))。

### 11.3 模板
- `python/templates/new-package`：Cookiecutter 模板，可快速建立新的 AutoGen 套件 ([source](python/templates/new-package))。

---

## 12. 測試與品質保證
- **Python**：
  - 執行 `poe test`, `poe mypy`, `poe pyright`，確保靜態型別與測試通過 ([source](pyproject.toml), [source](shared_tasks.toml))。
  - `poe docs-check` 驗證文件，`poe docs-check-examples` 檢查 API 參考範例程式碼區塊。
  - `poe samples-code-check` 保證 samples 可執行。
- **.NET**：
  - 在 `dotnet/` 使用 `dotnet test` ([source](dotnet/AutoGen.sln))。
  - `AutoGen.SourceGenerator` 需搭配 Roslyn Source Generator 測試 ([source](dotnet/test/AutoGen.SourceGenerator.Tests))。
- **CI**：倉庫包含 GitHub Actions（例如 `dotnet-build.yml`），可參考以建立自動驗證流程 ([source](.github/workflows))。

---

## 13. 日誌、遙測與除錯
- 事件記錄呼叫 `logging.getLogger(EVENT_LOGGER_NAME)`（[source](python/packages/autogen-agentchat/src/autogen_agentchat/__init__.py), [source](python/packages/autogen-core/src/autogen_core/__init__.py)）。
- `LLMCallEvent` / `LLMStreamStartEvent` / `LLMStreamEndEvent` / `ToolCallEvent` 提供 JSON 字串，可直接寫入記錄系統 ([source](python/packages/autogen-core/src/autogen_core/logging.py))。
- `MessageHandlerContext.agent_id()` 讓工具／模型呼叫取得目前 Agent 標識，便於追蹤 ([source](python/packages/autogen-core/src/autogen_core/_message_handler_context.py))。
- 遙測：`autogen_core._telemetry` 使用 OpenTelemetry Tracer Provider；可設 `AUTOGEN_DISABLE_RUNTIME_TRACING=true` 停用 ([source](python/packages/autogen-core/src/autogen_core/_telemetry/__init__.py), [source](python/packages/autogen-core/src/autogen_core/_telemetry/_tracing.py))。

---

## 14. 分散式架構與服務
- `docs/design/03 - Agent Worker Protocol.md` 描述 Worker 啟動、註冊 AgentType、接收 `Event` / `RpcRequest`、回覆 `RpcResponse` ([source](docs/design/03%20-%20Agent%20Worker%20Protocol.md))。
- `docs/design/05 - Services.md`：服務組件包括 Gateway（RPC + Event Bus bridge）、Registry、AgentState、Routing；Roadmap 包含排程與探索 ([source](docs/design/05%20-%20Services.md))。
- `protos/agent_worker.proto` 定義 `AgentRpc` 服務與 Control Channel（存取狀態 Save/Load）([source](protos/agent_worker.proto))。
- 文檔規劃後續提供排程與探索等服務能力，以改進跨 Worker 的 Agent Placement 與管理 ([source](docs/design/05%20-%20Services.md:23-25))。

---

## 15. .NET API 重點
- 舊版套件 `AutoGen.*` 保留 0.2 API；新版事件驅動 API 在 `Microsoft.AutoGen.*` ([source](dotnet/src))。
- 基礎程式碼：
  ```csharp
  var openAIKey = Environment.GetEnvironmentVariable("OPENAI_API_KEY");
  var config = new OpenAIConfig(openAIKey, "gpt-3.5-turbo");
  var assistant = new AssistantAgent(name: "assistant", systemMessage: "...", llmConfig: new ConversableAgentConfig { ConfigList = [config] }).RegisterPrintMessage();
  var user = new UserProxyAgent(name: "user", humanInputMode: HumanInputMode.ALWAYS).RegisterPrintMessage();
  await user.InitiateChatAsync(assistant, "Hey assistant...", maxRound: 10);
  ```
- 功能：
  - ConversableAgent 支援函式呼叫、.NET Code Execution（透過 `dotnet-interactive`）([source](dotnet/src/AutoGen.DotnetInteractive))。
  - 群組聊天、工具支援。
  - `AutoGen.SourceGenerator` 生成型別安全工具定義 ([source](dotnet/src/AutoGen.SourceGenerator))。
  - `AutoGen.DotnetInteractive` 可在 C#/F#/PowerShell/Python 執行程式碼區塊 ([source](dotnet/src/AutoGen.DotnetInteractive))。
  - `AutoGen.WebAPI`／`AutoGen.AzureAIInference` 等擴充 ([source](dotnet/src/AutoGen.WebAPI), [source](dotnet/src/AutoGen.AzureAIInference))。

---

## 16. 常見問題與支援
- `FAQ.md`：說明 0.4 為重寫版本，仍維護 0.2；鼓勵早期採用者嘗試並提供回饋；官方支持渠道為 GitHub Issues/Discussions 與新 Discord（`aka.ms/autogen-discord`）([source](FAQ.md))。
- `TRANSPARENCY_FAQS.md`：闡述 AutoGen 的用途限制、負責任 AI 評估、可能風險（資料偏差、不透明、誤用、隱私、安全等），建議配合內容審查與人類監督 ([source](TRANSPARENCY_FAQS.md))。
- `SUPPORT.md`, `SECURITY.md`, `CONTRIBUTING.md`：提供貢獻、回報安全問題、取得協助的程序 ([source](SUPPORT.md), [source](SECURITY.md), [source](CONTRIBUTING.md))。

---

## 17. 最佳實務與安全建議
- **模型金鑰管理**：使用環境變數或秘密管理工具，不將金鑰硬編碼於程式或設定檔（README Quickstart 示範以 `export OPENAI_API_KEY` 設定金鑰）([source](README.md:40), [source](https://microsoft.github.io/autogen/dev/user-guide/core-user-guide/installation.html#api-keys-from-environment-variables)).
- **程式執行**：優先在 Docker 等隔離環境執行程式碼，並透過 `approval_func` / `hil_mode` 保持人工審查 ([source](TRANSPARENCY_FAQS.md:54), [source](python/packages/autogen-ext/src/autogen_ext/code_executors/__init__.py:45), [source](https://microsoft.github.io/autogen/dev/user-guide/extensions-user-guide/code-executors/index.html#security-best-practices)).
- **MCP／工具**：僅連線可信 MCP 伺服器，避免外部工具執行惡意操作 ([source](README.md:96), [source](https://microsoft.github.io/autogen/dev/user-guide/extensions-user-guide/mcp/index.html#security-considerations)).
- **責任使用**：遵守模型供應商使用政策並導入內容審查與人類監督機制 ([source](TRANSPARENCY_FAQS.md:57), [source](https://microsoft.github.io/autogen/dev/user-guide/core-user-guide/best-practices.html#responsible-ai)).

---

## 18. 進一步閱讀與資源
- 官方文件（最新版）：https://microsoft.github.io/autogen/
- 設計文檔：`docs/design/*.md`
- Magentic-One 技術報告：https://arxiv.org/abs/2411.04468
- AutoGen 部落格與 Discord：README 徽章提供連結 ([source](README.md))。

---

## 19. 推薦下一步
- 依需求選擇 API 層級：快速原型使用 `autogen-agentchat`，複雜互動／分散式場景深入 `autogen-core`。
- 使用 `python/samples` 作為起點建立自訂代理或工作流。
- 若採用視覺化流程，安裝並試用 AutoGen Studio（Lite 或完整版）。
- 打算整合至企業環境時，評估 gRPC Worker Runtime 與服務拓撲，並導入適當的日誌／遙測管線。
- 持續關注 FAQ 與版本公告，必要時更新遷移程式碼。

---

> 本手冊內容根據倉庫最新程式碼與文件歸納，實作時請搭配對應 `.py`/`.cs` 檔案（例如 `python/packages/autogen-agentchat/src/autogen_agentchat/agents/_assistant_agent.py:1`、`python/packages/autogen-core/src/autogen_core/_single_threaded_agent_runtime.py:1`、`dotnet/src/Microsoft.AutoGen` 等）查閱細節。

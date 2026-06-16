# 分散式 Group Chat 範例詳細說明

## 範例概述
- 範例透過 `GrpcWorkerAgentRuntimeHost` 與多個 `GrpcWorkerAgentRuntime` 展現 AutoGen Core 的跨程序分散式聊天流程，包含寫作、編輯、群聊管理與 UI 代理協作，並以 Azure OpenAI 模型提供 LLM 能力。[README.md:1](README.md#L1)

## 系統架構
- Host：`run_host.py` 啟動 gRPC Host，等待分散式 runtime 連線並維持訊息總線。[run_host.py#L10-L18](run_host.py#L10)
- Writer／Editor Runtime：`run_writer_agent.py` 與 `run_editor_agent.py` 建立各自的 `GrpcWorkerAgentRuntime`，載入 `AppConfig` 後註冊 `BaseGroupChatAgent`，訂閱寫作者或編輯 topic 與群聊 topic，並在接收 `RequestToSpeak` 時透過 LLM 產生內容並串流至 UI。[run_writer_agent.py#L19-L45](run_writer_agent.py#L19)｜[run_editor_agent.py#L19-L46](run_editor_agent.py#L19)
- Group Chat Manager：`run_group_chat_manager.py` 建立管理代理，於啟動後模擬使用者訊息、要求下一位代理發言並串流重要訊息到 UI topic。[run_group_chat_manager.py#L19-L70](run_group_chat_manager.py#L19)
- UI Runtime：`run_ui.py` 註冊 `UIAgent`，使用 Chainlit 把 `MessageChunk` 逐 token 呈現在瀏覽器，觀察多代理互動流程。[run_ui.py#L23-L60](run_ui.py#L23)
- 代理行為：`_agents.py` 定義 `BaseGroupChatAgent` 如何維護聊天歷史與串流輸出、`GroupChatManager` 如何輪詢角色、`UIAgent` 接收訊息後更新介面。[ _agents.py#L20-L208](_agents.py#L20)

## 組態與工具
- `_types.py` 定義 YAML 設定結構（Host、Writer/Editor、UI 等）與 `MessageChunk`、`RequestToSpeak` 型別，`load_config` 會解析 `config.yaml` 並處理 Azure AD Token Provider。[ _types.py#L34-L78](_types.py#L34)｜[_utils.py#L12-L35](_utils.py#L12)
- `run.sh` 使用 tmux 同時啟動 host、UI、writer、editor 與 manager 五個程序，方便一次啟動完整示例。[README.md:28](README.md#L28)

## 執行步驟
- 依 README 指引建立虛擬環境、安裝 `autogen-ext[openai,azure,chainlit,rich]` 等依賴，並於 `config.yaml` 配置 Azure OpenAI 認證資訊。[README.md:12](README.md#L12)｜[README.md:18](README.md#L18)
- 透過 `./run.sh` 一鍵啟動所有程序，或依 README 所述分別在多個終端執行 `run_host.py`、`chainlit run run_ui.py --port 8001`、`run_editor_agent.py`、`run_writer_agent.py`、`run_group_chat_manager.py` 觀看協作過程。[README.md:28](README.md#L28)｜[README.md:41](README.md#L41)

## 延伸參考
- 建議閱讀 AutoGen Core 服務與 Worker Runtime 設計文件，理解 gRPC runtime 與 Topic 路由原理，並可搭配官方 Core 指南進一步擴充代理能力。[docs/design/05 - Services.md](../../docs/design/05%20-%20Services.md)｜[AutoGen Core User Guide](https://microsoft.github.io/autogen/stable/user-guide/core-user-guide/index.html)

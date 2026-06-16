# AutoGen AgentChat 套件說明

## 套件定位
- AgentChat 在 Core 之上提供高階多代理開發介面，內建可宣告式配置的代理、團隊與條件，適合作為應用程式組合的預設入口。[python/packages/autogen-agentchat/README.md](python/packages/autogen-agentchat/README.md)
- 官方使用指南涵蓋多代理模式、隊伍範例與遷移指引，可搭配程式碼快速實作。[https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/index.html](https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/index.html)

## 代理與訊息模型
- `AssistantAgent` 支援工具呼叫、手動交棒 (handoff)、流式回應與結構化輸出，可透過 Pydantic Config 靈活注入模型客戶端、工具與記憶體。[python/packages/autogen-agentchat/src/autogen_agentchat/agents/_assistant_agent.py#L70](python/packages/autogen-agentchat/src/autogen_agentchat/agents/_assistant_agent.py#L70)
- 訊息層由 `messages.py` 定義，區分 `BaseChatMessage`、`BaseAgentEvent` 與 `StructuredMessage` 等型別，並記錄模型耗用量與建立時間，利於條件判斷與審計。[python/packages/autogen-agentchat/src/autogen_agentchat/messages.py#L26](python/packages/autogen-agentchat/src/autogen_agentchat/messages.py#L26)
- 事件包含 `ToolCallRequestEvent`、`ModelClientStreamingChunkEvent` 等特殊節點，可在串流介面或 AutoGen Studio 中呈現代理的思考過程。[python/packages/autogen-agentchat/src/autogen_agentchat/agents/_assistant_agent.py#L45](python/packages/autogen-agentchat/src/autogen_agentchat/agents/_assistant_agent.py#L45)

## 團隊編排
- `RoundRobinGroupChat` 與其 Manager 透過循環輪詢選擇發言者、維護訊息快照並支援狀態儲存/載入，是建立互動式小組的基礎元件。[python/packages/autogen-agentchat/src/autogen_agentchat/teams/_group_chat/_round_robin_group_chat.py#L16](python/packages/autogen-agentchat/src/autogen_agentchat/teams/_group_chat/_round_robin_group_chat.py#L16)
- `teams` 命名空間另提供 Selector、Sequential、Magentic-One 等管理器，配合 `TerminationCondition` 可快速組合實際工作流。[python/packages/autogen-agentchat/src/autogen_agentchat/teams](python/packages/autogen-agentchat/src/autogen_agentchat/teams)
- 官方 Reference 文件列出各團隊類別與初始化參數，適合在撰寫 YAML/JSON 配置時查閱。[https://microsoft.github.io/autogen/stable/reference/python/autogen_agentchat.teams.html](https://microsoft.github.io/autogen/stable/reference/python/autogen_agentchat.teams.html)

## 使用者互動
- `ui/_console.py` 實作串流渲染器，可辨識 iTerm inline 圖片、累計 token 使用量並透過 `UserInputManager` 交織人類回饋，便於在 CLI 中觀察團隊行為。[python/packages/autogen-agentchat/src/autogen_agentchat/ui/_console.py#L1](python/packages/autogen-agentchat/src/autogen_agentchat/ui/_console.py#L1)
- `UserProxyAgent` 與 `UserInputRequestedEvent` 搭配，可在多代理對話中引入人工確認或資料輸入節點，常用於人機協作場景。[python/packages/autogen-agentchat/src/autogen_agentchat/ui/_console.py#L24](python/packages/autogen-agentchat/src/autogen_agentchat/ui/_console.py#L24)

## 開發建議
- 建立自訂代理時，可繼承 `BaseChatAgent` 並沿用訊息類型，避免破壞 AgentChat 與 AutoGen Studio 的序列化契約。[python/packages/autogen-agentchat/src/autogen_agentchat/agents/_assistant_agent.py#L90](python/packages/autogen-agentchat/src/autogen_agentchat/agents/_assistant_agent.py#L90)
- 若需導出配置檔供 YAML/JSON 使用，務必使用 `ComponentModel` 格式，以利 AutoGen Studio 或 CLI 工具直接載入。[python/packages/autogen-agentchat/src/autogen_agentchat/agents/_assistant_agent.py#L71](python/packages/autogen-agentchat/src/autogen_agentchat/agents/_assistant_agent.py#L71)

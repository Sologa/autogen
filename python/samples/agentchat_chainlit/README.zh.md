# Chainlit 與 AgentChat 多代理範例詳細說明

## 範例概述
- 範例透過 Chainlit 建立使用者介面，示範與單一 `AssistantAgent`、具評論員的 `RoundRobinGroupChat` 團隊，以及加入 `UserProxyAgent` 的審核流程互動，並支援模型串流輸出。[README.md:1](README.md#L1)

## 主要檔案與邏輯
- `app_agent.py`：啟動時讀取 `model_config.yaml` 建立 `ChatCompletionClient`，將 `get_weather` 工具注入 `AssistantAgent`，並在 `@cl.on_message` 事件中以 `on_messages_stream` 串流模型輸出至使用者端。[app_agent.py#L31-L68](app_agent.py#L31)  
- `app_team.py`：建立「助手／評論家」兩名代理，設定 `TextMentionTermination("APPROVE")` 決定對話終止，並在 `RoundRobinGroupChat.run_stream` 期間聚合串流訊息與最終任務結果。[app_team.py#L21-L97](app_team.py#L21)  
- `app_team_user_proxy.py`：額外註冊 `UserProxyAgent`，實作 `AskActionMessage` 讓使用者以按鈕核准或拒絕；透過 `TextMentionTermination("APPROVE", sources=["user"])` 控制當使用者核准時結束協作。[app_team_user_proxy.py#L13-L138](app_team_user_proxy.py#L13)

## 安裝與設定
- 需安裝 Chainlit、`autogen-agentchat`、`autogen-ext[openai]` 與 `pyyaml` 等套件，並依模型供應商需要切換 `autogen-ext` 額外選項。[README.md:10](README.md#L10)
- 建議從 `model_config_template.yaml` 複製與調整模型設定，另存為 `model_config.yaml` 提供所有腳本載入。[README.md:23](README.md#L23)

## 執行流程
- 單代理模式：在範例目錄執行 `chainlit run app_agent.py`，可先使用內建 Starter 詢問天氣並觀察工具反思流程。[README.md:28](README.md#L28)
- 團隊模式：執行 `chainlit run app_team.py` 以 Round-Robin 方式讓助手與評論家輪流回覆，直至評論家輸出 `APPROVE`。[README.md:42](README.md#L42)
- 含使用者審核的團隊：執行 `chainlit run app_team_user_proxy.py`，由 `UserProxyAgent` 要求使用者核准或拒絕團隊輸出，再由按鈕回饋結果。[README.md:56](README.md#L56)

## 延伸參考
- AutoGen AgentChat 官方指南提供更多代理、團隊與工具使用模式，建議在擴充範例前詳讀相關章節。[AutoGen AgentChat User Guide](https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/index.html)

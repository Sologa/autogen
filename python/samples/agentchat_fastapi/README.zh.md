# FastAPI 與 AgentChat 聊天範例詳細說明

## 範例概述
- 範例透過 FastAPI 提供 REST 與 WebSocket 服務，分別示範單一 `AssistantAgent` 會話與含 `UserProxyAgent` 的多代理團隊輪詢，並示範狀態／歷史持久化及瀏覽器 UI 整合。[README.md:1](README.md#L1)

## 主要檔案與邏輯
- `app_agent.py`：  
  - 以 `aiofiles` 讀取 `model_config.yaml` 建立模型客戶端，初始化 `AssistantAgent` 後於 `/chat` 路由呼叫 `on_messages` 回應使用者訊息。[app_agent.py#L40-L83](app_agent.py#L40)  
  - 每次交談後將 `save_state()` 結果寫入 `agent_state.json`，並同步維護 `agent_history.json` 供前端顯示歷史訊息。[app_agent.py#L85-L94](app_agent.py#L85)
- `app_team.py`：  
  - 建立「assistant」「yoda」「user」三名代理，將 `UserProxyAgent` 的輸入函式綁定到 WebSocket 收到的使用者訊息。[app_team.py#L45-L119](app_team.py#L45)  
  - 使用 `RoundRobinGroupChat.run_stream` 串流訊息至前端，並於隊列中過濾 `UserInputRequestedEvent` 以避免將操作提示寫回歷史檔。[app_team.py#L95-L155](app_team.py#L95)
- `app_agent.html`／`app_team.html` 提供基本網頁介面，透過 JavaScript 呼叫上述 API 或 WebSocket 端點。

## 安裝與設定
- 安裝 FastAPI、Uvicorn、`autogen-agentchat`、`autogen-ext[openai]` 與 `PyYAML` 等依賴，並根據目標模型供應商調整 `autogen-ext` 選項。[README.md:17](README.md#L17)
- 以 `model_config_template.yaml` 為範本建立 `model_config.yaml`，填入模型名稱與認證設定供伺服端載入。[README.md:25](README.md#L25)

## 執行流程
- 單代理服務：於目錄執行 `python app_agent.py` 啟動 API，瀏覽 `http://localhost:8001` 體驗對話並查看歷史記錄檔案。[README.md:30](README.md#L30)
- 多代理輪詢服務：執行 `python app_team.py` 於 `http://localhost:8002` 觀察助手、Yoda 風格代理與 `UserProxyAgent` 交替回覆及核准流程。[README.md:40](README.md#L40)
- 狀態檔案（`agent_state.json` / `team_state.json`）支援伺服器重啟後恢復對話；歷史檔（`agent_history.json` / `team_history.json`）提供前端顯示紀錄。[app_agent.py#L85-L95](app_agent.py#L85)｜[app_team.py#L120-L136](app_team.py#L120)

## 延伸參考
- 更多 AgentChat 團隊模式與模型配置可參考官方指南，以及 FastAPI 串流與 WebSocket 範例章節進一步擴充專案。[AutoGen AgentChat User Guide](https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/index.html)

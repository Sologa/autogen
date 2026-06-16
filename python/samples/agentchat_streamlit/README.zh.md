# Streamlit 與 AgentChat 聊天範例詳細說明

## 範例概述
- 利用 Streamlit 快速建立網頁聊天介面，展示如何在單頁應用中維持對話歷史並呼叫 AutoGen `AssistantAgent` 取得回覆。[README.md:1](README.md#L1)

## 主要檔案與邏輯
- `agent.py`：讀取 `model_config.yml` 建立 `ChatCompletionClient`，將其注入 `AssistantAgent`，並提供非同步 `chat` 方法呼叫 `on_messages`，回傳純文字內容供 UI 顯示。[agent.py#L8-L26](agent.py#L8)
- `main.py`：  
  - 初始化 Streamlit 頁面與 session state，首次載入時建立 `Agent` 實例並儲存於 `st.session_state["agent"]`。[main.py#L7-L18](main.py#L7)  
  - 使用 `st.chat_input` 取得訊息後，透過 `asyncio.run` 呼叫代理回覆並將消息追加至 `messages`，使 UI 每次重繪時回放完整對話。[main.py#L25-L34](main.py#L25)

## 安裝與設定
- 安裝 `streamlit`、`autogen-agentchat`、`autogen-ext[openai,azure]`（或相容選項）以及必要依賴，再新增 `model_config.yml` 指定模型與金鑰資訊。[README.md:7](README.md#L7)｜[README.md:21](README.md#L21)

## 執行步驟
- 於目錄執行 `streamlit run main.py` 啟動應用，瀏覽器開啟後即可輸入訊息並觀察代理回應。[README.md:41](README.md#L41)
- 對話記錄會留在 `st.session_state["messages"]` 中，重新整理頁面或持續互動時即會自動還原歷史訊息。[main.py#L17-L34](main.py#L17)

## 延伸參考
- 若需擴充更多代理或工具，可參考 AutoGen AgentChat 官方文件了解自訂工具、團隊與串流機制定義方式，再將其整合到 Streamlit UI 中。[AutoGen AgentChat User Guide](https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/index.html)

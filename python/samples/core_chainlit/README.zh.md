# Chainlit 與 AutoGen Core 整合範例詳細說明

## 範例概述
- 透過 Chainlit UI 展示 AutoGen Core runtime 的單代理與群聊協作流程，包含串流輸出、工具調用、Round-Robin 交談與 ClosureAgent 將結果推送至前端。[README.md:1](README.md#L1)

## 主要檔案
- `SimpleAssistantAgent.py`：實作可串流輸出的 `RoutedAgent`，支援工具呼叫與二次反思；訊息處理時會將串流片段透過 runtime 發布至 `task-results` topic，供 closure agent 轉給前端佇列。[SimpleAssistantAgent.py#L46-L178](SimpleAssistantAgent.py#L46)  
- `app_agent.py`：以 `SingleThreadedAgentRuntime` 註冊 `SimpleAssistantAgent` 與 `ClosureAgent`，把輸出放入 Chainlit session 的 `asyncio.Queue`，並在 `@cl.on_message` 中啟動 `runtime.send_message` 同步串流到 UI。[app_agent.py#L55-L111](app_agent.py#L55)
- `app_team.py`：建立助手與評論家兩名代理、GroupChatManager 與 ClosureAgent，使用 topic 分流並由 UI 協程消費隊列資料，展示多代理輪詢與審核流程。[app_team.py#L37-L201](app_team.py#L37)

## 安裝與設定
- 建議安裝 Chainlit、`autogen-core`、`autogen-ext[openai]` 與 `pyyaml`，並依 `model_config_template.yaml` 建立 `model_config.yaml` 填入模型資訊。[README.md:28](README.md#L28)｜[README.md:40](README.md#L40)

## 執行方式
- 單代理版：於目錄執行 `chainlit run app_agent.py`，可觀察 `SimpleAssistantAgent` 串流輸出與工具反思流程。[README.md:51](README.md#L51)
- 群聊版：執行 `chainlit run app_team.py`，體驗 GroupChatManager 以 Round-Robin 方式安排助手與評論家發言，當評論家輸出 `APPROVE` 即結束對話。[README.md:60](README.md#L60)｜[app_team.py#L50-L76](app_team.py#L50)

## 延伸參考
- AutoGen Core 的 Topic 與 runtime 設計可參考設計文件了解事件驅動流程；另外 Chainlit 串流顯示範例可擴充支援更多代理或 UI 元件。[docs/design/02 - Topics.md](../../docs/design/02%20-%20Topics.md)｜[AutoGen Core User Guide](https://microsoft.github.io/autogen/stable/user-guide/core-user-guide/index.html)

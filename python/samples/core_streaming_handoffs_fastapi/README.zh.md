# FastAPI 串流與代理移交範例詳細說明

## 範例概述
- 本範例示範如何在 AutoGen Core 中結合 FastAPI、串流輸出與多代理移交（handoff）模式，提供客戶服務情境下的分工代理：初始分診、銷售、問題與維修，並保留會話歷史與 JSON 狀態檔。[README.md:1](README.md#L1)

## 架構重點
- `app.py`：  
  - 於應用壽命週期載入 `model_config.yaml`，註冊三個 `AIAgent` 與一個 `UserAgent`，並建立 `asyncio.Queue` 作為串流通道；每位代理擁有自己的 topic 類型與委派工具。[app.py#L62-L172](app.py#L62)  
  - `/chat/completions` 端點會將使用者訊息封裝為 `UserTask` 發布給分診代理，並將佇列內容串流回瀏覽器，直到收到 `STREAM_DONE`。[app.py#L193-L135](app.py#L193)
- `agent_base.py`：`AIAgent` 會呼叫 `ChatCompletionClient.create_stream` 將串流片段推入佇列，遇到函式呼叫時可執行本地工具或委派至其他 topic，再把最終結果封裝為 `AgentResponse` 回傳給 `UserAgent`。[agent_base.py#L24-L134](agent_base.py#L24)
- `agent_user.py`：`UserAgent` 接收 `AgentResponse` 後以 `BufferedChatCompletionContext` 儲存最多 10 則歷史並寫入 `chat_history/history-<conversation>.json`，最後放入 `STREAM_DONE` 結束串流。[agent_user.py#L16-L44](agent_user.py#L16)
- `tools.py` 與 `tools_delegate.py`：定義實際服務操作（查詢商品、執行訂單、退款）以及移交工具，讓代理可透過函式呼叫把對話轉移到適合的團隊成員。[tools.py#L1-L71](tools.py#L1)｜[tools_delegate.py#L1-L24](tools_delegate.py#L1)
- `topics.py`：集中管理 triage / sales / issues_and_repairs / user 等 topic 常數，避免拼寫錯誤。[topics.py#L1-L13](topics.py#L1)

## 執行步驟
- 安裝 FastAPI、Uvicorn、`autogen-core`、`autogen-ext[openai]` 等依賴並建立 `model_config.yaml` 後，執行 `uvicorn app:app --host 0.0.0.0 --port 8501 --reload` 啟動服務。[README.md:29](README.md#L29)｜[README.md:46](README.md#L46)
- 於瀏覽器開啟 `http://localhost:8501` 使用內建前端體驗串流聊天；每筆對話會建立 `chat_history` JSON 以供後續分析或回放。[README.md:49](README.md#L49)｜[agent_user.py#L29-L39](agent_user.py#L29)

## 延伸參考
- AutoGen 官方文件的「Handoffs」設計模式詳述代理移交策略與最佳實務，可作為擴充更多部門或自動化流程的參考。[Handoffs Design Pattern](https://microsoft.github.io/autogen/stable/user-guide/core-user-guide/design-patterns/handoffs.html)

# FastAPI 串流回應範例詳細說明

## 範例概述
- 展示如何透過 AutoGen Core runtime 與 FastAPI 構建最小化的聊天 API，使用 `StreamingResponse` 將模型串流輸出逐段送回客戶端，同時維持多輪對話上下文。[README.md:1](README.md#L1)

## 主要模組
- `app.py`：  
  - 在 lifespan 中讀取 `model_config.yaml` 建立 `ChatCompletionClient`，註冊 `MyAgent` 後啟動 `SingleThreadedAgentRuntime`。[app.py#L84-L105](app.py#L84)  
  - `MyAgent.handle_user_message` 將傳入的訊息列表轉為 `UserMessage`／`AssistantMessage`，透過 `create_stream` 串流模型輸出並推入全域佇列，最後回傳完整結果字串。[app.py#L52-L81](app.py#L52)
- `chat_completions_stream`：檢查輸入格式後啟動背景任務送出訊息，並在生成器中持續取出佇列內容轉為 JSON 字串回傳，遇到 `STREAM_DONE` 或錯誤即結束串流。[app.py#L110-L135](app.py#L110)

## 使用方式
- 依 README 安裝 FastAPI、Uvicorn、`autogen-core`、`autogen-ext[openai]` 等套件後建立 `model_config.yaml`，於目錄執行 `uvicorn app:app --host 0.0.0.0 --port 8501 --reload`。[README.md:18](README.md#L18)｜[README.md:33](README.md#L33)
- 透過 `POST /chat/completions` 並傳入 `{ "messages": [...] }` 即可獲得串流回應，可使用 README 提供的 curl 或 Python `requests` 範例測試。[README.md:42](README.md#L42)

## 延伸參考
- 若需引入工具調用、代理協作或手動移交，可參考同目錄的進階 handoff 範例，以及 AutoGen Core 官方指南掌握更多 runtime 與串流設計模式。[AutoGen Core User Guide](https://microsoft.github.io/autogen/stable/user-guide/core-user-guide/index.html)

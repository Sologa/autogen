# Python 與 .NET 代理互操作範例詳細說明

## 範例概述
- 展示如何在 AutoGen 中透過 gRPC worker runtime 與 Protobuf 訊息讓 Python 代理與 .NET Agent Runtime 互通，包含訂閱 .NET 發布的話題與從 Python 發送訊息給 .NET 代理。[README.md:1](README.md#L1)

## 主要檔案
- `hello_python_agent.py`：  
  - 從 `.env` 取得 `AGENT_HOST`，調整為純主機名稱後以 `GrpcWorkerAgentRuntime`（Payload 設為 Protobuf）連線，註冊 `UserProxy` 並訂閱多個 .NET 定義的 topic，例如 `HelloTopic`、`agents.NewMessageReceived` 等。[hello_python_agent.py#L27-L50](hello_python_agent.py#L27)  
  - 示範如何發送 `NewMessageReceived` 與 `Output` Protobuf 訊息給 `HelloTopic`，供 .NET 代理處理。[hello_python_agent.py#L53-L66](hello_python_agent.py#L53)
- `user_input.py`：`UserProxy` 代理負責與使用者互動，收到 `Input` 時從終端取得文字並轉為 `NewMessageReceived` 訊息發回 runtime，收到 `Output` 則在主控台顯示內容。[user_input.py#L11-L34](user_input.py#L11)
- `protos/agent_events_pb2.py`：儲存與 .NET 代理共享的 Protobuf 訊息定義，Python 端依此序列化／反序列化跨語言事件。

## 執行步驟
- 先在 `dotnet/samples/Hello/Hello.AppHost` 執行 `dotnet run` 啟動 .NET Aspire AppHost（包含 .NET Agent Runtime 與 HelloAgent），再於本目錄執行 `python hello_python_agent.py` 即可與 .NET 端互動。[README.md:7](README.md#L7)
- 過程中終端會提示輸入訊息，輸入 `exit` 可離開；Python 代理同時也會自動發布示範訊息顯示跨語言傳遞效果。[hello_python_agent.py#L53-L66](hello_python_agent.py#L53)

## 延伸參考
- 若需擴充更多跨語言代理，可參考 AutoGen Core Worker Protocol 與 .NET 端文件，了解 topic 命名、序列化與安全認證配置方式。[docs/design/03 - Agent Worker Protocol.md](../../docs/design/03%20-%20Agent%20Worker%20Protocol.md)｜[AutoGen 官方文件](https://microsoft.github.io/autogen/stable/index.html)

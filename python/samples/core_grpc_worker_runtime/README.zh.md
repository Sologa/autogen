# gRPC Worker Runtime 範例詳細說明

## 範例概述
- 此目錄提供多個最小化示例，演示如何使用 `GrpcWorkerAgentRuntimeHost` 與 `GrpcWorkerAgentRuntime` 建立 AutoGen Core 的跨程序通訊，涵蓋發布訂閱、RPC 呼叫及訊息級聯等情境。[run_host.py#L8-L21](run_host.py#L8)

## 主要範例
- `run_host.py`：啟動 gRPC Host（預設 `localhost:50051`），持續運行直到收到終止訊號，是所有 worker 示例的共用服務端。[run_host.py#L8-L21](run_host.py#L8)
- 發布／訂閱 (`run_worker_pub_sub.py`)：  
  - 定義 `GreeterAgent` 與 `ReceiveAgent` 兩個路由代理，透過 `DefaultTopicId` 廣播問候、回覆與回饋訊息，並示範如何註冊自訂資料型別序列化器。[run_worker_pub_sub.py#L17-L84](run_worker_pub_sub.py#L17)  
  - 程式啟動後自動發出 `AskToGreet` 訊息，並在收到 `ReturnedGreeting` 後再送出 `Feedback` 形成閉環。[run_worker_pub_sub.py#L63-L86](run_worker_pub_sub.py#L63)
- RPC (`run_worker_rpc.py`)：  
  - `GreeterAgent` 內部使用 `send_message` 呼叫 `ReceiveAgent` 取得同步回覆，再轉為 `Feedback` 廣播，示範 gRPC worker 間的直接訊息傳遞流程。[run_worker_rpc.py#L45-L52](run_worker_rpc.py#L45)
- 級聯 (`run_cascading_worker.py` / `run_cascading_publisher.py`)：  
  - `CascadingAgent` 收到 `CascadingMessage` 後轉發下一回合訊息並通知 `ObserverAgent`，形成連鎖事件；publisher 範例負責發送初始訊息並註冊觀察者。[agents.py#L18-L42](agents.py#L18)｜[run_cascading_publisher.py#L6-L12](run_cascading_publisher.py#L6)  
  - 另可獨立啟動 `run_cascading_worker.py` 在不同程序註冊多個 `CascadingAgent` 觀察訊息傳遞。[run_cascading_worker.py#L8-L14](run_cascading_worker.py#L8)

## 使用步驟
- 先執行 `run_host.py` 啟動服務端，再視想測試的通訊模式個別執行 `run_worker_pub_sub.py`、`run_worker_rpc.py`、`run_cascading_publisher.py` 與 `run_cascading_worker.py` 觀察輸出。
- 每個 worker 皆支援 `Ctrl+C` 或訊號終止；如需觀察更詳細日誌，可取消註解各檔案內的 logging 設定。

## 延伸參考
- AutoGen Core Worker Runtime 與 Topic 概念可參考官方指南與設計文件，了解更多部署與擴充模式。[AutoGen Core User Guide](https://microsoft.github.io/autogen/stable/user-guide/core-user-guide/index.html)｜[docs/design/03 - Agent Worker Protocol.md](../../docs/design/03%20-%20Agent%20Worker%20Protocol.md)

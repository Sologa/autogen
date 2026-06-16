# 語意路由分散式範例詳細說明

## 範例概述
- 此範例示範如何利用 AutoGen Core 的語意路由代理，在分散式 gRPC runtime 架構下根據使用者輸入自動選擇對應的專責代理（如人資或財務），並透過 ClosureAgent 將回應回傳至使用者端。[README.md:1](README.md#L1)

## 主要元件
- `run_semantic_router.py`：  
  - 建立 `GrpcWorkerAgentRuntime`、註冊財務與人資的 `WorkerAgent`、`UserProxyAgent`、ClosureAgent 與 `SemanticRouterAgent`，並以 `MockIntentClassifier` 根據關鍵字決定路由目標。[run_semantic_router.py#L22-L118](run_semantic_router.py#L22)  
  - 終端互動中若輸入包含指定關鍵字則轉送至對應代理；輸入 `END` 則由代理發布 `TerminationMessage` 終止會話並提示使用者重新開始。[run_semantic_router.py#L61-L77](run_semantic_router.py#L61)
- `_semantic_router_agent.py`：實作 `SemanticRouterAgent`，接收 `UserProxyMessage` 後呼叫分類器決定目標代理；若無匹配則發布 `TerminationMessage`，避免訊息落入無效管道。[ _semantic_router_agent.py#L19-L62](_semantic_router_agent.py#L19)
- `_agents.py`：定義財務／人資 `WorkerAgent` 處理訊息與結束條件，`UserProxyAgent` 將代理回覆轉送給 ClosureAgent，再由 ClosureAgent 對外回傳最終結果。[ _agents.py#L11-L64](_agents.py#L11)
- `_semantic_router_components.py`：提供抽象化的代理登錄、意圖分類與訊息資料結構，便於替換更進階的路由策略。[ _semantic_router_components.py#L1-L120](_semantic_router_components.py#L1)

## 執行流程
- 先啟動 gRPC Host（可重用 `python/samples/core_grpc_worker_runtime/run_host.py`），再執行 `python run_semantic_router.py` 開始互動；輸入包含「finance/money/budget」等關鍵字會路由至財務代理，輸入「hr/human resources/employee」則轉至人資代理，其他內容會觸發終止訊息。[run_semantic_router.py#L38-L58](run_semantic_router.py#L38)
- 當代理回覆使用者或宣告結束時，`UserProxyAgent` 會把訊息轉給 ClosureAgent，並在主控台提示結果與下一步輸入。[run_semantic_router.py#L61-L77](run_semantic_router.py#L61)

## 延伸參考
- AutoGen Core 事件驅動與 Topic 架構可透過設計文件深入了解，亦可參考官方 Core 指南擴充更複雜的語意分類器或整合企業搜尋服務。[docs/design/02 - Topics.md](../../docs/design/02%20-%20Topics.md)｜[AutoGen Core User Guide](https://microsoft.github.io/autogen/stable/user-guide/core-user-guide/index.html)

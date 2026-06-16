# AutoGen Core 棋局代理範例詳細說明

## 範例概述
- 透過 AutoGen Core runtime 模擬兩個棋士代理互相對弈，示範如何使用 `ToolAgent` 提供函式工具、以 `tool_agent_caller_loop` 觸發函式呼叫並將結果回饋至主代理，整個流程都在事件驅動架構下完成。[README.md:1](README.md#L1)｜[main.py#L1-L78](main.py#L1)

## 主要組件
- `PlayerAgent`：訂閱預設 Topic，接收棋盤狀態訊息後透過 `tool_agent_caller_loop` 呼叫工具代理，取得 LLM 回應並重新發布棋步指令。[main.py#L41-L78](main.py#L41)
- 棋盤工具：針對黑白雙方各註冊 `get_legal_moves`、`make_move`、`get_board` 等函式，並在執行 `make_move` 後輸出棋盤文字與移動細節。[main.py#L144-L207](main.py#L144)
- `chess_game`：建立棋盤、註冊工具與代理後，將初始訊息送至白方，觸發整個對弈迴圈直到 runtime idle。[main.py#L144-L262](main.py#L144)
- `main()`：讀取 `model_config.yml` 創建 `ChatCompletionClient`，初始化 `SingleThreadedAgentRuntime`，啟動遊戲並於完成後釋放模型客戶端資源。[main.py#L248-L279](main.py#L248)

## 安裝與執行
- 安裝 `autogen-core`、`autogen-ext[openai,azure]`、`chess` 與 `pyyaml` 等依賴後建立 `model_config.yml`，於目錄執行 `python main.py` 即可啟動模擬對弈流程。[README.md:10](README.md#L10)｜[README.md:21](README.md#L21)

## 延伸參考
- AutoGen Core 工具代理與 Topic 機制可參考官方 Core 指南與設計文件，了解如何擴充其他棋類或回合制互動的代理模式。[AutoGen Core User Guide](https://microsoft.github.io/autogen/stable/user-guide/core-user-guide/index.html)｜[docs/design/02 - Topics.md](../../docs/design/02%20-%20Topics.md)

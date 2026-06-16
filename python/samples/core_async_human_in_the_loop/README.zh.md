# AutoGen Core 非同步人類介入範例詳細說明

## 範例概述
- 此範例展示如何在 AutoGen Core runtime 中實作「暫停等待人類輸入」的工作流程，包含狀態持久化、恢復、以及於人類補充資訊後繼續執行工具的流程。[README.md:1](README.md#L1)

## 系統架構
- `SlowUserProxyAgent`：訂閱 `scheduling_assistant_conversation` 主題，接收助理訊息後發布 `GetSlowUserMessage`，並透過 `BufferedChatCompletionContext` 保存最近五則訊息供恢復使用。[main.py#L96-L122](main.py#L96)
- `SchedulingAssistantAgent`：以系統提示建立行程助理，透過 `ScheduleMeetingTool` 完成函式呼叫；若從模型得到函式呼叫結果便發布 `TerminateMessage` 結束流程，否則繼續對話。[main.py#L148-L204](main.py#L148)
- `NeedsUserInputHandler` 與 `TerminationHandler`：分別攔截 `GetSlowUserMessage` 與 `TerminateMessage`，決定 runtime 是否須暫停等待人類輸入或已完成任務。[main.py#L214-L252](main.py#L214)
- `MockPersistence`：以記憶體模擬永續層，提供 `save_state`/`load_state` 範例，展示如何在等待期間保存整個 runtime 狀態。[main.py#L82-L94](main.py#L82)

## 執行流程
- `main()` 會先註冊代理與處理器，再依 `latest_user_input` 決定是否載入既有狀態與發送使用者訊息，最後啟動 runtime 並於需要人類輸入或流程完成時停止。[main.py#L254-L320](main.py#L254)
- `run_main()` 以遞迴方式在取得人類輸入後再次呼叫 `main()`，確保每次輸入都能恢復之前儲存的狀態並繼續對話。[main.py#L337-L355](main.py#L337)

## 執行方法
- 安裝 `autogen-ext[openai,azure]` 與 `pyyaml` 後，依 `model_config_template.yml` 建立模型設定檔，於目錄執行 `python main.py` 逐步體驗等待輸入、恢復與完成流程。[README.md:10](README.md#L10)｜[README.md:20](README.md#L20)

## 延伸參考
- AutoGen Core 的事件驅動模型與 Topic 概念可參考設計文件進一步理解代理註冊與訊息傳遞；官方 Core 使用指南亦對 runtime 與介入處理器有更完整的說明。[docs/design/01 - Programming Model.md](../../docs/design/01%20-%20Programming%20Model.md)｜[AutoGen Core User Guide](https://microsoft.github.io/autogen/stable/user-guide/core-user-guide/index.html)

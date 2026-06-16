# core_chainlit 與 core_distributed-group-chat 擴充分析

## 研究問題
- 釐清 `core_chainlit` 是否能改寫成多代理協作的 LaTeX `.tex` 語法修復工具。
- 評估 `core_distributed-group-chat` 是否能延伸為協同撰寫/修訂論文的流程模組。

## `core_chainlit`：改造成 LaTeX 語法修正協作工具

### 現有能力盤點
- 範例中透過 `SimpleAssistantAgent` 支援串流輸出、工具呼叫與多種訊息處理（使用者訊息、群聊轉錄、輪詢發言）。[來源](python/samples/core_chainlit/SimpleAssistantAgent.py#L46)  
- `app_team.py` 建立助理與評論代理，將訊息導入 `GroupChatManager` 並以 `ClosureAgent` 串流結果到 Chainlit UI。[來源](python/samples/core_chainlit/app_team.py#L95)  
- 系統可根據批注/指令切換回合、停止回應，適合加入不同角色（如語法解析、修復、驗證）。[來源](python/samples/core_chainlit/app_team.py#L37)

### 建議的實作步驟
1. **規劃代理角色**
   - 定義至少三個代理：`ParserAgent`（解析與定位錯誤）、`FixerAgent`（提出修正）、`ReviewerAgent`（檢查並決定是否接受）。可複製現有助理/評論設定，僅更換 `system_message` 與訂閱的 topic。[來源](python/samples/core_chainlit/app_team.py#L95)
2. **導入 LaTeX 工具鏈**
   - 新增 Python 工具類（繼承 `Tool`）包裝 `latexmk` 或 `chktex`，由 `SimpleAssistantAgent` 透過 `tools` 參數呼叫並回傳錯誤訊息。[來源](python/samples/core_chainlit/SimpleAssistantAgent.py#L91)
   - 在工具執行後將編譯結果寫入共享暫存（例如 `asyncio.Queue`），供其他代理檢視。
3. **擴充對話記錄與檔案管理**
   - 在 `GroupChatMessage` 攜帶檔案路徑、行號等額外欄位（可擴充 Pydantic 模型）。[來源](python/samples/core_chainlit/SimpleAssistantAgent.py#L37)
   - 新增協同儲存層，例如將代理修正寫入 Git 分支或臨時檔案，並在 UI 顯示差異。
4. **整合 Chainlit UI**
   - 利用既有 `ClosureAgent` 串流能力顯示每個代理的回饋，必要時擴充 `pass_msg_to_ui` 以標示代理職責。[來源](python/samples/core_chainlit/app_team.py#L160)
   - 於 Chainlit 動作加入「套用修正」「重新編譯」等按鈕，觸發對應代理流程。
5. **測試流程**
   - 撰寫腳本模擬錯誤 `.tex` 檔案，檢驗代理能否定位錯誤→提出修正→重新編譯並通過 `ReviewerAgent`。

### 風險與注意事項
- 需要處理長文與多檔案同步問題，可考慮為每輪輸入限制篇幅並分段處理。
- LaTeX 編譯耗時，建議以非同步佇列與超時控制避免阻塞 `SimpleAssistantAgent` 事件循環。[來源](python/samples/core_chainlit/SimpleAssistantAgent.py#L68)
- 若要在多文件上共用記憶，需要額外的狀態儲存或資料庫。

## `core_distributed-group-chat`：成為協寫論文模組

### 現有能力盤點
- 範例將 Writer、Editor、GroupChatManager、UI 等代理分散在不同 gRPC runtime，以主題訂閱協作並於 UI 串流輸出，適合延伸為跨階段任務。[來源](python/samples/core_distributed-group-chat/_agents.py#L20)
- `GroupChatManager` 可根據對話歷史挑選下一位發言者並宣告終止條件，利於導入論文分工（撰寫、審閱、參考文獻、數據檢驗等角色）。[來源](python/samples/core_distributed-group-chat/_agents.py#L69)
- 角色描述與行為可透過 `config.yaml` 的 system prompt 調整，避免硬編碼。[來源](python/samples/core_distributed-group-chat/config.yaml#L9)

### 建議的實作步驟
1. **角色設計**
   - 擴充 `config.yaml`，加入 `ResearcherAgent`（負責撰寫段落）、`CitationAgent`（查找並格式化引用）、`PolishAgent`（語言修飾）等角色描述與 topic。[來源](python/samples/core_distributed-group-chat/config.yaml#L9)
2. **文件狀態管理**
   - 在 `_agents.py` 的 `publish_message_to_ui_and_backend` 中加入寫檔或 Git API 呼叫，把每回合輸出合併進論文草稿版本控制。[來源](python/samples/core_distributed-group-chat/_agents.py#L195)
   - 建立共享儲存服務（如 Redis 或資料庫）存放段落、批註、任務清單，以便不同代理讀寫。
3. **任務調度**
   - 修改 `GroupChatManager` 選擇邏輯，使其根據論文進度（例如章節、待修段落）選擇合適代理，或維持最大回合數避免長時間迴圈。[來源](python/samples/core_distributed-group-chat/_agents.py#L121)
   - 可以加入「提交草稿」「待審查」等特殊訊號觸發後續流程。
4. **UI 與人類介入**
   - 擴充 `run_ui.py`，在 Chainlit 提供段落節點、審稿建議與歷史版本切換；必要時加入人工審核按鈕來覆核代理輸出。[來源](python/samples/core_distributed-group-chat/run_ui.py#L1)
5. **整合外部工具**
   - 透過 `BaseGroupChatAgent` 的工具化輸入，接入文獻搜尋 API、引用格式化工具、自動校對等服務。[來源](python/samples/core_distributed-group-chat/_agents.py#L49)
   - 若需編譯 LaTeX，可類似 `core_chainlit` 做法在某代理中執行編譯並回傳結果。

### 風險與注意事項
- 分散式架構需確保 gRPC 連線穩定並處理序列化格式，新增自訂訊息型別時要同步更新 `get_serializers`。[來源](python/samples/core_distributed-group-chat/_utils.py#L1)
- 多終端開發者同時操作時需要權限控管與鎖定策略，避免覆寫彼此內容。
- Azure OpenAI 設定位於 `client_config`，若改用其他模型需同步調整部署與認證。[來源](python/samples/core_distributed-group-chat/config.yaml#L25)

## 參考資料
- AutoGen Core 官方文件：<https://microsoft.github.io/autogen/stable/user-guide/core-user-guide/index.html>
- Chainlit 整合說明：<https://microsoft.github.io/autogen/stable/user-guide/core-user-guide/tutorial/teams.html>

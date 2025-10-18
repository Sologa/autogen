# PDF 格式審查代理系統規格草案（待補完）

> 本草案依據倉庫 `docs/` 全部文件（含 `docs/design/*.md` 與 `docs/dotnet/**`）整理。所有標記 `TODO:` 的項目仍須補齊，以確保實作與內部設計文檔一致。

---

## 1. 系統目標與輸出
- **任務描述**：審查指定 PDF（路徑：`/Users/xjp/Desktop/Survey-with-LLMs/Survey-for-survey-review-with-LLMs/SurveyX/sandbox/latex_citation_fix/agent_workspace/error_detection` 以下簡稱為 ROOT_PATH），逐頁找出格式問題，並在結果中標示頁碼與行號／定位資訊。
- **最終輸出**：
  - 格式：`JSON`
  - 欄位：`規則 ID、頁碼、行號/座標、問題敘述、嚴重度、原文片段`
  - 儲存位置與命名規則：`$ROOT_PATH/err_data/p{第幾頁}.json`

---

## 2. 架構概覽
```text
主程式/工作流
  ├─ Orchestrator Agent  (autogen_agentchat AssistantAgent)
  │     ├─ 呼叫工具: get_pdf_page_count, invoke_specialist
  │     └─ 管理 Specialist 回報 → 合併報告
  └─ Specialist Agent    (autogen_agentchat AssistantAgent 或 Team)
        ├─ 呼叫工具: get_text_from_page, extract_images_from_page
        ├─ 文字分析 LLM
        └─ 視覺分析 LLM（若頁面有圖片）
```
- Runtime：建議以 `SingleThreadedAgentRuntime`（`python/packages/autogen-core/src/autogen_core/_single_threaded_agent_runtime.py:83`）建構，並遵守 `docs/design/03 - Agent Worker Protocol.md` 所述的初始化（啟動 runtime、註冊 agent）、運作與終止流程。
- Agent 組織：可使用 `RoundRobinGroupChat`（`python/packages/autogen-agentchat/src/autogen_agentchat/teams/_group_chat/_round_robin_group_chat.py`）與 `TaskRunner`（`python/packages/autogen-agentchat/src/autogen_agentchat/base/_task.py:10`）組合。如採 `RoutedAgent`，需根據 `docs/design/02 - Topics.md` 使用 `@default_subscription` 與 `TopicId`（`type` + `source`）建立訂閱。
- 事件建模：所有訊息均以 CloudEvents 形式傳遞（`docs/design/01 - Programming Model.md`），須為事件設定唯一 `id`、`source`、`type`。
- TODO: 確認是否需要 `MagenticOneGroupChat` 式 orchestrator 功能或僅保留簡化模式。

---

## 3. 元件規格

### 3.1 PDF 工具模組（`pdf_tools.py`）
| 函式 | 來源參考 | 需求 | TODO |
| --- | --- | --- | --- |
| `get_pdf_page_count(pdf_path: str) -> int` | 需使用 `fitz.open(pdf_path)` | 回傳總頁數，錯誤時拋出自訂例外 | TODO: 定義例外類別、加入重試/快取策略（可參考 `MessageDroppedException` 框架與 `docs/design/05 - Services.md` 的可靠性建議） |
| `get_text_from_page(pdf_path: str, page_number: int, *, with_layout: bool) -> str` | 使用 `page.get_text("dict")` 或 `page.get_text("blocks")` | 需輸出模擬行號或 (bbox, text) 結構 | TODO: 行號演算法（多欄、頁首頁尾處理） |
| `extract_images_from_page(pdf_path: str, page_number: int) -> list[str]` | `page.get_images(full=True)` + `doc.extract_image(xref)` | 回傳圖檔路徑，需確保檔名格式一致 | TODO: 暫存目錄、檔案清理策略 |

> 根據 `docs/design/01 - Programming Model.md` 的「Agents subscribe to events」原則，PDF 工具應以 `FunctionTool`（`python/packages/autogen-core/src/autogen_core/tools/_function_tool.py`）或 `StaticStreamWorkbench` 形式提供，讓 Agent 透過事件驅動方式取用資料。

### 3.2 Specialist Agent
- 建議繼承 `AssistantAgent`（`python/packages/autogen-agentchat/src/autogen_agentchat/agents/_assistant_agent.py:123`）。
- 系統訊息草稿：
  ```
  你是 PDF 排版稽核專家，會自行呼叫工具以取得頁面文字與圖片，依照[格式規則表]提出問題列表。
  TODO: 貼上格式規則表
  ```
- 工具註冊：利用 `ComponentModel` / `ComponentLoader`（`python/packages/autogen-core/src/autogen_core/_component_config.py:150`）或程式碼直接注入。
- LLM 客戶端：
  - 文字：`OpenAIChatCompletionClient`（`python/packages/autogen-ext/src/autogen_ext/models/openai/_openai_client.py`）或 `Anthropic`。
  - 視覺：`autogen_ext.agents.web_surfer.MultimodalWebSurfer` 或具備視覺能力的 `OpenAIChatCompletionClient` 模型。
  - TODO: 選定模型、max tokens, temperature、`parallel_tool_calls` 設定與 retries；並於工作完成後呼叫 `model_client.close()` 釋放連線。
- 產出格式：`List[Dict[str, Any]]`，欄位需對應 §1 的輸出 schema。
- 錯誤處理：若工具異常需回傳錯誤結構供 Orchestrator 判斷（例如 `{"error": "...", "page": n}`）。
- 上下文管理：若 LLM 上下文限制可能超出，需設定 `model_context=BufferedChatCompletionContext(...)`（`python/packages/autogen-core/src/autogen_core/model_context/_buffered_chat_completion_context.py`）。
- Debug 與串流：可使用 `Console` 或 `RichConsole`（`python/packages/autogen-agentchat/src/autogen_agentchat/ui/_console.py`, `python/packages/autogen-ext/src/autogen_ext/ui/_rich_console.py`）在開發期驗證輸出。

### 3.3 Orchestrator Agent
- 可使用 `AssistantAgent` 或自訂 `RoutedAgent`（`python/packages/autogen-core/src/autogen_core/_routed_agent.py`）。
- 職責：
  1. 呼叫 `get_pdf_page_count` → 建立審查任務列表。
  2. 逐頁委派：可透過 `AgentTool`（`python/packages/autogen-agentchat/src/autogen_agentchat/tools/_agent.py`）將 Specialist 包裝成工具，於 Orchestrator 內部用 for 迴圈呼叫。
  3. 合併 Specialist 回傳 → 檢查 TODO 欄位是否完整 → 組成最終報告。
- 建議流程（擬程式）：
  ```python
  runtime = SingleThreadedAgentRuntime()
  orchestrator = AssistantAgent(..., tools=[specialist_tool, get_pdf_page_count_tool])
  task_result = await orchestrator.run(task="審查 /path/to.pdf")
  ```
- 如採 `RoutedAgent`，需以 `@message_handler` 裝飾器處理 `PageReviewCommand` 類訊息並使用 `publish_message` 觸發 Specialist（同 Core Quick Start 範例）。 
- TODO: 決定是否允許 Orchestrator 自主規劃（使用 `model_client_stream=True` 與 `max_tool_iterations>1`）或改由外層 Python 控制（`docs/design/01 - Programming Model.md` 指出可利用 Orchestrator agent 管理流程，也可由外部流程依 Topic 控制）。
- TODO: 補充終止條件與 `TerminationCondition` 設定（可參考 `autogen_agentchat.conditions.StopMessageTermination`）。

---

## 4. 實作步驟與待辦
1. **規則清單整理**  
   - TODO: 建立 `rules.yaml`（欄位：`id`, `description`, `severity`, `examples_ok`, `examples_ng`）。  
   - TODO: 在 Specialist 系統提示中引用，並於 Orchestrator 驗證輸出。

2. **工具開發**  
   - 參考 `python/packages/autogen-ext/src/autogen_ext/tools/code_execution/_code_execution.py` 的工具註冊方式及 `docs/design/05 - Services.md` 對服務組件的抽象（例如 Worker 與 Gateway 協作）。  
   - TODO: 為每支工具撰寫單元測試（使用樣本 PDF）。  
   - TODO: 錯誤時回傳一致結構（含 `error_code`, `details`）。

3. **Agent 組裝**  
   - 使用 `TaskRunner` 介面（`python/packages/autogen-agentchat/src/autogen_agentchat/base/_task.py:10`）讓 Orchestrator/Specialist 具備 `run` / `run_stream`，並結合 `docs/design/03 - Agent Worker Protocol.md` 描述的事件驅動迴圈來設計內部流程。  
   - TODO: 定義 `TaskResult` 中 `stop_reason` 的使用方式，並與 Orchestrator 終止條件對應。

4. **報告合併**  
   - TODO: 實作 `aggregate_findings(findings_per_page)`，輸出符合 §1 表格。  
   - 可利用 `pydantic` 模型建立 Schema（參考 `python/packages/autogen-agentchat/src/autogen_agentchat/messages.py`），並符合 `docs/design/04 - Agent and Topic ID Specs.md` 對 ID 字串的格式限制（例如 topic type 使用反向網域命名）。

5. **異常與重試策略**  
   - 依 `MessageDroppedException`（`python/packages/autogen-core/src/autogen_core/exceptions.py:10`）擴充必要例外；同時遵循 `docs/design/05 - Services.md` 的可靠性建議（例如 Registry、Routing 管理訂閱、Gateway 確保訊息送達）。  
   - TODO: 決定重試次數、退避機制、是否人工確認。

6. **測試與驗證**  
   - TODO: 準備標註過的 PDF 測試集。  
    - TODO: 建立 CI 任務執行工具、Agent 單元測試，並確保測試流程符合 `docs/dotnet/core/differences-from-python.md` 所述的訊息投遞邏輯（例如必要時決定是否允許 DeliverToSelf 行為）。

---

## 5. 未決議事項（請逐項確認）
- [ ] TODO: 決定 Orchestrator 是否採 Python 外部控制或純 Agent 內部迴圈。  
- [ ] TODO: 視覺模型選型與 API Key 管理。  
- [ ] TODO: 產出報告儲存位置、檔名與版本控制策略。  
- [ ] TODO: 需求是否包含人工稽核（Human-in-the-loop），若是需導入 `UserProxyAgent`（`python/packages/autogen-agentchat/src/autogen_agentchat/agents/_user_proxy_agent.py:84`）。
- [ ] TODO: 是否支援中途暫停/續跑，需保存 Orchestrator/Specialist 狀態（參考 `save_state` / `load_state` in `BaseChatAgent`，並結合 `docs/design/03 - Agent Worker Protocol.md` 的 state save/load 流程）。
- [ ] TODO: 根據 `docs/design/03 - Agent Worker Protocol.md` 及 `docs/design/05 - Services.md` 評估是否需支援分散式 gRPC Worker Runtime 作為後續擴充。

---

## 6. 參考程式碼佈局
- `python/packages/autogen-core/src/autogen_core/_single_threaded_agent_runtime.py`
- `python/packages/autogen-core/src/autogen_core/tools/_function_tool.py`
- `python/packages/autogen-agentchat/src/autogen_agentchat/agents/_assistant_agent.py`
- `python/packages/autogen-agentchat/src/autogen_agentchat/tools/_agent.py`
- `python/packages/autogen-agentchat/src/autogen_agentchat/messages.py`
- `python/packages/autogen-ext/src/autogen_ext/models/openai/_openai_client.py`
- `python/packages/autogen-ext/src/autogen_ext/tools/mcp/_workbench.py`（工具整合參考）
- `python/packages/autogen-ext/src/autogen_ext/code_executors/_common.py`（錯誤處理可借鏡）
- 設計文檔：`docs/design/01 - Programming Model.md`、`docs/design/02 - Topics.md`、`docs/design/03 - Agent Worker Protocol.md`、`docs/design/04 - Agent and Topic ID Specs.md`、`docs/design/05 - Services.md`
- .NET 對照：`docs/dotnet/core/index.md`、`docs/dotnet/core/differences-from-python.md`

> 完成各 TODO 後，請將本規格提升為正式文件並與 `implementation_plan_pdf_review_agent.md` 交叉驗證，確保需求與實作對齊。

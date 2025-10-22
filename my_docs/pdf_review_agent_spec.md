# PDF 格式審查代理系統規格

> 本規格已根據用戶決策更新。所有標記 `TODO:` 的項目仍須補齊。

---

## 1. 系統目標與輸出
- **任務描述**：審查指定 PDF（路徑：`/Users/xjp/Desktop/Survey-with-LLMs/Survey-for-survey-review-with-LLMs/SurveyX/sandbox/latex_citation_fix/agent_workspace/error_detection` 以下簡稱為 ROOT_PATH），逐頁找出格式問題，並產出單頁報告與一份最終的總結報告。
- **過程輸出 (Per-Page Output)**：
  - 格式：`JSONL` (每行一個 JSON 物件)
  - 儲存位置與命名規則：`$ROOT_PATH/p{第幾頁}.jsonl`
  - 範例 (以第五頁為例, `p5.jsonl`):
    ```json
    {"行號":"L5","問題敘述":"引用標記仍為占位符，顯示參考文獻未完成解析","嚴重度":"高","原文片段":"[? ], but non-autoregressive refinement stages can claw back quality after coarse token generation"}
    {"行號":"L6","問題敘述":"引用標記仍為占位符，參考文獻缺失","嚴重度":"高","原文片段":"little incremental latency [? ]."}
    {"行號":"L7","問題敘述":"引用標記仍為占位符，參考文獻缺失","嚴重度":"高","原文片段":"Residual block-wise streaming in end-to-end speech decoders imposes latency floors even at very low token rates [? ]."}
    {"行號":"L8","問題敘述":"引用標記仍為占位符，參考文獻缺失","嚴重度":"高","原文片段":"Information redistribution across quantizer channels reduces LM predictability pressure and aligns codec design with text-conditioned generation [? ]."}
    {"行號":"L26","問題敘述":"多個引用以占位符呈現，顯示文獻列表未解析","嚴重度":"高","原文片段":"and TTS continuity/MOS for synthesis. [? ? ? ]"}
    {"行號":"L36","問題敘述":"多個引用以占位符呈現，顯示文獻列表未解析","嚴重度":"高","原文片段":"collectively making tokens more predictable for speech LMs, lowering WER, and improving fidelity at competitive bitrates. [? ? ? ? ? ]"}
    {"行號":"L44","問題敘述":"多個引用以占位符呈現，顯示文獻列表未解析","嚴重度":"高","原文片段":"and hybrid continuous–discrete approaches like Whisper-GPT that combine spectrogram features with discrete tokens to mitigate context-length blow-up and improve perplexity and negative log-likelihood. [? ? ? ]"}
    {"行號":"L54","問題敘述":"多個引用以占位符呈現，顯示文獻列表未解析","嚴重度":"高","原文片段":"and contrasts acoustic versus semantic tokens for controllability alongside the latency–quality trade-offs of non-autoregressive refinement. [? ? ? ? ? ]"}
    ```
- **最終輸出 (Final Report)**：
  - 格式：`JSON` (一個包含所有頁面問題的單一檔案)
  - 儲存位置與命名規則：`$ROOT_PATH/final_report.json`

---

## 2. 架構概覽
```text
主程式/工作流
  ├─ Orchestrator Agent  (autogen_agentchat AssistantAgent)
  │     ├─ 呼叫工具: get_pdf_page_count, aggregate_findings
  │     ├─ 透過 FunctionTool 委派 Specialist 進行單頁審查
  │     ├─ 審核 Specialist 產出 -> 要求重做或確認通過
  │     └─ 合併所有單頁報告 -> 產出最終報告
  └─ Specialist Agent    (autogen_agentchat AssistantAgent)
        ├─ 呼叫工具: get_text_from_page, extract_images_from_page
        └─ 文字與視覺分析 LLM
```
- Runtime：以 `SingleThreadedAgentRuntime` 建構，對應 AutoGen 官方建議的單機嵌入式 Runtime 與既有 playbook 實作。[AUTOGEN_USER_MANUAL.md:43](AUTOGEN_USER_MANUAL.md#L43)・[https://microsoft.github.io/autogen/dev/user-guide/core-user-guide/framework/agent-and-agent-runtime.html](https://microsoft.github.io/autogen/dev/user-guide/core-user-guide/framework/agent-and-agent-runtime.html)・[my_docs/pdf_review_agent_playbook.md:253](my_docs/pdf_review_agent_playbook.md#L253)
- Agent 組織：採用兩個 `AssistantAgent` 協作的刻意規劃流程，Orchestrator 以 FunctionTool 調用 Specialist，符合手冊中對 AssistantAgent 工具使用的說明。[AUTOGEN_USER_MANUAL.md:53](AUTOGEN_USER_MANUAL.md#L53)・[my_docs/pdf_review_agent_playbook.md:249](my_docs/pdf_review_agent_playbook.md#L249)
- 事件建模：所有訊息均以 CloudEvents 形式傳遞，對應設計文件所述事件模型。[docs/design/01 - Programming Model.md:7](docs/design/01%20-%20Programming%20Model.md#L7)

---

## 3. 元件規格

### 3.1 工具集 (Toolsets)

#### 3.1.1 基礎工具：通用指令執行
- **說明**：提供執行 Shell/Python 腳本的基礎能力。
- **函式範例**：
  | 函式 | 需求 | 來源 |
  | --- | --- | --- |
  | `execute_shell(command: str) -> dict` | 執行單一 shell 指令並回傳 stdout/stderr。 | [my_docs/pdf_review_agent_playbook.md:282](my_docs/pdf_review_agent_playbook.md#L282) |
  | `aggregate_findings(directory: str) -> str` | 讀取指定目錄下所有的 `.jsonl` 檔案，將它們合併成一個單一的 JSON 格式字串。 | [my_docs/pdf_review_agent_playbook.md:102](my_docs/pdf_review_agent_playbook.md#L102) |

> **安全模型：沙箱化執行 (Security Model: Sandboxed Execution)**
>
> **提醒**：若需隔離，建議以 `DockerCommandLineCodeExecutor` 封裝 `execute_shell`，並於流程啟動時呼叫 `start()` 建立容器、結束時呼叫 `stop()` 清理；單次指令不會自動重新建立容器，必要時可透過 `restart()` 觸發重建。[my_docs/pdf_review_agent_playbook.md:279](my_docs/pdf_review_agent_playbook.md#L279)・[python/packages/autogen-ext/src/autogen_ext/code_executors/docker/_docker_code_executor.py:405](python/packages/autogen-ext/src/autogen_ext/code_executors/docker/_docker_code_executor.py#L405)・[python/packages/autogen-ext/src/autogen_ext/code_executors/docker/_docker_code_executor.py:495](python/packages/autogen-ext/src/autogen_ext/code_executors/docker/_docker_code_executor.py#L495)
>
> - **執行器 (Executor)**：AutoGen 官方模組提供 Docker 與本地執行器；如無隔離需求，可先使用本地版再依情境切換。[AUTOGEN_USER_MANUAL.md:397](AUTOGEN_USER_MANUAL.md#L397)
> - **映像檔 (Image)**：容器基底需事先安裝 Python、PyMuPDF、pdftotext 與 AutoGen 依賴，可參考 `my_docs/DOCKER_BUILD_GUIDE.md` 的建置流程。[my_docs/DOCKER_BUILD_GUIDE.md:31](my_docs/DOCKER_BUILD_GUIDE.md#L31)
> - **映像備份 (Image Archive)**：建置完成的映像以 `surveyx-env-latest.tar` 保存於 `/Volumes/My Book/surveyx_data/docker_build/`，需手動載入時執行 `docker load -i "/Volumes/My Book/surveyx_data/docker_build/surveyx-env-latest.tar"`。

#### 3.1.2 專業工具：PDF 格式審查
- **說明**：提供針對 PDF 文件內容和結構的高層次分析能力，主要由 Specialist Agent 使用。
| 函式 | 需求 | 來源 | 實作範例/策略 |
| --- | --- | --- | --- |
| `get_pdf_page_count(pdf_path: str) -> int` | 回傳總頁數，錯誤時拋出自訂例外。需整合重試與快取。 | [my_docs/pdf_review_agent_playbook.md:65](my_docs/pdf_review_agent_playbook.md#L65)・[https://pymupdf.readthedocs.io/en/latest/document.html#Document.page_count](https://pymupdf.readthedocs.io/en/latest/document.html#Document.page_count) | **例外類別**：<br> `class PdfProcessingError(Exception): pass`<br>`class PdfNotFoundError(PdfProcessingError): pass`<br><br>**重試策略 (使用 decorator)**：<br>`@retry(stop=stop_after_attempt(3), wait=wait_fixed(2))`<br>`def get_pdf_page_count(...): ...`<br><br>**快取策略 (使用 decorator)**：<br>`from functools import lru_cache`<br>`@lru_cache(maxsize=128)`<br>`def get_pdf_page_count(...): ...` |
| `get_text_from_page(pdf_path: str, page_number: int) -> str` | 透過 `execute_shell` 呼叫 `pdftotext -f {page_number} -l {page_number} {pdf_path} - | nl -ba`。 | [my_docs/pdf_review_agent_playbook.md:73](my_docs/pdf_review_agent_playbook.md#L73)・[https://www.manpagez.com/man/1/pdftotext/](https://www.manpagez.com/man/1/pdftotext/) | `command = f"pdftotext -f {page_number} -l {page_number} {pdf_path} - | nl -ba"` |
| `extract_images_from_page(pdf_path: str, page_number: int) -> list[str]` | 圖片暫存於 `tmp_pdf_assets` 目錄，並在流程結束時清理。 | [my_docs/pdf_review_agent_playbook.md:82](my_docs/pdf_review_agent_playbook.md#L82)・[https://pymupdf.readthedocs.io/en/latest/page.html#Page.get_images](https://pymupdf.readthedocs.io/en/latest/page.html#Page.get_images) | **清理策略函式**：<br>`def cleanup_temp_assets():`<br> `  print("This function will be implemented to clean up the tmp_pdf_assets directory.")`<br> `  # TODO: Implement the actual cleanup logic here.` |

### 3.2 Agent 規格與職責

#### 3.2.1 Specialist Agent
- **職責**：專注於 PDF 內容的深度分析，接收單一頁面的審查任務並回報 `jsonl` 格式的結構化結果。[my_docs/pdf_review_agent_playbook.md:247](my_docs/pdf_review_agent_playbook.md#L247)
- **持有工具**：僅持有 **3.1.2 的專業工具**，避免與 Orchestrator 共用基礎工具造成責任混淆。[my_docs/pdf_review_agent_playbook.md:202](my_docs/pdf_review_agent_playbook.md#L202)
- **LLM 客戶端**：
  - 文字與視覺模型: `gpt-5-nano`。[python/packages/autogen-ext/src/autogen_ext/models/openai/_model_info.py:22](python/packages/autogen-ext/src/autogen_ext/models/openai/_model_info.py#L22)
  - 參數: `parallel_tool_calls=True`，以便必要時並行呼叫工具。[python/packages/autogen-ext/src/autogen_ext/models/openai/_openai_client.py:1252](python/packages/autogen-ext/src/autogen_ext/models/openai/_openai_client.py#L1252)

#### 3.2.2 Orchestrator Agent
- **職責**：負責整體工作流程的規劃、任務的分派、結果的審核與最終報告的生成，並於必要時重試或重新指派 Specialist。[my_docs/pdf_review_agent_playbook.md:249](my_docs/pdf_review_agent_playbook.md#L249)
- **持有工具**：持有 **3.1.1 的基礎工具** (`execute_shell`, `aggregate_findings`)，並透過 FunctionTool 方式觸發 Specialist 審查函式。[my_docs/pdf_review_agent_playbook.md:203](my_docs/pdf_review_agent_playbook.md#L203)
- **LLM 客戶端**：
  - 模型: `gpt-5-mini`。[python/packages/autogen-ext/src/autogen_ext/models/openai/_model_info.py:21](python/packages/autogen-ext/src/autogen_ext/models/openai/_model_info.py#L21)
- **職責流程**：
  1. 呼叫 `get_pdf_page_count` → 建立審查任務列表。[my_docs/implementation_plan_pdf_review_agent.md:76](my_docs/implementation_plan_pdf_review_agent.md#L76)
  2. 逐頁委派任務給 Specialist Agent。[my_docs/implementation_plan_pdf_review_agent.md:82](my_docs/implementation_plan_pdf_review_agent.md#L82)
  3. 接收 Specialist 產出的 `.jsonl` 檔案，並儲存於 `$ROOT_PATH`。[my_docs/pdf_review_agent_playbook.md:260](my_docs/pdf_review_agent_playbook.md#L260)
  4. **審核**：檢查檔案內容是否為合法的 `JSONL` 格式。若否，要求 Specialist 針對該頁重做。[my_docs/pdf_review_agent_playbook.md:261](my_docs/pdf_review_agent_playbook.md#L261)
  5. 所有頁面審核通過後，呼叫 `aggregate_findings` 工具將 `$ROOT_PATH` 下的所有 `.jsonl` 檔案合併。[my_docs/pdf_review_agent_playbook.md:261](my_docs/pdf_review_agent_playbook.md#L261)
  6. 將合併後的結果儲存為 `$ROOT_PATH/final_report.json`，任務結束。[my_docs/pdf_review_agent_playbook.md:262](my_docs/pdf_review_agent_playbook.md#L262)

---

## 4. 實作步驟與待辦
1. **規則清單整理**
   - 建立 `rules.yaml`，內容如下：
     ```yaml
     - id: UNRESOLVED_CITATION
       description: "引用標記為占位符 [?]，顯示參考文獻未完成解析或引用錯誤。"
       severity: "high"
       examples_ok: "正常的引用格式，如 (Author, 2023)。"
       examples_ng: "文字中出現 [?]、[? ?] 等占位符。"
     ```

2. **工具開發**
   - API Keys 應透過 `.env` 檔案進行管理，並在程式啟動時載入至環境變數。
   - <span style="color:red">TODO: 為每支工具撰寫單元測試（使用樣本 PDF）。</span>
   - <span style="color:red">TODO: 錯誤時回傳一致結構（含 `error_code`, `details`）。</span>

3. **Agent 組裝**
   - <span style="color:red">TODO: 定義 `TaskResult` 中 `stop_reason` 的使用方式，並與 Orchestrator 終止條件對應。</span>

4. **報告合併**
   - <span style="color:red">TODO: 實作 `aggregate_findings(directory: str)` 函式，需能正確讀取目錄下所有 p{頁碼}.jsonl 檔案並合併為一個 JSON 字串。</span>

5. **異常與重試策略**
   - <span style="color:red">TODO: 決定 Agent 層級的重試次數（例如 Specialist 格式錯誤最多重做幾次）、退避機制。</span>

6. **測試與驗證**
   - <span style="color:red">TODO: 準備標註過的 PDF 測試集。</span>
   - <span style="color:red">TODO: 建立 CI 任務執行工具、Agent 單元測試。</span>

---

## 5. 未決議事項（請逐項確認）
- [ ] <span style="color:red">TODO: 最終報告的命名與版本控制策略（目前固定為 final_report.json）。</span>
- [ ] <span style="color:red">TODO: 是否支援中途暫停/續跑，需保存 Orchestrator/Specialist 狀態。</span>
- [ ] <span style="color:red">TODO: 根據 `docs/design/03 - Agent Worker Protocol.md` 及 `docs/design/05 - Services.md` 評估是否需支援分散式 gRPC Worker Runtime 作為後續擴充。</span>

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
